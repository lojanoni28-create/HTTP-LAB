# Deployment and Verification Guide

## 1. Initial Deployment

To start the virtual environment, execute the following command in the project root directory:

```bash
vagrant up
```

This brings up both nodes: `dns` (`192.168.57.10`) and `web` (`192.168.57.11`).

---

## 2. DNS Server Configuration (`ns1.olimpo.test`)

Access the DNS machine:

```bash
vagrant ssh dns
```

### 2.1 Forwarding and DNSSEC (`/etc/bind/named.conf.options`)

Ensure that the BIND options file includes Cloudflare (`1.1.1.1`) as forwarder and enforces DNSSEC validation:

```named
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

```named
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

```dns
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

```dns
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

```bash
sudo named-checkconf
sudo named-checkzone 57.168.192.in-addr.arpa /etc/bind/db.192.168.57
sudo systemctl restart bind9
```

> ![](<Screenshot 2026-11-10 122757-1.png>) 
> *(Shows `named-checkconf`, `named-checkzone` returning OK, and `systemctl restart bind9`)*

Verify DNS queries locally with `dig`:

```bash
dig @127.0.0.1 apolo.olimpo.test +short
dig @127.0.0.1 atenea.olimpo.test +short
dig @127.0.0.1 -x 192.168.57.11 +short
```

> ![](<Screenshot 2026-11-10 122827-1.png>)  
> *(Shows DNS lookups returning CNAME and PTR resolution for `mercurio.olimpo.test`)*

Log out of the machine:

```bash
exit
```

---

## 3. Web Server Configuration (`mercurio.olimpo.test`)

Access the web machine:

```bash
vagrant ssh web
```

### 3.1 DNS Resolver Verification

Verify that `/etc/resolv.conf` targets `192.168.57.10`:

```bash
cat /etc/resolv.conf
```

> ![](<Screenshot 2026-11-10 123301-1.png>)  
> *(Shows `domain olimpo.test`, `search olimpo.test`, and `nameserver 192.168.57.10`)*

### 3.2 Virtual Host Configurations

#### Apolo (`/etc/apache2/sites-available/apolo.olimpo.test.conf`)

```apache
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

```apache
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

```bash
sudo a2ensite apolo.olimpo.test.conf
sudo a2ensite atenea.olimpo.test.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

> ![](<Screenshot 2026-11-10 123832-1.png>) 
> *(Shows `a2ensite` enabling both configs, `Syntax OK`, and service reload)*

---

## 4. Verification from Host Machine (Windows 11)

### 4.1 Host File Configuration

To enable domain resolution from the Windows 11 host, append the following records to `C:\Windows\System32\drivers\etc\hosts`:

```text
192.168.57.11 apolo.olimpo.test
192.168.57.11 atenea.olimpo.test
```

> ![](<Screenshot 2026-11-10 124141-1.png>)
> *(Shows `hosts` file configuration inside VS Code)*

### 4.2 Browser Verification Tests

#### Test 1: Apolo Root Directory (`http://apolo.olimpo.test`)
Because indexing is enabled (`Options +Indexes`) and no index file exists, the directory listing is displayed.

> ![](<Screenshot 2026-11-10 124237-1.png>)  
> *(Shows browser displaying "Index of /" with `apolo.html`, `apolo1.txt`, and `apolo2.txt`)*

#### Test 2: Atenea Specific Resource (`http://atenea.olimpo.test/atenea.html`)
The resource exists and permissions are granted, rendering the page content.

> ![](<Screenshot 2026-11-10 131918-1.png>)  
> *(Shows browser displaying "Atenea site")*

#### Test 3: Atenea Root Directory Forbidden (`http://atenea.olimpo.test`)
Because directory indexing is disabled (`Options -Indexes`) and no index file exists, Apache denies access and returns the custom 403 error string.

> ![](<Screenshot 2026-11-10 124316-1.png>) 
> *(Shows browser displaying "You do not have permission to access atenea.olimpo.test")*

#### Test 4: Atenea Missing Resource (`http://atenea.olimpo.test/index.html`)
Because `index.html` does not exist, the custom 404 handler serves `/messages/404.html`.

> ![](<Screenshot 2026-11-10 131907-1.png>)  
> *(Shows browser displaying "Resource not found on atenea.olimpo.test")*