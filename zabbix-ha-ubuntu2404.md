# Zabbix 7 High Availability on Ubuntu 24.04

Two Zabbix Server nodes, an external MariaDB server, Keepalived VIP, and
HTTPS frontend

> **Scope and accuracy**
>
> This README documents the lab architecture and the configuration
> decisions used during deployment. Replace passwords and certificate
> details with your own values before using these examples. The database
> schema must be imported **once only** because both Zabbix Server nodes
> use the same database.
>
> The deployment was tested with Zabbix 7.0.31 on Ubuntu 24.04.
> Keepalived was configured to move a VIP based on an Apache health
> check. An Apache-only check does **not** guarantee that the VIP
> follows the Zabbix HA-active node; the limitation and a safer design
> are documented below.

## 1. Architecture

  Component                           Hostname                    Address
  ----------------------------------- ------------------- ---------------
  Zabbix Server node 1                `zabbix-01`           `10.10.10.21`
  Zabbix Server node 2                `zabbix-02`           `10.10.10.22`
  External MariaDB                    `zabbix-database`     `10.10.10.23`
  Keepalived virtual IP (VIP)         `zabbix.local`        `10.10.10.20`
  Network interface on Zabbix nodes   `ens34`               `/16` network

Both Zabbix Server nodes connect to the same MariaDB database, `zabbix`.
Zabbix's native HA mechanism elects one active server node and keeps the
other in standby. Keepalived provides a stable frontend address, while
Apache serves the web interface over HTTPS.

``` text
                       Users / Browsers
                              |
                  https://zabbix.local/
                              |
                       VIP 10.10.10.20
                         Keepalived
                              |
                    +---------+---------+
                    |                   |
             zabbix-01              zabbix-02
             10.10.10.21            10.10.10.22
             Zabbix + Apache        Zabbix + Apache
                    \                   /
                     \                 /
                      +-------+-------+
                              |
                     MariaDB 10.10.10.23
                       database: zabbix
```

**Important distinction:** Zabbix native HA manages the Zabbix Server
process, not the Apache frontend. Keepalived manages the VIP, not the
database or Zabbix HA election. Those mechanisms must be coordinated if
the VIP must always land on the node serving the frontend for the active
Zabbix node.

## 2. Prerequisites and variables

-   Ubuntu Server 24.04 on all three machines.
-   Administrative `sudo` access.
-   Network connectivity between the Zabbix nodes and MariaDB.
-   TCP `10051` between Zabbix components/agents as required.
-   TCP `10050` from Zabbix nodes to passive agents, if passive checks
    are used.
-   TCP `3306` from both Zabbix nodes to MariaDB.
-   TCP `80`/`443` from client networks to the frontend VIP.
-   TCP `112` (IP protocol VRRP, not a TCP/UDP port) between Keepalived
    peers, if using VRRP multicast. The example below uses unicast
    peers.

Use your actual network/security policy to limit access; do not expose
database or agent ports to untrusted networks.

## 3. Install the Zabbix repository and packages

Run on **both** Zabbix nodes:

``` bash
sudo apt update
sudo apt install -y wget ca-certificates
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo apt update
sudo apt install -y zabbix-server-mysql zabbix-frontend-php \
  zabbix-apache-conf zabbix-sql-scripts zabbix-agent2
```

Confirm package versions:

``` bash
zabbix_server -V
dpkg -l | grep -E 'zabbix-server-mysql|zabbix-frontend-php|zabbix-apache-conf'
```

> Keep both Zabbix Server nodes on compatible package versions. Upgrade
> both nodes in a planned maintenance window and follow the official
> Zabbix upgrade procedure.

## 4. Prepare the external MariaDB server

The database server in this deployment is `10.10.10.23`. The following
commands are examples for the MariaDB host.

### 4.1 Install and secure MariaDB

``` bash
sudo apt update
sudo apt install -y mariadb-server
sudo systemctl enable --now mariadb
sudo systemctl status mariadb --no-pager
sudo mariadb-secure-installation
```

Ensure MariaDB listens on an address reachable by both Zabbix nodes.
Review:

``` bash
sudo mariadb -e "SHOW VARIABLES LIKE 'bind_address';"
sudo ss -lntp | grep ':3306'
```

If you change the MariaDB bind address, edit the appropriate MariaDB
server configuration under `/etc/mysql/` and restart MariaDB. Avoid
binding to all interfaces unless firewall rules restrict access.

### 4.2 Create the database and user

Log in to MariaDB as an administrator:

``` bash
sudo mariadb
```

Run the SQL below. Replace `REPLACE_WITH_STRONG_PASSWORD` with a strong
secret. Use the same password in both Zabbix Server configuration files.

``` sql
CREATE DATABASE zabbix
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_bin;

CREATE USER 'zabbix'@'10.10.10.21'
  IDENTIFIED BY 'REPLACE_WITH_STRONG_PASSWORD';

CREATE USER 'zabbix'@'10.10.10.22'
  IDENTIFIED BY 'REPLACE_WITH_STRONG_PASSWORD';

GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'10.10.10.21';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'10.10.10.22';

FLUSH PRIVILEGES;
EXIT;
```

Test connectivity from **each** Zabbix node:

``` bash
mariadb -h 10.10.10.23 -u zabbix -p zabbix
```

If the client is not installed on a Zabbix node:

``` bash
sudo apt install -y mariadb-client
```

### 4.3 Import the Zabbix schema exactly once

Run this section **once**, from either Zabbix node, against the shared
MariaDB database. Do not repeat it on the second node. Re-importing into
the same database can produce errors such as
`Table 'role' already exists`.

On the node used for the import:

``` bash
sudo mariadb -h 10.10.10.23 -u zabbix -p zabbix
```

If the import procedure on your MariaDB configuration requires the
Zabbix function-creator workaround, connect as a MariaDB administrator
and run:

``` sql
SET GLOBAL log_bin_trust_function_creators = 1;
```

Then import the schema from the node that has the Zabbix SQL scripts
installed:

``` bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz \
  | mariadb --default-character-set=utf8mb4 \
      -h 10.10.10.23 -u zabbix -p zabbix
```

Enter the database user's password when prompted. Wait for the command
to finish and check its exit status:

``` bash
echo $?
```

A successful command normally returns `0`. Check that the schema exists:

``` bash
mariadb -h 10.10.10.23 -u zabbix -p zabbix \
  -e "SHOW TABLES LIKE 'users'; SELECT COUNT(*) FROM users;"
```

After importing, if you enabled `log_bin_trust_function_creators` only
for the import, revert it according to your MariaDB policy:

``` sql
SET GLOBAL log_bin_trust_function_creators = 0;
```

Do not run the schema import on node 2: both Zabbix nodes use the same
external database.

## 5. Configure Zabbix Server native HA

Edit `/etc/zabbix/zabbix_server.conf` on each node. Keep the database
settings identical, but give each node a unique `HANodeName` and
`NodeAddress`.

### 5.1 Node 1: `zabbix-01`

Back up the file first:

``` bash
sudo cp -a /etc/zabbix/zabbix_server.conf \
  /etc/zabbix/zabbix_server.conf.bak
sudo nano /etc/zabbix/zabbix_server.conf
```

Ensure the following settings are present and uncommented. Avoid
duplicate active definitions elsewhere in the file.

``` ini
DBHost=10.10.10.23
DBName=zabbix
DBUser=zabbix
DBPassword=REPLACE_WITH_STRONG_PASSWORD

HANodeName=zabbix-01
NodeAddress=10.10.10.21:10051

StatsAllowedIP=127.0.0.1
```

### 5.2 Node 2: `zabbix-02`

Use the same database credentials, but a different HA node name and
address:

``` ini
DBHost=10.10.10.23
DBName=zabbix
DBUser=zabbix
DBPassword=REPLACE_WITH_STRONG_PASSWORD

HANodeName=zabbix-02
NodeAddress=10.10.10.22:10051

StatsAllowedIP=127.0.0.1
```

Protect the configuration files because they contain database
credentials:

``` bash
sudo chown root:zabbix /etc/zabbix/zabbix_server.conf
sudo chmod 640 /etc/zabbix/zabbix_server.conf
```

### 5.3 Start and verify the HA nodes

Start the first node, then the second:

``` bash
sudo systemctl enable --now zabbix-server
sudo systemctl status zabbix-server --no-pager
```

Check logs on each node:

``` bash
sudo tail -n 100 /var/log/zabbix/zabbix_server.log
```

Check HA status on a running Zabbix Server node:

``` bash
sudo zabbix_server -R ha_status
```

The output should identify one active node and one standby node. The
active node normally accepts server connections on TCP `10051`; a
standby node may not listen on that port. Do not treat this behavior
alone as a failure.

Check listening sockets:

``` bash
sudo ss -lntp | grep ':10051'
```

If a service fails to start:

``` bash
sudo journalctl -u zabbix-server -n 100 --no-pager
sudo grep -E '^(DBHost|DBName|DBUser|DBPassword|HANodeName|NodeAddress|StatsAllowedIP)=' \
  /etc/zabbix/zabbix_server.conf
```

Do not publish command output containing passwords.

## 6. Configure the Zabbix frontend and Apache

The frontend was exposed at the site root over HTTPS, so users visit
`https://zabbix.local/` rather than `https://zabbix.local/zabbix/`.

### 6.1 Enable Apache modules and service

Run on both Zabbix nodes:

``` bash
sudo a2enmod ssl rewrite headers
sudo systemctl enable --now apache2
sudo systemctl status apache2 --no-pager
```

### 6.2 Configure the frontend database connection

The frontend configuration is:

``` text
/etc/zabbix/web/zabbix.conf.php
```

Back it up before editing:

``` bash
sudo cp -a /etc/zabbix/web/zabbix.conf.php \
  /etc/zabbix/web/zabbix.conf.php.bak
sudo nano /etc/zabbix/web/zabbix.conf.php
```

Ensure the database settings point to the external MariaDB server.
Preserve the existing PHP structure and variable names; do not replace
the entire file with this fragment.

``` php
$DB['TYPE']     = 'MYSQL';
$DB['SERVER']   = '10.10.10.23';
$DB['PORT']     = '3306';
$DB['DATABASE'] = 'zabbix';
$DB['USER']     = 'zabbix';
$DB['PASSWORD'] = 'REPLACE_WITH_STRONG_PASSWORD';
```

If the frontend uses TLS client certificate settings, configure paths
consistently on both nodes and verify that the Apache/PHP user can read
the required certificate and key files. Do not enable database TLS in
the frontend unless MariaDB TLS is configured correctly on the server
side as well.

Check PHP syntax:

``` bash
php -l /etc/zabbix/web/zabbix.conf.php
```

If the frontend uses certificate files under `/etc/zabbix/ssl/`, verify
permissions without making the private key world-readable:

``` bash
sudo ls -l /etc/zabbix/ssl/
sudo -u www-data test -r /etc/zabbix/ssl/ca.crt && echo "CA readable"
sudo -u www-data test -r /etc/zabbix/ssl/frontend.crt && echo "Frontend certificate readable"
sudo -u www-data test -r /etc/zabbix/ssl/frontend.key && echo "Frontend key readable"
```

Use the actual paths configured in your PHP file. Do not assume the
example filenames exist on a fresh installation.

### 6.3 Serve the site at `/` and disable `/zabbix/`

Ubuntu's Zabbix Apache configuration is usually available through
`/etc/apache2/conf-enabled/zabbix.conf`. Back up the active
configuration:

``` bash
sudo cp -a /etc/apache2/conf-enabled/zabbix.conf \
  /etc/apache2/conf-enabled/zabbix.conf.bak
sudo nano /etc/apache2/conf-enabled/zabbix.conf
```

In the configuration, the default mapping may include:

``` apache
Alias /zabbix /usr/share/zabbix
```

If the desired design is to serve the frontend from the virtual host's
document root instead, disable the `/zabbix` Alias and use an explicit
virtual host configuration. One possible pattern is shown below; adapt
the certificate paths and document root to the actual package layout.

Create `/etc/apache2/sites-available/zabbix-https.conf`:

``` apache
<VirtualHost *:80>
    ServerName zabbix.local
    Redirect permanent / https://zabbix.local/
</VirtualHost>

<VirtualHost *:443>
    ServerName zabbix.local
    DocumentRoot /usr/share/zabbix

    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/zabbix.local.crt
    SSLCertificateKeyFile /etc/apache2/ssl/zabbix.local.key

    <Directory /usr/share/zabbix>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/zabbix_error.log
    CustomLog ${APACHE_LOG_DIR}/zabbix_access.log combined
</VirtualHost>
```

Enable the site and validate the configuration:

``` bash
sudo a2ensite zabbix-https.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

If `/zabbix/` must return `404`, add this directive to the HTTPS virtual
host:

``` apache
RedirectMatch 404 ^/zabbix/?$
```

Then validate and reload Apache again:

``` bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

**Do not leave conflicting active Alias or VirtualHost directives
enabled.** The exact package configuration can differ; inspect
`apache2ctl -S` and the enabled files before applying this example.

## 7. Create HTTPS certificates

The lab used a private/self-signed CA and frontend certificates.
Browsers will show a trust warning unless the CA is installed and
trusted by client machines. A production deployment should use
certificates issued by the organization's trusted CA or another trusted
certificate authority.

Inspect the certificate subject, issuer, validity, and SANs:

``` bash
openssl x509 -in /etc/apache2/ssl/zabbix.local.crt \
  -noout -subject -issuer -dates -ext subjectAltName
```

A certificate for `zabbix.local` should contain `DNS:zabbix.local` in
its Subject Alternative Name (SAN). A Common Name alone is not
sufficient for modern browser hostname validation.

If creating a self-signed certificate for a lab, make sure it has a SAN.
Example OpenSSL configuration (`/etc/apache2/ssl/zabbix-local.cnf`):

``` ini
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
x509_extensions = v3_req

[dn]
CN = zabbix.local

[v3_req]
subjectAltName = @alt_names
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[alt_names]
DNS.1 = zabbix.local
```

Create the directory and certificate:

``` bash
sudo install -d -m 750 /etc/apache2/ssl
cd /etc/apache2/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout zabbix.local.key \
  -out zabbix.local.crt \
  -config zabbix-local.cnf
sudo chown root:root zabbix.local.key zabbix.local.crt
sudo chmod 600 zabbix.local.key
sudo chmod 644 zabbix.local.crt
```

This self-signed example is for a lab only. If using a CA-signed
certificate, follow the organization's certificate issuance and renewal
process. Copy the correct certificate and private key to both frontend
nodes securely, or use a certificate deployment/renewal process.

Test HTTPS locally against a specific node:

``` bash
curl -k -I --resolve zabbix.local:443:10.10.10.21 https://zabbix.local/
curl -k -I --resolve zabbix.local:443:10.10.10.22 https://zabbix.local/
```

`-k` skips certificate verification and is for troubleshooting only. For
normal use, trust the CA on the client and test without `-k`.

## 8. Configure Keepalived and the VIP

The VIP used in the lab is `10.10.10.20` on interface `ens34`. The two
node addresses are `.21` and `.22`.

Install Keepalived on both nodes:

``` bash
sudo apt update
sudo apt install -y keepalived
sudo systemctl enable keepalived
```

Confirm the interface name before using the configuration:

``` bash
ip -br address
ip route
```

### 8.1 Keepalived configuration

The lab used VRID `51`, node 1 priority `150`, node 2 priority `100`,
and unicast peer addresses. Replace `REPLACE_WITH_VRRP_PASSWORD` with a
lab-appropriate value; do not commit real secrets to GitHub.

Example for node 1, `/etc/keepalived/keepalived.conf`:

``` conf
global_defs {
    router_id ZABBIX_01
    script_security
}

vrrp_script chk_apache {
    script "/usr/bin/pgrep -x apache2"
    interval 2
    timeout 2
    fall 2
    rise 2
    weight -20
}

vrrp_instance VI_ZABBIX {
    state MASTER
    interface ens34
    virtual_router_id 51
    priority 150
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass REPLACE_WITH_VRRP_PASSWORD
    }

    unicast_src_ip 10.10.10.21
    unicast_peer {
        10.10.10.22
    }

    virtual_ipaddress {
        10.10.10.20/16 dev ens34
    }

    track_script {
        chk_apache
    }
}
```

Example for node 2, `/etc/keepalived/keepalived.conf`:

``` conf
global_defs {
    router_id ZABBIX_02
    script_security
}

vrrp_script chk_apache {
    script "/usr/bin/pgrep -x apache2"
    interval 2
    timeout 2
    fall 2
    rise 2
    weight -20
}

vrrp_instance VI_ZABBIX {
    state BACKUP
    interface ens34
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass REPLACE_WITH_VRRP_PASSWORD
    }

    unicast_src_ip 10.10.10.22
    unicast_peer {
        10.10.10.21
    }

    virtual_ipaddress {
        10.10.10.20/16 dev ens34
    }

    track_script {
        chk_apache
    }
}
```

> **Keepalived security note:** VRRP authentication settings should not
> be treated as a strong security boundary. Protect the network and
> restrict VRRP traffic between the intended peers.

Validate and start Keepalived on each node:

``` bash
sudo keepalived --config-test -f /etc/keepalived/keepalived.conf
sudo systemctl restart keepalived
sudo systemctl status keepalived --no-pager
```

Inspect VIP ownership and logs:

``` bash
ip address show dev ens34
sudo journalctl -u keepalived -n 100 --no-pager
```

Normally only one node should own `10.10.10.20` at a time.

### 8.2 Important limitation: Apache health is not Zabbix HA state

The Apache health check above only verifies that Apache is running. It
does not know which Zabbix Server node is the active HA node. During
testing, it was possible for the VIP to be on one node while Zabbix
Server's active role was on the other. In that situation, the web page
may work but an agent configured to use the VIP for active checks can
get connection errors because the VIP's node is in Zabbix standby and is
not listening on TCP `10051`.

For a robust production design, decide explicitly whether the VIP is
intended for:

1.  **The web frontend only**, or
2.  **Both the frontend and Zabbix Server/agent traffic**.

If it is for the web frontend only, an Apache health check may be
adequate, and active agents should use an endpoint that correctly
reaches the active Zabbix Server node.

If the same VIP must also represent the Zabbix HA-active node, a health
check that only tests Apache is insufficient. Integrate the Keepalived
decision with a reliable Zabbix HA-role check, or use separate
VIPs/endpoints for the frontend and Zabbix Server traffic. A custom
role-check script must be carefully tested for failover, timeouts, and
the case where the local node cannot query its role. Do not assume a
process check alone identifies the active Zabbix node.

## 9. Configure client name resolution

For a lab without internal DNS, add this entry to the client machine's
hosts file:

``` text
10.10.10.20 zabbix.local
```

On Linux, edit `/etc/hosts`. On Windows, edit the hosts file as
Administrator:

``` text
C:\Windows\System32\drivers\etc\hosts
```

For production, create a DNS A record for `zabbix.local` pointing to the
VIP (or use the organization's actual DNS suffix). All clients must
resolve the name to the VIP, not to node 1's physical IP.

Test resolution:

``` bash
getent hosts zabbix.local
```

## 10. Configure agents

For passive checks, the Agent's `Server` parameter must allow the actual
source IP address of the Zabbix Server connection. In the lab, the
Fedora 25 host at `10.10.10.149` rejected connections from `10.10.10.21`
because only the VIP `10.10.10.20` was allowed.

For a classic Zabbix Agent, an example configuration is:

``` ini
Server=10.10.10.20,10.10.10.21,10.10.10.22
ServerActive=10.10.10.20:10051
Hostname=pe3.tbl.local
```

The `Server` list controls which sources may connect for passive checks.
The `ServerActive` value controls where the agent requests active
checks; it must point to an endpoint that reaches the currently active
Zabbix Server. If the VIP can land on a Zabbix standby node, do not
assume that using the VIP for active checks will work until Keepalived
and Zabbix HA roles are coordinated.

Restart and inspect the agent:

``` bash
sudo systemctl restart zabbix-agent
sudo systemctl status zabbix-agent --no-pager
sudo ss -lntp | grep ':10050'
sudo tail -n 50 /var/log/zabbix/zabbix_agentd.log
```

The lab's passive connectivity test succeeded with:

``` bash
zabbix_get -s 10.10.10.149 -p 10050 -k agent.ping
```

Expected output:

``` text
1
```

On Ubuntu, if `zabbix_get` is not installed, install the matching Zabbix
package from the configured Zabbix repository:

``` bash
sudo apt install zabbix-get
```

## 11. Validation and failover tests

Perform these tests in a maintenance window. Keep a console or
out-of-band path to both nodes.

### 11.1 Validate database connectivity

From each Zabbix node:

``` bash
mariadb -h 10.10.10.23 -u zabbix -p zabbix \
  -e "SELECT 1;"
```

### 11.2 Validate Zabbix HA state

On the Zabbix nodes:

``` bash
sudo zabbix_server -R ha_status
sudo systemctl status zabbix-server --no-pager
sudo tail -n 50 /var/log/zabbix/zabbix_server.log
```

Verify that one node is active and the other is standby.

### 11.3 Validate frontend HTTPS

``` bash
curl -k -I --resolve zabbix.local:443:10.10.10.21 https://zabbix.local/
curl -k -I --resolve zabbix.local:443:10.10.10.22 https://zabbix.local/
curl -k -I https://zabbix.local/
```

The direct-node checks verify each Apache frontend independently; the
final check verifies the VIP path from a client with correct name
resolution.

### 11.4 Validate VIP ownership

On both nodes:

``` bash
ip address show dev ens34
sudo journalctl -u keepalived --since "-10 minutes" --no-pager
```

The VIP should be present on only one node.

### 11.5 Failover test

1.  Record the active Zabbix Server node using
    `zabbix_server -R ha_status`.
2.  Record the node currently holding `10.10.10.20`.
3.  Stop or isolate the active Zabbix Server node in a controlled
    maintenance window.
4.  Wait for Zabbix HA failover and verify the new active node.
5.  Verify VIP ownership independently; do not assume it follows the
    Zabbix HA role.
6.  Test `https://zabbix.local/` and check agent data/active-check logs.
7.  Restore the node and verify that HA returns to the expected state
    without two active nodes.

### 11.6 Useful diagnostics

``` bash
# Zabbix Server
sudo systemctl status zabbix-server --no-pager
sudo journalctl -u zabbix-server -n 100 --no-pager
sudo tail -n 100 /var/log/zabbix/zabbix_server.log
sudo zabbix_server -R ha_status

# Apache
sudo apache2ctl configtest
sudo apache2ctl -S
sudo systemctl status apache2 --no-pager
sudo tail -n 100 /var/log/apache2/zabbix_error.log

# Keepalived / VIP
sudo keepalived --config-test -f /etc/keepalived/keepalived.conf
sudo systemctl status keepalived --no-pager
sudo journalctl -u keepalived -n 100 --no-pager
ip address show dev ens34

# Network sockets
sudo ss -lntp | grep -E ':(80|443|10050|10051|3306)\b'

# Agent
sudo systemctl status zabbix-agent --no-pager
sudo tail -n 100 /var/log/zabbix/zabbix_agentd.log
```

## 12. Troubleshooting notes from the deployment

### `Table 'role' already exists` during SQL import

The schema was being imported into a database that already contained
Zabbix tables. Import the schema only once into the shared database. Do
not import it separately from node 2.

### Zabbix frontend cannot connect to the database

Check the frontend DB parameters in `/etc/zabbix/web/zabbix.conf.php`,
MariaDB grants, firewall, and database TLS settings. In this lab, the
frontend's database TLS option was found to be the cause of one
connection problem; enabling TLS in the frontend requires matching
server-side TLS configuration.

### Passive Agent error: `connection ... rejected, allowed hosts`

The source address of the connecting Zabbix Server must be included in
the Agent's `Server=` list. The lab's Agent rejected `10.10.10.21` while
only `10.10.10.20` was allowed. The fix was to allow the two physical
node IPs as well as the VIP.

### Active checks fail while passive checks work

Passive checks use `Server=` and incoming TCP `10050` connections.
Active checks use `ServerActive=` and an outgoing connection to TCP
`10051`. If the VIP is on a Zabbix standby node, active checks through
that VIP may fail. Verify the current HA role and VIP owner
independently.

### Items show timestamps outside the history storage period

Check the time on the agent and server, preferably with:

``` bash
date -u
timedatectl status
```

Correct system time and synchronization before troubleshooting item data
further. Old log entries may refer to earlier timestamps, so compare
fresh timestamps rather than relying only on old `grep` output.

### Unsupported template item keys on an old agent

A newer template may contain item keys not supported by an older agent.
For example, the lab's Zabbix Agent 6.0.19 reported
`system.sw.packages.get` as unsupported. Align the template with the
agent version or upgrade the agent where the operating system supports
it. This individual unsupported item does not necessarily mean the
entire agent connection is broken.

## 13. Security and operational checklist

-   [ ] Replace all example passwords and VRRP values.
-   [ ] Do not commit database passwords, private keys, or real internal
    secrets to Git.
-   [ ] Restrict TCP `3306` to the Zabbix nodes.
-   [ ] Restrict TCP `10050` to the Zabbix Server source addresses.
-   [ ] Allow TCP `10051` only where required.
-   [ ] Trust the HTTPS certificate's issuing CA on client machines.
-   [ ] Ensure the certificate SAN includes the DNS name users open.
-   [ ] Configure reliable time synchronization on all nodes.
-   [ ] Back up MariaDB and test restoration.
-   [ ] Test Zabbix HA and VIP failover separately.
-   [ ] Ensure the VIP design matches its purpose (web frontend, server
    traffic, or separate endpoints).
-   [ ] Document the recovery procedure and test it regularly.

## 14. Reference documentation

-   [Zabbix 7.0
    documentation](https://www.zabbix.com/documentation/7.0/en/manual)
-   [Zabbix HA
    documentation](https://www.zabbix.com/documentation/7.0/en/manual/concepts/server/ha)
-   [Zabbix installation from
    packages](https://www.zabbix.com/documentation/7.0/en/manual/installation/install_from_packages)
-   [Keepalived documentation](https://www.keepalived.org/manpage.html)
-   [Ubuntu Server
    documentation](https://documentation.ubuntu.com/server/)
