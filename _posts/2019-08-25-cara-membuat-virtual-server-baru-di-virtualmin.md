---
layout: post
title: Cara Membuat Virtual Server Baru di Virtualmin
date: '2019-08-25 11:09 +0700'
description: Langkah-langkah configure dan setup virtual server baru di Virtualmin.
excerpt: Langkah-langkah configure dan setup virtual server baru di Virtualmin.
meta: Virtualmin virtual server
categories:
  - tutorial
  - linux
tags:
  - virtualmin
  - webserver
preview: /assets/images/virtualmin.png
redirect_from:
  - /cara-membuat-virtual-server-baru-di-virtualmin/
lastmod: '2022-02-07 23:36 +0700'
---
_Hello_ pembaca sekalian. Kali ini saya akan menjelaskan langkah-langkah cepat untuk membuat _virtual server_ di **Virtualmin**.

## 1. Login ke Virtualmin ##

{% include image.html src="/assets/images/virtualmin1.webp" width="800" height="450" alt="Buka Virtualmin" %}

_Login_ ke **Virtualmin**. Setelah _login_ akan muncul tampilan seperti ini.

## 2. Create Virtualserver ##

{% include image.html src="/assets/images/virtualmin2.webp" width="800" height="450" alt="Membuat Virtualserver baru" %}

Klik _Create Virtualserver_ pada _sidebar_ sebelah kiri. Pilih jenis _server_ yang akan dibuat. Di sini kita akan membuat _Top-level server_. isikan juga nama _domain_ yang diinginkan dan _password_ untuk _user administration_.

## 3. Parameter ##

{% include image.html src="/assets/images/virtualmin3.webp" width="800" height="450" alt="Mengisi parameter" %}

_Scroll_ ke bawah. Pada pilihan _enabled features_ ceklis semua fitur seperti pada gambar. Jika _domain_ ingin menggunakan SSL, ceklis juga pilihan ***Setup SSL website too?***

Untuk pilihan _IP Address and forwarding_, biarkan saja seperti pada gambar. Cek lagi apakah semua langkah sudah benar, kemudian klik tombol _Create Server_, maka akan muncul tampilan seperti di bawah ini, kemudian tekan tombol _Return to virtual server details_

{% include image.html src="/assets/images/virtualmin4.webp" width="500" height="700" alt="Tunggu konfigurasi baru diterapkan" %}

## 4. Selesai ##

{% include image.html src="/assets/images/virtualmin5.webp" width="800" height="450" alt="Selesai" %}

Selesai! _Virtual server_ baru sudah dapat langsung diakses melalui _domain_ yang sudah dipilih.
