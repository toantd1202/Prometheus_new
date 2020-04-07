# Giới thiệu cơ bản về Prometheus
## Monitoring là gì? Khác gì với Logging và Tracing

+ `Monitoring`: là quá trình thu thập, phân tích và sử dụng thông tin có hệ thống để theo dõi tiến trình của chương trình nhằm đưa ra hướng dẫn, quyết định quản lý. Monitoring được tiến hành sau khi một chương trình đã bắt đầu và tiếp tục trong suốt thời gian thực hiện chương trình.

Các monitoring tool giúp đạt được các mục tiêu này bằng cách giám sát các số liệu như liệu ứng dụng hoặc dịch vụ có phản hồi hay không, tốc độ phản hồi và mức độ sử dụng của bộ nhớ, băng thông mạng hoặc thời gian CPU.

+ `Logging`: được tạo bởi các ứng dụng, máy chủ, cơ sở hạ tầng mạng. Logging được sử dụng để thể hiện các biến đổi trạng thái trong một ứng dụng. Khi xảy ra sự cố, chúng ta cần log để thiết lập thay đổi trạng thái gây ra lỗi. Hay nói cách khác, logging ghi lại toàn bộ hoạt động diễn ra trong từng thời điểm. Điều này đẫn đến khó khăn trong việc lưu trữ log data.

+ `Tracing`: Bằng cách truy tìm thông qua một ngăn xếp, các nhà phát triển có thể xác định các vấn đề và tập trung vào việc cải thiện hiệu suất. Khi xảy ra sự có, bạn có thể tìm hiểu và tìm ra cách để đến khắc phục sự cố đó.

Monitoring giúp quản lý hiệu suất ứng dụng, trong khi logging là để ghi lại tất cả về việc quản lý dữ liệu bên trong log files, tracing để truy đến các vấn đề của ứng dụng.

## Prometheus là gì?

Prometheus là một open-source systems monitoring và alerting ban đầu được xây dựng tại SoundCloud. Vào năm 2012 nhiều công ty, tổ chức đã đứng ra bảo trợ cho Prometheus và project này cực kỳ phát triển, có rất nhiều người dùng. Hiện tại nó không còn là một project độc lập mà được phát triển bởi rất nhiều công ty khác nhau. Nó sử dụng mã nguồn GoLang của google. Hiện tại thì Prometheus 100% là open source.

## Tính năng

+ Mô hình dữ liệu đa chiều - time series được xác định bởi tên của số liệu(metric) và các cặp key/value.

+ Ngôn ngữ truy vấn linh hoạt.

+ Hỗ trợ nhiều chế độ biểu đồ.

+ Nhiều chương trình tích hợp và hỗ trợ bởi bên thứ 3.

+ Hoạt động cảnh báo linh động dễ cấu hình.

+ Chỉ cần một máy chủ là có thể hoạt động được.

+ Hỗ trợ push các time series thông qua một gateway trung gian.

+ Các máy chủ/thiết bị giám sát có thể được phát hiện thông qua service discovery hoặc cấu hình tĩnh.

### Data model

+ Prometheus lưu trữ data dưới dạng **time series**.

+ Stream of timestamped values có cùng metric và kích thước nhãn(label).

+ Ngoài việc lưu trữ time series, Prometheus có thể tạo ra các time series.

Các `time series` được định danh duy nhất bằng `metric name` và `key-value pairs`- còn được gọi là `labels`. Key mô tả những gì bạn đang đo trong khi value lưu trữ giá trị đo thực tế, dưới dạng số.  

**Metric name** Tên số liệu chỉ định tính năng chung của hệ thống được đo. Ví dụ: http_requests_total - Tổng số yêu cầu HTTP nhận được.

**Lables** Sử dụng nhãn để phân biệt các đặc điểm của thứ đang được đo:

`api_http_requests_total -` Phân biệt các loại yêu cầu: `operation="create|update|delete"`

`api_request_duration_seconds` - Phân biệt các giai đoạn yêu cầu: `stage="extract|transform|load"`

**Sample**:  Các sample thành dữ liệu chuỗi thời gian thực tế. Mỗi mẫu bao gồm:

+ Một giá trị float64.

+ Dấu thời gian chính xác đến ms.

**Notation**: Đặt tên số liệu và bộ nhãn
```

<metric name>{<label name>=<label value>, …}

```
ví dụ: 
```
api_http_requests_total{method="POST", handler="/messages"}
```
### Metric types

Có 4 loại `metric`

**Counter**: Có giá trị là số, giá trị chỉ có thể được tăng lên chứ không thể giảm đi. Được sử dụng trong các trường hợp như đếm số request, task complete, errors occurred,..

**Gauge**: Tương tự như Counter. Tuy nhiên giá trị có thể lên hoặc xuống. Thường được sử dụng để đo các giá trị như giá trị nhiệt độ, hoặc giá trị bộ nhớ hiện tại đang sử dụng.

**Histogram**: Là metrics mà sẽ cung cấp nhiều time-series: 1 cho mỗi bucket, 1 cho tổng các giá trị và 1 cho đếm các events đã thấy. Ví dụ như thời gian response.

+ histogram_quantile(φ float, b instant-vector), φ tính toán lượng tử (0≤ φ ≤ 1), từ các buckets b của một histogram
ví dụ: Lượng tử được tính cho mỗi kết hợp nhãn trong `http_request_duration_seconds`. Để tổng hợp, sử dụng `sum()`. Vì nhãn `le` được yêu cầu bởi histogram_quantile(), nên nó phải được bao gồm trong mệnh đề `by`. Biểu thức sau đây tổng hợp 90th percentile  by job:

```

histogram_quantile(0.9, sum(rate(http_request_duration_seconds_bucket[10m])) by (job, le))

```

**Summary**: Cũng tương tự Histogram. Tuy nhiên nó có hỗ trợ thêm khả năng tính toán quantiles.

+ **φ-quantiles**  (0 ≤ φ ≤ 1) để quan sát sự kiện, <basename>{quantile="<φ>”}.

+ Tính tổng của tất cả các giá trị quan sát `<basename>_sum`.

+ Đếm các sự kiện đã được quan sát `<basename>_count`.
  
  
  **So sánh Histogram vs Summary**

|  | Histogram | Summary | 
| ------------- |:-------------:| -----:|
| Cấu hình yêu cầu | Xác định trước giới hcủa các buckets cho giá trị quan sát | Xác định trước φ-quantiles và sliding windows |
| Client performance | Các quan sát cần ít tài nguyên vì chỉ cần counters | Các quan sát tốn tài nguyên  hơn do phải stream những phép tính quantile |
|Server performance|Máy chủ phải tính toán lượng tử. Có thể sử dụng recording rules nếu quá trình tính toán đặc biệt mất quá nhiều thời gian|không tốn tài nguyên vì đã thực hiện trên máy chủ ứng dụng|
|Số time series|Một time series cho mỗi bucket được cấu hình|Nhiều time series cho mỗi lượng tử được cấu hình|
|Trường hợp sử dụng |tính trung bình hoặc phần trăm, có thể sử dụng giá trị gần đúng, xác định phạm vi của các giá trị|cần tính toán các lượng tử chính xác, khi không thể chắc chắn phạm vi của các giá trị|

### Job và Instances

+ **Instance**: một instance là 1 label, dùng để định danh duy nhất cho target(một đối tượng sẽ được Prometheus đi lấy dữ liệu) trong một job, instance : `<host>`:`<port>` (host, port có thể là địa chỉ IP và địa chỉ cổng).

+ **Job**: là một tập hợp các target chung một nhóm mục đích(vd: giám sát một nhóm các dịch vụ database,…).
ví dụ: job api-server

+ instance 1: `1.2.3.4:5670`

+ instance 2: `1.2.3.4:5671`

+ instance 3: `5.6.7.8:5670`

+ instance 4: `5.6.7.8:5671`



## Kiến trúc

![](https://github.com/toantd1202/buoc1/blob/master/Screenshot%20from%202020-03-30%2021-11-47.png?raw=true)

**Prometheus** thực hiện quá trình lấy các metric từ các job được chỉ định qua kênh trực tiếp hoặc thông qua dịch vụ Pushgateway trung gian. Sau đó nó sẽ lưu trữ dữ liệu thu thập được ở local host. Tiếp đến sẽ chạy các rule để xử lý các dữ liệu theo nhu cầu cũng như kiểm tra thực hiện các cảnh báo mong muốn.

Kênh trực tiếp là các Exporter ví dụ như:
+ cAdvisor: export các metrics của các docker service, các process trên server.
+ Node Exporter: export các metrics một node (ví dụ như là một server) như CPU, RAM của node, dung lượng ổ đĩa, số lượng request tới node đấy,...

## Các thành phần

+ Máy chủ Prometheus đảm bảo việc nhận dữ liệu và lưu trữ dữ liệu time-series.

+ Client libraries cho các ứng dụng.

+ Push Gateway Prometheus: sử dụng để hỗ trợ các job có thời gian thực hiện ngắn (tạm thời). Các tác vụ công việc này không tồn tại lâu đủ để Prometheus chủ động lấy dữ liệu. Vậy nên các metric sẽ được đẩy về Push Gateway rồi đẩy về Prometheus server.

+ Exporter hỗ trợ giám sát các dịch vụ hệ thống và gửi về Prometheus theo chuẩn Prometheus mong muốn.

+ AlertManager: dịch vụ quản lý, xử lý các cảnh báo (alert).

+ Có rất nhiều công cụ hỗ trợ khác.

## Một số thuật ngữ

+ **Time-series Data**: là chuỗi các điểm dữ liệu, thường bao gồm các phép đo liên tục được thực hiện từ cùng một nguồn trong một khoảng thời gian.

+ **Alert**: một cảnh báo là kết quả của việc đạt điều kiện thỏa mãn một rule cảnh báo được cấu hình trong Prometheus. Các cảnh báo sẽ được gửi đến dịch vụ AlertManager.

+ **Client Library**: một thư viện hỗ trợ người dùng có thể tự tùy chỉnh lập trình phương thức riêng để lấy dữ liệu từ hệ thống và đẩy dữ liệu metric về Prometheus.

+ **Endpoint**: nguồn dữ liệu của các metric mà Prometheus sẽ đi lấy thông tin.

+ **Exporter**: là một trương trình được sử dụng với mục đích thu thập, chuyển đổi các metric không ở dạng kiểu dữ liệu chuẩn Prometheus sang chuẩn dữ liệu Prometheus. Sau đó, exporter sẽ expose webservice api chứa thông tin các metric hoặc đẩy về Prometheus.

+ **Instance**: một instance là 1 label, dùng để định danh duy nhất cho target trong một job.

+ **Job**: là một tập hợp các target chung một nhóm mục đích(vd: giám sát một nhóm các dịch vụ database,…).

+ **PromQL**: là viết tắt của Prometheus Query Language, ngôn ngữ này cho phép thực hiệc các hoạt động liên quan đến dữ liệu metric.

+ **Sample**: là một giá trị đơn lẻ tại một thời điểm trong khoảng thời gian time series.

+ **Target**: định nghĩa một đối tượng sẽ được Prometheus đi lấy dữ liệu(scrape).

## Trường hợp sử dụng Prometheus

Prometheus hoạt động với việc ghi lại chuỗi thời gian hoàn toàn bằng số. Nó phù hợp với cả giám sát tập trung vào máy cũng như giám sát các kiến trúc hướng dịch vụ rất năng động. Trong microservice, sự hỗ trợ của nó cho việc thu thập và truy vấn dữ liệu đa chiều, đó là một thế mạnh đặc biệt.

Prometheus được thiết kế để đảm bảo độ tin cậy, là hệ thống sử dụng trong thời gian ngừng hoạt động để cho phép bạn chẩn đoán nhanh các sự cố. Mỗi máy chủ Prometheus là độc lập, không phụ thuộc vào lưu trữ mạng hoặc các dịch vụ từ xa khác. Bạn có thể dựa vào nó khi các phần khác trong cơ sở hạ tầng của bạn bị hỏng và bạn không cần thiết lập cơ sở hạ tầng mở rộng để sử dụng nó.

## Trường hợp không sử dụng Prometheus:

Prometheus values reliability. Bạn luôn có thể xem những số liệu thống kê có sẵn về hệ thống của bạn, ngay cả trong điều kiện thất bại. Nếu bạn cần độ chính xác 100%, chẳng hạn như thanh toán theo yêu cầu, Prometheus không phải là một lựa chọn tốt vì dữ liệu được thu thập có thể sẽ không được chi tiết và đầy đủ. Trong trường hợp như vậy, tốt nhất bạn nên sử dụng một số hệ thống khác để thu thập và phân tích dữ liệu để thanh toán và Prometheus cho phần còn lại của việc theo dõi của bạn.

