# Chương 14: Thiết kế YouTube

## Giới thiệu
YouTube là một nền tảng phát video trực tuyến khổng lồ hỗ trợ tải lên video, phát lại và các tương tác khác nhau. Chương này tập trung vào việc thiết kế một hệ thống phát video có khả năng mở rộng với các tính năng cốt lõi sau:
- **Tải lên video nhanh chóng**
- **Phát video mượt mà**
- **Khả năng thay đổi chất lượng video**
- **Chi phí cơ sở hạ tầng thấp**
- **Tính khả dụng và độ tin cậy cao**

### Số liệu thống kê chính (2020)
- **2 tỷ người dùng hoạt động hàng tháng**
- **5 tỷ video được xem mỗi ngày**
- **37% lưu lượng internet di động đến từ YouTube**
- Hỗ trợ **80 ngôn ngữ**
- **15,1 tỷ USD doanh thu quảng cáo** vào năm 2019

---

## Bước 1: Hiểu vấn đề và xác định phạm vi

### Các chức năng cốt lõi
1. Tải lên video
2. Xem video

### Các nền tảng hỗ trợ
- Ứng dụng di động, trình duyệt web và TV thông minh

### Giả định
- **Người dùng hoạt động hàng ngày (DAU):** 5 triệu
- **Kích thước video trung bình:** 300 MB
- **Giới hạn tải lên:** Tối đa 1 GB mỗi video
- **Nhu cầu lưu trữ hàng ngày:** 150 TB
- **Chi phí CDN:** 5 triệu * 5 video * 0,3GB * $0,02 = $150,000/ngày (sử dụng Amazon CloudFront)

---

## Bước 2: Thiết kế ở mức cao (High-Level Design)

### Các thành phần

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="High Level Design" width="400">
</div>

1. **Client:** Các thiết bị như điện thoại thông minh, máy tính và TV.
2. **CDN (Content Delivery Network):** Lưu trữ và phát video.
3. **API Servers:** Xử lý tất cả các tương tác của người dùng ngoại trừ phát video (ví dụ: tải lên, cập nhật metadata).
4. **Metadata Database:** Lưu trữ metadata của video (ví dụ: tiêu đề, mô tả, kích thước).
5. **Original Storage:** Lưu trữ Blob cho các video gốc được tải lên.
6. **Transcoding Servers:** Chuyển đổi video sang nhiều độ phân giải và định dạng khác nhau.
7. **Transcoded Storage:** Lưu trữ Blob cho các video đã được chuyển mã.


---

### Luồng công việc cốt lõi
#### 1. Luồng tải lên video
- **Quá trình song song:**
  1. Tải video lên bộ lưu trữ gốc.
  2. Cập nhật metadata của video trong cơ sở dữ liệu.

- **Tải lên video (Các bước):**

    <div style="margin-left:3rem">
        <img src="./images/video-uploading-flow.png" alt="Video Upload Flow" width="500">
    </div>

    - [1] Video được tải lên bộ lưu trữ blob. 
    - [2] Các máy chủ chuyển mã (transcoding) chuyển đổi video sang nhiều định dạng.
    - [3] Khi quá trình chuyển mã hoàn tất, hai bước sau được thực hiện song song.
        - [3a] Các video đã chuyển mã được gửi đến bộ lưu trữ video đã chuyển mã.
        - [3b] Các sự kiện hoàn tất chuyển mã được đưa vào hàng đợi hoàn tất. 
    - [3a.1] Video được phân phối đến CDN. 
    - [3b.1] Các bộ xử lý hoàn tất cập nhật metadata và thông báo cho người dùng. 



- **Tải lên Metadata (Các bước):**

    <div style="margin-left:3rem">
        <img src="./images/metadata-upload.png" alt="Metadata Upload" height="500">
    </div>

    - Client gửi một yêu cầu song song để cập nhật metadata của video.
    - Yêu cầu chứa metadata của video, bao gồm tên tệp, kích thước, định dạng, v.v.
    
       


#### 2. Luồng phát video (Streaming)

<div style="margin-left: 3em;">
  <img src="./images/video-streaming-flow.png" alt="Video Streaming Flow" height="400">
</div>

- Video được phát trực tiếp từ CDN bằng các máy chủ biên (edge servers) để giảm thiểu độ trễ.
- Một số giao thức phát trực tuyến phổ biến là MPEG_DASH, Apple HLS, Adobe HDS.
- *Các giao thức phát trực tuyến khác nhau hỗ trợ các kiểu mã hóa video và trình phát lại khác nhau.*

---

## Bước 3: Thiết kế chi tiết

### Chuyển mã video (Video Transcoding)
#### Tầm quan trọng
1. Video thô chiếm dung lượng lưu trữ lớn. Chuyển mã giúp giảm không gian lưu trữ.
2. Đảm bảo tính tương thích trên các thiết bị và trình duyệt khác nhau.
3. Điều chỉnh chất lượng video theo điều kiện mạng.

#### Các thành phần
- **Container:** Chứa video, âm thanh và metadata (ví dụ: MP4, AVI).
- **Codecs:** Các thuật toán nén và giải nén (ví dụ: H.264, VP9).

#### Mô hình Đồ thị có hướng không chu trình (DAG)
<div style="margin-left: 3em;">
    <img src="./images/dag-video-transcoding.png" alt="DAG Video Transcoding" width="600">
</div>

- Việc chuyển mã một video tiêu tốn nhiều tài nguyên tính toán và thời gian.
- Mô hình DAG xác định các tác vụ như mã hóa, tạo ảnh thu nhỏ (thumbnail) và đóng dấu bản quyền (watermarking).
- Cho phép tính song song cao trong quá trình xử lý video.


- Video gốc được chia thành video, âm thanh và metadata. 
    - Mã hóa video: Video được chuyển đổi để hỗ trợ các độ phân giải, codec, bitrate khác nhau.
    - Thumbnail: Có thể do người dùng tải lên hoặc hệ thống tự động tạo ra.
    - Watermark: Hình ảnh đè lên video chứa thông tin nhận dạng video.

---

### Kiến trúc chuyển mã video

<div style="margin-left: 3em;">
<img src="./images/video-transcoding-architecture.png" alt="Video Transcoding" width="600">
</div>

1. **Preprocessor:** Chia video thành các phân đoạn nhỏ hơn (căn chỉnh GOP). Nó có 4 nhiệm vụ chính.

    <div style="margin-left: 3em;">
        <img src="./images/dag-config.png" alt="DAG Config" width="500">
    </div>

    - Chia nhỏ video: Luồng video được chia thành các Group of Pictures (GOP) nhỏ hơn.
    - Nó chia nhỏ video theo căn chỉnh GOP cho các client cũ.
    - Nó tạo ra DAG dựa trên các tệp cấu hình mà lập trình viên viết. 
    - Nó lưu trữ các GOP và metadata trong bộ lưu trữ tạm thời trong trường hợp việc mã hóa thất bại, hệ thống có thể sử dụng dữ liệu đã lưu để thử lại.


2. **DAG Scheduler:** Tổ chức các tác vụ thành các giai đoạn tuần tự hoặc song song.
    <div style="margin-left: 3em;">
        <img src="./images/dag-scheduler.png" alt="DAG Scheduler" width="500">
    </div>

    - Nó chia biểu đồ DAG thành các giai đoạn của tác vụ và đưa chúng vào hàng đợi tác vụ trong bộ quản lý tài nguyên. 
    - Giai đoạn 1: video, âm thanh và metadata.
    - Tệp video được chia nhỏ tiếp thành hai tác vụ ở giai đoạn 2: mã hóa video và tạo thumbnail. 


3. **Resource Manager:** Chịu trách nhiệm quản lý hiệu quả việc phân bổ tài nguyên. Nó bao gồm 3 hàng đợi và một bộ lập lịch tác vụ.
    <div style="margin-left: 3em;">
        <img src="./images/resource-manager.png" alt="Resource Manager" width="700">
    </div>

    - Task queue: Hàng đợi ưu tiên chứa các tác vụ cần thực hiện.
    - Worker queue: Hàng đợi ưu tiên chứa thông tin sử dụng của các worker.
    - Running queue: Chứa các tác vụ đang chạy và các worker đang thực hiện chúng.
    - Task scheduler: Chọn tác vụ/worker tối ưu và hướng dẫn worker được chọn thực hiện công việc.


4. **Task Workers:** Thực hiện việc chuyển mã và các hoạt động khác.
    <div style="margin-left: 3em;">
        <img src="./images/task-worker.png" alt="Task Worker" width="250">
   </div>

    - Các worker khác nhau có thể chạy các tác vụ khác nhau.


5. **Temporary Storage:** Lưu trữ dữ liệu trung gian để thử lại khi cần.
    - Việc lựa chọn hệ thống lưu trữ phụ thuộc vào các yếu tố như loại dữ liệu, kích thước dữ liệu, tần suất truy cập, tuổi thọ dữ liệu, v.v. 
6. **Output:** Các video đã chuyển mã sẵn sàng để phân phối.


---

## Tối ưu hóa hệ thống

### Tối ưu hóa tốc độ
1. **Tải lên video song song:** Chia nhỏ video thành các phần nhỏ để tải lên nhanh hơn và có thể tiếp tục khi bị gián đoạn.

    <img src="./images/video-split.png" alt="Video Split" width="600">

2. **Trung tâm tải lên phân tán:** Sử dụng CDN làm trung tâm tải lên gần với người dùng.
3. **Xử lý song song:** Tách biệt các module bằng hàng đợi tin nhắn để đạt được tính song song cao.

    <img src="./images/message-queue1.png" alt="Message Queue" width="600">
    <img src="./images/message-queue2.png" alt="Message Queue" height="170" width="500">

### Tối ưu hóa an toàn
1. **Pre-Signed URLs:** Giới hạn việc tải video cho những người dùng được ủy quyền.

    <img src="./images/pres-signed-urls.png" alt="Pre Signed" width="500">

2. **Bảo vệ video:**
   - **Hệ thống DRM** (ví dụ: Apple FairPlay, Google Widevine).
   - **Mã hóa AES.**
   - **Đóng dấu bản quyền (Watermarking).**

### Tối ưu hóa tiết kiệm chi phí
1. Chỉ phân phối các video phổ biến qua CDN; các video ít phổ biến hơn được phân phối từ các máy chủ dung lượng cao.
2. Mã hóa theo yêu cầu đối với các video hiếm khi được truy cập.
3. Phân phối video theo khu vực dựa trên mức độ phổ biến.
4. Xây dựng CDN tùy chỉnh và hợp tác với các ISP để giảm chi phí băng thông.

---

## Xử lý lỗi
### Lỗi có thể phục hồi
- Thử lại các tác vụ tải lên, chuyển mã hoặc phân bổ tài nguyên bị lỗi.

### Lỗi không thể phục hồi
- Ngừng xử lý video bị lỗi định dạng và trả về mã lỗi.
