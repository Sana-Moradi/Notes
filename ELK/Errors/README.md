# Elasticsearch - Authentication error when check to see if index exist (shard error)
## In case of elasticsearch crashes because of disk usege (out of disk problem), you need to add more disk to it, then remove the old index file then restart the servie to reindex logs.


Remove index files in elasticsearch:
```
cd /var/log/elasticsearch/
rm -rf *.log*
rm -rf es-cluster-2022-*
echo ""> es-cluster_server.json
```

## OR
Remove index files in elasticsearch: 
```
rm -rf /var/lib/elasticsearch/*
```

> **Note:** Before all this, check the following items

```
 systemctl status firewall

 df -h

 lsblk
 ```
```
 netstat -nltp

 tcpdump -XXn -i any port 5601
 ```
 ```
 tail -f /var/log/elasticsearch/*

 tail -f /var/log/kibana/*

 systemctl status kibana

 systemctl status elasticsearch

 systemctl restart elasticsearch
 
 ```