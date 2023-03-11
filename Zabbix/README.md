# Install Zabbix 6.2 on Oracle Linux

# 1. Add repository

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/6.2/rhel/9/x86_64/zabbix-release-6.2-2.el9.noarch.rpm

dnf clean all
```

# 2. Install Zabbix server, frontend, agent

```bash
dnf install zabbix-server-mysql zabbix-web-mysql 
zabbix-apache-conf
zabbix-sql-scripts zabbix-selinux-policy zabbix-agent
systemctl start php-fpm 
systemctl enable php-fpm
systemctl start httpd
systemctl enable httpd
```

# 3. Install and config mysql 10.6

```bash
curl -LsS -O https://downloads.mariadb.com/MariaDB/mariadb_repo_setup
sudo bash mariadb_repo_setup --mariadb-server-version=10.6
dnf -y install mariadb-server && systemctl start mariadb && systemctl enable mariadb
systemctl status mariadb
```

# 4. Create initial database

```bash
sudo mysql -uroot -p'Pass@123456' -e "create database zabbix character set utf8mb4 collate utf8mb4_bin;"

sudo mysql -uroot -p'Pass@123456' -e "grant all privileges on zabbix.* to zabbix@localhost identified by 'Pass@123456';"

sudo zcat /usr/share/doc/zabbix-sql-scripts/mysql/server.sql.gz | mysql -uzabbix -p'Pass@123456' zabbix
```

```bash
mysql -uroot -p
password
mysql> set global log_bin_trust_function_creators = 0;
mysql> quit;
```

# 5. Configure the database for Zabbix server

```bash
vi /etc/zabbix/zabbix_server.conf

DBPassword=Pass@123456
```

# 6. Start Zabbix server and agent processes

```bash
systemctl restart zabbix-server zabbix-agent httpd php-fpm
systemctl enable zabbix-server zabbix-agent httpd php-fpm
```

# 7. Firewall Permanent

```bash
firewall-cmd --add-service={http,https} --permanent
firewall-cmd --add-port={10051/tcp,10050/tcp} --permanent
firewall-cmd --reload
```

# 8. Disable SElinux

# 9. reboot your server


# Related

```
* https://www.zabbix.com/download?zabbix=6.2&os_distribution=oracle_linux&os_version=9&components=server_frontend_agent&db=mysql&ws=apache

* https://techviewleo.com/install-and-configure-zabbix-on-oracle-linux/
```
