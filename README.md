# Lời mở đầu
Bài viết này hướng dẫn cách thiết lập môi trường biên dịch chéo (cross-compile) Qt cho Raspberry Pi 4. Thay vì cài đặt và biên dịch trực tiếp trên Raspberry Pi – vốn hạn chế về tài nguyên phần cứng, chúng ta sẽ tận dụng sức mạnh của máy tính phổ thông để biên dịch ứng dụng Qt, sau đó triển khai và chạy trên Raspberry Pi 4. 

Do sự khác biệt giữa kiến trúc ARM (trên Raspberry Pi) và x86-64 (máy tính phổ thông), ứng dụng Qt được biên dịch trên host x86-64 không thể chạy trực tiếp trên Pi. Để giải quyết vấn đề này, chúng ta sẽ copy sysroot của Pi 4 rôi Qt dựa vào nó để tạo ra ứng dụng có thể hoạt động trên Pi.

Tóm lại, quy trình gồm: copy sysroot từ Raspberry Pi về host, cấu hình Qt sử dụng sysroot đó để sinh ra ứng dụng dành cho kiến trúc ARM. Cuối cùng, ta triển khai lên Raspberry Pi thật để chạy.
# Chuẩn bị 
NOTE: Lưu ý các phiên bản Pi có thể ảnh hưởng tới kết quả

Ở đây bản Pi mình dùng là: 2022-04-04-raspios-bullseye-armhf

Máy host mình sử dụng là: Ubuntu 22.04.5 LTS x86_64 

# Chuẩn bị môi trường và cài các gói yêu cầu cho Pi

Cập nhật cho Pi
```Bash
sudo apt update && sudo apt full-upgrade -y
sudo apt reboot
```

Cài các gói package nền tảng hỗ trợ biên dịch
```Bash
sudo apt install -y libboost1.71-all-dev libudev-dev libinput-dev libts-dev \
libmtdev-dev libjpeg-dev libfontconfig1-dev libssl-dev libdbus-1-dev libglib2.0-dev \
libxkbcommon-dev libegl1-mesa-dev libgbm-dev libgles2-mesa-dev mesa-common-dev \
libasound2-dev libpulse-dev gstreamer1.0-omx libgstreamer1.0-dev \
libgstreamer-plugins-base1.0-dev  gstreamer1.0-alsa libvpx-dev libsrtp0-dev libsnappy-dev \
libnss3-dev "^libxcb.*" flex bison libxslt-dev ruby gperf libbz2-dev libcups2-dev \
libatkmm-1.6-dev libxi6 libxcomposite1 libfreetype6-dev libicu-dev libsqlite3-dev libxslt1-dev \
libavcodec-dev libavformat-dev libswscale-dev \
libx11-dev freetds-dev libsqlite0-dev libpq-dev libiodbc2-dev firebird-dev \
libgst-dev libxext-dev libxcb1 libxcb1-dev libx11-xcb1 libx11-xcb-dev \
libxcb-keysyms1 libxcb-keysyms1-dev libxcb-image0 libxcb-image0-dev libxcb-shm0 libxcb-shm0-dev \
libxcb-icccm4 libxcb-icccm4-dev libxcb-sync1 libxcb-sync-dev libxcb-render-util0 \
libxcb-render-util0-dev libxcb-xfixes0-dev libxrender-dev libxcb-shape0-dev libxcb-randr0-dev \
libxcb-glx0-dev libxi-dev libdrm-dev libxcb-xinerama0 libxcb-xinerama0-dev libatspi2.0-dev \
libxcursor-dev libxcomposite-dev libxdamage-dev libxss-dev libxtst-dev libpci-dev libcap-dev \
libxrandr-dev libdirectfb-dev libaudio-dev libxkbcommon-x11-dev
```
# Chuẩn bị cho máy tính host (Ubuntu)

Update cho ubuntu
```Bash
sudo apt update && sudo apt full-upgrade -y
```

Cài các package cần thiết 
```Bash
sudo apt install make build-essential libclang-dev ninja-build gcc git bison \
python3 gperf pkg-config libfontconfig1-dev libfreetype6-dev libx11-dev libx11-xcb-dev \
libxext-dev libxfixes-dev libxi-dev libxrender-dev libxcb1-dev libxcb-glx0-dev \
libxcb-keysyms1-dev libxcb-image0-dev libxcb-shm0-dev libxcb-icccm4-dev libxcb-sync-dev \
libxcb-xfixes0-dev libxcb-shape0-dev libxcb-randr0-dev libxcb-render-util0-dev \
libxcb-util-dev libxcb-xinerama0-dev libxcb-xkb-dev libxkbcommon-dev libxkbcommon-x11-dev \
libatspi2.0-dev libgl1-mesa-dev libglu1-mesa-dev freeglut3-dev libssl-dev
```

Tiếp theo chúng ta sẽ tải mã nguồn của CMake từ gỉthub về để build lại (Ở đây CMake đóng vai trò như trình quản lí biên dịch)
```Bash
git clone https://github.com/Kitware/CMake.git
cd CMake && ./bootstrap && make && sudo make install
```
