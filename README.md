# Overview


## Purpose 

This project is intended to provide access to GPU on Google Pixel Fold for general compute.


## Application

The ultimate goal is to repurpose Google Pixel Folds into a compute cluster.


## Background

A vGPU mechanism is necessary as GPU applications on Google Pixel Fold only run for Bionic. A vGPU mechanism allows a user to queue GPU commands from one device to another when the devices have different libc's.

Rutabaga is the full abstraction layer that handles dispatching of GPU API calls and outputing results. Below is the diagram of Rutabaga.

![Rutabaga Diagram](docs/rutabaga_document.png)


## Minimum Viable Product (MVP)

Our MVP is to display the interaction between host and guest applications.


# Android

Following section goes over how to build the applications in this repo on Android; the generated executables and object files are already available in this repo. If you are just interested in using this tool on pmOS, please skip to the pmOS section below.

While our Google Pixel Fold runs pmOS, which is a distribution of Alpine Linux, the executables for Rutabaga, libaemu, and GFXstream need to be built on Android OS to link properly against Bionic. The workaround is to compile on Android OS, pull the executables to pmOS, and copy the necessary shared object files into corresponding directories. 


## Setting up Termux on Android

1. Disable built-in Linux on Android
    1. Go to Settings -> System -> Developer Options -> Linux development environment
    2. Move the switch to the off position
2. Navigate to [website](https://f-droid.org/en/) 
3. Click on `Download F-Droid` button
4. Go to the Download and enable installation of apk from Chrome
5. Open F-Droid and search for Termux
6. Install Termux
    1. When warned that Termux is meant for an older version of Android, press `more information` then `Install anyway`
7. Start Termux and in notifications press `Acquire wakelock`
8. Now run the following commands in termux `pkg update; pkg upgrade; pkg install tur-repo; pkg install code-server`
9. To launch code server run: `code-server --auth none --bind-addr 0.0.0.0:8080`
10. You can now navigate to `https://<device IPv4 addr>:8080` to view the VScode instance and terminal of the phone


## Building the Executables on Android

All four executables to be built from scratch must be built on Android. The builds that are built on Android will work on pmOS if the lib and include files are copied properly (See Necessary Linux Dependencies). The necessary lib and include files have been copied to this repo so the executables may be run on pmOS. To build Rutabaga and Kumquat Media Server please refer to [website](https://crosvm.dev/book/appendix/rutabaga_gfx.html) for each step except for the GFXstream guest section. For this section add the flags `--libdir /data/data/com.termux/files/usr/lib --includedir /data/data/com.termux/files/usr/include` to the command `meson setup guest-build/ -Dvulkan-drivers="gfxstream" -Dgallium-drivers="" -Dopengl=false`. Once all necessary files are copied onto pmOS, the Kumquat executable will run; the client build may need to be rebuilt on pmOS (shown in below section).


# postmarketOS (pmOS)


## Flashing pmOS to Google Pixel Fold

1. Please visit [google drive](https://drive.google.com/drive/u/1/folders/1fJFoLv03OnqD1DCgAlVn815QidLx3vFL) and download `updated_mali_boot.img`, `vendor_boot_cgroup_memory.img`, and `starling_rootfs_v1_18_with_firmware.img`
2. Reboot the phone by holding the power and volume down button; once the phone starts to reboot, keep holding the volume down button to boot into fastboot mode
3. You will have to install adb/fastboot on your PC, please follow steps on [here](https://www.xda-developers.com/install-adb-windows-macos-linux/https://www.xda-developers.com/install-adb-windows-macos-linux/)
4. Connect your PC to Google Pixel Fold via USB-C and run the following commands in your terminal:
``` 
    fastboot oem uart enable
    fastboot oem disable-verification
    fastboot oem disable-verity
    fastboot erase init_boot
    fastboot erase boot
    fastboot flash boot "<path to updated_mali_boot.img>"
    fastboot erase super
    fastboot flash super "<path to starling_rootfs_v1_18_with_firmware.img>"
    fastboot erase vendor_boot
    fastboot flash vendor_boot "<path to vendor_boot_cgroup_memory.img>"
    fastboot reboot
```


## Setting up Environment on pmOS

1. If this is the first time booting the phone you will need to flash the phone. You can find instructions for this on the Vulkan support Google drive.
2. To connect to the phone you can either communicate via serial or by plugging the phone into the router. For MacOS users the router is the only option. Windows users may change their Internet Sharing IP to 172.16.42.1 to enable internet sharing with pmOS.
3. After ssh'ing into the phone, you will need to partition a large enough drive to hold the repo (~300 MB). To create a new partition: 
    1. `sudo mkdir /<name of dir where you want to make your mount>`
    2. `sudo mkfs.ext4 /dev/sdaX` where X can be any integer between 1 and 31 as long as nothing is mounted on that sda (you can check this by running `mount`)
    3. `sudo mount -t auto /dev/sdaX /<same dir you just made>`
    4. Now make a directory in `~` and run `sudo mount -t auto /dev/sdaX ~/<new dir>`.
    5. This new directory will be where you work going forward; run `chmod 777 <new dir>` to make it rw+ permissions.
    6. Now cd into your new directory in the home directory and run `sudo mkdir containerd`, then `sudo vi /etc/containerd/config.toml`.
    7. Change the root variable to point to `/home/user/<name>/containerd` then :wq the file to save and close it.
    8. Then run `sudo rc-service containerd restart`.
4. Now you need to remove iptables and add back iptables using `sudo apk del iptables` followed by `sudo apk add iptables` and then `sudo apk add git`.
5. Now you can configure a ssh key and link it to your GitHub account.
6. Now clone our repo and install the necessary dependencies.


## Necessary Linux Dependencies and Commands

Run `sudo nerdctl images` then run `sudo nerdctl rmi <image>` for the second image.
```
sudo rm -rf /home/google3/
sudo rm -rf /usr/local/google/
sudo chmod 777 /tmp/
sudo apk add rust
sudo apk add cargo
cd JunkyardVGPU
sudo cp -r postmarketOS/* /
sudo apk add vulkan-tools
sudo apk add vulkan-loader
sudo apk add meson
sudo apk add py3-mako
sudo apk add py3-yaml
sudo apk add py3-packaging
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


## Running Kumquat Media Server and Client Interface on pmOS

1. After installing the dependencies above you should be able to run the server and client interface without much modification!
2. Consider using a tool such as tmux to have two terminals open at the same time or just open two separate ssh windows -- one will be the server terminal and one will be the client.
3. In the server window `cd crosvm/rutabaga_gfx/kumquat/server` then `sudo ./target/debug/kumquat` and you should see the server instance start.
4. Switch to your other terminal and we need to set the below env variables to start the client interface:
   ```
   export MESA_LOADER_DRIVER_OVERRIDE=zink
   export VIRTGPU_KUMQUAT=1
   export VK_ICD_FILENAMES=/home/user/<name of your directory>/JunkyardVGPU/mesa/guest-build/src/gfxstream/guest/vulkan/gfxstream_vk_icd.aarch64.json 
   ```
5. To test, run `sudo /usr/bin/vulkaninfo` and you should see the result from vulkaninfo displayed on the terminal in which you ran the command and these lines on the screen running Kumquat server:
    ```
    I0519 00:14:09.898348   27089 VkDecoderGlobalState.cpp:972] Created VkInstance:0xb400007e5eadc410 for application:vulkaninfo engine:.                                                                  
    I0519 00:14:09.932460   27089 VkDecoderGlobalState.cpp:2039] Created VkDevice:0xb400007f0eb07580 for application:vulkaninfo engine: ASTC emulation:off CPU decoding:off.                                
    I0519 00:14:09.962327   27089 VkDecoderGlobalState.cpp:2286] Destroyed VkDevice:0xb400007f0eb07580 
    I0519 00:14:09.962595   27089 VkDecoderGlobalState.cpp:8683] Destroyed VkInstance:0xb400007e5eadc410 for application:vulkaninfo engine:.
    ```
6. Now you have the client successfully running! Please leave your environment as is and proceed to Setting up Ubuntu Container.


## Setting up Ubuntu Container

1. Navigate to your mounted directory inside your home directory (this set of instructions assumes that you have already done the previous section). Leave the server window as is; all the following can be done in the guest terminal.
2. run `sudo nerdctl pull ajayrr/opencl-kube:arm64`, this will download and unpack an Ubuntu image.
3. run `sudo nerdctl images` and find the opencl-kube:arm64 image, then run `sudo nerdctl run -v /tmp:/tmp -it <image id>`. This will start the container; for now type `exit`.
4. run `sudo nerdctl ps -a` to see all nerdctl processes, copy the container ID from this output. Then, run `sudo nerdctl start <container id>`.
5. run ` sudo nerdctl exec --user root -it <container id> /bin/bash`. This will log in as root to the container; you will notice that instead of google-felix-1 you will now see root@<container_id> as your device name in the command line.
6. run `ls` and ensure you are in `~`. Then, run `rm -rf *` to delete all files in the container home directory. Then, git clone the JunkyardVGPU repo in the container.
7. `cd JunkyardVGPU` and `cp -r Ubuntu/* /`. This will copy all of the necessary home directory shared object files. 
8. Now run `apt update` then `apt upgrade` and `apt autoremove mesa-vulkan-drivers` then `apt install vulkantools`.
9. Now set the following environment variables:
    ```
    export MESA_LOADER_DRIVER_OVERRIDE=zink
    export VIRTGPU_KUMQUAT=1
    export VK_ICD_FILENAMES=/home/ubuntu/JunkYardVGPU/mesa/guest-build/src/gfxstream/guest/vulkan/gfxstream_vk_icd.aarch64.json
   ```
10. Now you can run `vulkaninfo` in the guest terminal, and you shoud see similar results as the ones in the previous section. 


# Open Items


## Support for Color Buffer

8-bit RGBA color format is not supported for creating color buffer to display; hence, we had to comment out part of source code for GFXstream host that attempts to run OpenGL test program that uses 8-bit RGBA color format for creating color buffer to display.


## executorch

Models can be run locally using Bionic version of `vulkan_executor_runner` executable for executorch delegate provided by PyTorch. We attempted to run models remotely from Ubuntu Container using glibc version of `vulkan_executor_runner` executable for executorch; this resulted in segfault as the delegate combines C++ with Vulkan, and only Vulkan portion is forwarded to Kumquat server on pmOS followed by attempting to access virtual address that is valid only within Ubuntu Container.


## µVkCompute

We attempted to run benchmark executables of µVkCompute built for glibc remotely from Ubuntu Container; this caused host-guest inteface to break due to invalid file descriptor for BLOB as benchmark executables, unlike `vulkaninfo` exectable from Linux package manager, use unrecognized resource_id/context_id for creating BLOB.


## clvk

We installed glibc version of clvk within Ubuntu Container for forcing OpenCL commands to be executed only via GPU (not CPU). Then, we attemtped to run `clinfo` executable remotely from Ubuntu Container; Vulkan command were forwared to Kumquat server on pmOS, but the program aborted early without printing output.


# Resources, Repositories, and External Tools Used in this Repo

AEMU:           https://android.googlesource.com/platform/hardware/google/aemu

GFXStream:      https://android.googlesource.com/platform/hardware/google/gfxstream

Mesa:           https://gitlab.freedesktop.org/mesa/mesa.git

executorch:     https://docs.pytorch.org/executorch/stable/backends-vulkan.html

µVkCompute:     https://github.com/google/uVkCompute

clvk:           https://github.com/kpet/clvk
