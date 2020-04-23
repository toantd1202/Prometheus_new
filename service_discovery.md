
# Service discovery
## Vai trò
Để biết tất cả các thiết bị và dịch vụ đang ở đâu, chúng được đặt ra như thế nào. `Service discovery` (SD) sẽ cung cấp các thông tin đó cho Prometheus từ bất kỳ cơ sở dữ liệu nào bạn lưu trữ. Prometheus hỗ trợ nhiều nguồn thông tin dịch vụ phổ biến, như Consul, Amazon, EC2 và Kubernetes. Nếu cụ thể `source` của bạn chưa được hỗ trợ, bạn có thể sử dụng cơ chế `service discovery` dựa trên các file để móc nối tới nó. Điều này có thể thực hiện nhờ hệ thống quản lý cấu hình của bạn, như `Ansible` hoặc `Chef`, viết danh sách các thiết bị và dịch vụ mà họ biết về đúng định dạng hoặc tập lệnh chạy thường xuyên bạn sử dụng để lấy từ bất kỳ nguồn dữ liệu nào.
Bước đầu tiên ta phải thực hiện là xác định mục tiêu giám sát là gì, từ đó xác định những gì cần được loại bỏ. Nhãn là một phần quan trọng của Prometheus.
+ Nhãn mục tiêu cho phép chúng được nhóm lại và sắp xếp theo cách có ý nghĩa.
+ Nhãn mục tiêu cho phép bạn tổng hợp các mục tiêu thực hiện cùng một vai trò, trong cùng một môi trường hoặc được điều hành bởi cùng một nhóm.

Vì các nhãn mục tiêu được cấu hình trong Prometheus thay vì trong chính các ứng dụng và các exporters, điều này cho phép các team khác nhau của bạn có hệ thống phân cấp nhãn có ý nghĩa với chúng.

Một cơ chế khám phá dịch vụ tốt sẽ cung cấp cho bạn `metadata`. Đây có thể là tên của một dịch vụ, mô tả của nó, nhóm nào sở hữu nó, các thẻ có cấu trúc về nó hoặc bất cứ thứ gì khác mà bạn có thể thấy hữu ích. Siêu dữ liệu là những gì bạn sẽ chuyển đổi thành nhãn mục tiêu và nói chung bạn càng có nhiều siêu dữ liệu thì càng tốt.

Một cơ chế khám phá dịch vụ tốt sẽ cung cấp cho bạn `metadata`. Đó có thể là tên của một dịch vụ, mô tả của nó, nhóm nào sở hữu nó, các thẻ có cấu trúc về nó hoặc bất cứ thứ gì khác mà bạn có thể thấy hữu ích. `metadata` là những gì bạn sẽ chuyển đổi thành `target lable` và nói chung bạn càng có nhiều `metadata` thì càng tốt.

`service discovery` được thiết kế để tích hợp với cơ sở dữ liệu của các thiết bị và dịch vụ mà bạn đã có. Prometheus 2.2.1 hỗ trợ Azure, Consul, DNS, EC2, OpenStack, File, Kubernetes, Marathon, Nerve, Serverset và Triton.

## Một số loại `service discovery` cơ bản
### File
`File service discovery`, thường được gọi là `file SD`, không sử dụng mạng. Thay vào đó, nó đọc các target giám sát từ các tệp bạn cung cấp trên hệ thống tệp cục bộ. Điều này cho phép bạn tích hợp với các hệ thống SD, Prometheus không hỗ trợ ra khỏi `box` hoặc khi Prometheus có thể lồng nhau làm những việc bạn cần với siêu dữ liệu có sẵn.
Bạn có thể cung cấp các tệp ở định dạng `JSON` hoặc `YAML`. Phần mở rộng tệp phải là `.json` cho `JSON` và `.yml` hoặc `.yaml` cho `YAML`. 

Ví dụ: về JSON, tệp có tên là filesd.json. Bạn có thể có nhiều hoặc ít target như bạn muốn trong một tệp.
```
[
 {
   "targets": [ "host1:9100", "host2:9100" ],
   "labels": {
     "team": "infra",
     "job": "node"
   }
 },
{
   "targets": [ "host1:9090" ],
   "labels": {
     "team": "monitoring",
     "job": "prometheus"
   }
 }
]
```
Cấu hình trong Prometheus sử dụng `file_sd_configs` trong `scrape config`. Mỗi tệp cấu hình `SD` có một danh sách các `filepath` và bạn có thể sử dụng các khối trong tên tệp. Đường dẫn có liên quan đến thư mục làm việc của Prometheus, nghĩa là thư mục bạn khởi động Prometheus.
```
scrape_configs:
 - job_name: file
   file_sd_configs:
    - files:
       - '*.json'
```
Thông thường, bạn sẽ không cung cấp siêu dữ liệu để sử dụng với việc định vị lại khi sử dụng tệp SD, mà thay vào đó là nhãn đích cuối cùng mà bạn muốn có.
Nếu bạn truy cập http: // localhost: 9090 / dịch vụ khám phá trong trình duyệt 4 của bạn và nhấp vào hiển thị thêm, bạn sẽ thấy Hình 8-1, với cả nhãn công việc và nhóm từ filesd.json. 5 Vì các mục tiêu này được tạo thành, các mẩu tin lưu niệm sẽ thất bại, trừ khi bạn thực sự có host1 và host2 trên mạng của bạn.

### OpenStack

OpenStack exporter, xuất các `Prometheus metrics` từ `OpenStack cloud` đang chạy để prometheus sử dụng. Cấu hình nhận dạng và thông tin nhận dạng cloud nên sử dụng định dạng `os-client-config` và phải được chỉ định bằng cờ `--os-client-config`.

Theo mặc định, openstack_exporter hoạt động trên cổng 0.0.0.0:9180 tại `/metrics` URL.

+ **Metrics**

|Name|Sample Labels|Sample Value|
|:-----:|:------:|:-----:|
|openstack_neutron_agent_state|	adminState="up",hostname="compute-01",region="RegionOne",service="neutron-dhcp-agent"|1 or 0 (bool)|
|openstack_neutron_floating_ips|region="RegionOne"|4.0 (float)|
|openstack_neutron_networks|region="RegionOne"|	25.0 (float)|
|openstack_object_store_objects|region="RegionOne",container_name="test2"|1.0 (float)|
|...|...|...|

### Consul
`Consul service discovery` là một cơ chế khám phá dịch vụ sử dụng mạng, vì hầu như tất cả các cơ chế đều hoạt động. Nếu bạn chưa có hệ thống khám phá dịch vụ trong tổ chức của mình, `Consul` là một trong những hệ thống dễ dàng hơn cả để khởi động và chạy. `Consul` có một `agent` chạy trên mỗi máy của bạn và những phần tử này trò chuyện với nhau. Các ứng dụng chỉ nói chuyện với các `local agent` trên một máy. Một số agents cũng là máy chủ, cung cấp sự ổn định và nhất quán.

`Consul` không expose /metrics, do đó, các mẩu tin từ Prometheus của bạn sẽ fail. Nhưng nó vẫn cung cấp đủ để tìm tất cả các thiết bị của bạn đang chạy một `Consul agent`, và do đó nên chạy một `Node exporter` mà bạn có thể scrape.

`Consul` thường chạy trên cổng 8500, nên bạn không cần phải thực hiện bất kỳ cấu hình bổ sung nào vì `consul exporter` sử dụng cổng đó theo mặc định.

+ **Exported Metrics**

`metrics` đầu tiên bạn cần lưu ý ở đây là `consul_up`. Một số `exporters` sẽ trả lại `HTTP error` cho Prometheus khi tìm nạp dữ liệu không thành công, dẫn đến việc `up` được đặt thành 0 trong Prometheus. Nhưng nhiều `exporters` vẫn sẽ được scraped thành công trong kịch bản này và sử dụng một metric như `consul_up` để cho biết nếu có vấn đề. Theo đó, khi cảnh báo về việc `Consul` bị down, bạn nên kiểm tra cả `up` và `consul_up`. Nếu bạn dừng `consul` và sau đó kiểm tra /metrics bạn sẽ thấy giá trị thay đổi thành 0 và quay lại 1 lần nữa khi `consul` được chạy lại.

`consul_catalog_service_node_healthy` cho bạn biết về "sức khỏe" của các dịch vụ khác nhau trong `Consul node`.

`consul_serf_lan_member` là số lượng `consul agent` trong cụm. Bạn có thể tự hỏi liệu điều này có thể đến từ `leader` của cụm `consul`, nhưng hãy nhớ rằng mỗi `agent` có thể có một cái nhìn khác nhau về số lượng thành viên của cụm nếu có vấn đề như phân vùng mạng. Nói chung, bạn nên đưa ra các số liệu như thế này từ mọi thành viên của cụm và tổng hợp giá trị bạn muốn bằng cách sử dụng tổng hợp trong PromQL.

Ngoài ra còn có một số `metrics` về `consul exporter`: `consul_exporter_build_info` là thông tin xây dựng của nó và có nhiều `metrics` như `process_` và `go_` về quy trình và thời gian chạy `Go`. Chúng hữu ích cho việc debugging các vấn đề với chính `consul exporter`.

|Metrics|Meaning|Labels|
|:------:|:------:|:------:|
|consul_raft_peers|Có bao nhiêu `peers`(servers) trong cụm Raft||
|consul_serf_lan_member_status|TRạng thái của `member` trong cụm. 1=`Alive`, 2=`Leaving`, 3=`Left`, 4=`Failled`|member|
|consul_catalog_services|Có bao nhiêu dịch vụ trong cụm||
|consul_health_node_status|Kiểm tra trạng thái "sức khỏe" liên quan đến một nút|check, node, status|
|consul_health_service_status|Kiểm tra trạng thái "sức khỏe" liên quan đến một dịch vụ|check, node, service, status|
|consul_catalog_kv|Các giá trị cho các khóa được chọn trong danh `key/value` của Consul. Các khóa có giá trị không phải là số bị bỏ qua|key|

+ **Flags**

  + `consul.allow_stale`: Cho phép bất kỳ `Consul server` (non-leader) để phục vụ việc đọc.
  + `consul.ca-file`: Đường dẫn tệp đến bộ phận chứng nhận, được mã hóa PEM sử dụng để xác thực tính chính xác của server certificate.
  + `consul.cert-file`: Đường dẫn tệp đến certificate, được mã hóa `PEM` được sử dụng với private key để xác minh tính xác thực của exporter.
  + `consul.health-summary`: Thu thập thông tin về từng dịch vụ đã đăng ký và xuất `consul_catalog_service_node_healthy`. Điều này yêu cầu n+1 truy vấn `Consul API` để thu thập tất cả thông tin về từng dịch vụ. Thông tin kiểm tra "sức khỏe" cũng có sẵn thông qua `consul_health_service_status`, nhưng chỉ dành cho các dịch vụ có cấu hình kiểm tra "sức khỏe". Mặc định là true.
  + `consul.key-file`: Đường dẫn tệp đến private key được mã hóa `PEM`, sử dụng cùng với certificate để xác minh tính xác thực của exporter.
  + `consul.server-name`: Khi được cung cấp, nó sẽ ghi đè tên máy chủ cho `TLS certificate`. Nó có thể được sử dụng để đảm bảo rằng tên certificate khớp với tên máy chủ được khai báo.
  + `consul.timeout`: Hết thời gian yêu cầu HTTP tới consul.
  + `version`: Hiển thị phiên bản ứng dụng.

`PEM` là viết tắt của `Privacy Enhanced Mail`, `PEM` đã thất bại ở nhiệm vụ chính của nó là mang lại sự bảo mật cho email, nhưng nó vẫn là một định dạng chứng chỉ được các ứng dụng hỗ trợ. Về bản chất, các tệp `PEM` là các tệp `DER` được mã hóa `Base64`, ở đây số 0 và các tệp được mã hóa theo một chuỗi các ký tự có thể in được. Bằng cách này bạn có thể mở chúng bằng bất kỳ trình soạn thảo văn bản nào, kể cả `Notepad`.

`PEM` là định dạng chứng chỉ phổ biến nhất và là định dạng bạn dễ dàng gặp phải. Phần lớn các CA cung cấp Chứng chỉ `SSL` ở định dạng `PEM` với các phần mở rộng tệp khác nhau, chẳng hạn như `.pem`, `.crt`, `.cer` hoặc `.key`.

+ **Key/Value Checks**


`Exporter` này hỗ trợ lấy các cặp key/value từ  `Consul's KV` và đưa chúng đến Prometheus. Điều này có thể hữu ích, trong trường hợp, nếu bạn sử dụng `Consul KV` để lưu trữ kích thước cụm dự định của bạn và muốn vẽ biểu đồ giá trị đó so với giá trị thực được tìm thấy qua giám sát.

+ `kv.filter`: Chỉ lưu trữ khóa phù hợp với mẫu regex này.
+ `kv.prefix`: Tiền tố để tìm cặp KV.


Một prefix phải được cung cấp để kích hoạt tính năng này. Sử dụng `/` nếu bạn muốn tìm kiếm toàn bộ keyspace.


