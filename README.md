# Lời mở đầu
Bài viết này hướng dẫn cách thiết lập môi trường biên dịch chéo (cross-compile) Qt cho Raspberry Pi 4. Thay vì cài đặt và biên dịch trực tiếp trên Raspberry Pi – vốn hạn chế về tài nguyên phần cứng, chúng ta sẽ tận dụng sức mạnh của máy tính phổ thông để biên dịch ứng dụng Qt, sau đó triển khai và chạy trên Raspberry Pi 4. 

Do sự khác biệt giữa kiến trúc ARM (trên Raspberry Pi) và x86-64 (máy tính phổ thông), ứng dụng Qt được biên dịch trên host x86-64 không thể chạy trực tiếp trên Pi. Để giải quyết vấn đề này, chúng ta sẽ copy thư mục hệ thống của Raspi (sysroot) lên máy host, rồi dựa vào nó để Qt biên dịch theo kiếm trúc ARM

Tóm lại, quy trình gồm: Copy sysroot từ Raspberry Pi về host, build lại Qt dựa trên sysroot đó để sinh ra ứng dụng dành cho kiến trúc ARM. Cuối cùng, ta triển khai lên Raspberry Pi thật để chạy.
# Chuẩn bị 
NOTE: Lưu ý các phiên bản Pi có thể ảnh hưởng tới kết quả

Ở đây bản Pi mình dùng là: 2022-04-04-raspios-bullseye-armhf

Máy host mình sử dụng là: Ubuntu 22.04.5 LTS x86_64 

Phiên bản Qt mình sử dụng là Qt 6.3.0

Để tránh việc link tải các file có thể bị hỏng hoặc tốc độ chậm cũng như các phiên bản mình dùng về sau có thể bị xóa, mình đã up các file như CMake, qtbase,... lên Google Drive bao gồm các file ISO và img bạn có thể tải từ đây

https://drive.google.com/drive/folders/1J5X6xLGYgdjAFmeyzF2CfMrZihq9K3Lg?usp=sharing


# Chuẩn bị môi trường và cài các gói yêu cầu cho Pi

Cập nhật cho Pi
```Bash
sudo apt update && sudo apt full-upgrade -y
sudo apt reboot
```

Cài các gói package nền tảng
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

Giờ tạo một folder để chứa những gì tiếp theo chúng ta sẽ làm việc cùng
```Bash
cd ~
mkdir raspi && cd raspi
```

Tiếp theo chúng ta sẽ tải mã nguồn của CMake từ github về để build lại (Ở đây CMake đóng vai trò như trình quản lí biên dịch)
```Bash
git clone https://github.com/Kitware/CMake.git
cd CMake && ./bootstrap && make && sudo make install
```

Phiên bản CMake của mình ở đây là 

cmake version 4.1.20250910-g29633b2

CMake suite maintained and supported by Kitware (kitware.com/cmake).

# Tải và build lại source của Qt cho máy host

Ở đây chúng ta sẽ tải và build lại bản module qtbase-everywhere, đây là module cốt lõi của Qt

```Bash
cd ~/raspi

wget https://mirrors.sau.edu.cn/qt/archive/qt/6.3/6.3.0/submodules/qtbase-everywhere-src-6.3.0.tar.xz
mkdir folderSourceQt6 && cd folderSourceQt6
tar xf ../qtbase-everywhere-src-6.3.0.tar.xz && cd qtbase-everywhere-src-6.3.0

cmake -GNinja -DCMAKE_BUILD_TYPE=RelWithDebInfo \
-DINPUT_opengl=es2 -DQT_BUILD_EXAMPLES=OFF -DQT_BUILD_TESTS=OFF \
-DCMAKE_INSTALL_PREFIX=$HOME/raspi/qt6Host

cmake --build . --parallel 4

cmake --install .
```

Nếu quá trình tải chậm hoặc link có lỗi, hãy thử vào trang web chính của để copy link tải từ mirror khác

https://download.qt.io/archive/qt/6.3/6.3.0/submodules/qtbase-everywhere-src-6.3.0.tar.xz.mirrorlist

Sau khi hoàn thành các bước trên, chúng ta đã có Qt6 trên máy tính host của chúng ta rồi, tiếp theo sẽ là tạo trình biên dịch chéo giúp Qt tạo ra ứng dụng trên kiến trúc ARM

# Tải toolchain của Raspberry Pi
NOTE: toolchain bắt buộc nằm trong folder /opt
```Bash
sudo mkdir /opt/rpi && cd /opt/rpi
sudo wget www.ulasdikme.com/yedek/rpi-gcc-8.3.0_linux.tar.xz
sudo tar xf rpi-gcc-8.3.0_linux.tar.xz 
```

# Copy sysroot của Raspberry Pi

Lưu ý đảm bảo kết nối giữa Raspberry Pi và máy host, ở đây Raspi và máy host cùng chung một network

```Bash
cd ~/raspi
mkdir rpi_sysroot && cd rpi_sysroot
mkdir sysroot sysroot/usr

rsync -avz --rsync-path="sudo rsync" piuser@192.168.1.23:/usr/include sysroot/usr
rsync -avz --rsync-path="sudo rsync" piuser@192.168.1.23:/lib sysroot
rsync -avz --rsync-path="sudo rsync" piuser@192.168.1.23:/usr/lib sysroot/usr


wget https://raw.githubusercontent.com/riscv/riscv-poky/master/scripts/sysroot-relativelinks.py
chmod +x sysroot-relativelinks.py
python3 sysroot-relativelinks.py sysroot
```

# Biên dịch lại Qt dựa trên sysroot của Raspberry Pi

Tạo folder chứa 
```Bash
cd ~/raspi
mkdir qt-cross && cd qt-cross
```

Tiếp theo chúng ta sẽ tạo 1 file.cmake, đây sẽ là file cấu hình giúp cmake xác định đường dẫn tới sysroot của Raspi

Ở đây chúng ta sẽ đặt tên nó là toolchain.cmake

```Bash
cat << 'EOF' > toolchain.cmake
cmake_minimum_required(VERSION 3.16)

include_guard(GLOBAL)


set(CMAKE_SYSTEM_NAME Linux)

set(CMAKE_SYSTEM_PROCESSOR arm)


set(TARGET_SYSROOT $ENV{HOME}/raspi/rpi_sysroot/sysroot)


set(CROSS_COMPILER /opt/rpi/rpi-gcc-8.3.0/bin/arm-linux-gnueabihf)


set(CMAKE_SYSROOT ${TARGET_SYSROOT})


set(CMAKE_C_COMPILER ${CROSS_COMPILER}-gcc)

set(CMAKE_CXX_COMPILER ${CROSS_COMPILER}-g++)


set(CMAKE_LIBRARY_ARCHITECTURE arm-linux-gnueabihf)


set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fPIC -Wl,-rpath-link,${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE} -L${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE}")


set(CMAKE_C_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -Wl,-rpath-link,${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE} -L${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE}")


set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -Wl,-rpath-link,${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE} -L${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE}")


set(QT_COMPILER_FLAGS "-march=armv8-a -mfpu=crypto-neon-fp-armv8 -mtune=cortex-a72 -mfloat-abi=hard")

set(QT_COMPILER_FLAGS_RELEASE "-O2 -pipe")

set(QT_LINKER_FLAGS "-Wl,-O1 -Wl,--hash-style=gnu -Wl,--as-needed")


set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)

set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)

set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)

set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)


set(CMAKE_THREAD_LIBS_INIT "-lpthread")

set(CMAKE_HAVE_THREADS_LIBRARY 1)

set(CMAKE_USE_WIN32_THREADS_INIT 0)

set(CMAKE_USE_PTHREADS_INIT 1)

set(THREADS_PREFER_PTHREAD_FLAG ON)
EOF
```
Tiếp tục buld lại Qt bằng CMake, nhưng lần này sẽ dựa trên sysroot và toolchain dành cho kiến trúc ARM

Chúng ta sử dụng source qt mà bước bên trên chúng ta dã tải về

```Bash
tar xf ../qtbase-everywhere-src-6.3.0.tar.xz

cmake -GNinja -DCMAKE_BUILD_TYPE=Release -DQT_FEATURE_eglfs_egldevice=ON -DQT_FEATURE_eglfs_gbm=ON \
-DQT_BUILD_TOOLS_WHEN_CROSSCOMPILING=ON  -DQT_BUILD_EXAMPLES=OFF -DQT_BUILD_TESTS=OFF \
-DQT_HOST_PATH=$HOME/raspi/qt6Host -DCMAKE_STAGING_PREFIX=$HOME/raspi/qt6rpi \
-DCMAKE_INSTALL_PREFIX=$HOME/raspi/qt6crosspi -DCMAKE_PREFIX_PATH=$HOME/raspi/rpi_sysroot/sysroot/usr/lib/ \
-DCMAKE_TOOLCHAIN_FILE=$HOME/raspi/qt-cross/toolchain.cmake $HOME/raspi/qt-cross/qtbase-everywhere-src-6.3.0/

cmake --build . --parallel 4

cmake --install .
```

Sau khi hoàn tất chúng ta sẽ có 1 folder qt6rpi tại ~/raspi, đây là nơi chứa binaries mà Qt đã được build lại dựa trên sysroot và toolchain của Raspi

Giờ hãy gửi folder đó sang Raspi để nó có thể sử dụng

```Bash
rsync -avz --rsync-path="sudo rsync" $HOME/raspi/qt6rpi piuser@192.168.1.23:/usr/local
```

Vậy là về cơ bản Raspberry đã có bộ binaries của Qt dành cho kiến trúc ARM rồi, giờ hãy thử test một vài ứng dụng Qt Console cở bản xem chạy được chưa nhé

# Test thử biên dịch chéo

Giờ chúng ta sẽ tạo một Qt Console app Hello World để test xem liệu Raspi đã có thể chạy được source của Qt chưa nhé

Chúng ta cần tạo 1 file main.cpp và 1 file CMakeList.cmake 

File main.cpp
```Bash
cd ~/raspi

mkdir testApp && cd testApp

cat<<EOF > main.cpp 
#include <QCoreApplication>
#include <QDebug>

int main(int argc, char *argv[])
{
    QCoreApplication a(argc, argv);

    qDebug()<<"Hello world";
    return a.exec();
}
EOF
```

File CMakeList.cmake
```Bash
cat << 'EOF' > CMakeLists.txt
cmake_minimum_required(VERSION 3.5)

project(HelloQt6 LANGUAGES CXX)

set(CMAKE_INCLUDE_CURRENT_DIR ON)

set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Qt6Core)

set(CMAKE_C_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -Wl,-rpath-link, ${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE} -L${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -Wl,-rpath-link,${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE} -L${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE}")

add_executable(HelloQt6 main.cpp)

target_link_libraries(HelloQt6 Qt6::Core)
EOF
```

Hãy dùng Qt của raspi biên dịch nào
```Bash
$HOME/raspi/qt6rpi/bin/qt-cmake

cmake --build .
```

Sau đó chúng ta sẽ có 1 file tên là HelloQt6, đây chính là file thực thi của chương trình. (Tên file được định nghĩa trong CMakeLists.txt)

Giờ hãy gửi nó sang Raspi để thử thực thi file đó nào
```Bash
scp HelloQt6 piuser@192.168.1.23:~
```

# Chạy thử ứng dụng trên Raspi
Bây giờ hãy chuyển sang raspi của bạn, ở đây mình sẽ kết nối SSH với nó

Ở trên pi hãy thêm thư mục thư viện của Qt vào biến môi trường dành cho nó
```Bash
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/qt6rpi/lib/
```

Và giờ hãy chạy thử file HelloQt6 mà chúng ta đã gửi vào thư mục home lúc nãy

```Bash
cd $HOME

./HelloQt6
```

Nếu như không có bất kỳ lỗi gì thì xin chúc mừng bạn đã hoàn thành thiết lập môi trường biên dịch chéo dành cho Raspberry Pi 4 rồi

# Test thử một ứng dụng GUI với QML

Chúng ta mới chỉ cài qtbase, chỉ chạy được một số console app cơ bản, giờ hãy thử cài thêm 2 module nữa là qtshadertools và qtdeclarative nhé

```Bash
cd ~/raspi

wget https://ftp.jaist.ac.jp/pub/qtproject/archive/qt/6.3/6.3.0/submodules/qtdeclarative-everywhere-src-6.3.0.tar.xz
wget https://ftp.jaist.ac.jp/pub/qtproject/archive/qt/6.3/6.3.0/submodules/qtshadertools-everywhere-src-6.3.0.tar.xz
```

Sau đó chúng ta sẽ build 2 module này cho host trước

```Bash
cd folderSourceQt6
tar xf ../qtshadertools-everywhere-src-6.3.0.tar.xz
tar xf ../qtdeclarative-everywhere-src-6.3.0.tar.xz

cd ~/raspi/folderSourceQt6/qtshadertools-everywhere-src-6.3.0
$HOME/raspi/qt6Host/bin/qt-configure-module .
cmake --build . --parallel 4
cmake --install .

cd ~/raspi/folderSourceQt6/qtdeclarative-everywhere-src-6.3.0
$HOME/raspi/qt6Host/bin/qt-configure-module .
cmake --build . --parallel 4
cmake --install .
```

Giờ host của chúng ta đã có binaries của 2 module trên rồi, tiếp theo đó là build lại dành cho raspi

```Bash
cd ~/raspi/qt-cross
tar xf ../qtshadertools-everywhere-src-6.3.0.tar.xz
tar xf ../qtdeclarative-everywhere-src-6.3.0.tar.xz

cd ~/raspi/qt-cross/qtshadertools-everywhere-src-6.3.0
$HOME/raspi/qt6rpi/bin/qt-configure-module .
cmake --build . --parallel 4
cmake --install .

cd ~/raspi/qt-cross/qtdeclarative-everywhere-src-6.3.0
$HOME/raspi/qt6rpi/bin/qt-configure-module .
cmake --build . --parallel 4
cmake --install .
```

Giờ hãy gửi lại cho raspi bộ binaries sau khi đã thêm 2 module mới

```Bash
rsync -avz --rsync-path="sudo rsync" $HOME/raspi/qt6rpi piuser@192.168.1.23:/usr/local
```

# Kiểm thử GUI app

Hãy tạo một ứng dụng nhỏ thử nhé
```Bash
cd $HOME/raspi
mkdir qtGui
cd qtGui
```

File CMakeLists.txt
```Bash
cat << 'EOF' > CMakeLists.txt
cmake_minimum_required(VERSION 3.5)

project(HelloQt6Qml LANGUAGES CXX)
message("CMAKE_SYSROOT " ${CMAKE_SYSROOT})
message("CMAKE_LIBRARY_ARCHITECTURE " ${CMAKE_LIBRARY_ARCHITECTURE})
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Qt6 COMPONENTS Core Quick REQUIRED)

set(CMAKE_C_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -Wl,-rpath-link, ${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE} -L${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -Wl,-rpath-link,${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE} -L${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE}")

add_executable(HelloQt6Qml main.cpp)

target_link_libraries(HelloQt6Qml -lm -ldl Qt6::Core Qt6::Quick)
EOF
```

File main.cpp
```Bash
cat << 'EOF' > main.cpp
#include <QGuiApplication>
#include <QQmlApplicationEngine>

int main(int argc, char *argv[])
{
    QCoreApplication::setAttribute(Qt::AA_EnableHighDpiScaling);

    QGuiApplication app(argc, argv);

    QQmlApplicationEngine engine;
    const QUrl url(QStringLiteral("main.qml"));
    QObject::connect(&engine, &QQmlApplicationEngine::objectCreated,
                     &app, [url](QObject *obj, const QUrl &objUrl) {
        if (!obj && url == objUrl)
            QCoreApplication::exit(-1);
    }, Qt::QueuedConnection);
    engine.load(url);

    return app.exec();
}
EOF
```

File main.qml
```Bash
cat << 'EOF' > main.qml
import QtQuick 2.12
import QtQuick.Window 2.12

Window {
    visible: true
    width: 640
    height: 480
    title: qsTr("CROSS COMPILED QT6")


	Rectangle {
	    width: parent.width
	    height: parent.height

	    Rectangle {
		id: button

		width: 100
		height: 30
		color: "blue"
		anchors.centerIn: parent

		Text {
		    id: buttonText
		    text: qsTr("Button")
		    color: "white"
		    anchors.centerIn: parent
		}

		MouseArea {
		    anchors.fill: parent
		    onClicked: {
		        buttonText.text = qsTr("Clicked");
		        buttonText.color = "black";
		    }
		}
	    }
	}
}
EOF
```

Hãy buld thử và gửi nó sang Raspi để chạy
```Bash
$HOME/raspi/qt6rpi/bin/qt-cmake
cmake --build .
scp HelloQt6Qml main.qml piuser@192.168.1.23:~
```

