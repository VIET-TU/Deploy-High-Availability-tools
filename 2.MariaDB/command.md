# MariaDB Galera Cluster High Availability

# Thực hiện trên cả 3 servers

Cài đặt MariaDB

Cài đặt các công cụ cần thiết

```bash
 apt install -y net-tools telnet traceroute
```

Cài đặt mariadb server và mariadb client

```bash
 apt install -y mariadb-server mariadb-client
```

## kiểm tra trạng thái

```bash
 systemctl status mariadb
 # thư mục làm việc
 ls /etc/mysql/
conf.d  debian.cnf  debian-start  mariadb.cnf  mariadb.conf.d  my.cnf  my.cnf.fallback
 # Mở file mariadb.cnf
 vi /etc/mysql/mariadb.cnf
 # Import all .cnf files from configuration directory
!includedir /etc/mysql/conf.d/ # include tất cả nội dung ở file conf.d, tức là ta tạo bất cứ file nào trong này, thì mariadb sẽ điều hiểu
!includedir /etc/mysql/mariadb.conf.d/



```

---

# Thực hiện trên `server 1`

Cấu hình mariadb galera cluster

Thêm file cấu hình galera

```bash
 vi /etc/mysql/conf.d/galera.cnf
```

[GG][https://mariadb.com/kb/en/replication-and-binary-log-system-variables/]

```ini
[mysqld] # phần này đánh dấu bắt đầu các thiết lập cấu hình cho mysql server
binlog_format=ROW # chỉ định mysql sẽ ghi các thay đổi dữ liệu vào trong binlog theo từ Row, tức là theo từng hàng thay vì ghi theo câu lệnh statement chính là `sql`, ROW là lựa chọn  phổ biến khi ta chọn galera cluster vì nó đảm bảo động bộ hóa chính xác hơn giữa các node
default-storage-engine=innodb # xác định cái innodb sẽ là hệ thống lưu trữ mặc định cho các bảng mới, hộ trợ các tính năng như là giao dịch và khóa dòng
innodb_autoinc_lock_node=2 # chỉ định rằng cách innodb sử lý khóa tự động gia tăng, giá trị 2 nghĩa là cho phép nhiều giao dịch đồng thời chèn vào bảng, lựa chọ giá trị này sẽ cải thiện hiệu suất trong cái môi trường phân tán như là galera
bind-address=0.0.0.0 # cấy hình anywhere để cho các server có thể kết nối được với nhau

wsrep_on=ON # bật galera cluster, nếu mà cấu hình này không được bật thì mysql sẽ không tham gia và được vào cluster và nó sẽ không hiểu được đây là 1 cluster

wsrep_provider=/usr/lib/galera/libgalera_smm.so # để nó xác định được cái vị trí của galera profile libary, tức là thư viện này cung cấp các chúc năng cần thiết để tích hợp mysql với galera cluster

# Cluster Configuration
wsrep_cluster_name="mariadb-cluster-viettu" # tên cluster
wsrep_cluster_address="gcomm://192.168.72.11,192.168.72.12,192.168.72.13"

# Galera Synchronization Configuration
wsrep_sst_method=rsync # phương thức này để ta đồng bộ dữ liệu

# Node Configuration
wsrep_node_address="192.168.72.11" #
wsrep_node_name="server-1"
```

---

```ini
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0

# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Cluster Configuration
wsrep_cluster_name="mariadb-cluster-viettu"
wsrep_cluster_address="gcomm://192.168.72.11,192.168.72.12,192.168.72.13"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# Node Configuration
wsrep_node_address="192.168.72.11"
wsrep_node_name="server-1"
```

# Thực hiện trên `server 2`

Cấu hình mariadb galera cluster

Thêm file cấu hình galera

```bash
 vi /etc/mysql/conf.d/galera.cnf
```

Nội dung như sau

```ini
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0

# Galera Provider Configuration

wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Cluster Configuration

wsrep_cluster_name="mariadb-cluster-viettu"
wsrep_cluster_address="gcomm://192.168.72.11,192.168.72.12,192.168.72.13"

# Galera Synchronization Configuration

wsrep_sst_method=rsync

# Node Configuration

wsrep_node_address="192.168.72.12"
wsrep_node_name="server-2"
```

# Thực hiện trên server 3

Cấu hình mariadb galera cluster

Thêm file cấu hình galera

```bash
 vi /etc/mysql/conf.d/galera.cnf
```

Nội dung như sau

```ini
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0

# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Cluster Configuration
wsrep_cluster_name="mariadb-cluster-viettu"
wsrep_cluster_address="gcomm://192.168.72.11,192.168.72.12,192.168.72.13"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# Node Configuration
wsrep_node_address="192.168.72.13"
wsrep_node_name="server-3"
```

# Thực hiện trên `server 1`

Cấu hình mariadb galera cluster

## Tắt mariadb server

```bash
 systemctl stop mariadb
```

## Khởi tạo cụm mariadb galera cluster

```bash
 galera_new_cluster
```

## Kiểm tra cụm mariadb galera cluster tạo thành công chưa (thấy số lượng 1)

```bash
 mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size'"

#  root@server-1:~# mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
# +--------------------+-------+
# | Variable_name      | Value |
# +--------------------+-------+
# | wsrep_cluster_size | 1     |
# +--------------------+-------+
# root@server-1:~#
```

==> value 1 tức là mới chỉ khởi tạo ở trên server-1 và để server-2 và 3 được áp dụng thành công, thì cần restart trên server2,3 lại mariadb thì mới áp dụng được cấu hình mới

# Thực hiện trên `server 2` và `server 3`

Cấu hình mariadb galera cluster

Khởi động lại mariadb server để áp dụng cấu hình mới join vào cụm

```bash
 systemctl restart mariadb
```

# Quay lại `server-1` kiểm tra

```bash
root@server-1:~# mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     | # Server 2 và 3 đã được join vào cụm
+--------------------+-------+
```

===> bây giờ thêm table và data để kiểm tra có được đồng bộ sang server-2 và server-3 không

## kiểm tra server-2

```bash
mysql

# root@server-2:~# mysql
# MariaDB [(none)]> show databases;
# +--------------------+
# | Database           |
# +--------------------+
# | information_schema |
# | mysql              |
# | performance_schema | # db mặc định
# | sys                |
# +--------------------+
# 4 rows in set (0.001 sec)

# MariaDB [(none)]>
```

# Thực hiện trên server 1

Kiểm tra tính đồng bộ dữ liệu

Truy cập vào môi trường mariadb

```bash
 mysql
```

## Thêm dữ liệu

```sql
MariaDB [(None)]> create database viettudb;
MariaDB [(None)]> use viettudb;
MariaDB [viettudb]> create table series (Name VARCHAR(50) PRIMARY KEY, Lever VARCHAR(25), Release_date DATE);
MariaDB [viettudb]> insert into series (Name, Lever, Release_date) VALUES ('DevOps for Freshers', 'Basic/Freshers', '2024-01-15'), ('Xây dựng quy trình pipeline DevSecOps', 'Thực tế chuyên sâu', '2024-06-16'), ('HA tools', 'Advanced/Thực tế', '2024-08-18');
```

# Thực hiện trên `server 2` hoặc `server 3`

Kiểm tra tính đồng bộ dữ liệu

Truy cập vào môi trường mariadb

```bash
 mysql
```

Kiểm tra dữ liệu

```sql
MariaDB [(None)]> use viettudb;
MariaDB [viettudb]> show databases;
MariaDB [viettudb]> show tables;
MariaDB [viettudb]> select * from series;
```

## Đồng bộ thành công trên server-2

```bash
MariaDB [viettudb]> use viettudb;
Database changed
MariaDB [viettudb]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| viettudb           |
+--------------------+
5 rows in set (0.001 sec)

MariaDB [viettudb]> show tables;
+--------------------+
| Tables_in_viettudb |
+--------------------+
| series             |
+--------------------+
1 row in set (0.000 sec)

MariaDB [viettudb]> select * from series;
+-------------------------------------------+--------------------------+--------------+
| Name                                      | Lever                    | Release_date |
+-------------------------------------------+--------------------------+--------------+
| DevOps for Freshers                       | Basic/Freshers           | 2024-01-15   |
| HA tools                                  | Advanced/Thực tế         | 2024-08-18   |
| Xây dựng quy trình pipeline DevSecOps     | Thực tế chuyên sâu       | 2024-06-16   |
+-------------------------------------------+--------------------------+--------------+
3 rows in set (0.000 sec)

MariaDB [viettudb]>
```

==> làm sao để qua sát được trạng thái của cụm như là monitoring, rồi loadbalncing, failover

===> Sử dụng `MariaDB MaxScale` là một proxy database mã nguồn mở được phát triển bởi tập đoàn mariadb, được thiết kể để làm việc với các cơ sở dữ liệu mariadb và mysql, bản chất 2 cơ sở dữ liệu này là 1, `MaxScale` có thể là HA, Scaling và Security, failover tự động chuyển hướng các master khi mà một master này gặp lỗi thì nó sẽ tự đề cử một `Slave` khác nên làm master

# Thực hiện trên cả `3 servers`

Cài đặt Maxscale

## Cài đặt maxscale từ nguồn chính thức

```bash
 wget https://dlm.mariadb.com/2344079/MaxScale/6.4.1/packages/ubuntu/jammy/x86_64/maxscale-6.4.1-1.ubuntu.jammy.x86_64.deb
 dpkg -i --force-all maxscale-6.4.1-1.ubuntu.jammy.x86_64.deb
```

## kiểm tra

```bash
systemctl status maxscale

# root@server-1:~# systemctl status maxscale
# ○ maxscale.service - MariaDB MaxScale Database Proxy
#      Loaded: loaded (/usr/lib/systemd/system/maxscale.service; enabled; preset: enabled)
#      Active: inactive (dead)
# root@server-1:~#
```

==> chỉnh sửa cấu hình của nó để kết nối được tới mariadb cluster

```bash
# file cấu hình .cnf
root@server-1:~# ls /etc/maxscale.
# maxscale.cnf           maxscale.cnf.d/        maxscale.cnf.template  maxscale.modules.d/

#  Nên backup trước, để đảm bảo rằng ta có cấu hình sai thì ta vẫn hoàn toàn có  một phương án để khôi phục lại những cái cấu hình ban đầu hoặc là cấu hình của phiên bản trước
# thực hiện trên cả 3 server
cp /etc/maxscale.cnf  /etc/maxscale.cnf.org

```

## cầu hình đầy đủ 3 server

```bash
# chỉnh sủa
root@server-1:~# vi /etc/maxscale.cnf
```

---

```ini
[server1]
type=server
address=192.168.72.11
port=3306
protocol=MariaDBBackend

[server2]
type=server
address=192.168.72.12
port=3306
protocol=MariaDBBackend

[server3]
type=server
address=192.168.72.13
port=3306
protocol=MariaDBBackend
```

==> Xét tài khoản mật khẩu và chút nữa cũng cần tạo tài khoản trong mariadb thì như vậy `maxscale` mới có thể truy cập được vào trong database và tiến hành monitoring và replica cũng như đồng bộ dữ liệu và thực hiện các truy vấn nâng cao

```ini
[MariaDB-Monitor]
type=monitor
module=mariadbmon
servers=server1,server2,server3
user=maxscale
password=viettu
monitor_interval=2000
```

```ini
[Read-Only-Service]
type=service
router=readconnroute
servers=server1 # không insert được dữ liệu
user=maxscale
password=viettu
router_options=slave
```

```ini
[Read-Write-Service]
type=service
router=readwritesplit
servers=server1,server2,server3 # bởi vì khi server1 bị hỏng thì 2 server còn lại sẽ được đề cử lên làm master, vì thế nó có quyền cả đọc và ghi
user=maxscale
password=viettu
```

**lưu ý:** khi ta áp dụng vào trong doanh nghiệp, ta sẽ cần có rất nhiều tầng như firewall, proxy, ta sẽ cần phải mở chính xác những port của công cụ này

```ini
[Read-Only-Listener]
type=listener
service=Read-Only-Service
protocol=MariaDBClient
port=4008

[Read-Write-Listener]
type=listener
service=Read-Write-Service
protocol=MariaDBClient
port=4006
```

==> Nếu bây giờ khỏi động chắc chắn sẽ lỗi

```bash
root@server-1:~# netstat -tlpun
# Active Internet connections (only servers)
# Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
# tcp        0      0 0.0.0.0:4567            0.0.0.0:*               LISTEN      29263/mariadbd
# tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      29263/mariadbd
# tcp        0      0 127.0.0.54:53           0.0.0.0:*               LISTEN      11656/systemd-resol
# tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      11656/systemd-resol
# tcp6       0      0 :::22                   :::*                    LISTEN      1/systemd
# udp        0      0 127.0.0.54:53           0.0.0.0:*                           11656/systemd-resol
# udp        0      0 127.0.0.53:53           0.0.0.0:*                           11656/systemd-resol
# root@server-1:~#

# ==> mariadb vẫn đang chạy localhost 127.0.0.1 3 chỉnh sửa chạy anywhere

```

## làm trên cả 3 server

```bash
root@server-1:~# vi /etc/mysql/mariadb.conf.d/50-server.cnf

# sửa thành
bind-address            = 0.0.0.0

## restart lại mariadb trên cả 3 server
systemctl restart mariadb
```

==> Tạo một user trong mariadb mà ta đã định ở trong cấu hình của maxscale

# Thực hiện trên `server 1`

Tạo user maxscale trong mariadb

Truy cập vào môi trường mariadb

```bash
 mysql
```

Các câu lệnh tạo user

```sql
MariaDB [(None)]> CREATE USER 'maxscale'@'%' IDENTIFIED BY 'viettu'; -- '@'%' định nghĩa của của tất cả ip, giả sử chỗ % ta định nghĩa chính xác cái địa chỉ ip nào đó đi, thi ta chỉ có thể truy cập USER đó cho cái địa chỉ đó thôi
MariaDB [(None)]> GRANT REPLICATION CLIENT ON *.* TO 'maxscale'@'%'; -- gán quyền, quyền này sẽ cho phép người dùng truy cập thông tin về trạng thái của các máy chủ cơ sở dữ liệu, chẳng hanh các bảng sao, và các hoạt đông sao chép
MariaDB [(None)]> GRANT REPLICATION SLAVE ADMIN ON *.* TO 'maxscale'@'%'; -- thiết lập các sao chép và thực hiện các câu lệnh liên quan đến việc cập nhật cấu hình và điểu khiển các máy chủ sao chép
MariaDB [(None)]> GRANT REPLICA MONITOR ON *.* TO 'maxscale'@'%'; -- cho người dùng xem thông tin về các máy chủ và trạng thái của các server, thì nó cx chỉ liên quan đển user `maxscale` mà ta đang định nghĩa trong cấu hình
MariaDB [(None)]> FLUSH PRIVILEGES; -- chạy lệnh áp dụng gán quyền
```

## tiếp theo exit ra bên ngoài và start maxscale lên

```bash
root@server-1:~# systemctl start maxscale
root@server-1:~# maxctrl list servers
┌─────────┬───────────────┬──────┬─────────────┬─────────────────┬──────┐
│ Server  │ Address       │ Port │ Connections │ State           │ GTID │
├─────────┼───────────────┼──────┼─────────────┼─────────────────┼──────┤
│ server1 │ 192.168.72.11 │ 3306 │ 0           │ Master, Running │      │
├─────────┼───────────────┼──────┼─────────────┼─────────────────┼──────┤
│ server2 │ 192.168.72.12 │ 3306 │ 0           │ Running         │      │
├─────────┼───────────────┼──────┼─────────────┼─────────────────┼──────┤
│ server3 │ 192.168.72.13 │ 3306 │ 0           │ Running         │      │
└─────────┴───────────────┴──────┴─────────────┴─────────────────┴──────┘

```

==> dõ dàng ta có thể thấy một cụm có 3 server với `server1` hiện tại đang là `Master`
==> tương ứng như vậy ta cần copy cấu hình `maxscale` ở bên `server1` này sang `server2` và `server3`
==> Và ta không cần phải làm các bước như là tạo user và gán quyền nữa bởi vì dữ liệu đã được đồng bộ sang rồi

```bash
root@server-1:~# cat /etc/maxscale.cnf

# MaxScale documentation:
# https://mariadb.com/kb/en/mariadb-maxscale-6/

# Global parameters
#
# Complete list of configuration options:
# https://mariadb.com/kb/en/mariadb-maxscale-6-mariadb-maxscale-configuration-guide/

[maxscale]
threads=auto

# Server definitions
#
# Set the address of the server to the network
# address of a MariaDB server.
#

[server1]
type=server
address=192.168.72.11
port=3306
protocol=MariaDBBackend

[server2]
type=server
address=192.168.72.12
port=3306
protocol=MariaDBBackend

[server3]
type=server
address=192.168.72.13
port=3306
protocol=MariaDBBackend

# Monitor for the servers
#
# This will keep MaxScale aware of the state of the servers.
# MariaDB Monitor documentation:
# https://mariadb.com/kb/en/maxscale-6-monitors/

[MariaDB-Monitor]
type=monitor
module=mariadbmon
servers=server1,server2,server3
user=maxscale
password=viettu
monitor_interval=2000

# Service definitions
#
# Service Definition for a read-only service and
# a read/write splitting service.
#

# ReadConnRoute documentation:
# https://mariadb.com/kb/en/mariadb-maxscale-6-readconnroute/

[Read-Only-Service]
type=service
router=readconnroute
servers=server1
user=maxscale
password=viettu
router_options=slave

# ReadWriteSplit documentation:
# https://mariadb.com/kb/en/mariadb-maxscale-6-readwritesplit/

[Read-Write-Service]
type=service
router=readwritesplit
servers=server1,server2,server3
user=maxscale
password=viettu

# Listener definitions for the services
#
# These listeners represent the ports the
# services will listen on.
#

[Read-Only-Listener]
type=listener
service=Read-Only-Service
protocol=MariaDBClient
port=4008

[Read-Write-Listener]
type=listener
service=Read-Write-Service
protocol=MariaDBClient
port=4006
root@server-1:~#
```

## tiền hành backup server2 và server3

```bash
root@server-2:~# mv  /etc/maxscale.cnf /etc/maxscale.cnf.org
root@server-3:~# mv /etc/maxscale.cnf /etc/maxscale.cnf.org
```

# Trên server2 và server3

```bash
vi /etc/maxscale.cnf
```

```ini
# MaxScale documentation:
# https://mariadb.com/kb/en/mariadb-maxscale-6/

# Global parameters
#
# Complete list of configuration options:
# https://mariadb.com/kb/en/mariadb-maxscale-6-mariadb-maxscale-configuration-guide/

[maxscale]
threads=auto

# Server definitions
#
# Set the address of the server to the network
# address of a MariaDB server.
#

[server1]
type=server
address=192.168.72.11
port=3306
protocol=MariaDBBackend

[server2]
type=server
address=192.168.72.12
port=3306
protocol=MariaDBBackend

[server3]
type=server
address=192.168.72.13
port=3306
protocol=MariaDBBackend

# Monitor for the servers
#
# This will keep MaxScale aware of the state of the servers.
# MariaDB Monitor documentation:
# https://mariadb.com/kb/en/maxscale-6-monitors/

[MariaDB-Monitor]
type=monitor
module=mariadbmon
servers=server1,server2,server3
user=maxscale
password=viettu
monitor_interval=2000

# Service definitions
#
# Service Definition for a read-only service and
# a read/write splitting service.
#

# ReadConnRoute documentation:
# https://mariadb.com/kb/en/mariadb-maxscale-6-readconnroute/

[Read-Only-Service]
type=service
router=readconnroute
servers=server1
user=maxscale
password=viettu
router_options=slave

# ReadWriteSplit documentation:
# https://mariadb.com/kb/en/mariadb-maxscale-6-readwritesplit/

[Read-Write-Service]
type=service
router=readwritesplit
servers=server1,server2,server3
user=maxscale
password=viettu

# Listener definitions for the services
#
# These listeners represent the ports the
# services will listen on.
#

[Read-Only-Listener]
type=listener
service=Read-Only-Service
protocol=MariaDBClient
port=4008

[Read-Write-Listener]
type=listener
service=Read-Write-Service
protocol=MariaDBClient
port=4006
```

# Start maxscale trên server2 và server3

```bash
systemctl  start maxscale
# kiểm tra
root@server-2:~# maxctrl list servers
┌─────────┬───────────────┬──────┬─────────────┬─────────────────┬──────┐
│ Server  │ Address       │ Port │ Connections │ State           │ GTID │
├─────────┼───────────────┼──────┼─────────────┼─────────────────┼──────┤
│ server1 │ 192.168.72.11 │ 3306 │ 0           │ Master, Running │      │
├─────────┼───────────────┼──────┼─────────────┼─────────────────┼──────┤
│ server2 │ 192.168.72.12 │ 3306 │ 0           │ Running         │      │
├─────────┼───────────────┼──────┼─────────────┼─────────────────┼──────┤
│ server3 │ 192.168.72.13 │ 3306 │ 0           │ Running         │      │
└─────────┴───────────────┴──────┴─────────────┴─────────────────┴──────┘

```

==> Cả 3 server đều xem được trạng thái của cụm

# Bây giờ `server1` đang làm master ta sẽ tắt server 1 đi

```bash
reboot
```

# Qua server2 kiểm tra

```bash
root@server-2:~# maxctrl list servers
┌─────────┬───────────────┬──────┬─────────────┬─────────┬──────┐
│ Server  │ Address       │ Port │ Connections │ State   │ GTID │
├─────────┼───────────────┼──────┼─────────────┼─────────┼──────┤
│ server1 │ 192.168.72.11 │ 3306 │ 0           │ Down    │      │
├─────────┼───────────────┼──────┼─────────────┼─────────┼──────┤
│ server2 │ 192.168.72.12 │ 3306 │ 0           │ Running │      │
├─────────┼───────────────┼──────┼─────────────┼─────────┼──────┤
│ server3 │ 192.168.72.13 │ 3306 │ 0           │ Running │      │
└─────────┴───────────────┴──────┴─────────────┴─────────┴──────┘
root@server-2:~#
```

==> chờ đợi một chút để cho 2 server còn lại lên làm master

```bash
root@server-2:~# maxctrl list servers
┌─────────┬───────────────┬──────┬─────────────┬─────────────────┬──────┐
│ Server  │ Address       │ Port │ Connections │ State           │ GTID │
├─────────┼───────────────┼──────┼─────────────┼─────────────────┼──────┤
│ server1 │ 192.168.72.11 │ 3306 │ 0           │ Running         │      │
├─────────┼───────────────┼──────┼─────────────┼─────────────────┼──────┤
│ server2 │ 192.168.72.12 │ 3306 │ 0           │ Master, Running │      │
├─────────┼───────────────┼──────┼─────────────┼─────────────────┼──────┤
│ server3 │ 192.168.72.13 │ 3306 │ 0           │ Running         │      │
└─────────┴───────────────┴──────┴─────────────┴─────────────────┴──────┘
```

==> ta có thể thấy `server2` hiện tại đã được đề cử lên làm master, và `server1` khi được khởi động lên và trạng thái sẽ là running thì chính xác như vậy nó cũng sẽ được đồng bộ lại dữ liệu khi mà nó tắt đi

==> Còn các sử dụng VIP có thể áp dụng thêm
