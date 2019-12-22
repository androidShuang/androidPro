### ubantu16.04虚拟机编译android2.3源码（android系统源码情景分析）

特别注意：虚拟机的硬盘尽量大点我的是200G

#### 1.搭建环境
1.  操作系统要求

| Android版本                | 编译要求的Ubuntu最低版本 |
| -------------------------- | ------------------------ |
| Android 6.0至AOSP master   | Ubuntu 14.04             |
| Android 2.3.x至Android 5.x | Ubuntu 12.04             |
| Android 1.5至Android 2.2.x | Ubuntu 10.04             |

2. jdk版本要求

| Android版本                  | 编译要求的JDK版本 |
| ---------------------------- | ----------------- |
| AOSP的Android主线            | OpenJDK 8         |
| Android 5.x至android 6.0     | Oracle JDK 7      |
| Android 2.3.x至Android 4.4.x | Oracle JDK 6      |
| Android 1.5至Android 2.2.x   | Oracle JDK 5      |

官方编译环境搭建文档地址

[搭建环境地址](https://source.android.com/source/initializing#installing-required-packages-ubuntu-1404)

3. 编译环境

   首先确定android2.3使用jdk1.6

   下载jdk1.6 [jdk1.6下载](https://www.oracle.com/java/technologies/javase-java-archive-javase6-downloads.html#jdk-6u25-oth-JPR)

   ```
   ./jdk-6u45-linux-x64.bin
   ```

   上面命令是安装jdk

   设置环境变量

   ```shell
   sudo cp -r jdk1.6.0_45 /usr/local
   sudo vim /etc/profile
    
   //添加以下内容
   export JAVA_HOME=/usr/local/jdk1.6.0_45 
   export JRE_HOME=/usr/local/jdk1.6.0_45/jre  
   export CLASSPATH=.:$JAVA_HOME/lib:$JRE_HOME/lib:$CLASSPATH  
   export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$JAVA_HOME:$PATH 
   
   sudo source /etc/profile
   //生效设置
   //注销系统,查看jdk是否安装成功
   java -version
   javac -version
   ```

   安装gcc4(这块卡了好长时间)

   ```
   sudo add-apt-repository ppa:ubuntu-toolchain-r/test 
   sudo apt-get update
   sudo apt-get install gcc-4.4 g++-4.4 g++-4.4-multilib
   sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.4 40
   sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-5 50
   sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.4 40
   sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-5 50
    
   选择gcc4.4和g++4.4
   sudo update-alternatives --config gcc
   sudo update-alternatives --config g++
    
   查看是否安装成功
   g++ -v
   gcc -v
   ```

   如果成功会看到如下:

   ```
   gcc version 4.4.7 (Ubuntu/Linaro 4.4.7-8ubuntu1)
   ```

   make要使用3.81版本如果是3.81就不用变了，如果高于3.81就要降级

   1. 下载目标make源码： http://ftp.gnu.org/gnu/make/

   2. 解压make源码到随便一个目录。

   3. 执行./configuration

   4. 执行. build.sh

   5. 删除已安装的make：sudo apt-get remove make

   6. 执行make install

   7. 替换make文件：sudo cp make /usr/bin/make

   安装其他依赖

   ```
   sudo apt-get install bison
   sudo apt-get install zlib1g-dev
   sudo apt-get install lib32z1-dev
   sudo apt-get install flex
   sudo apt-get install libncurses5-dev
   apt-get install libncurses5-dev:i386
   sudo apt -get install libx11-dev
   sudo apt-get install gperf 
   sudo apt-get install libswitch-perl 
   sudo apt-get install libsdl1.2debian:i386
   ```

   编译android源码

   ```
   cd ./android
   vi dalvik/vm/native/dalvik_system_Zygote.c
   添加#include <sys/resource.h>
   .build/envsetup.sh
   lunch
   make
   ```

   说明一下lunch后的buidetype

   | 编译类型  | 使用情况                                                  |
   | --------- | --------------------------------------------------------- |
   | user      | 权限受限；适用于生产环境（没有root权和dedug等）           |
   | userdebug | 在user版本的基础上开放了root权限和debug权限.              |
   | eng       | 开发工程师的版本,拥有最大的权限,此外还附带了许多debug工具 |

   运行模拟器

   ```
   source build/envsetup.sh
   lunch
   emulator
   ```

   如果上面命令说找不到emulator就要加入到环境变量中

   ```
   export PATH=$PATH:/android/mydroid/out/host/linux-x86/bin
   export ANDROID_PRODUCT_OUT=/android/mydroid/out/target/product/generic
   //具体路径要根据您的源码位置进行变更
   ```

   导入到android studio中查看

   ```
   source build/envsetup.sh // 将执行文件设置为临时变量
   mmm development/tools/idegen/  //生成idegen.jar文件（#### build completed successfully (49 seconds) #### 标识生成idegen.jar文件）
   . development/tools/idegen/idegen.sh
   
   ```

   执行成功后在asop的根目录下生成android.ipr和android.iml两个个文件：

   1. android.ipr 一般保存了工程相关的设置，比如modules和modules libraries的路径，编译器配置，入口点等。

   2. android.iml 用来描述modules。它包括modules路径、 依赖关系，顺序设置等。一个项目可以包含多个 *.iml 文件。

   打开Android Studio，选择File->Open弹出路径选择框，输入相应的源码根路径，然后选择android.ipr文件，就开始导入源码,导入快慢和电脑性能有关，一般10至20分钟。

   至此整个源码编译完成，并可进行源码查看。

4. 编译android linux内核

   goldfish2.6.29的下载地址

   ```
   http://pan.baidu.com/s/1c196L3q 密码：ku02
   ```

   将压缩包解压缩

   ```shell
   tar -zxvf goldfish-android-goldfish-2.6.29.tar.gz -C kernel
   ```

   将kernel拷贝到源码根目录,进入到kernel目录修改Makefile文件

   ```
   #找到
   ARCH ?= (SUBARCH)
     CROSS_COMPILE?= 
   #改为
     ARCH ?= arm  #体系结构为arm
     CROSS_COMPILE     ?= arm-eabi-
   ```

   同时修改path，添加交叉编译环境

   ```
   vim /etc/profile
   export EABI_HOME=/android/mydroid/prebuilt/linux-x86/toolchain/arm-eabi-4.4.3
   export PATH=:$EABI_HOME/bin
   
   source /etc/profile
   #根据自己的路径进行修改
   ```

   在kernel目录下执行

   ```
   make goldfish_defconfig
   make
   ```

   最后得到Kernel: arch/arm/boot/zImage is ready这个输出证明内核源码也编译出来了。

   ```
   #让模拟器运行指定的img
   emulator -kernel ./kernel/common/arch/arm/boot/zImage & #后面这个&是后台运行的意思
   ```

   ![](.\image\编译android2.3内核驱动成功图.bmp)