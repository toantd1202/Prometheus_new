# 1. HA
Phần này, em vẫn đang gặp 1 chút vấn đề. Em đã tạo được 2 node `Prom` và 2 node `Alertmanager`. Nhưng có điều là:
+ 2 node `Prom1` và `Prom2` nó vẫn chưa hoạt động được như ý muốn, cụ thể là, khi em vào `Prom1`, kiểm tra thì đã thấy `Prom1` `up`, nhưng `Prom2` đã `down` và ngược lại với `Prom2`. Chính vì thế mà nó bắn cảnh báo luôn, mặc dù em chưa stop node nào. ^^

Tại `Prom1`:

![](https://user-images.githubusercontent.com/61723456/80315159-0906e800-8820-11ea-9979-90793474e024.png)

![](https://user-images.githubusercontent.com/61723456/80315170-1a4ff480-8820-11ea-979d-2e5c77f9346c.png)

Tại `Prom2`:

![](https://user-images.githubusercontent.com/61723456/80315183-2dfb5b00-8820-11ea-9195-4f5379363182.png)

![](https://user-images.githubusercontent.com/61723456/80315196-3bb0e080-8820-11ea-919b-359718e96d52.png)

+ Với 2 node `Alertmanager` thì có vẻ ok, nó đã nhận được `peer`.

![](https://user-images.githubusercontent.com/61723456/80315317-f0e39880-8820-11ea-8478-0a7c83f09c0d.png)

![](https://user-images.githubusercontent.com/61723456/80315329-ffca4b00-8820-11ea-8e5b-e671c455aa21.png)

Và đây, file `docker-compose.yml` và `prometheus.yml` của em:
```
docker-compose.yml

version: '3.6'
volumes:
  prometheus-data1:

services:
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
     - ./prometheus.yml:/etc/prometheus/prometheus.yml
     - ./rulefile.yml:/etc/prometheus/rulefile.yml
     - prometheus-data1:/prometheus
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
     - ./prometheus.yml:/etc/prometheus/prometheus.yml
     - ./rulefile.yml:/etc/prometheus/rulefile.yml
     - prometheus-data1:/prometheus
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
     - ./prometheus.yml:/etc/prometheus/prometheus.yml
     - ./rulefile.yml:/etc/prometheus/rulefile.yml
     - prometheus-data1:/prometheus
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
     - --cluster.peer=172.21.0.5:8001
     - --cluster.listen-address=:8001
     - --web.listen-address=:9093
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
     - --cluster.peer=172.21.0.4:8001
     - --cluster.listen-address=:8001
     - --web.listen-address=:9094
```

```
prometheus.yml

global:
  scrape_interval: 30s
scrape_configs:
- job_name: 'node'
  static_configs:
  - targets: ['localhost:9090']
alerting:
  alertmanagers:
  - scheme: http
    static_configs:
    - targets: ['am1:9093','am2:9094']
rule_files:
  - rulefile.yml
```

```
rulefile.yml

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
Phần HA này trong `prometheus docs` viết ít quá, nên em nhảy sang xem phần `Federation` luôn. ^^

# 2. Federation
Ở phần này, theo như yêu cầu, em tạo ra 2 node `PromX` và `PromY` tương ứng trong 1 `docker-compose`.

`PromX` cấu hình target giám sát host sử dụng `node_exporter`.

+ Tạo file `prometheus1.yml` cho `PromX`
```
global:
  scrape_interval: 30s
scrape_configs:
- job_name: 'promX'
  static_configs:
  - targets: ['promX:9090']
- job_name: 'node-exporter'
  static_configs:
  - targets: ['node-exporter:9100']
```
`PromY` sử dụng `federation` để collect chỉ các metric về CPU từ `PromX`.

+ Tạo file `prometheus2.yml` cho `PromY`
```
global:
  scrape_interval: 30s
scrape_configs:
- job_name: 'promY'
  static_configs:
  - targets: ['promY:9091']
- job_name: 'federate'
  scrape_interval: 30s
  honor_labels: true
  metrics_path: /federate
  params:
    match[]:
    - '{job="node-exporter"}'
  static_configs:
  - targets: ['promX:9090']
```
+ Tạo file `docker-compose.yml`
```
version: '3.6'
volumes:
  prometheus-data:
services:
  prometheus1:
    image: prom/prometheus
    container_name: promX
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --web.listen-address=0.0.0.0:9090
      - --storage.tsdb.path=/etc/prometheus
    ports:
      - 9090:9090
    volumes:
      - ./prometheus1.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus1
  prometheus2:
    image: prom/prometheus
    container_name: promY
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --web.listen-address=0.0.0.0:9091
      - --storage.tsdb.path=/etc/prometheus
    ports:
      - 9091:9091
    volumes:
      - ./prometheus2.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
  node-exporter:
    image: prom/node-exporter
    ports:
       - 9100:9100
```
Lần lượt kiểm tra các node PromX tại `localhost:9090` và PromY tại `localhost:9091` xem các target đã đúng như yêu cầu:

+ PromX:

![](https://user-images.githubusercontent.com/61723456/80314623-bc6ddd80-881c-11ea-8c7e-b64b2ced93c4.png)

+ PromY:

![](https://user-images.githubusercontent.com/61723456/80314637-d4ddf800-881c-11ea-9105-13350d599055.png)

Em làm phần này đến đây ổn chưa ạ! ^^
