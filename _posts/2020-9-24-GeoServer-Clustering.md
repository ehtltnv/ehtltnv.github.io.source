---
layout: post
title:  GeoServer Clustering
data:   2020-09-24
author: ehtltnv
categories: [GeoServer]
tags: [geoserver, docker]
---

__GeoServer__ Clustering 구축 

OS는 `Centos 7`를 사용 

`GeoServer`는 Multi Primary로 3개로 구성하고, `ActiveMQ`는 2개로 구성 

# 1. ActiveMQ 설정 

## 1.1. 기본 구성 

- ActiveMQ를 위한 Docker Container 생성 

```
<!-- Container 간 통신을 위한 network 생성 -->
$ docker network create gs_cluster 

<!-- GEOSERVER_DATA_DIR이 왜 필요한지는 확인 필요(TODO) -->
$ docker run -d --privileged --name mq1 --network gs_cluster \
-p 61666:61666 \
-v /opt/gs_cluster/data_dir:/opt/data/data_dir \
-e GEOSERVER_DATA_DIR=/opt/data/data_dir \
centos:7 /sbin/init 
```

- 필요 라이브러리 및 OpenJDK 설치
```
$ docker exec -it mq1 bash

$ yum install -y epel-release
$ yum install -y java-11-openjdk java-11-openjdk-devel unzip net-tools
``` 

## 1.2. GeoServer community plugin, Tomcat 다운로드 

- 공식 배포되는 ActiveMQ 대신 GeoServer plugin에 포함된 ActiveMQ 사용 

- ActiveMQ, Tomcat 다운로드 
  
```
$ docker exec -it mq1 bash

$ mkdir /opt/install && cd /opt/install
$ curl -OL https://build.geoserver.org/geoserver/2.17.x/community-latest/geoserver-2.17-SNAPSHOT-activeMQ-broker-plugin.zip
$ curl -OL http://apache.tt.co.kr/tomcat/tomcat-9/v9.0.38/bin/apache-tomcat-9.0.38.tar.gz
```

## 1.3. 실행 옵션 설정 및 실행 

- Tomcat 설정 

```
$ docker exec -it mq1 bash

$ cd /opt/install
$ tar zxvf apache-tomcat-9.0.38.tar.gz 
$ mv apache-tomcat-9.0.38 ../tomcat
$ unzip geoserver-2.17-SNAPSHOT-activeMQ-broker-plugin.zip 
$ mv activemqBroker-2.17-SNAPSHOT.war ../tomcat/webapps/broker.war

$ cd /opt/tomcat
$ vi ./bin/setenv.sh
export CATALINA_OPTS="${CATALINA_OPTS} -server -Dfile.encoding=UTF-8"
export CATALINA_OPTS="${CATALINA_OPTS} -Xms1g -Xmx1g -XX:+UseConcMarkSweepGC"
export CATALINA_OPTS="${CATALINA_OPTS} -DGEOSERVER_DATA_DIR=${GEOSERVER_DATA_DIR}"
# mq1:61666 : message 통신을 위한 설정. mq1은 Container 명(Container 실행 시 network를 설정하면 Container 사이에는 Container 명으로 통신 가능)
export CATALINA_OPTS="${CATALINA_OPTS} -Dactivemq.transportConnectors.server.uri=\"tcp://mq1:61666?maximumConnections=1000&wireFormat.maxFrameSize=104857600&jms.useAsyncSend=true&transport.daemon=true\""
```

- 실행
```
$ cd /opt/tomcat && ./bin/startup.sh
```

# 2. GeoServer 설정 

## 2.1. 기본 구성 

- GeoServer를 위한 Docker Container 생성 

```
$ docker run -d --privileged --name gs1 --network gs_cluster \
-p 18080:8080 \
-v /opt/gs_cluster/data_dir:/opt/data/data_dir \
-e GEOSERVER_DATA_DIR=/opt/data/data_dir \
-e GEOSERVER_LOG_LOCATION=/opt/data/logs/geoserver1.log \
-e CLUSTER_CONFIG_DIR=/opt/data/data_dir/cluster/geoserver1 \
centos:7 /sbin/init
```

- 필요 라이브러리 및 OpenJDK 설치

```
$ docker exec -it gs1 bash

$ mkdir /opt/data/logs

$ yum install -y epel-release
$ yum install -y java-11-openjdk java-11-openjdk-devel unzip net-tools
``` 

## 2.2. GeoServer, GeoServer Clustering plugin, Tomcat 다운로드 

- GeoServer 및 plugin 다운로드 

```
$ docker exec -it gs1 bash 

$ mkdir /opt/install && cd /opt/install 
$ curl -OL http://sourceforge.net/projects/geoserver/files/GeoServer/2.17.2/geoserver-2.17.2-war.zip 
$ curl -OL https://build.geoserver.org/geoserver/2.17.x/community-latest/geoserver-2.17-SNAPSHOT-jms-cluster-plugin.zip 
```

- Tomcat 다운로드 

```
$ cd /opt/install

$ curl -OL http://apache.tt.co.kr/tomcat/tomcat-9/v9.0.38/bin/apache-tomcat-9.0.38.tar.gz
```

## 2.3. Tomcat 실행 옵션 설정 및 실행 

- Tomcat 설정 

```
$ cd /opt/install
$ tar zxvf apache-tomcat-9.0.38.tar.gz 
$ mv apache-tomcat-9.0.38 ../tomcat
$ unzip geoserver-2.17.2-war.zip -d geoserver && cd geoserver
$ unzip geoserver.war -d geoserver
$ cp -r geoserver/data/* /opt/data/data_dir/.
$ mv geoserver /opt/tomcat/webapps/. && cd .. && rm -rf geoserver

$ cd /opt/tomcat 
$ vi ./bin/setenv.sh
export CATALINA_OPTS="${CATALINA_OPTS} -server -Dfile.encoding=UTF-8"
export CATALINA_OPTS="${CATALINA_OPTS} -Xms2g -Xmx2g -Xss1m"
export CATALINA_OPTS="${CATALINA_OPTS} -XX:+UseParallelGC"
```

- 실행
```
$ cd /opt/tomcat && ./bin/startup.sh
```

## 2.4. Clustering 설정 

- GeoServer plugin 추가 

```
$ cd /opt/tomcat && ./bin/shutdown.sh

$ cd /opt/install
$ unzip geoserver-2.17-SNAPSHOT-jms-cluster-plugin.zip -d geoserver && cd geoserver
$ curl -OL https://repo1.maven.org/maven2/org/apache/commons/commons-pool2/2.0/commons-pool2-2.0.jar
# cp * /opt/tomcat/webapps/geoserver/WEB-INF/lib/.
```

- Clustering 설정 추가 

```
$ mkdir -p $CLUSTER_CONFIG_DIR  && cd $CLUSTER_CONFIG_DIR
$ vi cluster.properties
toggleSlave=true
connection=enabled
topicName=VirtualTopic.geoserver
brokerURL=tcp://mq1:61666
durable=true
xbeanURL=./broker.xml
toggleMaster=true
embeddedBroker=disabled
CLUSTER_CONFIG_DIR=/opt/data/data_dir/cluster/geoserver1
embeddedBrokerProperties=embedded-broker.properties
connection.retry=10
readOnly=disabled
instanceName=geoserver1
group=geoserver-cluster
connection.maxwait=500
```

- Tomcat 실행 

```
$ cd /opt/tomcat && ./bin/startup.sh
```

## 2.5. GeoServer 추가 

- 설정이 되어 있는 GeoServer Container 복사 

```
$ docker commit gs1 gs_cluster:1.0

$ docker run -d --privileged --name gs2 --network gs_cluster \
-p 28080:8080 \
-v /opt/gs_cluster/data_dir:/opt/data/data_dir \
-e GEOSERVER_DATA_DIR=/opt/data/data_dir \
-e GEOSERVER_LOG_LOCATION=/opt/data/logs/geoserver2.log \
-e CLUSTER_CONFIG_DIR=/opt/data/data_dir/cluster/geoserver2 \
gs_cluster:1.0 /sbin/init
```

- 설정 변경 및 실행 

```
$ docker exec -it gs2 bash

$ mkdir -p $CLUSTER_CONFIG_DIR  && cd $CLUSTER_CONFIG_DIR
$ cp ../geoserver1/cluster.properties .
$ vi cluster.properties
CLUSTER_CONFIG_DIR=/opt/data/data_dir/cluster/geoserver2
instanceName=geoserver2
readOnly=disabled
brokerURL=tcp\://mq1\:61666
durable=true
embeddedBroker=disabled
toggleMaster=true
connection.retry=10
xbeanURL=./broker.xml
embeddedBrokerProperties=embedded-broker.properties
topicName=VirtualTopic.geoserver
connection=enabled
toggleSlave=true
connection.maxwait=500
group=geoserver-cluster

$ cd /opt/tomcat && ./bin/startup.sh
```

