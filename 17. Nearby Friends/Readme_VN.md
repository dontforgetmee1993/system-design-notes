# Chương 17: Bạn bè lân cận (Nearby Friends)

## Giới thiệu

Chương này tập trung vào việc thiết kế một backend có khả năng mở rộng cho một ứng dụng cho phép người dùng chia sẻ vị trí của họ và khám phá những người bạn đang ở **lân cận**.

Sự khác biệt lớn nhất so với chương về dịch vụ lân cận (proximity service) là trong bài toán này, **vị trí thay đổi liên tục**, trong khi ở chương đó, địa chỉ doanh nghiệp hầu như giữ nguyên.

---

## Bước 1: Hiểu vấn đề và xác định phạm vi thiết kế

Một số câu hỏi để dẫn dắt buổi phỏng vấn:
 * C: Khoảng cách địa lý bao nhiêu thì được coi là "lân cận"?
 * I: 5 dặm, con số này có thể cấu hình được.
 * C: Khoảng cách được tính theo đường chim bay hay có tính đến các yếu tố cản trở như dòng sông ở giữa?
 * I: Có, đó là một giả định hợp lý.
 * C: Ứng dụng có bao nhiêu người dùng?
 * I: 1 tỷ người dùng và 10% trong số đó sử dụng tính năng bạn bè lân cận.
 * C: Chúng ta có cần lưu trữ lịch sử vị trí không?
 * I: Có, nó có thể có giá trị cho ví dụ như học máy.
 * C: Chúng ta có thể giả định rằng những người bạn không hoạt động sẽ biến mất khỏi tính năng sau 10 phút không?
 * I: Có.
 * C: Chúng ta có cần lo lắng về GDPR, v.v. không?
 * I: Không, để cho đơn giản.

### **Yêu cầu chức năng**

 * Người dùng có thể thấy danh sách bạn bè lân cận trên ứng dụng di động. Mỗi người bạn có thông tin khoảng cách và mốc thời gian (timestamp) cho biết vị trí được cập nhật khi nào.
 * Danh sách bạn bè lân cận nên được cập nhật sau mỗi vài giây.

### **Yêu cầu phi chức năng**

- **Độ trễ thấp**: Quan trọng là phải nhận được cập nhật vị trí mà không bị trễ quá nhiều.
- **Độ tin cậy**: Mất mát thỉnh thoảng một vài điểm dữ liệu là có thể chấp nhận được, nhưng hệ thống nói chung phải luôn sẵn sàng.
- **Tính nhất quán cuối cùng**: Kho lưu trữ dữ liệu vị trí không cần tính nhất quán mạnh. Độ trễ vài giây trong việc nhận dữ liệu vị trí ở các bản sao khác nhau là có thể chấp nhận được.

### **Ước tính sơ bộ**

Một số ước tính để xác định quy mô tiềm năng:
 * Bạn bè lân cận là bạn bè trong bán kính 5 dặm.
 * Khoảng thời gian làm mới vị trí là 30 giây. Tốc độ đi bộ của con người chậm, vì vậy không cần cập nhật vị trí quá thường xuyên.
 * Trung bình, 100 triệu người dùng sử dụng tính năng này mỗi ngày với 10% người dùng đồng thời, tức là 10 triệu.
 * Trung bình, một người dùng có 400 người bạn, tất cả họ đều sử dụng tính năng bạn bè lân cận.
 * Ứng dụng hiển thị 20 bạn bè lân cận trên mỗi trang.
 * **QPS cập nhật vị trí** = 10 triệu / 30 == ~334k cập nhật mỗi giây.

---

## Bước 2: Đề xuất thiết kế ở mức cao và đạt được sự đồng thuận

Trước khi tìm hiểu thiết kế API và mô hình dữ liệu, chúng ta sẽ nghiên cứu giao thức truyền thông sẽ sử dụng vì nó ít phổ biến hơn mô hình truyền thông yêu cầu-phản hồi truyền thống.

### **Thiết kế ở mức cao**

Ở mức độ cao, chúng ta muốn thiết lập việc truyền tin nhắn hiệu quả giữa các bên (peers). Điều này có thể được thực hiện qua giao thức ngang hàng (peer-to-peer), nhưng điều đó không thực tế cho một ứng dụng di động với kết nối chập chờn và các ràng buộc nghiêm ngặt về tiêu thụ năng lượng.

Một cách tiếp cận thực tế hơn là sử dụng một backend chung làm cơ chế fan-out (phân phối) tới những người bạn mà bạn muốn tiếp cận:

<div style="margin-left:3rem">
    <img src="./images/fan-out-backend.png" alt="fan-out-backend" width="500" />
</div>

Backend làm nhiệm vụ gì?
 * Nhận cập nhật vị trí từ tất cả người dùng đang hoạt động.
 * Đối với mỗi bản cập nhật vị trí, tìm tất cả người dùng đang hoạt động sẽ nhận được nó và chuyển tiếp cho họ.
 * Không chuyển tiếp dữ liệu vị trí nếu khoảng cách giữa những người bạn vượt quá ngưỡng cấu hình.

Nghe có vẻ đơn giản nhưng thách thức là thiết kế hệ thống cho quy mô mà chúng ta đang vận hành.

Chúng ta sẽ bắt đầu với một thiết kế đơn giản trước và thảo luận về cách tiếp cận nâng cao hơn trong phần đi sâu chi tiết:

<div style="margin-left:3rem">
    <img src="./images/simple-high-level-design.png" alt="simple-high-level-design" width="500" />
</div>

- **Bộ cân bằng tải (Load balancer)**: phân bổ lưu lượng truy cập qua các máy chủ REST API cũng như các máy chủ web socket hai chiều.
- **Máy chủ REST API**: xử lý các tác vụ phụ trợ như quản lý bạn bè, cập nhật hồ sơ, v.v.
- **Máy chủ Websocket**: máy chủ có trạng thái (stateful), chuyển tiếp các yêu cầu cập nhật vị trí đến các client tương ứng. Nó cũng quản lý việc cung cấp cho client di động vị trí của bạn bè lân cận khi khởi tạo (sẽ thảo luận chi tiết sau).
- **Redis location cache**: được sử dụng để lưu trữ dữ liệu vị trí gần đây nhất cho mỗi người dùng đang hoạt động. Có một giá trị TTL được thiết lập cho mỗi mục nhập trong cache. Khi TTL hết hạn, người dùng không còn hoạt động và dữ liệu của họ sẽ bị xóa khỏi cache.
- **Cơ sở dữ liệu người dùng (User database)**: lưu trữ dữ liệu người dùng và tình bạn. Có thể sử dụng cơ sở dữ liệu quan hệ hoặc NoSQL cho mục đích này.
- **Cơ sở dữ liệu lịch sử vị trí (Location history database)**: lưu trữ lịch sử dữ liệu vị trí người dùng, không nhất thiết được sử dụng trực tiếp trong tính năng bạn bè lân cận, mà thay vào đó được sử dụng để theo dõi dữ liệu lịch sử cho mục đích phân tích.
- **Redis pubsub**: được sử dụng như một bus tin nhắn nhẹ cho phép các chủ đề (topics) khác nhau cho mỗi kênh người dùng để cập nhật vị trí.

<div style="margin-left:3rem">
    <img src="./images/redis-pubsub-usage.png" alt="redis-pubsub-usage" width="500" />
</div>

Trong ví dụ trên, các máy chủ websocket đăng ký các kênh của những người dùng đang kết nối với chúng và chuyển tiếp các cập nhật vị trí bất cứ khi nào chúng nhận được cho những người dùng phù hợp.

### **Cập nhật vị trí định kỳ**

Dưới đây là cách luồng cập nhật vị trí định kỳ hoạt động:

<div style="margin-left:3rem">
    <img src="./images/periodic-location-update.png" alt="periodic-location-update" width="500" />
</div>

 * Client di động gửi cập nhật vị trí đến bộ cân bằng tải.
 * Bộ cân bằng tải chuyển tiếp cập nhật vị trí đến kết nối liên tục của máy chủ websocket cho client đó.
 * Máy chủ websocket lưu dữ liệu vị trí vào cơ sở dữ liệu lịch sử vị trí.
 * Dữ liệu vị trí được cập nhật trong location cache. Máy chủ websocket cũng lưu dữ liệu vị trí trong bộ nhớ (in-memory) để tính toán khoảng cách sau đó cho người dùng đó.
 * Máy chủ websocket xuất bản dữ liệu vị trí trong kênh của người dùng qua Redis pub sub.
 * Redis pubsub phát sóng cập nhật vị trí đến tất cả những người đăng ký kênh người dùng đó, tức là các máy chủ chịu trách nhiệm về bạn bè của người dùng đó.
 * Các máy chủ web socket đã đăng ký nhận được bản cập nhật vị trí, tính toán xem bản cập nhật nên được gửi đến người dùng nào và gửi đi.

Dưới đây là phiên bản chi tiết hơn của cùng một luồng:

<div style="margin-left:3rem">
    <img src="./images/detailed-periodic-location-update.png" alt="detailed-periodic-location-update" width="500" />
</div>

Trung bình, sẽ có 40 cập nhật vị trí cần chuyển tiếp vì một người dùng có trung bình 400 người bạn và 10% trong số họ trực tuyến tại một thời điểm.

### **Thiết kế API**

Các routine Websocket chúng ta cần hỗ trợ:
 * Cập nhật vị trí định kỳ - người dùng gửi dữ liệu vị trí đến máy chủ websocket.
 * Client nhận cập nhật vị trí - máy chủ gửi dữ liệu vị trí của bạn bè và mốc thời gian.
 * Khởi tạo client websocket - client gửi vị trí người dùng, máy chủ gửi lại dữ liệu vị trí của bạn bè lân cận.
 * Đăng ký một người bạn mới - máy chủ websocket gửi một ID người bạn mà client di động phải theo dõi, ví dụ: khi người bạn đó trực tuyến lần đầu tiên.
 * Hủy đăng ký một người bạn - máy chủ websocket gửi một ID người bạn, client di động phải hủy đăng ký do ví dụ người bạn đó ngoại tuyến.

HTTP API - các payload yêu cầu/phản hồi truyền thống cho các trách nhiệm phụ trợ.

### **Mô hình dữ liệu**

 * Location cache sẽ lưu trữ ánh xạ giữa `user_id` và `lat,long,timestamp`. Redis là một lựa chọn tuyệt vời cho cache này vì chúng ta chỉ quan tâm đến vị trí hiện tại và nó hỗ trợ thu hồi dựa trên TTL mà chúng ta cần cho trường hợp sử dụng này.
 * Bảng lịch sử vị trí lưu trữ cùng một dữ liệu nhưng trong một bảng quan hệ với bốn cột nêu trên. Cassandra có thể được sử dụng cho dữ liệu này vì nó được tối ưu hóa cho các tác vụ ghi nặng.

---

## Bước 3: Thiết kế chi tiết

Hãy thảo luận về cách chúng ta mở rộng thiết kế ở mức cao để nó hoạt động ở quy mô mà chúng ta đang nhắm tới.

### **Mỗi thành phần mở rộng tốt như thế nào?**

- **API servers**: có thể dễ dàng mở rộng thông qua các nhóm tự động mở rộng (autoscaling groups) và sao chép các phiên bản máy chủ.
- **Websocket servers**: chúng ta có thể dễ dàng mở rộng các máy chủ ws, nhưng chúng ta cần đảm bảo đóng các kết nối hiện có một cách duyên dáng khi gỡ bỏ một máy chủ. Ví dụ: chúng ta có thể đánh dấu một máy chủ là "draining" (đang rút) trong bộ cân bằng tải và ngừng gửi các kết nối đến nó trước khi cuối cùng bị xóa khỏi nhóm máy chủ.
- **Khởi tạo client**: khi một client lần đầu tiên kết nối với một máy chủ, nó lấy danh sách bạn bè của người dùng, đăng ký các kênh của họ trên Redis pubsub, lấy vị trí của họ từ cache và cuối cùng chuyển tiếp đến client.
- **Cơ sở dữ liệu người dùng (User database)**: Chúng ta có thể sharding cơ sở dữ liệu dựa trên user_id. Cũng có thể hợp lý khi để dữ liệu người dùng/bạn bè qua một dịch vụ và API chuyên dụng, được quản lý bởi một nhóm riêng.
- **Location cache**: Chúng ta có thể sharding cache dễ dàng bằng cách triển khai nhiều nút Redis. Ngoài ra, TTL đặt giới hạn cho bộ nhớ tối đa mà chúng ta có thể chiếm dụng tại một thời điểm. Nhưng chúng ta vẫn muốn xử lý tải ghi lớn.
- **Máy chủ Redis pub/sub**: chúng ta tận dụng thực tế là không tốn bộ nhớ nếu có các kênh được khởi tạo nhưng không được sử dụng. Do đó, chúng ta có thể phân bổ trước các kênh cho tất cả người dùng sử dụng tính năng bạn bè lân cận để tránh phải xử lý ví dụ như việc tạo một kênh mới khi một người dùng trực tuyến và thông báo cho các máy chủ websocket đang hoạt động.

### **Đi sâu vào việc mở rộng thành phần Redis pub/sub**

Chúng ta sẽ cần khoảng 200GB bộ nhớ để duy trì tất cả các kênh pub/sub. Điều này có thể đạt được bằng cách sử dụng 2 máy chủ Redis với mỗi máy 100GB.

Tuy nhiên, với việc chúng ta cần đẩy khoảng 14 triệu cập nhật vị trí mỗi giây, chúng ta sẽ cần ít nhất 140 máy chủ Redis để xử lý lượng tải đó, giả sử rằng một máy chủ duy nhất có thể xử lý khoảng 100k lượt đẩy mỗi giây.

Do đó, chúng ta sẽ cần một cụm máy chủ Redis phân tán để xử lý tải CPU cực lớn này.

Để hỗ trợ một cụm Redis phân tán, chúng ta sẽ cần sử dụng một thành phần khám phá dịch vụ (service discovery), chẳng hạn như Zookeeper hoặc etcd, để theo dõi máy chủ nào đang hoạt động.

Dữ liệu chúng ta cần mã hóa trong thành phần khám phá dịch vụ là:

<div style="margin-left:3rem">
    <img src="./images/channel-distribution-data.png" alt="channel-distribution-data" width="500" />
</div>

Các máy chủ web socket sử dụng dữ liệu mã hóa đó, được lấy từ Zookeeper để xác định một kênh cụ thể nằm ở đâu. Để hiệu quả, dữ liệu vòng hash (hash ring) có thể được lưu vào cache trong bộ nhớ trên mỗi máy chủ websocket.

Về việc mở rộng cụm máy chủ lên hoặc xuống, chúng ta có thể thiết lập một công việc hàng ngày để mở rộng cụm khi cần dựa trên dữ liệu lưu lượng lịch sử. Chúng ta cũng có thể dự phòng dư thừa cụm để xử lý các đợt tăng vọt tải.

Cụm Redis có thể được coi là một máy chủ lưu trữ có trạng thái vì có một số trạng thái được duy trì cho các kênh và cần có sự phối hợp với những người đăng ký để họ chuyển giao cho các nút mới được cung cấp trong cụm.

Chúng ta phải lưu ý một số vấn đề tiềm ẩn trong quá trình mở rộng:
 * Sẽ có rất nhiều yêu cầu đăng ký lại từ các máy chủ web socket do các kênh bị di chuyển xung quanh.
 * Một số cập nhật vị trí có thể bị mất từ các client trong quá trình vận hành, điều này có thể chấp nhận được cho bài toán này, nhưng chúng ta vẫn nên giảm thiểu việc đó xảy ra. Cân nhắc thực hiện các hoạt động như vậy khi lưu lượng truy cập ở điểm thấp nhất trong ngày.
 * Chúng ta có thể tận dụng consistent hashing để giảm thiểu lượng kênh bị di chuyển trong trường hợp thêm/xóa máy chủ.

<div style="margin-left:3rem">
    <img src="./images/consistent-hashing.png" alt="consistent-hashing" width="500" />
</div>

### **Thêm/xóa bạn bè**

Bất cứ khi nào một người bạn được thêm/xóa, máy chủ websocket chịu trách nhiệm cho người dùng bị ảnh hưởng cần đăng ký/hủy đăng ký khỏi kênh của người bạn đó.

Vì tính năng "bạn bè lân cận" là một phần của một ứng dụng lớn hơn, chúng ta có thể giả định rằng một callback phía client di động có thể được đăng ký bất cứ khi nào bất kỳ sự kiện nào xảy ra và client sẽ gửi một tin nhắn đến máy chủ websocket để thực hiện hành động thích hợp.

### **Người dùng có nhiều bạn bè**

Chúng ta có thể đặt giới hạn cho tổng số bạn bè mà một người có thể có, ví dụ Facebook có giới hạn tối đa 5000 bạn bè.

Máy chủ websocket xử lý người dùng "cá voi" (whale user) có thể có tải cao hơn, nhưng miễn là chúng ta có đủ máy chủ web socket, mọi thứ sẽ ổn.

### **Những người lạ lân cận**

Nếu người phỏng vấn muốn cập nhật thiết kế để bao gồm một tính năng mà thỉnh thoảng chúng ta có thể thấy một người ngẫu nhiên hiện lên trên bản đồ bạn bè lân cận của mình thì sao?

Một cách để xử lý việc này là xác định một nhóm các kênh pubsub, dựa trên geohash:

<div style="margin-left:3rem">
    <img src="./images/geohash-pubsub.png" alt="geohash-pubsub" width="500" />
</div>

Bất kỳ ai trong geohash đó sẽ đăng ký kênh phù hợp để nhận các cập nhật vị trí cho những người dùng ngẫu nhiên:

<div style="margin-left:3rem">
    <img src="./images/location-updates-geohash.png" alt="location-updates-geohash" width="500" />
</div>

Chúng ta cũng có thể đăng ký nhiều geohash để xử lý các trường hợp ai đó ở gần nhưng nằm trong một geohash giáp ranh:

<div style="margin-left:3rem">
    <img src="./images/geohash-borders.png" alt="geohash-borders" width="500" />
</div>

### **Lựa chọn thay thế cho Redis pub/sub**

Một lựa chọn thay thế cho việc sử dụng Redis cho pub/sub là tận dụng Erlang - một ngôn ngữ lập trình đa năng, được tối ưu hóa cho các ứng dụng tính toán phân tán.

Với nó, chúng ta có thể tạo ra hàng triệu tiến trình Erlang nhỏ giao tiếp với nhau. Chúng ta có thể xử lý cả các kết nối websocket và các kênh pub/sub trong ứng dụng Erlang phân tán.

Tuy nhiên, thách thức khi sử dụng Erlang là nó là một ngôn ngữ lập trình đặc thù và có thể khó tìm được các nhà phát triển Erlang giỏi.

---

## Bước 4: Tổng kết

Chúng ta đã thiết kế thành công một hệ thống hỗ trợ các tính năng bạn bè lân cận.

Các thành phần cốt lõi:
- **Máy chủ Web socket**: giao tiếp thời gian thực giữa client và máy chủ.
- **Redis**: đọc và ghi nhanh dữ liệu vị trí + các kênh pub/sub.

Chúng ta cũng đã tìm hiểu cách mở rộng các máy chủ restful api, máy chủ websocket, lớp dữ liệu, máy chủ Redis pub/sub và chúng ta cũng đã khám phá một lựa chọn thay thế cho việc sử dụng Redis Pub/Sub. Chúng ta cũng đã tìm hiểu về tính năng "người lạ ngẫu nhiên lân cận".
