# Lời mở đầu
Bài viết này hướng dẫn cách thiết lập môi trường biên dịch chéo (cross-compile) Qt cho Raspberry Pi 4. Thay vì cài đặt và biên dịch trực tiếp trên Raspberry Pi – vốn hạn chế về tài nguyên phần cứng, chúng ta sẽ tận dụng sức mạnh của máy tính host (Linux) để biên dịch ứng dụng Qt, sau đó triển khai và chạy trên Raspberry Pi 4. Phương pháp này giúp tiết kiệm thời gian, tăng hiệu suất biên dịch, đồng thời mang lại sự thuận tiện trong quá trình phát triển và kiểm thử ứng dụng Qt.

Do sự khác biệt giữa kiến trúc ARM (trên Raspberry Pi) và x86-64 (trên máy tính phổ thông), ứng dụng Qt được biên dịch trên host x86-64 không thể chạy trực tiếp trên Raspberry Pi. Để giải quyết vấn đề này, chúng ta cần sử dụng cross-compiler dành cho ARM, kết hợp với sysroot – tức là bản sao môi trường hệ thống (thư viện, header, …) từ Raspberry Pi. Nhờ đó, ứng dụng sẽ được biên dịch thành binary ARM phù hợp với Raspberry Pi.

Tóm lại, quy trình gồm: copy sysroot từ Raspberry Pi về host, thiết lập Qt cross-compile toolchain (cross-compiler + sysroot), sau đó sử dụng host để biên dịch và sinh ra ứng dụng ARM. Cuối cùng, ta triển khai binary này lên Raspberry Pi thật để chạy.
# Chuẩn bị 
