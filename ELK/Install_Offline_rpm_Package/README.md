 
# Install offline logstash (there are 3 nodes that access to each other on port 22)
## Only node-01 access to  myregistry.com


Step 1: get package in local host 
```
yum install --downloadonly --downloaddir=/home logstash-7.17.3 
```


Step 2: go to myregistry.com//ui/admin/repositories/local 
```
Add repository -> local repository

Filter by package name type -> rmp

go to artifactory ->artifacts -> rpm-local -> 

-> deploy package (select file)

-> OR set me up (from local host -> push package to myregistry with command  [after login to myregistry])
```

Step 3: scp package to node-02 (note access to myregistry)

```
scp  ./java-11-openjdk-headless-11.0.18.0.10-2.el8_7.x86_64.rpm    root@192.168.100.5:/home
 ```

Step 4: install package 

```
cd /home
rpm  -Uvh   ./java-11-openjdk-headless-11.0.18.0.10-2.el8_7.x86_64.rpm
 ```
 Note: -U is used to upgrade an RPM package but will also install a package if it does not exist in the RPM database.


 Note: -i  is used to install a new package. Always use this for kernel installations and upgrades just in case.
```
 netstat -nltp
 systemctl status logstash
 ```

