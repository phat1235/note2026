# TRIỂN KHAI WIREGUARD VPN FULL TUNNEL
## VPS A sử dụng IP WAN của VPS B để truy cập Internet

## 1. Mục tiêu triển khai

Triển khai WireGuard VPN kết nối hai VPS Ubuntu 22.04 theo mô hình Full Tunnel.

Mục tiêu:
- VPS A kết nối VPN đến VPS B.
- Toàn bộ lưu lượng IPv4 Internet từ VPS A được định tuyến qua VPS B.
- VPS B thực hiện NAT (MASQUERADE) để VPS A sử dụng IP WAN của VPS B khi truy cập Internet.
- VPS A vẫn duy trì khả năng truy cập SSH thông qua IP public của chính nó.
- WireGuard tự khởi động khi VPS reboot.

### 1.1. Sơ đồ hệ thống

![](https://img.lightshot.app/DfO4RhSEQAehsTOkUuTrEw.png)
```text
                 INTERNET
                     |
          +----------------------+
          |       VPS B          |
          |     Ubuntu 22.04     |
          |                      |
          | WAN: 103.124.95.200  |
          | WG : 10.66.66.1      |
          | Port: UDP 51820      |
          |                      |
          | NAT MASQUERADE       |
          +----------+-----------+
                     |
                WireGuard VPN
                UDP 51820
                     |
          +----------+-----------+
          |       VPS A          |
          |     Ubuntu 22.04     |
          |                      |
          | WAN: 103.101.162.141 |
          | WG : 10.66.66.2      |
          |                      |
          | Full Tunnel         |
          +----------------------+
```
![](https://img.lightshot.app/BVJwpbuDR_GQxb45nx56OA.png)

## 2. Thông tin hạ tầng

| Thông số | VPS A | VPS B |
|---|---|---|
| Hệ điều hành | Ubuntu 22.04 | Ubuntu 22.04 |
| Public IP | 103.101.162.141 | 103.124.95.200 |
| Subnet WAN | /24 | /24 |
| Network Interface | eth0 | eth0 |
| WireGuard IP | 10.66.66.2 | 10.66.66.1 |
| WireGuard Port | Client | UDP 51820 |
| Vai trò | VPN Client | VPN Gateway |

### 2.1. Thông số WireGuard

| Thông số | Giá trị |
|---|---|
| VPN Network | 10.66.66.0/24 |
| VPS B | 10.66.66.1 |
| VPS A | 10.66.66.2 |
| UDP Port | 51820 |
| MTU | 1420 |
| Routing | Full Tunnel |
| NAT | MASQUERADE trên VPS B |

## 3. Triển khai VPS B – VPN Gateway

VPS B là máy chủ Gateway, chịu trách nhiệm nhận kết nối WireGuard từ VPS A và NAT lưu lượng ra Internet.

### Bước 1: Cài đặt WireGuard

```bash
apt update
apt install -y wireguard iptables curl
```

### Bước 2: Tạo PrivateKey và PublicKey

```bash
umask 077

wg genkey | tee /etc/wireguard/private.key | \
wg pubkey > /etc/wireguard/public.key
```

Xem PublicKey:

```bash
cat /etc/wireguard/public.key
```

Ví dụ PublicKey VPS B:

```text
f+5wtbX23QB2mNGDvu/AX674xQcvIV48zRqjj2egLS0=
```
![](https://img.lightshot.app/pNECtTw7R1W8XBEs-ez7nA.png)

Lưu lại PrivateKey và PublicKey để sử dụng cho cấu hình hai VPS.

Lưu ý:
- PrivateKey chỉ sử dụng trên chính VPS tạo ra nó.
- PublicKey được cung cấp cho VPS đối tác.
- Không chia sẻ PrivateKey ra bên ngoài.

### Bước 3: Bật IP Forwarding

Cho phép VPS B chuyển tiếp lưu lượng mạng từ WireGuard ra Internet.

```bash
cat > /etc/sysctl.d/99-wireguard.conf <<'EOF'
net.ipv4.ip_forward=1
EOF

sysctl --system
```

![](https://img.lightshot.app/eOsyetrYSEOx532liTpJWw.png)

Kiểm tra:

```bash
sysctl net.ipv4.ip_forward
```

Kết quả:

```text
net.ipv4.ip_forward = 1
```

### Bước 4: Xác định card mạng WAN

```bash
ip route get 1.1.1.1
```
![](https://img.lightshot.app/xNDT9MD3RyODHcvqSbtlTQ.png)

Ví dụ:

```text
1.1.1.1 via 103.124.95.1 dev eth0
```

Như vậy card mạng WAN là `eth0`.

### Bước 5: Tạo cấu hình WireGuard

```bash
vi /etc/wireguard/wg0.conf
```

Nội dung:

```ini
[Interface]
Address = 10.66.66.1/24
ListenPort = 51820
PrivateKey = B_PRIVATE_KEY

PostUp = iptables -t nat -A POSTROUTING -s 10.66.66.0/24 -o eth0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -s 10.66.66.0/24 -o eth0 -j MASQUERADE

[Peer]
PublicKey = A_PUBLIC_KEY
AllowedIPs = 10.66.66.2/32
```

Thay:
- `B_PRIVATE_KEY`: PrivateKey của VPS B.
- `A_PUBLIC_KEY`: PublicKey của VPS A.

Ví dụ cấu hình theo thông tin triển khai:

```ini
[Interface]
Address = 10.66.66.1/24
ListenPort = 51820
PrivateKey = B_PRIVATE_KEY

PostUp = iptables -t nat -A POSTROUTING -s 10.66.66.0/24 -o eth0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -s 10.66.66.0/24 -o eth0 -j MASQUERADE

[Peer]
PublicKey = A_PUBLIC_KEY
AllowedIPs = 10.66.66.2/32
```

![](https://img.lightshot.app/yi72JNwrSd2KpZn83dQvOQ.png)

**Lưu ý:** Không sử dụng PublicKey làm PrivateKey. Phải lấy chính xác giá trị từ `/etc/wireguard/private.key`.

### Bước 6: Mở UDP 51820 - BỎ QUA

Nếu sử dụng UFW:

```bash
ufw allow 51820/udp
```

### Bước 7: Khởi động WireGuard

```bash
systemctl enable --now wg-quick@wg0
```

Kiểm tra:

```bash
systemctl status wg-quick@wg0
wg show
ip addr show wg0
ss -lunp | grep 51820
```
Chạy thêm cho chắc
```
wg set wg0 peer 4fhCRgcii1oVHuKChGLSIoJ9DFN9gu1KiAv1kbVdPDc= allowed-ips 10.66.66.2/32
```

## 4. Triển khai VPS A – VPN Client

VPS A là máy khách, toàn bộ lưu lượng IPv4 Internet sẽ được định tuyến qua VPS B.

### Bước 1: Cài đặt WireGuard

```bash
apt update
apt install -y wireguard iptables curl
```

### Bước 2: Tạo PrivateKey và PublicKey

```bash
umask 077

wg genkey | tee /etc/wireguard/private.key | \
wg pubkey > /etc/wireguard/public.key
```

Kiểm tra PublicKey:

```bash
cat /etc/wireguard/public.key
```

Ví dụ PublicKey VPS A:

```text
A_PUBLIC_KEY
```

Cung cấp PublicKey này cho VPS B để khai báo Peer.

### Bước 3: Tạo cấu hình WireGuard

```bash
vi /etc/wireguard/wg0.conf
```

Nội dung:

```ini
[Interface]
Address = 10.66.66.2/24
PrivateKey = A_PRIVATE_KEY

PostUp = ip rule add from 103.101.162.141/32 table main priority 100
PostDown = ip rule del from 103.101.162.141/32 table main priority 100

[Peer]
PublicKey = B_PUBLIC_KEY
Endpoint = 103.124.95.200:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Thay:
- `A_PRIVATE_KEY`: PrivateKey VPS A.
- `B_PUBLIC_KEY`: PublicKey VPS B.



### Bước 4: Khởi động WireGuard

```bash
systemctl enable --now wg-quick@wg0
```

Kiểm tra:

```bash
systemctl status wg-quick@wg0
wg show
ip addr show wg0
ip rule
```

## 5. Kiểm tra và xác minh kết nối

### 5.1. Kiểm tra WireGuard Handshake

Trên VPS A:

```bash
wg show
```

Trên VPS B:

```bash
wg show
```

Kết quả mong muốn:

```text
latest handshake: X seconds ago
transfer: ... received, ... sent
```

Nếu thấy:

```text
transfer: 0 B received
```

Cần kiểm tra:
- PublicKey và PrivateKey có khớp hay không.
- UDP 51820 có bị firewall chặn hay không.
- VPS B có đang lắng nghe UDP 51820 hay không.
- Endpoint của VPS A có đúng IP VPS B hay không.

### 5.2. Kiểm tra kết nối VPN nội bộ

Trên VPS A:

```bash
ping -c 4 10.66.66.1
```

Trên VPS B:

```bash
ping -c 4 10.66.66.2
```

### 5.3. Kiểm tra định tuyến

Trên VPS A:

```bash
ip route get 103.124.95.200
ip route get 8.8.8.8
```

Mong muốn:
- IP WAN VPS B được truy cập qua eth0 để duy trì Endpoint.
- Lưu lượng Internet được định tuyến qua WireGuard.

### 5.4. Kiểm tra IP WAN

Trước khi bật VPN:

```bash
curl -4 ifconfig.me
```

Kết quả ban đầu:

```text
103.101.162.141
```

Sau khi bật VPN:

```bash
curl -4 --max-time 10 ifconfig.me
```

Kết quả mong muốn:

```text
103.124.95.200
```

Như vậy VPS A đã sử dụng IP WAN của VPS B để truy cập Internet.


![](https://img.lightshot.app/vTXPvdiUTFOCGvWL8dZNeg.png)
