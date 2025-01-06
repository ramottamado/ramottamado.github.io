---
layout: post
title: Setting up home directory encryption with LUKS on Fedora (and auto-mount on
  login)
date: 2024-02-04 11:50 +0700
categories:
  - linux
  - pam
  - security
tags:
  - linux
  - fedora
  - silverblue
keywords:
  - pam_mount
  - luks
  - encryption
---

I've been toying around with disk encryption with [Silverblue](https://fedoraproject.org/atomic-desktops/silverblue/),
and to be honest, if you only care about encrypting the home directory with the same passphrase as your login password,
using `crypttab` to auto-mount the encrypted partition at boot is a little annoying because you need to type the
password twice. The other solution is using `pam_mount`, but because some user services still runs a few seconds after
logout, `pam_mount` will never unmount your encrypted partition on logout. Facing this weird issue, I come up with a
workaround so I can still use `pam_mount` while ensuring the encrypted home directory will always be unmounted after
user logout.

## Setting up `pam_mount`


### The dreaded XML

To setup `pam_mount` to mount the encrypted home directory at user login, we need to create a `pam_mount.conf.xml` in
`/etc/security`. I use LUKS-encrypted partition as my home directory, so the relevant config will look like:

```xml
<volume
  user="ramot"
  fstype="auto"
  path="/dev/disk/by-uuid/<LUKS_PARTITION_UUID>"
  mountpoint="/var/home"
  options="crypto_name=homedir,allow_discard,defaults,noatime"
/>

<mkmountpoint enable="0" />
```

If you want to mount multiple BTRFS subvolumes, the config will be a little bit different[^1]. An example of two
subvolumes mount looks like:

```xml
<volume
  user="ramot"
  fstype="auto"
  path="/dev/disk/by-uuid/<LUKS_PARTITION_UUID>"
  mountpoint="/var/home"
  options="crypto_name=homedir,allow_discard,defaults,noatime,subvol=<HOME_SUBVOL>"
/>

<volume
  user="ramot"
  path="/dev/disk/by-uuid/<OTHER_SUBVOL_PARTITION_UUID>"
  mountpoint="/var/other"
  options="allow_discard,defaults,noatime,subvol=<OTHER_SUBVOL>"
/>

<mkmountpoint enable="0" />
```

Notice that I disable `mkmountpoint` directive, because I use SELINUX and I need to ensure that the `/var/home`
directory will always have the correct SELINUX security context. I also set the option `crypto_name` to `homedir` since
this will help us later in deciding the mapper name for the encrypted partition. You can also enable `relatime` if you
need Linux `atime` that much.

### The PAM directives

We will set the PAM directives for specifically GDM logins, I don't care about SSH & shell logins since I only SSH to my
machine if my user is already logged in and I rarely use shell login. We will start with the directives for GDM login
with password (fingerprint login is a little complicated, we'll get back to this later), fire up your favorite editor
and edit `/etc/pam.d/gdm-password` to looks like this:

```
auth     [success=done ignore=ignore default=bad] pam_selinux_permit.so
auth        substack      password-auth
auth        optional      pam_mount.so
auth        optional      pam_gnome_keyring.so
auth        include       postlogin

account     required      pam_nologin.so
account     include       password-auth

password    substack       password-auth
-password   optional       pam_gnome_keyring.so use_authtok

session     required      pam_selinux.so close
session     required      pam_loginuid.so
session     required      pam_selinux.so open
session     optional      pam_keyinit.so force revoke
session     required      pam_namespace.so
session     include       password-auth
session     optional      pam_mount.so
session     optional      pam_gnome_keyring.so auto_start
session     include       postlogin
```

We need to add `pam_mount.so` on auth & session module interface, just like in the documentation[^2]. We place it after
`password-auth` directive to enable `pam_mount`to get the passphrase from the password field in GDM login (since we only
want to type the passphrase once).

### The post-logout `systemd` hook

Now we need to override a `systemd` service to make sure the encrypted partition is unmounted after user logout. We
start by creating a shell script to check the mountpoints and unmount them, we also need to lock the LUKS encrypted
partition after we successfully unmount all the mountpoints. The shell script to do this will looks like this:

```bash
#!/bin/bash

HOMEDIR="/var/home"

# Try to unmount...
while /usr/bin/mountpoint -q $HOMEDIR
do
    /usr/bin/umount $HOMEDIR
    /usr/bin/mountpoint -q $HOMEDIR && sleep 5
done

# Notice that we set the crypto_name to 'homedir' on pam_mount.conf.xml 
/usr/sbin/cryptsetup close homedir

# In case the encrypted partition is already closed/locked, always return 0
# https://man.archlinux.org/man/cryptsetup.8.en#RETURN_CODES
if [ $? -eq 4 ]
then
    exit 0
fi
```

You will need to modify the script a little bit if you mount several subvolumes/partitions with `pam_mount`. For me,
this script is sufficient. Save the script as `/usr/local/bin/unmount-homedir` and run this command to start
editing the `systemd` service to call the script after successful user logout (assuming your UID is 1000):

```shell
sudo systemctl edit user-runtime-dir@1000.service
```

Then you need to add this directive:

```
[Service]
ExecStop=/usr/local/bin/unmount-homedir
```

This will tell `user-runtime-dir@1000.service` to run `/usr/local/bin/unmount-homedir` after your user has been logged
out. Or you can create `/etc/systemd/system/user-runtime-dir@1000.service.d/override.conf` with the content as above.

*Et voilà!* Now everything is set up accordingly.

## Bonus: Enable fingerprint authentication on everything except login

I told you before that
> fingerprint login is a little complicated

and we'll get back to this. Well, the problem with fingerprint login is the PAM stack won't be getting the passphrase
for the encrypted partition, causing `pam_mount` to not able to mount the encrypted partition at all. Theoretically, we
can work through this by disabling fingerprint login usinf `dconf` directives, but Fedora is not helping by locking down
the `/org/gnome/login-screen/enable-fingerprint-authentication` key. To solve this, we will need to create our custom
`authselect` profile to unlock this directive, and then set GDM to disable fingerprint on login.

### Creating custom `authselect` profile

Run this command to create the custom profile (you can name it everything) with `sssd` as base:

```shell
sudo authselect create-profile sssd-disable-fp-on-login -b sssd --symlink-meta --symlink-nsswitch --symlink-pam
```

The `--symlink-meta`, `--symlink-nsswitch` and `--symlink-pam` is needed because we only care about dconf locks and not
the entire PAM stack and nsswitch.

### Removing `dconf` locks

Fire up your favorite editor again and edit `/etc/authselect/custom/sssd-disable-fp-on-login/dconf-locks`. The file will
look like this:

```
/org/gnome/login-screen/enable-smartcard-authentication
/org/gnome/login-screen/enable-fingerprint-authentication
/org/gnome/login-screen/enable-password-authentication
/org/gnome/settings-daemon/peripherals/smartcard/removal-action {include if "with-smartcard-lock-on-removal"}
```

Now we only need to remove the line containing `enable-fingerprint-authentication`. Make sure the content of the file is
the same as this snippet:

```
/org/gnome/login-screen/enable-smartcard-authentication
/org/gnome/login-screen/enable-password-authentication
/org/gnome/settings-daemon/peripherals/smartcard/removal-action {include if "with-smartcard-lock-on-removal"}
```

### Applying custom `authselect` profile

Run this command to apply the profile that we created earlier:

```shell
sudo authselect select custom/sssd-disable-fp-on-login with-silent-lastlog with-mdns4 with-fingerprint
```

### Adding `dconf` directive to disable fingerprint login

Now we need to override `dconf` configuration for GDM login. Edit `/etc/dconf/db/gdm.d/99-fingerprint-no-login` to
create custom configuration for `dconf`. The file content should be like this:

```
[org/gnome/login-screen]
enable-fingerprint-authentication=false
```

Save the file and then run this command:

```shell
sudo dconf update
```

Now fingerprint will be disabled on GDM login, but enabled elsewhere.


[^1]: [Mount multiple subvolumes of a LUKS encrypted BTRFS through pam_mount](https://binfalse.de/2018/11/28/mount-multiple-subvolumes-of-a-luks-encrypted-btrfs-through-pam-mount/)
[^2]: [Arch Linux pam_mount manual](https://man.archlinux.org/man/pam_mount.8)