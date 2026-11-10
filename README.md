# Apache Virtual Hosts and DNS Infrastructure Lab

## Overview

This repository contains an automated multi-machine infrastructure configured with Vagrant and VirtualBox. The project deploys two interconnected Linux virtual machines (Debian Bookworm) to demonstrate DNS zone delegation, domain name aliasing, and Apache2 name-based virtual hosting.

The primary objective is to configure an authoritative BIND9 DNS service and serve multiple distinct virtual hosts (`apolo.olimpo.test` and `atenea.olimpo.test`) from a single Apache web server (`mercurio.olimpo.test`), implementing differentiated directory indexing policies and customized HTTP error handling (403 Forbidden and 404 Not Found).

## Architecture and Network Design

The environment is deployed on an isolated private network (`192.168.57.0/24`):

| Node | Hostname | IP Address | Service Roles |
|---|---|---|---|
| `dns` | `ns1.olimpo.test` | `192.168.57.10` | Authoritative DNS (BIND9), Forwarder (`1.1.1.1`), DNSSEC |
| `web` | `mercurio.olimpo.test` | `192.168.57.11` | Web Server (Apache2), Virtual Hosts (`apolo`, `atenea`) |

### Virtual Hosts Specifications

* **apolo.olimpo.test**:
  * Root directory: `/var/www/apolo.olimpo.test`
  * Directory Indexing: Enabled (`Options +Indexes`). Requests without a specific resource display directory contents.
  * Custom 403: Text string `"You do not have permission to access apolo.olimpo.test"`.
  * Custom 404: Static document located at `/messages/404.html`.
  * Logging: Independent access and error logs.

* **atenea.olimpo.test**:
  * Root directory: `/var/www/atenea.olimpo.test`
  * Directory Indexing: Disabled (`Options -Indexes`). Requests without a specific resource return a 403 Forbidden error.
  * Custom 403: Text string `"You do not have permission to access atenea.olimpo.test"`.
  * Custom 404: Static document located at `/messages/404.html`.
  * Logging: Independent access and error logs.

## Repository Structure

```text
.
├── README.md
├── Vagrantfile
├── dns-provision.sh
├── apache2-provision.sh
├── doc/
│   ├── README.md
│   └── images/
└── files/
    ├── apolo.olimpo.test/
        ├── messages
            ├── 404.html
        ├── apolo.html
        ├── apolo1.txt
        ├── apolo2.txt
    ├── atenea.olimpo.test/
        ├── messages
            ├── 404.html
        ├── atenea.html
        ├── atenea1.txt
        ├── atenea2.txt
    ├── mercurio.olimpo.test/
    └── ns1.olimpo.test/
```

For complete step-by-step provisioning instructions, configuration files, and visual validation, refer to `doc/README.md`.