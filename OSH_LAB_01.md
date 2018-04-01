# OSH_LAB_01

赵瑞 PB16120853

-------------------------

#### 环境：Ubuntu 16.04，QEMU 2.12.0-rc1，kernel Linux-4.15.14，busybox-1.28.1，GDB 8.1

------

### 调试跟踪工具安装

------

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

4. 编译内核（注意：在``make menuconfig``时候要把debug的东西选上。

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

   > 下面是错误的历史：
   >
   > 第一次安装的QEMU版本旧的时候，就是这里出了问题，出现的界面如下![52249924892](https://github.com/OSH-2018/1-ruizhao13/blob/master/pictures/1522499248925.png?raw=true)
   >
   > 还出现了QEMU的界面，不过没有内容，此时一切正常，我再开一个terminal来GDB，
   >
   > 过程如下：
   >
   > ```
   > ~$ cd linux/
   > ~/linux$ gdb
   > (gdb) file linux-4.15.14/vmlinux
   > (gdb) target remote:1234
   > (gdb) break start_kernel 
   > (gdb) c
   > ```
   >
   > 此时QEMU就显示了新内容，停在了start_kernel的部分。
   >
   > ![52249929163](https://github.com/OSH-2018/1-ruizhao13/blob/master/pictures/1522499291632.png?raw=true)
   >
   > 但是终端不可以用list来显示附近的代码，而且再次continue的话会显示该线程正在被使用，Google后有人说是GDB版本太久的bug，于是我更新了GDB，结果问题依然存在，后来同学也有这种问题，助教后来说是可以换Ubuntu17.10或者更新QEMU试试，我装了 Ubuntu17.10后又在前面的步骤遇到了新的问题，就回到Ubuntu16.04更新QEMU了，结果调试的问题小时了！看来真的是QUEM版本太旧的锅

   结果：![52251691331](https://github.com/OSH-2018/1-ruizhao13/blob/master/pictures/1522516913316.png?raw=true)

   再打开一个terminal：使用GDB来调试：

   ```shell
   $ gdb -tui
   ```

   得到界面如图：![52251709845](https://github.com/OSH-2018/1-ruizhao13/blob/master/pictures/1522517098454.png?raw=true)

   输入

   ```shell
   return
   (gdb) file linux/linux-4.15.14/vmlinux
   (gdb) target remote:1234
   ```

   ![52251723922](https://github.com/OSH-2018/1-ruizhao13/blob/master/pictures/1522517239221.png?raw=true)

   ​

------

### 关键事件

------

1. 设置断点：``start_kernel``，并开始运行

   ```shell
   (gdb) break start_kernel 
   (gdb) c
   ```

   ![52251751196](https://github.com/OSH-2018/1-ruizhao13/blob/master/pictures/1522517511966.png?raw=true)

2. 在```start_kernel```的初始之初你可以看到这两个变量：

   ```c
   char *command_line;
   char *after_dashes;
   ```

   第一个变量表示内核命令行的全局指针，第二个变量将包含`parse_args`函数通过输入字符串中的参数'name=value'，寻找特定的关键字和调用正确的处理程序。

3. 下一个函数是`set_task_stack_end_magic`，参数为`init_task`。`init_task`代表初始化进程(任务)数据结构:

   ```
   struct task_struct init_task = INIT_TASK(init_task);
   ```

   `task_struct` 存储了进程的所有相关信息。

4. `set_task_stack_end_magic` 初始化完毕后的下一个函数是 `smp_setup_processor_id`.此函数在`x86_64`架构上是空函数：

   ```
   void __init __weak smp_setup_processor_id(void)
   {
   }
   ```

   在此架构上没有实现此函数

5. 然后运行到``boot_cpu_init``，这里是激活第一个CPU事件![52255053933](https://github.com/OSH-2018/1-ruizhao13/blob/master/pictures/1522550539334.png?raw=true)

   进入这个函数观察：![52255084633](https://github.com/OSH-2018/1-ruizhao13/blob/master/pictures/1522550846332.png?raw=true)

   首先我们需要获取当前处理器的ID通过下面函数：

   ```
   int cpu = smp_processor_id();
   ```

   现在是0. 

6. Linux 内核的第一条打印信息：![52255098663](https://github.com/OSH-2018/1-ruizhao13/blob/master/pictures/1522550986632.png?raw=true)

   ![52255161222](https://github.com/OSH-2018/1-ruizhao13/blob/master/pictures/1522551612226.png?raw=true)

   调用了pr_notice函数。

   ```
   #define pr_notice(fmt, ...) \
       printk(KERN_NOTICE pr_fmt(fmt), ##__VA_ARGS__)
   ```

   pr_notice其实是printk的扩展，这里我们使用它打印了Linux 的banner。

   ```
   pr_notice("%s", linux_banner);
   ```

   打印的是内核的版本号以及编译环境信息。

7. 依赖于体系结构的初始化部分

   ![52255261853](https://github.com/OSH-2018/1-ruizhao13/blob/master/pictures/1522552618533.png?raw=true)

8. ``rest_init()``

   这是``start_kernel``的最后一个函数

   ![52255298415](https://github.com/OSH-2018/1-ruizhao13/blob/master/pictures/1522552984152.png?raw=true)

   ![52255300975](https://github.com/OSH-2018/1-ruizhao13/blob/master/pictures/1522553009752.png?raw=true)

   调用 ``rest_init()``函数进行最后的初始化工作,包括创建1号进程``(init)``,第一个内核线程等操作。最后，初始化结束。

------

### 实验总结

这次实验过程很艰辛，刚开始QEMU的版本问题和编译内核时没有选debug info等导致了前期花费了大量时间来搭调试环境。通过这次实验，对Linux和Linux内核不再陌生了，刚开始做实验的时候都不知道从何看起，熟悉了Linux的一些命令，学习了一些GDB的调试命令，虽然很艰辛，但是感觉很有收获。