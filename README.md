# command_ddos
Phương pháp bảo vệ máy chủ Linux từ các cuộc tấn công DDoS .
CONFIG_NETFILTER_XT_MATCH_STRING=m
Hạn chế 20 yêu cầu mỗi giây
iptables –new-chain car
iptables –insert OUTPUT 1 -p tcp –destination-port 80 -o eth0 –jump car
iptables –append car -m limit –limit 20/sec –jump RETURN
iptables –append car –jump DROP

Tối đa 10 kết nối đồng thời từ một địa chỉ IP
iptables -A INPUT-p tcp –dport 80 -m iplimit –iplimit-above 10 -j REJECT

Ngăn chặn hơn 10 SYN
iptables -I INPUT -p tcp –syn –dport 80 -j DROP -m iplimit –iplimit-above 10

20 kết nối vào một mạng
iptables -p tcp –dport 80 -m iplimit –iplimit-above 20 –iplimit-mask 24 -j REJECT

thuật toán khi botnet tấn công:

1: bot nhận lệnh từ người quản trị
2: bot sẽ chuyển đổi nói cách khác là di chuyển đến ip ,domain của mục tiêu
3: bot sẽ gửi yêu cầu trên IP ” GET / HTTP/1.0 ” đến máy chủ web của nạn nhân

Khi ta nhìn thấy các triệu chứng DDOS tấn công thì hãy kiểm tra access_log máy chủ web Apache . Nếu bị tắc với mục của hình thức ” GET / HTTP/1.0 ” , có nghĩa là các cuộc tấn công là một địa chỉ IP .

Phải làm gì? Trước hết, tạm thời vô hiệu hóa các địa chỉ IP tấn công , tốt nhất là thông qua iptables
iptables -A FORWARD -p tcp -s –dport 80 -j REJECT

Điều này sẽ cung cấp cho máy chủ của bạn ít nhất là một số cơ hội mà các botnet sẽ không có thời gian để ghi toàn bộ kênh đến máy chủ, và sau đó liên kết với nó sẽ bị mất. Sau khi ngắt kết nối truy cập của tên miền bị tấn công có thể được mở ra sau đó

Bước tiếp theo – là vô hiệu hóa các tên miền . Phương pháp đơn giản và hiệu quả nhất là nên khóa yêu cầu với các tên miền . Điều này ngăn cản việc chuyển đổi của tên miền và các chương trình sẽ không thể có được một địa chỉ IP . Một lần nữa, iptables sẽ làm điều đó :
iptables -I INPUT 1 -p tcp –dport 53 -m string –string “domain.com” –algo kmp -j DROP
iptables -I INPUT 2 -p udp –dport 53 -m string –string “domain.com” –algo kmp -j DROP
Lưu ý – nó sẽ chặn cổng TCP và UDP .

Điều này sẽ giúp bảo vệ các trang chính của trang web bằng cách lọc tất cả các yêu cầu của các botnet từ chương trình không thực hiện thiết lập khi chuyển hướng và sẽ tiếp tục làm việc để rỗng ” GET / HTTP/1.0 ”

Ta nên Làm thế nào ? Cách tốt nhất là đặt ở phía trước của apache Web Server Nginx nhanh chóng dễ dàng và cung cấp cho bạn các tập tin tĩnh . Tạo ra một tập tin index.html tĩnh , điều đó sẽ làm cho việc chuyển đổi sang một trang khác và thiết lập chuyển hướng sử dụng các thẻ html meta.
Khi bị tấn công DDOS , nginx sẽ có thể chịu được yêu cầu nhiều hơn cho các tập tin tĩnh trong apache .
Nếu nginx không làm việc, và máy chủ không giữ nhiều yêu cầu – kênh bị tắc
iptables -I INPUT 1 -p tcp –dport 80 -m string –string “GET / HTTP/1.0” –algo kmp -j DROP

Cấu hình FreeBSD để phát hiện và chống lại cuộc tấn công DDOS
net.inet.tcp.blackhole = 2
net.inet.udp.blackhole = 1

Bảo vệ chống lại các cuộc tấn công SYN
Một trong những chính là các cuộc tấn công phổ biến nhất SYN , trong đó tất cả các kết nối của máy chủ bị tấn công sẽ bị tràn và cố gắng kết nối không hợp lệ .
kern.ipc.somaxconn = 1024

Chuyển hướng ( redirect )
Một kẻ tấn công có thể sử dụng IP chuyển hướng đến thay đổi mục tiêu cho các máy chủ từ xa .Cả hai – gửi và chấp nhận chuyển hướng nên bị vô hiệu.
net.inet.icmp.drop_redirect = 1
net.inet.icmp.log_redirect = 1
net.inet.ip.redirect = 0
net.inet6.ip6.redirect = 0

Thiết lập IP cho UNIX hoạt động tối ưu :
Kích thước của bộ đệm nhận và truyền tải TCP ảnh hưởng trực tiếp đến tham số kích thước cửa sổ TCP . Tăng kích thước của cửa sổ sẽ cho phép truyền dữ liệu hiệu quả hơn , đặc biệt là với các thiết bị nặng , chẳng hạn như FTP và HTTP . Giá trị mặc định không tối ưu và cần phải được tăng lên đến 32.768 byte. Giá trị này không được nhiều hơn 64K nếu bạn không biết về những hạn chế RFC1323 và RFC2018 , và nếu không có sự hỗ trợ của cả hai bên.
FreeBSD :
sysctl-w net.inet.tcp.sendspace = 32768
sysctl-w net.inet.tcp.recvspace = 32768

Làm sạch các ARP :
Tùy chọn này nên đặt trong 20 phút theo mặc định.
FreeBSD :
sysctl-w net.link.ether.inet.max_age = 1200

Định tuyến gửi :
FreeBSD :
sysctl-w net.inet.ip.sourceroute = 0
sysctl-w net.inet.ip.accept_sourceroute = 0

cài đặt TIME_WAIT

FreeBSD :
sysctl-w net.inet.icmp.bmcastecho = 0

FreeBSD :
sysctl-w net.inet.icmp.maskrepl = 0
