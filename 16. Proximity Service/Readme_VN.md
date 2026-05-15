# Chương 16: Dịch vụ lân cận (Proximity Service)

## Giới thiệu
Một **dịch vụ lân cận** được thiết kế để tìm các địa điểm ở gần, chẳng hạn như nhà hàng, khách sạn, trạm xăng và các doanh nghiệp khác. Chức năng này được sử dụng trong các ứng dụng như **Google Maps** và **Yelp** để giúp người dùng khám phá các địa điểm trong một bán kính xác định.


## Bước 1: Hiểu vấn đề và xác định phạm vi

### **Yêu cầu chức năng**
1. **Tìm kiếm doanh nghiệp** dựa trên vị trí người dùng (vĩ độ, kinh độ) và bán kính tìm kiếm.
2. **Cho phép chủ doanh nghiệp** thêm, cập nhật hoặc xóa doanh nghiệp (không cần thời gian thực).
3. **Cung cấp thông tin chi tiết về doanh nghiệp** khi được yêu cầu.

### **Yêu cầu phi chức năng**
- **Độ trễ thấp**: Người dùng sẽ nhận được phản hồi nhanh chóng.
- **Quyền riêng tư dữ liệu**: Tuân thủ các quy định GDPR và CCPA.
- **Tính khả dụng cao**: Xử lý các đợt tăng đột biến trong giờ cao điểm tại các địa điểm đông đúc.

### **Ước tính sơ bộ**
- **100 triệu người dùng hoạt động hàng ngày**.
- **200 triệu doanh nghiệp** trong hệ thống.
- **Tính toán QPS tìm kiếm**:
  - Người dùng thực hiện **5 lần tìm kiếm mỗi ngày**.
  - **QPS tìm kiếm** = (100M × 5) / 86.400 ≈ **5.000 QPS**.

---

## Bước 2: Thiết kế ở mức cao (High-Level Design)

### **Thiết kế API**
#### **Tìm kiếm doanh nghiệp lân cận**
GET /v1/search/nearby

- **Tham số yêu cầu**:
  - `latitude`: Vĩ độ vị trí người dùng.
  - `longitude`: Kinh độ vị trí người dùng.
  - `radius`: Bán kính tìm kiếm (mặc định: 5000m).

#### **Các API doanh nghiệp**
| API Endpoint                     | Mô tả                                      |
|-----------------------------------|--------------------------------------------------|
| `GET /v1/businesses/{id}`         | Lấy thông tin chi tiết doanh nghiệp             |
| `POST /v1/businesses`             | Thêm một doanh nghiệp mới                        |
| `PUT /v1/businesses/{id}`         | Cập nhật chi tiết doanh nghiệp                  |
| `DELETE /v1/businesses/{id}`      | Xóa một doanh nghiệp khỏi hệ thống               |


### **Mô hình dữ liệu**
- Vì khối lượng đọc cao do hai tính năng được sử dụng rất phổ biến, một cơ sở dữ liệu quan hệ như MySQL là một lựa chọn phù hợp.
  - Tìm kiếm doanh nghiệp lân cận
  - Xem thông tin chi tiết của một doanh nghiệp

### **Schema dữ liệu**
- Các bảng cơ sở dữ liệu chính là bảng business (doanh nghiệp) và bảng geospatial index (chỉ mục địa không gian).
- Bảng business chứa thông tin chi tiết về một doanh nghiệp.

### **Kiến trúc hệ thống ở mức cao**
Hệ thống bao gồm hai phần: Dịch vụ dựa trên vị trí (Location based service - LBS) và dịch vụ liên quan đến doanh nghiệp (business related service).

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="HLD" width="400" />
</div>

- **Location-Based Service (LBS)**: 
  - Xử lý các truy vấn tìm kiếm dựa trên vị trí.
  - Dịch vụ nặng về đọc (read-heavy) và không có yêu cầu ghi.
  - QPS cao, đặc biệt là trong giờ cao điểm ở các khu vực đông dân cư và hệ thống là không trạng thái (stateless).
- **Business Service**: Xử lý hai loại yêu cầu.
  - Chủ doanh nghiệp tạo, cập nhật hoặc xóa doanh nghiệp.
  - Khách hàng xem thông tin chi tiết về một doanh nghiệp.
- **Load Balancer**: Định tuyến lưu lượng đến dịch vụ LBS và Business.
- **Database Cluster**: 
  - Sử dụng **kiến trúc primary-replica** cho khối lượng công việc nặng về đọc.
  - Có thể có một số sai lệch giữa dữ liệu được đọc bởi LBS và dữ liệu được ghi bởi cơ sở dữ liệu chính (primary).
  - Sự không nhất quán này không phải là vấn đề vì thông tin doanh nghiệp không được cập nhật trong thời gian thực.


---

## Bước 3: Các thuật toán để lấy các doanh nghiệp lân cận

### **Lựa chọn 1: Tìm kiếm hai chiều (Cách tiếp cận thô sơ)**

<div style="margin-left:3rem">
    <img src="./images/2d-search.png" alt="2D" width="250" />
</div>

Cách trực quan nhất là vẽ một vòng tròn với bán kính xác định trước và tìm tất cả các doanh nghiệp trong vòng tròn đó.

**Truy vấn SQL:**
```
SELECT business_id, latitude, longitude
FROM business
WHERE (latitude BETWEEN :lat - radius AND :lat + radius)
AND (longitude BETWEEN :long - radius AND :long + radius);
```
**Vấn đề:**
- **Không hiệu quả**: Đòi hỏi phải quét toàn bộ cơ sở dữ liệu.
- **Bị giới hạn bởi các chỉ mục một chiều** (vĩ độ/kinh độ).

Một cải tiến tiềm năng là xây dựng chỉ mục trên các cột kinh độ và vĩ độ, mặc dù điều này tốt hơn một chút nhưng vẫn rất chậm.

### Cách tiếp cận tốt hơn
- Vấn đề với cách tiếp cận trước đó là chỉ mục cơ sở dữ liệu chỉ có thể tăng tốc độ tìm kiếm theo một chiều.
- Một cách tiếp cận tối ưu là biểu diễn dữ liệu hai chiều thành một chiều bằng cách sử dụng đánh chỉ mục địa không gian (geospatial indexing).
  - Hash: Even grid, Geo Hash
  - Tree: Quadtree, Google S2, RTree

  <div style="margin-left:3rem">
    <img src="./images/geospatial-index-types.png" alt="2D" width="500" />
  </div>


### **Lựa chọn 2: Lưới chia đều (Evenly Divided Grid)**

  <div style="margin-left:3rem">
    <img src="./images/even-grid.png" alt="Even Grid" width="400" />
  </div>

- **Chia thế giới thành các lưới có kích thước cố định**.
- **Vấn đề**: Phân bố doanh nghiệp không đều (mật độ cao ở thành phố, thưa thớt ở nông thôn).

### **Lựa chọn 3: Geohash**
- Chia hành tinh thành bốn phần tư dọc theo kinh tuyến gốc và đường xích đạo. Sau đó chia mỗi lưới thành bốn lưới nhỏ hơn. 
- Mỗi lưới có thể được biểu diễn bằng cách xen kẽ giữa các bit kinh độ và vĩ độ.
- Lặp lại việc chia nhỏ này.

  <div style="margin-left:3rem">
    <img src="./images/geohash.png" alt="Geohash" width="300" />
    <img src="./images/geohash-1.png" alt="Geohash" width="285" />
  </div>


- **Mã hóa vĩ độ và kinh độ thành một chuỗi ký tự chữ số duy nhất**. Nó có 12 mức độ chính xác.
- **Cấu trúc lưới phân cấp** cho phép tìm kiếm hiệu quả.
- Độ chính xác phù hợp được chọn bằng cách sử dụng độ dài geohash tối thiểu theo bảng.
  <div style="margin-left:3rem">
    <img src="./images/geohash-radius-mapping.png" alt="Geohash Radius" width="400" />
  </div>
- Geohash đảm bảo rằng tiền tố chung giữa hai geohash càng dài thì chúng càng gần nhau.

- **Thách thức**:
  <div style="margin-left:3rem">
    <img src="./images/boundary-issue.png" alt="Boundary Issue" width="300" />
  </div>

  - **Vấn đề ranh giới** (các doanh nghiệp gần mép lưới có thể bị loại trừ).
    - Hai vị trí có thể rất gần nhau nhưng không có tiền tố chung nào cả (có thể ở hai phía khác nhau của đường xích đạo).
    - Hai vị trí có thể có tiền tố chung dài nhưng lại thuộc các geohash khác nhau.
  - Giải pháp: Cần tìm kiếm các lưới lân cận.


### **Lựa chọn 4: Quadtree**

  Quadtree là một cấu trúc dữ liệu cây chia không gian hai chiều một cách đệ quy thành bốn phần tư, với mỗi nút nội bộ có chính xác bốn nút con, đại diện cho bốn vùng phụ của không gian.
  - Quadtree là một cấu trúc dữ liệu trong bộ nhớ (in-memory) và nó chạy trên mỗi máy chủ LBS và được xây dựng khi khởi động máy chủ.

  <div style="margin-left:3rem">
    <img src="./images/quadtree.png" alt="Quadtree" width="500" />
  </div>

  - Nút gốc được chia nhỏ đệ quy thành 4 phần tư cho đến khi không còn nút nào có nhiều hơn x số lượng doanh nghiệp (trong trường hợp này là 100).

  <div style="margin-left:3rem">
    <img src="./images/building-quadtree.png" alt="Building Quadtree" width="500" />
  </div>

- Chỉ mục quadtree không tốn quá nhiều bộ nhớ (thường tính bằng GB) và có thể dễ dàng nằm gọn trong một máy chủ.
- Vì độ phức tạp thời gian để xây dựng cây là nlogn, có thể mất vài phút để xây dựng cây.
- **Hiệu quả cho các truy vấn tìm kiếm k-vị trí gần nhất** (ví dụ: tìm trạm xăng gần nhất).

  <div style="margin-left:3rem">
    <img src="./images/realworld-quadtree.png" alt="Real World Quadtree" width="400" />
  </div>

#### Cân nhắc vận hành
 - Đối với khoảng 200 triệu doanh nghiệp, có thể mất vài phút để xây dựng quadtree khi khởi động máy chủ.
 - Trong khi quadtree đang được xây dựng, nó không thể phục vụ lưu lượng truy cập, do đó một bản phát hành mới nên được triển khai dần dần cho một tập hợp con các máy chủ.
 - Khi cập nhật một doanh nghiệp hoặc thêm mới, cách tiếp cận dễ nhất là xây dựng lại quadtree một cách tăng tiến (dẫn đến nhiều việc vô hiệu hóa bộ nhớ cache).
 - Cũng có thể cập nhật quadtree ngay lập tức nhưng phức tạp hơn để triển khai (cần cơ chế khóa).

### **Lựa chọn 5: Google S2**
Nó ánh xạ một hình cầu thành một chỉ mục 1D dựa trên đường cong Hilbert. Hai điểm gần nhau trên đường cong Hilbert sẽ gần nhau trong không gian 1D.


  <div style="margin-left:3rem">
    <img src="./images/hilbert-curve.png" alt="Hilbert curve" width="300" />
    <img src="./images/geofence.png" alt="Geofence" width="355" />
  </div>

- **Chia trái đất thành các ô nhỏ bằng đường cong Hilbert**.
- Tuyệt vời cho geofencing (tạo hàng rào địa lý) vì nó có thể bao phủ các khu vực tùy ý với các cấp độ khác nhau.
- Geofencing cũng cho phép xác định các tham số bao quanh khu vực quan tâm.
- Một lợi thế khác là thay vì có một mức độ chính xác cố định, chúng ta có thể chỉ định mức độ tối thiểu, tối đa và số ô tối đa trong S2.


## So sánh đánh đổi 

#### Geohash
- Dễ sử dụng và triển khai - Không cần xây dựng/xây dựng lại cây.
- Hỗ trợ kết quả bán kính cố định.
- Cập nhật chỉ mục dễ dàng.
- Không thể điều chỉnh linh hoạt kích thước lưới dựa trên mật độ dân số.

#### Quadtree
- Khó triển khai hơn một chút.
- Hỗ trợ tìm nạp k doanh nghiệp gần nhất.
- Có thể điều chỉnh linh hoạt kích thước lưới dựa trên mật độ dân số.
- Cập nhật chỉ mục phức tạp hơn vì có thể cần xây dựng lại toàn bộ cây.

---

## Bước 4: Mở rộng cơ sở dữ liệu và chiến lược lưu trữ cache

### **Mở rộng bảng doanh nghiệp**
- **Sharding theo ID doanh nghiệp** đảm bảo phân phối dữ liệu đồng đều.
- Chúng ta có các hàng riêng biệt cho mỗi doanh nghiệp trong bảng.

| Geohash | Business ID |
|---------|------------|
| 9q9hvu  | 343        |
| 9q9hvu  | 347        |
| 9q9hvu  | 112        |

### **Mở rộng chỉ mục địa không gian**
- Có thể không phù hợp cho bảng geohash. Trong trường hợp này, mọi thứ có thể nằm gọn trong một máy chủ nên không có lý do kỹ thuật nào để sharding.
- Một cách tiếp cận tốt hơn là có các bản sao đọc (read-replicas) để hỗ trợ tải đọc.



---

### **Chiến lược Cache**
Lựa chọn khóa cache rõ ràng nhất là tọa độ vị trí, tuy nhiên nó có một vài vấn đề:
 - Tọa độ vị trí từ gps không chính xác tuyệt đối.
 - Người dùng có thể di chuyển làm tọa độ vị trí thay đổi.
 - Một khóa tốt hơn là geohash.

| Khóa Cache  | Giá trị Cache |
|------------|------------|
| `geohash`  | Danh sách các ID doanh nghiệp trong lưới đó |
| `business_id` | Chi tiết doanh nghiệp (tên, địa chỉ, đánh giá, v.v.) |

---

## Bước 5: Chiến lược triển khai và kiến trúc cuối cùng

### **Vùng (Region) và Vùng khả dụng (Availability Zones)**
- Triển khai LBS và Business Service **trên nhiều vùng**.

### **Xử lý cập nhật thời gian thực**
- **Các cập nhật doanh nghiệp được xử lý theo lô hàng ngày**.

### **Kiến trúc hệ thống cuối cùng**


  <div style="margin-left:3rem">
    <img src="./images/final-design.png" alt="Final Design" width="500" />
  </div>


Thuật toán cuối cùng trông như thế này:

## Các bước để truy xuất các doanh nghiệp lân cận
1. **Yêu cầu của người dùng:**  
   - Người dùng tìm kiếm các nhà hàng trong vòng **500 mét**.  
   - Client gửi **vĩ độ (37.776720), kinh độ (-122.416730) và bán kính (500m)** đến **bộ cân bằng tải**.

2. **Chuyển tiếp yêu cầu:**  
   - **Bộ cân bằng tải (LB)** chuyển tiếp yêu cầu đến **Dịch vụ dựa trên vị trí (LBS)**.

3. **Tính toán Geohash:**  
   - LBS xác định **độ dài geohash** khớp với bán kính.  
   - Sử dụng bảng tham chiếu, **500m tương ứng với độ dài geohash = 6**.

4. **Lấy các Geohash lân cận:**  
   - LBS tính toán **các geohash lân cận** để bao gồm các khu vực xung quanh.  
   - Kết quả là một danh sách:  
     ```
     [my_geohash, neighbor1_geohash, neighbor2_geohash, ..., neighbor8_geohash]
     ```

5. **Lấy các ID doanh nghiệp từ Redis:**  
   - Đối với mỗi geohash trong danh sách, LBS truy vấn **máy chủ Geohash Redis** để lấy **các ID doanh nghiệp**.  
   - Các truy vấn song song được sử dụng để giảm thiểu độ trễ.

6. **Truy xuất & Xếp hạng doanh nghiệp:**  
   - LBS lấy **chi tiết đầy đủ về doanh nghiệp** từ **máy chủ Business Info Redis**.  
   - Các doanh nghiệp được **sắp xếp theo khoảng cách** từ vị trí của người dùng.  
   - Các **kết quả đã xếp hạng** được gửi lại cho client.

## Các tối ưu hóa chính
- **Các cuộc gọi Redis song song**: Giảm thời gian phản hồi.  
- **Đánh chỉ mục Geohash**: Đảm bảo các truy vấn không gian hiệu quả.  
- **Caching**: Tăng tốc độ tra cứu và truy xuất dữ liệu doanh nghiệp.  

Phương pháp này đảm bảo việc truy xuất các doanh nghiệp gần vị trí người dùng **có độ trễ thấp và có khả năng mở rộng**.

---

### **Chọn phương pháp đánh chỉ mục tốt nhất**
| Phương pháp đánh chỉ mục | Ưu điểm | Nhược điểm |
|----------------|------|------|
| **Geohash** | Dễ triển khai, hiệu quả cho tìm kiếm lân cận | Vấn đề ranh giới, kích thước lưới cố định |
| **Quadtree** | Điều chỉnh linh hoạt theo mật độ, hỗ trợ truy vấn k-gần nhất | Phức tạp hơn, yêu cầu cân bằng lại cây |
| **Google S2** | Geofencing nâng cao, được sử dụng trong Google Maps | Khó triển khai hơn |

---

## Tài liệu tham khảo
1. [Thuật toán Geohash](https://www.movable-type.co.uk/scripts/geohash.html)
2. [Đánh chỉ mục Quadtree](https://en.wikipedia.org/wiki/Quadtree)
3. [Google S2 Geometry](https://s2geometry.io/)
