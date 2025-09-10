# Lời mở đầu
Bài viết này hướng dẫn cách thiết lập môi trường biên dịch chéo Qt cho Raspberry Pi 4. Thay vì cài đặt và biên dịch trực tiếp trên Raspberry Pi – vốn hạn chế về tài nguyên phần cứng, chúng ta sẽ sử dụng máy tính host (Linux) để biên dịch ứng dụng Qt, sau đó triển khai và chạy trên Raspberry Pi 4. Phương pháp này giúp tiết kiệm thời gian, tăng hiệu suất biên dịch, đồng thời đảm bảo khả năng phát triển và kiểm thử ứng dụng Qt một cách thuận tiện hơn.

Tổng quan qua về mục tiêu chúng ta hướng tới, vì sự khác biệt giữa kiến trúc ARM của raspi và x86-64 của các dòng máy laptop hay PC phổ thông, chúng ta không thể biên dịch ứng dụng Qt trên máy Host (x86-64) của chúng ta rồi gửi sang Raspi (ARM) được. Bởi vì khi đó ứng dụng Qt được build trên bộ binary của kiến trúc X86-64.

Chúng ta sẽ cần phải copy Sysroot của Raspi, cấu hình Qt sử dụng Sysroot đó để Qt sẽ sinh ra ứng dụng dựa trên kiếm trúc ARM, khi đó ứng dụng mới có thể chạy được trên Raspi

Tóm gọn là chúng ta sử dụng cấu hình mạnh mẽ của máy tính Host, biên dịch ứng dụng Qt dựa vào Sysroot của Raspi để sinh ra ứng dụng dành cho ARM và gửi nó lên thiết bị Raspi thật của chúng ta để sử dụng.
# Chuẩn bị 
