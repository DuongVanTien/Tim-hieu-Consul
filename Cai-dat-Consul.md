# Hương dẫn cài đặt Consul

Mô hình Lab

|Server|Hostname|IP|
|------|--------|--|
|Master|consul01|192.168.1.1|
|Client|consul02|192.168.1.2|

## 1. Cài đặt Consul binary. Thực hiện trên cả 2 Node

- **Download gói [tại đây](https://releases.hashicorp.com/consul/)**
```sh
yum install wget unzip -y
wget https://releases.hashicorp.com/consul/1.7.3/consul_1.7.3_linux_amd64.zip
```
- **Giải nén và copy đến thư mục binary**
```sh
unzip consul_1.7.3_linux_amd64.zip
cp consul /usr/local/bin/
```
- **Kiểm tra** 
```sh
$ consul -v
Consul v1.7.3
Protocol 2 spoken by default, understands 2 to 3 (agent will automatically use protocol >2 when speaking to compatible agents)
```

## 2. Thực hiện trên Node Master

- **Tạo user cho consul**
```sh
sudo groupadd --system consul
sudo useradd -s /sbin/nologin --system -g consul consul
```
- **Tạo thư mục config, data, log cho serivice sonsul.**
```sh
sudo mkdir -p /var/lib/consul /etc/consul.d /var/log/consul/
sudo chown -R consul:consul /var/lib/consul /etc/consul.d /var/log/consul/
sudo chmod -R 775 /var/lib/consul /etc/consul.d /var/log/consul/
```
- Sửa file `/etc/hosts
```sh
cat << EOF >> /etc/hosts 
192.168.1.1     consul01
192.168.1.2     consul02
```
- **Tạo file `/etc/systemd/system/consul.service` với nội dung sau:**
```sh
vim /etc/systemd/system/consul.service
[Unit]
Description="HashiCorp Consul - A service mesh solution"
Documentation=https://www.consul.io/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/consul.d/consul.hcl

[Service]
Type=simple
User=consul
Group=consul
ExecStart=/usr/local/bin/consul agent -config-dir=/etc/consul.d/
ExecReload=/usr/local/bin/consul reload
KillMode=process
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
- **Tạo Consul secret**
```sh
$ consul keygen
WxC/uaO9+p+3EdJsUWPi46QYXOYdptemDO4mC771cR4=
```
- **Tạo file `/etc/consul.d/consul.hcl` với nội dung sau:**
```
datacenter = "noisepalace"
data_dir = "/opt/consul"
encrypt = "WxC/uaO9+p+3EdJsUWPi46QYXOYdptemDO4mC771cR4="
retry_join = ["192.168.1.1"]
bind_addr = "192.168.1.1"

performance {
  raft_multiplier = 1
}
```
- **Tạo file `/etc/consul.d/server.hcl` với nội dung sau:**
```sh
server = true
bootstrap_expect = 1
ui = true
bind_addr = "192.168.1.1"
client_addr = "0.0.0.0"
```
- **Start service Consul**
```sh
systemctl daemon-reload
systemctl start consul
systemctl enable consul
```
- **Truy cập Web để kiểm tra:** http://192.168.1.1:8500

## 3. Thực hiện trên Node Client
- **Tạo file `/etc/systemd/system/consul-client.service` với nội dung sau:**
```sh
vim /etc/systemd/system/consul-client.service
[Unit]
Description=Consul Service Discovery Agent - Client Mode
Documentation=https://www.consul.io/
Requires=network-online.target
After=network-online.target
 
[Service]
Type=simple
User=consul
Group=consul
ExecStart=/usr/bin/consul agent -config-dir=/etc/consul.d/
ExecReload=/usr/bin/consul reload
TimeoutStartSec=0
Restart=on-failure
SyslogIdentifier=consul
KillMode=process
LimitNOFILE=65536
 
[Install]
WantedBy=default.target
```
- **Tạo file `/etc/consul.d/config.json` với nội dung sau:**
```sh
vim /etc/consul.d/config.json
{
    "server": false,
    "datacenter": "noisepalace",
    "data_dir": "/opt/consul",
    "encrypt": "WxC/uaO9+p+3EdJsUWPi46QYXOYdptemDO4mC771cR4=",
    "log_level": "INFO",
    "enable_syslog": true,
    "leave_on_terminate": true,
    "start_join": [
        "consul01"
    ]
}
```
Trong đó:
- `server: false`: Sử dụng ở Mode Client
- `datacenter`: khai báo thông tin datacenter khớp với data center trên Cluster Consul Server.
- `data_dir`: thư mục chứa data hoạt động của dịch vụ Consul Client.
- `encrypt`: chuỗi key dùng để mã hoá trao đổi thông tin giữa Consul Client và Consul Server. Chuỗi key này lấy ở phần cấu hình Consul Cluster.
- `log_level`: mức độ log.
- `enable_syslog`: kích hoạt ghi log ở chế độ syslog.
- `leave_on_terminate`: nếu kích hoạt thì khi agent nhận được tín hiệu TERM, sẽ gửi thông điệp ‘Leave’ rời khỏi cụm cluster.
- `start_join`: danh sách các máy chủ Consul Server mà Consul Client cần kết nối.

- **Start service Consul-client**
```sh
systemctl daemon-reload
systemctl start consul-client
systemctl enable consul-client
```
## 3. Khởi tạo Service

- **Sử dụng command khởi tạo Service**

***Lưu ý: Service tạo trên Node nào sẽ chỉ hoạt động trên Node đó***

Ví dụ: Khởi tạo Service `node_exporter` có tag `node-exporter`, port `9100`.
```sh
consul services register -name node_exporter -tag node-exporter -port 9100
```
Kiểm tra service vừa tạo bằn lệnh sau:
```sh
consul catalog services
```
Kiểm tra service vừa tạo có trên Node nào
```sh
consul catalog nodes -service=node_exporter
```
- **Sử dụng file `/etc/consul.d/service_web.json` khởi tạo Service**

***Lưu ý: File này phải nằm trong thư mục /etc/consul.d***

Tạo file  `/etc/consul.d/service_web.json` với nội dung sau:

*Tương tự như command trên: Tạo service `web`, tag `nginx`, port `80`*

```sh
vim /etc/consul.d/service_web.json
{
  "service": {
    "name": "web",
    "tags": [
      "nginx"
    ],
    "port": 80
  }
}
```
Restart lại service consul
```sh
systemctl restart consul
```
