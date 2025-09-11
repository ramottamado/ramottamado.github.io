---
layout: post
title: Setup Local DNS Server with CoreDNS & DNS Proxy Using Podman and systemd
date: 2025-09-11 18:25 +0700
categories:
  - linux
  - containerization
  - podman
tags:
  - linux
  - fedora
  - silverblue
  - podman
  - docker
  - dns
  - systemd
keywords:
  - coredns
  - dnsproxy
  - podman
  - dns
  - quadlet
  - systemd
---

In this article we will setup CoreDNS and DNS Proxy using Podman and systemd to serve DNS request for our homelab server or workstation.<!--more-->

## 0. Set up Podman pod

Create quadlet file `/etc/containers/systemd/ns.pod`.
```
[Unit]
Description=DNS Pod

[Pod]
PodName=ns
DNS=127.0.0.1
IP=10.88.0.2
HostName=ns.saoirse.home.arpa

[Install]
WantedBy=default.target
```

## 1. Set up DNS Proxy

Pull latest DNS Proxy official image.
```sh
sudo podman pull docker.io/adguard/dnsproxy:latest
```

Create config for DNS Proxy.
```yaml
# config.yaml
---
edns: true
listen-addrs:
  - "0.0.0.0"
listen-ports:
  - 54
max-go-routines: 0
ratelimit: 0
ratelimit-subnet-len-ipv4: 24
ratelimit-subnet-len-ipv6: 64
udp-buf-size: 0
upstream:
  - "https://1.1.1.1/dns-query"
  - "https://1.0.0.1/dns-query"
  - "https://8.8.8.8/dns-query"
  - "https://8.8.4.4/dns-query"
upstream-mode: 'load_balance'
timeout: '30s'
```

Save the file as `config.yaml`. We use port 54 because port 53 will be used by CoreDNS to serve DNS queries. DNS Proxy will act as DNS forwarder with the advantage of always connecting to DNS servers using HTTPS (DoH). We use both Cloudflare & Google public DNS servers with load balancing.

Then copy the config file to some directory, we will mount this directory to the container. I use `/var/opt/containers/dnsproxy`.
```sh
sudo mkdir -p /var/opt/containers/dnsproxy
sudo cp config.yaml /var/opt/containers/dnsproxy
```

Create quadlet file `/etc/containers/systemd/dnsproxy.container`.
```
[Unit]
Description=AdGuard DNS Proxy

[Service]
Restart=on-failure

[Container]
Pod=ns.pod
Image=docker.io/adguard/dnsproxy:latest
AutoUpdate=registry
CgroupsMode=no-conmon
Volume=/var/opt/containers/dnsproxy/config.yaml:/opt/dnsproxy/config.yaml:z
Memory=1g
```

## 2. Set up CoreDNS

Pull latest CoreDNS official image.
```sh
sudo podman pull docker.io/coredns/coredns:latest
```

Now we need to create configuration file for CoreDNS. Create `Corefile`.
```
. {
    forward . 0.0.0.0:54
    errors
}
```

This is the default config for our CoreDNS setup, ensuring all DNS queries will be forwarded to DNS Proxy. Since CoreDNS and DNS Proxy will run in the same pod, they will share the same host, hence `0.0.0.0`. We forward the queries to port 54 since DNS Proxy serve DNS requests on port 54.

Now we need to create zone files and reverse zone files for services running in our homelab. Create `db.saoirse.home.arpa`.
```
$ORIGIN saoirse.home.arpa.
$TTL 3600
@                   IN  SOA     ns.saoirse.home.arpa. tamado.tamado.codes. (
                        1       ; serial
                        7200    ; refresh
                        3600    ; retry
                        1209600 ; expire
                        3600    ; nxdomain ttl
                    )
                    IN  NS      ns.saoirse.home.arpa.
                    IN  A       10.88.0.1

ns                  IN  A       10.88.0.2
```

We can add other DNS records, e.g.
```
kafka-1             IN  A       10.88.0.11
kafka-2             IN  A       10.88.0.12
kafka-3             IN  A       10.88.0.13
postgres            IN  A       10.88.0.100
haproxy             IN  A       10.88.0.101
mail                IN  A       10.88.0.141
nifi                IN  A       10.88.0.181

calendar            IN  CNAME   haproxy
roundcube           IN  CNAME   haproxy

_mail._tcp          IN  SRV     0 100 80 mail

_8443._https.nifi   IN  HTTPS   0 .
```

Create reverse zone file `88.10.in-addr.arpa`.
```
$ORIGIN 88.10.in-addr.arpa.
$TTL 86400
@                   IN  SOA     ns.saoirse.home.arpa. tamado.tamado.codes. (
                        1       ; serial
                        7200    ; refresh
                        3600    ; retry
                        1209600 ; expire
                        3600    ; nxdomain ttl
                    )
                    IN  NS      ns.saoirse.home.arpa.

1.0                 IN  PTR     saoirse.home.arpa.
2.0                 IN  PTR     ns.saoirse.home.arpa.
11.0                IN  PTR     kafka-1.saoirse.home.arpa.
12.0                IN  PTR     kafka-2.saoirse.home.arpa.
13.0                IN  PTR     kafka-3.saoirse.home.arpa.
100.0               IN  PTR     postgres.saoirse.home.arpa.
101.0               IN  PTR     haproxy.saoirse.home.arpa.
141.0               IN  PTR     mail.saoirse.home.arpa.
181.0               IN  PTR     nifi.saoirse.home.arpa.
```

Now we need to make sure CoreDNS read these files for queries for these domains. Add these lines to the `Corefile`:
```
saoirse.home.arpa. {
    file /etc/coredns/db.saoirse.home.arpa
    log
    errors
}

88.10.in-addr.arpa. {
    file /etc/coredns/88.10.in-addr.arpa
    log
    errors
}
```

Now we need to copy the config files to a directory that will be mounted on the container. I will use `/var/opt/containers/coredns`. We will place `Corefile` on that directory and make a subdir for the zone files.
```sh
sudo mkdir -p /var/opt/containers/coredns/etc/coredns
sudo cp Corefile /var/opt/containers/coredns
sudo cp db.saoirse.home.arpa /var/opt/containers/coredns/etc/coredns
sudo cp 88.10.in-addr.arpa /var/opt/containers/coredns/etc/coredns
```

Then we create quadlet file `/etc/containers/systemd/coredns.container`.
```
[Unit]
Description=CoreDNS
After=dnsproxy.container
Requires=dnsproxy.container

[Service]
Restart=on-failure

[Container]
Pod=ns.pod
Image=docker.io/coredns/coredns:latest
AutoUpdate=registry
CgroupsMode=no-conmon
Volume=/var/opt/containers/coredns/Corefile:/Corefile:z
Volume=/var/opt/containers/coredns/etc/coredns:/etc/coredns:z
Memory=1g
```

## 3. Starting CoreDNS and DNS Proxy

Now we need to generate service files from the quadlet files. We simply need to reload systemd daemon.
```sh
sudo systemctl daemon-reload
```

Then we start the pod.
```sh
sudo systemctl start ns-pod
```

Observe the pod and the containers for any errors.
```sh
systemctl status ns-pod dnsproxy coredns
```

Test the DNS server.
```sh
dig @10.88.0.2 mail.saoirse.home.arpa A
dig @10.88.0.2 calendar.saoirse.home.arpa CNAME
dig @10.88.0.2 google.com A
dig @10.88.0.2 -x 10.88.0.100 PTR
```

If everything OK, and the addresses are resolved, we can use the pod IP (10.88.0.2) as our DNS server.

## 4. (Optional) Setting `systemd-resolved` to use CoreDNS as resolver

Create file `/etc/systemd/resolved.conf.d/99-coredns`.
```
[Resolve]
DNS=10.88.0.2
DNSOverTLS=no
Domains=~.
MulticastDNS=yes
Cache=yes
ReadEtcHosts=yes
```

We need to put `Domains=~.` to make sure it's going to be the default resolver for all networks. Then we need to restart `systemd-resolved`
```sh
sudo systemctl restart systemd-resolved
```