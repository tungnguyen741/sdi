# Thiết Kế Bộ Giới Hạn Truy Cập

Trong một hệ thống mạng, bộ giới hạn truy cập (rate limiter) là công cụ cho điều khiển lưu lượng truy cập được gửi đến bởi client hay một dịch vụ nào đó. Trong HTTP, bộ giới hạn truy cập sẽ giới hạn số lượng yêu cầu client được cho phép gửi trong một khoảng thời gian cụ thể. Nếu số lượng yêu cầu API vượt quá ngưỡng cho phép bởi bộ giới hạn, tất cả các lệnh gọi tiếp theo sẽ bị chặn. Ví dụ:
- Một người viết không thể viết hai bài một giây.
- Chỉ có thể tạo 10 tài khoản từ một địa chỉ IP.
- Chỉ có thể nhận thưởng 5 lần trên một tuần từ một thiết bị.

Trong bài hôm nay, ta sẽ trả lời câu hỏi thiết kế một bộ giới hạn truy cập. Trước tiên, ta sẽ xem qua các lợi ích khi sử dụng nó:
- Ngăn chặn cạn kiệt tài nguyên bởi tấn công DoS(Denial of Service) [1]. Hầu hết API công khai bởi các công ty lớn đều thực thi các kiểu giới hạn truy cập. Ví dụ, Twitter giới hạn số lượng tweet là 300 trong 3 giờ [2]. Google Docs giới hạn 300 người dùng trên 60 giây cho một lần yêu cầu đọc[3]. Một bộ giới hạn truy cập sẽ ngăn chặn các cuộc tấn công DoS, dù vô tình hay cố ý, bằng cách chặn các lệnh gọi vượt ngưỡng.
- Giảm chí phí. Việc giới hạn các yêu cầu vượt ngưỡng đồng nghĩa với ít server hơn và ít cấp phát tài nguyên hơn. Giới hạn truy cập là rất quan trọng với các công ty sử dụng API trả phí bên thứ 3. Ví dụ bạn bị tính phí cơ sở cho mỗi lần gọi cho các API bên ngoài như: kiểm tra tín dụng, thanh toán, truy xuất hồ sơ sức khỏe, ... Hạn chế số lượng yêu cầu là điều cần thiết để giảm chi phí.
- Ngăn chặn server bị sập. Để giảm tải cho server, bộ giới hạn được dùng để lọc các yêu cầu vượt ngưỡng gây ra bởi bot hoặc hành vi phá hoại từ người dùng.

## 1. Hiểu vấn đề và thiết lập phạm vi của thiết kế

Giới hạn truy cập có thể thực hiện bằng cách triển khai các thuật toán khác nhau, mỗi cái sẽ có ưu nhược riêng. Việc tương tác với người phỏng vấn sẽ giúp ứng viên hiểu rõ kiểu và phạm vi của bộ giới hạn mà họ cần xây dựng.

- **Ứng viên**: Kiểu triển khai của bộ giới hạn truy cập mà ta cần là gì? Nó sẽ là client-side hay server-side?
- **Người phỏng vấn**: Câu hỏi tốt đấy. Ta sẽ tập trung vào server-side.

- **Ứng viên**: Bộ giới hạn sẽ điều tiết yêu cầu dựa trên địa chỉ IP, ID người dùng hay một thuộc tính nào khác?
- **Người phỏng vấn**: Nó phải đủ linh hoạt để hỗ trợ các quy tắc điều tiết khác nhau.

- **Ứng viên**: Hệ thống của nó lớn cỡ nào? Một công ty startup hay một công ty lớn với đông đảo người dùng?
- **Người phỏng vấn**: Hệ phống phải xử lý một lượng lớn người dùng.

- **Ứng viên**: Hệ thống sẽ hoạt động trong môi trường phân tán?
- **Người phỏng vấn**: Đúng vậy.

- **Ứng viên**: Bộ giới hạn sẽ là một dịch vụ riêng biệt hay là được triển khai trong ứng dụng?
- **Người phỏng vấn**: Tuỳ bạn quyết định.

- **Ứng viên**: Ta có cần thông báo cho người dùng bị chặn (điều tiết) hay không?
- **Người phỏng vấn**: Dĩ nhiên.

Như vậy ta có các yêu cầu sau:
- Giới hạn chính xác các yêu cầu vượt ngưỡng.
- Độ trễ thấp. Bộ giới hạn truy cập không nên chậm hơn thời gian phản hồi HTTP.
- Sử dụng bộ nhớ ít nhất có thể.
- Có thể chia sẻ trên nhiều server hay tiến trình.
- Xử lý ngoại lệ. Hiển thị dọn dẹp ngoại lệ cho người dùng yêu cầu của họ bị điều tiết.
- Khả năng chịu lỗi cao. Bất kỳ sự cố nào với bộ giới hạn truy cập sẽ không ảnh hưởng đến toàn hệ thống.

## 2. Đề xuất thiết kế

**Nơi đặt bộ giới hạn truy cập?**

Ta có thể triển khai bộ giới hạn bằng cả server-side và client-side:
* Client-side: nhìn chung thì client là nơi không đáng tin cậy để thực thi bộ giới hạn truy cập, bởi vì các yêu cầu client có thể dễ dàng bị giả mạo bởi các tác nhân độc hại. Hơn nữa ta cũng không có quyền kiểm soát khi triển khai ở client.
* Server-side: Hình bên dưới cho ta thấy bộ giới hạn truy cập được đặt ở server-side.

![](./assets/server-side.png)

Bên cạnh đó ta cũng có một cách khác là đặt bộ giới hạn truy cập ở server API, để nó hoạt động giống như một middleware, thực hiện điều tiết yêu cầu đến API của bạn như hình bên dưới.

![](./assets/middleware.png)

Giả sử API của ta cho phép 2 yêu cầu trên một giây, và client gửi 3 yêu cầu đến server trong một giây. Hai yêu cầu đầu tiên được định tuyến đến server. Nhưng yêu cầu thứ ba bị bộ giới hạn chặn lại và trả về HTTP Status code 429. Phản hồi HTTP 429 biểu hiện một người dùng gửi quá nhiều yêu cầu.

![](./assets/middleware2.png)

Các microservices [4] đang trở nên phổ biến và bộ giới hạn truy cập thường được dùng cho triển khai trong API gateway. API gateway là dịch vụ quản lý hoàn toàn hỗ trợ giới hạn truy cập, chứng chỉ SSL, xác thực, IP được cho phép,... Bây giờ ta đã biệt API gateway là một dịch vụ hỗ trợ giới hạn truy cập.

Khi thiết kế bộ giới hạn truy cập, câu hỏi quan trọng nhất cần trả lời là: "Đâu là nơi bộ giới hạn nên triển khai, ở server-side hay là trong gateway". Không có một câu trả lời hoàn chỉnh, nó tuỳ thuộc vào công ty mà bạn đang làm việc, các công nghệ hiện tại, nguồn tài nguyên, độ ưu tiên của mục tiên,... Ở đây ta có vài hướng dẫn phổ biến.
- Đánh giá công nghệ hiện tại của bạn, chẳng hạn như ngôn ngữ lập trình, cache,... Đảm bảo rằng ngôn ngữ lập trình hiện tại của bạn hiệu quả để thực hiện giới hạn truy cập ở phía server.
- Xác định thuật toán giới hạn truy cập phù hợp với nhu cầu doanh nghiệp. Khi bạn triển khai mọi thứ ở phía server, bạn có toàn quyền kiểm soát thuật toán. Tuy nhiên, sự lựa chọn có thể bị hạn chế nếu bạn dùng gateway bên thứ 3.
- Nếu bạn sử dụng kiến trúc microservice bao gồm các API gateway trong thiết kế để thực hiện xác thức, lập danh sách IP cho phép,... bạn có thể thêm bộ giới hạn truy cập vào API gateway.
- Việc xây dựng dịch vụ giới hạn truy cập của riêng bạn mất nhiều thời gian. Nếu bạn không có đủ tài nguyên để triển khai bộ giới hạn truy cập, thì API gateway thương mại là một lựa chọn tốt hơn.

### Thuật toán cho giới hạn truy cập

Giới hạn truy cập có thể thực hiện bằng cách triển khai các thuật toán khác nhau, mỗi cái sẽ có ưu nhược riêng. Mặc dù ở bài viết này không tập trung vào các thuật toán, nhưng việc hiểu chúng sẽ giúp chọn đúng thuật toán hoặc kết hợp các thuật toán để phù hợp với các trường hợp sử dụng của chúng ta. Dưới đây là danh sách các thuật toán phổ biến:
* Token bucket
* Leaking bucket
* Fixed window counter
* Sliding window log
* Sliding window counter

**Thuật toán token bucket**

Thuật toán token bucket được sử dụng rộng rải để giới hạn truy cập. Nó đơn giản, dễ hiểu và được dùng phổ biến, các công ty như Amazon [5] và Stripe [6] đều sử dụng thuật toán này để điều tiết yêu cầu API.

Thuật toán token bucket hoạt động như sau:
- Một bucket là một thùng chứa có dung lượng đã xác định trước. Các token là các đồng tiền được đặt vào theo định kỳ. Khi mà bucket đã đầy thì không có token nào được thêm vào. Như hình bên dưới, dung lượng bucket là 4. Mỗi giây, bộ nạp sẽ đặt 2 token vào. Khi bucket đầy các token tiếp theo sẽ bị tràn ra ngoài.

![](./assets/token-bucket.png)

- Mỗi yêu cầu từ client ứng với một token. Khi yêu cầu đến ta kiểm tra trong bucket có đủ token không. 
    + Nếu đủ ta lấy một token cho mỗi yêu cầu, yêu cầu sẽ được xử lý tiếp.
    + Nếu không đủ, yêu cầu sẽ bị xoá.

![](./assets/token-request.png)

Hình bên dưới minh hoạ cách tiêu thụ và nạp token cũng như logic hoạt động của giới hạn truy cập.

![](./assets/token-flow.png)

Thuật toán token bucket nhận vào hai tham số:
- Kích cỡ bucket: số lượng tối đa token ở trong một bucket.
- Chu kỳ cấp phát: là số token được đặt vào bucket trong một giây.

Vậy thì ta sẽ cần bao nhiêu bucket? Không có câu trả lời cố định, nó tuỳ thuộc vào quy tắc mà ta thiết lập giới hạn truy cập. Ở đây có một vài ví dụ:
- Nếu ta cần các bucket khác nhau cho các API endpoint khác nhau. Ví dụ: một người dùng được cho phép đăng một bài viết một giây, có 150 người bạn một ngày và thích 5 bài viết một giây thì sẽ cần tới 3 bucket.
- Nếu ta cần điều tiết các yêu cầu dựa trên địa chỉ IP thì mỗi địa chỉ IP cần một bucket.
- Nếu hệ thống chỉ cho phép tối đa 10,000 yêu cầu trên một giây ta sẽ cần một bucket toàn cục chung cho tất cả yêu cầu.

*Ưu điểm*
1. Dễ dàng triển khai
2. Lưu trữ hiệu quả
3. Cho phép một loạt truy cập trong chu kỳ ngắn. Yêu cầu có thể được thực hiện miễn là vẫn còn token.

*Nhược điểm*
1. Hai tham số trong thuật toán là kích cỡ bucket và chu kỳ cấp phát. Tuy nhiên khá khó khăn để điều chỉnh chúng đúng cách.

**Thuật toán leaking bucket**

Thuật toán này tương tự token bucket nhưng ngoại trừ việc các yêu cầu được xử lý theo tần suất cố định. Nó thường dùng triển khai hàng đợi FIFO. Thuật toán hoạt động như sau:
- Khi một yêu cầu đến, hệ thống kiểm tra nếu hàng đợi vẫn còn chỗ nó sẽ được thêm vào hàng đợi.
- Ngược lại nó sẽ bị xoá.
- Yêu cầu được lấy từ hàng đợi và xử lý.

![](./assets/bucket.png)

Thuật toán nhận về hai tham số:
- Kích cỡ bucket: bằng với kích cỡ hàng đợi. Hàng đợi giữ yêu cầu cần được xử lý trong một tần suất cố định.
- Tần suất thoát ra: nó xác định bao nhiêu yêu cầu sẽ được xử lý trong một tần suất cố định thường là giây.

*Ưu điểm*
1. Lưu trữ hiệu quả với kích cỡ hàng đợi giới hạn.
2. Yêu cầu được xử lý trong một khoảng cố định do đó phù hợp với các trường hợp sử dụng cần tần suất thoát ra ổn định.

*Nhược điểm*
1. Một loạt truy cập sẽ lấp đầy hàng đợi và với các yêu cầu cũ nếu chúng không được xử lý kịp thời, các yêu cầu gần nhất sẽ bị giới hạn truy cập.
2. Có hai tham số trong thuật toán. Và không dễ dàng để điều chỉnh chúng đúng cách.

*Thuận toán fixed window counter*

Thuật toán hoạt động như sau:
- Thuật toán chia dòng thời gian thành các cửa sổ thời gian cố định và gán một bộ đếm cho mỗi cửa sổ.
- Mỗi yêu cầu sẽ tăng bộ đếm lên một.
- Khi bộ đếm đạt đến ngưỡng được xác định trước, các yêu cầu mới sẽ bị loại bỏ cho đến khi cửa sổ thời gian mới bắt đầu.

Ta lấy ví dụ cụ thể. Ở hình bên dưới đơn vị thời gian là 1s, hệ thống cho phép tối đa 3 yêu cầu trên một giây. Với mỗi cửa sổ thứ hai, nếu nhận được hơn 3 yêu cầu, các yêu cầu tiếp theo sẽ bị loại bỏ.

![](./assets/fixed-window.png)

Một vấn đề lớn với thuật toán này là một loạt lưu lượng truy cập ở các cạnh của cửa sổ thời gian có thể gây ra nhiều yêu cầu hơn định mức cho phép. Hãy xem xét trường hợp sau:

![](./assets/fixed-flow.png)

Trong hình trên hệ thống cho phép tối đa 5 yêu cầu mỗi phút, định mức khả dụng đặt lại gần bằng phút. Và có 5 yêu cầu trong khoảng thời gian từ 2:00:00 đến 2:01:00 và 5 yêu cầu khác trong khoảng 2:01:00 đến 2:02:00. Đối với khoảng thời gian một phút từ 2:00:30 đến 2:01:30, thì lại có 10 yêu cầu. Gấp đôi với số yêu cầu được cho phép.

*Ưu điểm*
1. Lưu trữ hiệu quả
2. Dễ hiểu
3. Đặt lại định mức khả dụng vào cuối cửa sổ đơn vị thời gian phù hợp với một số trường hợp sử dụng nhất định.

*Nhược điểm*
1. Lưu lượng truy cập tăng vọt ở các cạnh của cửa sổ có thể gây ra nhiều yêu cầu hơn định mức cho phép.

**Thuật toán Sliding window log**

Như đã thảo luận trước đây, thuật toán fixed window counter có một vấn đề lớn là: nó cho phép nhiều yêu cầu hơn đi qua các cạnh của cửa sổ. Thuật toán sliding window log khắc phục vấn đề đó. Nó hoạt động như sau:
- Thuật toán theo dõi các dấu thời gian của yêu cầu. Dữ liệu dấu thời gian thường được lưu trong bộ nhớ cache, chẳng hạn như các set được sắp xếp của Redis [8].
- Khi có yêu cầu mới, hãy xóa tất cả các dấu thời gian đã lỗi thời. Dấu thời gian lỗi thời được định nghĩa là những dấu cũ hơn thời điểm bắt đầu của cửa sổ thời gian hiện tại.
- Thêm dấu thời gian của yêu cầu mới vào log.
- Nếu kích thước log bằng hoặc thấp hơn số lượng cho phép, một yêu cầu được chấp nhận.
- Nếu không, nó bị từ chối.
Ta có hình minh hoạ như sau:

![](./assets/slicing.png)

Ở ví dụ này, bộ giới hạn truy cập cho phép 2 yêu cầu mỗi phút. Thông thường, dấu thời gian Linux được lưu ở log. Tuy nhiên, trong ví dụ này ta sử dụng biểu diễn thời gian dễ đọc hơn để dễ hiểu hơn.

- Log trống khi yêu cầu mới đến vào lúc 1:00:01. Do đó yêu cầu được cho phép.
- Một yêu cầu mới đến vào lúc 1:00:30, dấu thời gian 1:00:30 được lưu vào log. Sau khi lưu, kích cở log là 2 không lớn hơn con số cho phép. Do đó, yêu cầu vẫn được cho phép.
- Yêu cầu mới đến vào lúc 1:00:50 và được thêm dấu thời gian vào log. Sau khi chèn, kích cỡ log là 3, lớn hơn 2. Nên yêu cầu này bị từ chối, mặc dù dấu thời gian vẫn còn trong log.
- Yêu cầu mới đến vào lúc 1:01:40. Các yêu cầu trong khoảng [1:00:40, 1:01:40] nằm trong một khung thời gian, nhưng yêu cầu được gửi đến trước 1:00:40 đã lỗi thời. Hai dấu thời gian là 1:00:01 và 1:00:30 đều đã bị xoá khởi log. Do đó kích cở bây giờ là 2 nên yêu cầu được cho phép

*Ưu điểm*
1. Giới hạn truy cập được thực hiện bởi thuật toán này là rất chính xác. Trong bất kỳ cửa sổ luân phiên nào, các yêu cầu sẽ không vượt quá giới hạn truy cập.

*Nhược điểm*
1. Thuật toán tiêu thụ nhiều bộ nhớ vì ngay cả khi yêu cầu bị từ chối, dấu thời gian của nó vẫn được lưu lại.

**Thuật toán sliding window counter**

Là cách tiếp cận kết hợp fixed window counter và sliding window log. Thuật toán có thể triển khai bằng hai cách khác nhau. Chúng ta sẽ chỉ giải thích một cách triển khai trong thôi, cách còn lại sẽ được cung cấp tài liệu tham khảo ở cuối bài.

Ảnh dưới đây mô tả cách hoạt động của thuật toán

![](./assets/siding.png)

Giả sử bộ giới hạn truy cập cho phép tối đa 7 yêu cầu mỗi phút và có 5 yêu cầu trong phút trước và 3 yêu cầu trong phút hiện tại. Đối với một yêu cầu mới đến vị trí 30% trong phút hiện tại, số lượng yêu cầu trong cửa sổ luân phiên được tính bằng công thức sau:
> Số yêu cầu ở cửa sổ hiện tại + số yêu cầu ở cửa số trước * phần trăm chồng chéo của cửa sổ luân phiên và cửa sổ trước đó.

Sử dụng công thức trên ta có `3 + 5 * 0.7% = 6.5` yêu cầu. Tuỳ vào trường hợp mà con số có thể được làm tròn lên hoặc xuống. Ở đây ta làm tròn xuống còn 6.

Vì bộ giới hạn truy cập cho phép tối đa 7 yêu cầu mỗi phút, nên yêu cầu hiện tại có thể được thực hiện. Tuy nhiên, giới hạn sẽ đạt được sau khi nhận được thêm một yêu cầu.

Do giới hạn về không gian, chúng ta sẽ không thảo luận về cách triển khai khác ở đây. Bạn đọc quan tâm có thể tham khảo tài liệu tham khảo [9]. Thuật toán này tuy là cải tiến và kết hợp của hai thuật toán trên nhưng nó không hoàn hảo. Nó có ưu và nhược điểm sau:

*Ưu điểm*
1. Lưu trữ hiệu quả
2. Nó làm giảm lượng truy cập tăng đột biến vì truy cập dựa trên tỷ lệ trung bình của cửa sổ trước đó.

*Nhược điểm*
1. Nó chỉ hoạt động đối với cửa sổ xem lại không quá nghiêm ngặt. Đây là giá trị gần đúng của tỷ lệ thực tế vì nó giả định các yêu cầu trong cửa sổ trước đó được phân phối đồng đều. Tuy nhiên, vấn đề này có thể không quá tệ như bạn tưởng. Theo các thử nghiệm được thực hiện bởi Cloudflare [10], chỉ có 0,003% yêu cầu được cho phép sai hoặc truy cập bị giới hạn trong số 400 triệu yêu cầu.