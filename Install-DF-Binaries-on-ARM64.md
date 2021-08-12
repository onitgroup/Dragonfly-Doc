# Dragonfly installation Jetson Nano/Xavier

## Installation

> Note: The following has been run and validated on a Jetson Nano only

- Prerequirements: Jetson Nano/Xavier, 32 GB A1 SD, power supply 5V/20 A (barrel plug), fan, HDMI monitor, ETH network and cable, keyboard, mouse

- Nano/Xavier must have internet access

- Follow the tutorials regarding SD card creation and base system installation [here for **Jetson Xavier**](https://developer.nvidia.com/embedded/learn/get-started-jetson-xavier-nx-devkit) or [here for **Jetson Nano**](https://developer.nvidia.com/embedded/learn/get-started-jetson-nano-devkit).

While finalizing the installation it is suggested to follow these hints:

- give the device the name **"jetson"**
- give the user the name **"ubuntu"**
- check **"log in automatically"**

For the rest of the installation follow the defaults. Once up open the system settings app (gear icon top/right). Go to **"Brightness and Lock"** page and set **"Turn screen off..."** to **"Never"**, **"Lock"** to **"OFF"** and uncheck **"Require my password..."**.

- Open a console in the Jetson and determine the IP of `eth0`:

```bash
ifconfig eth0
```

It is recommended to continue the installation after the initial Nvidia setup from SSH, since you can simply copy and paste the instructions from this gist into the SSH window of the Xavier/Nano.

```bash
# From your PC on the same network
ssh ubuntu@ip-of-jetson
```

## Compilation

On the Jetson:

- Update
  
```bash
# Update, install requirements
sudo nvpmodel -m 0 && sudo jetson_clocks
sudo apt update -y && sudo apt upgrade -y
sudo apt install -y nano git yasm libgtk2.0-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev gstreamer1.0-libav gstreamer1.0-plugins-bad gstreamer1.0-tools gstreamer1.0-plugins-good gstreamer1.0-plugins-ugly gstreamer1.0-nice ant openjdk-11-jdk cmake ffmpeg libavcodec-dev libavdevice-dev libavfilter-dev libavformat-dev libavresample-dev libavutil-dev libpostproc-dev libswresample-dev libswscale-dev
sudo apt install deepstream-5.1 -y
```

- Make sure higher clocking is enabled:
  
```bash
# Prepare setup of proper clocking
sudo crontab -e
# You will have to choose your editor first. Choose 'nano'
# Copy the single line below to the end of the file
# Save the file by hitting 'CTRL-X' and 'Y' and 'ENTER' 
@reboot sleep 60 && /usr/bin/jetson_clocks
```

- Download and compile environment sources:
  
```bash
# Download ffmpeg sources and compile
wget https://dragonflydownloads.s3.eu-central-1.amazonaws.com/ffmpeg-3.4.2.tar.xz
tar -xvf ffmpeg-3.4.2.tar.xz -C .
cd ffmpeg-3.4.2
./configure --enable-nonfree --enable-pic --enable-shared
make -j$(nproc)
sudo make install
cd ..

# Download OpenCV3.4.1 sources and compile
wget https://dragonflydownloads.s3.eu-central-1.amazonaws.com/opencv-3.4.1.zip
unzip opencv-3.4.1.zip
cd opencv-3.4.1
mkdir build && cd build

# Set some environment variables
jvm=$( ls /usr/lib/jvm | grep "1.11.0-openjdk" )
export JAVA_HOME=/usr/lib/jvm/${jvm}
export ANT_HOME=/usr/share/ant/

# CMake build setup
cmake -DCMAKE_BUILD_TYPE=Release -DWITH_CUDA=OFF -DBUILD_opencv_cudabgsegm=OFF -DBUILD_opencv_cudalegacy=OFF -DBUILD_opencv_cudaobjdetect=OFF -DBUILD_opencv_cudaoptflow=OFF -DBUILD_opencv_dnn=OFF -DBUILD_opencv_stitching=OFF -DBUILD_opencv_superres=OFF -DBUILD_opencv_ts=OFF -DBUILD_opencv_python3=OFF -DBUILD_opencv_python_bindings_generator=OFF -DBUILD_opencv_videostab=OFF ..
make -j$(nproc)
sudo make install


# Setup Dragonfly
cd ~
mkdir dfja && cd dfja
wget https://dragonflyupdate.s3.eu-central-1.amazonaws.com/dragonfly-deployment-2.2/dfja-2.2-dist-arm64.zip
unzip dfja-2.2-dist-arm64.zip
rm dfja-2.2-dist-arm64.zip
```

## Runtime

On the Jetson:

```bash
# This should be run from a console on the Xavier
cd ~/dfja
java -jar dfja-2.2-dist.jar
# Open a browser on the Xavier and go to localhost:5000. The dashboard should appear.
```

## Known problems

- After first installation a repeated error message regarding some unrelated Tegra device is given in the console. It has been observed, that this message disappears after a reboot and a subsequent restart of the DFJA app

- The camera frame rate achieved on the Nano was pretty low (about 15 fps). No solution currently. Reason is most likely the very low capture rate of the V4L driver on these devices. Can be double checked with this GStreamer pipeline:

```bash
gst-launch-1.0 -v v4l2src device=/dev/video0 ! image/jpeg, width=640, height=480, framerate=30/1 ! jpegdec ! fpsdisplaysink 
```

I was not able to achieve much more than 15-22 fps :( 

Even with H.264 not much improvements:

```bash
gst-launch-1.0 -v v4l2src device=/dev/video1 ! video/x-h264, width=640, height=480, framerate=30/1 ! avdec_h264 ! videoconvert ! xvimagesink 
```

The a.m. GStreamer pipelines render to these CAM_SOURCE strings in DFJA (see `~/dfja/data/config/dragonfly2.properties`):

```bash
CAM_SOURCE=gstreamer:v4l2src device=/dev/video0 ! image/jpeg, width=640, height=480, framerate=30/1 ! jpegdec ! videoconvert ! appsink sync=false
CAM_SOURCE=gstreamer:v4l2src device=/dev/video1 ! video/x-h264, width=640, height=480, framerate=30/1 ! avdec_h264 ! videoconvert ! appsink sync=false
```
