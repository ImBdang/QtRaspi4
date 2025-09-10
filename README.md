# Lời mở đầu
Bài viết này hướng dẫn cách thiết lập môi trường biên dịch chéo (cross-compile) Qt cho Raspberry Pi 4. Thay vì cài đặt và biên dịch trực tiếp trên Raspberry Pi – vốn hạn chế về tài nguyên phần cứng, chúng ta sẽ tận dụng sức mạnh của máy tính phổ thông để biên dịch ứng dụng Qt, sau đó triển khai và chạy trên Raspberry Pi 4. 

Do sự khác biệt giữa kiến trúc ARM (trên Raspberry Pi) và x86-64 (máy tính phổ thông), ứng dụng Qt được biên dịch trên host x86-64 không thể chạy trực tiếp trên Pi. Để giải quyết vấn đề này, chúng ta sẽ copy sysroot của Pi 4 rôi Qt dựa vào nó để tạo ra ứng dụng có thể hoạt động trên Pi.

Tóm lại, quy trình gồm: copy sysroot từ Raspberry Pi về host, cấu hình Qt sử dụng sysroot đó để sinh ra ứng dụng dành cho ARM. Cuối cùng, ta triển khai binary này lên Raspberry Pi thật để chạy.
# Chuẩn bị 
