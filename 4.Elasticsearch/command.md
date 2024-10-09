# Elasticsearch Cluster High Availability

# Thực hiện trên cả 3 servers

Cài đặt ElasticSearch

## Tải và lưu khóa GPG dùng để xác thực các gói phần mềm Elasticsearch

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

## Cài đặt các gói công cụ cần thiết

```bash
sudo apt-get install apt-transport-https net-tools
```

## Thêm Elasticsearch 8.x vào danh sách các nguồn gói của hệ thống để cài bằng lệnh apt

```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

## Cài đặt ElasticSearch

```bash
sudo apt-get update && sudo apt-get install elasticsearch
```

## Kiểm tra

```bash
systemctl status elasticsearch
## file cấu hình
 ls /etc/elasticsearch/
# certs                   elasticsearch-plugins.example.yml  jvm.options    log4j2.properties  roles.yml  users_roles
# elasticsearch.keystore  elasticsearch.yml                  jvm.options.d  role_mapping.yml   users
```

# Thực hiện trên server 1

Cấu hình ES server 1 thành master của ES cluster

Để cấu hình cluster thì có 2 cách cấu hình: Cách 1 là khởi tạo một Node trước rồi từ từ join các Node còn lại vào Node master, Cách 2 join tất các Node vào một lúc
==> Sử dụng cách 1 thực tế hơn

## Mở file cấu hình elasticsearch và chỉnh sửa

```bash
vi /etc/elasticsearch/elasticsearch.yml
```

## Chỉnh sửa những dòng sau đây (lưu ý phải xem video bài giảng)

```bash
cluster.name: es-cluster-viettu
node.name: server-1
network.host: 192.168.72.11
http.port: 9200
transport.host: 0.0.0.0
```

## Khởi động ES để tạo cụm

```bash
systemctl start elasticsearch
systemctl status elasticsearch
```

## Di chuyển đến thư mục làm việc bashscript của ES

```bash
cd /usr/share/elasticsearch/bin/
```

## Tiến hành reset lại mật khẩu

```bash
./elasticsearch-reset-password -i -u elastic
>> 123456
```

## Kiểm tra trạng thái bằng câu lệnh

```bash
curl -k -u elastic:123456 https://192.168.72.11:9200/

# root@server-1:/usr/share/elasticsearch/bin# curl -k -u elastic:123456 https://192.168.72.11:9200/
# {
#   "name" : "server-1",
#   "cluster_name" : "es-cluster-viettu",
#   "cluster_uuid" : "ozKJPUQOT2yR9PVAey_7gw",
#   "version" : {
#     "number" : "8.15.0",
#     "build_flavor" : "default",
#     "build_type" : "deb",
#     "build_hash" : "1a77947f34deddb41af25e6f0ddb8e830159c179",
#     "build_date" : "2024-08-05T10:05:34.233336849Z",
#     "build_snapshot" : false,
#     "lucene_version" : "9.11.1",
#     "minimum_wire_compatibility_version" : "7.17.0",
#     "minimum_index_compatibility_version" : "7.0.0"
#   },
#   "tagline" : "You Know, for Search"
# }
```

## Kiểm tra trạng thái thông tin của ES cluster bằng lệnh

```bash
curl -k -u elastic:123456 https://192.168.72.11:9200/_cluster/health?pretty
# root@server-1:/usr/share/elasticsearch/bin# curl -k -u elastic:123456 https://192.168.72.11:9200/_cluster/health?pretty
# {
#   "cluster_name" : "es-cluster-viettu",
#   "status" : "green",
#   "timed_out" : false,
#   "number_of_nodes" : 1,
#   "number_of_data_nodes" : 1,
#   "active_primary_shards" : 1,
#   "active_shards" : 1,
#   "relocating_shards" : 0,
#   "initializing_shards" : 0,
#   "unassigned_shards" : 0,
#   "delayed_unassigned_shards" : 0,
#   "number_of_pending_tasks" : 0,
#   "number_of_in_flight_fetch" : 0,
#   "task_max_waiting_in_queue_millis" : 0,
#   "active_shards_percent_as_number" : 100.0
# }
```

## Khởi tạo token để cho server 2 và server 3 join vào ES cluster

```bash
./elasticsearch-create-enrollment-token -s node
>> <token>

# eyJ2ZXIiOiI4LjE0LjAiLCJhZHIiOlsiMTkyLjE2OC43Mi4xMTo5MjAwIl0sImZnciI6IjIxYzVhYTRhNWQ0N2M1YjdlZGUyZDc5YmRmYjc0MWQ4MjhhMGQwNzEwZDNmYTgxYjgwM2FlZDI3MThkY2UxYzgiLCJrZXkiOiJwTFJqZFpFQmJWX0FaWVh3UWF3OToxUGZCUVItUlJ0eXI1VTVscW1XQlV3In0=
```

# Thực hiện trên server 2 và server 3

Join ES server 2 vào ES cluster (đã thiết lập ở server 1)

## Di chuyển đến thư mục làm việc bashscript của ES

```bash
cd /usr/share/elasticsearch/bin/
```

## Xác nhận cấu hình token từ ES server 1

```bash
./elasticsearch-reconfigure-node --enrollment-token <token>
```

# Thực hiện trên server 2 và 3

```bash
vi /etc/elasticsearch/elasticsearch.yml
```

## Chỉnh sửa những dòng sau đây

```bash

## Ở đây có một trường khác với server-1
discovery.seed_hosts: ["192.168.72.11:9300"] ## tức là khi xảy ra lỗi thì những server nào được phép để cử lên làm master, hiện chỉ thấy có duy nhất server-1  của ta được cấu hình, nếu để mặc định như này server master 1 bị lỗi thì chắc chắn 2 server kia sẽ không được đề của làm master
```

# Thực hiện trên `server 1`

Thay đổi phần discovery.seed_hosts

## Mở lại file cấu hình ES và tìm đến phần Discovery

```bash
vi /etc/elasticsearch/elasticsearch.yml
```

## Sửa cấu hình thành như sau (tương ứng địa chỉ IP của bạn)

```bash
discovery.seed_hosts:
  - 192.168.72.11:9300 ## port 9300 dùng để giao tiếp giữa các node
  - 192.168.72.12:9300
  - 192.168.72.13:9300
```

==> server 1 gặp vấn đề thì 2 server kia cũng được phép làm master

## Khởi động lại ES

```bash
systemctl restart elasticsearch
```

## Thực hiện trên server 2 và server 3

Cấu hình thiết lập cụm

## Mở file cấu hình ES

```bash
vi /etc/elasticsearch/elasticsearch.yml
```

## Thay đổi các thông tin tương ứng server 2 và server 3 như sau

```yml
cluster.name: es-cluster-viettu
node.name: server-2 hoặc server-3
network.host: 192.168.72.12 hoặc 192.168.72.13
http.port: 9200
discovery.seed_hosts:
  - 192.168.72.11:9300
  - 192.168.72.12:9300
  - 192.168.72.13:9300
```

## Khởi động lại ES

```bash
systemctl start elasticsearch
```

## Kiểm tra trạng thái thông tin của ES cluster bằng lệnh

```bash
curl -k -u elastic:123456 https://192.168.72.12:9200/_cluster/health?pretty
# {
#   "cluster_name" : "es-cluster-viettu",
#   "status" : "green",
#   "timed_out" : false,
#   "number_of_nodes" : 2, ## đã có 2 Node
#   "number_of_data_nodes" : 2,
#   "active_primary_shards" : 1,
#   "active_shards" : 2,
#   "relocating_shards" : 0,
#   "initializing_shards" : 0,
#   "unassigned_shards" : 0,
#   "delayed_unassigned_shards" : 0,
#   "number_of_pending_tasks" : 0,
#   "number_of_in_flight_fetch" : 0,
#   "task_max_waiting_in_queue_millis" : 0,
#   "active_shards_percent_as_number" : 100.0
# }
```

```bash
    _cluster/health?pretty
# {
#   "cluster_name" : "es-cluster-viettu",
#   "status" : "green",
#   "timed_out" : false,
#   "number_of_nodes" : 3, ## 3 node
#   "number_of_data_nodes" : 3,
#   "active_primary_shards" : 1,
#   "active_shards" : 2,
#   "relocating_shards" : 0,
#   "initializing_shards" : 0,
#   "unassigned_shards" : 0,
#   "delayed_unassigned_shards" : 0,
#   "number_of_pending_tasks" : 0,
#   "number_of_in_flight_fetch" : 0,
#   "task_max_waiting_in_queue_millis" : 0,
#   "active_shards_percent_as_number" : 100.0
# }
```

## câu lệnh khác kiểm tra trên server 1

```bash
root@server-1:/usr/share/elasticsearch/bin# curl -k -u elastic:123456 https://192.168.72.11:9200/_cat/nodes
192.168.72.11 30 94 41 1.50 0.68 0.31 cdfhilmrstw * server-1 ## server-1 là master
192.168.72.12 38 96  1 0.00 0.05 0.10 cdfhilmrstw - server-2
192.168.72.13 33 97  4 0.23 0.22 0.09 cdfhilmrstw - server-3
```

# Kiểm tra khi server 1 có vấn đề thì server 2 và 3 có được đề cử lên làm master hay không

```bash
systemctl stop elasticsearch
```

## kiểm tra trên server-2

```bash
root@server-2:/usr/share/elasticsearch/bin#  curl -k -u elastic:123456 https://192.168.72.12:9200/_cat/nodes
192.168.72.12 43 96  0 0.03 0.03 0.07 cdfhilmrstw * server-2 ## server 2 được đề cử lên làm master
192.168.72.13 37 91 12 0.08 0.49 0.34 cdfhilmrstw - server-3
```

## khởi động ES ở server-1

```bash
systemctl start elasticsearch

curl -k -u elastic:123456 https://192.168.72.11:9200/_cat/nodes
192.168.72.12  8 96 0 0.00 0.01 0.06 cdfhilmrstw * server-2
192.168.72.11 42 91 0 0.01 0.28 0.28 cdfhilmrstw - server-1 ## server-1 đã lên nhưng hiện tại đang replica của server-2
192.168.72.13 29 97 0 0.37 0.23 0.12 cdfhilmrstw - server-3

```

==> Cụm ES hoạt động chính xác

---

# Thực hiện trên `server 1`

Thiết lập Kibana

## Cài đặt Kibana

Bởi vì đây là một bộ công cụ đi với nhau, thế nên là những cái thông tin, package khi ta thêm ở 2 câu lệnh cài es thì nó đã có kibana rồi

```bash
sudo apt-get update && sudo apt-get install kibana
## file cấu hình
ls /etc/kibana/
# kibana.keystore  kibana.yml  node.options
```

## Mở file cấu hình Kibana

```bash
vi /etc/kibana/kibana.yml
```

## Tạo thư mục certs cho kibana

Truy cập kibana bằng https

```bash
mkdir /etc/kibana/certs
```

## Tạo crt và key để truy cập kibana với https

```bash
openssl req -newkey rsa:4096 -nodes -sha256 -keyout /etc/kibana/certs/domain.key -subj "/CN=192.168.72.11" -addext "subjectAltName = DNS:192.168.72.11,IP:192.168.72.11" -x509 -days 365 -out /etc/kibana/certs/domain.crt
```

## Copy CA của ES sang Kibana

```bash
cp /etc/elasticsearch/certs/http_ca.crt /etc/kibana/certs/
```

## Mở file cấu hình Kibana

```bash
vi /etc/kibana/kibana.yml
```

## Sửa các nội dung sau

```yml
server.port: 5601
server.host: "0.0.0.0"
server.publicBaseUrl: "http://192.168.72.11:5601/"
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/certs/domain.crt
server.ssl.key: /etc/kibana/certs/domain.key
elasticsearch.hosts:
  - https://192.168.72.11:9200 # port 9200 dùng để sd, còn port 9300 dùng để gt các node
  - https://192.168.72.12:9200
  - https://192.168.72.13:9200
## bở vì ta đùng cái xác thực tự ký nên ssl phải thêm cấu hình
elasticsearch.ssl.certificate: /etc/kibana/certs/domain.crt
elasticsearch.ssl.key: /etc/kibana/certs/domain.key
elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/http_ca.crt"]
elasticsearch.ssl.verificationMode: full
```

## Thay đổi toàn bộ quyền thư mục /etc/kibana cho user kibana

```bash
chown -R kibana:kibana /etc/kibana/
```

## Tạo kibana token xác thực

```bash
curl -k -X POST -u elastic:123456 https://192.168.72.11:9200/_security/service/elastic/kibana/credential/token/kibana_token


# root@server-1:/usr/share/elasticsearch/bin# curl -k -X POST -u elastic:123456 https://192.168.72.11:9200/_security/service/elastic/kibana/credential/token/kibana_token
# {"created":true,"token":{"name":"kibana_token","value":"AAEAAWVsYXN0aWMva2liYW5hL2tpYmFuYV90b2tlbjo0YldkQzlrbVRyMk04cG13N0JCWXZB"}}

```

## Di chuyển đến thư mục làm việc bashscript của kibana

```bash
cd /usr/share/kibana/bin/
```

## Khởi chạy lệnh add elasticsearch.serviceAccountToken

```bash
./kibana-keystore add elasticsearch.serviceAccountToken
# xong nhập tocken vừa tạo
```

## Tiến hành khởi chạy Kibana

```bash
systemctl start kibana
```

## truy cập trình duyệt

```bash
https://192.168.72.11:5601/login?next=%2F

# username: elastic
# password: 123456
Truy cập devtools
PUT viettu_index

POST viettu_index/_doc
{"key1": "value1"}

GET /_cat/shards/viettu_index

# viettu_index 0 r STARTED 1 4.7kb 4.7kb 192.168.72.11 server-1
# viettu_index 0 p STARTED 1 4.7kb 4.7kb 192.168.72.12 server-2
## thắc mắc tại sao có  2 server ở đây
GET /viettu_index/_settings
{
  "viettu_index": {
    "settings": {
      "index": {
        "routing": {
          "allocation": {
            "include": {
              "_tier_preference": "data_content"
            }
          }
        },
        "number_of_shards": "1", # Tức là index này chỉ được chứa một master
        "provided_name": "viettu_index",
        "creation_date": "1724256983488",
        "number_of_replicas": "1", # Và chứa một replica
        "uuid": "SHb8DcLsQgKqOSO7ZE1sXQ",
        "version": {
          "created": "8512000"
        }
      }
    }
  }
}

## thay đổi replica
PUT /viettu_index/_settings
{
  "index": {
    "number_of_replicas": "2"
  }
}
# kiểm tra
GET /_cat/shards/viettu_index

viettu_index 0 r STARTED 1 4.7kb 4.7kb 192.168.72.11 server-1
viettu_index 0 r STARTED 1 4.7kb 4.7kb 192.168.72.13 server-3
viettu_index 0 p STARTED 1 4.7kb 4.7kb 192.168.72.12 server-2

==> Lúc này index này đã được replica trên cả 2 server 1 và 3

```
