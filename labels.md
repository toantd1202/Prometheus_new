

# Labels

+ Mỗi chuỗi thời gian được xác định duy nhất bởi metric name và các cặp tùy chọn key-value được gọi là nhãn.
+ Thay đổi bất kỳ giá trị nhãn nào, bao gồm thêm hoặc xóa nhãn, sẽ tạo ra chuỗi thời gian mới.
```
<metric name>{<label name>=<label value>, ...}
```
Ví dụ: số liệu `http_requests_total` biểu thị tất cả các điểm dữ liệu được Prometheus thu thập cho các dịch vụ hiển thị bộ đếm `http requests`. Vì có thể có nhiều dịch vụ hiển thị cùng một số liệu `http_requests_total`, các nhãn có thể được thêm vào từng điểm dữ liệu để chỉ định dịch vụ mà bộ đếm này áp dụng:
```
# Bộ đếm yêu cầu cho dịch vụ users-directory
Http_requests_total{service='users-directory'}

# Bộ đếm yêu cầu cho dịch vụ Lịch sử thanh toán
Http_requests_total{service='billing-history'}
```

+ Các nhãn có thể được kết hợp theo một số cách khác nhau bằng cách sử dụng các hàm, để đáp ứng một loạt các yếu cầu từ tất cả các dữ liệu được Prometheus thu thập.
```
# Số lượng yêu cầu GET thành công tới điểm cuối GET / users /: id của dịch vụ users-directory
Sum (http_Vquests_total {service='users-directory', method='GET', endpoint='/user/: id', status='200'})
```

+ Lọc dựa trên nhãn
Như được mô tả trong các ví dụ trên, có thể lọc một số liệu dựa trên giá trị của một trong các nhãn:
```
# Chỉ xem xét dịch vụ users-directory
Http_requests_total{service='users-thư mục}'

# Lọc các yêu cầu thành công
Http_requests_total{status='200'}
```
Prometheus coi mỗi sự kết hợp độc lập giữa nhãn và giá trị nhãn là một chuỗi thời gian khác nhau. Kết quả, nếu một nhãn có một tập hợp các giá trị có thể không bị ràng buộc, Prometheus sẽ rất khó lưu trữ tất cả các chuỗi thời gian này. Để tránh các vấn đề về hiệu suất, không nên sử dụng nhãn cho các bộ dữ liệu có số lượng thẻ cao (ví dụ: id duy nhất của khách hàng).
## Instrumentation labels
`Intrumentation labels`, như cái tên đã cho thấy, nó đến từ thiết bị của bạn. Chúng là những thứ được biết đến trong ứng dụng hoặc thư viện của bạn, chẳng hạn như loại `HTTP requests` mà nó nhận được, cơ sở dữ liệu mà nó nói đến và các chi tiết cụ thể nội bộ khác.
Các `labels` này được tạo trong ứng dụng.
ví dụ: ở đây, có thể thấy, `labelname=['path']` được định nghĩa.
```
import http.server
from prometheus_client import start_http_server, Counter
REQUESTS = Counter('hello_worlds_total',
	'Hello Worlds requested.',
	labelnames=['path'])
class MyHandler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
	REQUESTS.labels(self.path).inc()
	self.send_response(200)
	self.end_headers()
	self.wfile.write(b"Hello World")
if __name__ == "__main__":
     start_http_server(8000)
     server = http.server.HTTPServer(('localhost', 8001), MyHandler)
     server.serve_forever()
```

Nếu truy cập `http:// localhost: 8001/` và `http://localhost:8001/foo`, thì trên trang `/metrics` tại `http://localhost:8000/metrics` bạn sẽ thấy chuỗi thời gian cho mỗi đường dẫn:
```
# HELP hello_worlds_total Hello Worlds requested.
# TYPE hello_worlds_total counter
hello_worlds_total{path="/favicon.ico"} 6.0
hello_worlds_total{path="/"} 4.0
hello_worlds_total{path="/foo"} 1.0
```
Nên cẩn thận khi xác định `intrumentation labels` để tránh trường hợp các nhãn có khả năng được sử dụng làm `target labels`, chẳng hạn như `env`, `cluster`, `service`, `team`, `zone` và `region`.
Ta có thể chỉ định bất kỳ số lượng nhãn nào khi xác định số liệu và các giá trị theo cùng thứ tự trong lệnh gọi nhãn:
```
REQUESTS = Counter('hello_worlds_total',
	'Hello Worlds requested.',
	labelnames=['path', 'method'])
class MyHandler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
	REQUESTS.labels(self.path, self.command).inc()
	self.send_response(200)
	self.end_headers()
	self.wfile.write(b"Hello World")
```
Python và Go cũng cho phép cung cấp cả `label names` và `values`, mặc dù tên nhãn vẫn phải khớp với các định nghĩa trong  `metric`. Điều này có thể làm cho việc trộn lẫn thứ tự các đối số của trở nên khó khăn hơn, và đó là một rủi ro thực sự, thì có quá nhiều nhãn.

Không thể có các `label names` khác nhau cho một `metric`, các `client libraries` sẽ ngăn chặn nó. Khi làm việc với các `metrics`, điều quan trọng là phải biết nhãn nào bạn có, vì vậy ta phải biết trước `label names` khi thực hiện `instrumentation`. Nếu không biết nhãn, ta cần phải có một công cụ giám sát dựa trên `logs` cho trường hợp sử dụng cụ thể đó.

Giá trị được trả về cho bạn bởi `labels method` trong Python được gọi là một `child`. Ta có thể lưu trữ `child` này để sử dụng sau, nó đem lại thuận lợi là không phải tìm kiếm nó trong mỗi `instrumentation event`, tiết kiệm thời gian trong hiệu suất mã được gọi hàng trăm ngàn lần một giây.

## Target labels
 bạn sẽ có một nhãn `job` để chỉ ra loại sự việc bạn đang theo dõi và nhãn `instance` để xác định quy trình cụ thể.
ví dụ:
```
  - job_name: 'node'
    static_configs:
    - targets: ['192.168.1.117:9100']
      labels:
        instance: 'linux-ina'
    - targets: ['192.168.1.138:9100']
      labels:
        instance: 'linux-inb'
```
+ **Target label** phải không đổi.
Vì các `taget lable` là thông tin xác định của target, nếu thay đổi bất kỳ trong số chúng thì target đó sẽ thay đổi - cũng như với time series bạn scrape từ nó. Điều này sẽ dẫn đến tất cả các `graphs`, `alerts` và các biểu thức khác bị thay đổi.

Điều này có nghĩa là nhãn mục tiêu cần giữ nguyên kể từ khi theo dõi được thiết lập cho đến khi nó kết thúc.
- Có một số nhãn để xác định thứ bạn đang theo dõi, thông thường là nhãn `job` hoàn thành vai trò này và nên được chỉ định trong hầu hết mọi biểu thức để tránh làm việc trên các mục tiêu không liên quan.
 - Nhãn `instance` thường xác định duy nhất một target, do đó không cần thêm nhãn `host` hoặc `alias`.

`Target labels` là một đề xuất đầu tiên trong đó bạn có thể tận dụng tính đa chiều của Prometheus data model. Để làm cho chúng hữu ích nhất có thể và để tránh các sự cố xảy ra, hãy giữ các nhãn mục tiêu không đổi trong suốt vòng đời của mục tiêu và giữ số lượng `target labels` ở mức thấp nhất có thể.

## Relabel
`Relabeling` là một công cụ mạnh mẽ để tự động viết lại bộ nhãn của mục tiêu trước khi nó được scraped. Nhiều bước dán nhãn lại có thể được cấu hình cho mỗi cấu hình scrape. Chúng được áp dụng cho `label` của từng target theo thứ tự xuất hiện trong tệp cấu hình.

Ban đầu, ngoài các nhãn trên mỗi `target` được định cấu hình, nhãn `job` của mục tiêu được đặt thành giá trị `job_name` của cấu hình cạo tương ứng. Nhãn `__address__` được đặt thành `<host>:<port>`, địa chỉ của `target`. Sau khi dán nhãn lại, nhãn `instance` được đặt thành giá trị của `__address__` theo mặc định, nếu nó không được đặt trong khi dán nhãn lại. Các nhãn `__scheme__` và `__metrics_path__` được đặt thành đường dẫn sơ đồ và số liệu của target tương ứng. Nhãn `__param_<name>` được đặt thành giá trị của tham số URL được truyền đầu tiên được gọi là `<name>`.

Các nhãn bổ sung có tiền tố `__meta_` có thể có sẵn trong giai đoạn dán nhãn lại. Chúng được thiết lập bởi cơ chế khám phá dịch vụ cung cấp target và khác nhau giữa các cơ chế.

Các nhãn bắt đầu bằng `__` sẽ bị xóa khỏi bộ nhãn sau khi hoàn thành việc dán nhãn lại.

Metric relabeling được áp dụng cho các mẫu là bước cuối cùng trước khi ingestion. Nó có định dạng cấu hình và hoạt động tương tự như việc dán nhãn lại target. Metric relabeling không áp dụng cho các mốc thời gian được tạo tự động như `up`.
+ Relabeling hoạt động như sau:
  - Một danh sách các nhãn nguồn được xác định.
  - Đối với mỗi mục tiêu, các giá trị của các nhãn đó được nối với một dấu phân cách.
  - Một biểu thức chính quy được khớp với chuỗi kết quả.
  - Một giá trị mới dựa trên các kết quả khớp được gán cho nhãn khác.

Nhiều `relabeling rules` có thể được xác định cho mỗi cấu hình scrape. Một ví dụ đơn giản mà ghép hai nhãn thành một:
```
relabel_configs:
- source_labels: ['label_a', 'label_b']
  separator:     ';'
  regex:         '(.*);(.*)'
  replacement:   '${1}-${2}'
  target_label:  'label_c'
```
(`<regex>`: Biểu thức chính quy mà giá trị trích xuất được khớp)

Quy tắc này biến đổi một target với bộ nhãn:
```
{
  "job": "job1",
  "label_a": "foo",
  "label_b": "bar"
}
```

Vào một target với bộ nhãn:
```
{
  "job": "job1",
  "label_a": "foo",
  "label_b": "bar",
  "label_c": "foo-bar"
}
```

+ `relabel_config` và `metric_relable_configs`
  - Prometheus cần biết những gì cần scrape, đó là qúa trình phát hiện dịch vụ và relabel_configs hướng đến. Cấu hình Relabel cho phép chọn mục tiêu bạn muốn scraped và `target label` sẽ là gì. Vì vậy, nếu bạn muốn nói " scrape máy này chứ không phải máy đó", hãy sử dụng `relabel_configs`.

  - Ngược lại, `metric_relabel_configs` được áp dụng sau khi scrape xảy ra, nhưng trước khi dữ liệu được hệ thống lưu trữ nhập vào. Vì vậy, nếu có một số `expensive metrics` mà bạn muốn loại bỏ hoặc các nhãn đến từ chính scrape (ví dụ: từ /metrics page) mà bạn muốn thao tác đó là nơi áp dụng `metric_relabel_configs`.

`Alert relabeling` được áp dụng cho các cảnh báo trước khi chúng được gửi đến `Alertmanager`. Nó có định dạng cấu hình và hoạt động tương tự như việc dán nhãn lại mục tiêu.

+ `action` mặc định cho việc `relabelling` là `replace` và đó là hoạt động được sử dụng phổ biến nhất. Bên cạnh đó cũng có những hoạt động khác được cung cấp.
 - `replace`: Kết hợp `regex` với `source_labels` được nối. Sau đó, đặt `target_label` thành `replacement`, với các tham chiếu nhóm khớp (`${1}`, `${2}`, ...) thay thế bằng giá trị của chúng. Nếu `regex` không match, không có sự thay thế nào diễn ra.
 - `Keep`: bỏ các mục tiêu mà `regex` không khớp với `source_labels`.
 - `Drop`: Thả các mục tiêu mà `regex` khớp với `source_labels`.
 - `Hashmod`: Đặt `target_label` thành `modulus` hàm băm của `source_labels` được nối.
 -`labelmap`: Kết hợp `regex` với tất cả các tên nhãn. Sau đó, sao chép các giá trị của nhãn phù hợp vào tên nhãn được thay thế bằng tham chiếu `replacement` ($ {1}, $ {2}, ...) thay thế bằng giá trị của chúng.
 - `Labeldrop`: Kết hợp `regex` với tất cả các tên nhãn. Bất kỳ nhãn phù hợp sẽ được xóa khỏi bộ nhãn.
 - `Labelkeep`: Kết hợp `regex` với tất cả các tên nhãn. Bất kỳ nhãn nào không khớp sẽ bị xóa khỏi bộ nhãn.
