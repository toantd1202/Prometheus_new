
# Cài đặt Prometheus 
+ #### Tạo file prometheus.yml

![](https://github.com/toantd1202/buoc1/blob/master/Screenshot%20from%202020-04-01%2011-19-50.png?raw=true)

Phần `global` là cấu hình chung cho tất cả các `scrape_configs`, các config bên trong mỗi job scrape sẽ được ưu tiên dùng hơn cái `global` nếu có trùng lặp. Theo cấu hình trên, prometheus sẽ có một `target` để pull `metrics` về có tên là `prometheus`, tự động `scrape` mỗi 30s tới service đang chạy ở `localhost:9090`.
+ #### Thiết lập Prometheus với docker
cấu hình để chạy prometheus lên với docker:

![](https://github.com/toantd1202/buoc1/blob/master/Screenshot%20from%202020-04-01%2011-25-00.png?raw=true)

``` 
$ sudo docker-compose up
```
![](https://github.com/toantd1202/buoc1/blob/master/Screenshot%20from%202020-04-01%2021-35-21.png?raw=true)

kiểm tra hoạt động: 127.0.0.1:9090

![](https://github.com/toantd1202/buoc1/blob/master/Screenshot%20from%202020-04-01%2021-38-04.png?raw=true)

#### Prometheus + Grafana
Chỉnh sửa, thêm code vào docker-compose.yml
```
version: '3.6'

volumes:
  grafana-data:
  prometheus-data:

services:
  prometheus:
    image: prom/prometheus:v2.12.0
    command:
      - --config.file=/etc/prometheus/prometheus.yml
    ports:
      - 9090:9090
    volumes:
      - prometheus-data:/prometheus
  grafana:
    image: grafana/grafana
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports:
      - 3000:3000
```
![](https://github.com/toantd1202/buoc1/blob/master/Screenshot%20from%202020-04-01%2022-02-53.png?raw=true)

Truy cập [http://localhost:3000](http://localhost:3000/) để vào trang dashboard của Grafana với username là `admin` và password là `admin`.

![enter image description here](https://github.com/toantd1202/buoc1/blob/master/Screenshot%20from%202020-04-01%2022-17-52.png?raw=true)


#### Cấu hình node exporter
Theo dõi các số liệu về hạ tầng, bao gồm CPU, memory, disk usage cũng như số liêu I/O, network.
+ thêm nội dung vào file prometheus.yml:
```
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: prometheus
    scrape_interval: 30s
    static_configs:
      - targets:
        - localhost:9090
  - job_name: 'node-exporter'
    static_configs:
         - targets: ['node-exporter:9100']
```
+ thêm nội dung vào file docker-compose.yml
```
version: '3.6'

volumes:
  grafana-data:
  prometheus-data: {}

services:
  prometheus:
    image: prom/prometheus:v2.12.0
    command:
      - --config.file=/etc/prometheus/prometheus.yml
    ports:
      - 9090:9090
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
  grafana:
    image: grafana/grafana
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports:
      - 3000:3000
  node-exporter:
    image: prom/node-exporter
    ports:
       - 9100:9100
```
+ docker-compose up

![](https://github.com/toantd1202/buoc1/blob/master/Screenshot%20from%202020-04-01%2023-19-21.png?raw=true)

kiểm tra thành quả:

![](https://github.com/toantd1202/buoc1/blob/master/Screenshot%20from%202020-04-02%2001-03-52.png?raw=true)

# Cấu hình Prometheus rules

Prometheus hỗ trợ hai loại rules có thể được cấu hình và sau đó được đánh giá theo các khoảng thời gian: `Recording rules` và `Alerting rules`. Để thêm các rules trong Prometheus, hãy tạo một tệp chứa các câu lệnh rule cần thiết và để Prometheus tải tệp qua trường rule_files trong Prometheus configuration. Rule files sử dụng YAML.

## Alerting rules

Quy tắc cảnh báo cho phép bạn xác định các điều kiện cảnh báo dựa trên biểu thức Prometheus và gửi thông báo tới dịch vụ bên ngoài. Bất cứ khi nào biểu thức cảnh báo dẫn đến một hoặc nhiều thành phần vectơ tại một thời điểm nhất định, cảnh báo sẽ được tính là hoạt động cho các label của các thành phần này.

ví dụ: 
```
groups:
- name: example
  rules:
  - alert: HighRequestLatency
    expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
    for: 10m
    labels:
      severity: page
    annotations:
      summary: High request latency
```
Option `for` khiến Prometheus phải chờ trong một khoảng thời gian nhất định giữa lần đầu tiên gặp một phần tử vectơ đầu ra biểu thức mới và tính một cảnh báo là bắn cho phần tử này. Trong trường hợp này, Prometheus sẽ kiểm tra xem cảnh báo có tiếp tục hoạt động trong mỗi lần đánh giá trong 10 phút trước khi bắn cảnh báo không. Các yếu tố đang hoạt động, nhưng chưa kích hoạt, đang ở trạng thái chờ xử lý.

`labels` cho phép chỉ định một bộ labels bổ sung được đính kèm với cảnh báo. Bất kỳ nhãn xung đột hiện có sẽ được ghi đè. Các giá trị nhãn có thể được tạo templated.

`annotations` chỉ định một bộ nhãn thông tin có thể được sử dụng để lưu trữ thông tin bổ sung dài hơn như alert description hoặc runbook links. Các giá trị annotation có thể được tạo templated.

  

## Templating

Prometheus hỗ trợ tạo templating trong các annotations và label alerts, cũng như trong các console pages. Templates có khả năng chạy các truy vấn đối với cơ sở dữ liệu cục bộ, lặp lại dữ liệu, sử dụng các điều kiện, dữ liệu định dạng, v.v ... Ngôn ngữ Prometheus templating dựa trên hệ thống Go templating.

Cấu trúc dữ liệu chính để xử lý dữ liệu time series là sample, được định nghĩa là:
```
type sample struct {
        Labels map[string]string
        Value  float64
}
```

  

## Recording rules

Recording rules cho phép bạn tính toán trước các biểu thức cần thiết lưu kết quả của chúng dưới dạng một time series mới. Truy vấn kết quả được tính toán trước thường sẽ nhanh hơn nhiều so với thực hiện biểu thức gốc mỗi khi cần. Điều này đặc biệt hữu ích cho dashboards, cần truy vấn cùng một biểu thức liên tục mỗi lần chúng refresh.

Recording và alerting rules tồn tại trong một nhóm quy tắc. Các quy tắc trong một group được chạy tuần tự theo một khoảng thời gian. Tên của recording và alerting rules phải là metric names hợp lệ.

+ tạo 1 rule file thưc hiện tính phần trăm memory free
![](https://github.com/toantd1202/buoc1/blob/master/Screenshot%20from%202020-03-31%2021-36-02.png?raw=true)

`record: <string>` (Tên của chuỗi thời gian để đầu ra. Phải là một metric name hợp lệ)
`expr: <string>` (Biểu thức PromQL để đánh giá. Mỗi chu kỳ đánh giá này được đánh giá tại thời điểm hiện tại và kết quả được ghi lại dưới dạng một time series mới với metric name được đưa ra bởi 'record').

+ thêm `recording rule` mới tạo vào `prometheus.yml` tại mục `rule_files`
![](https://github.com/toantd1202/buoc1/blob/master/Screenshot%20from%202020-04-01%2010-00-55.png?raw=true)

![](https://github.com/toantd1202/buoc1/blob/master/Screenshot%20from%202020-04-02%2010-09-20.png?raw=true)


