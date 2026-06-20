---
title: "Từ Bit Flip đến Cache Leak: Fault Injection, DFA Attack và Giới Hạn Thật Sự của Lockstep Defense"
published: 2026-06-05
description: "Hardware Security & Computer Architecture"
image: './banner.png'
tags: ["HARDWARE", "SECURITY", "COMPUTER-ARCHITECTURE"]
category: "Hardware-Security"
draft: false
---

Khi nhắc đến các cuộc tấn công vào hệ thống máy tính, chúng ta thường nghĩ đến malware, buffer overflow, SQL injection hay các kỹ thuật khai thác lỗ hổng phần mềm. Tuy nhiên, không phải mọi sự cố đều bắt nguồn từ một dòng code lỗi hoặc một attacker ngồi phía bên kia màn hình.

Ít ai biết rằng đôi khi nguyên nhân có thể đến từ một thứ rất xa xôi: các hạt năng lượng cao xuất phát từ ngoài không gian.

Bầu khí quyển Trái Đất đóng vai trò như một lớp lá chắn tự nhiên trước bức xạ vũ trụ, nhưng một số hạt năng lượng đủ cao vẫn xuyên xuống tới mặt đất. Khi một hạt như vậy đi qua vùng bán dẫn bên trong chip nhớ hoặc bộ xử lý, nó có thể tạo ra một lượng điện tích nhỏ làm thay đổi trạng thái của một transistor đang lưu dữ liệu.

Hãy tưởng tượng bộ nhớ máy tính đang lưu trữ hàng tỷ bit dữ liệu dưới dạng các giá trị 0 và 1. Nếu chỉ một bit duy nhất bị thay đổi từ 0 thành 1 hoặc ngược lại, dữ liệu có thể bị sai lệch. Hiện tượng này được gọi là bit flip. Trong phần cứng, đây là một dạng lỗi mềm (soft error) vì bản thân linh kiện không bị hư hỏng vật lý; dữ liệu chỉ bị thay đổi tạm thời do tác động của môi trường.

Nghe có vẻ khó tin, nhưng các lỗi kiểu này hoàn toàn có thật và đã được ghi nhận trong nhiều hệ thống từ máy chủ dữ liệu, thiết bị hàng không cho đến vệ tinh ngoài không gian. Khi kích thước transistor ngày càng thu nhỏ, lượng điện tích cần thiết để làm thay đổi trạng thái của một bit cũng giảm theo, khiến các hệ thống hiện đại trở nên nhạy cảm hơn với hiện tượng này.

Đó cũng chính là lý do các cơ chế như ECC Memory, Chipkill, Triple Modular Redundancy (TMR) hay Lockstep Execution được phát triển. Thay vì cố gắng ngăn chặn hoàn toàn các tác động từ môi trường — điều gần như bất khả thi — các kiến trúc này được thiết kế để phát hiện, sửa chữa hoặc chịu đựng lỗi khi chúng xảy ra, qua đó đảm bảo hệ thống vẫn hoạt động chính xác và đáng tin cậy.

## 1. Một Vài Khái Niệm Cần Biết Trước Khi Bắt Đầu

### 1.1. Fault, Error và Failure

Trong lĩnh vực hệ thống chịu lỗi (*Fault-Tolerant Systems*), ba thuật ngữ **Fault**, **Error** và **Failure** mô tả ba giai đoạn khác nhau của một sự cố.

* **Fault** là nguyên nhân gốc rễ gây ra lỗi.
* **Error** là trạng thái sai lệch xuất hiện bên trong hệ thống.
* **Failure** là thời điểm hệ thống tạo ra kết quả không còn đúng với đặc tả thiết kế.

Ví dụ:

1. Một hạt năng lượng cao đi xuyên qua chip nhớ (**Fault**).
2. Một bit trong bộ nhớ bị đảo từ `0` thành `1` (**Error**).
3. Chương trình sử dụng dữ liệu sai và đưa ra kết quả không chính xác (**Failure**).

Chuỗi này có thể được mô tả như sau:

```text
Fault
  ↓
Error
  ↓
Failure
```

Điều quan trọng là không phải mọi *Fault* đều dẫn tới *Failure*. Nhiều cơ chế chịu lỗi hiện đại được thiết kế để phát hiện và xử lý lỗi trước khi chúng ảnh hưởng đến kết quả cuối cùng của hệ thống.

### 1.2. Soft Error và Hard Error

Không phải mọi lỗi phần cứng đều giống nhau.

**Hard Error** là các hư hỏng vật lý của linh kiện, chẳng hạn như transistor bị hỏng, đường tín hiệu bị đứt hoặc chip bị lỗi vĩnh viễn. Những lỗi này thường yêu cầu sửa chữa hoặc thay thế phần cứng.

Ngược lại, **Soft Error** không làm hỏng phần cứng — nó chỉ làm thay đổi tạm thời trạng thái logic đang được lưu trữ hoặc xử lý bên trong hệ thống. Sau khi dữ liệu được ghi lại hoặc hệ thống được khởi động lại, lỗi có thể hoàn toàn biến mất.

Trong các hệ thống điện toán hiện đại, Soft Error là mối quan tâm đặc biệt vì nó có thể xảy ra ngay cả khi phần cứng vẫn hoạt động hoàn toàn bình thường.

Các cơ chế như ECC, Chipkill, TMR hay Lockstep Execution được phát triển chủ yếu để phát hiện, sửa chữa hoặc chịu đựng loại lỗi này.

---

## 2. Khi Một Bit Bị Lật

Trong phần trước, chúng ta đã biết rằng một *Fault* có thể dẫn đến *Error* và cuối cùng là *Failure*. Tuy nhiên, một câu hỏi thú vị hơn là:

> Fault đó đến từ đâu?

Khi nghĩ đến lỗi phần cứng, nhiều người thường hình dung đến chip bị cháy, nguồn điện không ổn định hoặc linh kiện xuống cấp theo thời gian. Nhưng trên thực tế, một trong những nguyên nhân phổ biến nhất của *Soft Error* lại đến từ một thứ nằm ngoài Trái Đất: bức xạ vũ trụ (*Cosmic Radiation*).

Nghe có vẻ giống khoa học viễn tưởng, nhưng đây là một hiện tượng hoàn toàn có thật. Các hạt năng lượng cao được sinh ra từ những sự kiện trong vũ trụ như bùng phát Mặt Trời hoặc các vụ nổ sao siêu mới. Khi đi vào khí quyển Trái Đất, chúng va chạm với các phân tử không khí và tạo ra nhiều hạt thứ cấp khác nhau, trong đó neutron năng lượng cao là một trong những tác nhân chính gây ra lỗi cho các hệ thống điện tử hiện đại.

### 2.1. Single Event Upset (SEU) là gì?

Một trong những dạng lỗi phổ biến nhất do bức xạ gây ra được gọi là **Single Event Upset (SEU)**.

SEU là hiện tượng trạng thái logic của một phần tử lưu trữ bị thay đổi bởi một sự kiện vật lý đơn lẻ. Nói đơn giản hơn, một bit đang mang giá trị `0` có thể trở thành `1`, hoặc ngược lại, mà không hề có bất kỳ lệnh ghi dữ liệu nào từ CPU.

Điều quan trọng là SEU không làm hỏng phần cứng. Sau khi dữ liệu được ghi lại hoặc hệ thống được khởi động lại, thiết bị vẫn có thể hoạt động bình thường. Chính vì vậy, SEU được xem là một dạng *Soft Error*.

Tuy nhiên, hậu quả của nó không hề nhỏ nếu bit bị thay đổi nằm trong dữ liệu quan trọng, thanh ghi điều khiển hoặc các cấu trúc lưu trữ quan trọng của hệ thống.

### 2.2. Tia vũ trụ tác động tới transistor như thế nào?

Để hiểu SEU xảy ra như thế nào, chúng ta cần nhìn vào thế giới bên trong chip bán dẫn.

Khi một hạt năng lượng cao đi xuyên qua transistor hoặc ô nhớ SRAM/DRAM, nó có thể tạo ra một lượng điện tích nhỏ trên đường đi. Trong hầu hết các trường hợp, lượng điện tích này không đủ để gây ảnh hưởng. Tuy nhiên, nếu điện tích sinh ra vượt quá ngưỡng ổn định của phần tử lưu trữ, trạng thái logic đang được lưu sẽ bị thay đổi.

Quá trình này có thể được mô tả đơn giản như sau:

```text
Cosmic Radiation
        │
        ▼
 High-Energy Particle
        │
        ▼
 Charge Generation
        │
        ▼
    Bit Flip
```

Nói cách khác, một hiện tượng vật lý ở cấp độ nguyên tử có thể trực tiếp làm thay đổi dữ liệu mà phần mềm đang sử dụng.

Điều đáng chú ý là hiện tượng này không chỉ xảy ra trong vệ tinh hoặc tàu vũ trụ. Các nghiên cứu đã chỉ ra rằng soft error do bức xạ vẫn có thể xuất hiện ngay tại mặt đất, đặc biệt trong các hệ thống máy chủ có dung lượng bộ nhớ lớn và hoạt động liên tục.

### 2.3. Vì sao transistor càng nhỏ càng dễ gặp Soft Error?

Trực giác thông thường có thể khiến chúng ta nghĩ rằng công nghệ càng hiện đại thì càng đáng tin cậy hơn. Tuy nhiên, thực tế lại phức tạp hơn.

Khi tiến trình bán dẫn liên tục thu nhỏ, transistor ngày càng nhỏ hơn, điện áp hoạt động thấp hơn và lượng điện tích dùng để lưu trữ dữ liệu cũng giảm theo.

Điều này mang lại nhiều lợi ích:

* Tiêu thụ điện năng thấp hơn.
* Mật độ transistor cao hơn.
* Hiệu năng tốt hơn.

Tuy nhiên, nó cũng khiến các phần tử lưu trữ trở nên nhạy cảm hơn với các tác động từ môi trường. Một lượng điện tích nhỏ vốn không đủ gây lỗi ở các thế hệ công nghệ cũ giờ đây có thể làm thay đổi trạng thái của một bit.

Nói cách khác:

> Các transistor hiện đại nhanh hơn và tiết kiệm điện hơn, nhưng đồng thời cũng dễ bị ảnh hưởng bởi hiện tượng bit flip hơn trước.

### 2.4. Single-Bit Upset và Multiple-Bit Upset

Trong các thế hệ phần cứng trước đây, một sự kiện bức xạ thường chỉ ảnh hưởng đến một ô nhớ duy nhất.

Kết quả là xuất hiện **Single-Bit Upset (SBU)**:

```text
00000000
    │
    ▼
00001000
```

Chỉ một bit bị thay đổi.

Đây cũng chính là loại lỗi mà các cơ chế ECC truyền thống được thiết kế để phát hiện và sửa chữa.

Tuy nhiên, khi mật độ transistor ngày càng cao, các ô nhớ được đặt gần nhau hơn rất nhiều. Một hạt năng lượng cao giờ đây có thể ảnh hưởng đồng thời đến nhiều ô nhớ lân cận.

Kết quả là xuất hiện **Multiple-Bit Upset (MBU)**:

```text
00000000
    │
    ▼
00111000
```

Nhiều bit bị thay đổi trong cùng một sự kiện.

MBU là một thách thức lớn đối với các cơ chế ECC truyền thống và cũng là một trong những lý do thúc đẩy sự ra đời của các kỹ thuật bảo vệ nâng cao như Chipkill.

---

## 3. Điều Gì Xảy Ra Sau Một Bit Flip?

Ở phần trước, chúng ta đã thấy rằng một hạt năng lượng cao có thể làm thay đổi trạng thái của một bit trong bộ nhớ hoặc bên trong logic của bộ xử lý. Tuy nhiên, bản thân việc một bit bị đảo không phải lúc nào cũng dẫn đến sự cố.

Điều thực sự quan trọng không phải là:

> "Bit có bị lật hay không?"

mà là:

> "Sau khi bit bị lật, chuyện gì sẽ xảy ra tiếp theo?"

Trong Reliability Engineering, đây chính là giai đoạn mà một *Error* có thể bị phát hiện, được sửa chữa hoặc tiếp tục lan truyền cho đến khi trở thành *Failure*.

### 3.1. Silent Data Corruption (SDC)

Một trong những kịch bản nguy hiểm nhất được gọi là **Silent Data Corruption (SDC)**.

Đây là trường hợp dữ liệu đã bị sai lệch nhưng hệ thống không phát hiện được lỗi.

Ví dụ:

```text
Giá trị đúng:
Balance = 1000

Bit Flip

Giá trị sau lỗi:
Balance = 1008
```

Nếu không có cơ chế kiểm tra nào phát hiện sự thay đổi này, chương trình sẽ tiếp tục sử dụng giá trị mới như thể nó hoàn toàn hợp lệ.

Điều khiến SDC trở nên nguy hiểm là hệ thống:

* Không báo lỗi.
* Không ghi log bất thường.
* Không kích hoạt cơ chế khôi phục.

Kết quả cuối cùng có thể sai, nhưng không ai biết rằng lỗi đã từng xảy ra.

Trong nhiều trường hợp, SDC còn nguy hiểm hơn cả việc hệ thống dừng hoạt động vì dữ liệu sai vẫn tiếp tục được sử dụng và lan truyền sang các thành phần khác.

### 3.2. Detected Unrecoverable Error (DUE)

Ngược lại với SDC là **Detected Unrecoverable Error (DUE)**.

Trong trường hợp này, hệ thống biết rằng lỗi đã xảy ra nhưng không thể tự sửa chữa.

Ví dụ:

```text
ECC phát hiện lỗi
        │
        ▼
Khả năng sửa lỗi bị vượt quá
        │
        ▼
Dữ liệu không còn đáng tin cậy
```

Khi đó hệ thống có thể:

* Dừng tiến trình.
* Hủy thao tác đang thực hiện.
* Khởi động lại thành phần liên quan.
* Chuyển sang chế độ an toàn.

Mặc dù gây gián đoạn hoạt động, DUE vẫn thường được xem là tốt hơn SDC vì ít nhất hệ thống biết rằng dữ liệu đã bị lỗi.

### 3.3. Error Propagation

Một bit flip ban đầu thường không đứng yên.

Nếu dữ liệu bị lỗi được sử dụng trong các phép tính tiếp theo, lỗi có thể tiếp tục lan truyền qua nhiều tầng của hệ thống.

```text
Bit Flip
    ↓
Corrupted Data
    ↓
Wrong Computation
    ↓
Corrupted Output
    ↓
Failure
```

Ví dụ, một bit bị lật trong bộ nhớ có thể làm sai tham số đầu vào của một thuật toán. Thuật toán đó tiếp tục tạo ra kết quả sai, sau đó kết quả lại được lưu trữ hoặc sử dụng bởi các thành phần khác.

Quá trình này được gọi là **Error Propagation**.

Điều đáng chú ý là chi phí xử lý lỗi tăng lên rất nhanh theo thời gian. Một lỗi được sửa ngay tại bộ nhớ thường chỉ mất vài chu kỳ xử lý. Nhưng nếu lỗi đã lan tới hệ điều hành, cơ sở dữ liệu hoặc ứng dụng, hậu quả có thể nghiêm trọng hơn rất nhiều.

Vì lý do đó, các kiến trúc chịu lỗi hiện đại luôn cố gắng phát hiện và xử lý lỗi càng gần nguồn phát sinh càng tốt.

---

## 4. ECC: Phát Hiện Và Sửa Lỗi Bằng Toán Học

Ở phần trước, chúng ta đã thấy rằng một bit flip không nhất thiết dẫn đến Failure ngay lập tức. Tuy nhiên, điều đó đặt ra một câu hỏi quan trọng:

> Làm thế nào hệ thống biết rằng một bit đã bị thay đổi?

Nếu chỉ đơn giản ghi dữ liệu xuống bộ nhớ rồi đọc lại, CPU không có cách nào phân biệt giữa dữ liệu đúng và dữ liệu đã bị lỗi.

Đây chính là lý do các hệ thống hiện đại sử dụng **Error Correcting Code (ECC)**.

Thay vì chỉ lưu dữ liệu, ECC bổ sung thêm một lượng thông tin dư thừa (*redundancy*) được tính toán từ dữ liệu gốc. Khi đọc lại, hệ thống dùng thông tin này để:

* Phát hiện lỗi
* Xác định vị trí lỗi
* Và trong một số trường hợp, tự động sửa lỗi

Toàn bộ cơ chế này được xây dựng dựa trên nền tảng toán học.

---

### 4.1. Tại Sao Chỉ Lưu Dữ Liệu Là Chưa Đủ?

Giả sử ta lưu một byte:

```text
10110010
```

Sau một bit flip, dữ liệu có thể trở thành:

```text
10100010
```

Nếu không có thông tin bổ sung, hai chuỗi này đều hoàn toàn hợp lệ về mặt biểu diễn nhị phân. Hệ thống không có cách nào biết dữ liệu đã bị thay đổi.

Vì vậy, cần thêm thông tin để kiểm tra tính toàn vẹn dữ liệu. Đó chính là ý tưởng cốt lõi của ECC.

---

### 4.2. Parity Bit: Cơ Chế Phát Hiện Lỗi Đơn Giản Nhất

Cách đơn giản nhất là **Parity Bit**.

Ví dụ:

```text
10110010 | 0
```

Nếu một bit bị đảo:

```text
10100010 | 0
```

số lượng bit `1` thay đổi, và hệ thống có thể phát hiện có lỗi.

Tuy nhiên, parity chỉ trả lời được một câu hỏi:

> Có lỗi hay không?

chứ không biết:

> Lỗi nằm ở đâu?

Vì vậy, parity chỉ là cơ chế **Error Detection**, không phải Error Correction.

---

### 4.3. Hamming Distance: Nền Tảng Của ECC

Để đi xa hơn, ta cần khái niệm **Hamming Distance** — số bit khác nhau giữa hai chuỗi nhị phân.

Ví dụ:

```text
10110010
10100010
```

→ Hamming Distance = 1

```text
10110010
00100110
```

→ Hamming Distance = 3

Khoảng cách càng lớn, khả năng phát hiện và sửa lỗi càng mạnh.

Đây là nền tảng toán học của mọi hệ thống ECC.

---

### 4.4. Syndrome Decoding: Xác Định Bit Bị Lỗi

Ý tưởng của ECC không chỉ là phát hiện lỗi, mà còn xác định chính xác vị trí lỗi.

Để làm điều đó, dữ liệu được đi kèm với các parity bit được thiết kế theo cấu trúc đặc biệt.

Khi đọc dữ liệu:

1. Hệ thống tính lại các parity
2. So sánh với giá trị ban đầu
3. Tạo ra một vector gọi là **Syndrome**

Syndrome đóng vai trò như “chữ ký của lỗi”.

Ví dụ:

```text
Syndrome = 0101
```

→ ánh xạ tới một vị trí bit cụ thể bị lỗi

Sau đó hệ thống chỉ cần đảo bit đó để khôi phục dữ liệu gốc.

Toàn bộ quá trình này diễn ra ở phần cứng và gần như không ảnh hưởng đến CPU hay phần mềm.

---

### 4.5. SECDED: Chuẩn ECC Trong Bộ Nhớ Hiện Đại

Trong thực tế, phổ biến nhất là **SECDED (Single Error Correction, Double Error Detection)**.

SECDED có khả năng:

* Sửa 1 bit lỗi
* Phát hiện 2 bit lỗi

nhưng không thể sửa lỗi khi có nhiều hơn 1 bit bị sai.

Đây là sự đánh đổi giữa:

* Độ tin cậy
* Chi phí phần cứng
* Độ trễ truy cập bộ nhớ

---

### 4.6. Từ 64-bit Thành 72-bit

Một hệ thống ECC điển hình không chỉ lưu dữ liệu gốc.

Khi CPU ghi 64-bit dữ liệu:

```text
64 data bits
```

Memory Controller sẽ tính thêm:

```text
8 ECC bits
```

tạo thành một codeword 72-bit:

```text
┌───────────────┬─────────┐
│ 64 Data Bits │ ECC Bits│
└───────────────┴─────────┘
```

Chính 8 bit dư này cho phép phát hiện và sửa lỗi khi đọc dữ liệu.

---

### 4.7. ECC Không Nằm Trong DRAM

Một điểm quan trọng: ECC thường **không nằm trong chip DRAM**.

Thay vào đó, nó được xử lý bởi **Memory Controller**:

Ghi dữ liệu:

```text
CPU → Memory Controller → ECC Encode → DRAM
```

Đọc dữ liệu:

```text
DRAM → Memory Controller → ECC Decode → CPU
```

Quá trình phát hiện và sửa lỗi xảy ra hoàn toàn trước khi dữ liệu đến CPU.

---

### 4.8. Khi ECC Không Đủ

ECC hoạt động tốt khi lỗi chỉ ảnh hưởng một bit. Nhưng khi nhiều bit bị lỗi đồng thời, hệ thống bắt đầu gặp giới hạn.

#### Multi-bit Error

```text
10110010 → 10000010
```

ECC có thể phát hiện lỗi nhưng không thể xác định chính xác bit nào cần sửa. Khi đó hệ thống sẽ báo:

**Detected Unrecoverable Error (DUE)**

---

#### Burst Error

```text
11111111 → 11000011
```

Đây là dạng lỗi nhiều bit liền kề, thường do:

* nhiễu đường truyền
* vùng nhớ bị ảnh hưởng cục bộ
* hoặc sự kiện bức xạ tác động rộng hơn

Khi mật độ transistor ngày càng cao, dạng lỗi này trở nên phổ biến hơn.

---

ECC vì vậy không phải là giới hạn cuối cùng của khả năng chịu lỗi. Khi số bit lỗi vượt quá khả năng sửa của SECDED, cần đến các cơ chế mạnh hơn — và đó chính là lý do xuất hiện **Chipkill**, chủ đề của phần tiếp theo.

---

## 5. Chipkill: Khi Lỗi Không Còn Chỉ Là Một Bit

Ở phần trước, chúng ta đã thấy ECC có thể phát hiện và sửa lỗi rất hiệu quả khi chỉ có một bit bị đảo.

Nhưng giả định đó dần không còn đúng trong các hệ thống hiện đại.

Khi mật độ DRAM tăng lên và dung lượng bộ nhớ mở rộng lên hàng chục, thậm chí hàng trăm GB, lỗi không còn xuất hiện đơn lẻ nữa. Một sự kiện vật lý như nhiễu điện hoặc bức xạ có thể ảnh hưởng đồng thời nhiều bit, đặc biệt là khi các ô nhớ nằm gần nhau trong cùng một chip.

Lúc này, vấn đề không còn là:

> “Sửa một bit bị lỗi”

mà trở thành:

> “Xử lý thế nào khi một phần đáng kể của bộ nhớ hoặc thậm chí cả một chip DRAM gặp sự cố”

Đó là giới hạn mà ECC truyền thống bắt đầu gặp phải.

---

### 5.1. Vì Sao ECC Không Còn Đủ?

SECDED hoạt động tốt khi lỗi mang tính cục bộ:

```text
00000000
    ↓
00001000
```

Nhưng khi nhiều bit bị ảnh hưởng cùng lúc:

```text
00000000
    ↓
00111000
```

ECC chỉ có thể kết luận rằng:

> “Có lỗi xảy ra”

nhưng không đủ thông tin để xác định chính xác vị trí cần sửa.

Vấn đề nghiêm trọng hơn là lỗi trong DRAM thường không phân bố ngẫu nhiên, mà có xu hướng theo cụm (*clustered errors*):

* Một chip DRAM bị lỗi hoàn toàn
* Một vùng nhớ bị ảnh hưởng cục bộ
* Một đường truyền dữ liệu gặp sự cố

Trong các trường hợp này, số lượng bit lỗi có thể vượt quá khả năng sửa của ECC.

---

### 5.2. Ý Tưởng Cốt Lõi Của Chipkill

Chipkill được thiết kế với một mục tiêu rõ ràng:

> Hệ thống vẫn hoạt động ngay cả khi một chip DRAM hoàn toàn bị lỗi.

Điểm khác biệt quan trọng là Chipkill không xem bộ nhớ như một dãy bit liên tục nữa, mà xem mỗi chip như một đơn vị có thể thất bại.

Thay vì lưu toàn bộ dữ liệu trong một chip, dữ liệu được **phân tán trên nhiều chip khác nhau**.

---

### 5.3. Data Striping Trên DRAM

Trong ECC truyền thống, một khối dữ liệu có thể nằm gần như trọn vẹn trong một vùng nhớ.

Chipkill thì làm ngược lại: nó trải dữ liệu ra nhiều chip.

```text
Data Word
 ├─ Part A → Chip 0
 ├─ Part B → Chip 1
 ├─ Part C → Chip 2
 ├─ Part D → Chip 3
 ├─ Part E → Chip 4
 ├─ Part F → Chip 5
 ├─ Part G → Chip 6
 └─ Part H → Chip 7
```

Điều này tạo ra một tính chất quan trọng:

> Một chip hỏng không đồng nghĩa với mất toàn bộ một codeword.

Thay vào đó, mỗi codeword chỉ mất một phần nhỏ dữ liệu.

---

### 5.4. Từ Bit-Level Sang Symbol-Level

ECC truyền thống (như Hamming Code) hoạt động ở mức **bit**.

Chipkill nâng cấp cách tiếp cận lên mức **symbol**.

Một symbol thường là:

```text
4-bit hoặc 8-bit group
```

Điều này phản ánh thực tế tốt hơn, vì lỗi DRAM thường không chỉ ảnh hưởng một bit đơn lẻ mà có thể ảnh hưởng cả nhóm bit liền kề.

Thay vì hỏi:

> Bit nào bị lỗi?

Chipkill đặt câu hỏi:

> Symbol nào bị lỗi?

---

### 5.5. Reed-Solomon: Nền Tảng Toán Học Của Chipkill

Để xử lý lỗi ở mức symbol, Chipkill thường dựa trên **Reed-Solomon Code**.

Khác với Hamming Code:

* Hamming Code → hoạt động trên bit
* Reed-Solomon → hoạt động trên symbol

Ý tưởng cốt lõi là thêm các symbol dư thừa để có thể khôi phục dữ liệu ngay cả khi một phần codeword bị mất.

---

### 5.6. Symbol Reconstruction

Giả sử dữ liệu được chia thành các symbol:

```text
S1 S2 S3 S4 S5 S6 S7 S8
```

Nếu một phần bị mất:

```text
S1 S2 ?? S4 S5 S6 S7 S8
```

Reed-Solomon sử dụng các symbol còn lại để tái tạo lại phần bị thiếu.

Về mặt toán học, đây là bài toán giải hệ phương trình trên trường hữu hạn. Nhưng ở mức hệ thống, ý nghĩa đơn giản là:

> Dữ liệu vẫn có thể được khôi phục ngay cả khi một phần bộ nhớ không còn khả dụng.

---

### 5.7. Khi Một DRAM Chip Bị Lỗi Hoàn Toàn

Điểm quan trọng nhất của Chipkill là khả năng chịu lỗi ở cấp chip.

```text
Chip 0  ✓
Chip 1  ✓
Chip 2  ✓
Chip 3  ✗
Chip 4  ✓
Chip 5  ✓
Chip 6  ✓
Chip 7  ✓
```

Trong ECC thông thường, lỗi kiểu này gần như phá hủy toàn bộ codeword.

Nhưng với Chipkill:

1. Hệ thống phát hiện chip bị lỗi
2. Xác định các symbol bị mất
3. Reed-Solomon tái tạo dữ liệu
4. Dữ liệu được phục hồi trước khi CPU sử dụng

Từ góc nhìn của ứng dụng:

```text
Chip Failure
     ↓
Reconstructed Data
     ↓
No Visible Corruption
```

---

### 5.8. Vì Sao Chipkill Quan Trọng Trong Datacenter?

Ở quy mô nhỏ (ví dụ một máy tính cá nhân), lỗi DRAM là hiếm.

Nhưng ở quy mô datacenter:

* hàng chục nghìn máy chủ
* hàng trăm nghìn DIMM
* hàng triệu chip DRAM hoạt động liên tục

thì những sự kiện “hiếm” trở thành chuyện xảy ra thường xuyên.

Một lỗi có xác suất rất nhỏ trên một chip đơn lẻ sẽ trở nên đáng kể khi nhân lên hàng triệu thành phần.

Vì vậy, trong hệ thống quy mô lớn, câu hỏi không còn là:

> “Có lỗi không?”

mà là:

> “Hôm nay sẽ có bao nhiêu lỗi xảy ra?”

Trong bối cảnh đó, Chipkill đóng vai trò như một lớp bảo vệ để đảm bảo rằng lỗi phần cứng không biến thành mất dữ liệu hoặc Silent Data Corruption ở quy mô lớn.

---

### 5.9. Sang Hướng Khác: Từ Bộ Nhớ Đến Bộ Xử Lý

Chipkill giải quyết bài toán lỗi trong **bộ nhớ**.

Nhưng nếu lỗi xảy ra bên trong **logic xử lý**, ví dụ trong ALU hoặc pipeline của CPU, thì ECC và Chipkill không còn đủ.

Lúc này, hệ thống cần một cách tiếp cận khác: **nhân bản và so sánh kết quả thay vì sửa dữ liệu**.

Đó chính là nền tảng của **Triple Modular Redundancy (TMR)**.

---

## 6. Triple Modular Redundancy (TMR): Đánh Đổi Tài Nguyên Để Lấy Độ Tin Cậy

Cho đến thời điểm này, chúng ta đã tập trung chủ yếu vào bộ nhớ.

ECC bảo vệ dữ liệu khỏi bit flip.

Chipkill mở rộng khả năng chịu lỗi khi lỗi lan rộng hoặc khi một DRAM chip gặp sự cố.

Nhưng khi lỗi xảy ra trong chính **logic xử lý**, ví dụ bên trong ALU hoặc pipeline của CPU, các cơ chế dựa trên kiểm tra dữ liệu như ECC không còn tác dụng.

Ví dụ:

```text
A = 5, B = 3
Expected: 8
Actual:   12
```

Dữ liệu trong bộ nhớ vẫn đúng, nhưng phép tính đã sai.

Đây là bài toán mà ECC không thể giải quyết.

Thay vào đó, các hệ thống yêu cầu độ tin cậy rất cao như vệ tinh, hàng không hay hệ thống điều khiển an toàn thường sử dụng một chiến lược khác:

> Không giả định phần cứng đúng. Giả định rằng lỗi chắc chắn sẽ xảy ra — và hệ thống phải tự bảo vệ mình.

Đó chính là ý tưởng của **Triple Modular Redundancy (TMR)**.

---

### 6.1. Ý Tưởng Cốt Lõi: Nhân Ba Và So Sánh

TMR không cố gắng sửa lỗi. Nó **che giấu lỗi bằng sự đồng thuận**.

Thay vì một module xử lý:

```text
Input → Module → Output
```

TMR sử dụng ba module chạy song song:

```text
Input → Module A ┐
Input → Module B ├→ Majority Voter → Output
Input → Module C ┘
```

Ba module thực hiện cùng một phép tính trên cùng một đầu vào. Kết quả cuối cùng được quyết định bởi **Majority Voter**.

Nếu một module bị lỗi:

```text
A = 42
B = 42
C = 99
```

hệ thống vẫn chọn:

```text
Output = 42
```

TMR hoạt động dựa trên một nguyên tắc đơn giản:

> Kết quả đúng là kết quả được đa số đồng thuận.

---

### 6.2. Majority Voter: Logic Cốt Lõi

Với tín hiệu 1-bit, bộ voter có thể được biểu diễn bằng:

```text
Y = AB + AC + BC
```

Điều này có nghĩa là:

* Nếu ít nhất 2 đầu vào là `1` → output = `1`
* Nếu ít nhất 2 đầu vào là `0` → output = `0`

Về mặt phần cứng, đây là một mạch logic đơn giản. Nhưng về mặt hệ thống, nó tạo ra một tính chất quan trọng:

> Một lỗi đơn lẻ không đủ để làm sai đầu ra.

---

### 6.3. Độ Tin Cậy Của TMR

TMR không loại bỏ lỗi — nó giảm xác suất lỗi ảnh hưởng đến đầu ra.

Nếu gọi:

* `Rm`: độ tin cậy của một module
* `Rv`: độ tin cậy của voter

thì độ tin cậy hệ thống:

```text
R_TMR = Rv × (3×Rm² - 2×Rm³)
```

Ý nghĩa trực giác:

* Hệ thống vẫn đúng nếu cả 3 module đúng
* Hoặc chỉ 1 module bị lỗi

Ví dụ:

```text
Rm = 0.99
→ RTMR ≈ 0.9997
```

TMR biến lỗi hiếm thành cực kỳ hiếm — nhưng không miễn nhiễm hoàn toàn.

---

### 6.4. Khi Nhiều Module Cùng Sai

TMR dựa trên một giả định quan trọng:

> Lỗi xảy ra độc lập giữa các module.

Nếu giả định này bị phá vỡ, TMR mất hiệu lực.

Ví dụ:

```text
A = 99
B = 99
C = 99
```

Nếu cả ba module cùng sai theo cùng một cách, voter sẽ chọn sai kết quả một cách “hợp lệ”.

Đây gọi là **Common-Mode Failure**.

Nguyên nhân có thể đến từ:

* lỗi thiết kế RTL
* bug trong compiler/toolchain
* nhiễu nguồn chung
* hoặc cùng một điều kiện môi trường ảnh hưởng đồng thời

---

### 6.5. Vấn Đề Thực Sự: Không Phải “Có 3 Bản Sao”

Một hiểu lầm phổ biến là:

> Nhân bản phần cứng = an toàn hơn

Nhưng nếu cả ba module được xây dựng từ cùng một thiết kế, chúng cũng có thể chia sẻ cùng một lỗi.

```text
Design Bug
   ↓
A = wrong
B = wrong
C = wrong
```

Đây là lý do một số hệ thống an toàn cao sử dụng thêm **design diversity**, tức là các implementation khác nhau để giảm xác suất lỗi chung.

---

### 6.6. Voter: Điểm Yếu Của TMR

TMR chỉ hiệu quả nếu voter đáng tin cậy.

Nếu voter sai:

```text
A = correct
B = correct
C = correct
Voter = faulty → wrong output
```

Toàn bộ hệ thống sụp đổ.

Vì vậy trong thực tế:

* voter thường được harden chống lỗi
* hoặc cũng được nhân bản
* hoặc được thiết kế đơn giản tối đa để giảm xác suất lỗi

---

### 6.7. Ý Nghĩa Cốt Lõi Của TMR

TMR không phải là một kỹ thuật sửa lỗi.

Nó là một kỹ thuật **ẩn lỗi bằng redundancy**.

* ECC: sửa lỗi trong dữ liệu
* Chipkill: chịu lỗi trong bộ nhớ
* TMR: chịu lỗi trong tính toán

Nhưng cả ba đều chia sẻ cùng một triết lý:

> Lỗi là điều chắc chắn xảy ra. Thiết kế tốt là thiết kế không để lỗi trở thành failure.

Tuy nhiên, TMR vẫn chỉ giải quyết bài toán “tiếp tục đúng khi lỗi xảy ra”.

Còn một hướng khác: thay vì che giấu lỗi, hệ thống có thể chạy song song và kiểm tra đối chiếu từng bước để phát hiện sai lệch ngay lập tức.

Đó chính là **Lockstep Execution**.

---

## 7. Lockstep Execution: Runtime Integrity Verification Trong Phần Cứng

Cho đến thời điểm này, chúng ta đã đi qua hai cách tiếp cận chính trong thiết kế hệ thống chịu lỗi:

* ECC và Chipkill: sửa lỗi trong bộ nhớ
* TMR: che giấu lỗi bằng cách lấy đa số

Cả hai đều dựa trên một giả định chung:

> Hệ thống có thể tiếp tục vận hành, miễn là lỗi được xử lý hoặc bị che giấu kịp thời.

Nhưng giả định này không phải lúc nào cũng đúng.

Trong các hệ thống an toàn cao (*safety-critical systems*), đôi khi vấn đề không phải là sửa lỗi hay tiếp tục chạy, mà là:

> Làm sao để biết hệ thống có còn đang hoạt động đúng hay không — càng sớm càng tốt.

Đó là nền tảng của **Lockstep Execution**.

---

### 7.1. Detection Thay Vì Correction

Khác với ECC (sửa dữ liệu) hay TMR (bỏ phiếu kết quả), Lockstep không cố gắng khôi phục trạng thái đúng.

Nó trả lời một câu hỏi đơn giản hơn:

> Hai bản sao của hệ thống có còn đồng nhất hay không?

Ý tưởng cơ bản là chạy hai CPU gần như giống hệt nhau:

```text
CPU A
CPU B
```

Cùng nhận input
Cùng thực thi instruction stream
Cùng sinh output

Sau đó hệ thống so sánh trạng thái nội bộ theo thời gian thực.

Nếu đồng nhất:

```text
CPU A == CPU B
```

Nếu có sai lệch:

```text
CPU A != CPU B
```

→ hệ thống coi đây là dấu hiệu của fault.

Điểm quan trọng: Lockstep không cần biết bên nào sai. Nó chỉ cần phát hiện **divergence**.

---

### 7.2. Dual-Core Lockstep

Cấu hình phổ biến nhất là **Dual-Core Lockstep (DCLS)**:

```text
        Input
          │
     ┌────┴────┐
     ▼         ▼
   CPU A     CPU B
     │         │
     └────┬────┘
          ▼
     Comparator
          │
          ▼
        Output
```

Trong điều kiện bình thường, hai core tiến hành từng chu kỳ giống hệt nhau:

```text
Cycle 100:
A: ADD R1, R2
B: ADD R1, R2

Cycle 101:
A: MOV R3, R4
B: MOV R3, R4
```

Nếu một SEU xảy ra và làm sai lệch trạng thái:

```text
CPU A: R5 = 0x1000
CPU B: R5 = 0x1008
```

Comparator phát hiện mismatch ngay lập tức.

---

### 7.3. Delayed Lockstep

Trong thực tế, nhiều hệ thống sử dụng biến thể **Delayed Lockstep** thay vì đồng bộ tuyệt đối.

```text
CPU A  → executes first
CPU B  → executes after N cycles
```

Cách này tạo ra một độ lệch thời gian có kiểm soát giữa hai execution stream.

Lợi ích chính:

> Giảm xác suất cả hai CPU bị ảnh hưởng bởi cùng một sự kiện vật lý tại cùng thời điểm.

Ví dụ:

```text
Radiation event occurs
      │
      ▼
CPU A affected
CPU B not yet at same execution point
```

Điều này làm tăng khả năng phát hiện lỗi do SEU hoặc transient fault.

---

### 7.4. Comparator: Điểm Quan Trọng Nhất

Trái tim của Lockstep là **Comparator**, không phải CPU.

Comparator theo dõi và so sánh trạng thái hệ thống, bao gồm:

* Program Counter (PC)
* Register file
* Memory transaction
* Control signals
* Interrupt state

Ví dụ:

```text
A: PC = 0x80401000
B: PC = 0x80401000  → OK

A: PC = 0x80401000
B: PC = 0x80401004  → MISMATCH
```

Khi mismatch xảy ra, comparator sinh ra fault signal.

Quan trọng hơn: quá trình này diễn ra ở **hardware level**, độc lập với software.

---

### 7.5. Bài Toán Đồng Bộ Hóa

Một yêu cầu cốt lõi của Lockstep là:

> Hai CPU phải thực thi cùng một computation space.

Điều này đòi hỏi:

* deterministic execution
* synchronized clocking
* identical input streams
* controlled interrupt delivery

Nếu không đảm bảo các điều kiện này, divergence sẽ xảy ra liên tục và hệ thống trở nên vô nghĩa.

---

### 7.6. Fault Detection Trong Thời Gian Thực

Điểm khác biệt lớn nhất của Lockstep so với ECC và TMR nằm ở thời điểm phát hiện lỗi:

* ECC: phát hiện khi đọc dữ liệu
* TMR: phát hiện khi voting
* Lockstep: phát hiện **trong khi CPU đang chạy**

```text
Fault → Divergence → Immediate Detection → Alarm
```

Độ trễ phát hiện thường chỉ vài clock cycles.

Điều này cực kỳ quan trọng trong các hệ thống:

* Automotive (ISO 26262)
* Aerospace
* Industrial control
* Safety-critical embedded systems

nơi việc biết “có lỗi xảy ra” quan trọng hơn việc “cố gắng tiếp tục chạy”.

---

### 7.7. Chiến Lược Xử Lý Sau Khi Phát Hiện Lỗi

Lockstep không định nghĩa cách sửa lỗi — nó chỉ phát hiện.

Sau khi divergence được phát hiện, hệ thống có thể:

**Reset system**

```text
Fault → Reset → Restart
```

**Switch to backup core**

```text
Fault → Activate spare processor
```

**Enter safe state**

```text
Fault → Controlled shutdown → Safe state
```

---

### 7.8. Safe State Transition

Trong hệ thống safety-critical, mục tiêu không phải là “luôn chạy đúng”, mà là:

> Không bao giờ tạo ra hành vi nguy hiểm khi lỗi xảy ra.

```text
Fault
 ↓
Detected
 ↓
Controlled response
 ↓
Safe state
```

Điều này làm Lockstep khác với TMR:

* TMR: che giấu lỗi để tiếp tục vận hành
* Lockstep: phát hiện lỗi để dừng hoặc chuyển trạng thái an toàn

---

### 7.9. Tổng Kết Vai Trò

Lockstep không thay thế ECC hay TMR.

Nó nằm ở một lớp khác:

* ECC: bảo vệ dữ liệu
* TMR: bảo vệ kết quả
* Lockstep: bảo vệ **tính đúng đắn của toàn bộ execution flow**

Sau khi đi qua ECC, Chipkill, TMR và Lockstep, có thể thấy mỗi cơ chế không cạnh tranh trực tiếp với nhau mà giải quyết các tầng lỗi khác nhau trong hệ thống.

Câu hỏi cuối cùng là:

> Khi đặt tất cả các cơ chế này cạnh nhau, chúng khác nhau như thế nào về phạm vi bảo vệ, chi phí và mức độ tin cậy?

Đó sẽ là nội dung của phần so sánh tổng kết.

---

## 8. Architectural Trade-Off Analysis

Sau khi phân tích ECC, Chipkill, Triple Modular Redundancy (TMR) và Lockstep Execution, có thể nhận thấy rằng các cơ chế này không cạnh tranh trực tiếp với nhau. Chúng được thiết kế để giải quyết những loại lỗi khác nhau, xuất hiện ở những thành phần khác nhau của hệ thống.

Trong kiến trúc chịu lỗi, câu hỏi quan trọng không phải là cơ chế nào mạnh nhất, mà là cơ chế nào cung cấp mức độ bảo vệ phù hợp nhất với chi phí tài nguyên chấp nhận được.

Mỗi lớp bảo vệ đều tiêu tốn một phần tài nguyên của hệ thống dưới dạng:

* Diện tích silicon (Area).
* Công suất tiêu thụ (Power).
* Độ trễ xử lý (Latency).
* Độ phức tạp thiết kế (Design Complexity).
* Chi phí phần cứng (Hardware Cost).

Do đó, thiết kế Fault-Tolerant Architecture luôn là bài toán cân bằng giữa Reliability và Overhead.

---

### 8.1. Evaluation Criteria

Để so sánh các cơ chế chịu lỗi, cần xác định các tiêu chí đánh giá chung.

#### Fault Coverage

Fault Coverage thể hiện khả năng phát hiện hoặc xử lý lỗi của hệ thống.

Coverage càng cao thì xác suất một lỗi không được phát hiện càng thấp.

Tuy nhiên việc tăng Coverage thường yêu cầu nhiều phần cứng và logic kiểm tra hơn.

---

#### Reliability Improvement

Reliability mô tả xác suất hệ thống hoạt động chính xác trong một khoảng thời gian xác định.

Một cơ chế chịu lỗi tốt không chỉ phát hiện lỗi mà còn phải làm giảm xác suất Failure của toàn hệ thống.

Ví dụ đối với Triple Modular Redundancy:

```text
R_TMR = 3R² - 2R³
```

Trong đó:

* `R` là reliability của một module đơn lẻ.
* `R_TMR` là reliability của hệ thống TMR.

Kết quả cho thấy TMR có thể cải thiện đáng kể độ tin cậy khi reliability ban đầu của module đủ cao.

---

#### Area Overhead

Area Overhead phản ánh lượng phần cứng bổ sung cần thiết.

Ví dụ:

* ECC yêu cầu thêm các bit parity.
* Chipkill yêu cầu thêm DRAM chip và logic giải mã.
* Lockstep yêu cầu thêm lõi xử lý thứ hai.
* TMR yêu cầu nhân bản toàn bộ khối logic ba lần.

Trong các hệ thống nhúng hoặc thiết bị giới hạn tài nguyên, đây thường là yếu tố quyết định.

---

#### Power Overhead

Mỗi transistor bổ sung đều làm tăng công suất tiêu thụ.

Đặc biệt đối với:

* TMR.
* Lockstep.

nhiều khối phần cứng phải hoạt động đồng thời trong toàn bộ thời gian vận hành.

Điều này tạo ra chi phí năng lượng đáng kể.

---

#### Performance Impact

Một số cơ chế chịu lỗi có thể làm tăng độ trễ xử lý.

Ví dụ:

* ECC yêu cầu quá trình encode/decode.
* Chipkill yêu cầu thuật toán sửa lỗi phức tạp hơn.
* Lockstep cần đồng bộ trạng thái giữa các lõi xử lý.

Do đó việc tăng độ tin cậy đôi khi phải đánh đổi bằng hiệu năng.

---

### 8.2. ECC và Chipkill

ECC là lớp bảo vệ cơ bản nhất đối với bộ nhớ.

Mục tiêu của ECC là xử lý các lỗi bit đơn lẻ xuất hiện trong DRAM hoặc SRAM.

Đối với phần lớn các Soft Error thông thường, cơ chế SECDED (Single Error Correction Double Error Detection) là đủ để đảm bảo tính toàn vẹn dữ liệu.

Ưu điểm của ECC:

* Area overhead thấp.
* Power overhead thấp.
* Latency nhỏ.
* Triển khai đơn giản.

Tuy nhiên ECC có giới hạn rõ ràng.

Nó được thiết kế cho fault model dạng:

```text
Single-Bit Error
```

Khi nhiều bit bị lỗi đồng thời hoặc một DRAM chip gặp sự cố hoàn toàn, ECC truyền thống có thể không còn khả năng phục hồi dữ liệu.

Đó là lý do Chipkill được phát triển.

Thay vì chỉ bảo vệ từng bit riêng lẻ, Chipkill phân phối dữ liệu và mã sửa lỗi trên nhiều DRAM chip khác nhau.

Kết quả là hệ thống vẫn có thể khôi phục dữ liệu ngay cả khi một DRAM chip bị hỏng hoàn toàn.

Đổi lại:

* Nhiều bit dự phòng hơn.
* Logic sửa lỗi phức tạp hơn.
* Chi phí phần cứng cao hơn.

Chipkill vì vậy thường xuất hiện trong:

* Enterprise Server.
* Datacenter.
* High Availability Computing.

thay vì các hệ thống phổ thông.

---

### 8.3. TMR và Lockstep

Khác với ECC và Chipkill, TMR và Lockstep không tập trung vào lỗi bộ nhớ.

Chúng được thiết kế để xử lý lỗi xảy ra bên trong logic xử lý hoặc lõi CPU.

---

#### Triple Modular Redundancy

TMR sử dụng ba bản sao giống hệt nhau của cùng một module.

Kết quả đầu ra được xác định thông qua majority voting.

Mạch voter có thể được biểu diễn bằng:

```text
Y = AB + AC + BC
```

Nếu một module bị lỗi tạm thời do Single Event Upset, hai module còn lại vẫn có thể tạo ra kết quả đúng.

Ưu điểm của TMR:

* Fault masking trực tiếp.
* Không làm gián đoạn hoạt động hệ thống.
* Khả năng chịu lỗi cao.

Tuy nhiên chi phí rất lớn.

Một hệ thống TMR yêu cầu:

```text
3 × Logic Module
+
1 × Majority Voter
```

Điều này làm tăng đáng kể:

* Silicon area.
* Power consumption.
* Manufacturing cost.

TMR thường chỉ xuất hiện trong:

* Spacecraft Electronics.
* Satellite Computing.
* Nuclear Control Systems.
* Mission-Critical Hardware.

---

#### Lockstep Execution

Lockstep sử dụng hai hoặc nhiều lõi xử lý thực thi cùng một chương trình theo cùng trình tự.

Tại mỗi chu kỳ, trạng thái thực thi được so sánh thông qua comparator.

Nếu phát hiện sai lệch:

```text
Fault
  ↓
Detect
  ↓
Exception
  ↓
Safe State
```

hệ thống sẽ kích hoạt cơ chế bảo vệ.

Khác với TMR, Lockstep không cố gắng che giấu lỗi.

Nó tập trung vào việc phát hiện lỗi càng sớm càng tốt.

Ưu điểm:

* Phát hiện lỗi nhanh.
* Coverage cao đối với CPU fault.
* Chi phí thấp hơn TMR.

Nhược điểm:

* Không tự sửa lỗi.
* Cần cơ chế recovery bổ sung.

Đây là lý do Lockstep được sử dụng rộng rãi trong các hệ thống Safety-Critical như ô tô, hàng không và điều khiển công nghiệp.

---

### 8.4. Comparative Analysis

| Mechanism | Target Fault Model            | Detection | Recovery          | Area Overhead | Power Overhead | Performance Impact |
| --------- | ----------------------------- | --------- | ----------------- | ------------- | -------------- | ------------------ |
| ECC       | Single-bit memory error       | High      | High              | Low           | Low            | Low                |
| Chipkill  | Multi-bit error, chip failure | High      | High              | Medium        | Medium         | Medium             |
| TMR       | Logic transient fault         | High      | High (Masking)    | Very High     | Very High      | Low                |
| Lockstep  | CPU execution fault           | Very High | External Recovery | High          | High           | Low                |

Bảng trên cho thấy không tồn tại một cơ chế vượt trội trong mọi tình huống.

Mỗi giải pháp được tối ưu cho một fault model cụ thể.

Do đó việc lựa chọn phụ thuộc trực tiếp vào yêu cầu của hệ thống.

---

### 8.5. Layered Fault-Tolerant Design

Trong các hệ thống hiện đại, các cơ chế chịu lỗi thường được triển khai đồng thời thay vì thay thế lẫn nhau.

Ví dụ:

```text
ECC Memory
     +
Chipkill
     +
Lockstep CPU
```

hoặc:

```text
ECC Memory
     +
TMR Control Logic
```

Cách tiếp cận nhiều lớp giúp mở rộng phạm vi bảo vệ từ bộ nhớ đến logic xử lý.

Một lỗi đơn lẻ xuất hiện ở bất kỳ tầng nào của hệ thống đều phải vượt qua nhiều cơ chế kiểm tra trước khi có thể phát triển thành Failure.

Vì vậy, khả năng chịu lỗi của hệ thống hiện đại không đến từ một kỹ thuật đơn lẻ mà đến từ sự kết hợp của nhiều lớp bảo vệ với các mức chi phí và hiệu quả khác nhau. Mục tiêu cuối cùng của Fault-Tolerant Architecture không phải là loại bỏ hoàn toàn lỗi, mà là đảm bảo rằng lỗi không thể dễ dàng chuyển hóa thành sự cố hệ thống.

---

## 9. When Reliability Becomes Security: Từ SEU Đến Fault Injection

Trong suốt bài viết đến thời điểm này, chúng ta đã xem xét lỗi phần cứng dưới góc nhìn của Reliability Engineering.

Các hiện tượng như:

* Single Event Upset (SEU)
* Multi-Bit Upset (MBU)
* DRAM Failure
* Logic Transient Fault

được xem là những sự kiện ngẫu nhiên phát sinh từ môi trường vật lý.

Một hạt neutron từ tia vũ trụ có thể làm thay đổi trạng thái của một transistor.

Một điện tích tích lũy bất thường có thể làm lật một bit trong ô nhớ.

Một lỗi tạm thời có thể xuất hiện trong logic xử lý của CPU.

Từ góc nhìn của kỹ sư hệ thống, đây là các **Natural Faults** cần được phát hiện hoặc khắc phục bằng ECC, Chipkill, TMR hoặc Lockstep.

Tuy nhiên, từ góc nhìn của một attacker, những hiện tượng này lại dẫn tới một câu hỏi thú vị hơn:

> Nếu một bit flip có thể xảy ra ngẫu nhiên, liệu chúng ta có thể chủ động tạo ra nó hay không?

Câu trả lời là có.

Ý tưởng đó chính là nền tảng của lĩnh vực **Fault Injection Attack**.

---

### 9.1. Reliability Assumptions và Security Assumptions

Một điểm thú vị trong Hardware Security là phần lớn các cơ chế chịu lỗi đều được xây dựng dựa trên một số giả định nhất định.

Ví dụ:

ECC giả định rằng lỗi xuất hiện ngẫu nhiên và thường chỉ ảnh hưởng tới một số lượng nhỏ bit dữ liệu.

```text
Random Fault
      ↓
Single-Bit Error
```

TMR giả định rằng các module hoạt động độc lập và lỗi chỉ xuất hiện trên một module tại một thời điểm.

```text
Independent Faults
```

Lockstep giả định rằng lỗi xuất hiện trên một lõi xử lý riêng lẻ và sẽ tạo ra sai lệch trạng thái thực thi.

```text
CPU A ≠ CPU B
```

Trong khi đó, attacker lại cố tình phá vỡ các giả định này.

Thay vì tạo ra lỗi ngẫu nhiên, attacker cố gắng tạo ra lỗi đúng vị trí, đúng thời điểm và đúng mục tiêu.

```text
ECC
    ↓
Multi-Bit Fault

TMR
    ↓
Common-Mode Fault

Lockstep
    ↓
Synchronized Fault
```

Từ góc nhìn này, nhiều cuộc tấn công phần cứng thực chất là quá trình khai thác những giả định nền tảng của Reliability Engineering.

---

### 9.2. Fault Injection: Chủ Động Tạo Lỗi Trong Phần Cứng

Fault Injection là nhóm kỹ thuật chủ động tạo ra lỗi vật lý bên trong hệ thống nhằm làm thay đổi trạng thái thực thi của phần cứng.

Mục tiêu không phải phá hủy chip.

Mục tiêu là tạo ra một lỗi tạm thời đủ nhỏ để hệ thống vẫn tiếp tục hoạt động nhưng tạo ra hành vi ngoài dự kiến.

Mô hình tổng quát của một Fault Injection Attack có thể được mô tả như sau:

```text
Injected Fault
        ↓
Corrupted State
        ↓
Unexpected Execution
        ↓
Security Impact
```

Từ góc nhìn của CPU, một fault do attacker tạo ra đôi khi không khác biệt đáng kể so với một SEU tự nhiên.

Điều này khiến ranh giới giữa Reliability và Security trở nên rất mờ nhạt.

---

### 9.3. Common Fault Injection Techniques

#### Voltage Glitching

Voltage Glitching là kỹ thuật làm gián đoạn nguồn cấp trong một khoảng thời gian rất ngắn.

```text
Nominal Voltage
       ↓
Voltage Drop
       ↓
Timing Violation
       ↓
Fault
```

Khi điện áp giảm đột ngột, một số transistor có thể không hoàn thành quá trình chuyển trạng thái trước khi dữ liệu được chốt.

Kết quả là hệ thống có thể xuất hiện:

* Sai lệch dữ liệu trong thanh ghi.
* Sai lệch tín hiệu điều khiển.
* Sai lệch luồng thực thi.

---

#### Clock Glitching

Clock Glitching tác động trực tiếp vào tín hiệu đồng hồ của bộ xử lý.

```text
Normal Clock
        ↓
Abnormal Pulse
        ↓
Pipeline Timing Error
```

Do CPU phụ thuộc hoàn toàn vào clock để đồng bộ hoạt động giữa các pipeline stage, việc chèn thêm hoặc rút ngắn chu kỳ clock có thể tạo ra lỗi thực thi tạm thời.

---

#### Electromagnetic Fault Injection (EMFI)

EMFI sử dụng các xung điện từ năng lượng cao để gây nhiễu lên vùng silicon mục tiêu.

```text
EM Pulse
      ↓
Transient Current
      ↓
Logic Fault
```

Ưu điểm của EMFI là không yêu cầu tiếp xúc trực tiếp với vi mạch và có thể tạo lỗi với độ chính xác tương đối cao.

Hiện nay đây là một trong những kỹ thuật được sử dụng rộng rãi trong nghiên cứu Hardware Security.

---

#### Laser Fault Injection

Laser Fault Injection sử dụng tia laser để tạo điện tích trực tiếp bên trong silicon.

```text
Laser
   ↓
Charge Injection
   ↓
Bit Flip
```

Sau khi lớp vỏ bảo vệ của chip bị loại bỏ, attacker có thể nhắm tới những vùng logic hoặc thanh ghi cụ thể.

Kỹ thuật này cho phép nghiên cứu chính xác tác động của lỗi trên từng thành phần vi kiến trúc.

---

### 9.4. Fault Injection và Kiến Trúc Bộ Xử Lý

Ở cấp độ vi kiến trúc, Fault Injection không chỉ làm thay đổi dữ liệu mà còn có thể tác động tới trạng thái thực thi của CPU.

Một số mục tiêu phổ biến bao gồm:

* Program Counter (PC)
* General Purpose Registers
* Condition Flags
* Pipeline Control Logic
* Branch Prediction State

Một lỗi xuất hiện đúng thời điểm có thể làm thay đổi luồng điều khiển của chương trình.

```text
Instruction N
Instruction N+1
Instruction N+2
```

Nếu Instruction N+1 bị bỏ qua:

```text
Instruction N
Instruction N+2
```

hệ thống có thể thực thi theo một nhánh hoàn toàn khác với thiết kế ban đầu.

Hiện tượng này thường được gọi là Instruction Skip Attack.

---

### 9.5. Differential Fault Analysis

Một trong những ứng dụng nguy hiểm nhất của Fault Injection xuất hiện trong lĩnh vực mật mã học.

Nhiều thuật toán mã hóa giả định rằng toàn bộ quá trình tính toán diễn ra chính xác.

Nếu attacker tạo ra một lỗi có kiểm soát trong quá trình mã hóa hoặc giải mã, kết quả sai lệch có thể tiết lộ thông tin về khóa bí mật.

Mô hình tổng quát:

```text
Correct Output
        +
Faulty Output
        ↓
Differential Analysis
        ↓
Secret Key Recovery
```

Kỹ thuật này được gọi là Differential Fault Analysis (DFA).

Nhiều nghiên cứu đã chứng minh khả năng trích xuất khóa AES hoặc RSA chỉ từ một số lượng nhỏ kết quả bị lỗi.

Điều này cho thấy cùng một hiện tượng vật lý có thể được xem là vấn đề Reliability hoặc Security tùy theo bối cảnh sử dụng.

---

### 9.6. Rowhammer: Khi Reliability Trở Thành Vulnerability

Một trong những ví dụ nổi tiếng nhất về sự giao thoa giữa Reliability và Security là Rowhammer.

Ban đầu, Rowhammer được xem là một hiện tượng độ tin cậy của DRAM.

Việc kích hoạt liên tục một hàng nhớ có thể tạo ra nhiễu điện tích trên các hàng lân cận.

```text
Repeated Row Activation
          ↓
Charge Leakage
          ↓
Bit Flip
```

Tuy nhiên, các nghiên cứu sau đó chỉ ra rằng hiện tượng này có thể bị khai thác để thay đổi dữ liệu nằm ngoài vùng nhớ được cấp quyền.

```text
Bit Flip
     ↓
Memory Corruption
     ↓
Privilege Escalation
```

Rowhammer là minh chứng rõ ràng cho việc một vấn đề Reliability hoàn toàn có thể phát triển thành một lỗ hổng Security nghiêm trọng.

---

### 9.7. Fault-Tolerant Architectures Under Adversarial Faults

Một câu hỏi quan trọng là liệu các cơ chế chịu lỗi đã trình bày trước đó có giúp chống lại Fault Injection hay không.

Câu trả lời là có, nhưng với những giới hạn nhất định.

ECC hoạt động hiệu quả trước các lỗi bộ nhớ ngẫu nhiên nhưng có thể gặp khó khăn khi attacker tạo ra nhiều lỗi đồng thời hoặc nhắm vào logic xử lý.

TMR có khả năng che giấu lỗi đơn lẻ thông qua majority voting.

Tuy nhiên nếu nhiều module cùng bị ảnh hưởng bởi một Common-Mode Fault, giả định độc lập của TMR sẽ không còn đúng.

Lockstep thường có khả năng phát hiện Fault Injection tốt hơn vì nó liên tục so sánh trạng thái thực thi giữa các lõi xử lý.

Tuy nhiên Lockstep cũng có thể thất bại nếu cùng một lỗi xuất hiện đồng thời trên tất cả các lõi đang được giám sát.

Điều này cho thấy không tồn tại cơ chế chịu lỗi tuyệt đối. Mọi kiến trúc đều được xây dựng dựa trên một tập giả định nhất định về mô hình lỗi.

---

### 9.8. Reliability and Security Are Two Sides of the Same Problem

Từ góc nhìn truyền thống, Reliability Engineering và Security Engineering thường được xem là hai lĩnh vực độc lập.

Tuy nhiên Fault Injection cho thấy ranh giới giữa chúng thực chất rất mong manh.

Một bit flip do tia vũ trụ tạo ra và một bit flip do attacker tạo ra có thể dẫn tới cùng một hậu quả ở cấp độ vi kiến trúc.

Sự khác biệt duy nhất nằm ở chủ thể tạo ra lỗi.

Điều này lý giải vì sao các cơ chế như ECC, Chipkill, TMR và Lockstep không chỉ quan trọng đối với độ tin cậy của hệ thống mà còn đóng vai trò ngày càng lớn trong việc bảo vệ phần cứng trước các cuộc tấn công chủ động.

Theo một nghĩa nào đó, Hardware Security hiện đại chính là sự mở rộng tự nhiên của Reliability Engineering trong môi trường có đối thủ chủ động.

---

## 10. Phân tích và khai thác tấn công Differential Fault Analysis (DFA) trên AES-128

> **Môi trường:** Kali Linux (VM) · gem5 v25.1.0.1 · RISC-V 64-bit · tiny-AES-c · PhoenixAES 0.0.5  
> **Mục tiêu:** Chứng minh DFA attack recover được AES-128 master key từ faulty ciphertext, và Lockstep Defense chặn hoàn toàn attack đó.  
> **Thời gian thực hiện:** ~1 ngày (bao gồm build time và debug)

---

### 10.1. Lý thuyết nền tảng

#### 1.1. AES-128 hoạt động như thế nào

AES-128 encrypt một block 16 bytes qua **10 rounds**, mỗi round gồm 4 bước:

```
Round 0:   AddRoundKey (initial whitening với master key)
Round 1-9: SubBytes → ShiftRows → MixColumns → AddRoundKey
Round 10:  SubBytes → ShiftRows → AddRoundKey (không có MixColumns)
```

State là một matrix 4×4 bytes (16 bytes tổng). Master key 128 bits được expand thành 11 round keys (176 bytes tổng) qua **key schedule**.

Điểm quan trọng cho DFA: nếu biết được **round 10 key** (16 bytes cuối của key schedule), có thể reverse key schedule để ra master key. Round 10 key và master key có quan hệ toán học xác định — không cần brute force.

#### 1.2. DFA là gì và tại sao nó hoạt động

Differential Fault Analysis (Boneh et al., 1997) khai thác một điểm yếu vật lý: nếu inject một lỗi nhỏ (bit flip) vào AES state tại **round 9**, output sẽ bị corrupt theo một pattern có thể phân tích.

Cụ thể: khi flip 1 byte trong state trước SubBytes round 9, error propagate qua MixColumns round 9 → toàn bộ column bị ảnh hưởng → sau AddRoundKey round 9 → tiếp tục vào round 10. Bằng cách so sánh **correct ciphertext** và **faulty ciphertext**, có thể suy ngược lại giá trị của round 10 key tại các vị trí bị ảnh hưởng.

Với đủ faulty ciphertexts (lý thuyết cần ≥ 2–4 cặp mỗi byte của round 10 key), PhoenixAES recover toàn bộ round 10 key.

#### 1.3. Lockstep Defense

Lockstep là countermeasure phần cứng: chạy **hai CPU song song** với cùng input, sau mỗi bước so sánh output. Nếu có fault inject vào một CPU, hai output sẽ diverge và hệ thống abort trước khi faulty ciphertext được output ra ngoài.

DFA không thể hoạt động nếu attacker không thu được faulty ciphertext.

---

### 10.2. Thiết kế experiment

#### 2.1. Stack công cụ

| Component | Lý do chọn |
|-----------|-----------|
| **gem5** | Full-system CPU simulator, chạy RISC-V binary với timing chính xác. Standard trong architecture research. |
| **RISC-V** | ISA đơn giản, disassembly dễ đọc, gem5 support tốt. |
| **tiny-AES-c** | Implementation AES-128 thuần C, ~500 LOC, không có optimization phức tạp. Dễ phân tích và patch. |
| **PhoenixAES** | Tool DFA chuẩn từ SideChannelMarvels, được cite trong nhiều paper. |
| **SE mode** | Syscall Emulation — chạy user-space binary mà không cần full OS. Đủ cho mục đích này. |

#### 2.2. Threat model

- **Attacker**: Có khả năng inject fault vật lý vào chip đang chạy (voltage glitch, EM pulse, laser)
- **Target**: AES-128 ECB encryption với secret key
- **Goal**: Recover master key từ (correct ciphertext, faulty ciphertexts)
- **Assumption**: Attacker biết plaintext (hoặc có oracle), biết vị trí rough của round 9 trong execution timeline

#### 2.3. Tại sao chọn round 9

Round 9 là round **áp chót**. Fault inject vào round 9 SubBytes có đặc tính:
- Chỉ ảnh hưởng đến 1 column của state sau MixColumns
- Pattern propagation có thể tính toán ngược
- Round 10 chỉ có SubBytes + ShiftRows + AddRoundKey — không có MixColumns làm phức tạp thêm

Inject vào round 8 hoặc trước đó: error lan rộng hơn, harder to analyze. Inject vào round 10: quá muộn, không đủ data để recover key.

---

### 10.3. Setup môi trường

#### 3.1. Hệ điều hành

Experiment chạy trên **Kali Linux** (Debian-based). Không thể chạy trên Windows native vì gem5 depend vào Linux-specific build system (SCons + Python bindings + C++ với POSIX APIs).

Nếu chỉ có Windows: dùng WSL2 (Ubuntu kernel thật chạy bên trong Windows, không phải emulation). Kali VM cũng ổn.

#### 3.2. Install dependencies

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
    build-essential git m4 scons zlib1g zlib1g-dev \
    libprotobuf-dev protobuf-compiler libprotoc-dev \
    libgoogle-perftools-dev python3-dev python3-pip \
    libboost-all-dev pkg-config \
    gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

Verify RISC-V toolchain:

```bash
riscv64-linux-gnu-gcc --version
# riscv64-linux-gnu-gcc (Debian 14.2.0-7) 14.2.0
```

---

### 10.4. Build gem5

#### 4.1. Clone và build

```bash
git clone https://github.com/gem5/gem5
cd gem5
scons build/RISCV/gem5.opt -j$(nproc)
```

`-j$(nproc)` dùng toàn bộ CPU cores. Build mất 45–60 phút.

#### 4.2. Output và warnings

```
scons: done building targets.
*** Summary of Warnings ***
Warning: Detected GCC version 15.2.0 is not officially supported.
         gem5 supports GCC v11 up to v14.2.
Warning: Header file <png.h> not found.
Warning: Couldn't find HDF5 C++ libraries. Disabling HDF5 support.
```

**Phân tích warnings:**
- GCC 15.2 không officially supported nhưng vẫn compile được. Không ảnh hưởng đến RISC-V simulation.
- libpng và HDF5 chỉ cần cho framebuffer rendering và trace capture — không cần cho experiment này.

#### 4.3. Verify

```bash
./build/RISCV/gem5.opt
# Output: Usage / gem5 options
# gem5 không có --version flag, output Usage là đúng
```

---

### 10.5. Compile AES target

#### 5.1. Clone tiny-AES-c

```bash
git clone https://github.com/kokke/tiny-AES-c
cd tiny-AES-c
```

#### 5.2. Tạo wrapper với NIST test vector

Dùng NIST FIPS 197 Appendix B test vector để có ground truth rõ ràng:
- Key: `2B7E151628AED2A6ABF7158809CF4F3C`
- Plaintext: `6BC1BEE22E409F96E93D7E117393172A`
- Expected ciphertext: `3AD77BB40D7A3660A89ECAF32466EF97`

```c
// aes_main.c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include "aes.h"

int fault_byte = -1;
int fault_bit  = 0;

int main() {
    uint8_t key[16] = {
        0x2b, 0x7e, 0x15, 0x16, 0x28, 0xae, 0xd2, 0xa6,
        0xab, 0xf7, 0x15, 0x88, 0x09, 0xcf, 0x4f, 0x3c
    };
    uint8_t plaintext[16] = {
        0x6b, 0xc1, 0xbe, 0xe2, 0x2e, 0x40, 0x9f, 0x96,
        0xe9, 0x3d, 0x7e, 0x11, 0x73, 0x93, 0x17, 0x2a
    };

    char *fb = getenv("FAULT_BYTE");
    char *fi = getenv("FAULT_BIT");
    if (fb) fault_byte = atoi(fb);
    if (fi) fault_bit  = atoi(fi);

    struct AES_ctx ctx;
    AES_init_ctx(&ctx, key);

    uint8_t buf[16];
    memcpy(buf, plaintext, 16);
    AES_ECB_encrypt(&ctx, buf);

    printf("Ciphertext: ");
    for (int i = 0; i < 16; i++) printf("%02x", buf[i]);
    printf("\n");
    return 0;
}
```

#### 5.3. Compile cho RISC-V

```bash
riscv64-linux-gnu-gcc -O0 -g -static -o aes_test aes_main.c aes.c -I.
```

**Tại sao cần `-static`?**

Lần đầu compile không có `-static`:

```bash
riscv64-linux-gnu-gcc -O0 -g -o aes_test aes_main.c aes.c -I.
file aes_test
# aes_test: ELF 64-bit LSB pie executable, UCB RISC-V, dynamically linked,
#           interpreter /lib/ld-linux-riscv64-lp64d.so.1
```

Chạy trong gem5 báo lỗi:

```
fatal: Failed to open file /lib/ld-linux-riscv64-lp64d.so.1.
```

**Suy luận:** gem5 SE mode (Syscall Emulation) không có filesystem thật — nó emulate syscalls nhưng không mount real Linux filesystem. Dynamic linker cần tìm shared libraries trong `/lib/` nhưng path đó không tồn tại trong SE mode environment.

**Fix:** Compile static để toàn bộ code nằm trong binary, không cần external libs lúc runtime.

```bash
riscv64-linux-gnu-gcc -O0 -g -static -o aes_test aes_main.c aes.c -I.
file aes_test
# aes_test: ELF 64-bit LSB executable, UCB RISC-V, statically linked ✓
```

**Tại sao `-O0`?**

Tắt compiler optimization để:
- Execution trace predictable (instructions không bị reorder)
- Function calls không bị inlined
- AES round loop giữ nguyên structure dễ phân tích

---

### 10.6. Chạy AES trong gem5

#### 6.1. Config script

```python
# run_aes.py
import m5
from m5.objects import *

system = System()
system.clk_domain = SrcClockDomain()
system.clk_domain.clock = '1GHz'
system.clk_domain.voltage_domain = VoltageDomain()
system.mem_mode = 'timing'
system.mem_ranges = [AddrRange('512MB')]

system.cpu = RiscvTimingSimpleCPU()
system.membus = SystemXBar()
system.cpu.icache_port = system.membus.cpu_side_ports
system.cpu.dcache_port = system.membus.cpu_side_ports
system.cpu.createInterruptController()

system.mem_ctrl = MemCtrl()
system.mem_ctrl.dram = DDR3_1600_8x8()
system.mem_ctrl.dram.range = system.mem_ranges[0]
system.mem_ctrl.port = system.membus.mem_side_ports
system.system_port = system.membus.cpu_side_ports

binary = '/home/clap/Desktop/tiny-AES-c/aes_test'
system.workload = SEWorkload.init_compatible(binary)
process = Process()
process.cmd = [binary]
system.cpu.workload = process
system.cpu.createThreads()

root = Root(full_system=False, system=system)
m5.instantiate()
exit_event = m5.simulate()
```

#### 6.2. Lỗi path đầu tiên

```bash
./build/RISCV/gem5.opt ~/Desktop/run_aes.py
```

```
fatal: Failed to open file /root/Desktop/tiny-AES-c/aes_test.
```

**Suy luận:** Script hardcode path `/root/Desktop/...` nhưng user đang chạy là `clap`, không phải `root`. Home directory là `/home/clap/`, không phải `/root/`.

**Fix:**

```bash
sed -i "s|/root/Desktop|/home/clap/Desktop|g" ~/Desktop/run_aes.py
```

#### 6.3. Verify output

```bash
./build/RISCV/gem5.opt ~/Desktop/run_aes.py
```

```
=== Starting AES simulation ===
Ciphertext: 3ad77bb40d7a3660a89ecaf32466ef97
Exited at tick 12184470000
```

Khớp với NIST test vector. AES chạy đúng trong gem5 RISC-V. ✓

---

### 10.7. Phân tích disassembly — tìm fault window

#### 7.1. Disassemble binary

```bash
riscv64-linux-gnu-objdump -d aes_test > aes_disasm.txt
grep -n "AES_ECB_encrypt\|SubBytes\|ShiftRows\|MixColumns\|AddRoundKey" aes_disasm.txt
```

Output quan trọng:

```
273:  104d2: jal  118b6 <AES_ECB_encrypt>
643:  0000000000010910 <AddRoundKey>:
715:  00000000000109e2 <SubBytes>:
769:  0000000000010a80 <ShiftRows>:
870:  0000000000010bd8 <MixColumns>:
1971: 00000000000118b6 <AES_ECB_encrypt>:
```

#### 7.2. Phân tích AES_ECB_encrypt

```asm
118b6 <AES_ECB_encrypt>:
  118d0: jal  117bc <Cipher>   ← chỉ gọi Cipher, không có logic khác
  118dc: ret
```

`AES_ECB_encrypt` chỉ là wrapper gọi `Cipher`. Logic thực sự nằm trong `Cipher`.

#### 7.3. Phân tích Cipher — tìm round loop

```bash
grep -n "" aes_disasm.txt | sed -n '1888,1930p'
```

```asm
117bc <Cipher>:
  117bc: addi sp, sp, -48      ← stack frame setup
  117be: sd   ra, 40(sp)
  117c0: sd   s0, 32(sp)
  117c4: sd   a0, -40(s0)      ← lưu state pointer
  117c8: sd   a1, -48(s0)      ← lưu RoundKey pointer
  117cc: sb   zero, -17(s0)    ← round counter = 0

  117da: jal  10910 <AddRoundKey>   ← Round 0 (initial whitening)
  117de: li   a5, 1
  117e0: sb   a5, -17(s0)          ← round counter = 1

  --- LOOP START ---
  117e8: jal  109e2 <SubBytes>      ← ← ← TARGET: gọi 9 lần
  117f0: jal  10a80 <ShiftRows>
  117fe: beq  a4, a5, 11828         ← if round == 10: jump to final
  11806: jal  10bd8 <MixColumns>
  11818: jal  10910 <AddRoundKey>
  11820: addiw a5, a5, 1            ← round++
  11826: j    117e4                 ← loop back
  --- LOOP END ---

  11828:                            ← final round landing
  11834: jal  10910 <AddRoundKey>   ← AddRoundKey round 10
  11840: ret
```

**Suy luận về structure:** AES-128 có 10 rounds. Loop chạy với `round = 1` đến `round = 10`. Khi `round == 10` thì skip MixColumns và jump ra. Instruction tại `0x117e8` (jal SubBytes) được execute **9 lần** (round 1–9) trước khi final round landing.

#### 7.4. Dùng gem5 execution trace để đếm rounds

```bash
./build/RISCV/gem5.opt --debug-flags=Exec ~/Desktop/run_aes.py 2>&1 \
  | grep "117e8\|117f0\|11806\|11818" > ~/Desktop/exec_trace.txt
head -50 ~/Desktop/exec_trace.txt
```

Output (extract):

```
126476:9029861000:  T0 : 0x117e8 @Cipher+44 : jal ra, -3590   ← Round 1 SubBytes
127027:9080545000:  T0 : 0x117f0 @Cipher+52 : jal ra, -3440   ← Round 1 ShiftRows
...
129179:9252774000:  T0 : 0x117e8 @Cipher+44 : jal ra, -3590   ← Round 2 SubBytes
...
148100:10812745000: T0 : 0x117e8 @Cipher+44 : jal ra, -3590   ← Round 9 SubBytes ← TARGET
148651:10863351000: T0 : 0x117f0 @Cipher+52 : jal ra, -3440
...
150803:11035659000: T0 : 0x117e8 @Cipher+44 : jal ra, -3590   ← Round 10 SubBytes (final)
```

**Fault window xác định:** Round 9 SubBytes bắt đầu tại tick `10812745000`.

gem5 dùng **ticks** (1 tick = 1 picosecond ở 1GHz → 1000 ticks = 1 cycle).

```
Round 1:  tick  9029861000
Round 2:  tick  9252774000
Round 3:  tick  9475351000
Round 4:  tick  9698381000
Round 5:  tick  9921481000
Round 6:  tick 10144134000
Round 7:  tick 10367004000
Round 8:  tick 10590015000
Round 9:  tick 10812745000  ← INJECT HERE
Round 10: tick 11035659000
```

---

### 10.8. Fault Injection — 3 approaches, 2 thất bại

#### 8.1. Approach 1: Modify register qua gem5 checkpoint (THẤT BẠI)

**Ý tưởng:** Chạy gem5 đến fault point, save checkpoint, modify register trong checkpoint file, restore và tiếp tục.

```python
# fault_inject.py
m5.simulate(FAULT_TICK - m5.curTick())

# Attempt 1: dùng threadContexts API
tc = system.cpu.threadContexts[0]
val = tc.readIntReg(10)
```

**Lỗi:**

```
AttributeError: object 'RiscvTimingSimpleCPU' has no attribute 'threadContexts'
```

**Suy luận:** gem5 v25 thay đổi API. `threadContexts` không còn là attribute trực tiếp của CPU object. Documentation cũ không còn chính xác.

**Attempt 2:** Dùng `cpuList[0].getContext(0)`:

```python
tc = system.cpu.cpuList[0].getContext(0)
sp = tc.readIntReg(2)
```

Lỗi tiếp: `No module named 'm5.mem'` khi cố đọc physical memory qua Python API.

**Attempt 3:** Save checkpoint, modify file trực tiếp.

```bash
m5.checkpoint('/home/clap/Desktop/ckpt_fault')
```

Checkpoint saved. File `m5.cpt` chứa:

```
regs.integer=0 0 0 0 0 0 0 0 236 23 1 0 ...
```

Đây là 32 registers của RISC-V, mỗi register 8 bytes (little-endian). Modify x10 (register thứ 10):

```python
reg_idx = 10 * 8
x10_bytes = bytes(vals[reg_idx:reg_idx+8])
x10_val = struct.unpack('<Q', x10_bytes)[0]
# 0x7ffffffffffffc78
x10_faulty = x10_val ^ 0x01
# 0x7ffffffffffffc79
```

Restore checkpoint → ciphertext:

```
e6d77bb40d7a36c7a89e67f324dcef97
```

Khác với correct! Nhưng khi thử flip các bit khác của x10, chỉ có duy nhất 1 fault work.

**Suy luận tại sao thất bại:**

Register x10 tại thời điểm đó là `0x7ffffffffffffc78` — đây là **stack pointer value** (địa chỉ), không phải AES state data. Flip bit 0 của stack pointer làm misalign một memory access nào đó → tình cờ tạo ra faulty output. Nhưng đây không phải DFA thật — không phải fault vào AES state.

Cần tìm **physical address** của AES state array và modify memory, không phải modify pointer.

**Attempt 4:** Tìm physical address qua virtual→physical mapping trong checkpoint.

```python
VADDR = 0x7ffffffffffffc78  # địa chỉ của state buffer trong main()
# Tìm page mapping
vpage = (VADDR // PAGE_SIZE) * PAGE_SIZE  # 0x7ffffffffffff000
# Found: paddr = 0x0000000000076c78
```

Đọc 16 bytes tại physical address đó từ `system.physmem.store0.pmem`:

```
aa aa aa aa aa aa aa aa aa aa aa aa aa aa aa aa
```

**Suy luận tại sao thất bại:** `0xaa` là pattern của uninitialized memory trong gem5. Checkpoint được save đúng lúc `jal SubBytes` được **fetch** nhưng trước khi SubBytes thực sự **setup stack frame và load data**. AES state chưa được copy vào vùng nhớ đó tại thời điểm checkpoint.

Save checkpoint muộn hơn (+50000 ticks vào bên trong SubBytes):

```python
FAULT_TICK = 10812745000 + 50000
m5.simulate(FAULT_TICK - m5.curTick())
m5.checkpoint(ckpt_dir2)
```

Đọc lại memory → `55 55 55 55 55 55 55 55...`

`0x55` cũng là uninitialized pattern. Đọc register s0 (frame pointer của SubBytes) → tính `s0 - 40` để tìm state pointer → dereference → `0xaaaaaaaaaaaaaaaa`.

**Kết luận approach 1:** gem5 SE mode memory layout rất khó predict. AES state lúc SubBytes đang xử lý nằm trong **registers**, không phải memory. `RiscvTimingSimpleCPU` với in-order execution: state được load vào registers từng byte, process, rồi store lại. Tại một tick ngẫu nhiên bên trong SubBytes, state không có địa chỉ stable trong memory để modify.

#### 8.2. Approach 2: GemFI framework (THẤT BẠI)

**Ý tưởng:** Dùng GemFI — fault injection framework được build sẵn cho gem5.

```bash
git clone https://github.com/crispy245/GemFI
```

```
fatal: repository 'https://github.com/crispy245/GemFI/' not found
```

**Repo bị xóa hoặc private.** Không có alternative fork tìm được.

#### 8.3. Approach 3: Source-level fault injection (THÀNH CÔNG)

**Insight:** Thay vì cố inject fault ở hardware level (register/memory trong simulator), inject trực tiếp vào source code với một hook được kích hoạt qua environment variable.

Đây là **standard approach** trong DFA research — nhiều paper inject fault "conceptually" tại đúng computation point mà không cần hardware-level glitch. Về mặt cryptographic analysis, điều quan trọng là fault xảy ra tại **đúng round và đúng loại operation**, không phải mechanism vật lý.

**Patch `aes.c`** — thêm fault hook vào Cipher loop:

```c
// Trước patch:
for (round = 1; ; ++round)
{
    SubBytes(state);
    ...
}

// Sau patch:
for (round = 1; ; ++round)
{
    // DFA fault injection hook — kích hoạt qua env var
    if (round == 9 && fault_byte >= 0 && fault_byte < 16) {
        uint8_t* s = (uint8_t*)state;
        s[fault_byte] ^= (1 << fault_bit);
    }
    SubBytes(state);
    ...
}
```

Thêm extern declaration ở đầu `aes.c`:

```c
extern int fault_byte;
extern int fault_bit;
```

Define globals trong `aes_main.c`:

```c
int fault_byte = -1;  // -1 = no fault
int fault_bit  = 0;
```

Đọc từ environment variable trong `main()`:

```c
char *fb = getenv("FAULT_BYTE");
char *fi = getenv("FAULT_BIT");
if (fb) fault_byte = atoi(fb);
if (fi) fault_bit  = atoi(fi);
```

**Lỗi compile:**

```
undefined reference to 'fault_byte'
```

**Suy luận:** `cat >` trong bash heredoc ghi đè `aes_main.c` nhưng lần đầu file có content cũ từ một attempt trước đó (có `extern uint8_t* get_state_ptr(void)` và không có `int fault_byte`). Linker tìm symbol `fault_byte` được declared trong `aes.c` nhưng không tìm thấy definition vì `aes_main.c` cũ không define nó.

**Fix:** Ghi lại hoàn toàn `aes_main.c` với đúng content, compile lại.

**Verify:**

```bash
FAULT_BYTE=0 FAULT_BIT=0 # chạy qua gem5 với env vars
```

Kết quả: faulty ciphertext khác correct. Approach hoạt động.

---

### 10.9. Collect faulty ciphertexts

#### 9.1. Automation script

Flip từng bit trong 16 bytes AES state = 16 × 8 = **128 combinations**:

```python
# collect_faults2.py
import subprocess

GEM5 = '/home/clap/Desktop/gem5/build/RISCV/gem5.opt'
correct = '3ad77bb40d7a3660a89ecaf32466ef97'
faulty_set = set()

def run_gem5(fault_byte=-1, fault_bit=0):
    script = f"""
import m5
from m5.objects import *
# ... gem5 setup ...
process.env = ['FAULT_BYTE={fault_byte}', 'FAULT_BIT={fault_bit}']
# ...
m5.simulate()
"""
    with open('/tmp/gem5_run.py', 'w') as f:
        f.write(script)
    result = subprocess.run([GEM5, '/tmp/gem5_run.py'],
                           capture_output=True, text=True)
    for line in result.stdout.split('\n'):
        if 'Ciphertext:' in line:
            return line.split('Ciphertext:')[1].strip()
    return None

for byte_idx in range(16):
    for bit in range(8):
        ct = run_gem5(byte_idx, bit)
        if ct and ct != correct and ct not in faulty_set:
            faulty_set.add(ct)
            print(f"byte={byte_idx} bit={bit}: {ct}")

with open('faulty_ciphertexts.txt', 'w') as f:
    for ct in faulty_set:
        f.write(ct + '\n')
```

#### 9.2. Kết quả — 128 faulty ciphertexts

```
byte=0  bit=0: 5dd77bb40d7a3616a89eeff32406ef97
byte=0  bit=1: 9fd77bb40d7a36b0a89e54f3241aef97
byte=0  bit=2: 0ad77bb40d7a364fa89ee9f324daef97
byte=0  bit=3: a0d77bb40d7a368ea89edef3247aef97
...
byte=7  bit=0: 3ad7dbb40dd23660d09ecaf32466ef5b
...
byte=15 bit=7: b3d77bb40d7a36cda89e8ef3243eef97
Total: 128 unique faulty ciphertexts
```

**Pattern quan sát được:**

- Fault vào byte 0: ảnh hưởng đến bytes 0, 4–7 (1 column của AES state)
- Fault vào byte 1: ảnh hưởng đến bytes 1, khác nhau về pattern
- Các fault khác nhau tạo ra partial changes ở các vị trí predictable

Đây là **đúng behavior** của DFA: fault ở một byte propagate theo AES MixColumns column structure.

---

### 10.10. DFA Key Recovery với PhoenixAES

#### 10.1. Install PhoenixAES

```bash
pip3 install phoenixAES --break-system-packages
```

#### 10.2. Tìm đúng API

Lần đầu gọi:

```python
phoenixAES.crack_file('faulty_ciphertexts.txt',
                      bytes.fromhex(correct),
                      verbose=True)
```

Lỗi:

```
TypeError: 'int' object is not iterable
```

**Suy luận:** `crack_file` nhận correct ciphertext theo cách khác. Đọc docstring:

```python
help(phoenixAES.crack_file)
# crack_file(r9_filename, lastroundkeys=[], encrypt=True, ...)
# :param r9_filename: file containing the output reference on the FIRST LINE
#                     and glitched outputs on next lines, as hex strings
```

**Vấn đề:** File format yêu cầu correct ciphertext là **dòng đầu tiên** của file, không phải parameter riêng.

#### 10.3. Tạo file đúng format

```bash
echo "3ad77bb40d7a3660a89ecaf32466ef97" > dfa_input.txt
cat faulty_ciphertexts.txt >> dfa_input.txt
```

#### 10.4. Chạy PhoenixAES

```python
import phoenixAES

result = phoenixAES.crack_file('dfa_input.txt', verbose=True)
print(f"Round 10 key: {result}")
```

Output:

```
Last round key #N found:
D014F9A8C9EE2589E13F0CC8B6630CA6
Round 10 key: D014F9A8C9EE2589E13F0CC8B6630CA6
```

#### 10.5. Verify round 10 key

PhoenixAES trả về round 10 key. Cần verify bằng cách expand master key và check kết quả:

```python
from Crypto.Cipher import AES

def expand_key(master):
    # Standard AES-128 key schedule
    sbox = [0x63,0x7c,0x77,...] # full S-box
    rcon = [0x01,0x02,0x04,0x08,0x10,0x20,0x40,0x80,0x1b,0x36]
    w = [list(master[i:i+4]) for i in range(0, 16, 4)]
    for i in range(4, 44):
        temp = w[i-1][:]
        if i % 4 == 0:
            temp = xor_w(sub_word(rot_word(temp)), [rcon[i//4-1], 0, 0, 0])
        w.append(xor_w(w[i-4], temp))
    return w

master = bytes.fromhex('2B7E151628AED2A6ABF7158809CF4F3C')
w = expand_key(master)
r10 = bytes(b for word in w[40:44] for b in word)
print(r10.hex().upper())
# D014F9A8C9EE2589E13F0CC8B6630CA6  ✓
```

#### 10.6. Note về reverse key schedule

Attempt đầu tiên viết reverse key schedule để đi từ round 10 key về master key — kết quả sai (`2B008376...` thay vì `2B7E15...`). Lý do: logic reverse ở `rcon` index bị off-by-one.

Thay vì debug reverse algorithm, verify theo chiều thuận: expand known master key → round 10 key → compare với PhoenixAES output. Match → confirm attack đúng.

```
Round 10 key từ expand(master): D014F9A8C9EE2589E13F0CC8B6630CA6
PhoenixAES recovered:           D014F9A8C9EE2589E13F0CC8B6630CA6
Match: True ✓

Encrypt verify:
  AES(master, plaintext) = 3ad77bb40d7a3660a89ecaf32466ef97
  Expected NIST vector   = 3ad77bb40d7a3660a89ecaf32466ef97
  Match: True ✓
```

---

### 10.11. Lockstep Defense

#### 11.1. Thiết kế

Lockstep chạy AES **hai lần** với cùng key và plaintext:
- **CPU A**: chạy bình thường, có thể bị fault
- **CPU B**: chạy clean (fault_byte = -1 cứng)

Sau khi cả hai xong, comparator so sánh output. Nếu khác nhau → fault detected → abort.

**Tại sao Lockstep chặn được DFA?**

DFA cần **faulty ciphertext output** để phân tích. Nếu hệ thống abort trước khi output ciphertext ra ngoài, attacker không có gì để crack.

#### 11.2. Implementation

```c
// aes_lockstep.c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include "aes.h"

int fault_byte = -1;
int fault_bit  = 0;

int main() {
    uint8_t key[16] = { 0x2b, 0x7e, ... };
    uint8_t plaintext[16] = { 0x6b, 0xc1, ... };

    char *fb = getenv("FAULT_BYTE");
    char *fi = getenv("FAULT_BIT");
    if (fb) fault_byte = atoi(fb);
    if (fi) fault_bit  = atoi(fi);

    // --- CPU A: có thể bị fault ---
    struct AES_ctx ctx_a;
    AES_init_ctx(&ctx_a, key);
    uint8_t buf_a[16];
    memcpy(buf_a, plaintext, 16);
    AES_ECB_encrypt(&ctx_a, buf_a);

    // --- CPU B: chạy clean ---
    int saved_fb = fault_byte;
    fault_byte = -1;          // disable fault cho CPU B
    struct AES_ctx ctx_b;
    AES_init_ctx(&ctx_b, key);
    uint8_t buf_b[16];
    memcpy(buf_b, plaintext, 16);
    AES_ECB_encrypt(&ctx_b, buf_b);
    fault_byte = saved_fb;    // restore

    // --- Lockstep comparator ---
    if (memcmp(buf_a, buf_b, 16) != 0) {
        printf("[LOCKSTEP] FAULT DETECTED — outputs diverged!\n");
        printf("[LOCKSTEP] CPU A: ");
        for (int i = 0; i < 16; i++) printf("%02x", buf_a[i]);
        printf("\n[LOCKSTEP] CPU B: ");
        for (int i = 0; i < 16; i++) printf("%02x", buf_b[i]);
        printf("\n[LOCKSTEP] Execution ABORTED — no ciphertext output\n");
        return 1;  // abort, ciphertext KHÔNG được output
    }

    // Chỉ output nếu hai CPU đồng thuận
    printf("Ciphertext: ");
    for (int i = 0; i < 16; i++) printf("%02x", buf_a[i]);
    printf("\n");
    return 0;
}
```

#### 11.3. Compile và test

```bash
riscv64-linux-gnu-gcc -O0 -g -static -o aes_lockstep aes_lockstep.c aes.c -I.
```

gem5 run script hỗ trợ pass arguments:

```python
# run_lockstep.py
import sys
fault_byte = sys.argv[1] if len(sys.argv) > 1 else "-1"
fault_bit  = sys.argv[2] if len(sys.argv) > 2 else "0"
# ...
process.env = [f'FAULT_BYTE={fault_byte}', f'FAULT_BIT={fault_bit}']
```

#### 11.4. Demo kết quả

**Scenario 1: Không có fault**

```bash
./build/RISCV/gem5.opt run_lockstep.py -- -1 0
```

```
Ciphertext: 3ad77bb40d7a3660a89ecaf32466ef97
```

Không có fault → hai CPU đồng thuận → output bình thường. ✓

**Scenario 2: Fault inject, byte 0 bit 0**

```bash
./build/RISCV/gem5.opt run_lockstep.py -- 0 0
```

```
[LOCKSTEP] FAULT DETECTED — outputs diverged!
[LOCKSTEP] CPU A: 5dd77bb40d7a3616a89eeff32406ef97
[LOCKSTEP] CPU B: 3ad77bb40d7a3660a89ecaf32466ef97
[LOCKSTEP] Execution ABORTED — no ciphertext output
```

Fault detected. Không có output. Attacker không có faulty ciphertext để crack. ✓

**Scenario 3: Fault inject, byte 5 bit 3**

```bash
./build/RISCV/gem5.opt run_lockstep.py -- 5 3
```

```
[LOCKSTEP] FAULT DETECTED — outputs diverged!
[LOCKSTEP] CPU A: 2bd77bb40d7a3646a89e6ef324ddef97
[LOCKSTEP] CPU B: 3ad77bb40d7a3660a89ecaf32466ef97
[LOCKSTEP] Execution ABORTED — no ciphertext output
```

Lockstep detect 100% trong tất cả 128 fault scenarios. ✓

---

### 10.12. Kết quả tổng hợp

#### 12.1. Attack pipeline

```
Correct run:
  plaintext: 6bc1bee22e409f96e93d7e117393172a
  key:       2b7e151628aed2a6abf7158809cf4f3c
  output:    3ad77bb40d7a3660a89ecaf32466ef97

Fault run (byte=0, bit=0):
  fault injected before SubBytes round 9
  output:    5dd77bb40d7a3616a89eeff32406ef97  ← faulty

... (128 total faulty ciphertexts)

PhoenixAES:
  input:  correct + 128 faulty ciphertexts
  output: Round 10 key = D014F9A8C9EE2589E13F0CC8B6630CA6

Key schedule verify:
  expand(2B7E151628AED2A6ABF7158809CF4F3C)[round 10]
  = D014F9A8C9EE2589E13F0CC8B6630CA6  ← MATCH ✓
```

#### 12.2. Defense pipeline

```
Fault injected:
  CPU A output: 5dd77bb40d7a3616a89eeff32406ef97
  CPU B output: 3ad77bb40d7a3660a89ecaf32466ef97
  Divergence → ABORT

Attacker thu được: (nothing)
PhoenixAES input:  (nothing)
Attack result:     FAILED ✗
```

#### 12.3. Bảng số liệu

| Item | Giá trị |
|------|---------|
| Plaintext | `6bc1bee22e409f96e93d7e117393172a` |
| Master key (secret) | `2b7e151628aed2a6abf7158809cf4f3c` |
| Correct ciphertext | `3ad77bb40d7a3660a89ecaf32466ef97` |
| Fault target | Round 9, trước SubBytes |
| Faulty ciphertexts collected | 128 |
| Round 10 key (recovered) | `d014f9a8c9ee2589e13f0cc8b6630ca6` |
| Master key match | ✓ |
| Lockstep detection rate | 128/128 (100%) |

---

### 10.13. Lessons Learned và Technical Notes

#### 13.1. gem5 v25 API thay đổi so với documentation

Documentation online và nhiều tutorial dùng API của gem5 v20–v22:
- `cpu.threadContexts[0]` → không còn tồn tại trong v25
- `m5.mem` module → không tồn tại
- Một số config parameter thay đổi tên

**Lesson:** Với tools evolve nhanh như gem5, luôn check source code hoặc RELEASE-NOTES thay vì rely vào tutorial cũ. Lỗi `AttributeError` là signal rõ nhất API đã thay đổi.

#### 13.2. gem5 SE mode memory layout

SE mode emulate syscalls nhưng không có real address space translation như full-system mode. Virtual→physical mapping trong SE mode phụ thuộc vào khi nào program thực sự write vào memory. Uninitialized regions (`0xaa`, `0x55`) là pattern của gem5 memory initialization — không phải AES data.

**Lesson:** Khi làm fault injection ở memory level trong SE mode, cần chạy đến một point **sau khi** data đã được write vào target address. Timing rất quan trọng.

#### 13.3. GemFI repo unavailable

`github.com/crispy245/GemFI` — repo bị deleted/private tại thời điểm experiment. Không có archived copy.

**Lesson:** Dependency vào third-party repos là fragile. Source-level fault injection là fallback approach tốt và thực ra cleaner hơn cho reproducibility.

#### 13.4. Source-level fault injection vs hardware-level

Source-level inject (patch source code, compile, chạy) vs hardware-level inject (modify register/memory trong simulator) — cả hai đều valid cho mục đích cryptographic analysis. Điều quan trọng là:

1. Fault xảy ra tại đúng **round** (round 9)
2. Fault là **byte-level flip** vào AES state
3. Fault xảy ra **trước SubBytes** (không phải sau)

Source-level patch đảm bảo tất cả 3 điều này một cách chính xác và reproducible.

#### 13.5. PhoenixAES file format

`crack_file()` không nhận correct ciphertext làm parameter — nó đọc từ **dòng đầu tiên** của file. Đây là convention của tool, không được document rõ trên README nhưng có trong docstring.

---

### 10.14. References

1. Boneh, D., DeMillo, R. A., & Lipton, R. J. (1997). *On the Importance of Checking Cryptographic Protocols for Faults.* EUROCRYPT 1997. Springer, LNCS 1233.

2. Giraud, C. (2004). *DFA on AES.* AES4 — 4th Conference on the Advanced Encryption Standard. Springer.

3. Brier, E., Dottax, E., & Prouff, E. (2009). *Differential Fault Analysis on Stripped-Down AES.* ACM Workshop on Wireless Security.

4. gem5 simulator: https://www.gem5.org · v25.1.0.1

5. tiny-AES-c: https://github.com/kokke/tiny-AES-c

6. PhoenixAES / JeanGrey: https://github.com/SideChannelMarvels/JeanGrey

7. NIST FIPS 197: Advanced Encryption Standard. Appendix B — Cipher Example.

8. Luk, C.-K. et al. (2005). *Pin: Building Customized Program Analysis Tools with Dynamic Instrumentation.* PLDI 2005. (Background on dynamic binary instrumentation, context for SE mode)

---

## 11. Kết Luận

Khi bắt đầu bài viết này, chúng ta chỉ nói về một hiện tượng rất đơn giản:

Một bit thay đổi trạng thái.

```text
0 → 1

hoặc

1 → 0
```

Trong những hệ thống đầu tiên, một bit flip có thể chỉ dẫn tới một lỗi tính toán nhỏ hoặc một chương trình bị crash.

Tuy nhiên, khi mật độ transistor tiếp tục tăng và máy tính ngày càng trở thành hạ tầng của xã hội hiện đại, tác động của những lỗi tưởng chừng rất nhỏ này đã thay đổi hoàn toàn.

Một bit flip xuất hiện trong:

* Bộ nhớ DRAM.
* Cache.
* Register File.
* Logic xử lý của CPU.

đều có thể trở thành điểm khởi đầu của một chuỗi sự kiện dẫn tới sai lệch trạng thái hệ thống.

```text
Bit Flip
    ↓
Error
    ↓
State Corruption
    ↓
Incorrect Computation
    ↓
System Failure
```

Đây chính là lý do các kiến trúc chịu lỗi ra đời.

Thay vì giả định phần cứng luôn hoạt động hoàn hảo, Fault-Tolerant Architecture xuất phát từ một quan điểm thực tế hơn:

> Lỗi là điều không thể tránh khỏi.

Do đó nhiệm vụ của hệ thống không phải là loại bỏ hoàn toàn lỗi, mà là phát hiện, cô lập và kiểm soát hậu quả của lỗi trước khi chúng phát triển thành failure.

---

Trong bài viết, chúng ta đã khảo sát bốn cơ chế chịu lỗi quan trọng đang được sử dụng rộng rãi trong các hệ thống hiện đại.

ECC bổ sung khả năng phát hiện và sửa lỗi trực tiếp trong bộ nhớ.

Chipkill mở rộng phạm vi bảo vệ từ lỗi bit đơn lẻ sang lỗi ở cấp DRAM chip.

TMR sử dụng nhân bản phần cứng và majority voting để che giấu lỗi trong logic xử lý.

Lockstep liên tục giám sát tính nhất quán của trạng thái thực thi nhằm phát hiện sai lệch ở cấp CPU.

Mặc dù hoạt động ở các tầng khác nhau của hệ thống, tất cả các cơ chế này đều hướng tới cùng một mục tiêu:

```text
Fault
  ↓
Detect
  ↓
Contain
  ↓
Recover
```

Thay vì để một fault đơn lẻ phát triển thành failure của toàn hệ thống.

---

Tuy nhiên, có lẽ kết luận thú vị nhất không nằm ở Reliability Engineering.

Nó nằm ở giao điểm giữa Reliability và Security.

Trong nhiều thập kỷ, các hiện tượng như SEU, soft error hay transient fault được xem là những vấn đề thuần túy về độ tin cậy.

Nhưng khi các kỹ thuật như:

* Voltage Glitching
* Clock Glitching
* Electromagnetic Fault Injection
* Laser Fault Injection

xuất hiện, cộng đồng nghiên cứu nhận ra rằng những hiện tượng vật lý tương tự hoàn toàn có thể được tạo ra một cách có chủ đích.

Một bit flip không còn chỉ là hậu quả của tia vũ trụ.

Nó có thể trở thành công cụ tấn công.

```text
Natural Fault
        ↓
Reliability Problem

Injected Fault
        ↓
Security Problem
```

Từ góc nhìn của phần cứng, hai hiện tượng này đôi khi gần như không thể phân biệt.

Điều đó khiến ranh giới giữa Reliability Engineering và Hardware Security ngày càng mờ đi.

---

Có thể nói rằng phần lớn các cơ chế được trình bày trong bài viết đều được thiết kế để chống lại lỗi ngẫu nhiên.

Tuy nhiên, chính các cơ chế đó ngày nay cũng đang đóng vai trò quan trọng trong việc phát hiện hoặc giảm thiểu các cuộc tấn công Fault Injection.

ECC, Chipkill, TMR và Lockstep không chỉ giúp hệ thống đáng tin cậy hơn.

Chúng còn giúp hệ thống khó bị khai thác hơn.

Đó là một ví dụ điển hình cho việc các nghiên cứu về Reliability cuối cùng lại trở thành nền tảng của Hardware Security hiện đại.

---

Cuối cùng, có lẽ bài học quan trọng nhất là:

> Trong các hệ thống hiện đại, câu hỏi không còn là liệu lỗi có xảy ra hay không, mà là hệ thống sẽ phản ứng như thế nào khi lỗi xảy ra.

Một kiến trúc tốt không phải là kiến trúc không bao giờ gặp lỗi.

Một kiến trúc tốt là kiến trúc có thể tiếp tục hoạt động an toàn, đáng tin cậy và có thể kiểm soát được ngay cả khi lỗi xuất hiện.

Và đôi khi, toàn bộ câu chuyện đó bắt đầu chỉ từ một bit bị lật.
