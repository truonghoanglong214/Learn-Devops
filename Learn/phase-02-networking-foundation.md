# Phase 2 — Networking Foundation

## Mục tiêu
- Hiểu cách dữ liệu di chuyển giữa các thành phần trong hệ thống.
- Bảo mật và truy cập server từ xa một cách an toàn.
- Chẩn đoán được lỗi mạng cơ bản — kỹ năng troubleshoot cốt lõi của DevOps.

---

## Kiến thức sẽ học

- **Mô hình TCP/IP & OSI (mức thực dụng)**, IP, subnet, port.
- **DNS** (A, CNAME, record, phân giải tên miền) — để gắn domain cho ShopLite sau này.
- **HTTP/HTTPS, status code, header, request/response**.
- **SSH** (key-based auth, config, port forwarding).
- **Firewall** (ufw / iptables, security group).
- **Công cụ chẩn đoán mạng** (ping, curl, dig, netstat/ss, traceroute, tcpdump cơ bản).
- **Khái niệm load balancing & reverse proxy** (lý thuyết, để chuẩn bị Phase 6).

---

## Kiến thức chi tiết

### 1. Mô hình TCP/IP (thực dụng)

#### 1.1 Bốn tầng của TCP/IP

Mô hình TCP/IP có 4 tầng, mỗi tầng đảm nhiệm một vai trò riêng biệt trong quá trình truyền dữ liệu qua mạng:

**Tầng 1 — Link Layer (Tầng liên kết)**

Tầng này xử lý việc truyền dữ liệu giữa các thiết bị trên cùng một mạng vật lý (local network segment). Đây là tầng thấp nhất, làm việc trực tiếp với phần cứng.

- **Ethernet**: Giao thức phổ biến nhất ở tầng này. Ethernet dùng MAC address để xác định thiết bị trong cùng mạng LAN.
- **MAC Address (Media Access Control)**: Địa chỉ 48-bit, gán cứng vào card mạng (NIC) của thiết bị. Ví dụ: `00:1A:2B:3C:4D:5E`. MAC address là duy nhất trên toàn cầu (về lý thuyết), không thể route qua internet nhưng dùng để giao tiếp trong cùng subnet.
- **ARP (Address Resolution Protocol)**: Dùng để tìm MAC address tương ứng với một IP address trong cùng mạng. Khi máy A muốn gửi packet tới 192.168.1.10, nó broadcast ARP request "ai có IP 192.168.1.10?" và máy có IP đó trả lời với MAC address của nó.
- **Frame**: Đơn vị dữ liệu ở tầng Link. Frame bao gồm MAC source, MAC destination, payload và checksum.

**Tầng 2 — Internet Layer (Tầng mạng)**

Tầng này xử lý việc định tuyến (routing) gói tin qua nhiều mạng khác nhau, từ nguồn tới đích, dù cách nhau bao nhiêu hop.

- **IP (Internet Protocol)**: Giao thức cốt lõi. Mỗi gói tin (packet) mang địa chỉ IP nguồn và đích. Routers đọc IP header để quyết định gửi packet tới đâu tiếp theo.
- **ICMP (Internet Control Message Protocol)**: Dùng để gửi thông báo lỗi và kiểm tra kết nối. Lệnh `ping` dùng ICMP Echo Request/Reply. `traceroute` dùng ICMP Time Exceeded.
- **Routing**: Quá trình router quyết định đường đi của packet. Routing table chứa các entry dạng "gửi tới 10.0.0.0/8 thì forward qua interface eth1 tới next-hop 192.168.1.1".
- **Fragmentation**: Nếu packet lớn hơn MTU (Maximum Transmission Unit, thường 1500 bytes), IP layer sẽ chia nhỏ (fragment) và reassemble ở đầu nhận.

**Tầng 3 — Transport Layer (Tầng vận chuyển)**

Tầng này xử lý việc truyền dữ liệu giữa hai ứng dụng cụ thể trên hai máy tính. Port number được dùng để phân biệt các ứng dụng khác nhau trên cùng một máy.

- **TCP (Transmission Control Protocol)**: Connection-oriented, đảm bảo dữ liệu đến đúng thứ tự và không bị mất. Phù hợp khi cần reliability: HTTP, HTTPS, SSH, database.
- **UDP (User Datagram Protocol)**: Connectionless, không đảm bảo delivery. Nhanh hơn TCP vì không có overhead. Phù hợp: DNS, video streaming, gaming.
- **Segment**: Đơn vị dữ liệu của Transport Layer. TCP segment có sequence number, acknowledgement number, flags.
- **Port**: Số 16-bit (0–65535), xác định ứng dụng/service cụ thể. Well-known ports: 0–1023, registered: 1024–49151, dynamic/ephemeral: 49152–65535.

**Tầng 4 — Application Layer (Tầng ứng dụng)**

Tầng cao nhất, nơi các ứng dụng giao tiếp trực tiếp. Protocol ở đây xác định format và ngữ nghĩa của dữ liệu.

- **HTTP/HTTPS**: Web traffic (port 80/443)
- **DNS**: Domain name resolution (port 53, UDP và TCP)
- **SSH**: Secure shell (port 22)
- **SMTP/IMAP/POP3**: Email (port 25, 143, 110)
- **FTP**: File transfer (port 21)

---

#### 1.2 IPv4 — Địa chỉ IP phiên bản 4

IPv4 là phiên bản IP phổ biến nhất hiện nay, dù IPv6 đang dần thay thế.

**Cấu trúc IPv4:**
- 32-bit số nhị phân, chia làm 4 octet (mỗi octet 8 bit)
- Biểu diễn dạng dotted decimal notation: `192.168.1.100`
- Mỗi octet có giá trị từ 0 đến 255
- Ví dụ nhị phân: `11000000.10101000.00000001.01100100` = `192.168.1.100`

**Các phần của địa chỉ IP:**
- **Network portion**: Xác định mạng nào. Tất cả máy trong cùng mạng có cùng network portion.
- **Host portion**: Xác định máy cụ thể trong mạng đó.
- Subnet mask xác định phần nào là network, phần nào là host.

---

#### 1.3 CIDR Notation — Phân chia mạng con

CIDR (Classless Inter-Domain Routing) là cách biểu diễn subnet mask gọn gàng bằng prefix length.

**Cách đọc CIDR:**

```
192.168.1.0/24
           ^^
           prefix length = 24 bit đầu là network, 8 bit sau là host
```

**Bảng tính nhanh số địa chỉ theo prefix:**

| CIDR  | Số địa chỉ    | Số host dùng được | Dùng cho                    |
|-------|---------------|-------------------|-----------------------------|
| /32   | 1             | 1 (host route)    | Loopback, static route      |
| /31   | 2             | 2                 | Point-to-point link         |
| /30   | 4             | 2                 | Small link                  |
| /29   | 8             | 6                 | Tiny subnet                 |
| /28   | 16            | 14                | Very small subnet           |
| /27   | 32            | 30                | Small subnet                |
| /26   | 64            | 62                | Small subnet                |
| /25   | 128           | 126               | Medium subnet               |
| /24   | 256           | 254               | Class C, phổ biến nhất      |
| /23   | 512           | 510               | 2 Class C                   |
| /22   | 1024          | 1022              | 4 Class C                   |
| /20   | 4096          | 4094              | Large subnet                |
| /16   | 65536         | 65534             | Class B                     |
| /8    | 16,777,216    | 16,777,214        | Class A                     |

**Ví dụ tính subnet 192.168.1.0/24:**
- Network address: `192.168.1.0` (bit host = tất cả 0)
- Broadcast address: `192.168.1.255` (bit host = tất cả 1)
- Usable range: `192.168.1.1` tới `192.168.1.254` (254 host)
- Subnet mask: `255.255.255.0`

**Ví dụ 10.0.0.0/16:**
- Network: `10.0.0.0`
- Broadcast: `10.0.255.255`
- Usable: `10.0.0.1` tới `10.0.255.254` (65534 host)

---

#### 1.4 Private IP Ranges — Dải địa chỉ private

RFC 1918 định nghĩa ba dải địa chỉ private, không được route trên internet công cộng:

| Dải               | CIDR         | Số địa chỉ    | Dùng cho thường gặp         |
|-------------------|--------------|---------------|-----------------------------|
| `10.0.0.0`        | `10.0.0.0/8` | ~16.7 triệu   | Cloud VPC, enterprise LAN   |
| `172.16.0.0`      | `172.16.0.0/12` | ~1 triệu   | Docker default bridge       |
| `192.168.0.0`     | `192.168.0.0/16` | 65536     | Home router, office LAN     |

**Lưu ý thực tế:**
- `172.17.0.0/16` là dải Docker bridge mặc định
- AWS VPC thường dùng `10.0.0.0/16` hoặc `172.16.0.0/12`
- Container trong Docker Compose thường nhận IP từ `172.18.0.0/16` hoặc cao hơn
- `127.0.0.0/8` là loopback range, `127.0.0.1` là localhost

---

#### 1.5 Network Address, Broadcast Address, Usable Range

Mỗi subnet có 2 địa chỉ đặc biệt không được gán cho host:

**Network Address (địa chỉ mạng):**
- Địa chỉ đầu tiên trong subnet (host bits = tất cả 0)
- Dùng để xác định subnet, không gán cho host
- Ví dụ: `192.168.10.0/24` -> network address là `192.168.10.0`

**Broadcast Address (địa chỉ quảng bá):**
- Địa chỉ cuối cùng trong subnet (host bits = tất cả 1)
- Packet gửi tới broadcast được nhận bởi TẤT CẢ host trong subnet
- Ví dụ: `192.168.10.0/24` -> broadcast là `192.168.10.255`

**Usable Host Range:**
- Từ network+1 tới broadcast-1
- `192.168.10.0/24`: usable là `192.168.10.1` – `192.168.10.254`
- `10.0.0.0/28`: usable là `10.0.0.1` – `10.0.0.14` (14 host)

---

#### 1.6 NAT — Network Address Translation

NAT cho phép nhiều thiết bị dùng private IP chia sẻ một public IP để kết nối internet.

**Cách hoạt động của NAT:**
1. Máy nội bộ `192.168.1.100:54321` gửi request tới `8.8.8.8:53`
2. Router (NAT gateway) nhận packet, thay source IP:port thành `203.0.113.1:12345` (public IP)
3. Router lưu mapping: `203.0.113.1:12345 <-> 192.168.1.100:54321` vào NAT table
4. Server nhận request từ `203.0.113.1:12345`, trả response về đó
5. Router nhận response, tra NAT table, forward về `192.168.1.100:54321`

**Các loại NAT:**
- **SNAT (Source NAT)**: Thay đổi IP nguồn. Dùng khi outbound traffic từ private network ra internet. Đây là loại phổ biến nhất trong home/office router.
- **DNAT (Destination NAT)**: Thay đổi IP đích. Dùng để forward port từ public IP vào server nội bộ. Còn gọi là port forwarding.
- **Masquerading**: Biến thể của SNAT, dùng khi public IP thay đổi động (ISP dynamic IP).

**NAT trong cloud (AWS):**
- **NAT Gateway**: Cho phép EC2 trong private subnet kết nối internet outbound
- **Internet Gateway**: Cho EC2 trong public subnet truy cập trực tiếp internet
- **Security Group**: Stateful firewall ở mức instance (AWS-specific)
- **Network ACL**: Stateless firewall ở mức subnet (AWS-specific)

**NAT trong Docker:**
- Khi container expose port (ví dụ `-p 80:8080`), Docker dùng iptables DNAT
- Traffic tới `host:80` được NAT tới `container_ip:8080`

---

#### 1.7 Port Numbers — Số cổng

Port là số 16-bit (1–65535) xác định service/ứng dụng cụ thể trên một host.

**Well-known ports quan trọng cho DevOps:**

| Port  | Protocol | Service             | Ghi chú                            |
|-------|----------|---------------------|------------------------------------|
| 22    | TCP      | SSH                 | Remote shell, SCP, SFTP            |
| 53    | TCP/UDP  | DNS                 | UDP thường, TCP cho zone transfer   |
| 80    | TCP      | HTTP                | Web không mã hóa                   |
| 443   | TCP      | HTTPS               | Web mã hóa TLS                     |
| 5000  | TCP      | ASP.NET Core HTTP   | .NET dev server, default HTTP port |
| 5001  | TCP      | ASP.NET Core HTTPS  | .NET dev server, default HTTPS port|
| 3306  | TCP      | MySQL               | Database                           |
| 5432  | TCP      | PostgreSQL          | Database ShopLite dùng             |
| 5672  | TCP      | RabbitMQ            | AMQP message broker                |
| 6379  | TCP      | Redis               | Cache, session store               |
| 8080  | TCP      | HTTP alt            | Alt web port, Tomcat, Jenkins      |
| 8443  | TCP      | HTTPS alt           | Alt HTTPS                          |
| 9090  | TCP      | Prometheus          | Metrics scraping                   |
| 9100  | TCP      | Node Exporter       | System metrics                     |
| 3100  | TCP      | Loki                | Log aggregation                    |

**Ephemeral ports (dynamic ports):**
- Range `49152–65535` (hoặc `32768–60999` trên Linux)
- Client dùng khi kết nối tới server (source port ngẫu nhiên)
- Ví dụ: browser mở tab tới `google.com:443` dùng source port `54321`

---

#### 1.8 TCP — Transmission Control Protocol

TCP là giao thức connection-oriented, đảm bảo:
- **Reliability**: Dữ liệu luôn đến đích (retransmit nếu mất)
- **Ordering**: Dữ liệu đến đúng thứ tự (sequence numbers)
- **Error detection**: Checksum, acknowledgement
- **Flow control**: Ngăn sender gửi quá nhanh
- **Congestion control**: Ngăn quá tải mạng

**TCP 3-Way Handshake — Thiết lập kết nối:**

```
Client                          Server
  |                               |
  |------- SYN (seq=100) -------->|   Client muốn kết nối
  |                               |   Server nhận, ghi nhớ
  |<-- SYN-ACK (seq=200,ack=101) -|   Server đồng ý, gửi seq của mình
  |                               |
  |------- ACK (ack=201) -------->|   Client xác nhận
  |                               |
  |======= DATA TRANSFER =========|   Kết nối được thiết lập
```

- **SYN (Synchronize)**: Client gửi segment với SYN flag, kèm ISN (Initial Sequence Number) ngẫu nhiên
- **SYN-ACK**: Server đồng ý, gửi SYN của mình + ACK xác nhận SYN của client
- **ACK**: Client xác nhận SYN của server. Kết nối thiết lập xong.

**TCP 4-Way Teardown — Đóng kết nối:**

```
Client                          Server
  |------- FIN ------------------>|   Client xong, muốn đóng
  |<------ ACK -------------------|   Server xác nhận
  |<------ FIN -------------------|   Server cũng xong
  |------- ACK ------------------>|   Client xác nhận
  |                               |   Cả hai bên đóng
```

**TCP States quan trọng:**
- `ESTABLISHED`: Kết nối đang active
- `TIME_WAIT`: Đã gửi FIN cuối, chờ đảm bảo đối phương nhận (2 * MSL, thường 60-120 giây)
- `CLOSE_WAIT`: Đã nhận FIN của đối phương, chờ application đóng
- `SYN_RECV`: Nhận SYN, đã gửi SYN-ACK, chờ ACK (dễ bị SYN flood attack)
- `LISTEN`: Server đang chờ kết nối

**Quan sát TCP states:**
```bash
ss -tn state established
ss -tn state time-wait
ss -tn | grep SYN_RECV
```

---

#### 1.9 UDP — User Datagram Protocol

UDP là giao thức connectionless, không có:
- Handshake (không thiết lập kết nối)
- Acknowledgement (không xác nhận nhận được)
- Ordering (không đảm bảo thứ tự)
- Retransmission (không gửi lại nếu mất)

**Ưu điểm UDP:**
- Latency thấp hơn TCP (không overhead của handshake, ACK)
- Phù hợp khi mất một ít packet không sao (video call, gaming)
- Phù hợp khi application tự xử lý reliability (DNS có timeout retry)

**Dùng UDP khi:**
- DNS lookup (query nhỏ, nếu mất thì retry ngay)
- Video/audio streaming (trễ hơn là tệ hơn mất frame)
- Online gaming (latency quan trọng hơn reliability)
- NTP (Network Time Protocol)
- DHCP (Dynamic Host Configuration Protocol)

---

### 2. DNS Chi tiết

#### 2.1 Cách hoạt động của DNS

Khi bạn gõ `api.shoplite.com` vào browser, quá trình phân giải tên miền diễn ra như sau:

**Bước 1 — Kiểm tra local cache:**
Browser kiểm tra DNS cache của chính nó. Nếu đã từng query `api.shoplite.com` và TTL chưa hết, dùng kết quả cache.

**Bước 2 — Kiểm tra /etc/hosts:**
OS đọc file `/etc/hosts`. Nếu có entry cho hostname, dùng IP đó luôn, bỏ qua DNS hoàn toàn.

```
# /etc/hosts
127.0.0.1       localhost
192.168.1.100   shoplite-dev.local
10.0.1.5        db.shoplite.internal
```

**Bước 3 — Kiểm tra /etc/resolv.conf:**
File này chỉ định DNS server nào sẽ được hỏi.

```
# /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
search shoplite.internal
```

**Bước 4 — Query Recursive Resolver:**
OS gửi query tới DNS server được cấu hình (ví dụ `8.8.8.8` — Google DNS). Đây là **recursive resolver**, nó sẽ làm tất cả công việc thay cho client.

**Bước 5 — Recursive Query Process:**

```
Recursive Resolver (8.8.8.8)
    |
    |-- Query Root Nameserver: "ai quản lý .com?"
    |   Root NS trả lời: "hỏi a.gtld-servers.net"
    |
    |-- Query TLD Nameserver (a.gtld-servers.net): "ai quản lý shoplite.com?"
    |   TLD NS trả lời: "hỏi ns1.cloudflare.com"
    |
    |-- Query Authoritative NS (ns1.cloudflare.com): "api.shoplite.com là gì?"
        Authoritative NS trả lời: "api.shoplite.com -> 1.2.3.4, TTL=300"
```

**Bước 6 — Trả kết quả về client:**
Recursive resolver cache kết quả và trả về cho client. Client dùng IP `1.2.3.4` để kết nối.

---

#### 2.2 DNS Record Types

**A Record — IPv4 Address:**
```
api.shoplite.com.     300    IN    A    1.2.3.4
www.shoplite.com.     300    IN    A    1.2.3.4
shoplite.com.         300    IN    A    1.2.3.4
```
Dùng để map hostname tới IPv4. Đây là record phổ biến nhất.

**AAAA Record — IPv6 Address:**
```
api.shoplite.com.     300    IN    AAAA    2001:db8::1
```
Tương tự A record nhưng cho IPv6. Tên "AAAA" vì IPv6 gấp 4 lần độ dài IPv4.

**CNAME Record — Canonical Name (Alias):**
```
www.shoplite.com.     300    IN    CNAME    shoplite.com.
cdn.shoplite.com.     300    IN    CNAME    d1234abcd.cloudfront.net.
```
- Map một tên tới một tên khác (alias)
- Resolver sẽ tiếp tục phân giải tên đích cho tới khi ra A record
- **Không được dùng CNAME cho apex domain** (root domain như `shoplite.com`), vì nó xung đột với NS và MX records
- Cloudflare "CNAME flattening" là workaround cho vấn đề này

**MX Record — Mail Exchange:**
```
shoplite.com.     300    IN    MX    10    mail1.shoplite.com.
shoplite.com.     300    IN    MX    20    mail2.shoplite.com.
```
Xác định mail server nhận email cho domain. Số ưu tiên nhỏ hơn = ưu tiên hơn.

**TXT Record — Text:**
```
shoplite.com.     300    IN    TXT    "v=spf1 include:sendgrid.net ~all"
shoplite.com.     300    IN    TXT    "google-site-verification=abc123"
_dmarc.shoplite.com. 300 IN   TXT    "v=DMARC1; p=quarantine"
```
Dùng cho:
- **SPF**: Chỉ định server nào được phép gửi email từ domain
- **DKIM**: Xác thực chữ ký email
- **DMARC**: Policy xử lý email fail SPF/DKIM
- Domain ownership verification (Google, AWS, etc.)
- Let's Encrypt DNS-01 challenge

**NS Record — Nameserver:**
```
shoplite.com.     86400    IN    NS    ns1.cloudflare.com.
shoplite.com.     86400    IN    NS    ns2.cloudflare.com.
```
Xác định nameserver nào có thẩm quyền (authoritative) cho domain. Được set khi đăng ký domain.

**PTR Record — Pointer (Reverse DNS):**
```
4.3.2.1.in-addr.arpa.    300    IN    PTR    api.shoplite.com.
```
Map từ IP về hostname (ngược với A record). Dùng cho:
- Email spam filtering (mail server thường check PTR)
- Log analysis
- Xác minh server identity

---

#### 2.3 TTL — Time To Live

TTL (tính bằng giây) xác định bao lâu record được cache bởi resolver.

**TTL thấp vs TTL cao:**

| TTL    | Ưu điểm                           | Nhược điểm                         |
|--------|-----------------------------------|------------------------------------|
| Thấp (60–300s) | Thay đổi DNS có hiệu lực nhanh | Nhiều query hơn, tải DNS server cao |
| Cao (3600–86400s) | Ít query, resolver cache tốt | Thay đổi DNS lan chậm             |

**Quy trình thay đổi DNS record an toàn:**
1. **1 tuần trước**: Giảm TTL về 60 giây (để cache cũ hết nhanh)
2. **Ngày thay đổi**: Cập nhật IP trong record
3. **Chờ 1-2 phút**: TTL 60s đảm bảo hầu hết resolver đã nhận record mới
4. **Xác nhận**: Dùng `dig` từ nhiều server để kiểm tra propagation
5. **Sau khi ổn định**: Tăng TTL về giá trị cao hơn

**Kiểm tra DNS propagation:**
```bash
# Hỏi nhiều DNS server khác nhau
dig @8.8.8.8 api.shoplite.com A
dig @1.1.1.1 api.shoplite.com A
dig @208.67.222.222 api.shoplite.com A   # OpenDNS
```

---

#### 2.4 /etc/hosts và /etc/resolv.conf

**File /etc/hosts:**
```
# Syntax: IP   hostname   [alias...]
127.0.0.1       localhost
127.0.1.1       my-server my-server.local
192.168.1.100   shoplite-dev.local shoplite-dev
10.0.1.5        postgres.shoplite.internal postgres
10.0.1.6        redis.shoplite.internal redis
```

Dùng cho:
- Local development (không cần setup DNS server)
- Override DNS (debug, test redirect traffic)
- Container-to-container với tên thay vì IP (khi không dùng Docker network)
- Block websites (0.0.0.0 ads.tracking.com)

**File /etc/resolv.conf:**
```
# Primary DNS
nameserver 8.8.8.8
# Secondary DNS (fallback)
nameserver 8.8.4.4
# Domain search list (tự thêm suffix khi query short name)
search shoplite.internal local
# Options
options ndots:5 timeout:2 attempts:3
```

Trên hệ thống dùng `systemd-resolved`, file này thường là symlink tới `/run/systemd/resolve/stub-resolv.conf`. Để chỉnh, dùng `resolvectl` hoặc sửa `/etc/systemd/resolved.conf`.

---

#### 2.5 Lệnh dig — DNS Interrogation

`dig` (Domain Information Groper) là công cụ query DNS mạnh nhất, output chi tiết hơn `nslookup`.

**Các lệnh dig cơ bản:**
```bash
# Query A record mặc định
dig google.com

# Query type record cụ thể
dig google.com A
dig google.com AAAA
dig google.com MX
dig google.com TXT
dig google.com NS
dig google.com CNAME

# Hỏi DNS server cụ thể
dig @8.8.8.8 google.com A
dig @1.1.1.1 api.shoplite.com A

# Xem toàn bộ quá trình phân giải (full trace)
dig +trace google.com

# Reverse lookup (IP -> hostname)
dig -x 8.8.8.8
dig -x 142.250.185.46

# Output ngắn gọn (chỉ answer section)
dig +short google.com
dig +short google.com MX

# Hiện tất cả record
dig google.com ANY

# Query không recursive (chỉ hỏi authoritative)
dig +norecurse @ns1.google.com google.com

# Set query timeout
dig +time=5 +tries=2 google.com
```

**Đọc output của dig:**
```
; <<>> DiG 9.18.1 <<>> google.com
;; QUESTION SECTION:
;google.com.                    IN      A         <- Query gì

;; ANSWER SECTION:
google.com.             213     IN      A       142.250.185.46   <- Kết quả, TTL còn 213s

;; AUTHORITY SECTION:
google.com.             53      IN      NS      ns1.google.com.  <- Authoritative NS

;; ADDITIONAL SECTION:
ns1.google.com.         258985  IN      A       216.239.32.10    <- Thêm info

;; Query time: 23 msec       <- Thời gian query
;; SERVER: 8.8.8.8#53        <- DNS server đã hỏi
;; WHEN: ...                 <- Thời điểm query
;; MSG SIZE  rcvd: 55        <- Kích thước response
```

---

### 3. HTTP/HTTPS Chi tiết

#### 3.1 Cấu trúc HTTP Request

```
GET /api/products?category=electronics&page=1 HTTP/1.1
Host: api.shoplite.com
Accept: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
User-Agent: Mozilla/5.0 (compatible; curl/7.88)
Cache-Control: no-cache
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
Connection: keep-alive

[request body - trống với GET]
```

**Các thành phần:**
- **Request line**: Method + URL path + HTTP version
- **Headers**: Key-value pairs, mỗi cặp một dòng
- **Blank line**: Dòng trống ngăn cách header và body
- **Body**: Dữ liệu gửi kèm (POST, PUT, PATCH)

#### 3.2 HTTP Methods

| Method  | Mục đích             | Idempotent | Body | Cached |
|---------|----------------------|------------|------|--------|
| GET     | Lấy dữ liệu         | Có         | Không | Có    |
| POST    | Tạo mới resource    | Không      | Có   | Không  |
| PUT     | Thay thế resource   | Có         | Có   | Không  |
| PATCH   | Cập nhật một phần   | Không      | Có   | Không  |
| DELETE  | Xóa resource        | Có         | Không | Không |
| OPTIONS | CORS preflight      | Có         | Không | Có    |
| HEAD    | Như GET nhưng không có body response | Có | Không | Có |

**Idempotent**: Gọi nhiều lần kết quả giống gọi một lần. PUT thay thế toàn bộ resource nên gọi nhiều lần kết quả như nhau. POST tạo mới mỗi lần.

**CORS Preflight (OPTIONS):**
Browser gửi OPTIONS request trước khi gửi cross-origin request thực sự để hỏi server: "Tôi có thể gửi POST với header Authorization từ domain A tới domain B không?"

---

#### 3.3 HTTP Status Codes

**2xx — Success:**

| Code | Name              | Dùng khi                                    |
|------|-------------------|---------------------------------------------|
| 200  | OK                | Request thành công, có response body        |
| 201  | Created           | POST tạo resource mới thành công            |
| 202  | Accepted          | Request nhận nhưng xử lý bất đồng bộ       |
| 204  | No Content        | Thành công nhưng không có body (DELETE)     |

**3xx — Redirection:**

| Code | Name              | Dùng khi                                    |
|------|-------------------|---------------------------------------------|
| 301  | Moved Permanently | URL đổi vĩnh viễn, browser cache            |
| 302  | Found             | Redirect tạm thời                           |
| 304  | Not Modified      | Cache còn valid, không gửi body lại         |
| 307  | Temporary Redirect | Redirect tạm, giữ nguyên method            |
| 308  | Permanent Redirect | Redirect vĩnh viễn, giữ nguyên method      |

**4xx — Client Error:**

| Code | Name                  | Dùng khi                                |
|------|-----------------------|-----------------------------------------|
| 400  | Bad Request           | Request malformed, validation fail      |
| 401  | Unauthorized          | Chưa authenticate (cần login)           |
| 403  | Forbidden             | Đã authenticate nhưng không có quyền   |
| 404  | Not Found             | Resource không tồn tại                  |
| 405  | Method Not Allowed    | Method không được phép với endpoint     |
| 409  | Conflict              | Conflict với state hiện tại (duplicate) |
| 422  | Unprocessable Entity  | Validation error (dùng nhiều trong API) |
| 429  | Too Many Requests     | Rate limit bị vượt                      |

**5xx — Server Error:**

| Code | Name                  | Dùng khi                                |
|------|-----------------------|-----------------------------------------|
| 500  | Internal Server Error | Lỗi không xác định trong server        |
| 502  | Bad Gateway           | Upstream server trả lỗi (Nginx->App)   |
| 503  | Service Unavailable   | Server quá tải hoặc maintenance        |
| 504  | Gateway Timeout       | Upstream server không trả lời kịp      |

**Phân biệt 401 vs 403:**
- `401`: "Bạn là ai? Tôi không biết bạn." — Cần gửi credentials
- `403`: "Tôi biết bạn là ai, nhưng bạn không được phép." — Không có quyền

**Phân biệt 502 vs 504:**
- `502`: Nginx nhận được response từ upstream nhưng response là lỗi/invalid
- `504`: Nginx không nhận được response trong thời gian timeout

---

#### 3.4 HTTP Headers Quan trọng

**Request Headers:**
```
# Định dạng data gửi lên
Content-Type: application/json
Content-Type: multipart/form-data; boundary=----FormBoundary7MA4YWxk
Content-Type: application/x-www-form-urlencoded

# Authenticate
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjF9.abc123
Authorization: Basic dXNlcjpwYXNz   # Base64(user:pass)
Authorization: AWS4-HMAC-SHA256 ...  # AWS Signature

# Cache
Cache-Control: no-cache              # Luôn phải validate với server
Cache-Control: no-store              # Không cache gì cả
If-Modified-Since: Wed, 21 Oct 2023 07:28:00 GMT
If-None-Match: "abc123"              # ETag

# CORS
Origin: https://app.shoplite.com    # Trình duyệt tự gửi khi cross-origin

# Tracing
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
X-Correlation-ID: txn-12345

# Forwarding (khi qua reverse proxy)
X-Forwarded-For: 1.2.3.4, 10.0.0.1
X-Forwarded-Proto: https
X-Real-IP: 1.2.3.4
```

**Response Headers:**
```
# Định dạng response
Content-Type: application/json; charset=utf-8
Content-Length: 1234

# CORS
Access-Control-Allow-Origin: https://app.shoplite.com
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400

# Cache
Cache-Control: public, max-age=3600
ETag: "abc123"
Last-Modified: Wed, 21 Oct 2023 07:28:00 GMT

# Security
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'

# Rate limiting
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1698920400
```

---

#### 3.5 HTTPS và TLS

HTTPS = HTTP + TLS (Transport Layer Security). Mọi HTTP traffic đều được mã hóa trong tunnel TLS.

**TLS Handshake (TLS 1.3 simplified):**
```
Client                                Server
  |                                     |
  |-- ClientHello (supported versions, ciphers, random) -->|
  |                                     |
  |<-- ServerHello (chosen version, cipher, random) -------|
  |<-- Certificate (server's TLS cert) --------------------|
  |<-- CertificateVerify (signed with private key) --------|
  |<-- Finished -------------------------------------------|
  |                                     |
  |-- Finished (client verify cert, generate session key) ->|
  |                                     |
  |======== Encrypted Application Data ====================>|
```

**Certificate Chain:**
```
Root CA (VeriSign, Let's Encrypt, DigiCert)
    |-- signs -->
    Intermediate CA (Let's Encrypt Authority X3)
        |-- signs -->
        Leaf Certificate (api.shoplite.com)
            - Subject: api.shoplite.com
            - Valid from: 2024-01-01
            - Valid until: 2024-04-01
            - Public key: ...
            - Signed by: Let's Encrypt Authority X3
```

Browser tin tưởng Root CA (built-in trong OS/browser). Root CA ký Intermediate CA. Intermediate CA ký certificate của website. Browser verify chain này.

**Let's Encrypt với Certbot:**
```bash
# Cài certbot
apt install certbot python3-certbot-nginx

# Cấp certificate cho domain (Nginx)
certbot --nginx -d shoplite.com -d www.shoplite.com -d api.shoplite.com

# Renew thủ công
certbot renew

# Test renew (dry run)
certbot renew --dry-run

# Xem certificate
certbot certificates
```

---

#### 3.6 curl — HTTP Client từ Command Line

```bash
# GET đơn giản
curl https://api.shoplite.com/health

# Verbose output (xem full request/response headers)
curl -v https://api.shoplite.com/health

# Chỉ xem response headers
curl -I https://api.shoplite.com/health

# POST với JSON body
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"Laptop","price":999.99,"category":"electronics"}' \
  https://api.shoplite.com/products

# POST với file JSON
curl -X POST \
  -H "Content-Type: application/json" \
  -d @product.json \
  https://api.shoplite.com/products

# Với Bearer token
curl -H "Authorization: Bearer $TOKEN" \
  https://api.shoplite.com/orders

# Follow redirects
curl -L https://bit.ly/something

# Follow redirects và lưu output
curl -L -o output.html https://example.com

# Bỏ qua SSL certificate error (DEV ONLY - không bao giờ dùng production)
curl -k https://self-signed.example.com

# Timeout 10 giây
curl --connect-timeout 10 --max-time 30 https://api.shoplite.com/health

# Specify DNS resolution thủ công (không cần DNS propagation)
curl --resolve api.shoplite.com:443:1.2.3.4 https://api.shoplite.com/health

# Upload file
curl -X POST \
  -F "image=@product.jpg" \
  -F "name=Laptop" \
  https://api.shoplite.com/products/upload

# PUT request
curl -X PUT \
  -H "Content-Type: application/json" \
  -d '{"price":899.99}' \
  https://api.shoplite.com/products/123

# DELETE request
curl -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  https://api.shoplite.com/products/123

# Xem timing chi tiết
curl -w "\n\nDNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
  -o /dev/null -s https://api.shoplite.com/health

# Gửi qua proxy
curl -x http://proxy:3128 https://api.shoplite.com/health
```

---

### 4. SSH Chi tiết

#### 4.1 Cơ chế xác thực bằng key pair

SSH key-based authentication an toàn hơn password vì:
- Private key không bao giờ rời khỏi máy client
- Không thể brute force (key quá dài)
- Không bị phishing (không có password để lừa)
- Có thể revoke key cụ thể mà không ảnh hưởng key khác

**Quá trình xác thực:**
1. Client gửi public key ID muốn dùng
2. Server kiểm tra `~/.ssh/authorized_keys`, tìm matching key
3. Server tạo random challenge, mã hóa bằng public key, gửi về client
4. Client giải mã bằng private key, gửi lại proof
5. Server verify proof, cho phép đăng nhập

---

#### 4.2 Tạo và quản lý SSH key

```bash
# Tạo ED25519 key (recommended)
ssh-keygen -t ed25519 -C "your@email.com" -f ~/.ssh/shoplite_key
# -t: algorithm
# -C: comment (label để nhận biết key)
# -f: output file

# Tạo RSA key 4096 bit (nếu cần tương thích với hệ thống cũ)
ssh-keygen -t rsa -b 4096 -C "your@email.com" -f ~/.ssh/shoplite_rsa

# Xem public key
cat ~/.ssh/shoplite_key.pub
# Output: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... your@email.com

# Set đúng permissions
chmod 700 ~/.ssh                          # Thư mục .ssh: chỉ owner đọc/ghi/execute
chmod 600 ~/.ssh/shoplite_key             # Private key: chỉ owner đọc/ghi
chmod 644 ~/.ssh/shoplite_key.pub         # Public key: owner rw, others r
chmod 600 ~/.ssh/authorized_keys          # authorized_keys: chỉ owner rw
chmod 600 ~/.ssh/config                   # config file: chỉ owner rw

# Nếu permissions sai, SSH sẽ từ chối dùng key và báo lỗi:
# "WARNING: UNPROTECTED PRIVATE KEY FILE!"
```

**ED25519 vs RSA:**
- **ED25519**: Curve25519 elliptic curve, key nhỏ (256-bit), nhanh hơn, an toàn hơn, recommended cho 2024+
- **RSA**: Key lớn hơn (2048-4096 bit), chậm hơn, tương thích rộng hơn với hệ thống cũ

---

#### 4.3 Copy public key lên server

```bash
# Cách 1: ssh-copy-id (tự động, recommended)
ssh-copy-id -i ~/.ssh/shoplite_key.pub ubuntu@1.2.3.4
ssh-copy-id -i ~/.ssh/shoplite_key.pub -p 2222 ubuntu@1.2.3.4  # port khác

# Cách 2: Thủ công
# Trên máy local:
cat ~/.ssh/shoplite_key.pub
# Copy output

# Trên server:
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... your@email.com" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Cách 3: Qua pipe (nếu đã có password access)
cat ~/.ssh/shoplite_key.pub | ssh ubuntu@1.2.3.4 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

---

#### 4.4 Cấu hình sshd (server-side)

File: `/etc/ssh/sshd_config`

```bash
# Tắt password login (bắt buộc sau khi đã setup key)
PasswordAuthentication no
PubkeyAuthentication yes

# Tắt root login
PermitRootLogin no

# Chỉ cho phép user cụ thể
AllowUsers ubuntu deploy

# Đổi port SSH (giảm noise từ bot scan)
Port 2222

# Giới hạn authentication attempts
MaxAuthTries 3

# Timeout idle connection
ClientAliveInterval 300
ClientAliveCountMax 2

# Tắt X11 forwarding (không cần với server)
X11Forwarding no

# Tắt TCP forwarding (nếu không cần)
AllowTcpForwarding yes   # Để yes nếu cần SSH tunneling

# Banner
Banner /etc/ssh/banner.txt
```

```bash
# Restart sshd sau khi sửa config
systemctl restart sshd

# Kiểm tra config syntax trước khi restart (tránh bị lock out)
sshd -t

# Check trạng thái
systemctl status sshd
```

**QUAN TRỌNG:** Luôn giữ một session SSH cũ mở khi sửa sshd_config. Nếu sửa sai và restart, session cũ vẫn còn để khắc phục.

---

#### 4.5 ~/.ssh/config — File cấu hình SSH client

File này giúp tạo alias và lưu cấu hình cho từng server, tránh phải nhớ IP và options mỗi lần.

```
# ~/.ssh/config

# Production server
Host shoplite-prod
    HostName 1.2.3.4
    User ubuntu
    IdentityFile ~/.ssh/shoplite_key
    Port 22
    ServerAliveInterval 60
    ServerAliveCountMax 3
    Compression yes

# Staging server
Host shoplite-staging
    HostName 5.6.7.8
    User ubuntu
    IdentityFile ~/.ssh/shoplite_key

# Bastion host (jump server)
Host bastion
    HostName bastion.shoplite.com
    User ubuntu
    IdentityFile ~/.ssh/shoplite_key
    ServerAliveInterval 30

# Private server qua bastion (ProxyJump)
Host shoplite-db
    HostName 10.0.1.5
    User ubuntu
    IdentityFile ~/.ssh/shoplite_key
    ProxyJump bastion
    # Hoặc: ProxyCommand ssh -W %h:%p bastion

# Development server với port forwarding tự động
Host shoplite-dev
    HostName 192.168.1.100
    User developer
    IdentityFile ~/.ssh/dev_key
    LocalForward 15432 localhost:5432    # Tự động forward PostgreSQL
    LocalForward 16379 localhost:6379    # Tự động forward Redis

# Wildcard cho tất cả server trong VPC
Host 10.0.*
    User ubuntu
    IdentityFile ~/.ssh/shoplite_key
    ProxyJump bastion
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```

**Sau khi có config:**
```bash
# Thay vì: ssh -i ~/.ssh/shoplite_key ubuntu@1.2.3.4
ssh shoplite-prod

# Thay vì: ssh -i key -J ubuntu@bastion ubuntu@10.0.1.5
ssh shoplite-db
```

---

#### 4.6 SSH Tunneling — Port Forwarding

**Local Port Forwarding (truy cập dịch vụ từ xa qua SSH):**
```bash
# Forward port local:5432 -> rds-endpoint:5432 qua bastion
# Sau đó kết nối database: psql -h localhost -p 5432
ssh -L 5432:rds-endpoint.rds.amazonaws.com:5432 ubuntu@bastion

# Forward port local:15432 (tránh conflict với local postgres)
ssh -L 15432:rds-endpoint:5432 ubuntu@bastion

# Multiple forward trong một command
ssh -L 5432:db:5432 -L 6379:redis:6379 ubuntu@bastion

# Forward với -N (no command) và -f (background)
ssh -N -f -L 5432:rds-endpoint:5432 ubuntu@bastion
```

**Remote Port Forwarding (expose local port ra internet):**
```bash
# Expose localhost:3000 thành server:8080
# Người ngoài có thể truy cập server:8080 sẽ tới localhost:3000
ssh -R 8080:localhost:3000 ubuntu@server

# Dùng cho: demo local dev tới client, webhook testing
```

**Dynamic Port Forwarding (SOCKS Proxy):**
```bash
# Tạo SOCKS5 proxy tại local:1080
ssh -D 1080 ubuntu@server

# Dùng với curl (qua proxy)
curl --socks5 localhost:1080 https://internal-service.shoplite.com

# Dùng với browser (cấu hình proxy trong Firefox/Chrome)
```

---

#### 4.7 Chuyển file qua SSH

```bash
# scp — secure copy
# Upload file lên server
scp -i ~/.ssh/shoplite_key config.env ubuntu@1.2.3.4:/home/ubuntu/app/

# Download file từ server
scp -i ~/.ssh/shoplite_key ubuntu@1.2.3.4:/home/ubuntu/logs/app.log ./

# Copy thư mục (recursive)
scp -r -i ~/.ssh/shoplite_key ./dist ubuntu@1.2.3.4:/var/www/shoplite/

# rsync — sync thư mục, hiệu quả hơn scp
# Chỉ gửi file thay đổi, hỗ trợ resume

# Upload thư mục, bỏ node_modules
rsync -avz --progress \
  --exclude=node_modules \
  --exclude=.git \
  --exclude=.env \
  -e "ssh -i ~/.ssh/shoplite_key" \
  ./app ubuntu@1.2.3.4:/home/ubuntu/

# Download logs
rsync -avz \
  -e "ssh -i ~/.ssh/shoplite_key" \
  ubuntu@1.2.3.4:/var/log/shoplite/ \
  ./logs/

# Dry run (xem sẽ làm gì mà không thực sự copy)
rsync -avzn --exclude=node_modules . ubuntu@server:/app/
```

---

#### 4.8 ssh-agent — Quản lý key trong memory

ssh-agent giữ private key trong memory, không cần nhập passphrase mỗi lần kết nối.

```bash
# Khởi động ssh-agent
eval $(ssh-agent -s)

# Thêm key vào agent
ssh-add ~/.ssh/shoplite_key

# Thêm key với thời hạn (3600 giây = 1 giờ)
ssh-add -t 3600 ~/.ssh/shoplite_key

# Liệt kê key đang có trong agent
ssh-add -l

# Xóa key khỏi agent
ssh-add -d ~/.ssh/shoplite_key

# Xóa tất cả key
ssh-add -D

# Thêm vào ~/.bashrc hoặc ~/.zshrc để tự động:
# eval $(ssh-agent -s)
# ssh-add ~/.ssh/shoplite_key 2>/dev/null
```

**SSH Agent Forwarding (cẩn thận khi dùng):**
```bash
# Cho phép server bạn SSH tới dùng key của bạn để SSH tiếp sang server khác
ssh -A ubuntu@bastion    # ForwardAgent yes trong ~/.ssh/config

# RỦI RO: Nếu root trên bastion bị compromise, họ có thể dùng key của bạn
# Thay vào đó, dùng ProxyJump (an toàn hơn)
```

---

### 5. Firewall với UFW

#### 5.1 UFW Cơ bản

UFW (Uncomplicated Firewall) là frontend của iptables, dễ dùng hơn nhiều.

```bash
# Kiểm tra trạng thái
ufw status
ufw status verbose
ufw status numbered    # Xem với số thứ tự để xóa

# Bật/tắt UFW
ufw enable             # CẢNH BÁO: Đảm bảo rule SSH đã có trước khi enable
ufw disable

# Reset về mặc định (xóa tất cả rules)
ufw reset
```

**Default policy — Chính sách mặc định:**
```bash
# Từ chối tất cả traffic vào (an toàn, chỉ mở những gì cần)
ufw default deny incoming

# Cho phép tất cả traffic ra
ufw default allow outgoing

# Từ chối tất cả traffic vào (policy nghiêm ngặt hơn)
ufw default deny forward
```

---

#### 5.2 Thêm và xóa rules

```bash
# Cho phép SSH (PHẢI làm trước khi enable UFW)
ufw allow 22/tcp
ufw allow ssh           # Tương đương, dùng service name

# Cho phép SSH CHỈ từ IP cụ thể (an toàn hơn)
ufw allow from 203.0.113.10 to any port 22
ufw allow from 203.0.113.0/24 to any port 22   # Cả subnet

# Cho phép HTTP và HTTPS
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow http           # Tương đương 80
ufw allow https          # Tương đương 443

# Block port PostgreSQL từ bên ngoài
ufw deny 5432
ufw deny 6379            # Block Redis

# Rate limiting SSH (giảm brute force)
ufw limit 22/tcp
# Tự động block IP nếu kết nối quá 6 lần trong 30 giây

# Allow range port
ufw allow 8000:8080/tcp

# Allow cả TCP và UDP
ufw allow 53

# Xóa rule bằng số (xem từ ufw status numbered)
ufw delete 3

# Xóa rule bằng specification
ufw delete allow 80/tcp
ufw delete allow from 203.0.113.10 to any port 22
```

---

#### 5.3 Cấu hình UFW cho ShopLite

Cấu hình production security baseline:

```bash
# Reset và thiết lập từ đầu
ufw reset

# Chính sách mặc định
ufw default deny incoming
ufw default allow outgoing

# SSH — chỉ từ IP văn phòng và IP cá nhân của team
ufw allow from 203.0.113.10 to any port 22     # IP admin
ufw allow from 203.0.113.20 to any port 22     # IP dev

# Web traffic — public
ufw allow 80/tcp
ufw allow 443/tcp

# Monitoring từ trong VPC (không public)
# ufw allow from 10.0.0.0/8 to any port 9090   # Prometheus
# ufw allow from 10.0.0.0/8 to any port 3100   # Loki

# Enable
ufw enable

# Verify
ufw status verbose
```

---

#### 5.4 iptables cơ bản

UFW là wrapper cho iptables. Biết cơ bản iptables giúp debug:

```bash
# Xem tất cả rules
iptables -L -n -v                    # -n: numeric, -v: verbose
iptables -L -n -v --line-numbers     # Kèm số thứ tự

# Xem NAT rules (Docker, port forwarding)
iptables -t nat -L -n -v

# Thêm rule (manual, nhưng UFW tốt hơn)
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -j DROP

# Xem chains
iptables -L INPUT -n -v
iptables -L OUTPUT -n -v
iptables -L FORWARD -n -v
```

**Docker và iptables:**
Docker tự quản lý iptables rules cho container networking. Khi bạn expose port (`-p 80:8080`), Docker thêm DNAT rule vào chain `DOCKER` và `PREROUTING`. UFW rules không block Docker published ports theo mặc định — đây là behavior quan trọng cần biết.

---

### 6. Network Troubleshooting Toolkit

#### 6.1 Kiểm tra kết nối cơ bản

```bash
# ping — test ICMP connectivity
ping -c 4 google.com             # Gửi 4 packet
ping -c 4 192.168.1.1            # Test gateway
ping -c 4 8.8.8.8                # Test internet (không cần DNS)
ping -i 0.5 -c 10 server         # Interval 0.5s, 10 packet
ping -f server                   # Flood ping (test network speed)

# traceroute — xem đường đi packet
traceroute google.com
traceroute -n google.com         # No DNS lookup (nhanh hơn)
traceroute -T google.com         # Dùng TCP thay UDP (bypass firewall)

# mtr — kết hợp traceroute + ping, realtime stats
mtr google.com
mtr -n google.com                # No DNS
mtr --report google.com          # Output dạng report (không interactive)
mtr -r -c 100 google.com         # 100 packets, report mode

# Đọc mtr output:
# Column "Loss%": % packet bị mất tại hop đó
# Column "Avg": average RTT
# "???" hoặc loss cao ở hop giữa thường do firewall block ICMP, không phải thực sự mất
```

---

#### 6.2 Kiểm tra ports và listening services

```bash
# ss — Socket Statistics (thay netstat)
ss -tulpn                        # TCP+UDP Listening ports + Process
# -t: TCP
# -u: UDP
# -l: Listening only
# -p: Show process
# -n: Numeric (không resolve port/hostname)

# Ví dụ output:
# Netid  State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
# tcp    LISTEN  0       128     0.0.0.0:22           0.0.0.0:*         users:(("sshd",pid=1234))
# tcp    LISTEN  0       511     0.0.0.0:80           0.0.0.0:*         users:(("nginx",pid=5678))

# Xem established connections
ss -tp                           # TCP với process info
ss -tn state established         # Chỉ established
ss -tn | grep ESTABLISHED

# Xem connection tới/từ IP cụ thể
ss -tn dst 1.2.3.4
ss -tn src 192.168.1.100

# lsof — list open files (kể cả network sockets)
lsof -i :3000                    # Process dùng port 3000
lsof -i :5432                    # Process dùng PostgreSQL port
lsof -i TCP:80 -i TCP:443        # Multiple ports
lsof -i TCP -sTCP:LISTEN         # Tất cả TCP listening

# Tìm process đang chiếm port
lsof -i :3000 | grep LISTEN
fuser 3000/tcp                   # Kill-friendly: fuser -k 3000/tcp
```

---

#### 6.3 Bắt gói tin với tcpdump

```bash
# Capture HTTP traffic trên eth0
tcpdump -i eth0 port 80

# Capture tất cả traffic từ/tới IP
tcpdump -i eth0 host 1.2.3.4

# Capture DNS queries
tcpdump -i eth0 port 53

# Lưu vào file để phân tích với Wireshark
tcpdump -i eth0 port 80 -w capture.pcap

# Đọc từ file
tcpdump -r capture.pcap

# Verbose + không resolve hostname
tcpdump -i eth0 -nn port 80

# Capture với filter phức tạp
tcpdump -i eth0 'tcp port 80 and (host 1.2.3.4 or host 5.6.7.8)'

# Capture với size limit
tcpdump -i eth0 -C 100 -W 5 -w capture.pcap  # Max 5 file, 100MB mỗi file

# Interface list
tcpdump -D
ip link show
```

---

#### 6.4 Quy trình debug "Service không kết nối được"

Khi nhận được ticket "API không response", "không connect được database", đây là quy trình có hệ thống:

**Step 1: Kiểm tra connectivity (Ping)**
```bash
ping -c 4 <service_ip>
# Nếu không ping được: network issue, instance down, hoặc ICMP bị block
# Tiếp tục dù ping fail vì ICMP có thể bị firewall block
```

**Step 2: Kiểm tra port có mở không**
```bash
# Telnet test (nếu có)
telnet <service_ip> <port>

# nc (netcat) — tốt hơn telnet
nc -zv <service_ip> <port>
nc -zv api.shoplite.com 443
# -z: zero-I/O mode (chỉ check port)
# -v: verbose

# Nếu "Connection refused": port không có gì listen
# Nếu "Connection timed out": firewall block, host down
# Nếu "Connected": port mở
```

**Step 3: Service có đang listen không**
```bash
# Trên server đó
ss -tulpn | grep <port>
lsof -i :<port>
# Nếu không thấy gì: service chưa start hoặc start trên interface sai
```

**Step 4: Test HTTP response**
```bash
curl -v http://<ip>:<port>/health
curl -v https://<domain>/health
# Xem full request/response headers
# HTTP 200: OK
# HTTP 502: upstream error (Nginx -> app)
# Connection refused: service không listen
```

**Step 5: Kiểm tra firewall**
```bash
ufw status verbose
iptables -L -n -v | grep <port>
# Nếu trên cloud: check Security Group (AWS), Firewall Rules (GCP)
```

**Step 6: Kiểm tra logs của service**
```bash
# systemd service
journalctl -u nginx -n 100 --no-pager
journalctl -u postgresql -f                  # Follow

# Application logs
tail -f /var/log/nginx/error.log
tail -f /var/log/nginx/access.log

# Docker container
docker logs <container_name> --tail 100
docker logs <container_name> -f              # Follow
```

**Step 7: Docker-specific**
```bash
docker ps                                    # Container running?
docker inspect <container>                   # Container config, IP, ports
docker network ls                            # Networks
docker network inspect <network>             # Container IPs trong network

# Vào container để debug
docker exec -it <container> /bin/bash
docker exec -it <container> sh
```

**Checklist debug nhanh:**
```
[ ] Service process đang chạy? (ps aux | grep service)
[ ] Service listening đúng port? (ss -tulpn)
[ ] Service binding đúng interface? (0.0.0.0 vs 127.0.0.1)
[ ] Firewall allow port? (ufw status)
[ ] DNS resolve đúng? (dig domain)
[ ] TLS cert valid? (curl -v https://domain)
[ ] Container trong đúng network? (docker network inspect)
[ ] Env variables đúng? (docker exec env)
```

---

### 7. Network Architecture của ShopLite

#### 7.1 Sơ đồ kiến trúc mạng

```
Internet (Public)
        |
        | HTTPS :443
        | HTTP  :80
        |
+-------+--------+
|   Ubuntu Server |  <-- Public IP: 1.2.3.4
|   UFW Firewall  |  <-- Chỉ mở: 22(từ IP cụ thể), 80, 443
|                 |
|  +------------+ |
|  |   Docker   | |
|  |            | |
|  | +---------+| |
|  | |  Nginx  || |  <-- Container: nginx:443, nginx:80
|  | | :443,:80|| |      Expose ra host: -p 80:80, -p 443:443
|  | +---------+| |
|  |     |       | |
|  |  +--+--+    | |
|  |  |     |    | |
|  | +--+  +--+  | |
|  | |FE|  |BE|  | |  <-- Frontend :3000, Backend :8080 (ASP.NET Core)
|  | |:3|  |:8|  | |      Không expose ra host
|  | +--+  +--+  | |
|  |       |     | |
|  |    +--+--+  | |
|  |    |  |  |  | |
|  |  +-+  +-++  | |
|  |  |PG| |RD|  | |  <-- PostgreSQL :5432, Redis :6379
|  |  |:5| |:6|  | |      Không expose ra host
|  |  +--+ +--+  | |
|  |              | |
|  | [shoplite-net]| |  <-- Docker bridge network
|  +------------+ |
+----------------+
```

#### 7.2 Docker Network Setup

```yaml
# docker-compose.yml (liên quan networking)
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"       # Host:Container
      - "443:443"
    networks:
      - shoplite-net
    depends_on:
      - frontend
      - backend

  frontend:
    build: ./frontend
    expose:
      - "3000"        # Chỉ expose trong Docker network, không ra host
    networks:
      - shoplite-net

  backend:
    build: ./backend
    expose:
      - "8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ASPNETCORE_URLS=http://+:8080
      - ConnectionStrings__Default=Host=postgres;Port=5432;Database=shoplite;Username=user;Password=pass
      - ConnectionStrings__Redis=redis:6379
    networks:
      - shoplite-net
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:16
    expose:
      - "5432"        # Không public, chỉ backend access được
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - shoplite-net

  redis:
    image: redis:7-alpine
    expose:
      - "6379"
    networks:
      - shoplite-net

networks:
  shoplite-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

volumes:
  pgdata:
```

#### 7.3 Nginx như Reverse Proxy

```nginx
# /etc/nginx/sites-available/shoplite
server {
    listen 80;
    server_name shoplite.com www.shoplite.com;
    return 301 https://$server_name$request_uri;   # Redirect HTTP -> HTTPS
}

server {
    listen 443 ssl;
    server_name shoplite.com www.shoplite.com;

    ssl_certificate     /etc/letsencrypt/live/shoplite.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/shoplite.com/privkey.pem;

    # Frontend routes
    location / {
        proxy_pass         http://frontend:3000;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }

    # API routes
    location /api/ {
        proxy_pass         http://backend:8080;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }
}
```

---

### 8. Load Balancing Concepts (Chuẩn bị Phase 6)

#### 8.1 Tại sao cần Load Balancer?

Khi một server không đủ xử lý traffic:
- Scale vertically: nâng cấp server lớn hơn (giới hạn, đắt)
- Scale horizontally: thêm nhiều server cùng loại (linh hoạt, rẻ hơn)

Load balancer phân phối request đến nhiều server (pool/backend), đảm bảo:
- Không server nào bị quá tải
- Nếu một server down, traffic tự động chuyển sang server khác
- Có thể thêm/bớt server mà không down service

#### 8.2 Thuật toán phân phối

**Round Robin:**
```
Request 1 -> Server A
Request 2 -> Server B
Request 3 -> Server C
Request 4 -> Server A (cycle lại)
```
Đơn giản, phân phối đều. Phù hợp khi tất cả request có workload tương đương.

**Weighted Round Robin:**
```
Server A: weight 3  -> nhận 3/6 = 50% request
Server B: weight 2  -> nhận 2/6 = 33% request
Server C: weight 1  -> nhận 1/6 = 17% request
```
Phù hợp khi server có capacity khác nhau.

**Least Connections:**
Request được gửi tới server đang có ít connection nhất.
Phù hợp khi request có duration khác nhau nhiều (long-polling, websocket).

**IP Hash (Sticky Sessions):**
```
Hash(client_IP) % number_of_servers = server_index
```
Cùng một IP luôn được gửi tới cùng một server. Dùng khi session phải được duy trì trên một server cụ thể (không stateless).

**Least Response Time:**
Gửi tới server có response time trung bình thấp nhất. Yêu cầu monitoring response time.

---

#### 8.3 Health Checks

Load balancer liên tục kiểm tra server còn sống không:

**Passive Health Check:**
Theo dõi response thực tế. Nếu server trả lỗi quá nhiều lần, đánh dấu unhealthy.

**Active Health Check:**
Load balancer proactively gửi request tới `/health` endpoint. Nếu không nhận được `200 OK` sau timeout, đánh dấu unhealthy.

```nginx
# Nginx upstream health check (nginx plus)
upstream backend {
    server backend1:8000;
    server backend2:8000;
    server backend3:8000;

    health_check interval=10 fails=3 passes=2 uri=/health;
}
```

**Health Check Endpoint chuẩn (ASP.NET Core):**

ASP.NET Core có built-in health checks qua `Microsoft.Extensions.Diagnostics.HealthChecks`:
```json
GET /health
{
  "status": "Healthy",
  "totalDuration": "00:00:00.0123456",
  "entries": {
    "database": { "status": "Healthy", "duration": "00:00:00.0050000" },
    "redis":    { "status": "Healthy", "duration": "00:00:00.0020000" }
  }
}
```

Cấu hình trong `Program.cs`:
```csharp
builder.Services.AddHealthChecks()
    .AddNpgsql(connectionString, name: "database")
    .AddRedis(redisConnection, name: "redis");

app.MapHealthChecks("/health");
```

---

#### 8.4 L4 vs L7 Load Balancing

**L4 Load Balancing (Layer 4 — Transport):**
- Hoạt động ở TCP/UDP level
- Không đọc nội dung request (chỉ nhìn IP và port)
- Nhanh hơn, overhead thấp
- Không thể routing dựa vào URL, header, cookie
- Ví dụ: AWS NLB (Network Load Balancer), HAProxy TCP mode
- Dùng cho: TCP services, database, low-latency requirement

**L7 Load Balancing (Layer 7 — Application):**
- Hoạt động ở HTTP/HTTPS level
- Đọc được URL, headers, cookies, body
- Có thể routing dựa vào content (path-based, host-based)
- SSL termination tại load balancer
- Ví dụ: Nginx, HAProxy HTTP mode, AWS ALB (Application Load Balancer)
- Dùng cho: Web application, microservices, API gateway

**Routing examples với L7:**
```nginx
# Path-based routing
/api/products  -> product-service:8001
/api/orders    -> order-service:8002
/api/users     -> user-service:8003
/              -> frontend:3000

# Host-based routing
api.shoplite.com     -> api-backend
app.shoplite.com     -> frontend
admin.shoplite.com   -> admin-backend
```

---

#### 8.5 So sánh Load Balancer phổ biến

| Tool         | Layer | Protocol         | Dùng khi                                |
|--------------|-------|------------------|-----------------------------------------|
| Nginx        | L7    | HTTP/HTTPS/TCP   | Web server + LB, phổ biến, cấu hình dễ |
| HAProxy      | L4+L7 | TCP/HTTP         | High performance, feature-rich          |
| AWS ALB      | L7    | HTTP/HTTPS/gRPC  | AWS, tích hợp tốt với ECS/EKS           |
| AWS NLB      | L4    | TCP/UDP/TLS      | AWS, ultra-low latency                  |
| Traefik      | L7    | HTTP/HTTPS/TCP   | Docker/Kubernetes native, auto-config   |
| Cloudflare   | L7    | HTTP/HTTPS       | CDN + LB, DDoS protection               |

---

## Kỹ năng đạt được
- Truy cập server an toàn bằng SSH key.
- Cấu hình firewall mở/đóng đúng port.
- Dùng curl/dig/ss để debug kết nối và dịch vụ.

---

## Thực hành

**Môi trường:** Ubuntu Server (Phase 1).

**Lab:**
- Tắt đăng nhập password, chỉ cho phép SSH key.
- Cấu hình firewall: chỉ mở port cần thiết (SSH, HTTP, HTTPS, port ứng dụng).
- Dùng curl test API backend, dùng dig kiểm tra phân giải tên miền.
- Vẽ lại sơ đồ luồng mạng của ShopLite (port nào, service nào nói chuyện với ai).

**Công cụ:** OpenSSH, ufw, curl, dig, ss.

---

## Bài tập

- **Bắt buộc:** Thiết lập SSH key login và vô hiệu hóa password login.
- **Nâng cao:** Cấu hình SSH config (`~/.ssh/config`) với alias + jump host.
- **Mô phỏng doanh nghiệp:** Viết tài liệu "network security baseline" liệt kê các port được mở và lý do.

---

## Deliverable
- [ ] Server chỉ truy cập được bằng SSH key, firewall đã cấu hình.
- [ ] Sơ đồ luồng mạng (network diagram) của ShopLite.
- [ ] File tài liệu network baseline.

---

## Tiêu chí hoàn thành
- [ ] SSH vào server bằng key, password login bị tắt.
- [ ] Giải thích được toàn bộ port đang mở và vì sao.
- [ ] Debug được một lỗi "không kết nối được service" bằng curl/ss.

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
