# Project Overview
## Purpose 
The purpose of this project is to be able to run the Vulkan test suite on a Google Pixel Fold running Alpine Linux, using a VGPU. 

## Application 
The application of this project is for an effort to repurpose Google Pixel Folds for compute. The future goal is to eventually be able to queue graphics API calls to a cluster of repurposed Pixels. This project's goal is to access the Pixel GPU in a server-client type interaction. 

## Background 
A VGPU system is necessary as the Pixel Fold does runs Bionic libc rather than GNU libc. A VGPU system allows a user to queue GPU commands from one device to another when the devices have different libc versions.  The tools that make up the VGPU interface are GFXstream and Rutabaga. 

GFXstream is a the engine which helps to forward graphics API calls between the host and guest device.
Rutabaga is the full abstraction layer which handles the dispatching of the GPU API calls and the display visualization. Below is the diagram of Rutabaga.

![Rutabaga Diagram](docs/rutabaga_document.png)


## Project MVP 
The MVP of our project is to display the interaction between the host and guest gfxstream applications. In our case Kumquat Media server is the host application and any vulkan command can be the guest application. 


# Building the VGPU

## Termux, Android, Alpine Linux and more 
While our current Pixel Build runs Alpine Linux currently the executables for Rutabaga, libaemu, and gfxstream need to be built in Android OS to link properly against Bionic. The reason for this is that the some of the libraries required are not available in Alpine Linux so the executables fail to build. The workaround is to compile on Android OS, pull the executables to the Linux build, and copy the necessary shared object files into system/lib64 and usr/include; this step is covered below in the Necessary Linux Dependencies section.

## Setting up Termux on Android OS  
1. Disable built-in Linux on Android
    1. Go to Settings -> System -> Developer Options -> Linux development environment
    2. Move the switch to the off position
2. Navigate to this [website](https://f-droid.org/en/) 
3. Click on `Download F-Droid` button
4. Go to the Download and enable installation of apk from Chrome
5. open F-Droid and search for Termux
6. Install Termux
    1. When warned that Termux is meant for an older version of Android, press `more information` then `Install anyway`
7. Start Termux and in notifications press `Acquire wakelock`
8. Now run the following commands in termux `pkg update; pkg upgrade; pkg install tur-repo; pkg install code-server`
9. To launch code server run: `code-server --auth none --bind-addr 0.0.0.0:8080`
10. You can now navigate to `https://<device IPv4 addr>:8080` to view the VScode instance and terminal of the phone

## Setting up your environment on the Alpine Linux Build
1. If this is the first time booting the phone you will need to flash the phone. You can find instructions on this in the Vulkan support google drive. 
2. To connect to the phone you can either communicate via Serial or by plugging the phone into the router and ssh'ing in. For MacOS users the router is the only option. Windows users may change their Internet Sharing IP subnet to 172.16.42.1-255 to enable internet sharing with the Linux build 
3. After ssh'ing into the Linux phone, you will need to partition a large enough drive to hold the repo (~300 MB). To create a new partition: 
    1. `sudo mkdir /<name of dir where you want to make your mount>`
    2. `sudo mkfs.ext4 /dev/sdaX` where X can be any integer between 1 and 31 as long as nothing is mounted on that sda, you can check this by running `mount`
    3. `sudo mount -t auto /dev/sdaX /<same dir you just made>`
    4. Now make a directory in `~` and run `sudo mount -t auto /dev/sdaX ~/<new dir>`
    5. This new directory will be where you work going forward, run `chmod 777 <new dir>` to make it rw+ permissions 
4. You now need to remove iptables and add back iptables using `sudo apk del iptables` followed by `sudo apk add iptables` and then `sudo apk add git`
5. You can now configure a ssh key and link it to your github account
6. Now clone our repo.
   
## Building the Executables on Android
All four executables to be built from scratch must be built on Android. The builds which are made on Android will work on PMOS if the lib and include files are copied properly (See Necessary Linux Dependencies). The necessary lib and include files have been copied to this repo so the executables may be run on the PMOS build. To build Rutabaga and kumquat media server please refer to this [website](https://crosvm.dev/book/appendix/rutabaga_gfx.html) for each step except for the gfxstream guest section. For this section add the flags `--libdir /data/data/com.termux/files/usr/lib --includedir /data/data/com.termux/files/usr/include` to the command `meson setup guest-build/ -Dvulkan-drivers="gfxstream" -Dgallium-drivers="" -Dopengl=false`. Once all necessary files are copied onto PMOS, the kumquat executable will run; the client build may need to be rebuilt on the PMOS build (shown in below section)

## Building the Application on PMOS
1. Clone this repo
2. (optional) Cut your own branch
3. Install dependencies as listed below in Necessary Linux Dependencies
4. Build gfxstream guest client
    ```
    cd mesa
    sudo meson setup --includedir /usr/include --libdir /usr/lib --wipe  guest-build/ -Dvulkan-drivers=gfxstream -Dgallium-drivers=zink -Dopengl=false
    sudo  meson install -C guest-build
    ```
5. (optional) commit your binaries to your branch


## Necessary Linux Dependencies and commands to run before executing
```
sudo chmod 777 /tmp/
sudo apk add rust
sudo apk add cargo
cd JunkyardVGPU
sudo cp -r dev/__properties__/ /dev/
sudo cp -r usr/include/* /usr/include/
sudo cp -r usr/lib/* /usr/lib/
sudo cp -r usr/bin/* /usr/bin/
sudo cp -r system/lib64/* /system/lib64/
sudo cp -r system/bin/* /system/bin/
sudo apk add vulkan-tools
sudo apk add vulkan-loader
sudo apk add meson
sudo apk add py3-mako
sudo apk add py3-yaml
sudo apk add cmake
sudo apk add libdrm-dev
sudo apk add byacc
sudo apk add flex
sudo apk add wayland-dev
sudo apk add libxcb-dev
sudo apk add libx11-dev
sudo apk add libxshmfence-dev
sudo apk add libxrandr-dev
sudo apk add g++
export CC=/usr/bin/gcc
export CXX=/usr/bin/g++

```
## Running the executables 
1. To run both executables on the same device, consider using a tool like tmux
2. You can execute kumquat media server by navigating to crosvm/rutabaga_gfx/kumquat/server/ and executing `sudo ./target/debug/kumquat`, this terminal will be the 'server' terminal
3. Any vulkan commands queued in the other terminal (the client terminal) will be run in the kumquat server terminal
4. In the guest terminal run these environment variables: 
    ```
    export MESA_LOADER_DRIVER_OVERRIDE=zink
    export VIRTGPU_KUMQUAT=1
    ```

5. To test, run `sudo /usr/bin/vulkaninfo` and you should see the result from vulkan-info displayed on the terminal you typed in and these lines on the screen running kumquat server: 
```
I0519 00:14:09.898348   27089 VkDecoderGlobalState.cpp:972] Created VkInstance:0xb400007e5eadc410 for application:vulkaninfo engine:.                                                                  
I0519 00:14:09.932460   27089 VkDecoderGlobalState.cpp:2039] Created VkDevice:0xb400007f0eb07580 for application:vulkaninfo engine: ASTC emulation:off CPU decoding:off.                                
I0519 00:14:09.962327   27089 VkDecoderGlobalState.cpp:2286] Destroyed VkDevice:0xb400007f0eb07580 
I0519 00:14:09.962595   27089 VkDecoderGlobalState.cpp:8683] Destroyed VkInstance:0xb400007e5eadc410 for application:vulkaninfo engine:.
```

# Resources, Repositories, and External Tools used in this Repo

AEMU:           https://android.googlesource.com/platform/hardware/google/aemu
GFXStream:      https://android.googlesource.com/platform/hardware/google/gfxstream
Mesa:           https://gitlab.freedesktop.org/mesa/mesa.git 


