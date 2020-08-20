# BASIC AUTH

Prometheus không hỗ trợ trực tiếp `basic authentication` (còn gọi là `basic auth`) cho các kết nối từ trình duyệt cho Prometheus và API HTTP. Nếu muốn thực thi `basic auth` cho các kết nối đó, một đề xuất là sử dụng Prometheus kết hợp với `reverse proxy` và áp dụng xác thực ở lớp proxy.
Có thể sử dụng bất kỳ reverse proxy nào tùy thích với Prometheus.

## 1. Nginx server
Giả sử nếu muốn chạy một phiên bản Prometheus đằng sau một `nginx server` đang chạy trên `localhost:12321` và để tất cả các endpoint Prometheus có sẵn thông qua `/prometheus`. Do đó, URL đầy đủ cho Prometheus `/ metrics` sẽ là:
```
	http://localhost:12321/prometheus/metrics
```

Giả sử rằng mục địch là yêu cầu username và password từ tất cả người dùng truy cập vào Prometheus. TRong ví dụ này, `admin` được sử dụng làm username và chọn bất kỳ password nào tùy thích.

Việc đầu tiên là tạo một tệp `.htpasswd` để lưu username/password nhờ công cụ `htpasswd` và lưu trữ nó trong thư mục `/etc/nginx`:
```
	mkdir -p /etc/nginx
	htpasswd -c /etc/nginx/.htpasswd admin
```
với command `htpasswd -c /etc/nginx/.htpasswd admin`, một option xuất hiện cho phép nhập password cho username `admin` và được lưu trong thư mục `.htpasswd`

Một ví dụ về tệp cấu hình `nginx.conf` (được lưu trữ tại `/etc/nginx/.htpasswd`). Với cấu hình này, nginx sẽ thực thi basic auth với tất cả các kết nối đến điểm cuối `/prometheus` (proxy tới Prometheus):

![](https://user-images.githubusercontent.com/61723456/89814592-8d7e3c00-db6d-11ea-8217-cd9632973eda.png)

Start `nginx` bằng cách sử dụng cấu hình từ trên:
```
	nginx -c /etc/nginx/nginx.conf
```

Sau mỗi lần tác động vào file config, phải thực hiện restart nginx server để nginx nhận cấu hình mới:
```
	sudo systemctl restart nginx
```

### Testing
Sử dụng `cURL` để tương tác với thiết lập nginx/Prometheus tại local:
```
	curl --head http://localhost:12321/prometheus/graph
```

Thao tác này sẽ trả về phản hồi `401 Unauthorized` vì chưa cung cấp được username và password hợp lệ. Phản hồi cũng sẽ chứa tiêu đề `WWW-Authenticate: Basic realm="Prometheus"` do nginx cung cấp, cho biết rằng vùng xác thực cơ bản của Prometheus, được chỉ định bởi tham số `auth_basic` cho nginx, được thực thi.

Để truy cập thành công các điểm cuối Prometheus bằng cách sử dụng basic auth, ví dụ như `/metrics`, hãy cung cấp tên người dùng thích hợp bằng cách sử dụng cờ `-u` và cung cấp mật khẩu khi được nhắc:
```
	curl -u admin http://localhost:12321/prometheus/metrics
	Enter host password for user 'admin':
```

kết quả sau đó:

![](https://user-images.githubusercontent.com/61723456/89852207-90524e80-dbb8-11ea-97f9-e8139d3d98cd.png)

Kiểm tra hoạt động trên trình duyệt:

![](https://user-images.githubusercontent.com/61723456/89852270-b2e46780-dbb8-11ea-973c-1a6d2530d7d8.png)

## 2. Docker
Như ý tưởng sử dụng `nginx server` để thực hiện reverse proxy, trong đề xuất này, sử dụng một image dựng một container chạy song song với prometheus, thực hiện thiết lập basic auth cho các enpoint của prometheus. Nó có tên `caddy`.

Thực hiện, sử dụng docker compose để tiến hành dựng container cho prometheus, alertmanager, grafana và một số exporter cần dùng.
```
version: '2.1'

networks:
  monitor-net:
    driver: bridge

volumes:
    prometheus_data: {}
    grafana_data: {}

services:

  prometheus:
    image: prom/prometheus:v2.20.0
    container_name: prometheus
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
    restart: unless-stopped
    expose:
      - 9090
    networks:
      - monitor-net
    labels:
      org.label-schema.group: "monitoring"

  alertmanager:
    image: prom/alertmanager:v0.21.0
    container_name: alertmanager
    volumes:
      - ./alertmanager:/etc/alertmanager
    command:
      - '--config.file=/etc/alertmanager/config.yml'
      - '--storage.path=/alertmanager'
    restart: unless-stopped
    expose:
      - 9093
    networks:
      - monitor-net
    labels:
      org.label-schema.group: "monitoring"
  grafana:
    image: grafana/grafana:7.1.1
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=${ADMIN_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${ADMIN_PASSWORD:-admin}
      - GF_USERS_ALLOW_SIGN_UP=false
    restart: unless-stopped
    expose:
      - 3000
    networks:
      - monitor-net
    labels:
      org.label-schema.group: "monitoring"
....
```

Đoạn config trên là để thực hiện việc cài các thành phần cho việc dựng hệ thống giám sát đơn giản bằng cách sử dụng các container. Các container này chạy trên localhost với port lần lượt 
```
	prometheus:9090
	alertmanager:9093
	grafana:3000
```

Tiếp theo, tạo file prometheus.yml để cấu cấu hình cho prometheus thực hiện việc scrape metrics. Để nhận cảnh báo từ alertmanager, tạo file alertmanager.yml.

Thực hiện việc chính, reverse proxy. Trong file docker-compose.yml, thêm cấu hình để thiết lập thêm container có tên `caddy`. Container này sẽ làm nhiệm vụ đătj xác thực trên các cổng chạy được cấu hình trong `Caddyfile`
```
.....
 caddy:
    image: stefanprodan/caddy
    container_name: caddy
    ports:
      - "3000:3000"
      - "9090:9090"
      - "9093:9093"
    volumes:
      - ./caddy:/etc/caddy
    environment:
      - ADMIN_USER=${ADMIN_USER:-admin}
      - ADMIN_PASSWORD=${ADMIN_PASSWORD:-admin}
    restart: unless-stopped
    networks:
      - monitor-net
    labels:
      org.label-schema.group: "monitoring"
``` 

`Caddyfile`

```
:9090 {
    basicauth / {$ADMIN_USER} {$ADMIN_PASSWORD}
    proxy / prometheus:9090 {
            transparent
        }

    errors stderr
    tls off
}

:9093 {
    basicauth / {$ADMIN_USER} {$ADMIN_PASSWORD}
    proxy / alertmanager:9093 {
            transparent
        }

    errors stderr
    tls off
}
```

### Testing
Alertmanager:

![](https://user-images.githubusercontent.com/61723456/90798551-46960080-e33c-11ea-9989-c90dfe4c3aa8.png)

Prometheus:

![](https://user-images.githubusercontent.com/61723456/90798605-57df0d00-e33c-11ea-906d-b85173a15295.png)

![](https://user-images.githubusercontent.com/61723456/90798694-780ecc00-e33c-11ea-8f28-eddf89cb6277.png)

