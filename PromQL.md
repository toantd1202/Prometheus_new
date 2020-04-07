# Tìm hiểu về PromQL (Operators, Functions)

Prometheus cung cấp một ngôn ngữ truy vấn chức năng gọi là PromQL (Prometheus Query Language - Ngôn ngữ truy vấn Prometheus) cho phép người dùng chọn và tổng hợp time series data theo thời gian thực. Kết quả của một biểu thức có thể được hiển thị dưới dạng biểu đồ, được xem dưới dạng dữ liệu dạng bảng trong trình duyệt biểu thức của Prometheus hoặc được sử dụng bởi các hệ thống bên ngoài thông qua API HTTP.

  

## Operators

### Binary operators

Ngôn ngữ truy vấn của Prometheus hỗ trợ các toán tử logic và số học cơ bản. Đối với các hoạt động giữa hai vectơ tức thời, cách xử lý phù hợp có thể được sửa đổi.

#### Arithmetic binary operators

Các toán tử số học nhị phân sau tồn tại trong Prometheus:

+ `+` (addition)

+ `-` (subtraction)

+ `*` (multiplication)

+ `/` (division)

+ `%` (modulo)

+ `^` (power/exponentiation)

Toán tử số học nhị phân được xác định giữa các cặp vô hướng/vô hướng (scalars/scalars), vector/vô hướng (vector/scalars) và các cặp giá trị vector/vector.

**Between two scalars**, như những phép tính toán giưa các hằng số.
**Between an instant vector and a scalar**, các toán tử được áp dụng cho giá trị của mọi mẫu dữ liệu trong vectơ. Ví dụ, nếu một vectơ tức thời của chuỗi thời gian được nhân với một số bất kỳ, kết quả là một vectơ khác trong đó mọi giá trị mẫu của vectơ gốc được nhân với số đó.

**Between two instant vectors**, một toán tử số học nhị phân được áp dụng cho mỗi mục trong vectơ bên trái và phần tử khớp của nó trong vectơ bên phải. Kết quả được truyền vào vector kết quả với các nhãn nhóm trở thành bộ nhãn đầu ra. Metric name được loại bỏ.

#### So sánh toán tử nhị phân

Các toán tử so sánh nhị phân sau tồn tại trong Prometheus:

+ `==` (equal)

+ `!=` (not-equal)

+ `>` (greater-than)

+ `<` (less-than)

+ `>=` (greater or equal)

+ `<=` (less or equal)

Toán tử so sánh được xác định giữa các cặp giá trị scalar/scalar, vector/scalar và vector/vector. Theo mặc định chúng là các bộ lọc. Cách xử lý của họ có thể được sửa đổi bằng việc cung cấp `bool` sau toán tử, sẽ trả về 0 hoặc 1 cho giá trị thay vì lọc.

**Between two scalars** công cụ sửa đổi `bool` phải được cung cấp và các toán tử này trả về một giá trị vô hướng khác là 0 (sai) hoặc 1 (đúng), tùy thuộc vào giá trị so sánh.

**Between an instant vector and a scalar**, các toán tử này được áp dụng cho giá trị của mọi mẫu dữ liệu trong vectơ và các phần tử vectơ, kết quả so sánh là sai được loại bỏ khỏi vectơ kết quả. Nếu công cụ sửa đổi `bool` được cung cấp, các phần tử vectơ sẽ bị loại bỏ thay vào đó có giá trị 0 và các phần tử vectơ sẽ được giữ có giá trị 1.

**Between two instant vectors** các toán tử này hoạt động như một bộ lọc theo mặc định, được áp dụng cho các phần tương đồng. Các phần tử vectơ mà biểu thức không đúng hoặc không tìm thấy sự tương đồng ở phía bên kia của biểu thức bị loại bỏ khỏi kết quả, trong khi các phần tử khác được truyền vào một vectơ kết quả với các grouping labels được đặt thành nhãn đầu ra. Nếu công cụ sửa đổi `bool` được cung cấp, các phần tử vectơ sẽ bị loại bỏ thay vào đó có giá trị 0 và các phần tử vectơ sẽ được giữ giá trị 1, với các grouping label lại trở thành bộ nhãn đầu ra.

**Logical/set binary operators** chỉ xác định giữa các instant vectors:

+ `and` (intersection)

+ `or` (union)

+ `unless` (complement-bổ xung)

`Vector1` `and` `vector2` dẫn đến một vectơ bao gồm các phần tử của `vector1` có các phần tử trong `vector2` với các bộ nhãn khớp chính xác. Tên và giá trị số liệu được chuyển từ vectơ bên trái.  
  
`Vector1` `or` `vector2` dẫn đến một vectơ chứa tất cả các phần tử gốc (bộ nhãn + giá trị) của `vector1` và tất cả các phần tử của `vector2` không có bộ nhãn phù hợp trong `vector1`.  
  
`Vector1` `unless` `vector2` dẫn đến một vectơ bao gồm các phần tử của vectơ1 mà không có phần tử nào trong vectơ 2 với các bộ nhãn khớp chính xác. Tất cả các yếu tố phù hợp trong cả hai vectơ được loại bỏ.

#### Vector matching

Các hoạt động giữa các vectơ cố gắng tìm một phần tử phù hợp trong vectơ bên phải cho mỗi mục ở phía bên trái. Có hai loại matching cơ bản: `One-to-one` và `many-to-one/ one-to-many`.

**One-to-one vector matches**

One-to-one tìm thấy một cặp mục duy nhất từ mỗi bên của hoạt động. Trong trường hợp mặc định, đó là một hoạt động theo định dạng `vector1` `<operator>` `vector2`. Hai mục khớp nhau nếu chúng có cùng labels và values tương ứng. Từ khóa `ignoring` cho phép bỏ qua các nhãn nhất định khi matching, trong khi từ khóa `on` cho phép giảm tập hợp các nhãn được xem xét vào danh sách được cung cấp:

``` 
 <vector expr> <bin-op> ignoring(<label list>) <vector expr>
 <vector expr> <bin-op> on(<label list>) <vector expr>
```
Example input:
```
method_code:http_errors:rate5m{method="get", code="500"}  24
method_code:http_errors:rate5m{method="get", code="404"}  30
method_code:http_errors:rate5m{method="put", code="501"}  3
method_code:http_errors:rate5m{method="post", code="500"} 6
method_code:http_errors:rate5m{method="post", code="404"} 21

method:http_requests:rate5m{method="get"}  600
method:http_requests:rate5m{method="del"}  34
method:http_requests:rate5m{method="post"} 120
```
Example query:
```
method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
```
Điều này trả về một vectơ kết quả chứa một phần các yêu cầu HTTP với mã trạng thái là 500 cho mỗi phương thức, được đo trong 5 phút qua. Nếu không bỏ qua (mã) thì sẽ không có kết quả trùng khớp vì các số liệu không chia sẻ cùng một bộ nhãn. Các mục nhập với phương thức `put` và `del` không khớp và sẽ không hiển thị trong kết quả:

``` 
  {method="get"}  0.04      //  24 / 600
  {method="post"} 0.05     //    6 / 120
```

**Many-to-one and one-to-many vector matches**

Liên quan đến trường hợp mỗi phần tử vectơ ở `one`-side có thể khớp với nhiều phần tử ở 'many'-side. Điều này phải được yêu cầu rõ ràng bằng cách sử dụng công cụ sửa đổi `group_left` hoặc `group_right`, trong đó left/right xác định vectơ nào có cardinality cao hơn.
```
<vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
```
Danh sách nhãn được cung cấp với công cụ sửa đổi nhóm chứa các nhãn bổ sung từ `one`-side được đưa vào result metrics. Đối với `on`, một nhãn chỉ có thể xuất hiện ở một trong các danh sách. Mỗi time series của vector kết quả phải được xác định duy nhất.

  
Group modifiers chỉ có thể được sử dụng để comparison và arithmetic. Các toán tử như `and`, `unless` và `or` thì các toán tử phù hợp với tất cả các mục có thể trong vector bên phải theo mặc định.
Example query:
```
method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m
```
Trong trường hợp này, vectơ bên trái chứa nhiều mục nhập trên mỗi `method label value`. Vì vậy, chúng ta chỉ ra điều này bằng cách sử dụng `group_left`. Các phần tử từ phía bên phải hiện được khớp với nhiều phần tử có cùng nhãn phương thức ở bên trái:

```

`{method="get", code="500"}  0.04            //  24 / 600`
`{method="get", code="404"}  0.05            //  30 / 600`
`{method="post", code="500"} 0.05            //   6 / 120`
`{method="post", code="404"} 0.175           //  21 / 120`
```

`Many-to-one` và `one-to-many`  matching là các trường hợp sử dụng nâng cao cần được xem xét cẩn thận. Thông thường việc sử dụng đúng cách ignoring(label) cung cấp kết quả mong muốn.

#### Aggregation operations

Prometheus hỗ trợ các toán tử tổng hợp tích hợp sau, có thể được sử dụng để tổng hợp các phần tử của một instant vector, dẫn đến một vector mới có ít phần tử hơn với các giá trị tổng hợp:

+ `sum` (Tính tổng theo kích thước)

+ `min` (Chọn tối thiểu trên kích thước)

+ `max` (Chọn tối đa trên kích thước)

+ `avg` (Tính trung bình trên các kích thước)

+ `stddev` (calculate population standard deviation over dimensions)

+ `stdvar` (calculate population standard variance over dimensions)

+ `count` (Đếm số phần tử trong vectơ)

+ `count_values` (Đếm số phần tử có cùng giá trị)

+ `bottomk` (Các phần tử k nhỏ nhất theo giá trị mẫu)

+ `topk` (Phần tử k lớn nhất theo giá trị mẫu)

+ `quantile` (calculate φ-quantile (0 ≤ φ ≤ 1) over dimensions)

Các toán tử này có thể được sử dụng để tổng hợp trên tất cả các kích thước nhãn hoặc duy trì các kích thước riêng biệt bằng cách bao gồm một mệnh đề `without` hoặc `by`. Các mệnh đề này có thể được sử dụng trước hoặc sau biểu thức.
```
<aggr-op> [without|by (<label list>)] ([parameter,] <vector expression>)
```
or
```
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```
`label list` là danh sách các nhãn không được trích dẫn có thể bao gồm dấu phẩy, nghĩa là cả hai `(label1, label2)` và `(label1, label2, )` là cú pháp hợp lệ.

`Without` loại bỏ các nhãn được liệt kê khỏi vector kết quả, trong khi tất cả các nhãn khác được bảo toàn đầu ra. `by` làm ngược lại và bỏ các nhãn không được liệt kê trong mệnh đề by, ngay cả khi các giá trị nhãn của chúng giống hệt nhau giữa tất cả các phần tử của vectơ.

`parameter` chỉ được yêu cầu cho `Count_values`, `quantile`, `topk` và `bottomk`.

`count_values` xuất một time series cho mỗi giá trị mẫu duy nhất. Mỗi loạt có một nhãn bổ sung. Tên của nhãn đó được cho bởi tham số tổng hợp và giá trị nhãn là giá trị mẫu duy nhất. Giá trị của mỗi time series là số lần giá trị mẫu có mặt.

`Topk` và `bottomk` khác với các bộ tổng hợp khác ở chỗ một tập hợp con của các mẫu đầu vào, bao gồm các nhãn gốc, được trả về trong result vector. `by` và `without` được sử dụng để bucket vector đầu vào.

ví dụ:  
Nếu  metric http_requests_total có time series xuất hiện theo nhãn `application`, `instance` và `group` labels, chúng ta có thể tính tổng số yêu cầu HTTP được xem cho mỗi ứng dụng và nhóm qua tất cả các trường hợp thông qua:
```
sum without (instance) (http_requests_total)
```
Tương đương với:
```
sum by (application, group) (http_requests_total)
```
Nếu chúng ta chỉ quan tâm đến tổng số yêu cầu HTTP mà chúng ta đã thấy trong tất cả các ứng dụng, chúng ta chỉ cần viết:
```
sum(http_requests_total)
```
Để đếm số nhị phân chạy mỗi build version, chúng ta có thể viết:
```
count_values("version", build_version)
```
Để có được 5 yêu cầu HTTP lớn nhất trong tất cả các trường hợp chúng ta có thể viết:
```
topk(5, http_requests_total)
```
#### Binary operator precedence
Danh sách sau đây cho thấy sự ưu tiên của các toán tử nhị phân trong Prometheus, từ cao nhất đến thấp nhất.
1.  `^`
2.  `*`, `/`, `%`
3.  `+`, `-`
4.  `==`, `!=`, `<=`, `<`, `>=`, `>`
5.  `and`, `unless`
6.  `or`

Các toán tử trên cùng mức độ ưu tiên sẽ tính từ trái qua. Ví dụ: 2 * 3% 2 tương đương với (2 * 3)% 2. Tuy nhiên `^`thì ngược lại, vì vậy 2 ^ 3 ^ 2 tương đương với 2 ^ (3 ^ 2).
## Functions
Một số hàm có đối số mặc định.
 ví dụ: `year(v=vector (time()) instant-vector)`
 Điều này có nghĩa là có một đối số `v` là một instant vector, nếu không được cung cấp, nó sẽ mặc định là giá trị của biểu thức `vector(time ())`.

### abs()

`abs(v instant-vector)` trả về vector đầu vào với tất cả các giá trị mẫu được chuyển đổi thành giá trị tuyệt đối của chúng.

### absent()

`absent(v instant-vector)` trả về một vectơ trống nếu vectơ truyền cho nó có giá trị và vector có giá trị 1 nếu vectơ truyền cho nó không có phần tử.

Điều này hữu ích để cảnh báo khi không có time series tồn tại cho một kết hợp metric name và label nhất định.

```
absent(nonexistent{job="myjob"})
# => {job="myjob"}
absent(nonexistent{job="myjob",instance=~".*"})
# => {job="myjob"}
absent(sum(nonexistent{job="myjob"}))
# => {}
```

Trong hai ví dụ đầu tiên, absent () cố gắng nhanh chóng về việc lấy nhãn của vectơ đầu ra 1 phần tử từ vectơ đầu vào. :))

#### absent_over_time()

`absent_over_time(v range-vector)` trả về một vectơ trống nếu range-vector được truyền cho nó có bất kỳ phần tử nào và có giá trị 1 nếu vectơ phạm vi được truyền cho nó không có phần tử.

Điều này hữu ích để cảnh báo khi không có chuỗi thời gian tồn tại cho metric name và label comparison nhất định trong một khoảng thời gian nhất định.

```
absent_over_time(nonexistent{job="myjob"}[1h])
# => {job="myjob"}
absent_over_time(nonexistent{job="myjob",instance=~".*"}[1h])
# => {job="myjob"}
absent_over_time(sum(nonexistent{job="myjob"})[1h:])
# => {}
```

Trong hai ví dụ đầu tiên, `absent_over_time ()` cố gắng nhanh chóng về việc lấy nhãn của vectơ đầu ra 1 phần tử từ vectơ đầu vào.

#### ceil()

`ceil(v instant-vector)` làm tròn các giá trị mẫu của tất cả các phần tử trong v cho đến số nguyên gần nhất.

#### changes()

Đối với mỗi input time series, `changes(v range-vector)` trả về số lần giá trị của nó đã thay đổi trong phạm vi thời gian được cung cấp dưới dạng một instant vector.

  

#### clamp_max()

`clamp_max (v instant-vector , max scalar)` clamps các giá trị mẫu của tất cả các phần tử trong `v` để có giới hạn trên là `max`.

#### clamp_min()  
`clamp_min (v instant-vector, min scalar)` clamps các giá trị mẫu của tất cả các phần tử trong `v` để có giới hạn dưới là `min`.

#### day_of_month()

`day_of_month (v=vector(time()) instant-vector)` trả về ngày trong tháng cho mỗi thời điểm đã cho trong UTC. Giá trị trả về là từ 1 đến 31.

#### day_of_week()

`day_of_week (v=vector (time ()) instant-vector)` trả về ngày trong tuần cho mỗi thời điểm đã cho trong UTC. Các giá trị được trả về là từ 0 đến 6, trong đó 0 có nghĩa là Sunday.

#### days_in_month()

`days_in_month (v=vector(time ()) instant-vector)` trả về số ngày trong tháng cho mỗi lần nhất định trong UTC. Giá trị trả về là từ 28 đến 31.

#### delta()

  
`delta (v range-vector)` tính toán sự khác biệt giữa giá trị đầu tiên và giá trị cuối cùng của mỗi phần tử time series trong một range-vector `v`, trả về một instant vector với các vùng deltas đã cho và các nhãn tương đương. Delta được ngoại suy để bao trùm phạm vi toàn thời gian như được chỉ định trong bộ chọn vectơ phạm vi, do đó có thể nhận được kết quả không nguyên ngay cả khi các giá trị mẫu là tất cả các số nguyên.

ví dụ: Biểu thức ví dụ sau đây trả về chênh lệch nhiệt độ CPU trong khoảng thời gian từ bây giờ đến 2 giờ trước:

```
delta(cpu_temp_celsius{host="zeus"}[2h])
```

`delta` chỉ nên được sử dụng với gauges.

#### deriv()

`deriv(v range-vector)` Tính đạo hàm mỗi giây của time series trong một vectơ phạm vi `v`, sử dụng hồi quy tuyến tính đơn giản.

`deriv` chỉ nên được sử dụng với gauges.

#### exp()

`exp(v instant-vector)` tính hàm số mũ cho tất cả các phần tử trong `v`. Các trường hợp đặc biệt là:

+ exp(+Inf) = +Inf

+ exp(NaN) = NaN

#### floor()

  
`floor(v instant-vector)` làm tròn các giá trị mẫu của tất cả các phần tử trong `v` xuống số nguyên gần nhất.

#### histogram_quantile()

`histogram_quantile(φ float, b Instant-vector)` tính toán φ-quantile (0 ≤ φ ≤ 1) từ các buckets `b` của biểu đồ. Các mẫu trong `b` là số lượng quan sát trong mỗi nhóm. Mỗi mẫu phải có nhãn `le` trong đó giá trị label biểu thị giới hạn trên của bucket. (Các mẫu không có nhãn như vậy sẽ bị bỏ qua) Loại số liệu biểu đồ tự động cung cấp time series với hậu tố `_bucket` và labels phù hợp.

Sử dụng hàm rate() để chỉ định cửa sổ thời gian cho phép tính lượng tử.

Ví dụ: Một histogram metric được gọi là `http_request_duration_seconds`. Để tính phân vị thứ 90 của thời lượng yêu cầu trong 10m cuối cùng, hãy sử dụng biểu thức sau:

```
histogram_quantile(0.9, rate(http_request_duration_seconds_bucket[10m]))
```
Lượng tử được tính cho mỗi kết hợp nhãn trong `http_request_duration_seconds`. Để tổng hợp, sử dụng hàm `sum()` xung quanh hàm rate (). Vì nhãn `le` được yêu cầu bởi `histogram_quantile()`, nên nó phải được bao gồm trong mệnh đề `by`. Biểu thức sau đây tổng hợp phân vị thứ 90 theo công việc:

```
histogram_quantile(0.9, sum(rate(http_request_duration_seconds_bucket[10m])) by (job, le))
```
Để tổng hợp mọi thứ, chỉ xác định nhãn `le`:
```
histogram_quantile(0.9, sum(rate(http_request_duration_seconds_bucket[10m])) by (le))
```
#### increase()

increase(v range-vector) tính toán mức tăng của chuỗi thời gian trong range vector. Mức tăng được ngoại suy để bao trùm phạm vi toàn thời gian như được chỉ định trong bộ chọn range vector, do đó có thể nhận được kết quả không phải là số nguyên ngay cả khi bộ đếm chỉ tăng theo số nguyên.

Biểu thức ví dụ sau đây trả về số lượng yêu cầu HTTP được đo trong 5 phút cuối, mỗi chuỗi thời gian trong vectơ phạm vi:

```
increase(http_requests_total{job="api-server"}[5m])
```

`increase` chỉ nên được sử dụng với counters.

  

#### rate()
`rate(v range-vector)` tính tốc độ tăng trung bình mỗi giây của chuỗi thời gian trong range-vector.

Biểu thức ví dụ sau đây trả về tốc độ của các yêu cầu HTTP trên mỗi giây, được đo trong 5 phút cuối, mỗi chuỗi thời gian trong vectơ phạm vi:

```
rate(http_requests_total{job="api-server"}[5m])
```
`rate` chỉ nên được sử dụng với counters. Nó là phù hợp nhất để cảnh báo và để vẽ đồ thị của các counters chuyển động chậm.

Lưu ý: rằng khi kết hợp `rate()` với toán tử tổng hợp (ví dụ: `sum()`) hoặc hàm tổng hợp theo thời gian (bất kỳ hàm nào kết thúc bằng `_over_time`), trước tiên hãy luôn lấy `rate()` sau đó tổng hợp. Mặt khác, `rate()` không thể phát hiện bộ đếm resets khi target của bạn khởi động lại.

-----
END GAME
