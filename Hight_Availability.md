# Hight Availability

![](https://user-images.githubusercontent.com/61723456/80203296-26398c00-8651-11ea-8c09-10e17a72ee6c.png)

Alertmanager hỗ trợ cấu hình để tạo một cụm cho `high availability`. Điều này có thể được cấu hình bằng cách sử dụng các cờ `--cluster-*`.

Điều quan trọng không phải là cân bằng tải giữa Prometheus và Alertmanager của nó, mà thay vào đó, Prometheus chỉ vào danh sách tất cả các Alertmanager.

Để tạo một cụm Alertmanager có tính sẵn sàng cao, các `instances` cần phải được cấu hình để liên lạc với nhau.
+ `--cluster.listen-address` string: địa chỉ cluster (mặc định là `0.0.0.0:9094`)
+ `--cluster.advertise-address` string: cluster advertise address
+ `--cluster.peer` value: khởi tạo peer (Lặp lại cờ cho mỗi peer bổ sung)
+ `--cluster.peer-timeout` value: Thời gian chờ peer (mặc định 15s).
+ `--cluster.gossip-interval` value: Tốc độ truyền tin nhắn cụm (mặc định 200ms)
+ `--cluster.tcp-timeout` value: Giá trị thời gian chờ cho các kết nối tcp, đọc và ghi (mặc định 10s)
+ `--cluster.reconnect-interval` value: Khoảng thời gian kết nối lại với các peer bị mất (mặc định 10s)

Cổng được chọn trong cờ `cluster.listen-address` là cổng cần được chỉ định trong cờ `cluster.peer` của các peer khác.

Cờ `cluster.advertise-address` là bắt buộc nếu instance không có địa chỉ IP.

Để `Prometheus 1.4`(hoặc phiên bản mới hơn) trỏ vào nhiều `Alertmanager`, ta định cấu hình chúng trong tệp cấu hình prometheus.yml, ví dụ:
```
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - alertmanager1:9093
      - alertmanager2:9093
      - alertmanager3:9093
```

Việc triển khai Alertmanager hy vọng tất cả các cảnh báo sẽ được gửi đến tất cả các Alertmanager để đảm bảo tính sẵn sàng cao.
