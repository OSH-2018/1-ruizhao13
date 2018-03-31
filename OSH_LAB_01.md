# OSH_LAB_01

####环境：Ubuntu 16.04，QEMU 2.12.0-rc1，kernel Linux-4.15.14，busybox-1.28.1

-------------

1. 下载[kernel](https://www.kernel.org/)、[busybox](http://www.busybox.net/)、[QEMU](https://www.qemu.org/)

   都是下载源码编译的。

   > 这三个的版本的匹配真的很重要，刚开始我装QEMU是直接```apt install qemu```，装的QEMU版本比较旧，于是后面在GDB调试的时候就出现了问题

2. 安装QEMU，先解压，然后如下：

   ```shell
   $ cd qemu-2.12.0-rc1/
   $ sudo make clean
   $ ./configure 
   $ sudo make
   $ sudo make install
   ```

   为了后面方便，把QEMU加入path

   ```shell
   ln -s /usr/bin/qemu-system-x86_64 /usr/bin/qemu  
   ```

3. 安装busybox，也是先解压，然后如下安装

   > 注意！：在第二行命令时，要修改一些东西
   >
   > 因为Linux运行环境当中是不带动态库的，所以必须以静态方式来编译Busybox。修改
   >
   > **Busybox Settings --->** 
   >     **Build Options --->** 
   >          **[\*] Build Busybox as a static binary(no shared libs)**

   ```shell
   $ sudo make defconfig  
   $ sudo make menuconfig  
   $ sudo make 
   $ sudo make install 
   ```

2. 编译内核（注意：在``make menuconfig``时候要把debug的东西选上。

   ```
   ~$ cd linux/
   ~/linux$ ls
   build-initrd.sh  busybox-1.28.2.tar.bz2  linux-4.15.14.tar.gz
   busybox-1.28.2   linux-4.15.14
   $ cd linux-4.15.14/
   $ sudo su
   # make clean
   # make menuconfig
   # make -j10
   ```

5. 准备根文件系统

   1. 我在``/home/username/``下 建立了``myfile``文件夹，在这个 文件夹下：

      ```
      dd if=/dev/zero of=busyboxinitrd4M.img bs=4096 count=1024    
      mkfs.ext3 busyboxinitrd4M.img              
      mkdir rootfs  
      sudo mount -o loop busyboxinitrd4M.img rootfs/  
      ```

   2. 回到Busybox目录：输入如下命令：

      ```shell
      make CONFIG_PREFIX= /home/ruizhao/myfile/rootfs/ install 
      ```

   3. 再次回到``myfile``目录下：

      ```shell
      umount rootfs  
      ```

   ​

6. 开始调试

   ```
   ~/linux$ qemu-system-x86_64 -kernel ./linux-4.15.14/arch/x86_64/boot/bzImage ../myfile/busyboxinitrd4M.img -append "root=/dev/ram init=/bin/ash" -append nokaslr -s -S
   ```

   >  下面是错误的历史：
   >
   >  第一次安装的QEMU版本旧的时候，就是这里出了问题，出现的界面如下![52249924892](C:\Users\ruizhao\AppData\Local\Temp\1522499248925.png)
   >
   >  还出现了QEMU的界面，不过没有内容，此时一切正常，我再开一个terminal来GDB，
   >
   >  过程如下：
   >
   >  ```
   >  ~$ cd linux/
   >  ~/linux$ gdb
   >  (gdb) file linux-4.15.14/vmlinux
   >  (gdb) target remote:1234
   >  (gdb) break start_kernel 
   >  (gdb) c
   >  ```
   >
   >  此时QEMU就显示了新内容，停在了start_kernel的部分。
   >
   >  ![52249929163](C:\Users\ruizhao\AppData\Local\Temp\1522499291632.png)
   >
   >  但是不可以用list来显示，而且再次continue的话会显示该线程正在被使用，Google后有人说是gdb版本太久的bug，于是我更新了gdb，结果问题依然存在，后来同学也有这种问题，助教后来说是可以换Ubuntu17.10或者更新QEMU试试，我装了 Ubuntu17.10后又在前面的步骤遇到了新的问题，就回到Ubuntu16.04更新QEMU了，结果调试的问题小时了！看来真的是QUEM版本太旧的锅

   结果：![52251691331](C:\Users\ruizhao\AppData\Local\Temp\1522516913316.png)

   再打开一个terminal：使用GDB来调试：

   ```shell
   $ gdb -tui
   ```

   得到界面如图：![52251709845](C:\Users\ruizhao\AppData\Local\Temp\1522517098454.png)

   输入

   ```shell
   return
   (gdb) file linux/linux-4.15.14/vmlinux
   (gdb) target remote:1234
   ```

   ![52251723922](C:\Users\ruizhao\AppData\Local\Temp\1522517239221.png)

   然后设置断点：``start_kernel``，并开始运行

   ```shell
   (gdb) break start_kernel 
   (gdb) c
   ```

   ![52251751196](C:\Users\ruizhao\AppData\Local\Temp\1522517511966.png)

   ​

