
# AlertManager
## 1. Alerting

![download](https://user-images.githubusercontent.com/61723456/79052492-10939200-7c61-11ea-8367-147b8a1091c7.jpg)

Cảnh báo với `Prometheus` được tách thành hai phần.
+ Khi các `Alerting rules` trong các máy chủ `Prometheus` được thỏa mãn, thông báo sẽ được gửi đến `Alertmanager`.
+ `Alertmanager` sau đó quản lý các cảnh báo đó, bao gồm `silencing` (im lặng), `inhibition` (ức chế), `aggregation` (tổng hợp) và gửi thông báo qua các phương thức như email, hệ thống thông báo cuộc gọi hoặc nền chat platforms.

Các bước chính để thiết lập cảnh báo và thông báo là:
+ Cài đặt và cấu hình `Alertmanager`
+ Cấu hình `Prometheus` để giao tiếp với `Alertmanager`
+ Tạo quy tắc cảnh báo trong `Prometheus`

## 2. Alertmanager
`Alertmanager` xử lý các cảnh báo được gửi bởi các ứng dụng khách như `Prometheus server`. Nó đảm nhiệm việc sao chép, nhóm và định tuyến chúng đến chính xác các máy thu như `email`. Nó cũng quan tâm đến `silencing` và `inhibition` cảnh báo.

**Grouping**

`Grouping` phân loại cảnh báo có tính chất tương tự thành một thông báo. Nó đặc biệt hữu ích trong thời gian ngừng hoạt động dài, khi nhiều hệ thống bị lỗi cùng một lúc và hàng trăm đến hàng nghìn cảnh báo có thể được kích hoạt đồng thời.
Ví dụ: Một hệ thống với nhiều server mất kết nối đến cơ sở dữ liệu, thay vì rất nhiều cảnh báo được gửi về `Alertmanager` thì `Grouping` giúp cho việc giảm số lượng cảnh báo trùng lặp, bằng một cảnh báo để chúng ta có thể biết được chuyện gì đang xảy ra với hệ thống đó.

**Inhibition**

`inhibition` là một khái niệm ngăn chặn thông báo cho một số cảnh báo nhất định nếu một số cảnh báo khác đã được kích hoạt.

Ví dụ: Một cảnh báo được kích hoạt thông báo rằng toàn bộ cụm không thể truy cập được. `Alertmanager` có thể được cấu hình để tắt tất cả các cảnh báo khác liên quan đến cụm này nếu cảnh báo cụ thể đó được kích hoạt. Điều này ngăn thông báo cho hàng trăm hoặc hàng ngàn cảnh báo đến mà không liên quan đến vấn đề thực tế.

**Silences**

`Silence` là một cách đơn giản để tắt cảnh báo trong một thời gian nhất định. Nó được cấu hình dựa trên việc khớp với các điều kiện thì sẽ không có cảnh báo nào được gửi khi đó.

## 3. Cài đặt
+ Xử lý file `docker-compose.yml`

```
services:
  ...
  alertmanager:
     image: prom/alertmanager
     volumes:
       - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
     command:
       - --config.file=/etc/alertmanager/alertmanager.yml
     ports:
       - 9093:9093
```

+ Tạo `rule_files` để xác định điều kiện đưa ra cảnh báo. Ơ đây, em tạo một trường hợp đơn giản là đưa ra cảnh báo khi có một container down.

```
alert_file.yml

groups: 
  - name: custom_rules 
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m

```

+  Cấu hính để `Prometheus` trỏ đến `Alertmanager`.

```
prometheus.yml
...
alerting:
  alertmanagers:
  - scheme: http
  - static_configs:
    - targets: ['alertmanager:9093']
 rule_files:
    - “alert_file.yml”
...
```
Kiểm tra hoạt động:
+ Prometheus:

![Screenshot from 2020-04-11 23-32-05](https://user-images.githubusercontent.com/61723456/79052536-5fd9c280-7c61-11ea-8973-ee37e9de15fe.png)

Hiện tại, có 3 container đang chạy là: `node-exporter`, `prometheus`,  `cadvisor`.
Tắt một container để kiểm tra tính nằng `alerts` của prometheus. Ơ đây, em chọn tắt `node-exporter`. Sau khoảng 6 phút thì `alerts` lần lượt chuyển từ `Inactive` sang `Pending` và cuối cùng là `Firing`:

![Screenshot from 2020-04-11 23-44-10](https://user-images.githubusercontent.com/61723456/79052588-d4146600-7c61-11ea-884c-bb2575096501.png)

Chuyển tới kiểm tra hoạt động của `Alertmanager` xem đã nhận được cảnh báo chưa:
đã thấy nhận được cảnh báo `node-exporter:9100` đã bị down. Tới đây thì việc cài đặt `Alertmanager` cho `Prometheus` cơ bản đã hoàn thành. 

![Screenshot from 2020-04-11 23-45-55](https://user-images.githubusercontent.com/61723456/79052601-eb535380-7c61-11ea-8d72-f19bef9e8970.png)

Tiếp theo, để gửi các thông báo đến cho người dùng.
+ File cấu hình kênh nhận thông báo
Các kiểu dữ liệu:
  - `<duration>`: a duration matching the regular expression `[0-9]+(ms|[smhdwy])`
  - `<labelname>`: a string matching the regular expression `[a-zA-Z_][a-zA-Z0-9_]*`
  - `<labelvalue>`: a string of unicode characters
  - `<filepath>`: a valid path in the current working directory
  - `<boolean>`: a boolean that can take the values true or false
  - `<string>`: a regular string
  - `<secret>`: a regular string that is a secret, such as a password
  - `<tmpl_string>`: a string which is template-expanded before usage
  - `<tmpl_secret>`: a string which is template-expanded before usage that is a secret

+ file config cảnh báo:
```
global:
# The smarthost and SMTP sender used for mail notifications.
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'sender@gmail.com'
  smtp_auth_username: 'sender@gmail.com'
  smtp_auth_password: 'abcxyz@123'
#route default
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 3h 
  receiver: default
#route-child
  routes:
  - match:
      severity: warning
    receiver: gmail
  - match:
      severity: critical
    receiver: slack
  - match_re:
      service: ^(foo1|foo2|baz)$
    receiver: slack

# Inhibition rules allow to mute a set of alerts given that another alert is
# firing.
# We use this to mute any warning-level notifications if the same alert is 
# already critical.
inhibit_rules:
- source_match:
    severity: 'critical'
  target_match:
    severity: 'warning'
  # Apply inhibition if the alertname is the same.
  equal: ['alertname']

#receiver default
receivers:
- name: 'default'
  email_configs:
  - to: 'sysadmin1@gmail.com, sysadmin2@gmail.com'
  slack_configs:
  - send_resolved: true
    username: 'monitor'
    channel: '#default'
    api_url: 'https://hooks.slack.com/services/xxxxxxxxxxx/xxxxxxxxxxx/xxxxxxxxxx'
#receiver gmail
- name: 'gmail'
  email_configs:
  - to: 'man1@gmail.com'
#receiver man2
- name: 'slack'
  slack_configs:
    - send_resolved: true
      username: 'monitor'
      channel: '#general'
      api_url: 'https://hooks.slack.com/services/xxxxxxxxxxx/xxxxxxxxxxx/xxxxxxxxxx'

```
+ **Phần global**: là nơi khai báo giá trị các biến toàn cục được sử dụng trong file cấu hình này.
```
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'sender@gmail.com'
  smtp_auth_username: 'sender@gmail.com'
  smtp_auth_password: 'abcxyz@123'
```

Cụ thể ở đây là để cấu hình những thông tin cần thiết để có thể gửi cảnh báo đến 1 hộp thư gmail.

+ **Phần route**: Là nơi cấu hình các thông tin đường đi mặc định. Tức là mặc định các cảnh báo sẽ được gưỉ theo đường này nếu không cấu hình các đường đi con khác.
```
#route default
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 30s
  repeat_interval: 2m 
  receiver: default
```
  + Trong đó:
    - group_by: Dòng này có ý nghĩa prometheus sẽ gom những thông báo có cùng `alertname` thành 1  và chỉ gửi duy nhất 1 thông báo đó-thông báo này sẽ có chứa những thông báo riêng rẽ.
    - group_wait: Sau khi một cảnh báo được taọ ra. Phải đợi khoảng thời gian này thì cảnh báo mới được gửi đi. 
    - group_interval: Sau khi cảnh báo đầu tiên gửi đi, phải đợi 1 khoảng thời gian được cấu hình ở đây thì các cảnh báo sau mới được gửi đi. 
    - repeat_interval: 3h: Sau khi cảnh báo được gửi đi thành công. Sau khoảng thời gian này, nếu vấn đề vẫn còn tồn tại, prometheus sẽ tiếp tục gửi đi cảnh báo sau khoảng thời gian này.

+ **Phần routes**: Là nơi cấu hình các đường đi con. Prometheus sẽ dựa vào labels để chọn ra đường đi. Chúng ta có thể khai báo labels với tền đầy đủ hoặc sử dụng `regular expression`.
```
 routes:
  - match:
      severity: warning
    receiver: gmail
  - match:
      severity: critical
    receiver: slack
```

Nếu thông báo có nhãn `severity` với giá trị `warning` thì sẽ gửi đến đường đi gmail. Nếu thông báo có nhãn`severity` với giá trị là `critical` thì sẽ gửi đến đường đi slack.
Ngoài ra, ta có thể sử dụng `regular expression` để match các labels, để từ đó tìm ra đường đi.
```  
  - match_re:
      service: ^(foo1|foo2|baz)$
    receiver: slack
```

Các labels là foo1 hoặc foo2 hoặc baz sẽ được gửi đến slack.

+ **Inhinition**: Có nghĩa là khi 1 cảnh báo được gửi đi, thì các cảnh báo phụ khác không cần phải gửi đi nữa.

```
inhibit_rules:
- source_match:
    severity: 'critical'
  target_match:
    severity: 'warning'
  # Apply inhibition if the alertname is the same.
  equal: ['alertname']
```

Ví dụ: Khi cảnh báo có nhãn `critical` được gửi đi, thì các cảnh báo `warning` không cần phải gửi đi nữa, áp dụng với các cảnh báo có cùng `alertname`.

+ **Phần receivers**: Là nơi sẽ cấu hình các thông tin nơi nhận.
```
receivers:
- name: 'default'
  email_configs:
  - to: 'sysadmin1@gmail.com, sysadmin2@gmail.com'
  slack_configs:
  - send_resolved: true
    username: 'monitor'
    channel: '#default'
    api_url: 'https://hooks.slack.com/services/xxxxxxxxxxx/xxxxxxxxxxx/xxxxxxxxxx'
```
Nơi nhận mặc đinh: Thông báo cùng lúc sẽ được gửi đến các địa chỉ `sysadmin1@gmail.com`, `sysadmin2@gmail.com` và channel default của kênh slack.
Ngoài ra ta có thể cấu hình bổ sung thêm các đường đi khác.
```
#receiver gmail
- name: 'gmail'
  email_configs:
  - to: 'man1@gmail.com'

#receiver man2
- name: 'slack'
  slack_configs:
    - send_resolved: true
      username: 'monitor'
      channel: '#general'
      api_url: 'https://hooks.slack.com/services/xxxxxxxxxxx/xxxxxxxxxxx/xxxxxxxxxx'
```
+ Từ cấu hình định tuyến có thể giải quyết các vấn đề:
  - Gửi cảnh báo cùng lúc đến nhiều nơi.
  - Sau khi đẩy cảnh báo, nếu vẫn còn tồn tại vấn đề, sau một khoảng thời gian có thể tiếp tục đẩy cảnh báo đến người khác.
  - Phân mức cảnh báo, gửi đến các đối tượng khác nhau: Việc phân mức cảnh báo sẽ là do mình tự đặt theo nhãn chứ không có sẵn các mức cảnh báo. Ví dụ như với cảnh báo A, thì có nhãn là `warning`, cảnh báo B sẽ có nhãn là `critical`. Thì trong phần cấu hình `route `đường đi của cảnh báo, mình sẽ dùng tùy chọn `match`, lọc ra các labels nào sẽ đi đường nào,có hỗ trợ match bằng cách dùng `regular expression`.
  - Tính năng im lặng, không gửi cảnh báo nào trong 1 khoảng thời gian. Có hỗ trợ, cấu hình trên giao diện web của alertmanager. (Silences)
  - Tính năng khi 1 cảnh báo được gửi đi thì có thể các cảnh báo khác không cần phải gửi đi nữa. (Inhibition)


+ Tạo file alertmanager.yml để thiết lập hoạt động gửi tin nhắn.
Cụ thể, gửi thông báo tới gmail và slack.
```
global:

    slack_api_url: 'https://hooks.slack.com/services/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'

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
      auth_password: "password"   
    slack_configs:
      - channel: 'toantd1202'
```

+ Vào Gmail xem có thư báo không nào:

![](https://user-images.githubusercontent.com/61723456/79052690-8815f100-7c62-11ea-831e-07e06b592b35.png)

+ Vào Slack kiểm tra thông báo:

![](https://user-images.githubusercontent.com/61723456/79052695-9401b300-7c62-11ea-8ea1-928d1a053295.png)

