# Apa itu Pritunl ?
Pritunl adalah server VPN perusahaan sumber terbuka yang mendukung protokol OpenVPN, IPsec, dan WireGuard. Hal ini memungkinkan pengguna untuk membuat jaringan pribadi virtual yang aman, menghubungkan jaringan cloud, dan menyediakan akses jarak jauh dengan fitur keamanan tingkat lanjut. Aspek unik termasuk peering VPC multi-cloud, otentikasi perangkat TPM dan Apple Secure Enclave, dan sistem plugin Python yang dapat disesuaikan.

Berikut adalah step by step untuk setup a self-hosted Pritunl VPN di AWS EC2 instance.

Task yang akan di implementasikan:
1. How to install and configure Pritunl
2. How to create and set up VPN users (clients)
3. How to connect and authenticate users using a VPN client app
4. How to configure split tunneling to route only AWS traffic through the VPN

## Setting Pritunl VPN on EC2 Instance
Pritunl menggunakan `MongoDB` untuk mengakses user data dan `VPN Configuration`. Di setup ini, kita akan menginstall `MongoDB` di `EC2 Instance` yang sama, untuk `production HA (High Availability)` setup. Sebaikan `database` di tempatkan pada instance yang terpisah dan khusus. 

## How it work ?
- Berikut cara kerjanya.
Pengguna pertama-tama membuat **tunnel VPN** ke server **Pritunl** yang berada di **public subnet AWS** menggunakan **Pritunl VPN client**.

Setelah proses autentikasi berhasil, sebuah **tunnel yang aman dan terenkripsi** akan dibuat melalui internet dari **workstation lokal** menuju **server VPN Pritunl**.

Ketika pengguna mencoba mengakses **resource private AWS** yang berada pada jaringan yang telah dikonfigurasi di Pritunl, koneksi dari workstation lokal akan mencapai server VPN melalui **tunnel terenkripsi** tersebut. Setelah itu, trafik akan **didekripsi** oleh server VPN sehingga dapat mengakses **resource private di AWS**.

## Pritunl VPN Workflow 
Berikut diagram yang memperlihatkan flow traffic mengakses AWS Private Resources menggunakan Pritunl VPN server.
![workflow-pritunl](./images/workflow-pritunl.png)

## Implementation

Di implementasi awal kita memerlukan untuk setup:
1. AWS EC2 Instance, saya menggunakan spesifikasi instances berikut
	-  instance type: t2.large 
	-  storage: 15 GB
	-  VPC: default
	-  Security Group: default vpc
2. Security Group, Inbound rule: 
	![security-group](./images/inbound-rule.png)

Lakukan remote pada instance, untuk implementasi konfigurasi `pritunl vpn`. bisa menggunakan `connect with browser` atau `melakukan ssh dari cmd menggunakan key pair` 

### Menambahkan Pritunl, OpenVPN, dan MongoDB Repositories

Menambahkan repository untuk menginstall Pritunl, OpenVPN, dan MongoDB

1. menambahkan repo `mongodb`
```bash
sudo tee /etc/apt/sources.list.d/mongodb-org.list << EOF
deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse
EOF
```

2. menambahkan repo `openvpn`
```bash
sudo tee /etc/apt/sources.list.d/openvpn.list << EOF
deb [ signed-by=/usr/share/keyrings/openvpn-repo.gpg ] https://build.openvpn.net/debian/openvpn/stable noble main
EOF
```

3. menambahkan repo `pritunl vpn`
```bash
sudo tee /etc/apt/sources.list.d/pritunl.list << EOF
deb [ signed-by=/usr/share/keyrings/pritunl.gpg ] https://repo.pritunl.com/stable/apt noble main
EOF
```

### Menambahkan GPG Keys

Install GPG tools untuk menghandle verifikasi keys dari packages resource
```bash
sudo apt --assume-yes install gnupg
```

Menambahkan  GPG Keys untuk packages

Download GPG Keys `MongoDB` packages
```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor --yes
```

Download GPG Keys `OpenVPN` packages
```bash
curl -fsSL https://raw.githubusercontent.com/pritunl/pgp/master/pritunl_repo_pub.asc | sudo gpg -o /usr/share/keyrings/pritunl.gpg --dearmor --yes
```

Download GPG Keys `Pritunl` packages
```bash
curl -fsSL https://raw.githubusercontent.com/pritunl/pgp/master/pritunl_repo_pub.asc | sudo gpg -o /usr/share/keyrings/pritunl.gpg --dearmor --yes
```

### Install Pritunl, OpenVPN, MongoDB, dan Wireguard

`Pritunl` bukan sebuah `VPN` secara langsung, sebaliknya `Pritunl` berfungsi sebagai layer management untuk server `vpn` seperti `OpenVPN` dan `WireGuard`. `Pritunl` menghandle bagian user management, connection routing, dan menyediakan antarmuka web.


Pertama lakukan update packages dan install packages yang dibutuhkan:
```bash
sudo apt update
sudo apt --assume-yes install pritunl openvpn mongodb-org wireguard wireguard-tools
```

Disable firewall untuk menghindari masalah connection
```bash
sudo ufw disable
```

Start dan enable Pritunl dan MongoDB Services
```bash
sudo systemctl start pritunl mongod
sudo systemctl enable pritunl mongod
```


`Pritunl` mendukung protokol `OpenVPN` dan `WireGuard`, perlu menginstall setidaknya salah satu dari keduanya. Secara default, `pritunl` menggunakan `OpenVPN`. `Pritunl` akan membuat file konfigurasi sementara di direktori `/tmp`. Kita dapat melihat detail koneksi dengan menggunakan argumen `--management` yang terdapat di file konfigurasi sementara tersebut.