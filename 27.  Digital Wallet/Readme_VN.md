# Chương 27: Ví điện tử

## Giới thiệu
Các **nền tảng thanh toán** thường có một **dịch vụ ví**, cho phép khách hàng lưu trữ tiền trong ứng dụng và có thể rút ra sau đó.

Bạn cũng có thể sử dụng nó để thanh toán hàng hóa & dịch vụ hoặc chuyển tiền cho những người dùng khác cùng sử dụng dịch vụ **ví điện tử**. Việc này có thể nhanh hơn và rẻ hơn so với việc thực hiện qua các kênh thanh toán thông thường.

<div style="margin-left:3rem">
    <img src="./images/digital-wallet.png" alt="digital-wallet" width="500" />
</div>

---

## Bước 1: Hiểu vấn đề và thiết lập phạm vi thiết kế
 * C: Chúng ta chỉ nên tập trung vào việc chuyển tiền giữa các ví điện tử? Chúng ta có cần hỗ trợ các hoạt động nào khác không?
 * I: Hãy tập trung vào việc chuyển tiền giữa các ví điện tử vào lúc này.
 * C: Hệ thống cần hỗ trợ bao nhiêu giao dịch mỗi giây (TPS)?
 * I: Hãy giả định là 1 triệu TPS.
 * C: Một ví điện tử có các yêu cầu nghiêm ngặt về tính chính xác. Chúng ta có thể giả định rằng các đảm bảo về giao dịch (transactional guarantees) là đủ không?
 * I: Nghe có vẻ ổn.
 * C: Chúng ta có cần chứng minh tính chính xác không?
 * I: Chúng ta có thể làm điều đó thông qua đối soát (reconciliation), nhưng việc đó chỉ phát hiện ra sự sai lệch chứ không chỉ ra nguyên nhân gốc rễ. Thay vào đó, chúng ta muốn có thể phát lại (replay) dữ liệu từ đầu để tái cấu trúc lịch sử.
 * C: Chúng ta có thể giả định yêu cầu về độ sẵn sàng là 99.99% không?
 * I: Có.
 * C: Chúng ta có cần xem xét việc đổi ngoại tệ không?
 * I: Không, nó nằm ngoài phạm vi.

Tóm tắt những gì chúng ta phải hỗ trợ:
 * Hỗ trợ chuyển số dư giữa hai tài khoản.
 * Hỗ trợ 1 triệu TPS.
 * Độ tin cậy là 99.99%.
 * Hỗ trợ các giao dịch (transactions).
 * Hỗ trợ khả năng tái lập (reproducibility).

### **Ước tính nhanh (Back-of-the-envelope estimation)**
Một cơ sở dữ liệu quan hệ truyền thống, được cung cấp trên đám mây có thể hỗ trợ khoảng 1000 TPS.

Để đạt được 1 triệu TPS, chúng ta sẽ cần 1000 node cơ sở dữ liệu. Nhưng nếu mỗi lần chuyển tiền có hai chặng (trừ tiền và cộng tiền), thì thực tế chúng ta cần hỗ trợ 2 triệu TPS.

Một trong những mục tiêu thiết kế của chúng ta sẽ là tăng TPS mà một node đơn lẻ có thể xử lý để chúng ta có thể có ít node cơ sở dữ liệu hơn.

| TPS mỗi node | Số lượng node |
|--------------|---------------|
| 100          | 20,000        |
| 1,000        | 2,000         |
| 10,000       | 200           |

---

## Bước 2: Đề xuất thiết kế cấp cao và đạt được sự đồng thuận

### **Thiết kế API**
Chúng ta chỉ cần hỗ trợ một endpoint cho buổi phỏng vấn này:
```
POST /v1/wallet/balance_transfer - chuyển số dư từ ví này sang ví khác
```

Các tham số yêu cầu - from_account, to_account, amount (kiểu string để không làm mất độ chính xác), currency, transaction_id (khóa idempotency).

Ví dụ phản hồi:
```
{
    "status": "success"
    "transaction_id": "01589980-2664-11ec-9621-0242ac130002"
}
```

### **Giải pháp Sharding trong bộ nhớ (In-memory sharding solution)**
Ứng dụng ví của chúng ta duy trì số dư tài khoản cho mọi tài khoản người dùng.

Một cấu trúc dữ liệu tốt để biểu diễn điều này là một `map<user_id, balance>`, có thể được triển khai bằng cách sử dụng lưu trữ Redis trong bộ nhớ.

Vì một node Redis không thể chịu được 1 triệu TPS, chúng ta cần phân vùng (partition) cụm Redis của mình thành nhiều node.

Ví dụ về thuật toán phân vùng:
```
String accountID = "A";
Int partitionNumber = 7;
Int myPartition = accountID.hashCode() % partitionNumber;
```

Zookeeper có thể được sử dụng để lưu trữ số lượng phân vùng và địa chỉ của các node Redis vì nó là một bộ lưu trữ cấu hình có tính sẵn sàng cao.

Cuối cùng, dịch vụ ví là một dịch vụ không trạng thái (stateless) chịu trách nhiệm thực hiện các hoạt động chuyển tiền. Nó có thể dễ dàng mở rộng theo chiều ngang:

<div style="margin-left:3rem">
    <img src="./images/wallet-service.png" alt="wallet-service" width="500" />
</div>

Mặc dù giải pháp này giải quyết được các vấn đề về khả năng mở rộng, nhưng nó không cho phép chúng ta thực hiện việc chuyển số dư một cách nguyên tử (atomically).

### **Giao dịch phân tán (Distributed transactions)**
Một cách tiếp cận để xử lý các giao dịch là sử dụng giao thức cam kết hai pha (two-phase commit - 2PC) trên nền tảng các cơ sở dữ liệu quan hệ được sharding:

<div style="margin-left:3rem">
    <img src="./images/distributed-transactions-relational-dbs.png" alt="distributed-transactions-relational-dbs" width="500" />
</div>

Dưới đây là cách hoạt động của giao thức cam kết hai pha (2PC):

<div style="margin-left:3rem">
    <img src="./images/2pc-protocol.png" alt="2pc-protocol" width="500" />
</div>

 * Bộ điều phối (coordinator - dịch vụ ví) thực hiện các hoạt động đọc và ghi trên nhiều cơ sở dữ liệu như bình thường.
 * Khi ứng dụng đã sẵn sàng cam kết giao dịch, bộ điều phối yêu cầu tất cả các cơ sở dữ liệu chuẩn bị (prepare).
 * Nếu tất cả các cơ sở dữ liệu trả lời là "có" (yes), thì bộ điều phối yêu cầu các cơ sở dữ liệu cam kết (commit) giao dịch.
 * Ngược lại, tất cả các cơ sở dữ liệu được yêu cầu hủy bỏ (abort) giao dịch.

Nhược điểm của phương pháp 2PC:
 * Hiệu năng không cao do tranh chấp khóa (lock contention).
 * Bộ điều phối là một điểm lỗi duy nhất (single point of failure).

### **Giao dịch phân tán sử dụng Try-Confirm/Cancel (TC/C)**
TC/C là một biến thể của giao thức 2PC, hoạt động với các giao dịch bù đắp (compensating transactions):
 * Bộ điều phối yêu cầu tất cả các cơ sở dữ liệu dự trữ tài nguyên cho giao dịch.
 * Bộ điều phối thu thập các phản hồi từ DB - nếu có, DB được yêu cầu thử-xác nhận (try-confirm). Nếu không, DB được yêu cầu thử-hủy (try-cancel).

Một điểm khác biệt quan trọng giữa TC/C và 2PC là 2PC thực hiện một giao dịch duy nhất, trong khi ở TC/C, có hai giao dịch độc lập.

Dưới đây là cách TC/C hoạt động theo các pha:

| Pha | Hoạt động | A                   | C                   |
|-----|-----------|---------------------|---------------------|
| 1   | Thử (Try) | Thay đổi số dư: -$1 | Không làm gì        |
| 2   | Xác nhận (Confirm) | Không làm gì | Thay đổi số dư: +$1 |
|     | Hủy (Cancel) | Thay đổi số dư: +$1 | Không làm gì        |

Pha 1 - thử (try):

<div style="margin-left:3rem">
    <img src="./images/try-phase.png" alt="try-phase" width="500" />
</div>

 * bộ điều phối bắt đầu giao dịch cục bộ trong DB của A để giảm số dư của A đi 1$.
 * DB của C được cung cấp một lệnh NOP (không hoạt động), lệnh này không làm gì cả.

Pha 2a - xác nhận (confirm):

<div style="margin-left:3rem">
    <img src="./images/confirm-phase.png" alt="confirm-phase" width="500" />
</div>

 * nếu cả hai DB đều trả lời "có", pha xác nhận bắt đầu.
 * DB của A nhận được NOP, trong khi DB của C được hướng dẫn tăng số dư của C thêm 1$ (giao dịch cục bộ).

Pha 2b - hủy (cancel):

<div style="margin-left:3rem">
    <img src="./images/cancel-phase.png" alt="cancel-phase" width="500" />
</div>

 * Nếu bất kỳ hoạt động nào trong pha 1 thất bại, pha hủy bắt đầu.
 * DB của A được hướng dẫn tăng số dư của A thêm 1$, DB của C nhận được NOP.

Dưới đây là sự so sánh giữa 2PC và TC/C:

|      | Pha đầu tiên                                            | Pha thứ hai: thành công            | Pha thứ hai: thất bại                     |
|------|---------------------------------------------------------|------------------------------------|-------------------------------------------|
| 2PC  | các giao dịch chưa được thực hiện xong                  | Cam kết/Hủy tất cả các giao dịch   | Hủy tất cả các giao dịch                  |
| TC/C | Tất cả các giao dịch đã hoàn thành - được cam kết hoặc hủy | Thực thi các giao dịch mới nếu cần | Đảo ngược giao dịch đã được cam kết       |

TC/C còn được gọi là một giao dịch phân tán bằng cách bù đắp. Hoạt động cấp cao được xử lý trong logic nghiệp vụ.

Các đặc điểm khác của TC/C:
 * Không phụ thuộc vào loại cơ sở dữ liệu, miễn là cơ sở dữ liệu hỗ trợ các giao dịch.
 * Các chi tiết và sự phức tạp của các giao dịch phân tán cần được xử lý trong logic nghiệp vụ.

### **Các chế độ lỗi của TC/C**
Nếu bộ điều phối gặp sự cố giữa chừng, nó cần khôi phục trạng thái trung gian của mình.
Điều đó có thể được thực hiện bằng cách duy trì các bảng trạng thái pha, được cập nhật một cách nguyên tử trong các phân mảnh cơ sở dữ liệu:

<div style="margin-left:3rem">
    <img src="./images/phase-status-tables.png" alt="phase-status-tables" width="500" />
</div>

Bảng đó chứa những gì:
 * ID và nội dung của giao dịch phân tán.
 * trạng thái của pha thử (try) - chưa gửi, đã gửi, đã nhận phản hồi.
 * tên pha thứ hai - xác nhận hoặc hủy.
 * trạng thái của pha thứ hai.
 * cờ thứ tự sai (out-of-order flag - sẽ được giải thích sau).

Một lưu ý khi sử dụng TC/C là có một khoảnh khắc ngắn mà trạng thái các tài khoản không nhất quán với nhau khi một giao dịch phân tán đang được xử lý:

<div style="margin-left:3rem">
    <img src="./images/unbalanced-state.png" alt="unbalanced-state" width="500" />
</div>

Điều này có thể chấp nhận được miễn là chúng ta luôn khôi phục từ trạng thái này và người dùng không thể sử dụng trạng thái trung gian để ví dụ như tiêu số tiền đó.
Điều này được đảm bảo bằng cách luôn thực hiện các lệnh trừ tiền trước khi thực hiện các lệnh cộng tiền.

| Các lựa chọn pha Thử | Tài khoản A | Tài khoản C |
|----------------------|-------------|-------------|
| Lựa chọn 1           | -$1         | NOP         |
| Lựa chọn 2 (không hợp lệ) | NOP       | +$1         |
| Lựa chọn 3 (không hợp lệ) | -$1       | +$1         |

Lưu ý rằng lựa chọn 3 từ bảng trên là không hợp lệ vì chúng ta không thể đảm bảo thực thi nguyên tử các giao dịch trên các cơ sở dữ liệu khác nhau mà không dựa vào 2PC.

Một trường hợp biên cần giải quyết là thực thi sai thứ tự:

<div style="margin-left:3rem">
    <img src="./images/out-of-order-execution.png" alt="out-of-order-execution" width="500" />
</div>

Có khả năng một cơ sở dữ liệu nhận được một hoạt động hủy (cancel) trước khi nhận được một hoạt động thử (try). Trường hợp biên này có thể được xử lý bằng cách thêm một cờ thứ tự sai trong bảng trạng thái pha của chúng ta.
Khi nhận được một hoạt động thử, trước tiên chúng ta kiểm tra xem cờ thứ tự sai có được đặt hay không và nếu có, một lỗi sẽ được trả về.

### **Giao dịch phân tán sử dụng Saga**
Một cách tiếp cận phổ biến khác là sử dụng Sagas - một tiêu chuẩn để triển khai các giao dịch phân tán với kiến trúc microservices.

Dưới đây là cách nó hoạt động:
 * tất cả các hoạt động được sắp xếp theo một trình tự. Tất cả các hoạt động là độc lập trong cơ sở dữ liệu của riêng chúng.
 * các hoạt động được thực thi từ đầu đến cuối.
 * khi một hoạt động thất bại, toàn bộ quá trình bắt đầu khôi phục (rollback) cho đến khi quay lại điểm bắt đầu bằng các hoạt động bù đắp.

<div style="margin-left:3rem">
    <img src="./images/saga.png" alt="saga" width="500" />
</div>

Làm thế nào để điều phối luồng công việc? Có hai hướng tiếp cận chúng ta có thể thực hiện:
 * Biên đạo (Choreography) - tất cả các dịch vụ tham gia vào một saga đăng ký các sự kiện liên quan và thực hiện phần việc của mình trong saga.
 * Điều phối (Orchestration) - một bộ điều phối duy nhất hướng dẫn tất cả các dịch vụ thực hiện công việc của mình theo đúng thứ tự.

Thách thức của việc sử dụng mô hình biên đạo là logic nghiệp vụ bị chia nhỏ trên nhiều dịch vụ, giao tiếp một cách bất đồng bộ.
Hướng tiếp cận điều phối xử lý tốt sự phức tạp, vì vậy nó thường là hướng tiếp cận được ưu tiên trong hệ thống ví điện tử.

Dưới đây là sự so sánh giữa TC/C và Saga:

|                                           | TC/C            | Saga                     |
|-------------------------------------------|-----------------|--------------------------|
| Hành động bù đắp                          | Trong pha Hủy   | Trong pha Rollback       |
| Điều phối trung tâm                       | Có              | Có (mô hình điều phối)   |
| Thứ tự thực thi hoạt động                 | bất kỳ          | tuyến tính               |
| Khả năng thực thi song song               | Có              | Không (thực thi tuyến tính) |
| Có thể thấy trạng thái không nhất quán một phần | Có        | Có                       |
| Logic ứng dụng hay cơ sở dữ liệu          | Ứng dụng        | Ứng dụng                 |

Sự khác biệt chính là TC/C có thể chạy song song, vì vậy quyết định của chúng ta dựa trên yêu cầu về độ trễ - nếu chúng ta cần đạt được độ trễ thấp, chúng ta nên chọn hướng tiếp cận TC/C.

Bất kể hướng tiếp cận nào chúng ta chọn, chúng ta vẫn cần hỗ trợ kiểm tra (auditing) và phát lại lịch sử để khôi phục từ các trạng thái lỗi.

### **Event sourcing (Nguồn sự kiện)**
Trong thực tế, một ứng dụng ví điện tử có thể được kiểm toán và chúng ta phải trả lời một số câu hỏi nhất định:
 * Chúng ta có biết số dư tài khoản tại bất kỳ thời điểm nào không?
 * Làm thế nào chúng ta biết số dư hiện tại và lịch sử là chính xác?
 * Làm thế nào chúng ta chứng minh được logic hệ thống là chính xác sau khi thay đổi mã nguồn?

Event sourcing là một kỹ thuật giúp chúng ta trả lời những câu hỏi này.

Nó bao gồm bốn khái niệm:
 * lệnh (command) - hành động dự định từ thế giới thực, ví dụ: chuyển 1$ từ tài khoản A sang B. Cần phải có một thứ tự toàn cục, do đó chúng được đưa vào một hàng đợi FIFO.
   * các lệnh, không giống như các sự kiện, có thể thất bại và có một số tính ngẫu nhiên do ví dụ như IO hoặc trạng thái không hợp lệ.
   * các lệnh có thể tạo ra không hoặc nhiều sự kiện.
   * việc tạo sự kiện có thể chứa tính ngẫu nhiên như IO bên ngoài. Điều này sẽ được xem xét lại sau.
 * sự kiện (event) - sự thật lịch sử về các sự kiện đã xảy ra trong hệ thống, ví dụ: "đã chuyển 1$ từ A sang B".
   * không giống như các lệnh, các sự kiện là những sự thật đã xảy ra trong hệ thống của chúng ta.
   * tương tự như các lệnh, chúng cần được sắp xếp thứ tự, do đó, chúng được đưa vào một hàng đợi FIFO.
 * trạng thái (state) - những gì đã thay đổi do kết quả của một sự kiện. Ví dụ: một kho lưu trữ key-value giữa tài khoản và số dư của họ.
 * máy trạng thái (state machine) - thúc đẩy quá trình event sourcing. Nó chủ yếu xác thực các lệnh và áp dụng các sự kiện để cập nhật trạng thái hệ thống.
   * máy trạng thái nên mang tính xác định (deterministic), do đó, nó không nên đọc IO bên ngoài hoặc dựa vào tính ngẫu nhiên.

<div style="margin-left:3rem">
    <img src="./images/event-sourcing.png" alt="event-sourcing" width="500" />
</div>

Dưới đây là cái nhìn động về event sourcing:

<div style="margin-left:3rem">
    <img src="./images/dynamic-event-sourcing.png" alt="dynamic-event-sourcing" width="500" />
</div>

Đối với dịch vụ ví của chúng ta, các lệnh là các yêu cầu chuyển số dư. Chúng ta có thể đưa chúng vào một hàng đợi FIFO, chẳng hạn như Kafka:

<div style="margin-left:3rem">
    <img src="./images/command-queue.png" alt="command-queue" width="500" />
</div>

Dưới đây là bức tranh toàn cảnh:

<div style="margin-left:3rem">
    <img src="./images/wallet-service-state-macghine.png" alt="wallet-service-state-machine" width="500" />
</div>

 * máy trạng thái đọc các lệnh từ hàng đợi lệnh.
 * trạng thái số dư được đọc từ cơ sở dữ liệu.
 * lệnh được xác thực. Nếu hợp lệ, hai sự kiện cho mỗi tài khoản sẽ được tạo ra.
 * sự kiện tiếp theo được đọc và áp dụng bằng cách cập nhật số dư (trạng thái) trong cơ sở dữ liệu.

Ưu điểm chính của việc sử dụng event sourcing là khả năng tái lập (reproducibility). Trong thiết kế này, tất cả các hoạt động cập nhật trạng thái đều được lưu dưới dạng lịch sử bất biến của tất cả các thay đổi số dư.

Số dư lịch sử luôn có thể được tái cấu trúc bằng cách phát lại các sự kiện từ đầu.
Vì danh sách sự kiện là bất biến và máy trạng thái mang tính xác định, chúng ta được đảm bảo sẽ thành công trong việc phát lại bất kỳ trạng thái trung gian nào.

<div style="margin-left:3rem">
    <img src="./images/historical-states.png" alt="historical-states" width="500" />
</div>

Tất cả các câu hỏi liên quan đến kiểm toán được hỏi ở đầu phần này đều có thể được giải quyết bằng cách dựa vào event sourcing:
 * Chúng ta có biết số dư tài khoản tại bất kỳ thời điểm nào không? - các sự kiện có thể được phát lại từ đầu cho đến thời điểm mà chúng ta quan tâm.
 * Làm thế nào chúng ta biết số dư lịch sử và hiện tại là chính xác? - tính chính xác có thể được xác minh bằng cách tính toán lại tất cả các sự kiện từ đầu.
 * Làm thế nào chúng ta chứng minh được logic hệ thống là chính xác sau khi thay đổi mã nguồn? - chúng ta có thể chạy các phiên bản mã khác nhau dựa trên cùng các sự kiện và xác minh rằng kết quả của chúng là giống hệt nhau.

Việc trả lời các truy vấn của khách hàng về số dư của họ có thể được giải quyết bằng kiến trúc CQRS - có thể có nhiều máy trạng thái chỉ đọc chịu trách nhiệm truy vấn trạng thái lịch sử, dựa trên danh sách các sự kiện bất biến:

<div style="margin-left:3rem">
    <img src="./images/cqrs-architecture.png" alt="cqrs-architecture" width="500" />
</div>

---

## Bước 3: Thiết kế chi tiết
Trong phần này, chúng ta sẽ khám phá một số tối ưu hóa hiệu năng vì chúng ta vẫn được yêu cầu mở rộng quy mô lên 1 triệu TPS.

### **Event sourcing hiệu năng cao**
Tối ưu hóa đầu tiên chúng ta sẽ khám phá là lưu các lệnh và sự kiện vào ổ đĩa cục bộ thay vì một kho lưu trữ bên ngoài như Kafka.

Điều này tránh được độ trễ mạng và ngoài ra, vì chúng ta chỉ thực hiện các thao tác thêm vào cuối (append), thao tác đó thường nhanh đối với ổ cứng HDD.

Tối ưu hóa tiếp theo là lưu đệm các lệnh và sự kiện gần đây trong bộ nhớ để tiết kiệm thời gian tải chúng lại từ ổ đĩa.

Ở cấp độ thấp, chúng ta có thể đạt được các tối ưu hóa nói trên bằng cách tận dụng một lệnh gọi là `mmap`, lệnh này lưu trữ dữ liệu trong ổ đĩa cục bộ cũng như lưu đệm nó trong bộ nhớ:

<div style="margin-left:3rem">
    <img src="./images/mmap-optimization.png" alt="mmap-optimization" width="500" />
</div>

Tối ưu hóa tiếp theo chúng ta có thể thực hiện là lưu trữ trạng thái trong hệ thống tệp cục bộ bằng cách sử dụng SQLite - một cơ sở dữ liệu quan hệ cục bộ dựa trên tệp. RocksDB cũng là một lựa chọn tốt khác.

Cho mục đích của chúng ta, chúng ta sẽ chọn RocksDB vì nó sử dụng cây merge có cấu trúc log (LSM), được tối ưu hóa cho các hoạt động ghi.
Hiệu năng đọc được tối ưu hóa thông qua lưu đệm.

<div style="margin-left:3rem">
    <img src="./images/rocks-db-approach.png" alt="rocks-db-approach" width="500" />
</div>

Để tối ưu hóa khả năng tái lập, chúng ta có thể định kỳ lưu các bản sao nhanh (snapshots) vào ổ đĩa để không phải tái lập một trạng thái nhất định từ đầu mọi lúc. Chúng ta có thể lưu trữ các bản sao nhanh dưới dạng các tệp nhị phân lớn trong bộ lưu trữ tệp phân tán, ví dụ: HDFS:

<div style="margin-left:3rem">
    <img src="./images/snapshot-approach.png" alt="snapshot-approach" width="500" />
</div>

### **Event sourcing hiệu năng cao và đáng tin cậy**
Tất cả các tối ưu hóa đã thực hiện cho đến nay đều tuyệt vời, nhưng chúng khiến dịch vụ của chúng ta trở thành dịch vụ có trạng thái (stateful). Chúng ta cần giới thiệu một số hình thức nhân bản (replication) cho mục đích tin cậy.

Trước khi làm điều đó, chúng ta nên phân tích loại dữ liệu nào cần độ tin cậy cao trong hệ thống của chúng ta:
 * trạng thái (state) và bản sao nhanh (snapshot) luôn có thể được tạo lại bằng cách tái lập chúng từ danh sách sự kiện. Do đó, chúng ta chỉ cần đảm bảo độ tin cậy của danh sách sự kiện.
 * người ta có thể nghĩ rằng chúng ta luôn có thể tạo lại danh sách sự kiện từ danh sách lệnh, nhưng điều đó không đúng, vì các lệnh mang tính không xác định.
 * kết luận là chúng ta chỉ cần đảm bảo độ tin cậy cao cho danh sách sự kiện.

Để đạt được độ tin cậy cao cho các sự kiện, chúng ta cần nhân bản danh sách đó trên nhiều node. Chúng ta cần đảm bảo:
 * không có dữ liệu nào bị mất.
 * thứ tự tương đối của dữ liệu trong một tệp nhật ký vẫn giữ nguyên trên các bản sao.

Để đạt được điều này, chúng ta có thể sử dụng một thuật toán đồng thuận, chẳng hạn như Raft.

Với Raft, có một node dẫn đầu (leader) đang hoạt động và có các node theo sau (followers) đang ở trạng thái thụ động. Nếu một leader gặp sự cố, một trong các followers sẽ thay thế.
Miễn là hơn một nửa số node vẫn hoạt động, hệ thống sẽ tiếp tục chạy.

<div style="margin-left:3rem">
    <img src="./images/raft-replication.png" alt="raft-replication" width="500" />
</div>

Với hướng tiếp cận này, tất cả các node đều cập nhật trạng thái dựa trên danh sách sự kiện. Raft đảm bảo leader và followers có cùng một danh sách sự kiện.

### **Event sourcing phân tán**
Cho đến nay, chúng ta đã thiết kế được một hệ thống có hiệu năng node đơn cao và đáng tin cậy.

Một số hạn chế chúng ta phải giải quyết:
 * Dung lượng của một nhóm Raft đơn lẻ là có hạn. Đến một lúc nào đó, chúng ta cần sharding dữ liệu và triển khai các giao dịch phân tán.
 * Trong kiến trúc CQRS, luồng yêu cầu/phản hồi chậm. Khách hàng sẽ cần định kỳ thăm dò (poll) hệ thống để biết khi nào ví của họ đã được cập nhật.

Việc thăm dò không mang tính thời gian thực, do đó, có thể mất một lúc để người dùng biết được sự thay đổi trong số dư của họ. Ngoài ra, nó có thể làm quá tải các dịch vụ truy vấn nếu tần suất thăm dò quá cao:

<div style="margin-left:3rem">
    <img src="./images/polling-approach.png" alt="polling-approach" width="500" />
</div>

Để giảm tải cho hệ thống, chúng ta có thể giới thiệu một reverse proxy, công cụ này thay mặt người dùng gửi các lệnh và thăm dò phản hồi cho họ:

<div style="margin-left:3rem">
    <img src="./images/reverse-proxy.png" alt="reverse-proxy" width="500" />
</div>

Điều này làm giảm tải hệ thống vì chúng ta có thể lấy dữ liệu cho nhiều người dùng bằng một yêu cầu duy nhất, nhưng nó vẫn không giải quyết được yêu cầu nhận thông tin theo thời gian thực.

Một thay đổi cuối cùng chúng ta có thể làm là để các máy trạng thái chỉ đọc đẩy (push) các phản hồi trở lại reverse proxy ngay khi chúng có sẵn. Điều này có thể mang lại cho người dùng cảm giác rằng các cập nhật diễn ra theo thời gian thực:

<div style="margin-left:3rem">
    <img src="./images/push-state-machines.png" alt="push-state-machines" width="500" />
</div>

Cuối cùng, để mở rộng hệ thống hơn nữa, chúng ta có thể sharding hệ thống thành nhiều nhóm Raft, nơi chúng ta triển khai các giao dịch phân tán trên chúng bằng một bộ điều phối thông qua TC/C hoặc Sagas:

<div style="margin-left:3rem">
    <img src="./images/sharded-raft-groups.png" alt="sharded-raft-groups" width="500" />
</div>

Dưới đây là một ví dụ về vòng đời của một yêu cầu chuyển số dư trong hệ thống cuối cùng của chúng ta:
 * Người dùng A gửi một giao dịch phân tán đến bộ điều phối Saga với hai hoạt động - `A-1` và `C+1`.
 * Bộ điều phối Saga tạo một bản ghi trong bảng trạng thái pha để theo dõi trạng thái của giao dịch.
 * Bộ điều phối xác định phân vùng nào nó cần gửi lệnh đến.
 * Leader Raft của phân vùng 1 nhận lệnh `A-1`, xác thực nó, chuyển đổi nó thành một sự kiện và nhân bản nó trên các node khác trong nhóm Raft.
 * Kết quả sự kiện được đồng bộ hóa với máy trạng thái đọc, máy này sẽ đẩy phản hồi trở lại bộ điều phối.
 * Bộ điều phối tạo một bản ghi cho biết hoạt động đã thành công và tiếp tục với hoạt động tiếp theo - `C+1`.
 * Hoạt động tiếp theo được thực thi tương tự như hoạt động đầu tiên - xác định phân vùng, lệnh được gửi, thực thi, máy trạng thái đọc đẩy lại phản hồi.
 * Bộ điều phối tạo một bản ghi cho biết hoạt động 2 cũng thành công và cuối cùng thông báo cho khách hàng về kết quả.

---

## Bước 4: Tổng kết
Dưới đây là sự phát triển trong thiết kế của chúng ta:
 * Chúng ta bắt đầu từ một giải pháp sử dụng Redis trong bộ nhớ. Vấn đề với hướng tiếp cận này là nó không phải là bộ lưu trữ bền vững.
 * Chúng ta chuyển sang sử dụng các cơ sở dữ liệu quan hệ, trên đó chúng ta thực hiện các giao dịch phân tán bằng 2PC, TC/C hoặc saga phân tán.
 * Tiếp theo, chúng ta giới thiệu event sourcing để làm cho tất cả các hoạt động có thể kiểm toán được.
 * Chúng ta bắt đầu bằng cách lưu trữ dữ liệu vào bộ lưu trữ bên ngoài bằng cơ sở dữ liệu và hàng đợi bên ngoài, nhưng điều đó không hiệu năng.
 * Chúng ta tiến hành lưu trữ dữ liệu trong bộ lưu trữ tệp cục bộ, tận dụng hiệu năng của các hoạt động chỉ thêm vào cuối (append-only). Chúng ta cũng sử dụng bộ đệm để tối ưu hóa đường dẫn đọc.
 * Hướng tiếp cận trước đó, mặc dù hiệu năng, nhưng không bền vững. Do đó, chúng ta đã giới thiệu sự đồng thuận Raft với nhân bản để tránh các điểm lỗi duy nhất.
 * Chúng ta cũng áp dụng CQRS với một reverse proxy để quản lý vòng đời của giao dịch thay mặt cho người dùng.
 * Cuối cùng, chúng ta phân vùng dữ liệu của mình trên nhiều nhóm Raft, được điều phối bằng cơ chế giao dịch phân tán - TC/C hoặc saga phân tán.
