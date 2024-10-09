# Redis Sentinel High Availability

- Redis Sentinel và Cluster có gì khác biệt [gg][redis sentinel vs redis cluster]
- Redis sentinel tập trung vào khả năng tự động chuyển đổi dự phòng, tức là failover cho một cụm redis master slave, tức là ta sẽ cài một cụm redis master slave rồi cấu hình thêm sentinel như một hệ thống giám sát và quản lý các redis trên server đó, để khi mà ví dụ redis master nỗi chẳng hạn thì nó sẽ tự động đề cử một slave khác nên làm master
- Redis cluster thì mục đích chính là cung cấp khả năng phân tán dữ liệu chính là data shearing và tính HA, thì redis cluster sẽ tự động chia dữ liệu thành nhiều phần và phân phối chúng ở trên nhiều node
- Và mô hình redis sentinel và redis cluster cũng sẽ có sự khác nhau, thì redis sentinel sẽ triển khai theo master slave, còn redis cluster thì phân tán dữ liệu thành các slot và mỗi node thì chịu trách nhiệm cho một hoặc nhiều slot

==> Redis sentinal sẽ phù hợp cho những trường hợp cần tính HA cao, đồng thời các triển khai bằng sentinel sẽ đơn giản hơn redis cluster, nên thực sentinel sẽ được ưa chuộng hơn
==> Trong hệ thống có khả năng mở rộng quy mô và phân tán dữ liệu thì redis cluster sẽ tối ưu hơn

# Thực hiện trên cả `3 servers`

Cài đặt Redis

## Cài đặt redis và redis-sentinel

**Lưu ý:** trong thực tế để cài được một công cụ nào đó thì có 3 cách chính, cách 1: add repo của công cụ đó trên internet, cách 2: viết file repo riêng của công cụ, cách 3: install trực tiếp nếu OS đã có sẵn công cụ đó. VD cài theo cách 3, giả sử ubuntu có sẵn redis thì ta chỉ cần `apt install redis` là xong, nhưng nó chính xác là version mặc định trong cái od đó rồi, nhưng ta muốn cài version chính xác và version khác version OS chằng hạn thì lúc đó ta sẽ phải áp dụng những phương pháp trên để mà có thể kéo được các repo và package của công cụ đó về trên server

```bash
sudo apt install -y lsb-release curl gpg
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
sudo apt update -y
sudo apt install -y redis redis-sentinel
```

## Kiểm tra

```bash
systemctl status redis-server
systemctl enable redis-server redis-sentinel
```

==> Cài một cụm redis trước, rồi mới sử dụng redis sentinel để giám sát và quản lý đảm bảo tính HA và failover

# Thực hiện trên `server 1`

Cấu hình redis cluster

## Mở file cấu hình

```bash
 vi /etc/redis/redis.conf
```

Chỉnh sửa các nội dung sau: đế cấu để mà tạo thành một cụm redis

```conf
bind 0.0.0.0
masterauth viettu # mật khảu  xác thực cụm
requirepass viettu # mật khẩu để kết nối từ client vào
```

# Thực hiện trên server 2 và server 3

Cấu hình redis cluster

## Mở file cấu hình

```bash
vi /etc/redis/redis.conf
```

## Chỉnh sửa các nội dung sau

```bash
bind 0.0.0.0
masterauth viettu
requirepass viettu
replicaof 192.168.72.11 6379
```

# Thực hiện trên cả 3 servers

Cấu hình redis cluster

## Khởi động lại redis

```bash
systemctl restart redis-server
systemctl status redis-server
```

==> Tạo dữ liệu trong redis server-1, để server 2,3 có được đồng bộ data không

# Thực hiện trên server 1

Kiểm tra HA

## Truy cập vào môi trường redis

```bash
redis-cli
```

## Tiến hành thêm dữ liệu

```bash
> > auth viettu
> > SET viettu_popular_series '[{"name": "DevOps for Freshers"}, {"name": "Pipeline DevSecOps"}, {"name": "HA tools"}]'
> > GET viettu_popular_series

# root@server-1:~# redis-cli
# 127.0.0.1:6379> auth viettu
# OK
# 127.0.0.1:6379> SET viettu_popular_series '[{"name": "DevOps for Freshers"}, {"name": "Pipeline DevSecOps"}, {"name": "HA tools"}]'
# OK
# 127.0.0.1:6379> GET viettu_popular_series
# "[{\"name\": \"DevOps for Freshers\"}, {\"name\": \"Pipeline DevSecOps\"}, {\"name\": \"HA tools\"}]"
# 127.0.0.1:6379>
```

# Thực hiện trên server 2 và server 3

Kiểm tra HA

## Truy cập vào môi trường redis

```bash
redis-cli
```

## Tiến hành kiểm tra dữ liệu

```bash
>> auth viettu
>> GET viettu_popular_series

# root@server-2:~# redis-cli
# 127.0.0.1:6379> auth viettu
# OK
# 127.0.0.1:6379>  GET viettu_popular_series
# "[{\"name\": \"DevOps for Freshers\"}, {\"name\": \"Pipeline DevSecOps\"}, {\"name\": \"HA tools\"}]"
# 127.0.0.1:6379>
# 127.0.0.1:6379> set test_key_1 "test_value_1"
# (error) READONLY You can't write against a read only replica.
```

==> Cài thành công cụm redis với 1 master và 2 slave
==> Nhưng hiện tại chưa đảm vảo failover,

## Thử tắt redis trên server 1 đi

```bash
systemctl stop redis-server
```

==> Trên server 2 và 3 chưa có cấu hình nào để kiểm soát rồi tự đề cử lên làm master
==> lúc server 2 và 3 vẫn là replica nên không thể set data

==> Sử dụng sentinel

---

# Thực hiện trên cả 3 servers

Cấu hình sentinel

## Mở file cấu hình

```bash
vi /etc/redis/sentinel.conf
```

## Chỉnh sửa các thông tin sau

```conf
protected-mode no
sentinel monitor mymaster 192.168.72.11 6379 2 # mặc định hiện tại server này đang là master, số 2 có thể hiểu khi set con số này, ví dụ server master 1 chết tức là cần có 2 server khác xác nhận rằng server 1 đã bị lỗi thì nó mới được phép chạy failover (hiểu đơn giản khi server 1 bị lỗi thì server 2 và 3 cùng phải xác nhận rằng server 1 đã gặp vấn đề, cái quá trình failover tức là để cử lên làm master mới hoạt động), xác định rằng  server của ta càn bao nhiêu thành viên để kiểm tra
sentinel auth-pass mymaster viettu # password này chính là masterauth cấu hình trên
## kiểm tra trong vòng bao lâu
sentinel down-after-milliseconds mymaster 5000
## Tối đa quá trình chuyển giao là bao nâu
sentinel failover-timeout mymaster 60000 ## 60s
## khi  có 1 master bị lỗi, và một slave khác được đề cử lên làm master thì số lượng slave còn lại được đồng bộ sẽ như thế nào
sentinel parallel-syncs mymaster 1 # ví dụ trong khoảng một lúc là ta sẽ đồng bộ mấy replica, thì thường để những con số này cho những cụm ít thế này là 1 thôi, để giảm áp lực server master phải chịu, nếu nhiều hơn có thể 2 or 3
```

## Khởi động lại sentinel

```bash
systemctl restart redis-sentinel
systemctl status redis-sentinel

tail -n 100 /var/log/redis/redis-sentinel.log
# 3076:X 20 Aug 2024 10:52:21.114 # +monitor master mymaster 127.0.0.1 6379 quorum 2
# 3555:X 20 Aug 2024 11:35:00.930 * Sentinel ID is 478243212827cb3017360e580e3215024904b6fd
# 3555:X 20 Aug 2024 11:35:00.930 # +monitor master mymaster 192.168.72.11 6379 quorum 2
# 3555:X 20 Aug 2024 11:35:00.931 * +slave slave 192.168.72.12:6379 192.168.72.12 6379 @ mymaster 192.168.72.11 6379
# 3555:X 20 Aug 2024 11:35:00.946 * Sentinel new configuration saved on disk
# 3555:X 20 Aug 2024 11:35:00.946 * +slave slave 192.168.72.13:6379 192.168.72.13 6379 @ mymaster 192.168.72.11 6379

## ta có thấy redis đã tiến hành monitor mymaster, rồi các slave tương ứng
```

## xem thông tin và kiểm ra sâu hơn một chút

```bash
## port sentinel 26379
redis-cli -p 26379 sentinel masters

# root@server-1:~# redis-cli -p 26379 sentinel masters
# 1)  1) "name"
#     2) "mymaster"
#     3) "ip"
#     4) "192.168.72.11"
#     5) "port"
#     6) "6379"
#     7) "runid"
#     8) "63154b0926dff89234e3e669aad9763c7297347d"
#     9) "flags"
#    10) "master"
#    11) "link-pending-commands"
#    12) "0"
#    13) "link-refcount"
#    14) "1"
#    15) "last-ping-sent"
#    16) "0"
#    17) "last-ok-ping-reply"
#    18) "876"
#    19) "last-ping-reply"
#    20) "876"
#    21) "down-after-milliseconds"
#    22) "5000"
#    23) "info-refresh"
#    24) "6310"
#    25) "role-reported"
#    26) "master"
#    27) "role-reported-time"
#    28) "277380"
#    29) "config-epoch"
#    30) "0"
#    31) "num-slaves"
#    32) "2"
#    33) "num-other-sentinels"
#    34) "2"
#    35) "quorum"
#    36) "2"
#    37) "failover-timeout"
#    38) "60000"
#    39) "parallel-syncs"
#    40) "1"
# root@server-1:~#

## ==> Đây chỉ thông tin vừa cấu hình ở trên

## kiểm tra slave của mymaster
redis-cli -p 26379 sentinel slaves mymaster

# root@server-1:~# redis-cli -p 26379 sentinel slaves mymaster
# 1)  1) "name"
#     2) "192.168.72.13:6379"
#     3) "ip"
#     4) "192.168.72.13"
#     5) "port"
#     6) "6379"
#     7) "runid"
#     8) "1568e59bf355c0da7c7e014cb217459c289e3648"
#     9) "flags"
#    10) "slave"
#    11) "link-pending-commands"
#    12) "0"
#    13) "link-refcount"
#    14) "1"
#    15) "last-ping-sent"
#    16) "0"
#    17) "last-ok-ping-reply"
#    18) "523"
#    19) "last-ping-reply"
#    20) "523"
#    21) "down-after-milliseconds"
#    22) "5000"
#    23) "info-refresh"
#    24) "4447"
#    25) "role-reported"
#    26) "slave"
#    27) "role-reported-time"
#    28) "396021"
#    29) "master-link-down-time"
#    30) "0"
#    31) "master-link-status"
#    32) "ok"
#    33) "master-host"
#    34) "192.168.72.11"
#    35) "master-port"
#    36) "6379"
#    37) "slave-priority"
#    38) "100"
#    39) "slave-repl-offset"
#    40) "83314"
#    41) "replica-announced"
#    42) "1"
# 2)  1) "name"
#     2) "192.168.72.12:6379"
#     3) "ip"
#     4) "192.168.72.12"
#     5) "port"
#     6) "6379"
#     7) "runid"
#     8) "b1cd4746de39c93a94726f181441b28cca12c494"
#     9) "flags"
#    10) "slave"
#    11) "link-pending-commands"
#    12) "0"
#    13) "link-refcount"
#    14) "1"
#    15) "last-ping-sent"
#    16) "0"
#    17) "last-ok-ping-reply"
#    18) "523"
#    19) "last-ping-reply"
#    20) "523"
#    21) "down-after-milliseconds"
#    22) "5000"
#    23) "info-refresh"
#    24) "4447"
#    25) "role-reported"

## => Có 2 slave tương ứng là server có ip .12 và .13
```

# stop redis trên server 1

```bash
systemctl  stop redis-server

root@server-2:~# redis-cli -p 26379 sentinel masters
# 1)  1) "name"
#     2) "mymaster"
#     3) "ip"
#     4) "192.168.72.13" ## Ta có thể thấy hiện tại server 3 đã được đề của làm master
#     5) "port"
#     6) "6379"
#     7) "runid"
#     8) "1568e59bf355c0da7c7e014cb217459c289e3648"
#     9) "flags"
#    10) "master"
#    11) "link-pending-commands"
#    12) "0"
#    13) "link-refcount"
#    14) "1"
#    15) "last-ping-sent"
#    16) "0"
#    17) "last-ok-ping-reply"
#    18) "507"
#    19) "last-ping-reply"
#    20) "507"
#    21) "down-after-milliseconds"
#    22) "5000"
#    23) "info-refresh"
#    24) "1121"
#    25) "role-reported"
#    26) "master"
#    27) "role-reported-time"
#    28) "42210"
#    29) "config-epoch"
#    30) "1"
#    31) "num-slaves"
#    32) "2"
#    33) "num-other-sentinels"
#    34) "2"
#    35) "quorum"
#    36) "2"
#    37) "failover-timeout"
#    38) "60000"
#    39) "parallel-syncs"
#    40) "1"


root@server-2:~# redis-cli -p 26379 sentinel slaves mymaster
# 1)  1) "name"
#     2) "192.168.72.12:6379" ## server 2 vẫn như vậy
#     3) "ip"
#     4) "192.168.72.12"
#     5) "port"
#     6) "6379"
#     7) "runid"
#     8) "b1cd4746de39c93a94726f181441b28cca12c494"
#     9) "flags"
#    10) "slave"
#    11) "link-pending-commands"
#    12) "0"
#    13) "link-refcount"
#    14) "1"
#    15) "last-ping-sent"
#    16) "0"
#    17) "last-ok-ping-reply"
#    18) "785"
#    19) "last-ping-reply"
#    20) "785"
#    21) "down-after-milliseconds"
#    22) "5000"
#    23) "info-refresh"
#    24) "5858"
#    25) "role-reported"
#    26) "slave"
#    27) "role-reported-time"
#    28) "157468"
#    29) "master-link-down-time"
#    30) "0"
#    31) "master-link-status"
#    32) "ok"
#    33) "master-host"
#    34) "192.168.72.13"
#    35) "master-port"
#    36) "6379"
#    37) "slave-priority"
#    38) "100"
#    39) "slave-repl-offset"
#    40) "143513"
#    41) "replica-announced"
#    42) "1"
# 2)  1) "name"
#     2) "192.168.72.11:6379" ## Server 1 đã bị chuyển tành savle
#     3) "ip"
#     4) "192.168.72.11"
#     5) "port"
#     6) "6379"
#     7) "runid"
#     8) ""
#     9) "flags"
#    10) "s_down,slave,disconnected" ## chú ý là server 1 trạng thái hiện tại đang là  "s_down". "slave" và "disconnected" tức là server 1 đang có vấn đề kết nối, bởi vì ta vừa stop
#    11) "link-pending-commands"
#    12) "3"
#    13) "link-refcount"
#    14) "1"
#    15) "last-ping-sent"
#    16) "157468"
#    17) "last-ok-ping-reply"
#    18) "157468"
#    19) "last-ping-reply"
#    20) "157468"
#    21) "s-down-time"
#    22) "152391"
#    23) "down-after-milliseconds"
#    24) "5000"
#    25) "info-refresh"
#    26) "0"
#    27) "role-reported"
#    28) "slave"
#    29) "role-reported-time"
#    30) "157468"
#    31) "master-link-down-time"
#    32) "0"
#    33) "master-link-status"
#    34) "err"
#    35) "master-host"
#    36) "?"
#    37) "master-port"
#    38) "0"
#    39) "slave-priority"
#    40) "100"
#    41) "slave-repl-offset"
#    42) "0"
#    43) "replica-announced"
#    44) "1"

```

## và khi server-3 được lên làm master ta có thể thêm dữ liệu

```bash

```

## kiểm tra 2 server được để cử làm master chưa

```bash
root@server-3:~# redis-cli
127.0.0.1:6379> auth viettu
OK
127.0.0.1:6379> set test_key_1 "test_value_1"
OK
127.0.0.1:6379>

## ta có thể thấy server 3 đã hoàn toàn có thể thêm được dữ liệu
```

## Tiến hành bật redis ở server 1 lên

```bash
systemctl  start redis-server
## rõ ràng lúc này server 1 là slave của server 3, tương tự như là server 2

root@server-1:~# redis-cli
127.0.0.1:6379> auth viettu
OK
127.0.0.1:6379> get test_key_1
"test_value_1"
127.0.0.1:6379>
```

# Thực hiện trên `server 1` (Giống cái trên thôi)

Kiểm tra failover

## Truy cập vào môi trường redis

```bash
redis-cli -p 26379 sentinel masters
redis-cli -p 26379 sentinel slaves mymaster
redis-cli -p 26379 sentinel sentinels mymaster
systemctl stop redis-server
```

===> Vậy là cụm redis HA đươc cài đặt thành công

==> Cấu hình nâng cao hơn, chuyển sâu hơn

```bash
root@server-1:~# vi /etc/redis/redis.conf

```

```conf
 maxmemory <bytes> # là giới hạn dung lượng bộ nhớ mà redis có thể sử dụng, vd ta để maxmmemory là 260MB, 1G
appendonly no # mặc định là no, đây là tùy chọn ghi log các lệnh, đảm bảo dữ liệu không bị mất, nên bật nên

appendfsync everysec # everysec mỗi giây # tùy chọn này sẽ quy định tần suất mà redis sẽ ghi dữ liệu từ logs xuống ổ cứng, đây là một phần rất quan trọng trong việc đảm bảo độ tin cậy và tính bền vững của dữ liệu
# ngoài ra còn
# appendfsync always # always là sau mỗi lần thực thi, always mạnh hơn cái everysec và tùy chọn always sẽ ảnh hưởng nghiêm trọng đến hiệu năng, bởi vì nó buộc redis phải ghi xuống liên tục, thậm trí còn nhanh hơn cả second, vì vậy dùng everysec sẽ đảm bảo hiệu năng tốt hơn, nhưng vẫn có khả năng mất mát dữ liệu trong trương hợp hệ thống gặp sự cố, vì vậy nếu hệ thống yêu cầu độ tin cậy cao và không chấp nhận mất mát dữ liệu thì sử dụng always
# appendfsync no
```
