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

Để cải thiện tính khả dụng của Promthues, người dùng thường triển khai hai hoặc nhiều Máy chủ Prometheus. Chúng có cấu hình giống hệt nhau bao gồm cấu hình `job` và cấu hình `alert`. Khi Prometheus servers bị lỗi, nó có thể đảm bảo rằng Promthues liên tục hoạt động.

![](https://user-images.githubusercontent.com/61723456/80332029-ebfc0480-8873-11ea-8b12-4b3c9dbd1b6d.png)

Đồng thời, cơ chế `cluster` cho phép gửi cảnh báo trên Alertmanager ngay cả khi Prometheus Severs khác nhau gửi cùng một cảnh báo tới Alertmanager, Alertmanager cũng có thể tự động hợp nhất các cảnh báo này vào một thông báo và gửi nó đến người nhận.

Mặc dù Alertmanager có thể xử lý nhiều báo động từ cùng một Prometheus servers cùng một lúc. Tuy nhiên, do sự tồn tại của một Alertmanager duy nhất, cấu trúc này có một điểm rủi ro rõ ràng. Khi Alertmanager failures tại một điểm duy nhất, tất cả các dịch vụ báo động tiếp theo sẽ không hợp lệ.

Như được hiển thị bên dưới, cách trực tiếp nhất là cố gắng triển khai nhiều bộ Alertmanager. Tuy nhiên, vì Alertmanager không tồn tại và không hiểu sự tồn tại của nhau, nên sẽ có một vấn đề là thông báo cảnh báo được gửi đi nhiều lần bởi các Alertmanager khác nhau nhiều lần.

![](https://user-images.githubusercontent.com/61723456/80332111-18b01c00-8874-11ea-87c6-c970a7e5b1c4.png)

Để giải quyết vấn đề này, như hình dưới đây. Alertmanager giới thiệu cơ chế `Gossip`. Cơ chế `Gossip` cung cấp một cơ chế truyền thông tin giữa nhiều `Alertmanager`. Đảm bảo rằng chỉ có một thông báo cảnh báo được gửi đến Người nhận khi nhiều Thông báo cảnh báo nhận được cùng một thông tin báo động.

![](https://user-images.githubusercontent.com/61723456/80332244-7e9ca380-8874-11ea-8d38-a9b5c36d831a.png)

Một chút về `Gossip`

`Gossip` là một giao thức được sử dụng rộng rãi trong các hệ thống phân tán để đạt được sự trao đổi thông tin và đồng bộ hóa trạng thái giữa các nút phân tán. Trạng thái đồng bộ hóa giao thức `Gossip` tương tự như `tin đồn` hoặc `lan truyền vi rút`, như hiển thị bên dưới:

![](https://user-images.githubusercontent.com/61723456/80332294-a1c75300-8874-11ea-9552-7ef54eb1fe2c.png)

## Xây dựng hệ thống
Để cho phép liên lạc giữa các nút Alertmanager, các tham số tương ứng cần được đặt khi Alertmanager bắt đầu. Các thông số chính bao gồm:
+ --cluster.listen-address string: Địa chỉ nghe dịch vụ cụm hiện tại
+ cluster.peer value: Địa chỉ dịch vụ `cluster` của các trường hợp khác được liên kết trong quá trình khởi tạo.

Xác định một node Alertmanager1, trong đó dịch vụ `Alertmanager` chạy trên cổng `9093` và địa chỉ dịch vụ `cluster` chạy trên cổng `8001`.
```
  alertmanager1:
    image: prom/alertmanager
    container_name: am1
    ports:
     - 9093:9093
    volumes:
     - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command: 
     - --config.file=/etc/alertmanager/alertmanager.yml
     - --storage.path=/alertmanager 
     - --cluster.peer=am2:8001
     - --cluster.listen-address=:8001
     - --web.listen-address=:9093
```
![](https://user-images.githubusercontent.com/61723456/80340752-6dab5c80-888b-11ea-912b-19f8e0262e19.png)

Xác định một node Alertmanager2, trong đó dịch vụ chính chạy trên cổng 9094 . Để tạo am1, am2 thành một cụm, khi am2 bắt đầu, xác định tham số `--cluster.peer` trỏ đến địa chỉ dịch vụ cluster của am1: 8001.
```
  alertmanager2:
    image: prom/alertmanager
    container_name: am2
    ports:
     - 9094:9094
    volumes:
     - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command: 
     - --config.file=/etc/alertmanager/alertmanager.yml
     - --storage.path=/alertmanager 
     - --cluster.peer=am1:8001
     - --cluster.listen-address=:8001
     - --web.listen-address=:9094
```
![](https://user-images.githubusercontent.com/61723456/80340795-81ef5980-888b-11ea-9183-c57b14fbc68d.png)

Tiếp theo, tạo hai node Prometheus. `Prom1` chạy trên cổng 9090, sử dụng `config.file` là file `promtheus1.yml`
```
  prometheus1:
    container_name: prom1
    image: prom/prometheus
    ports:
     - 9090:9090
    command:
     - --config.file=/etc/prometheus/prometheus.yml
     - --web.listen-address=0.0.0.0:9090
     - --storage.tsdb.path=/etc/prometheus 
    volumes:
     - ./prometheus1.yml:/etc/prometheus/prometheus.yml
     - ./rulefile.yml:/etc/prometheus/rulefile.yml
     - prometheus-data1:/prometheus
```
`Prom2` chạy trên cổng 9091, sử dụng `config.file` là file `promtheus2.yml`
```
  prometheus2:
    container_name: prom2
    image: prom/prometheus
    ports:
     - 9091:9091
    command:
     - --config.file=/etc/prometheus/prometheus.yml
     - --web.listen-address=0.0.0.0:9091
     - --storage.tsdb.path=/etc/prometheus 
    volumes:
     - ./prometheus2.yml:/etc/prometheus/prometheus.yml
     - ./rulefile.yml:/etc/prometheus/rulefile.yml
     - prometheus-data1:/prometheus
```
Lần lượt các file `prometheus1.yml` và `prometheus2.yml`:
```
prometheus1.yml

global:  
  scrape_interval:  15s   
scrape_configs:  
      - job_name: 'prometheus'  
        static_configs: 
             - targets: ['prometheus1:9090','prometheus2:9091']
      - job_name: 'alertmanager'  
        static_configs: 
             - targets: ['am1:9093','am2:9094']
rule_files:  
  - rulefile.yml
alerting:
  alertmanagers:
  - scheme: http
    static_configs:
    - targets:
      - alertmanager1:9093
      - alertmanager2:9094
```

```
prometheus2.yml

global:   
scrape_configs:  
      - job_name: 'prometheus'  
        static_configs: 
             - targets: ['prometheus1:9090','prometheus2:9091']
      - job_name: 'alertmanager'  
        static_configs: 
             - targets: ['am1:9093','am2:9094']
rule_files:  
  - rulefile.yml
alerting:
  alertmanagers:
  - scheme: http
    static_configs:
    - targets:
      - am1:9093
      - am2:9094
```
![](https://user-images.githubusercontent.com/61723456/80340563-0e4d4c80-888b-11ea-99b0-142a2c83cec2.png)

![](https://user-images.githubusercontent.com/61723456/80340598-1e652c00-888b-11ea-8e87-05a65687c8c3.png)

Tạo rulefile.yml cho kịch bản stop bất kỳ một node.
```
groups:
  - name: default
    rules:
    - alert: InstanceDown
      expr: up == 0
      for: 20s
      annotations:
        summary: "Instance {{ $labels.instance }} down"
        description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 20 seconds."
```
Stop node Prom2, am2:
![](https://user-images.githubusercontent.com/61723456/80340358-b0b90000-888a-11ea-8855-43be34762a82.png)

![](https://user-images.githubusercontent.com/61723456/80340409-c75f5700-888a-11ea-8a45-c06ca099f1e2.png)


Tạo file `alertmanager.yml` để cấu hình gửi cản báo tới gmail.
```
global:
route:
  receiver: 'team-Toan'
  group_by: [alertname, datacenter, app]

receivers:
  - name: 'team-Toan'
    email_configs:
    - to: 'toantd1202@gmail.com'
      from: 'toantd1202@gmail.com'
      smarthost: smtp.gmail.com:587
      auth_username: "toantd1202@gmail.com"
      auth_identity: "toantd1202@gmail.com"
      auth_password: "PASSWORD" 
```
![](https://user-images.githubusercontent.com/61723456/80340993-d5fa3e00-888b-11ea-9f2c-3294c84b4c88.png)

## Kết Luận
Để `configure` chính xác `cluster` nút Alertmanager, dịch vụ phải được cấu hình với danh sách các `peers` nằm trong `cluster`. Chế độ `cluster` này ngăn Alertmanager gửi thông báo trùng lặp.

Một vấn đề là khó có thể giữ dữ liệu đồng bộ. Thông thường các trường hợp `parallel Prometheus` không có dữ liệu giống hệt nhau. Có một vài lý do cho việc này. Một lý do là khoảng thời gian `scrape` có thể khác nhau, vì mỗi trường hợp có `clock` riêng. Một lý do khác là, trong trường hợp `failures`, một trong các nút Prometheus có thể bỏ lỡ một số dữ liệu, điều đó có nghĩa là về lâu dài không có trường hợp Prometheus nào có bộ dữ liệu hoàn chỉnh.
