1. QNX Momentics IDE使用的是什么编译器？
   - GCC
2. QNX的内核是在是在什么时候编译的？
   - 由 BlackBerry QNX 编译 (用于发布):
    当BlackBerry QNX 公司在发布 QNX Neutrino RTOS 以及对应的 QNX SDP时，会发布内部编译好内核的二进制版本。主要提供带这些配置的二进制文件：
        1. ARMv7, AArch64, x86, x86-64----针对不同目标架构
        2. SMP - 对称多处理， non-SMP，带/不带指令插装 ---针对不同配置
   - 由开发者构建系统镜像时 (集成阶段)
    当开发者使用 QNX Momentics IDE 和 SDP 为他们的特定硬件平台构建一个完整的、可启动的操作系统镜像（即.ifs 文件）时，开发者不会重新从源代码编译 QNX 内核本身。开发者在这个阶段做的是：
        1. 选择一个由 BlackBerry QNX 预编译好的、适合目标的内核二进制版本文件
        2. 编写构建文件（.build 文件）。
                描述镜像中需要包含哪些组件，包括选定的内核、各种系统服务管理器（如 devc-ser8250 串口驱动, io-pkt 网络栈等）、共享库、配置文件以及开发者自己的应用程序。
        3. 链接和打包
                使用 SDP 提供的工具（如 mkifs - make image filesystem）根据构建文件将所有这些组件（包括预编译的内核）链接和打包在一起，生成最终的操作系统镜像。
3. 构建文件书写语法是什么？
   QNX 的 OS Image 构建文件（通常使用 .build 扩展名）是给 mkifs (Make Image File System) 工具使用的脚本。它定义了镜像的内容、属性和启动行为。其语法相对直接，主要包含以下几类元素：
    - 属性 (Attributes): 使用方括号 [...] 定义全局属性或特定文件的属性。 
        [image=...]:通常是文件的第一行，标记镜像定义的开始
        [virtual=cpu,binary] 或 [virtual=cpu,elf]: 定义一个虚拟地址空间段，通常用于放置内核和主要的可执行文件。cpu 需替换为目标架构（如 armle-v7, x86_64）。binary 表示原始二进制，elf 表示 ELF 格式。可以添加 +compress 来压缩内容。
        [physical=...]: 定义物理地址空间段（较少用于通用 OS 镜像，更多用于 IPL）。
        [boot=...]: 指定包含启动脚本的文件（通常指向下面的 [type=script] 部分）。
        [+script] 或 [type=script] ... [-script]: 定义内联启动脚本。这是镜像启动后最先执行的命令序列。
        [ perms=...]: 设置文件权限（如 perms=0755）。
        [uid=...], [gid=...]: 设置用户和组 ID（通常是 0，即 root）。
        [type=link] target=source: 在镜像内部创建符号链接。
        [type=file] (通常省略，是默认类型)。
    - 文件包含 (File Inclusion):
        host_path=image_path: 将主机（编译环境）上的文件 host_path 复制到镜像内的 image_path。这是最常用的方式。路径通常使用环境变量，如 $QNX_TARGET/armle-v7/boot/sys/procnto-smp-instr=/boot/sys/procnto-smp-instr。
        image_path={ inline content }: 直接在构建文件中定义一个小文件的内容。
    - 注释 (Comments): 以 # 开头的行是注释。
4. 为什么没有创建构建脚本，在IDE上创建的一个helloworld程序也能调试运行？
   这个问题揭示了 QNX Momentics IDE 的两种主要工作模式：应用程序开发 和 系统构建。
    当创建一个简单的 "Hello World" C/C++ 项目（不是 System Builder Project）并在 IDE 中运行或调试它时，你并没有在构建一个完整的、自启动的操作系统镜像。这时发生的是：
    1. 目标环境已存在: 你需要先有一个已经运行着 QNX Neutrino RTOS 的目标系统。这个目标系统可以是：
        - 真实的硬件开发板，上面已经烧录了一个 QNX 系统（可能是 BSP 自带的预编译镜像，或者是你之前构建的某个镜像）。
        - 一个虚拟机（如 VMware, VirtualBox）中运行的 QNX。
        - QNX 通过 QEMU 模拟器运行。
    2. 编译应用程序: IDE 使用 QNX SDP 提供的交叉编译器（GCC）将你的 "Hello World" 源代码编译成一个针对目标 QNX 系统和架构的可执行文件（例如 hello_world）。这只是一个普通的应用程序二进制文件，不是 OS 镜像。
    3. 部署 (可选): IDE 通过网络（通常利用目标上运行的 qconn 服务）将编译好的可执行文件传输到正在运行的目标 QNX 系统的某个目录（例如 /tmp）。
    4. 远程执行/调试: IDE 指示目标系统上的调试代理（如 pdebug 或 gdbserver，通常也是通过 qconn 交互）加载并运行你刚刚传输过去的 hello_world 可执行文件。如果是调试会话，IDE 会与调试代理通信，控制程序的执行（设置断点、单步执行、查看变量等）。
    
    关键区别在于： 你是在一个已经存在的、功能完备的 QNX 操作系统上运行和调试你的单个应用程序，而不是从零开始创建一个包含内核、驱动和你的应用程序的启动镜像。IDE 帮你处理了编译、传输和远程启动/调试的细节，让你感觉好像直接在本地运行一样。
    只有当你需要：
       - 定制操作系统启动过程。
       - 包含特定的驱动或系统服务。
       - 让你的应用程序成为系统启动的一部分（例如，自启动）。
       - 创建一个可以独立烧录到硬件上并启动的完整系统时。 
    你才需要创建 System Builder Project 和编写 .build 构建脚本来生成 .ifs 操作系统镜像。

