[TOC]

查看hdfs的信息

```bash
curl -u admin:admin -H "X-Requested-By:ambari" -X GET http://10.99.99.12:18080/api/v1/clusters/OSS3/services/HDFS
```

ecs-hdp-1上的namenode正常，ecs-hdp-12上的namenode部署失败，ecs-hdp-12原有secondary_namenode进程

命令中的 10.99.99.12 为 Ambari Server 的IP（端口默认为 8080），OSS3 为 cluster 名字，HDFS 为 Service 的名字。

停止hdfs

```bash
curl -u admin:admin -H "X-Requested-By: ambari" -X PUT -d '{"RequestInfo": {"context":"Stop Service"},"Body":{"ServiceInfo":{"state":"INSTALLED"}}}' http://10.99.99.12:18080/api/v1/clusters/OSS3/services/HDFS
```

 查看各主机的组件角色
 
```bash
curl -u admin:admin -i http://10.99.99.12:18080/api/v1/clusters/OSS3/host_components?HostRoles/component_name=NAMENODE
 
curl -u admin:admin -i http://10.99.99.12:18080/api/v1/clusters/OSS3/host_components?HostRoles/component_name=SECONDARY_NAMENODE
 
 
curl -u admin:admin -i http://10.99.99.12:18080/api/v1/clusters/OSS3/host_components?HostRoles/component_name=JOURNALNODE
 
curl -u admin:admin -i  http://10.99.99.12:18080/api/v1/clusters/OSS3/host_components?HostRoles/component_name=ZKFC
```

删除zkfc

```bash
curl -u admin:admin -H  "X-Requested-By: ambari"  -X DELETE  http://10.99.99.12:18080/api/v1/clusters/OSS3/hosts/ecs-hdp-1/host_components/ZKFC
curl -u admin:admin -H  "X-Requested-By: ambari"  -X DELETE  http://10.99.99.12:18080/api/v1/clusters/OSS3/hosts/ecs-hdp-12/host_components/ZKFC
```

删除journalnode

```bash
curl -u admin:admin -H "X-Requested-By: ambari" -X GET http://10.99.99.12:18080/api/v1/clusters/OSS3/host_components?HostRoles/component_name=JOURNALNODE

curl -u admin:admin -H "X-Requested-By: ambari" -X DELETE http://10.99.99.12:18080/api/v1/clusters/OSS3/hosts/ecs-hdp-1/host_components/JOURNALNODE
curl -u admin:admin -H "X-Requested-By: ambari" -X DELETE http://10.99.99.12:18080/api/v1/clusters/OSS3/hosts/ecs-hdp-2/host_components/JOURNALNODE
curl -u admin:admin -H "X-Requested-By: ambari" -X DELETE http://10.99.99.12:18080/api/v1/clusters/OSS3/hosts/ecs-hdp-3/host_components/JOURNALNODE
```

删除额外的namenode：

```bash
curl -u admin:admin -H  "X-Requested-By: ambari"  -X DELETE http://10.99.99.12:18080/api/v1/clusters/OSS3/hosts/ecs-hdp-12/host_components/NAMENODE`
```


启用SECONDARY_NAMENODE

```bash
#step1
curl -u admin:admin -H  "X-Requested-By: ambari"  -X POST -d  '{"host_components" : [{"HostRoles":{"component_name":"SECONDARY_NAMENODE"}}] }'  http://10.99.99.12:18080/api/v1/clusters/OSS3/hosts?Hosts/host_name=ecs-hdp-12
 
#step2 启用SECONDARY_NAMENODE
curl -u admin:admin -H  "X-Requested-By: ambari"  -X PUT -d  '{"RequestInfo":{"context":"Enable Secondary NameNode"},"Body":{"HostRoles":{"state":"INSTALLED"}}}'  http://10.99.99.12:18080/api/v1/clusters/OSS3/hosts/ecs-hdp-12/host_components/SECONDARY_NAMENODE

#step3
curl -u admin:admin -H  "X-Requested-By: ambari"  -X GET  "http://10.99.99.12:18080/api/v1/clusters/OSS3/host_components?HostRoles/component_name=SECONDARY_NAMENODE&fields=HostRoles/state"
```
