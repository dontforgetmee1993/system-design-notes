# Chương 18: Google Maps

## Giới thiệu

Chúng ta sẽ thiết kế một phiên bản đơn giản của **Google Maps**.

Một số sự thật về Google Maps:
 * Bắt đầu vào năm 2005
 * Cung cấp nhiều dịch vụ khác nhau - hình ảnh vệ tinh, bản đồ đường phố, điều kiện giao thông thời gian thực, lập kế hoạch lộ trình
 * Đến năm 2021, có 1 tỷ người dùng hoạt động hàng ngày, bao phủ 99% thế giới, 25 triệu cập nhật thông tin vị trí thời gian thực hàng ngày

---

## Bước 1: Hiểu vấn đề và xác định phạm vi thiết kế

Mẫu câu hỏi và trả lời giữa ứng viên và người phỏng vấn:
 * C: Chúng ta đang đối mặt với bao nhiêu người dùng hoạt động hàng ngày?
 * I: 1 tỷ DAU.
 * C: Chúng ta nên tập trung vào những tính năng nào?
 * I: Cập nhật vị trí, điều hướng, ETA (thời gian dự kiến đến), hiển thị bản đồ.
 * C: Dữ liệu đường bộ lớn đến mức nào? Chúng ta có quyền truy cập vào nó không?
 * I: Chúng ta thu thập dữ liệu đường bộ từ nhiều nguồn khác nhau, đó là hàng TB dữ liệu thô.
 * C: Chúng ta có nên tính đến các điều kiện giao thông không?
 * I: Có, chúng ta nên làm vậy để ước tính thời gian chính xác.
 * C: Còn các chế độ di chuyển khác nhau - đi bộ, xe đạp, lái xe?
 * I: Chúng ta nên hỗ trợ các chế độ đó.
 * C: Còn chỉ đường nhiều điểm dừng thì sao?
 * I: Hãy không tập trung vào điều đó trong phạm vi buổi phỏng vấn này.
 * C: Các địa điểm kinh doanh và ảnh?
 * I: Câu hỏi hay, nhưng không cần xem xét những thứ đó.

Chúng ta sẽ tập trung vào ba tính năng chính - cập nhật vị trí người dùng, dịch vụ điều hướng bao gồm cả ETA, hiển thị bản đồ.

### **Yêu cầu phi chức năng**

- **Độ chính xác**: người dùng không nên nhận được chỉ đường sai.
- **Điều hướng mượt mà**: Người dùng nên trải nghiệm việc hiển thị bản đồ mượt mà.
- **Sử dụng dữ liệu và pin**: Client nên sử dụng ít dữ liệu và pin nhất có thể. Điều này quan trọng đối với các thiết bị di động.
- Các yêu cầu chung về tính khả dụng và khả năng mở rộng.

### **Kiến thức cơ bản về bản đồ (Map 101)**

Trước khi đi sâu vào thiết kế, có một số khái niệm liên quan đến bản đồ mà chúng ta nên hiểu.

#### Hệ thống định vị

Trái đất là một hình cầu, quay quanh trục của nó. Các vị trí được xác định bằng vĩ độ (latitude - bạn ở xa về phía bắc/nam bao nhiêu) và kinh độ (longitude - bạn ở xa về phía đông/tây bao nhiêu):

<div style="margin-left:3rem">
    <img src="./images/partitioning-system.png" alt="partitioning-system" width="500" />
</div>

#### Chuyển từ 3D sang 2D

Quá trình chuyển đổi các điểm từ không gian 3D sang mặt phẳng 2D được gọi là "phép chiếu bản đồ" (map projection).

Có nhiều cách khác nhau để thực hiện việc này và mỗi cách đều có ưu và nhược điểm riêng. Hầu hết tất cả đều làm biến dạng hình học thực tế.

<div style="margin-left:3rem">
    <img src="./images/map-projections.png" alt="map-projections" width="500" />
</div>

Google Maps đã chọn một phiên bản sửa đổi của phép chiếu Mercator gọi là "Web Mercator".

#### Địa mã hóa (Geocoding)

Geocoding là quá trình chuyển đổi địa chỉ thành tọa độ địa lý.

Quá trình ngược lại được gọi là "địa mã hóa ngược" (reverse geocoding).

Một cách để đạt được điều này là sử dụng nội suy (interpolation) - tận dụng dữ liệu từ các nguồn khác nhau (ví dụ: GIS) nơi mạng lưới đường phố được ánh xạ tới không gian tọa độ địa lý.

#### Geohashing

Geohashing là một hệ thống mã hóa mã hóa một khu vực địa lý thành một chuỗi các chữ cái và chữ số.

Nó mô tả thế giới như một bề mặt phẳng và chia nhỏ nó thành bốn phần tư một cách đệ quy:

<div style="margin-left:3rem">
    <img src="./images/geohashing.png" alt="geohashing" width="500" />
</div>

#### Hiển thị bản đồ (Map rendering)

Hiển thị bản đồ được thực hiện thông qua việc chia ô (tiling). Thay vì hiển thị toàn bộ bản đồ dưới dạng một hình ảnh lớn tùy chỉnh, thế giới được chia nhỏ thành các ô (tiles) nhỏ hơn.

Client chỉ tải xuống các ô có liên quan và hiển thị chúng giống như ghép một bức tranh mosaic.

Có các ô khác nhau cho các mức thu phóng (zoom levels) khác nhau. Client chọn các ô phù hợp dựa trên mức thu phóng của người dùng.

Ví dụ, thu phóng toàn bộ thế giới sẽ chỉ tải xuống một ô 256x256 duy nhất, đại diện cho cả thế giới.

#### Xử lý dữ liệu đường bộ cho các thuật toán điều hướng

Trong hầu hết các thuật toán định tuyến, các giao lộ được đại diện dưới dạng các nút (nodes) và các con đường được đại diện dưới dạng các cạnh (edges):

<div style="margin-left:3rem">
    <img src="./images/road-representation.png" alt="road-representation" width="500" />
</div>

Hầu hết các thuật toán điều hướng sử dụng một phiên bản sửa đổi của thuật toán Dijkstra hoặc A*.

Hiệu suất tìm đường rất nhạy cảm với kích thước của đồ thị. Để hoạt động ở quy mô lớn, chúng ta không thể đại diện toàn bộ thế giới dưới dạng một đồ thị và chạy thuật toán trên đó.

Thay vào đó, chúng ta sử dụng một kỹ thuật tương tự như chia ô - chúng ta chia nhỏ thế giới thành các đồ thị nhỏ dần.

Các ô định tuyến (routing tiles) giữ các tham chiếu đến các ô lân cận và các thuật toán có thể ghép nối một đồ thị đường bộ lớn hơn khi nó đi qua các ô được kết nối với nhau:

<div style="margin-left:3rem">
    <img src="./images/routing-tiles.png" alt="routing-tiles" width="500" />
</div>

Kỹ thuật này cho phép chúng ta giảm đáng kể băng thông bộ nhớ và chỉ tải các ô chúng ta cần cho cặp điểm xuất phát/đích cụ thể.

Tuy nhiên, đối với các lộ trình dài hơn, việc ghép nối các ô định tuyến nhỏ và chi tiết vẫn sẽ tốn thời gian/bộ nhớ. Thay vào đó, có các ô định tuyến với mức độ chi tiết khác nhau và thuật toán sẽ sử dụng các ô có độ chi tiết phù hợp, dựa trên điểm đến mà chúng ta hướng tới:

<div style="margin-left:3rem">
    <img src="./images/map-routing-hierarchical.png" alt="map-routing-hierarchical" width="500" />
</div>

### **Ước tính sơ bộ**

Về lưu trữ, chúng ta cần lưu:
 * Bản đồ thế giới - ước tính khoảng ~70PB dựa trên tất cả các ô chúng ta cần lưu trữ, nhưng đã tính đến việc nén các ô rất giống nhau (ví dụ: sa mạc rộng lớn).
 * Metadata - kích thước không đáng kể, nên chúng ta có thể bỏ qua trong tính toán.
 * Thông tin đường bộ - được lưu trữ dưới dạng các ô định tuyến (routing tiles).

QPS ước tính cho các yêu cầu điều hướng - 1 tỷ DAU với 35 phút sử dụng mỗi tuần -> 5 tỷ phút mỗi ngày.
Giả sử các yêu cầu cập nhật gps được gộp lại theo lô, chúng ta đạt tới 200k QPS và 1 triệu QPS khi tải đỉnh.

---

## Bước 2: Đề xuất thiết kế ở mức cao và đạt được sự đồng thuận

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

### **Dịch vụ vị trí (Location service)**

<div style="margin-left:3rem">
    <img src="./images/location-service.png" alt="location-service" width="500" />
</div>

Nó chịu trách nhiệm ghi lại các bản cập nhật vị trí của người dùng:
 * Các cập nhật vị trí được gửi sau mỗi `t` giây.
 * Các luồng dữ liệu vị trí có thể được sử dụng để cải thiện dịch vụ theo thời gian, ví dụ như cung cấp ETA chính xác hơn, theo dõi dữ liệu giao thông, phát hiện các con đường bị đóng, phân tích hành vi người dùng, v.v.

Thay vì gửi cập nhật vị trí đến máy chủ mọi lúc, chúng ta có thể gộp các bản cập nhật phía client và gửi theo lô (batch) thay thế:

<div style="margin-left:3rem">
    <img src="./images/location-update-batches.png" alt="location-update-batches" width="500" />
</div>

Mặc dù có sự tối ưu hóa này, đối với một hệ thống quy mô như Google Maps, tải vẫn sẽ rất đáng kể. Do đó, chúng ta có thể tận dụng một cơ sở dữ liệu được tối ưu hóa cho các tác vụ ghi nặng như Cassandra.

Chúng ta cũng có thể tận dụng Kafka để xử lý luồng (stream processing) hiệu quả các bản cập nhật vị trí, phục vụ cho việc phân tích sâu hơn.

Ví dụ payload yêu cầu cập nhật vị trí:

```
POST /v1/locations
Parameters
  locs: JSON encoded array of (latitude, longitude, timestamp) tuples.
```

### **Dịch vụ điều hướng (Navigation service)**

Thành phần này chịu trách nhiệm tìm các lộ trình nhanh nhất giữa A và B trong một thời gian hợp lý (một chút độ trễ là chấp nhận được). Lộ trình không nhất thiết phải là nhanh nhất tuyệt đối, nhưng độ chính xác là rất quan trọng.

Ví dụ payload yêu cầu:

```
GET /v1/nav?origin=1355+market+street,SF&destination=Disneyland
```

Ví dụ phản hồi:

```json
{
  "distance": {"text":"0.2 mi", "value": 259},
  "duration": {"text": "1 min", "value": 83},
  "end_location": {"lat": 37.4038943, "Ing": -121.9410454},
  "html_instructions": "Head <b>northeast</b> on <b>Brandon St</b> toward <b>Lumin Way</b><div style=\"font-size:0.9em\">Restricted usage road</div>",
  "polyline": {"points": "_fhcFjbhgVuAwDsCal"},
  "start_location": {"lat": 37.4027165, "lng": -121.9435809},
  "geocoded_waypoints": [
    {
       "geocoder_status" : "OK",
       "partial_match" : true,
       "place_id" : "ChIJwZNMti1fawwRO2aVVVX2yKg",
       "types" : [ "locality", "political" ]
    },
    {
       "geocoder_status" : "OK",
       "partial_match" : true,
       "place_id" : "ChIJ3aPgQGtXawwRLYeiBMUi7bM",
       "types" : [ "locality", "political" ]
    }
  ],
  "travel_mode": "DRIVING"
}
```

Các thay đổi giao thông và định tuyến lại (reroute) vẫn chưa được tính đến, những điều đó sẽ được giải quyết trong phần đi sâu chi tiết.

### **Hiển thị bản đồ (Map rendering)**

Việc lưu giữ toàn bộ tập dữ liệu các ô bản đồ phía client là không khả thi vì nó có kích thước hàng petabyte.

Chúng cần được lấy theo yêu cầu từ máy chủ, dựa trên vị trí của client và mức thu phóng.

Khi nào các ô mới nên được lấy - trong khi người dùng đang thu phóng và trong khi điều hướng, khi họ đang di chuyển tới một ô mới.

Làm thế nào các ô bản đồ nên được phân phối cho client?
 * Chúng có thể được tạo động, nhưng điều đó gây tải cực lớn lên máy chủ và cũng làm cho việc lưu cache trở nên khó khăn.
 * Các ô bản đồ được phục vụ tĩnh, dựa trên geohash của chúng mà client có thể tính toán. Chúng có thể được lưu trữ tĩnh và phục vụ từ một CDN.

<div style="margin-left:3rem">
    <img src="./images/static-map-tiles.png" alt="static-map-tiles" width="500" />
</div>

CDN cho phép người dùng lấy các ô bản đồ từ các máy chủ point-of-presence (POP) gần người dùng nhất để giảm thiểu độ trễ:

<div style="margin-left:3rem">
    <img src="./images/cdn-vs-no-cdn.png" alt="cdn-vs-no-cdn" width="500" />
</div>

Các tùy chọn cần xem xét để xác định các ô bản đồ:
 * Geohash cho ô bản đồ có thể được tính toán phía client. Nếu đúng như vậy, chúng ta nên cẩn thận rằng chúng ta cam kết thực hiện kiểu tính toán ô bản đồ này về lâu dài vì việc buộc client cập nhật là rất khó.
 * Ngoài ra, chúng ta có thể có API đơn giản để tính toán URL các ô bản đồ thay mặt cho client với chi phí là một cuộc gọi API bổ sung.

<div style="margin-left:3rem">
    <img src="./images/map-tile-url-calculation.png" alt="map-tile-url-calculation" width="500" />
</div>

---

## Bước 3: Thiết kế chi tiết

### **Mô hình dữ liệu**

Hãy thảo luận về cách chúng ta lưu trữ các loại dữ liệu khác nhau mà chúng ta đang xử lý.

#### Các ô định tuyến (Routing tiles)

Tập dữ liệu đường bộ ban đầu được thu thập từ nhiều nguồn khác nhau. Nó được cải thiện theo thời gian dựa trên dữ liệu cập nhật vị trí.

Dữ liệu đường bộ là không cấu trúc. Chúng ta có một quy trình xử lý ngoại tuyến (offline processing pipeline) định kỳ, chuyển đổi dữ liệu thô này thành các ô định tuyến dựa trên đồ thị mà ứng dụng của chúng ta cần.

Thay vì lưu trữ các ô này trong cơ sở dữ liệu vì chúng ta không cần bất kỳ tính năng cơ sở dữ liệu nào, chúng ta có thể lưu trữ chúng trong bộ lưu trữ đối tượng S3, đồng thời lưu cache chúng một cách mạnh mẽ.

Chúng ta cũng có thể tận dụng các thư viện để nén danh sách kề (adjacency lists) thành các tệp nhị phân một cách hiệu quả.

#### Dữ liệu vị trí người dùng

Dữ liệu vị trí người dùng rất hữu ích cho việc cập nhật các điều kiện giao thông và thực hiện tất cả các loại phân tích khác.

Chúng ta có thể sử dụng Cassandra để lưu trữ loại dữ liệu này vì bản chất của nó là ghi nặng.

Ví dụ về một hàng dữ liệu:

<div style="margin-left:3rem">
    <img src="./images/user-location-data-torw.png" alt="user-location-data-row" width="500" />
</div>

#### Cơ sở dữ liệu địa mã hóa (Geocoding database)

Cơ sở dữ liệu này lưu trữ các cặp khóa-giá trị gồm cặp vĩ độ/kinh độ và địa điểm.

Chúng ta có thể sử dụng Redis vì tốc độ truy cập đọc nhanh của nó, do chúng ta có các tác vụ đọc thường xuyên và ghi không thường xuyên.

#### Hình ảnh đã tính toán trước của bản đồ thế giới

Như chúng ta đã thảo luận, chúng ta sẽ tính toán trước các hình ảnh chia ô bản đồ và lưu trữ chúng trong CDN.

<div style="margin-left:3rem">
    <img src="./images/precomputed-map-tile-image.png" alt="precomputed-map-tile-image" width="500" />
</div>

### **Các dịch vụ (Services)**

#### Dịch vụ vị trí (Location service)

Hãy tập trung vào thiết kế cơ sở dữ liệu và cách vị trí người dùng được lưu trữ chi tiết cho dịch vụ này.

<div style="margin-left:3rem">
    <img src="./images/location-service-diagram.png" alt="location-service-diagram" width="500" />
</div>

Chúng ta có thể sử dụng cơ sở dữ liệu NoSQL để tạo điều kiện cho tải ghi nặng từ các bản cập nhật vị trí. Chúng ta ưu tiên tính khả dụng hơn tính nhất quán vì dữ liệu vị trí người dùng thường xuyên thay đổi và trở nên cũ khi các bản cập nhật mới đến.

Chúng ta sẽ chọn Cassandra làm lựa chọn cơ sở dữ liệu vì nó phù hợp hoàn hảo với tất cả các yêu cầu của chúng ta.

Ví dụ về hàng dữ liệu chúng ta sẽ lưu:

<div style="margin-left:3rem">
    <img src="./images/user-location-row-example.png" alt="user-location-row-example" width="500" />
</div>

 * `user_id` là khóa phân vùng (partition key) để nhanh chóng truy cập tất cả các bản cập nhật vị trí cho một người dùng cụ thể.
 * `timestamp` là khóa phân cụm (clustering key) để lưu trữ dữ liệu được sắp xếp theo thời gian nhận được bản cập nhật vị trí.

Chúng ta cũng tận dụng Kafka để truyền các bản cập nhật vị trí đến các dịch vụ khác cần dữ liệu này cho nhiều mục đích khác nhau:

<div style="margin-left:3rem">
    <img src="./images/location-update-streaming.png" alt="location-update-streaming" width="500" />
</div>

#### Hiển thị bản đồ (Rendering map)

Các ô bản đồ được lưu trữ ở nhiều mức thu phóng khác nhau. Ở mức thu phóng thấp nhất, toàn bộ thế giới được đại diện bởi một ô 256x256 duy nhất.

Khi các mức thu phóng tăng lên, số lượng các ô bản đồ tăng gấp bốn lần:

<div style="margin-left:3rem">
    <img src="./images/zoom-level-increases.png" alt="zoom-level-increases" width="500" />
</div>

Một tối ưu hóa chúng ta có thể sử dụng là không gửi toàn bộ thông tin hình ảnh qua mạng, mà thay vào đó đại diện cho các ô dưới dạng vector (các đường dẫn và đa giác) và để client tự hiển thị các ô một cách linh hoạt.

Điều này sẽ giúp tiết kiệm băng thông đáng kể.

#### Dịch vụ điều hướng (Navigation service)

Dịch vụ này chịu trách nhiệm tìm các lộ trình nhanh nhất:

<div style="margin-left:3rem">
    <img src="./images/navigation-service.png" alt="navigation-service" width="500" />
</div>

Hãy đi qua từng thành phần trong hệ thống phụ này.

Đầu tiên, chúng ta có dịch vụ địa mã hóa (geocoding service) phân giải một địa chỉ thành một vị trí gồm cặp vĩ độ/kinh độ.

Ví dụ yêu cầu:

```
https://maps.googleapis.com/maps/api/geocode/json?address=1600+Amphitheatre+Parkway,+Mountain+View,+CA
```

Ví dụ phản hồi:

```json
{
   "results" : [
      {
         "formatted_address" : "1600 Amphitheatre Parkway, Mountain View, CA 94043, USA",
         "geometry" : {
            "location" : {
               "lat" : 37.4224764,
               "lng" : -122.0842499
            },
            "location_type" : "ROOFTOP",
            "viewport" : {
               "northeast" : {
                  "lat" : 37.4238253802915,
                  "lng" : -122.0829009197085
               },
               "southwest" : {
                  "lat" : 37.4211274197085,
                  "lng" : -122.0855988802915
               }
            }
         },
         "place_id" : "ChIJ2eUgeAK6j4ARbn5u_wAGqWA",
         "plus_code": {
            "compound_code": "CWC8+W5 Mountain View, California, United States",
            "global_code": "849VCWC8+W5"
         },
         "types" : [ "street_address" ]
      }
   ],
   "status" : "OK"
}
```

Dịch vụ lập kế hoạch lộ trình (route planner service) tính toán một lộ trình đề xuất, được tối ưu hóa về thời gian di chuyển theo điều kiện giao thông hiện tại.

Dịch vụ tìm đường ngắn nhất (shortest-path service) chạy một biến thể của thuật toán A* dựa trên các ô định tuyến trong bộ lưu trữ đối tượng để tính toán một con đường tối ưu:
 * Nó nhận các cặp xuất phát/đích, chuyển đổi chúng thành các cặp vĩ độ/kinh độ và suy ra các geohash từ các cặp đó để lấy ra các ô định tuyến.
 * Thuật toán bắt đầu từ ô định tuyến ban đầu và bắt đầu duyệt qua nó cho đến khi tìm thấy một con đường đủ tốt tới ô đích.

<div style="margin-left:3rem">
    <img src="./images/shortest-path-service.png" alt="shortest-path-service" width="500" />
</div>

Dịch vụ ETA được gọi bởi bộ lập kế hoạch lộ trình để lấy thời gian ước tính dựa trên các thuật toán học máy, dự đoán ETA dựa trên dữ liệu giao thông.

Dịch vụ xếp hạng (ranker service) chịu trách nhiệm xếp hạng các con đường khả thi khác nhau dựa trên các bộ lọc mà người dùng truyền vào, ví dụ: cờ để tránh các con đường có thu phí hoặc xa lộ.

Dịch vụ cập nhật (updater service) cập nhật không đồng bộ một số cơ sở dữ liệu quan trọng để giữ cho chúng luôn mới.

#### Cải tiến - ETA thích ứng và định tuyến lại (rerouting)

Một cải tiến chúng ta có thể thực hiện là cập nhật thích ứng các lộ trình đang di chuyển dựa trên dữ liệu giao thông mới nhất có sẵn.

Một cách để triển khai việc này là lưu trữ những người dùng hiện đang điều hướng qua một lộ trình trong cơ sở dữ liệu bằng cách lưu tất cả các ô mà họ dự kiến sẽ đi qua.

Dữ liệu có thể trông như thế này:

```
user_1: r_1, r_2, r_3, …, r_k
user_2: r_4, r_6, r_9, …, r_n
user_3: r_2, r_8, r_9, …, r_m
...
user_n: r_2, r_10, r21, ..., r_l
```

Nếu một vụ tai nạn giao thông xảy ra trên một ô nào đó, chúng ta có thể xác định tất cả người dùng có lộ trình đi qua ô đó và định tuyến lại cho họ.

Để giảm lượng ô chúng ta lưu trữ trong cơ sở dữ liệu, chúng ta có thể thay vào đó lưu trữ ô định tuyến xuất phát và một vài ô định tuyến ở các cấp độ phân giải khác nhau cho đến khi ô đích cũng được bao gồm:

```
user_1, r_1, super(r_1), super(super(r_1)), ...
```

<div style="margin-left:3rem">
    <img src="./images/adaptive-eta-data-storage.png" alt="adaptive-eta-data-storage" width="500" />
</div>

Sử dụng cách này, chúng ta chỉ cần kiểm tra xem ô cuối cùng của một người dùng có bao gồm ô bị tai nạn giao thông hay không để xem liệu người dùng có bị ảnh hưởng hay không.

Chúng ta cũng có thể theo dõi tất cả các lộ trình khả thi cho một người dùng đang điều hướng và thông báo cho họ nếu có một lộ trình định tuyến lại nhanh hơn.

#### Các giao thức phân phối (Delivery protocols)

Chúng ta có một vài tùy chọn, cho phép chúng ta chủ động đẩy dữ liệu từ máy chủ đến client:
 * Thông báo đẩy di động (Mobile push notifications) không hoạt động vì payload bị hạn chế và nó không khả dụng cho các ứng dụng web.
 * WebSocket nói chung là một lựa chọn tốt hơn so với long-polling vì nó có dấu chân tính toán ít hơn trên máy chủ.
 * Chúng ta cũng có thể sử dụng server-sent events (SSE) nhưng nghiêng về web socket hơn vì chúng hỗ trợ giao tiếp hai chiều, điều này có thể hữu ích cho ví dụ như tính năng giao hàng chặng cuối (last-mile delivery).

---

## Bước 4: Tổng kết

Đây là thiết kế cuối cùng của chúng ta:

<div style="margin-left:3rem">
    <img src="./images/final-design.png" alt="final-design" width="500" />
</div>

Một tính năng bổ sung chúng ta có thể cung cấp là điều hướng nhiều điểm dừng, có thể bán cho các khách hàng doanh nghiệp như Uber hoặc Lyft để xác định con đường tối ưu cho việc ghé thăm một tập hợp các địa điểm.
