# Deployment and Verification Guide

## 1. Initial Deployment

To start the virtual environment, execute the following command in the project root directory:

```
vagrant up
```

This brings up both nodes: `dns` (`192.168.57.10`) and `web` (`192.168.57.11`).

---

## 2. DNS Server Configuration (`ns1.olimpo.test`)

Access the DNS machine:

```
vagrant ssh dns
```

### 2.1 Forwarding and DNSSEC (`/etc/bind/named.conf.options`)

Ensure that the BIND options file includes Cloudflare (`1.1.1.1`) as forwarder and enforces DNSSEC validation:

```
options {
    directory "/var/cache/bind";

    forwarders {
        1.1.1.1;
    };

    dnssec-validation yes;
    listen-on-v6 { any; };
};
```

### 2.2 Zone Declaration (`/etc/bind/named.conf.local`)

Declare direct and reverse master zones:

```
zone "olimpo.test" {
    type master;
    file "/etc/bind/db.olimpo.test";
};

zone "57.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.57";
};
```

### 2.3 Forward Zone File (`/etc/bind/db.olimpo.test`)

Define authoritative NS records, A records for hosts, and CNAME aliases for the websites:

```
$TTL    604800
@       IN      SOA     ns1.olimpo.test. admin.olimpo.test. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.olimpo.test.
ns1     IN      A       192.168.57.10
mercurio IN     A       192.168.57.11

; Aliases for virtual hosts
apolo   IN      CNAME   mercurio.olimpo.test.
atenea  IN      CNAME   mercurio.olimpo.test.
```

### 2.4 Reverse Zone File (`/etc/bind/db.192.168.57`)

Configure PTR records for IP-to-hostname reverse lookups:

```
$TTL    604800
@       IN      SOA     ns1.olimpo.test. admin.olimpo.test. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.olimpo.test.
10      IN      PTR     ns1.olimpo.test.
11      IN      PTR     mercurio.olimpo.test.
```

### 2.5 Validation and Service Restart

Validate syntax and restart the BIND9 service:

```
sudo named-checkconf
sudo named-checkzone 57.168.192.in-addr.arpa /etc/bind/db.192.168.57
sudo systemctl restart bind9
```

<img width="1232" height="260" alt="Screenshot 2026-11-10 122757" src="https://github.com/user-attachments/assets/5812cf7c-daa1-4c9d-903c-b386cd22a13d" />
> *(Shows `named-checkconf`, `named-checkzone` returning OK, and `systemctl restart bind9`)*

Verify DNS queries locally with `dig`:

```
dig @127.0.0.1 apolo.olimpo.test +short
dig @127.0.0.1 atenea.olimpo.test +short
dig @127.0.0.1 -x 192.168.57.11 +short
```

<img width="814" height="150" alt="Screenshot 2026-11-10 122827" src="https://github.com/user-attachments/assets/6dc33f12-75c6-4f17-b4fc-598ac53d12fd" />
> *(Shows DNS lookups returning CNAME and PTR resolution for `mercurio.olimpo.test`)*

Log out of the machine:

```
exit
```

---

## 3. Web Server Configuration (`mercurio.olimpo.test`)

Access the web machine:

```
vagrant ssh web
```

### 3.1 DNS Resolver Verification

Verify that `/etc/resolv.conf` targets `192.168.57.10`:

```
cat /etc/resolv.conf
```

<img width="660" height="175" alt="Screenshot 2026-11-10 123301" src="https://github.com/user-attachments/assets/e419dd5c-189c-4ed0-9259-e9a294a1e577" />
> *(Shows `domain olimpo.test`, `search olimpo.test`, and `nameserver 192.168.57.10`)*

### 3.2 Virtual Host Configurations

#### Apolo (`/etc/apache2/sites-available/apolo.olimpo.test.conf`)

```
<VirtualHost *:80>
    ServerName apolo.olimpo.test
    DocumentRoot /var/www/apolo.olimpo.test

    <Directory /var/www/apolo.olimpo.test>
        Options +Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorDocument 403 "You do not have permission to access apolo.olimpo.test"
    ErrorDocument 404 /messages/404.html

    ErrorLog ${APACHE_LOG_DIR}/apolo.olimpo.test.error.log
    CustomLog ${APACHE_LOG_DIR}/apolo.olimpo.test.access.log combined
</VirtualHost>
```

#### Atenea (`/etc/apache2/sites-available/atenea.olimpo.test.conf`)

```
<VirtualHost *:80>
    ServerName atenea.olimpo.test
    DocumentRoot /var/www/atenea.olimpo.test

    <Directory /var/www/atenea.olimpo.test>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorDocument 403 "You do not have permission to access atenea.olimpo.test"
    ErrorDocument 404 /messages/404.html

    ErrorLog ${APACHE_LOG_DIR}/atenea.olimpo.test.error.log
    CustomLog ${APACHE_LOG_DIR}/atenea.olimpo.test.access.log combined
</VirtualHost>
```

### 3.3 Enabling Sites and Reloading Apache

Enable both virtual hosts and reload the Apache service:

```
sudo a2ensite apolo.olimpo.test.conf
sudo a2ensite atenea.olimpo.test.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

<img width="849" height="346" alt="Screenshot 2026-11-10 123832" src="https://github.com/user-attachments/assets/b5cc5338-79f7-4322-a4b5-5420a9dba847" /> 
> *(Shows `a2ensite` enabling both configs, `Syntax OK`, and service reload)*

---

## 4. Verification from Host Machine (Windows 11)

### 4.1 Host File Configuration

To enable domain resolution from the Windows 11 host, append the following records to `C:\Windows\System32\drivers\etc\hosts`:

```
192.168.57.11 apolo.olimpo.test
192.168.57.11 atenea.olimpo.test
```

<img width="912" height="683" alt="Screenshot 2026-11-10 124141" src="https://github.com/user-attachments/assets/3c7663d2-f2fd-486a-96df-d3effbd3b77a" />
> *(Shows `hosts` file configuration inside VS Code)*

### 4.2 Browser Verification Tests

#### Test 1: Apolo Root Directory (`http://apolo.olimpo.test`)
Because indexing is enabled (`Options +Indexes`) and no index file exists, the directory listing is displayed.

<img width="1919" height="1030" alt="Screenshot 2026-11-10 124237" src="https://github.com/user-attachments/assets/5e1c813d-60a8-4d88-afb5-7071bd9155be" />
> *(Shows browser displaying "Index of /" with `apolo.html`, `apolo1.txt`, and `apolo2.txt`)*

#### Test 2: Atenea Specific Resource (`http://atenea.olimpo.test/atenea.html`)
The resource exists and permissions are granted, rendering the page content.

<img width="1910" height="1034" alt="Screenshot 2026-11-10 131918" src="https://github.com/user-attachments/assets/031c9cf1-32af-4766-8b04-247a7140be04" />
> *(Shows browser displaying "Atenea site")*

#### Test 3: Atenea Root Directory Forbidden (`http://atenea.olimpo.test`)
Because directory indexing is disabled (`Options -Indexes`) and no index file exists, Apache denies access and returns the custom 403 error string.

<img width="1919" height="1030" alt="Screenshot 2026-11-10 124316" src="https://github.com/user-attachments/assets/c0a6ffee-2f7b-48ca-aed8-57c0399b5023" />
> *(Shows browser displaying "You do not have permission to access atenea.olimpo.test")*

#### Test 4: Atenea Missing Resource (`http://atenea.olimpo.test/index.html`)
Because `index.html` does not exist, the custom 404 handler serves `/messages/404.html`.

<img width="1232" height="260" alt="Screenshot 2026-11-10 122757" src="https://github.com/user-attachments/assets/70cb274e-4eb9-48e1-98f0-85374eedb346" />
> *(Shows browser displaying "Resource not found on atenea.olimpo.test")*
