# DNS & Web Server Deployment on Ubuntu Server 

## Overview 
Deployed a DNS Server and Web Server on Ubuntu Server as part of a self-study infrastructure project, simulating a small enterprise setup. 

## Tech Stack 
- OS: Ubuntu Server
- DNS: BIND9
- Web Server: Apache2

## Setup Overview
  This setup simulates how a small network resolves and serves a website internally. The DNS Server resolves [domain] to the Web Server's IP; Apache serves the site on that Domain.

## Prerequisites
- Ubuntu Server installed
- Root/Sudo access
- Static IP configured

## Installation & Configuration 

### 1. Install Packages
```bash
sudo apt update
sudo apt install bind9 apache2 -y
```
### 2. Configure BIND9 
BIND9 needs to be told which domains (zones) it is authoritative for, 
and where to find the DNS records for each one. This is done in two steps: 
first registering the zones in `named.conf.local`, then creating the 
actual zone files that hold the records.
#### a)  Define the zones in named.conf.local 
Edit `/etc/bind/named.conf.local` to declare the forward zone (resolves 
domain names to IP addresses) and the reverse zone (resolves IP addresses 
back to domain names). 
​```bash
sudo vim /etc/bind/named.conf.local
​```
```
zone "okikiola.xyz" {
      type master;
      file "/etc/bind/db.okikiola.xyz";
};
zone "18.168.192.in-addr.arpa" {
      type master;
      file "/etc/bind/db.192";
};
```
    
- The first block makes this server **authoritative** for `okikiola.xyz`, 
  meaning it's the source of truth for that domain's records, and points to 
  the zone file where those records live.
- The second block is the **reverse zone**. Its name is the network portion 
  of your IP range written backwards (e.g. for `192.168.18.x`, the reverse 
  zone is `18.168.192.in-addr.arpa`). This lets tools like `nslookup` or 
  `dig -x` resolve an IP address back to a hostname.

#### b) Create the forward zone file
```bash
sudo cp /etc/bind/db.local /etc/bind/db.okikiola.xyz
sudo vim /etc/bind/db.okikiola.xyz
```
```
$TTL    604800
@       IN      SOA     okikiola.xyz. admin.okikiola.xyz. (
                              3      ; Serial
                           604800    ; Refresh
                           604800    ; Refresh
                            86400    ; Retry
                          2419200    ; Expire
                            604800 ) ; Negative Cache TTL
@      IN       NS       ns.okikiola.xyz.
@      IN       A        192.168.18.101
ns     IN       A        192.168.18.101
www    IN       A        192.168.18.101
```
This file holds the actual **A records**, mapping hostnames (`www.okikiola.xyz`) to IP addresses (`192.168.18.101`). This is what lets a client ask " what's the IP for www.okikiola.xyz?" and get an answer. 
#### c) Create the reverse zone file
```bash
sudo cp /etc/bind/db.127 /etc/bind/db.192
sudo vim /etc/bind/db.192
```
```
$TTL    604800
@       IN      SOA     okikiola.xyz. admin.okikiola.xyz. (
                               3        ; Serial
                          604800        ; Refresh
                           86400        ; Retry
                         2419200        ; Expire
                          604800 )      ; Negative Cache TTL
@       IN       NS      ns.okikiola.xyz.
101     IN       PTR     ns.okikiola.xyz.
```
This file holds the **PTR record**, which does the opposite of the forward 
zone — it maps the IP address back to a hostname, so `dig -x 192.168.18.101` 
returns `ns.okikiola.xyz` instead of an empty result.
#### d) Check syntax and restart
```bash
sudo named-checkconf
sudo named-checkzone okikiola.xyz /etc/bind/db.okikiola.xyz
sudo systemctl restart bind9
```
### 3. Configure Apache2
Apache needs a **virtual host** pointing to your domain, and a **document root** where the site files live.
```bash
sudo mkdir /var/www/okikiola.xyz
sudo vim /etc/apache2/sites-available/okikiola.xyz.conf
```
```
<VirtualHost *:80>
     ServerName okikiola.xyz
     ServerAlias www.okikiola.xyz
     DocumentRoot /var/www/okikiola.xyz
     ErrorLog ${APACHE_LOG_DIR}/okikiola.xyz-error.log
     CustomLog ${APACHE_LOG_DIR}/okikiola.xyz-access.log combined
</VirtualHost>
```
- `ServerName` ties this virtual host to the domain resolved by DNS.
- `DocumentRoot` is the folder Apache serves files from when that domain is requested.

Enable the site and reload Apache:
```bash
sudo a2ensite okikiola.xyz.conf
sudo systemctl reload apache2
```
### 4. Restart & Enable Services
```bash
sudo systemctl restart bind9
sudo systemctl restart apache2
```
## Testing & Verification
​### DNS Resolution
​```bash
dig okikiola.xyz
nslookup okikiola.xyz
​```
![DNS resolution test](screenshots/dig-test.png)
Confirms the domain correctly resolves to the web server's IP address.
### Web Server Access
Navigated to `http://okikiola.xyz` in a browser to confirm Apache is serving the site using the domain name, not just the raw IP.
![Web server running](screenshots/apache-test.png)
### Reverse DNS
​```bash
dig -x 192.168.18.101
​```
![DNS resolution test](screenshots/dig2-test.png)
Confirms the IP resolves back to the correct hostname.
