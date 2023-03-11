 
# Backup and Restore elasticsearch cluster using snapshot (in Kibana)
## there are 3 Nodes of elasticSearch
```
 192.168.195.136 
 192.168.195.137
 192.168.195.138
```

## For each node:
## Create directory for the storing location of the snapshot 
```
cd /

mkdir /BackupES

chown -R elasticsearch:elasticsearch /BackupES

```
```Note: the directory must be shared between servers```
# Installing NFS
```
 NFS Server: 192.168.195.136

 NFS client:192.168.195.137

 NFS client:192.168.195.138
 ```
 # Configure the firewall (For each node)
 ```
 yum -y install firewalld

 Then

 systemctl start firewalld.service
 systemctl enable firewalld.service
 firewall-cmd--permanent--zone=public--add-service=ssh
 firewall-cmd--permanent--zone=public--add-service=nfs
 firewall-cmd--reload
 ```

# Installing NFS (For each node) 
```
yum -y install NFS-utils

systemctl enable NFS-Server.service
systemctl start NFS-Server.service
```
# Change permision (For each node)

```
chown -R nfsnobody:nfsnobody /BackupES
chmode 755 /BackupES
```
**192.168.195.136:**
```
vi /etc/exports

/BackupES   192.168.195.137(rw,sync,no_root_squash,no_subtree_check)
/BackupES   192.168.195.138(rw,sync,no_root_squash,no_subtree_check)

Then

exportfs -arv

```
**192.168.195.137:**
```
vi /etc/exports

/BackupES   192.168.195.136(rw,sync,no_root_squash,no_subtree_check)


Then

exportfs -arv

```
**192.168.195.138:**
```
vi /etc/exports

/BackupES   192.168.195.136(rw,sync,no_root_squash,no_subtree_check)


Then

exportfs -arv

```

**192.168.195.137:**
```

mount 192.168.195.136:/BackupES /BackupES

```

**192.168.195.138:**
```

mount 192.168.195.136:/BackupES /BackupES

```

# Testing (On the client)
```
df -h 

     192.168.195.136:/BackupES    6.2G   1.1G   5.2G   18%   /BackupES
```

# Related
```
*  http://cafedb.ir/text-57/text-58 
```

