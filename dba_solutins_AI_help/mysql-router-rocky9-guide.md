# MySQL Router 9.7 on Rocky Linux 9.6 — Install & Bootstrap Guide

Deploying MySQL Router on application VMs in front of a 3-node MySQL 9.7 InnoDB Cluster.

Run Steps 1–3 and 5–8 on **each app VM**. Step 4 runs **once**, from the cluster.

Examples assume cluster nodes `db1`, `db2`, `db3` and cluster admin account `clusteradmin`. Substitute your own names.

---

## 1. Check the network first (most bootstrap failures start here)

From each app VM:

```bash
# Each cluster node must resolve by the name it reports to the cluster (report_host / hostname)
getent hosts db1 db2 db3

# Classic protocol (and X protocol, if used) must be reachable
nc -zv db1 3306; nc -zv db2 3306; nc -zv db3 3306
nc -zv db1 33060
```

Router learns member addresses from the cluster metadata. If the hostnames the cluster uses don't resolve on the app VM, the bootstrap succeeds but connections fail later. If DNS isn't an option, add the names to `/etc/hosts`.

On the DB nodes, make sure firewalld allows the app subnet on 3306 (and 33060).

## 2. Add the MySQL repository

Use the same repository you used on the DB nodes. The easiest way is to copy it over:

```bash
# Option A: copy from a DB node
scp db1:/etc/yum.repos.d/mysql-community*.repo /etc/yum.repos.d/
scp db1:/etc/pki/rpm-gpg/RPM-GPG-KEY-mysql* /etc/pki/rpm-gpg/

# Option B: install the EL9 release RPM from https://dev.mysql.com/downloads/repo/yum/
sudo dnf install -y https://dev.mysql.com/get/<el9-release-rpm-name>.noarch.rpm
```

Confirm the 9.7 LTS repos are the enabled ones. The server subrepo is `mysql-9.7-lts-community`; Router and Shell come from the matching *tools* repo:

```bash
dnf repolist enabled | grep -i mysql
```

If 8.4 is enabled instead, list all repo IDs first and then switch:

```bash
dnf repolist all | grep mysql
sudo dnf config-manager \
  --disable mysql-8.4-lts-community mysql-tools-8.4-lts-community \
  --enable  mysql-9.7-lts-community mysql-tools-9.7-lts-community
```

## 3. Install Router (and Shell for testing)

```bash
sudo dnf install -y mysql-router-community mysql-shell mysql-community-client
mysqlrouter --version     # should report 9.7.x
```

The RPM creates a local system user and group named `mysqlrouter`, and Router runs as that user by default.

Keep the Router version the same as or newer than the server version. 9.7 with 9.7 is ideal.

## 4. Create a dedicated Router account (once, on the cluster)

From any machine with mysqlsh, connect to the primary:

```bash
mysqlsh clusteradmin@db1:3306
```

```js
var c = dba.getCluster();
c.setupRouterAccount('routeradmin@%');   // or restrict: 'routeradmin@10.10.20.%'
c.status();                              // confirm all 3 members ONLINE
```

All routers can share this one account. Router uses it at runtime to read the cluster metadata.

## 5. Bootstrap Router on each app VM

```bash
sudo mysqlrouter --bootstrap clusteradmin@db1:3306 \
  --user=mysqlrouter \
  --account=routeradmin \
  --name="$(hostname -s)" \
  --conf-bind-address=127.0.0.1
```

- `clusteradmin` is used only to register this router in the cluster metadata. `routeradmin` is what Router uses afterward, and its password goes into Router's keyring.
- `--conf-bind-address=127.0.0.1` is the recommended setting when Router runs on the same VM as the app, because only local apps can use it. Leave it out (or use `0.0.0.0`) if other hosts need to connect.
- Without `--directory`, this is a system-wide bootstrap. It writes `/etc/mysqlrouter/mysqlrouter.conf`, and the state and keyring go under `/var/lib/mysqlrouter/`.

Default ports printed at the end of the bootstrap:

| Port        | Purpose                                         |
|-------------|-------------------------------------------------|
| 6446        | Classic protocol, read/write (goes to primary)  |
| 6447        | Classic protocol, read-only (goes to secondaries) |
| 6448 / 6449 | X protocol, RW / RO                             |
| 6450        | Read/write splitting (classic)                  |
| 8443        | Router REST API                                 |

## 6. SELinux and firewall

Rocky runs SELinux in enforcing mode. Start the service, and if it fails to bind, check for denials:

```bash
sudo ausearch -m avc -ts recent | grep -i router

# if you see port denials:
sudo dnf install -y policycoreutils-python-utils
sudo semanage port -a -t mysqld_port_t -p tcp 6446-6450
```

Open the firewall only if remote hosts connect to this Router:

```bash
sudo firewall-cmd --permanent --add-port=6446-6450/tcp
sudo firewall-cmd --reload
```

## 7. Start and enable the service

```bash
sudo systemctl enable --now mysqlrouter
sudo systemctl status mysqlrouter
sudo tail -f /var/log/mysqlrouter/mysqlrouter.log
```

## 8. Verify

```bash
mysql -h127.0.0.1 -P6446 -u appuser -p -e "SELECT @@hostname, @@read_only;"   # primary, read_only=0
mysql -h127.0.0.1 -P6447 -u appuser -p -e "SELECT @@hostname, @@read_only;"   # a secondary, read_only=1
```

Use TCP (`-h127.0.0.1`), not `localhost`, which would try a socket.

From mysqlsh on the cluster:

```js
dba.getCluster().listRouters()
```

All your app VMs should appear in the list.

Point your app at:

- `127.0.0.1:6446` for writes
- `127.0.0.1:6447` for reads
- `127.0.0.1:6450` for automatic read/write splitting

Your application user needs to exist on the cluster as usual. Router just passes its credentials through.

## 9. Test failover (recommended)

Keep a loop running against port 6446:

```bash
while true; do mysql -h127.0.0.1 -P6446 -u appuser -p'***' -N -e "SELECT @@hostname"; sleep 1; done
```

While it runs, switch the primary from mysqlsh:

```js
dba.getCluster().setPrimaryInstance('db2:3306')
```

Router should notice the change within seconds, and new connections should land on `db2`. Connections that were open during the switch get dropped, so your app needs to reconnect and retry.

---

## Common gotchas

- **Re-bootstrapping:** add `--force` to overwrite an existing config. Use the same `--name` so you don't leave stale entries in the metadata. Remove old ones with `c.removeRouterMetadata('host::name')`.
- **"Can't connect to remote MySQL server" after a successful bootstrap:** this is almost always the hostname-resolution problem from Step 1.
- **TLS:** Router 9.x uses TLS on both the client and server sides by default, and creates self-signed certificates at bootstrap. If an older client library fails the TLS handshake, update the driver rather than turning TLS off.
- **Automation:** for scripted deployments, pass `--account-create=never` and feed the passwords in on stdin, so they don't end up in shell history.

## References

- [MySQL Router 9.7: Installing on Linux](https://dev.mysql.com/doc/mysql-router/9.0/en/mysql-router-installation-linux.html)
- [MySQL 9.7: Installing using the MySQL Yum Repository](https://dev.mysql.com/doc/refman/9.7/en/linux-installation-yum-repo.html)
- [MySQL Yum Repository downloads](https://dev.mysql.com/downloads/repo/yum/)
