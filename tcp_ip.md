# Mô hình TCP/IP và TCP Connection — Tài liệu chi tiết

## Mục lục

1. [Tổng quan mô hình TCP/IP](#1-tổng-quan-mô-hình-tcpip)
2. [Từng tầng: chức năng, vị trí chạy, giao thức](#2-từng-tầng-chức-năng-vị-trí-chạy-giao-thức)
3. [TCP là gì](#3-tcp-là-gì)
4. [TCP connection thực chất là gì](#4-tcp-connection-thực-chất-là-gì)
5. [Ba bước bắt tay (handshake) và bốn bước đóng (teardown)](#5-ba-bước-bắt-tay-handshake-và-bốn-bước-đóng-teardown)
6. [Cấu trúc header TCP](#6-cấu-trúc-header-tcp)
7. [Cách gói tin truyền qua mạng — encapsulation](#7-cách-gói-tin-truyền-qua-mạng--encapsulation)
8. [Sliding window & flow control](#8-sliding-window--flow-control)
9. [Congestion control](#9-congestion-control)
10. [Ví dụ thực tế: một HTTP request đi qua TCP như thế nào](#10-ví-dụ-thực-tế-một-http-request-đi-qua-tcp-như-thế-nào)
11. [Tổng kết các khái niệm cốt lõi](#11-tổng-kết-các-khái-niệm-cốt-lõi)

---

## 1. Tổng quan mô hình TCP/IP

Mô hình TCP/IP (còn gọi là Internet Protocol Suite) là bộ khung 4 tầng mô tả cách dữ liệu di chuyển từ ứng dụng trên máy này sang ứng dụng trên máy khác qua mạng. Đây là mô hình thực tế được dùng trên Internet (khác với mô hình OSI 7 tầng chỉ mang tính lý thuyết/giáo dục).

```
┌─────────────────────────────────────┐
│  4. Application Layer                │  HTTP, DNS, SMTP, FTP...
├─────────────────────────────────────┤
│  3. Transport Layer                  │  TCP, UDP
├─────────────────────────────────────┤
│  2. Internet Layer                   │  IP, ICMP, ARP
├─────────────────────────────────────┤
│  1. Link Layer (Network Access)      │  Ethernet, WiFi, driver, NIC
└─────────────────────────────────────┘
```

So sánh nhanh với OSI 7 tầng:

| OSI (7 tầng)                       | TCP/IP (4 tầng) |
| ---------------------------------- | --------------- |
| Application, Presentation, Session | Application     |
| Transport                          | Transport       |
| Network                            | Internet        |
| Data Link, Physical                | Link            |

**Nguyên tắc cốt lõi**: mỗi tầng chỉ nói chuyện với tầng tương ứng ở máy đối diện, thông qua tầng ngay dưới nó ở máy mình. Tầng Transport ở máy A "cảm giác" như đang nói chuyện trực tiếp với tầng Transport ở máy B, dù thực tế dữ liệu phải đi qua Internet layer → Link layer → dây mạng → ngược lại.

---

## 2. Từng tầng: chức năng, vị trí chạy, giao thức

### 2.1. Application Layer (tầng ứng dụng)

- **Chạy ở đâu**: trong process của ứng dụng, ở **user space** (không gian người dùng, không phải kernel).
- **Chức năng**: định nghĩa cách dữ liệu được format và trao đổi theo ngữ nghĩa của ứng dụng cụ thể — ví dụ HTTP định nghĩa request/response, DNS định nghĩa cách tra cứu tên miền.
- **Giao thức tiêu biểu**: HTTP/HTTPS, DNS, SMTP, FTP, SSH, WebSocket.
- **Ví dụ**: khi trình duyệt gọi `fetch('https://example.com')`, chính trình duyệt (ở user space) tạo ra chuỗi text HTTP request — đây là dữ liệu tầng Application.

### 2.2. Transport Layer (tầng giao vận)

- **Chạy ở đâu**: trong **kernel** của hệ điều hành. Ứng dụng không tự thực hiện logic TCP/UDP — nó gọi vào kernel qua **socket API** (system call như `connect()`, `send()`, `recv()`).
- **Chức năng**: đảm bảo dữ liệu đến đúng ứng dụng (qua port), và tùy giao thức mà đảm bảo độ tin cậy, thứ tự, kiểm soát luồng.
- **Hai giao thức chính**:
  - **TCP**: connection-oriented, tin cậy, có thứ tự, kiểm soát luồng và tắc nghẽn.
  - **UDP**: connectionless, không đảm bảo tin cậy hay thứ tự, nhẹ và nhanh hơn — dùng cho streaming, game, DNS query.
- **Đơn vị dữ liệu**: gọi là **segment** (TCP) hoặc **datagram** (UDP).

### 2.3. Internet Layer (tầng mạng)

- **Chạy ở đâu**: cũng trong **kernel**, ngay dưới Transport layer.
- **Chức năng**: định tuyến (routing) — xác định đường đi của gói tin từ máy nguồn tới máy đích qua nhiều router trung gian, dựa vào địa chỉ IP.
- **Giao thức chính**: IP (IPv4/IPv6), ICMP (dùng cho ping, thông báo lỗi), ARP (map địa chỉ IP sang địa chỉ MAC trong mạng LAN).
- **Đơn vị dữ liệu**: gọi là **packet**.
- **Đặc điểm quan trọng**: IP là giao thức **best-effort** — không đảm bảo gói tin sẽ đến, đến đúng thứ tự, hay không bị trùng lặp. Đây chính là lý do TCP cần tồn tại: để "vá" các đảm bảo mà IP không cung cấp.

### 2.4. Link Layer (tầng liên kết / truy cập mạng)

- **Chạy ở đâu**: một phần trong kernel (driver), một phần trong **firmware của card mạng (NIC)**.
- **Chức năng**: truyền dữ liệu qua môi trường vật lý cụ thể (dây đồng, cáp quang, sóng WiFi) trong phạm vi một mạng LAN/segment, dùng địa chỉ MAC để xác định thiết bị.
- **Giao thức chính**: Ethernet, WiFi (802.11), PPP.
- **Đơn vị dữ liệu**: gọi là **frame**.

---

## 3. TCP là gì

TCP (Transmission Control Protocol) là giao thức tầng Transport, cung cấp cho ứng dụng một **luồng byte tin cậy, có thứ tự, song công (full-duplex)** giữa hai điểm cuối, xây dựng trên nền một mạng gói tin không đáng tin cậy (IP).

### Các đặc tính cốt lõi của TCP

| Đặc tính                | Ý nghĩa                                                                                                   |
| ----------------------- | --------------------------------------------------------------------------------------------------------- |
| **Connection-oriented** | Phải thiết lập connection (handshake) trước khi truyền dữ liệu                                            |
| **Reliable (tin cậy)**  | Đảm bảo dữ liệu đến nơi, dùng ACK + retransmit nếu mất gói                                                |
| **Ordered (có thứ tự)** | Dữ liệu được ghép lại đúng thứ tự gửi đi, dù các gói có thể đến sai thứ tự qua mạng                       |
| **Byte-stream**         | Ứng dụng thấy một luồng byte liên tục, không thấy ranh giới gói tin — TCP tự quyết định cách chia segment |
| **Full-duplex**         | Cả hai bên có thể gửi và nhận đồng thời trên cùng một connection                                          |
| **Flow control**        | Bên nhận có thể báo cho bên gửi biết "chậm lại", tránh làm tràn buffer                                    |
| **Congestion control**  | Tự động điều chỉnh tốc độ gửi dựa trên tình trạng tắc nghẽn của mạng                                      |

### TCP không phải là...

- Không phải một "đường dây" vật lý — không có kết nối liên tục nào được giữ trên mạng.
- Không phải là thứ đảm bảo mã hóa hay bảo mật (đó là việc của TLS, chạy trên TCP).
- Không phải là thứ duy nhất trên tầng Transport — UDP cũng tồn tại song song, phục vụ mục đích khác.

---

## 4. TCP connection thực chất là gì

Đây là phần quan trọng nhất để hiểu đúng bản chất TCP.

### 4.1. Không có "dây nối" — chỉ có trạng thái đồng bộ

TCP connection **không phải một thực thể tồn tại trên mạng**. Mỗi gói tin đi qua Internet độc lập với nhau, có thể đi qua các đường (route) khác nhau, đến sai thứ tự, hoặc bị mất. "Connection" thực chất là:

> Hai bảng trạng thái (state) nằm trong bộ nhớ kernel của hai máy, được giữ đồng bộ với nhau thông qua việc trao đổi sequence number và acknowledgment number.

Nói cách khác, TCP là một **"ảo giác được thỏa thuận"** về một luồng byte đáng tin cậy, xây trên nền một mạng gói tin không đáng tin cậy.

### 4.2. Transmission Control Block (TCB)

Ở mỗi đầu, kernel giữ một cấu trúc dữ liệu gọi là **TCB**, chứa:

- **4-tuple định danh** (khóa duy nhất của connection): `(source IP, source port, destination IP, destination port)`
- **Sequence number** hiện tại — mình đã gửi dữ liệu tới byte thứ mấy
- **Acknowledgment number** — đối phương đã xác nhận nhận tới byte thứ mấy
- **Send buffer / Receive buffer** — vùng nhớ tạm giữ dữ liệu chờ gửi hoặc chờ ứng dụng đọc
- **Receive window** — dung lượng còn trống trong receive buffer, dùng cho flow control
- **Trạng thái hiện tại** trong state machine (xem mục 4.3)
- **Timer** để phát hiện timeout và kích hoạt retransmit

**Điểm quan trọng**: đây là bản sao trạng thái **riêng của mỗi máy**. Nếu một bên bị crash đột ngột (mất điện, rút dây) mà không gửi được gói FIN, bên còn lại **không có cách nào biết ngay lập tức** — nó vẫn tưởng connection còn sống cho tới khi hết timeout hoặc thử gửi dữ liệu mà không nhận được ACK.

### 4.3. State machine của TCP

Một connection TCP di chuyển qua các trạng thái sau (rút gọn, phía server đang lắng nghe):

```
CLOSED
  │
  │ (server) gọi listen()
  ▼
LISTEN
  │
  │ nhận SYN từ client
  ▼
SYN_RECEIVED  ──────────────►  ESTABLISHED
                (nhận ACK)          │
                                    │ trao đổi dữ liệu
                                    │
                              (gửi/nhận FIN)
                                    ▼
                         FIN_WAIT / CLOSE_WAIT
                                    │
                                    ▼
                              TIME_WAIT / CLOSED
```

Phía client:

```
CLOSED
  │ gọi connect(), gửi SYN
  ▼
SYN_SENT
  │ nhận SYN-ACK, gửi ACK
  ▼
ESTABLISHED
```

**`TIME_WAIT` đáng chú ý**: sau khi đóng connection, bên chủ động đóng (thường là client) giữ trạng thái này trong một khoảng thời gian (thường 2×MSL — Maximum Segment Lifetime) trước khi giải phóng hẳn. Mục đích: đảm bảo các gói tin trễ (duplicate, delayed) từ connection cũ không bị hiểu nhầm là thuộc về một connection mới dùng lại cùng 4-tuple.

---

## 5. Ba bước bắt tay (handshake) và bốn bước đóng (teardown)

### 5.1. Thiết lập connection — 3-way handshake

| Bước | Gói tin         | Nội dung                  | Ý nghĩa                                            |
| ---- | --------------- | ------------------------- | -------------------------------------------------- |
| 1    | Client → Server | `SYN, seq=x`              | "Tôi muốn kết nối, số thứ tự của tôi bắt đầu từ x" |
| 2    | Server → Client | `SYN-ACK, seq=y, ack=x+1` | "Đồng ý, số của tôi bắt đầu từ y, tôi đã nhận x"   |
| 3    | Client → Server | `ACK, ack=y+1`            | "Tôi đã nhận y, có thể bắt đầu gửi dữ liệu"        |

Sau bước 3, cả hai bên **tự chuyển** trạng thái nội bộ sang `ESTABLISHED`. Không có "thông báo đồng thời" nào — mỗi bên tự quyết định dựa trên gói tin nó nhận được.

**Tại sao cần 3 bước chứ không phải 2?** Vì TCP là full-duplex — cả hai chiều (client→server và server→client) đều cần số thứ tự riêng và đều cần được xác nhận riêng. Bước 2 gộp chung SYN của server và ACK cho SYN của client để tiết kiệm một round-trip.

### 5.2. Đóng connection — 4-way teardown

Vì TCP là full-duplex, việc đóng connection cũng cần đóng độc lập từng chiều:

| Bước | Gói tin      | Ý nghĩa                                            |
| ---- | ------------ | -------------------------------------------------- |
| 1    | A → B: `FIN` | "Tôi không còn gì để gửi nữa"                      |
| 2    | B → A: `ACK` | "Đã nhận, nhưng tôi có thể vẫn còn dữ liệu để gửi" |
| 3    | B → A: `FIN` | "Giờ tôi cũng xong rồi"                            |
| 4    | A → B: `ACK` | "Đã nhận, đóng hoàn toàn"                          |

Đây là lý do gọi là **half-close**: một bên có thể ngừng gửi (đã FIN) nhưng vẫn tiếp tục nhận dữ liệu từ bên kia cho tới khi cả hai FIN đều hoàn tất.

---

## 6. Cấu trúc header TCP

Mỗi segment TCP có một header khoảng 20 byte (không tính options), gắn trước phần dữ liệu (payload):

| Trường                | Kích thước | Ý nghĩa                                                                    |
| --------------------- | ---------- | -------------------------------------------------------------------------- |
| Source port           | 16 bit     | Port của ứng dụng gửi                                                      |
| Destination port      | 16 bit     | Port của ứng dụng nhận                                                     |
| Sequence number       | 32 bit     | Vị trí byte đầu tiên của segment này trong luồng dữ liệu                   |
| Acknowledgment number | 32 bit     | Byte tiếp theo mà bên gửi mong nhận được (chỉ có ý nghĩa khi flag ACK bật) |
| Flags                 | —          | `SYN`, `ACK`, `FIN`, `RST`, `PSH`, `URG` — điều khiển hành vi connection   |
| Window size           | 16 bit     | Bên gửi header này còn bao nhiêu chỗ trống trong receive buffer            |
| Checksum              | 16 bit     | Kiểm tra lỗi truyền                                                        |
| Options               | thay đổi   | Ví dụ MSS (Maximum Segment Size), window scaling, SACK                     |

**Lưu ý quan trọng**: header TCP **không chứa địa chỉ IP**. TCP chỉ biết về port; việc gói tin đi đường nào để tới đúng máy là trách nhiệm của tầng IP bên dưới.

---

## 7. Cách gói tin truyền qua mạng — Encapsulation

Khi ứng dụng gửi dữ liệu, dữ liệu đi qua từng tầng và được "đóng gói" thêm header ở mỗi tầng — gọi là **encapsulation**:

```
Application data:        [ HTTP request text ]

Transport (TCP):    [TCP header][ HTTP request text ]           → gọi là SEGMENT

Internet (IP):   [IP header][TCP header][ HTTP request text ]   → gọi là PACKET

Link (Ethernet): [MAC header][IP header][TCP header][data][MAC trailer]  → gọi là FRAME
```

Ở máy nhận, quá trình diễn ra ngược lại — gọi là **decapsulation**: mỗi tầng lột header của mình ra, rồi chuyển phần còn lại lên tầng trên.

### Ví dụ cụ thể: gửi request từ máy A đến máy B qua một router

1. **Application (A)**: trình duyệt tạo chuỗi `GET / HTTP/1.1\r\nHost: example.com...`
2. **Transport (A)**: kernel cắt chuỗi này thành segment(s), gắn header TCP với `source port = 51000` (port tạm do OS cấp), `destination port = 443`
3. **Internet (A)**: gắn header IP với `source IP = A`, `destination IP = B`
4. **Link (A)**: gắn header Ethernet với địa chỉ MAC của A và địa chỉ MAC của **router gần nhất** (không phải MAC của B — vì B ở mạng khác)
5. Frame đi qua dây/wifi tới router
6. **Router**: lột header Link, đọc header IP để biết đích, tìm route tiếp theo, gắn lại header Link mới (MAC của router và MAC của chặng tiếp theo), gửi tiếp — IP và TCP header giữ nguyên suốt hành trình, chỉ header Link thay đổi ở mỗi chặng
7. Lặp lại cho tới khi frame tới máy B
8. **Máy B** lột từng lớp: Link → thấy IP header, chuyển lên Internet layer → lột IP header, thấy TCP header, chuyển lên Transport layer → TCP layer khớp 4-tuple với TCB đang có, đưa dữ liệu vào receive buffer đúng connection, ghép đúng thứ tự theo sequence number → khi ứng dụng ở B gọi `read()`, dữ liệu được trả lên Application layer

**Điểm mấu chốt**: header IP và TCP giữ nguyên xuyên suốt hành trình (trừ vài trường hợp đặc biệt như NAT sửa địa chỉ/port), chỉ có header Link layer bị tháo ra và gắn lại ở mỗi router trung gian, vì mỗi chặng chỉ có ý nghĩa trong phạm vi một mạng LAN.

---

## 8. Sliding window & flow control

TCP không gửi hết dữ liệu một lúc rồi chờ ACK cho từng byte — điều đó sẽ rất chậm vì phải chờ round-trip cho mỗi đơn vị nhỏ. Thay vào đó, TCP dùng cơ chế **sliding window**:

- Bên gửi được phép gửi nhiều segment liên tiếp mà không cần chờ ACK từng cái, miễn là tổng dữ liệu chưa được ACK không vượt quá **window size** mà bên nhận công bố.
- Khi ACK cho các byte đầu tiên về, "cửa sổ" trượt tới, cho phép gửi thêm dữ liệu mới.
- **Flow control** chính là việc bên nhận dùng trường `window size` trong header để nói "tôi chỉ còn chỗ trống bằng này trong buffer, đừng gửi nhiều hơn" — tránh làm tràn bộ nhớ của bên nhận nếu nó xử lý dữ liệu chậm hơn tốc độ gửi.

---

## 9. Congestion control

Khác với flow control (bảo vệ **bên nhận**), congestion control bảo vệ **bản thân mạng** khỏi bị quá tải. TCP tự ước lượng "sức chứa" của đường truyền bằng cách:

- Bắt đầu gửi chậm (**slow start**) — tăng dần lượng dữ liệu gửi mỗi round-trip nếu không có mất gói
- Khi phát hiện mất gói (không nhận ACK đúng hạn, hoặc nhận duplicate ACK) → coi đó là dấu hiệu tắc nghẽn, **giảm mạnh tốc độ gửi**
- Sau đó tăng dần trở lại theo cơ chế **congestion avoidance** (thường là tăng tuyến tính)

Các thuật toán cụ thể (Reno, CUBIC, BBR...) khác nhau ở cách tính "tăng bao nhiêu, giảm bao nhiêu", nhưng nguyên lý chung là dùng việc mất gói (hoặc độ trễ tăng, với BBR) làm tín hiệu gián tiếp để suy đoán tình trạng mạng — vì TCP không có cách nào hỏi trực tiếp router "mạng có đang tắc nghẽn không?".

---

## 10. Ví dụ thực tế: một HTTP request đi qua TCP như thế nào

Giả sử bạn gõ `https://example.com` vào trình duyệt:

1. **DNS resolution** (UDP, không phải TCP): trình duyệt tra cứu IP của `example.com`, ví dụ ra `93.184.216.34`
2. **Mở TCP connection**: trình duyệt gọi socket API để `connect()` tới `93.184.216.34:443`
   - Kernel thực hiện 3-way handshake: SYN → SYN-ACK → ACK
   - TCB được tạo với 4-tuple: `(IP_của_bạn, port_tạm, 93.184.216.34, 443)`
3. **TLS handshake** (chạy trên TCP, sau khi TCP đã ESTABLISHED): trao đổi khóa mã hóa
4. **Gửi HTTP request**: trình duyệt gọi `write()`, dữ liệu được TCP chia thành segment, đóng gói IP, đóng gói Ethernet, gửi đi
5. **Server nhận**: các tầng lột header ngược lại, TCP ghép đúng thứ tự, đưa dữ liệu (đã giải mã TLS) lên tầng ứng dụng — web server đọc được HTTP request
6. **Server trả response**: đi ngược lại quy trình, qua cùng connection (cùng 4-tuple nhưng port đảo ngược ở góc nhìn header)
7. **Đóng connection**: sau khi trao đổi xong (hoặc theo cơ chế keep-alive giữ connection để dùng lại cho nhiều request), một bên gửi FIN, bắt đầu 4-way teardown

Toàn bộ quá trình từ bước 2 đến bước 7 diễn ra trên **một TCP connection duy nhất**, được nhận diện xuyên suốt chỉ bằng 4-tuple đó — dù hàng chục hoặc hàng trăm gói tin có thể đã đi qua nhiều router khác nhau ở giữa.

---

## 11. Tổng kết các khái niệm cốt lõi

| Khái niệm                    | Bản chất                                                                                                  |
| ---------------------------- | --------------------------------------------------------------------------------------------------------- |
| **Mô hình TCP/IP**           | 4 tầng: Application (user space) → Transport → Internet → Link (2 tầng dưới cùng chạy trong kernel)       |
| **TCP**                      | Giao thức tầng Transport, cung cấp luồng byte tin cậy, có thứ tự, full-duplex                             |
| **TCP connection**           | Không phải thực thể trên mạng — là hai TCB đồng bộ trạng thái với nhau qua sequence/ack number            |
| **4-tuple**                  | `(source IP, source port, destination IP, destination port)` — khóa định danh duy nhất của một connection |
| **Segment / Packet / Frame** | Tên gọi đơn vị dữ liệu ở tầng Transport / Internet / Link                                                 |
| **Handshake 3 bước**         | SYN → SYN-ACK → ACK, thiết lập số thứ tự ban đầu cho cả hai chiều                                         |
| **Teardown 4 bước**          | FIN/ACK độc lập cho từng chiều, vì TCP là full-duplex                                                     |
| **Sliding window**           | Cho phép gửi nhiều dữ liệu trước khi nhận ACK, tăng thông lượng                                           |
| **Flow control**             | Bên nhận giới hạn tốc độ gửi để tránh tràn buffer của chính nó                                            |
| **Congestion control**       | TCP tự điều chỉnh tốc độ dựa trên tín hiệu gián tiếp (mất gói, độ trễ) để tránh làm tắc nghẽn mạng        |
