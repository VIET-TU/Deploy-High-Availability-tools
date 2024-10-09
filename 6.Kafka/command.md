# Kafka Cluster High Availability

Mô hình
![alt text](<Kafka-Diagrams_3C copy 9.png>)

- Bởi vì kafka được phát triển bằng java, nên ta cần cài đặt java

# Thực hiện trên cả 3 servers

Cài đặt Java

## Cài đặt java và net-tools (trong OS sẵn có, hoặc là cài thủ công bằng cách thêm các repo cụ thể thì các này sẽ tối ưu hơn trong thực tế vì sẽ cài được chính xác phiên bản yêu cầu cx như lưu trữ và tùy chỉnh có thể từ phiên bản, vị trí cấu hình, hay lưu log)

```bash
sudo apt-get install -y default-jre net-tools --fix-missing

# kiểm tra version
java -version
```

## Cài đặt zookeeper (trên os đã có gói zookeeper)

```bash
apt install -y zookeeperd
```
