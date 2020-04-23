# Node-exporter
Prometheus Node Exporter là một chương trình được viết bằng ngôn ngữ Golang. Exporter là một chương trình được sử dụng với mục đích thu thập, chuyển đổi các metric không ở dạng kiểu dữ liệu chuẩn Prometheus sang chuẩn dữ liệu Prometheus. Sau đó exporter sẽ expose web service api chứa thông tin các metrics hoặc đẩy về Prometheus.

![](https://user-images.githubusercontent.com/61723456/79734523-0fc5c480-8321-11ea-87c6-150b32418a20.png)

`Node Exporter` này sẽ đi thu thập các thông số về máy chủ Linux như : ram, load, cpu, disk, network,…. từ đó tổng hợp và xuất ra kênh truy cập các metrics hệ thống này ở port TCP 9100 để Prometheus đi lấy dữ liệu metric cho việc giám sát.

`Node Exporter` sẽ xuất thông tin metric về hệ thống máy chủ Linux trên dịch vụ web với url ‘/metrics‘ và `port` mà nó đang lắng nghe.
+ Cấu hình `Target Node Exporter` trên Prometheus Server.

Khai báo máy chủ Linux đang chạy `Node Exporter` trong một Target trên Prometheus. Thêm địa chỉ IP của máy chủ Linux vào phần ‘targets‘ cùng port `Node Exporter` chạy 9100 TCP.

```
- job_name: 'node_exporter_metrics'
    scrape_interval: 5s
    static_configs:
      - targets: ['IP_linux_server:9100']
```

+ CPU cllector

`Metric` chính từ trình thu thập `cpu` là `node_cpu_seconds_total`, đây là bộ đếm cho biết mỗi CPU dành bao nhiêu thời gian cho mỗi chế độ. Các nhãn là `cpu` và `mode`.

```
node_cpu_seconds_total{cpu="0",mode="idle"} 48649.88
node_cpu_seconds_total{cpu="0",mode="iowait"} 169.99
node_cpu_seconds_total{cpu="0",mode="irq"} 0
node_cpu_seconds_total{cpu="1",mode="idle"} 9413.55
node_cpu_seconds_total{cpu="1",mode="iowait"} 57.41
node_cpu_seconds_total{cpu="1",mode="irq"} 0
```

Đối với mỗi `CPU`, các chế độ sẽ tổng hợp tăng thêm một mỗi giây. Điều này cho phép bạn tính tỷ lệ thời gian nhàn rỗi trên tất cả các `CPU` bằng biểu thức `PromQL`:

```
avg without(cpu, mode)(rate(node_cpu_seconds_total{mode="idle"}[1m]))
```

Điều này hoạt động khi nó tính toán thời gian nhàn rỗi mỗi giây trên mỗi `CPU` và sau đó tính trung bình trên tất cả các `CPU` trong máy.
Ta có thể khái quát hóa điều này để tính tỷ lệ thời gian dành cho mỗi chế độ cho một máy bằng cách sử dụng:
```
avg without(cpu)(rate(node_cpu_seconds_total[1m]))
```
+ **Filesystem Collector**
`filesystem collector` không thu thập được các số liệu về các hệ thống tệp `mounted`. Các cờ `--collector.filesystem.ignored-mount-points` và `--collector.filesystem.ignored-fs` cho phép hạn chế các `filesystems` nào được bao gồm (mặc định loại trừ các hệ thống tập tin giả khác nhau). Vì sẽ không có `Node exporter` chạy bằng root, nên sẽ cần đảm bảo rằng các quyền của tệp cho phép nó sử dụng lệnh gọi hệ thống `statfs` trên các điểm gắn kết được quan tâm.
Tất cả các metric từ trình thu thập này được gắn với tiền tố `node_filesystem_` và có nhãn `device`, `fstype` và `mountpoint`.

```
node_filesystem_size_bytes{device="/dev/sda5",fstype="ext4",mountpoint="/"} 9e+10
```

Các `filesystem metrics` phần lớn là hiển nhiên. Một điều tinh tế bạn nên biết là sự khác biệt giữa `node_filesystem_avail_bytes` và `node_filesystem_free_bytes`. Trên `filesystems Unix`, một số không gian được dành riêng cho người dùng root, do đó họ vẫn có thể thực hiện mọi việc khi người dùng lấp đầy tất cả không gian có sẵn. `node_filesystem_avail_bytes` là không gian có sẵn cho người dùng và khi cố gắng tính toán dung lượng đĩa đã sử dụng, ta nên sử dụng theo:
```
node_filesystem_avail_bytes
/
node_filesystem_size_bytes
```

`node_filesystem_files` và `node_filesystem_files_free` cho biết số lượng nút và số lượng trong số chúng là miễn phí, tương đương với số lượng tệp mà hệ thống tệp của bạn có.

+ **Diskstats Collector**
Trình thu thập đĩa phát ra các số liệu I/O của đĩa từ `/proc/diskstats`. Theo mặc định, cờ `--collector.diskstats.ignored-devices` cố gắng loại trừ những thứ không phải là đĩa thực, chẳng hạn như phân vùng và thiết bị loopback:
```
node_disk_io_now{device="sda"} 0
```
Tất cả các metrics đều có nhãn `device` và hầu hết tất cả đều là `counters`, như sau:
```
#Số lượng I/O đang tiến hành.
node_disk_io_now

#Incremented when I/O is in progress.
node_disk_io_time_seconds_total

#Bytes read by I/Os.
node_disk_read_bytes_total

#The time taken by read I/Os.
node_disk_read_time_seconds_total

#The number of complete I/Os.
node_disk_reads_completed_total

#Bytes written by I/Os.
node_disk_written_bytes_total

#The time taken by write I/Os.
node_disk_write_time_seconds_total

#The number of complete write I/Os.
node_disk_writes_completed_total
```
Bạn có thể tính thời gian trung bình cho một lần đọc I/O với:
```
rate(node_disk_read_time_seconds_total[1m])
/
rate(node_disk_reads_completed_total[1m])
```

+ **Netdev collector**
Trình thu thập `netdev` hiển thị số liệu về các thiết bị mạng của bạn với tiền tố
`node_network_` và một nhãn `device`.
```
node_network_receive_bytes_total{device="lo"} 8.3213967e+07
node_network_receive_bytes_total{device="wlan0"} 7.0854462e+07
```
`node_network_receive_bytes_total` và `node_network_transmit_bytes_total` là các số liệu chính bạn sẽ quan tâm vì bạn có thể tính toán băng thông mạng vào và ra với chúng.
```
rate(node_network_receive_bytes_total[1m])
```

Bạn cũng có thể quan tâm đến `node_network_receive_packets_total` và `node_network_transmit_packets_total`, theo dõi các gói vào và ra, tương ứng.

Nút_exporter sẽ hiển thị tất cả các số liệu từ các trình thu thập được bật theo mặc định. Đây là cách được khuyến nghị để thu thập số liệu để tránh sai sót khi so sánh số liệu của các nhóm khác nhau.

# SNMP-exporter
SNMP là gì: `SNMP` viết tắt của `Simple Network Management Protocol` (Giao thức quản lý mạng đơn giản)
`SNMP` được sử dụng để quản lý các thiết bị mạng (chủ yếu được gọi là Đối tượng được quản lý) bằng cách đặt giá trị cho một số thuộc tính nhất định và giám sát các thiết bị mạng bằng cách bỏ phiếu các số liệu cần thiết từ thiết bị.
`SNMP` hoạt động dựa trên mô hình `Client-Server`.

+`SNMP MID` và `OID`
  + `MIB` là viết tắt của `Management Information Base` (Cơ sở thông tin quản lý) và là tập hợp các định nghĩa xác định các thuộc tính của đối tượng được quản lý trong thiết bị cần quản lý. Các tệp `MIB` được viết theo định dạng độc lập và thông tin đối tượng mà chúng chứa được sắp xếp theo thứ bậc. `SNMP` có thể truy cập các mẩu thông tin khác nhau.
 + `OID` hay `Object Identifiers` (Mã định danh đối tượng) xác định duy nhất các đối tượng được quản lý trong `MIB`.

`SNMP exporter` là gì: `SNMP exporter` là một công cụ thu thập dữ liệu từ thiết bị được quản lý và hiển thị dưới dạng định dạng mà được máy chủ Prometheus chấp nhận.
`SNMP exporter` đọc tệp cấu hình, mặc định `snmp.yml`, và cấu hình chứa `OID` để walk/get từ thiết bị và thông tin đăng nhập để sử dụng trong trường hợp nếu đó là `SNMPv2` hoặc `SNMPv3`.

Ví dụ, tạo `snmp.yml` cho bộ định tuyến của `Cisco`.
```
Cisco:
 version: 3
 auth:
 username: snmpUser
 password: yourPassword
 auth_protocol: SHA
 priv_protocol: DES
 security_level: authPriv
 priv_password: privacyPassword
 walk:
  - 1.3.6.1.2.1.1 # sysInfo
  - 1.3.6.1.2.1.2.2 # ifTable
  - 1.3.6.1.2.1.31.1.1 # ifXTable
  metrics:
  #sysInfo
  - name: sysUpTime
    oid: 1.3.6.1.2.1.1.3
    type: counter
    lookups:
    - labels:
      labelname: sysDescr
      oid: 1.3.6.1.2.1.1.1.0
      type: DisplayString
    - labels:
      labelname: sysName
      oid: 1.3.6.1.2.1.1.5.0
      type: DisplayString
    - labels:
      labelname: sysLocation
      oid: 1.3.6.1.2.1.1.6.0
      type: DisplayString
    - labels:
      labelname: sysContact
      oid: 1.3.6.1.2.1.1.4.0
      type: DisplayString
  #Interfaces
  #Interface ifIndex
  - name: ifIndex
    oid: 1.3.6.1.2.1.2.2.1.1
    type: gauge
    indexes:
    - labelname: ifIndex
      type: Integer
    lookups:
    - labels:
      - ifIndex
      labelname: ifDescr
      oid: 1.3.6.1.2.1.2.2.1.2
      type: DisplayString
    - labels:
      - ifIndex
      labelname: ifName
      oid: 1.3.6.1.2.1.31.1.1.1.1
      type: DisplayString
    - labels:
      - ifIndex
      labelname: ifAlias
      oid: 1.3.6.1.2.1.31.1.1.1.18
      type: DisplayString
  #Interface Type
  - name: ifType
    oid: 1.3.6.1.2.1.2.2.1.3
    type: gauge
    indexes:
    - labelname: ifIndex
      type: Integer
    lookups:
    - labels:
      - ifIndex
      labelname: ifDescr
      oid: 1.3.6.1.2.1.2.2.1.2
      type: DisplayString
    - labels:
      - ifIndex
      labelname: ifName
      oid: 1.3.6.1.2.1.31.1.1.1.1
      type: DisplayString
    - labels:
      - ifIndex
      labelname: ifAlias
      oid: 1.3.6.1.2.1.31.1.1.1.18
      type: DisplayString
```

+ The above example has
  + SNMP module "Cisco", ta có thể có bất kỳ số lượng modules nào nếu muốn.
  + Các modules xác định phiên bản `SNMP` để sử dụng .ie: `version: 3`
  + Khối `auth`: chứa tất cả tên người dùng và mật khẩu `SNMP` để liên lạc với thiết bị được quản lý.
  + `walk`: khối này chứa tất cả các `OID` để đi tìm đến các đối tượng. 
  + `metrics`: khối xác định các số liệu sẽ được thu thập, type và những gì tra cứu nên được áp dụng sau khi thu thập.

![](https://user-images.githubusercontent.com/61723456/79736009-5c120400-8323-11ea-8bb8-50389d0337eb.png)

ví dụ:
```
#Interface Speed
  - name: ifSpeed
    oid: 1.3.6.1.2.1.2.2.1.5
    type: gauge
    indexes:
    - labelname: ifIndex
      type: Integer
    lookups:
    - labels:
      - ifIndex
      labelname: ifDescr
      oid: 1.3.6.1.2.1.2.2.1.2
      type: DisplayString
    - labels:
      - ifIndex
      labelname: ifName
      oid: 1.3.6.1.2.1.31.1.1.1.1
      type: DisplayString
```

+ Make Prometheus collect data

prometheus.yml file
```
# Sample config for Prometheus.
global:
  scrape_interval: 5m
  scrape_timeout: 10s
  evaluation_interval: 1m

# A scrape configuration containing exactly one endpoint to scrape:
scrape_configs:
  # Cisco
  - job_name: 'Cisco'
    scrape_interval: 120s
    scrape_timeout: 120s
    file_sd_configs:
        - files :
          - /etc/prometheus/targetCisco.yml
    # SNMP device.
    metrics_path: /snmp
    params:
      module: [Cisco] #which OID's we will be querying in
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: 127.0.0.1:9116  # The SNMP exporter's real hostname:port.
```

 Việc chạy máy chủ Prometheus bây giờ sẽ chạy một `job` có tên `Cisco` để thăm dò các thiết bị được chỉ định trong `scrape_configs` (`static_configs` hoặc `file_sd_configs`) và thu thập dữ liệu để lưu trữ trong TSDB.
Chúng ta nên có thể xem dữ liệu trong Prometheus bằng Truy vấn và trực quan hóa dữ liệu trong biểu đồ hoặc bảng điều khiển.

# HAProxy
`HAProxy` (viết tắt của High Availability Proxy), là một phần mềm mã nguồn mở về TCP/HTTP Load Balancer và giải pháp proxy có thể chạy trên Linux, Solaris và FreeBSD. HAProxy giúp cải thiện hiệu suất và độ tin cậy của môi trường máy chủ bằng cách phân phối khối lượng công việc trên nhiều server (ví dụ: web, application, database).

Các tính năng của HAProxy-Exporter:

+ Simple: HAProxy-Exporter tìm nạp số liệu thống kê thông qua `HTTP endpoint` hoặc `Unix socket`.
+ Highly scalable: HAProxy-Exporter có thể xuất số liệu thống kê của hàng ngàn HAProxy.
+ Pluggable: Xuất số liệu của thông qua Beamium(Beamium là một single binary nó thực hiện một việc là: scraping rồi pushing).
+ Versatile: HAProxy-Exporter có thể đẩy các metrics vào các tệp.

`HAProxy exporter` là một exporter điển hình. Trước tiên cần tạo một tệp cấu hình được gọi là haproxy.cfg.
ví dụ: haproxy.cfg ủy quyền cho Node exporter trên cổng 1234, với cổng 1235 là status port.
```
defaults
  mode http
  timeout server 5s
  timeout connect 5s
  timeout client 5s
frontend frontend
  bind *:1234
  use_backend backend
backend backend
  server node_exporter 127.0.0.1:9100
frontend monitoring
  bind *:1235
  no log
  stats uri /
  stats enable
```
Cấu hình này sẽ ủy quyền http://localhost:1234 cho Node exporter (nếu nó vẫn đang chạy). Phần quan trọng của cấu hình này là:
```
frontend monitoring
  bind *:1235
  no log
  stats uri /
  stats enable
```
Điều này mở ra một giao diện HAProxy trên http://localhost:1235 với các báo cáo thống kê, đặc biệt, đầu ra các metrics được phân tách bằng dấu phẩy (comma-separated value-CSV) trên `http://localhost: 1235/;csv` mà HAProxy exporter sẽ sử dụng.

HAProxy có `frontends`, `backends` và `servers`. Mỗi cái đều có metrics với các tiền tố `haproxy_frontend_`, `haproxy_backend_` và `haproxy_server_`, tương ứng. 
Ví dụ: `haproxy_server_bytes_out_total` là một counter với số byte mà mỗi máy chủ đã trả về.

Vì một `backend` có thể có nhiều `servers`, ta có thể tìm thấy tính chính xác của metrics gây ra các vấn đề `haproxy_server_`. Cờ dòng lệnh `--haproxy.server-metric-` cho phép giới hạn metrics nào được trả về.

Trước phiên bản 1.8.0, `HAProxy` chỉ có thể sử dụng một lõi `CPU`, vì vậy để sử dụng nhiều lõi trên các phiên bản cũ hơn, ta phải chạy nhiều `HAProxy processes`. Theo đó, ta cũng phải chạy một `HAProxy exporter` cho mỗi `HAProxy process`.

`HAProxy exporter` có một đặc điểm đáng chú ý khác mà cho đến nay rất ít exporter khác đã thực hiện. Trong Unix, thông thường các trình tiện ích có tệp `pid` chứa `ID` tiến trình của chúng, được sử dụng bởi `init system` để kiểm soát tiến trình. Nếu ta có một file như vậy và chuyển nó trong cờ dòng lệnh `--haproxy.pid-file`, thì `HAProxy exporter` sẽ bao gồm các metrics `haproxy_ process_` về HAProxy process, ngay cả khi việc scraping HAProxy không thành công. Các metrics này giống như các `process_ metrics`.

Ta có thể định cấu hình `HAProxy exporter` scraped bởi Prometheus giống như bất kỳ exporter nào khác:

```
global:
  scrape_interval: 10s
scrape_configs:
 - job_name: haproxy
   static_configs:
    - targets:
        - localhost:9101
```
