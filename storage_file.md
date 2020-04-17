
# Storage
Prometheus bao gồm một cơ sở dữ liệu với các chuỗi thời gian ở local on-disk, nhưng cũng tùy ý tích hợp với các hệ thống storage remote.

![Screenshot from 2020-04-14 21-51-51](https://user-images.githubusercontent.com/61723456/79238995-3fe11380-7e9a-11ea-8b52-8354209f47d2.png)

# Các loại lưu trữ Prometheus
## Local storage
Cơ sở dữ liệu các timeseries được Prometheus lưu trữ tại `local` theo định dạng tùy chỉnh trên các ổ đĩa (HDD/SSD).
### On-disk layout
Nhóm các mẫu được ghi vào thành các `blocks` trong hai giờ lưu dữ liệu cuối cùng. Mỗi khối này bao gồm một thư mục chứa một hoặc nhiều tệp `chunk`, `chunk` chứa tất cả các mẫu timeseries cho cửa sổ thời gian đó, cũng như `metadata file` và `index file` (indexes: các `metric names` và `labels` theo chuỗi thời gian trong các tệp `chunk`). Khi chỗi được xóa thông qua API, các bản ghi xóa sẽ được lưu trữ trong các `tombstone file` riêng biệt (thay vì xóa dữ liệu ngay lập tức khỏi các tệp `chunk`).

`block` cho các mẫu hiện đang đến, được giữ trong bộ nhớ và chưa hoàn toàn tồn tại. Nó được bảo vệ, chống lại các sự cố bởi `write-ahead-log` (WAL), có thể được phát lại khi máy chủ Prometheus khởi động lại sau một sự cố. Dữ liệu được lấy bởi `prometheus` không được flash ngay lập tức mà được đặt trong bộ nhớ. Hãy tưởng tượng nếu ngừng hoạt động trong một thời gian, dữ liệu sẽ không bị mất. Vai trò của `wal log` là làm điều này. Các tệp log được lưu trữ trong thư mục `wal` , trong các `segments` có kích thước `128MB`. Các tệp này chứa dữ liệu thô chưa được nén, vì vậy chúng lớn hơn đáng kể so với các `block file` thông thường. Prometheus sẽ giữ tối thiểu 3 tệp `wal`, tuy nhiên các máy chủ với lưu lượng truy cập cao, có thể thấy nhiều hơn ba tệp `WAL` vì nó cần giữ dữ liệu thô ít nhất hai giờ.

Cấu trúc của thư mục dữ liệu của máy chủ Prometheus sẽ trông giống như thế này:
```
./data
├── 01BKGV7JBM69T2G1BGBGM6KB12
│   └── meta.json
├── 01BKGTZQ1SYQJTR4PB43C8PD98
│   ├── chunks
│   │   └── 000001
│   ├── tombstones
│   ├── index
│   └── meta.json
├── 01BKGTZQ1HHWHV8FBJXW1Y3W0K
│   └── meta.json
├── 01BKGV7JC0RY8A6MACW02A2PJD
│   ├── chunks
│   │   └── 000001
│   ├── tombstones
│   ├── index
│   └── meta.json
└── wal
    ├── 00000002
    └── checkpoint.000001
```

+ Một hạn chế của bộ nhớ cục bộ là nó không được gộp hoặc nhân rộng.
+ Sử dụng RAID, `snapshots` để sao lưu, chứa kế hoạch, v.v., được đề xuất để cải thiện độ bền. Với độ bền lưu trữ thích hợp và lập kế hoạch lưu trữ nhiều năm dữ liệu trong `local storage` là có thể.
+ Ngoài ra, bộ nhớ ngoài có thể được sử dụng thông qua API đọc/ghi từ xa.

### Compaction
+ Các `two-hour blocks` ban đầu được nén thành các khối dài hơn trong background.
+ Việc nén sẽ tạo ra các khối lớn hơn tới 10% thời gian lưu, hoặc 31 ngày, tùy theo mức nào nhỏ hơn.

### Operational aspects (Tính toán sizing)
Prometheus có một số cờ cho phép cấu hình bộ nhớ cục bộ:
+ `--storage.tsdb.path`: Xác định nơi Prometheus viết cơ sở dữ liệu của nó. Mặc định cho `data/`.
+ `--storage.tsdb.retention.time`: Xác định khi nào cần xóa dữ liệu cũ. Mặc định là 15ngày. Ghi đè `--storage.tsdb.retention`, cờ này được đặt cho bất kỳ thứ gì ngoài các giá trị mặc định.
+ `--storage.tsdb.retention.size`: Xác định số byte tối đa mà các khối lưu trữ có thể sử dụng (lưu ý rằng điều này không bao gồm kích thước WAL). Dữ liệu cũ nhất sẽ bị xóa đầu tiên. Mặc định là 0 hoặc disabled. Cờ này là thử nghiệm và có thể được thay đổi trong phiên bản tương lai. Các đơn vị được hỗ trợ: KB, MB, GB, PB. 
ví dụ: `512MB`
+ `--storage.tsdb.wal-compression`: Cờ này cho phép nén `log WAL`. Tùy thuộc vào dữ liệu, bạn có thể mong đợi kích thước `WAL` được giảm một nửa. Lưu ý rằng nếu bật cờ này và sau đó hạ cấp Prometheus xuống phiên bản bên dưới 2.11.0, bạn sẽ cần xóa WAL vì nó sẽ không thể đọc được.

Trung bình, Prometheus chỉ sử dụng khoảng 1-2 byte cho mỗi mẫu. Do đó, để lập kế hoạch dung lượng của máy chủ Prometheus, bạn có thể sử dụng công thức:
```
needed_disk_space = retention_time_seconds * ingested_samples_per_second * bytes_per_sample
```
Để điều chỉnh tốc độ lấy mẫu trong một giây, có thể giảm số chuỗi thời gian scrape(ít target hơn hoặc ít chuỗi hơn trên mỗi target) hoặc bạn có thể tăng khoảng thời gian scrape. Tuy nhiên, việc giảm số lượng series có thể hiệu quả hơn, do nén các mẫu trong một chuỗi.

Nếu `local storage` bị hỏng vì bất kỳ lý do gì, cách tốt nhất là tắt Prometheus và xóa toàn bộ thư mục lưu trữ. Các hệ thống tệp không tuân thủ `POSIX`, không được hỗ trợ bởi `local storage` của Prometheus, các lỗi có thể xảy ra, mà không có khả năng phục hồi. Bạn có thể thử xóa các thư mục khối riêng lẻ để giải quyết vấn đề, điều này có nghĩa là mất một cửa sổ thời gian có giá trị khoảng hai giờ cho mỗi thư mục khối. Một lần nữa, `local storage` của Prometheus không có nghĩa là lưu trữ lâu dài.

(`POSIX` (Portable Operating System Interface) là 1 chuẩn hệ điều hành biến thể của HĐH Unix được định ra bởi IEEE Computer Society để duy trì tính tương thích giữa các hệ điều hành. Chuẩn POSIX định nghĩa API (Application Programming Interface), cùng với commandline shells và những giao diện hữu ích (utility interfaces) khác.)

Nếu cả chính sách duy trì time và size được chỉ định, bất kỳ chính sách nào được kích hoạt, nó sẽ được sử dụng ngay lúc đó.

Dọn dẹp `expired block` xảy ra trên một nền lịch trình. Có thể mất đến hai giờ để loại bỏ các `expired block`. Chúng phải được hết hạn trước khi chúng được dọn.

### Remote storage
`local storage` bị giới hạn bởi các nút đơn, Cho môi trường kinh doanh, điều đó là không thể thoản mãn, do đó bạn phải sử dụng lưu trữ từ xa. Thay vì cố gắng giải quyết tập trung lưu trữ trong Prometheus, nó có một bộ giao diện cho phép tích hợp với hệ thống `remote storage`.

![Screenshot from 2020-04-14 21-47-47](https://user-images.githubusercontent.com/61723456/79238502-a44fa300-7e99-11ea-99d7-c2b67c40158f.png)

`Prometheus'remote storage` cần phải được thực hiện với sự trợ giúp của một bộ chuyển đổi. Các bộ chuyển đổi sẽ cung cấp chức năng ghi URL,  đọc URL tới `prometheus`, khi `prometheus` có được các dữ liệu, nó sẽ ghi vào các `local` và sau đó ghi URL để điều khiển từ xa.
+ Prometheus có thể viết samples rằng nó tiến hành đến một `remote URL` trong một dạng chuẩn hóa.
+ Prometheus có thể đọc (lại) dữ liệu samples từ một `remote URL` trong một dạng chuẩn hóa.

`read` và `write protocol` đều sử dụng một cách linh hoạt giao thức nén đệm mã hóa HTTP. Các giao thức được coi là không ổn định như APIs, có thể thay đổi để sử dụng gRPC tốt hơn HTTP/2 trong tương lai, khi nhảy giữa Prometheus và `remote storage` có thể an toàn được giả định để hỗ trợ HTTP/2.

**Remote read**

![](https://user-images.githubusercontent.com/61723456/79238633-d234e780-7e99-11ea-9da2-a46f5a3cc4ae.png)

Khi cấu hình, Prometheus lưu trữ các truy vấn (ví dụ qua HTTP API) được gửi đến cả `local` và `remote storage`, các kết quả là đồng nhất.

Lưu ý rằng để duy trì độ tin cậy thì các vấn đề về `alerting` và `recording rule` được đánh giá là nên chỉ sử dụng các `local TSDB`.

+ `<remote_read>`
```
# URL của các endpoint để truy vấn.
url: <string>

# An optional list of equality matchers which have to be
# present in a selector to query the remote read endpoint.
required_matchers:
  [ <labelname>: <labelvalue> ... ]

# Thời gian chờ cho yêu cầu ccủa điểm remote read.
[ remote_timeout: <duration> | default = 1m ]

# Việc đọc có nên được thực hiện cho các truy vấn trong khoảng thời gian mà 
# bộ nhớ cục bộ nên có dữ liệu đầy đủ.
[ read_recent: <boolean> | default = false ]

# Đặt tiêu đề `Authorization` trên mỗi yêu cầu đọc từ xa với tên người dùng và
# mật khẩu được cấu hình. password và password_file là mutually exclusive.
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

# Đặt tiêu đề `Authorization` trên mỗi yêu cầu remote read với mã thông báo
# mang được cấu hình. Nó mutually exclusive với `bearer_token_file`.
[ bearer_token: <string> ]

# Đặt tiêu đề `Authorization` trên mỗi yêu cầu remote read với mã thông báo mang
# được đọc từ tệp được cấu hình. Nó mutually exclusive với `bearer_token`.
[ bearer_token_file: /path/to/bearer/token/file ]

# Cấu hình cài đặt yêu cầu TLS của remote read.
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]
```
**Remote write**

![](https://user-images.githubusercontent.com/61723456/79238692-e7aa1180-7e99-11ea-82a1-7325cf4a6004.png)

+ Khi cấu hình, Prometheus chuyển tiếp mẫu nó lấy được cho một hoặc nhiều xa remote storage.
+ Hàng đợi là một tập hợp các "shards" được quản lý động: tất cả các mẫu cho bất kỳ chuỗi thời gian cụ thể nào (tức là số liệu duy nhất) sẽ kết thúc trên cùng một shard.
+ Hàng đợi tự động chia tỷ lệ lên hoặc xuống số lượng phân đoạn ghi vào `remote storage` để theo kịp tốc độ dữ liệu đến.

Điều này cho phép Prometheus quản lý `remote storage` trong khi chỉ sử dụng các tài nguyên cần thiết và với cấu hình tối thiểu.
+ `<remote_write>`
```
# The URL of the endpoint to send samples to.
url: <string>

# Timeout for requests to the remote write endpoint.
[ remote_timeout: <duration> | default = 30s ]

# List of remote write relabel configurations.
write_relabel_configs:
  [ - <relabel_config> ... ]

# Sets the `Authorization` header on every remote write request with the
# configured username and password.
# password and password_file are mutually exclusive.
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

# Sets the `Authorization` header on every remote write request with
# the configured bearer token. It is mutually exclusive with `bearer_token_file`.
[ bearer_token: <string> ]

# Sets the `Authorization` header on every remote write request with the bearer token
# read from the configured file. It is mutually exclusive with `bearer_token`.
[ bearer_token_file: /path/to/bearer/token/file ]

# Configures the remote write request's TLS settings.
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]

# Configures the queue used to write to remote storage.
queue_config:
  # Number of samples to buffer per shard before we block reading of more
  # samples from the WAL. It is recommended to have enough capacity in each
  # shard to buffer several requests to keep throughput up while processing
  # occasional slow remote requests.
  [ capacity: <int> | default = 500 ]
  # Maximum number of shards, i.e. amount of concurrency.
  [ max_shards: <int> | default = 1000 ]
  # Minimum number of shards, i.e. amount of concurrency.
  [ min_shards: <int> | default = 1 ]
  # Maximum number of samples per send.
  [ max_samples_per_send: <int> | default = 100]
  # Maximum time a sample will wait in buffer.
  [ batch_send_deadline: <duration> | default = 5s ]
  # Initial retry delay. Gets doubled for every retry.
  [ min_backoff: <duration> | default = 30ms ]
  # Maximum retry delay.
  [ max_backoff: <duration> | default = 100ms ]
```
Giống như cho `remote_read`, cấu hình đơn giản nhất chỉ là một `remote storage URL`, cộng với một phương thức xác thực.
Bạn có thể sử dụng `write_relabel_configs` để `relabel` hoặc hạn chế số liệu bạn ghi vào `remote storage`. Ví dụ: một cách sử dụng phổ biến là bỏ một số tập hợp con của số liệu:
```
writeRelabelConfigs:
 # drop all metrics of this name across all jobs
 - sourceLabels: ["__name__"]
   regex: some_metric_prefix_to_drop_.*
   action: drop
```
