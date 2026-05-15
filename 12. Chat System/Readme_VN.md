# Chương 12: Thiết kế Hệ thống Chat (Chat System)

## Giới thiệu
**Hệ thống chat** hỗ trợ nhắn tin theo thời gian thực giữa những người dùng. Chương này tập trung vào việc thiết kế một ứng dụng chat bao gồm:
- **Chat 1-1 (One-on-One Chat)**
- **Chat Nhóm (tối đa 100 người dùng)**
- **Hiển thị trạng thái Trực tuyến/Ngoại tuyến**
- **Hỗ trợ đa thiết bị**
- **Thông báo đẩy (Push Notifications)**

Hệ thống nhắm mục tiêu **50 triệu người dùng hoạt động hàng ngày (DAU)** và lưu trữ lịch sử trò chuyện vĩnh viễn.

---

## Bước 1: Hiểu rõ vấn đề

### Yêu cầu
1. **Tính năng:**
   - Chat 1-1 và chat nhóm (tối đa 100 thành viên).
   - Tin nhắn văn bản (lên tới 100.000 ký tự).
   - Hiển thị trạng thái trực tuyến/ngoại tuyến.
   - Hỗ trợ cho nhiều thiết bị.
   - Thông báo đẩy.
2. **Quy mô:** Thiết kế cho 50 triệu DAU.
3. **Lưu trữ:** Lịch sử trò chuyện vĩnh viễn.

---

## Bước 2: Thiết kế tổng quan

### Giao thức giao tiếp
1. **Phía người gửi (Sender Side):** HTTP để gửi tin nhắn, tận dụng các kết nối liên tục (persistent connections) để đạt hiệu quả.

      <div style="margin-left:2rem">
      <img src="./images/basic-design.png" alt="Basic Design" width="500">    
      <div>

2. **Phía người nhận (Receiver Side):**
   - **Thăm dò ý kiến (Polling):**
      - Máy khách (client) định kỳ hỏi máy chủ xem có tin nhắn nào không.
      - Không hiệu quả do các yêu cầu dư thừa, thường xuyên.

         <img src="./images/polling.png" alt="Polling" width="400">    

   - **Thăm dò ý kiến dài (Long Polling):** 
      - Giữ kết nối mở cho đến khi tin nhắn đến. 
      - Không hiệu quả đối với người dùng không hoạt động.

         <img src="./images/long-polling.png" alt="Long Polling" width="400">

   - **WebSocket:** 
      - Một kết nối liên tục, hai chiều để giao tiếp theo thời gian thực, được chọn cho cả việc gửi và nhận tin nhắn.
      - Sử dụng giao thức WebSockets (ws) để gửi và nhận tin nhắn.

         <img src="./images/websocket.png" alt="Websocket"  width="400" >    
   
---

### Các thành phần

<div style="margin-left:5rem">
   <img src="./images/high-level-stateless-arch.png" alt="High Level Architecture" height="350">    
   <img src="./images/high-level-statefull-arch.png" alt="High Level Architecture" height="350" width="550">
</div>

1. **Dịch vụ phi trạng thái (Stateless Services):**
   - Xử lý đăng ký, đăng nhập và quản lý hồ sơ người dùng.
   - Tích hợp với tính năng khám phá dịch vụ (service discovery) để đề xuất máy chủ chat tốt nhất.
2. **Dịch vụ có trạng thái (Stateful Services):**
   - Các máy chủ chat duy trì các kết nối WebSocket liên tục.
   - Chịu trách nhiệm gửi và đồng bộ hóa tin nhắn.
3. **Tích hợp bên thứ ba (Third-Party Integration):**
   - Các dịch vụ thông báo đẩy (push notification) thông báo cho người dùng về các tin nhắn mới.
   - Tham khảo chương Hệ thống thông báo (Notification System) để biết cách triển khai thông báo.

---
### Thiết kế

Máy khách duy trì một kết nối WebSocket liên tục tới máy chủ chat để nhắn tin theo thời gian thực.

<div style="margin-left:3rem">
      <img src="./images/high-level-design.png" alt="High Level Design" width="450"> 
</div>

- Máy chủ chat hỗ trợ gửi/nhận tin nhắn.
- Máy chủ hiện diện (Presence servers) quản lý trạng thái trực tuyến/ngoại tuyến.
- Máy chủ API xử lý mọi thứ bao gồm đăng nhập người dùng, đăng ký, thay đổi hồ sơ, v.v.
- Máy chủ thông báo gửi thông báo đẩy.
- Cuối cùng, kho lưu trữ khóa-giá trị (key-value store) được sử dụng để lưu trữ lịch sử trò chuyện. Kho lưu trữ khóa-giá trị dành cho cơ sở dữ liệu của dữ liệu lịch sử trò chuyện vì các lý do sau:
   - Cho phép mở rộng theo chiều ngang dễ dàng.
   - Kho lưu trữ KV cung cấp độ trễ rất thấp để truy cập dữ liệu.
   - Cơ sở dữ liệu quan hệ không xử lý tốt phần đuôi dài (long tail) của dữ liệu. Khi các chỉ mục (indexes) phát triển lớn, truy cập ngẫu nhiên rất tốn kém.
   - Kho lưu trữ KV được áp dụng bởi các ứng dụng chat đáng tin cậy đã được chứng minh khác. Ví dụ, cả Facebook Messenger và Discord.


Sau đây là các mô hình dữ liệu cho chat 1-1 và chat nhóm.
   - Khóa chính (primary key) là ID tin nhắn, giúp quyết định trình tự tin nhắn.
   - Đối với chat nhóm, khóa chính kết hợp là (channel_id, message_id). 
      - Các ID có thể được tạo bằng trình tạo số thứ tự 64-bit toàn cục như Snowflake.
      - Một cách tiếp cận tốt hơn là sử dụng trình tạo số thứ tự cục bộ (local sequence number generator). Cục bộ nghĩa là các ID chỉ là duy nhất trong một nhóm.
      - Lý do tại sao ID cục bộ hoạt động là việc duy trì trình tự tin nhắn trong một kênh 1-1 hoặc kênh nhóm là đủ.
      
      <img src="./images/one-to-one-chat.png" alt="One to one chat design" width="300">   
      <img src="./images/group-chat.png" alt="Group chat design" width="300">   


## Bước 3: Đi sâu vào thiết kế

### Khám phá dịch vụ (Service Discovery)

<div style="margin-left:3rem">
   <img src="./images/zookeeper.png" alt="Zookeeper" width="400">   
</div>

- Vai trò chính của khám phá dịch vụ là đề xuất máy chủ chat tốt nhất cho máy khách dựa trên các tiêu chí như vị trí địa lý, công suất máy chủ. 
- Sử dụng **Apache Zookeeper** để phân bổ các máy chủ chat dựa trên các tiêu chí như vị trí địa lý và công suất máy chủ.
- Đảm bảo phân phối tải hiệu quả và giảm thiểu độ trễ.

### Luồng tin nhắn (Messaging Flows)
#### Chat 1-1

1. Người dùng A gửi một tin nhắn đến Máy chủ Chat 1.
2. Máy chủ Chat 1 gán một ID tin nhắn duy nhất và lưu trữ tin nhắn trong kho lưu trữ khóa-giá trị.
3. Nếu Người dùng B đang trực tuyến, tin nhắn được chuyển tiếp đến Máy chủ Chat 2, duy trì kết nối WebSocket liên tục.
4. Nếu Người dùng B ngoại tuyến, một thông báo đẩy sẽ được gửi đi.

#### Chat Nhóm

<div style="margin-left:3rem">
   <img src="./images/group-chat-flow.png" alt="Group Chat Flow" width="400">  
</div>

- Các tin nhắn được sao chép vào hộp thư đến riêng lẻ cho từng người nhận trong nhóm.
- Đơn giản hóa việc đồng bộ nhưng trở nên tốn kém đối với các nhóm lớn hơn.
- Ở phía người nhận, một người nhận có thể nhận tin nhắn từ nhiều người dùng. Mỗi người nhận có một hộp thư đến (hàng đợi đồng bộ hóa tin nhắn) chứa các tin nhắn từ các người gửi khác nhau.

---

#### Đồng bộ hóa tin nhắn (Message Synchronization)

Nhiều người dùng có nhiều thiết bị. Chúng ta cần đồng bộ hóa tin nhắn giữa các thiết bị.
Mỗi thiết bị duy trì một biến gọi là `cur_max_message_id`, biến này theo dõi ID tin nhắn mới nhất trên thiết bị. Các tin nhắn thỏa mãn hai điều kiện sau được coi là tin nhắn mới:

<div style="margin-left:3rem">
   <img src="./images/message-synchronization.png" alt="Message Synchronization"  width="400">  
</div>

- ID người nhận bằng ID người dùng hiện đang đăng nhập.
- ID tin nhắn trong kho lưu trữ khóa-giá trị lớn hơn `cur_max_message_id`.

---

### Trạng thái trực tuyến (Online Presence)
1. **Cơ chế nhịp tim (Heartbeat Mechanism):** 
   <div style="margin-left:3rem">
      <img src="./images/heartbeat-mechanism.png" alt="Heartbeat Mechanism" width="400"> 
   </div>
   
   - Các máy khách (clients) gửi các nhịp tim (heartbeats) định kỳ đến máy chủ hiện diện để chỉ ra rằng chúng đang trực tuyến. 
   - Nếu không nhận được nhịp tim nào trong một ngưỡng (ví dụ x = 30), người dùng được đánh dấu là ngoại tuyến.

     

2. **Mô hình truyền phát (Fanout Model):** 

   <div style="margin-left:3rem">
      <img src="./images/fanout-presence.png" alt="Fanout Presence" width="400"> 
   </div>

   - Các bản cập nhật trạng thái hiện diện được đẩy cho bạn bè bằng mô hình xuất bản-đăng ký (publish-subscribe) trong đó mỗi cặp bạn bè duy trì một kênh.
   - Khi trạng thái trực tuyến của Người dùng A thay đổi, nó sẽ xuất bản sự kiện lên ba kênh, kênh A-B, A-C và A-D. 
   - Ba kênh đó được đăng ký bởi Người dùng B, C và D tương ứng, những người sẽ nhận được bản cập nhật trạng thái trực tuyến.
   - Thiết kế trên có hiệu quả đối với các nhóm người dùng nhỏ.

---

## Các cân nhắc bổ sung
### Khả năng mở rộng (Scalability)
- **Mở rộng theo chiều ngang (Horizontal Scaling):** Thêm máy chủ khi số lượng người dùng tăng lên.
- **Cân bằng tải (Load Balancing):** Phân phối lưu lượng truy cập đồng đều giữa các máy chủ.
- **Bộ nhớ đệm (Caching):** Giảm tải cơ sở dữ liệu và cải thiện độ trễ.

### Xử lý lỗi (Error Handling)
- **Cơ chế thử lại (Retry Mechanisms):** Xử lý các lỗi gửi tin nhắn bằng cách thử lại và đưa vào hàng đợi.
- **Lỗi máy chủ (Server Failures):** Sử dụng khám phá dịch vụ để phân bổ máy chủ mới trong trường hợp có lỗi.

### Các phần mở rộng trong tương lai
1. **Hỗ trợ phương tiện truyền thông (Media Support):** Thêm tính năng xử lý cho ảnh và video, bao gồm nén và lưu trữ đám mây.
2. **Mã hóa đầu cuối (End-to-End Encryption):** Đảm bảo quyền riêng tư của tin nhắn.
3. **Bộ nhớ đệm phía máy khách (Client-Side Caching):** Giảm việc truyền dữ liệu để có hiệu suất tốt hơn.
4. **Cải thiện thời gian tải (Improved Load Times):** Sử dụng các mạng lưới bộ nhớ đệm phân phân tán theo địa lý.