# Hương dẫn cài đặt Consul

Mô hình Lab:


|Server|Hostname|IP|
|------|--------|--|
|SwarmMaster|host1|10.159.19.77|
|SwarmClient|host2|10.159.19.84|

Stack consul gồm service consul master và consul client trong đó:
 - Service consul master: nằm trên host2
 - Service consul client: nằm trên các host còn lại

## 1. Cài đặt Consul

### 1.1. Trên host Swarm master, tạo file stack `consul_master_client.yml` như sau:
```sh
version: "3.7"

networks:
  consul:
    driver: overlay
services:
 consul-master:
  hostname: ${HOSTNAME}
  image: consul:latest
  restart: always
  ports:
   - "8300:8302"
   - "8301-8302:8301-8302/udp"
   - "8500:8500"
   - "8600:8600/udp"  
  networks:
      consul:
        aliases:
          - consul.cluster
  entrypoint:
      - consul
      - agent
      - -server
      - -ui
      - -node=server-1 
      - -bootstrap-expect=1
      - -datacenter=lab
      - -client=0.0.0.0
      - -retry-join=consul.cluster
      - -bind={{ GetInterfaceIP "eth0" }}
      - -data-dir= /opt/consul
      - -encrypt=pju04BkafjtJ19xONoFGWL/BQG1/qO+TEojredSrTTM=
      - -raft-protocol=1
  volumes:
      - /sys:/host/sys:ro
      - /:/rootfs:ro
      - /proc:/host/proc:ro
  deploy:
   mode: global
   restart_policy:
     condition: on-failure
   placement:
     constraints:
       - node.hostname==minio8-dev

 consul-client:
  hostname: ${HOSTNAME}
  container_name: consul_client
  image: consul:latest
  restart: always
  networks:
    consul:
      aliases:
       - consul.cluster.client
  entrypoint:
      - consul
      - agent
      - -datacenter=lab
      - -retry-join=consul.cluster
      - -bind={{ GetInterfaceIP "eth0" }}
      - -data-dir= /opt/consul
      - -config-dir=/consul/config
      - -encrypt=pju04BkafjtJ19xONoFGWL/BQG1/qO+TEojredSrTTM=
      - -raft-protocol=1
  volumes:
      - /sys:/host/sys:ro
      - /:/rootfs:ro
      - /proc:/host/proc:ro
      - /home/sysadmin/consul/c_advisor.json:/consul/config/c_advisor.json
      - /home/sysadmin/consul/node_exporter.json:/consul/config/node_exporter.json
  deploy:
   mode: global
   restart_policy:
     condition: on-failure
   placement:
     constraints: [node.platform.os == linux]
```

### 1.2. Trên host Swarm master, khởi tạo stack consul
```sh
docker stack deploy -c consul_master_client.yml consul_master_client
```

### 1.3. Kiểm tra service consul
```sh
docker stack services consul_master_client
```

Kết quả:

```sh
D                  NAME                                 MODE                REPLICAS            IMAGE               PORTS
71p4i2owh7kv        consul_master_client_consul-master   global              1/1                 consul:latest       *:8300->8302/tcp, *:8500->8500/tcp, *:8301-8302->8301-8302/udp, *:8600->8600/udp
ktahddz14sur        consul_master_client_consul-client   global              1/1                 consul:latest       
```

### 1.4 Kiểm tra việc cài đặt
Trên host2, truy cập vào shell của container consul master
```sh
docker exec -it c77cadff3569 /bin/sh
```

Thực hiện lệnh
```sh
consul members
```

Kết quả:
```sh
Node          Address        Status  Type    Build  Protocol  DC   Segment
server-1      10.0.6.3:8301  alive   server  1.8.0  2         lab  <all>
04e080f86ab7  10.0.6.6:8301  alive   client  1.8.0  2         lab  <default>
```

## 2. Cấu hình Discovery trên Consul
### 2.1. Cấu hình để Consul tự động lấy metric mới từ node exporter và gửi về Prometheus
Trên các host swarm client, tại thư mục `/home/sysadmin/consul`, tạo file `node_exporter.json` với nội dung

```sh
{
  "service":
  {"name": "node_exporter",
   "tags": ["node_exporter&c_advisor", "prometheus"],
   "address": "10.159.19.77",
   "port": 9102
  }
}

```

### 2.2. Cấu hình để Consul tự động lấy metric container mới từ cAdvisor và gửi về Prometheus
Trên host minIO, tại thư mục `/home/sysadmin/consul`, tạo file `c_advisor.json` với nội dung
```sh
{
  "service":
  {"name": "c_advisor",
   "tags": ["node_exporter&c_advisor", "prometheus"],
   "address": "10.159.19.77",
   "port": 8082
  }
}
```

### 2.2. Update lại service consul client để cập nhật các cấu hình trên
```sh
docker service update --force ktahddz14sur
```

Trong đó `ktahddz14sur` là ID của service consul client.
