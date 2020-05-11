Dẫn chứng code mới: [netstat](https://github.com/prometheus/node_exporter/blob/master/collector/netstat_linux.go)

# import
+ "bufio": Gói bufio thực hiện việc đệm I/O. Nó chứa một đối tượng `io.Reader` hoặc `io.Writer`, tạo một đối tượng khác (Reader hoặc Writer) cũng thực hiện trên giao diện nhưng cung cấp bộ đệm và một số trợ giúp cho I/O văn bản.
+ "io": Gói io cung cấp các giao diện cơ bản cho các nguyên hàm I/O. Công việc chính của nó là chứa các gói trong os, vào các giao diện chung được chia sẻ để tóm lược hóa chức năng.
+ "os": Gói os cung cấp giao diện độc lập với nền tảng cho chức năng hệ điều hành.
Giao diện os được thiết kế đồng nhất trên tất cả các hệ điều hành.
+ "regexp": Gói regexp thực hiện tìm kiếm biểu thức thông thường. Cú pháp của các biểu thức chính quy được chấp nhận là cùng một cú pháp chung được sử dụng bởi Perl, Python và các ngôn ngữ khác.
+ "strconv": Gói strconv thực hiện chuyển đổi của các loại dữ liệu cơ bản. Các chuyển đổi số phổ biến nhất là Atoi (chuỗi thành int) và Itoa (int thành chuỗi).

```
const (
	netStatsSubsystem = "netstat"
)
```
khai báo hằng số `netStatsSubsystem` nhận giá trị là chuỗi `netstat`

```
var (
	netStatFields = kingpin.Flag("collector.netstat.fields", "Regexp of fields to return for netstat collector.").Default("^(.*_(InErrors|InErrs)|Ip_Forwarding|Ip(6|Ext)_(InOctets|OutOctets)|Icmp6?_(InMsgs|OutMsgs)|TcpExt_(Listen.*|Syncookies.*|TCPSynRetrans)|Tcp_(ActiveOpens|InSegs|OutSegs|PassiveOpens|RetransSegs|CurrEstab)|Udp6?_(InDatagrams|OutDatagrams|NoPorts|RcvbufErrors|SndbufErrors))$").String()
)
```
`kingpin`: Kingpin là một trình phân tích cú pháp dòng lệnh an toàn. Nó hỗ trợ cờ, lệnh lồng nhau và đối số vị trí.
+ `Flag("collector.netstat.fields", "Regexp of fields to return for netstat collector.")`: khai báo cờ `-collector.netstat.fields` được lưu vào `netStatFields`
+ `Default()`: xác định các giá trị mặc định cho cờ.

![](https://user-images.githubusercontent.com/61723456/81504134-c100be80-9311-11ea-9228-d74c3fc23ca9.png)

```
func registerCollector(collector string, isDefaultEnabled bool, factory func(logger log.Logger) (Collector, error)) {
	var helpDefaultState string
	if isDefaultEnabled {
		helpDefaultState = "enabled"
	} else {
		helpDefaultState = "disabled"
	}

	flagName := fmt.Sprintf("collector.%s", collector)
	flagHelp := fmt.Sprintf("Enable the %s collector (default: %s).", collector, helpDefaultState)
	defaultValue := fmt.Sprintf("%v", isDefaultEnabled)

	flag := kingpin.Flag(flagName, flagHelp).Default(defaultValue).Action(collectorFlagAction(collector)).Bool()
	collectorState[collector] = flag

	factories[collector] = factory
}
```
+ `registerCollector("netstat", defaultEnabled, NewNetStatCollector)`: tạo ra định dạng dữ liệu được collecter mà prometheus có thể hiểu và sử dụng, với `collector`=`netstat`, `isDefaultEnabled`=`defaultEnabled`, `factory`=`NewNetStatCollector`.

NewNetStatCollector lấy và trả về một Collector mới hiển thị các số liệu thống kê mạng.
```
func NewNetStatCollector(logger log.Logger) (Collector, error) {
	pattern := regexp.MustCompile(*netStatFields)
	return &netStatCollector{
		fieldPattern: pattern,
		logger:       logger,
	}, nil
}
```

`NewNetStatCollector(logger log.Logger) (Collector, error)`
`pattern := regexp.MustCompile(*netStatFields)` Tạo các biến toàn cục với các biểu thức dựa trên các giá trị của cờ `netStatFields`.
Trả về các giá trị cho struct `netStatCollector`
```
return &netStatCollector{
		fieldPattern: pattern,
		logger:       logger,
	}
```

Tạo ra các metrics
```
func (c *netStatCollector) Update(ch chan<- prometheus.Metric) error {
	netStats, err := getNetStats(procFilePath("net/netstat"))
	if err != nil {
		return fmt.Errorf("couldn't get netstats: %s", err)
	}
	snmpStats, err := getNetStats(procFilePath("net/snmp"))
	if err != nil {
		return fmt.Errorf("couldn't get SNMP stats: %s", err)
	}
	snmp6Stats, err := getSNMP6Stats(procFilePath("net/snmp6"))
	if err != nil {
		return fmt.Errorf("couldn't get SNMP6 stats: %s", err)
	}
	// Hợp nhất kết quả của snmpStats vào netStats (có thể xung đột, nhưng các khóa luôn là duy nhất cho trường hợp sử dụng đã cho).
	for k, v := range snmpStats {
		netStats[k] = v
	}
	for k, v := range snmp6Stats {
		netStats[k] = v
	}
	for protocol, protocolStats := range netStats {
		for name, value := range protocolStats {
			key := protocol + "_" + name
			v, err := strconv.ParseFloat(value, 64)
			if err != nil {
				return fmt.Errorf("invalid value %s in netstats: %s", value, err)
			}
			if !c.fieldPattern.MatchString(key) {
				continue
			}
			ch <- prometheus.MustNewConstMetric(
				prometheus.NewDesc(
					prometheus.BuildFQName(namespace, netStatsSubsystem, key),
					fmt.Sprintf("Statistic %s.", protocol+name),
					nil, nil,
				),
				prometheus.UntypedValue, v,
			)
		}
	}
	return nil
}
```
Các file lấy dữ liệu như sau:
+ file netstat
![](https://user-images.githubusercontent.com/61723456/81504323-b561c780-9312-11ea-82df-ed620ae33dfa.png)

+ file snmp
![](https://user-images.githubusercontent.com/61723456/81504338-cc081e80-9312-11ea-9096-f3c75437c9b0.png)

+ file snmp6
![](https://user-images.githubusercontent.com/61723456/81504357-e7732980-9312-11ea-9088-d9342f3a5a39.png)

+ `Update()`: cung cấp chức năng để thực hiện các chương trình Go an toàn, tự cập nhật (hoặc các mục tiêu, một tệp khác).

```
func MustNewConstMetric(desc *Desc, valueType ValueType, value float64, labelValues ...string) Metric {
	m, err := NewConstMetric(desc, valueType, value, labelValues...)
	if err != nil {
		panic(err)
	}
	return m
}
```
+ `getNetStats(procFilePath("net/netstat"))`: dữ liệu được lấy từ file với đường dẫn cụ thể chỉ đến file.
trong đó:
```
func procFilePath(name string) string {
	return filepath.Join(*procPath, name)
}
```

Lấy ra dữ liệu từ file `netstats`:
```
func getNetStats(fileName string) (map[string]map[string]string, error) {
	file, err := os.Open(fileName)
	if err != nil {
		return nil, err
	}
	defer file.Close()

	return parseNetStats(file, fileName)
}
```

+ ` getNetStats`: sử dụng `os.Open()` để mở file lấy dữ liệu, ở đây cụ thể là đường dẫn tới `netstat`.

Phân tích cú pháp, dữ liệu NetStats
```
func parseNetStats(r io.Reader, fileName string) (map[string]map[string]string, error) {
	var (
		netStats = map[string]map[string]string{}
		scanner  = bufio.NewScanner(r)
	)

	for scanner.Scan() {
		nameParts := strings.Split(scanner.Text(), " ")
		scanner.Scan()
		valueParts := strings.Split(scanner.Text(), " ")
		// Remove trailing :.
		protocol := nameParts[0][:len(nameParts[0])-1]
		netStats[protocol] = map[string]string{}
		if len(nameParts) != len(valueParts) {
			return nil, fmt.Errorf("mismatch field count mismatch in %s: %s",
				fileName, protocol)
		}
		for i := 1; i < len(nameParts); i++ {
			netStats[protocol][nameParts[i]] = valueParts[i]
		}
	}

	return netStats, scanner.Err()
}
```

+ `parseNetStats(r io.Reader, fileName string)`: thực hiện đọc file có tên `netstat`

định nghĩa 1 số biến:
```
	var (
		netStats = map[string]map[string]string{}
		scanner  = bufio.NewScanner(r)
	)
```
+ `strings.Split()`:Split() method trong Golang (được xác định trong thư viện strings) chia một chuỗi thành một danh sách các chuỗi con bằng cách phát hiện một dấu phân cách (ví dụ: ",") được chỉ định. Phương thức trả về các chuỗi con dưới dạng một slice.
ví dụ:
```
s := strings.Split("a,b,c", ",")
fmt.Println(s)
// Output: [a b c]
```

Lấy dữ liệu SNMP6Stats
```
func getSNMP6Stats(fileName string) (map[string]map[string]string, error) {
	file, err := os.Open(fileName)
	if err != nil {
		// On systems with IPv6 disabled, this file won't exist.
		// Do nothing.
		if os.IsNotExist(err) {
			return nil, nil
		}

		return nil, err
	}
	defer file.Close()

	return parseSNMP6Stats(file)
}
```

Phân tích cú pháp trong SNMPStats:
```
func parseSNMP6Stats(r io.Reader) (map[string]map[string]string, error) {
	var (
		netStats = map[string]map[string]string{}
		scanner  = bufio.NewScanner(r)
	)

	for scanner.Scan() {
		stat := strings.Fields(scanner.Text())
		if len(stat) < 2 {
			continue
		}
		// Expect to have "6" in metric name, skip line otherwise
		if sixIndex := strings.Index(stat[0], "6"); sixIndex != -1 {
			protocol := stat[0][:sixIndex+1]
			name := stat[0][sixIndex+1:]
			if _, present := netStats[protocol]; !present {
				netStats[protocol] = map[string]string{}
			}
			netStats[protocol][name] = stat[1]
		}
	}

	return netStats, scanner.Err()
}
```
+ `strings.Fields()`:  để tách một chuỗi thành các chuỗi con loại bỏ bất kỳ ký tự khoảng trắng nào, bao gồm cả dòng mới.
ví dụ:
```
s := strings.Fields(" a \t b \n")
fmt.Println(s)
// Output: [a b]
```
