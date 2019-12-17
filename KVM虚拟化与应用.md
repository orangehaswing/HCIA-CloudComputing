# KVM虚拟化与应用

## 虚拟化概念

### 虚拟化技术演变

虚拟化技术的演变分为软件模拟、虚拟化层翻译、容器虚拟化三个阶段

其中虚拟化层分为

- 软件捕获翻译，即软件全虚拟化
- 改造虚拟机内核加虚拟化层翻译，即半虚拟化
- 硬件支持的虚拟化层翻译，即硬件支持的全虚拟化

**软件模拟**

通过软件完全模拟CPU、芯片组、磁盘、网卡等计算机硬件。典型产品是Bochs，QEMU。

 ![1576299916(1)](resources\1576299916(1).jpg)

**虚拟化层翻译**

X86指令机分为4个特权模式：Ring 0(操作系统)；Ring 1(驱动程序)；Ring 2(驱动程勋)；Ring 3(应用程序)。

 ![1576300134(1)](resources\1576300134(1).jpg)

通过虚拟化引擎，捕获虚拟机的指令并处理，但不能对硬件进行操作。称为软件全虚拟化方案。

**改造虚拟机操作系统**

Xen对虚拟机的操作系统内核改造，使虚拟机对特殊指令进行更改，和虚拟化层一起配合工作。称为半虚拟化方案。

**对CPU指令改造**

Intel推出 VT - x，增加VMX root operation和VMX non-root operation。 称为硬件支持的全虚拟化方案。

- VMM运行在VMX root operation模式
- 虚拟机运行在VMX non-root operation模式

![1576300417(1)](resources\1576300417(1).jpg)

**容器虚拟化**

原理是基于CGroups、Namespace技术将进程隔离，每个进程像一台单独的虚拟机，有自己的资源，根目录，独立进程编号，被隔离的内存空间。目前最热的容器虚拟化技术是Docker。

### KVM架构

KVM是内核一个模块，用户空间通过QEMU模拟硬件提供给虚拟机使用。

 ![1576300750(1)](resources\1576300750(1).jpg)

大多数平台通过Libvirt完成对KVM虚拟机的管理。

### QEMU和KVM

QEMU可以模拟的硬件：

- X86架构处理器，AMD64架构处理器，PowerPC架构等
- 虚拟多种设备，包括网卡，多CPU，IDE设备，软驱，显卡，声卡，USB，键盘鼠标等
- 内建DHCP，DNS，SMB，TFTP服务器

缺点：纯软件模拟，非常慢。KVM作为一个内核模块，没有用户空间管理工具，借助QEMU管理。QEMU借助KVM加速，提升虚拟机性能。

### libvirt和KVM

libvirt瑟吉欧虚拟化的管理工具，由3部分组成：

- 一套API的lib库，支持主流编程语言
- libvirtd服务
- 命令行工具virsh

libvirt实现对虚拟机的管理：创建，启动，关闭，暂停，恢复，迁移，销毁以及虚拟机网卡，硬盘，CPU，内存设备的热添加。

libvirt支持远程宿主机管理，通道可以是：

- SSH
- TCP
- 基于TCP的TLS

管理分为两个方面：

1. 存储池资源管理，支持本地文件系统目录，裸设备，lvm，nfs，iscsi等。在虚拟机磁盘格式上支持qcow2，vmdk，raw格式。
2. 网络资源管理，支持Linux桥，VLAN，多网卡绑定管理。还支持open vSwitch，nat和路由方式网络。

### 企业级产品

- VMware：X86平台虚拟化引擎，非开源产品，大部分收费。
- HyperV：微软的虚拟化产品
- Xen：最早开源虚拟化引擎
- KVM：吸取其他虚拟化技术优点，支持硬件虚拟化，性能优异。

**KVM优势**

1. 开源
2. 性能优异
3. 免费
4. =广泛免费的技术支持

## CPU和内存虚拟化

### 多CPU架构

多CPU主要有3中架构：SMP，MMP，NUMA架构。

**SMP技术**

多个CPU通过一个总线访问存储器，被称为一直内存访问(UMA)

 ![1576301976(1)](resources\1576301976(1).jpg)

**MPP技术**

一种分布式存储器模式。一个分布式存储器具有多个结点，每个结点有自己的存储器，可以配置为SMP或非SMP模式。近似一个SMP横向扩展集群。

**NUMA技术**

每个处理器都有自己的存储器，每个处理器也可以访问别的处理器的存储器。

 ![1576302197(1)](resources\1576302197(1).jpg)

 ![1576302228(1)](resources\1576302228(1).jpg)

### NUMA调优

NUMA架构每个处理器都可以访问自己和别的处理器的存储器。访问自己的存储器要比访问别的存储器快很多。速度相差10 ~ 100倍。所以NUMA调优目的是让处理器尽量访问自己的存储器。

`numactl --hardware`查看CPU硬件情况

`numastat`命令查看每个节点的内存统计

Linux默认自动NUMA平衡策略，可以关闭。

`virsh numatune`命令查看修改虚拟机NUMA配置

**内存KSM**

KSM技术可以合并相同的内存页，即使是不同的NUMA节点。

```xml
<memoryBacking>
	<nosharepages/>
</memoryBacking>	
```



### CPU绑定

`virsh vcpuinfo`命令查看虚拟机VCPU和物理CPU对应关系。

**在线绑定虚拟机CPU**

`virsh emulatorpin 21 26 - 31 --live`命令使编号为21的虚拟机CPU在26 - 31这些物理CPU之间调度。

`virsh vcpuinfo 21`查看虚拟机VCPU调度信息

`virsh vcpupin 21 0 28`强制编号为21的虚拟机VCPU 0和物理CPU28绑定。

原理：是Libvirt和CGroup来实现，通过CGroup直接绑定KVM虚拟机进程。通过CGroup不仅可以做CPU绑定，还可以限制虚拟机磁盘，网络资源控制。

应用场景：

- 系统CPU压力比较大
- 多核CPU压力不平衡，通过 cpu pinning技术人工进行匹配

### CPU热添加

```shell
1. 在虚拟机中看到4个CPU
cat /proc/interrupts

2. 把CPU在线修改成5个
virsh setvcpus CentOS7 5 --live

3. 在虚拟机将第5个CPU激活
echo 1 > /sys/devices/system/cpu/cpu4/online
```

### CPU host - passthrough技术

为了保证虚拟机在不同宿主机之间迁移的兼容性，Libvirt对CPU提炼标准的几种类型：486，pentium，pentium2，kvm64，qemu64等。包含CPU型号，生产商，每种CPU特性定义等。

CPU模式配置

1. custom模式

2. host-model模式：根据物理CPU，选择最靠近的标准CPU型号

3. host - passthrough模式：直接将物理CPU暴露给虚拟机使用，在虚拟机上完全可以看到物理CPU型号，xml配置

   `<cpu mode='host-model'/>`

应用场景：

- 需要将物理CPU一些特性传给虚拟机使用，比如虚拟机嵌套(nested)
- 需要在虚拟机里面看到和物理CPU一模一样的CPU品牌型号，在公有云很有意义，用户体验好

### CPU Nested技术

Nested技术，是在虚拟机上运行虚拟机，即KVM on KVM。KVM是将物理CPU特性全部传给虚拟机，所以理论上可以嵌套N层，但事实上，嵌套两层已经非常慢了。

配置方式：

```shell
1. 打开KVM内核模块Nested特性
rmmode kvm-intel
modprobe kvm-intel nested=1

2. 第一层虚拟机配置文件，要将物理机CPU的特性全部传给虚拟机，使用CPU HOST技术
<cpu mode='host-passthrough'/>

3. 和宿主机一样，将一层虚拟机按照宿主机配置，安装相应组件，然后安装第二层虚拟机
```

### KSM内存合并

宿主机的内存压缩采用KSM(Kernel SamePage Merging)技术。原理是将相同内存分页进行合并。

打开服务

- KSM服务
- ksmtuned服务

关闭服务

```sh
service ksm stop
service ksmtuned stop
chkconfig ksm off
chkconfig ksmtuned off
```

阻止个别虚拟机内存压缩

```xml
<memoryBacking>
	<nosharepages/>
</memoryBacking>	
```

查看KSM运行

```c
pages_shared: 有多少共享内存页正在被使用
pages_sharing:how 有多少节点被共享并且多少被保存
pages_unshared: 内存被合并时有多少内存页独特但反复被检查
pages_volatile: 多少内存页改变太快被放置
full_scans: 多少次可以合并区域被扫描
```

应用场景

- 生产环境慎用
- 测试环境推荐使用
- 桌面虚拟机推荐使用

### 内存气球

KVM内存气球技术可以在虚拟机之间按照需要调节内存大小，提高利用率。

配置要求：

- 虚拟机安装virt balloon驱动，内核开启CONFIG_VIRTIO_BALLOON

- xml配置文件添加

  ```xml
  <memballon model='virtio'>
  	<alias name='balloon0'/>
  </memballoon>	
  ```

有两种操作

- 膨胀：虚拟机内存被拿掉给宿主机
- 压缩：宿主机的内存还给虚拟机

*注意：气球技术优点是内存可以超用；缺点是可能造成内存不够使用影响性能。*

### 内存限制

可以将虚拟机内存限定在一定范围内，通过virsh或xml设定

`virsh memtune virtual_machine --parameter size`

可选参数：

- hard_limit: 虚拟机可以使用的最大内存。
- soft_limit: 竞争时的内存。
- swap_hard_limit: 最大内存加swap。
- min_guarantee: 最低保证给虚拟机使用的内存。

写到xml配置文件中：

```xml
<memory unit='KiB'>8388608</memory>
<currentMemory unit='KiB'>4194304</currentMemory>
<memtune>
	<hard_limit unit='KiB'>9437184</hard_limit>
	<soft_limit unit='KiB'>7340032</soft_limit>
	<min_guarantee unit='KiB'>4194304</min_guarantee>
	<swap_hard_limit unit='KiB'>10488320</swap_hard_limit>
</memtune>
```

应用场景：内存限制和内存气球技术结合，将内存气球限制在一定范围内，避免内存被气球无限压缩。

### 巨型页内存

内存页大小是4KB，但也可以是2MB或1GB的巨型页。

巨型页优点：

- 可以使用swap，内存默认2MB，使用后被分割为4KB
- 对用户透明，不需要用户特殊配置
- 不需要root权限
- 不需要依赖某种库文件

巨型页手工配置坏处：必须手工配置虚拟机数量，可用内存，虚拟机的启动，关闭，迁移都需要重新配置，不能使用swap。

## 网络虚拟化

一个完整的数据包从虚拟机到物理机的路径是：

虚拟机 -> QEMU虚拟网卡 -> 虚拟化层 -> 内核网桥 -> 物理网卡

 ![1576326246(1)](resources\1576326246(1).jpg)

### 半虚拟化网卡

在KVM中，默认网络设置是QEMU在Linux用户空间模拟出来并提供给虚拟机的。由于网路I/O的过程需要虚拟化引擎参与，产生大量vm exit，vm entry，效率低下。

半虚拟化网卡通过驱动对操作系统做了改造。使用较多的是Virtio技术。 Virtio驱动因为改造虚拟机操作系统，让虚拟机可以直接和虚拟化层通信，从而大大提高虚拟机的性能。

![1576326425(1)](resources\1576326425(1).jpg)

**配置**

方法有两种：

方法有两种：

1. 在虚拟化启动命令中假如virtio-net-pci参数

   QEMU-system-x86_64 c -drive file=/imagexpbase.qcow2,if=virtio -m 384 -netdev type=tap,script=/etc/KVM/QEMU-ifup,id=net0 -device virtio-net-pci,netdev=net0

2. 使用Libvirt管理KVM虚拟机，可以修改xml配置文件

   ```xml
   <interface type='bridge'>
   	<mac address='fa:16:3e:fc:e0:c0'/>
   	<source bridge='br0'/>
   	<model type='Virtio'/>
   	<address type='pci' domain='0x0000' bus='0x00' slot='0x30' function='0x00'/>
   </interface>	
   ```

**应用场景**

window虚拟机使用网卡独占技术是不错的性能解决方案，稳定性好。Linux内核默认集成Virtio驱动。

### MacVTap 和 vhost-net 技术

 ![1576327167(1)](resources\1576327167(1).jpg)

- vhost_net技术使虚拟机的网络通信绕过用户空间的虚拟化层，直接和内核通信。
- MacVTap跳过内核的网桥。

**MacVTap** 

传统Linux网络虚拟化采用是TAP+Bridge方式，将虚拟机连接到虚拟的TAP网卡，然后将TAP网卡加入Bridge。

MacVTap 实现基于传统的MacVLan。每一台MacVTap设备拥有一台对应的Linux字符设备，并拥有与TAP设备一样的IOCTL接口，因此能直接被KVM/QEMU使用。

有三种不同工作模式：

1. VEPA：同一物理网卡下的MacVTap设备之间的流量要发送到外部交换机，在由外部交换机转发到服务器。
2. Bridge：类似传统的Linux Bridge。同一物理网卡下MacVTap设备可以直接进行以太网帧的交换。
3. Private：同一物理网卡下的MacVTap设备互相无法连通，无论外部交换机支不支持hairpin模式。

**vhost_net**

Virtio的后端处理程序是由用户空间的QEMU提供。为减少延迟，提高性能，比较新的内核中增加一个vhost_net的驱动模块，在内核中实现Virtio的后端处理程序。

### 网卡的中断与多队列

X86体系结构采用中断机制协同处理器和其他设备的工作。当一台设备需要与处理器通信时，就会向处理器发出一个中断信号。在响应一个中断时，Linux内核会调用一个叫中断处理程序的函数来处理中断。

为了实现快速执行，必须将一些繁重且不重要的任务从中断处理程序中剥离出来，这部分是"下半部"。有3种方法处理下半部：

- 软中断
- tasklet
- 工作队列

网卡收到数据需要产生中断，内核调用响应。如果CPU不及时把网卡缓存复制到内存，后续数据包会因为缓存溢出而丢弃。这工作需要立即完成，剩下的交给软中断。后来出现网卡队列技术，网卡多队列就是网卡的数据请求可以通过多个CPU处理。

**物理网卡中断与多队列**

RSS是网卡的硬件特性，实现多队列，将不同的流分发到不同的CPU上，同一数据流时钟在同一CPU上，避免TCP的顺序性和CPU的并行性发生冲突。

基于流的负载均衡，解决了顺序协议和Cache并行的冲突及Cache热度问题。

**绑定中断**

依靠irqbalance服务优化中断分配，会自动收集系统数据及分析使用模式，并依据系统负载状况将工作状态置于Performamce mode 或 Power-save mode。

- Performamce mode，irqbalance会将中断尽可能均匀分发给各个CPU core。
- Power-save mode，irqbalance会将中断集中分配给第一个CPU，以保证其他空闲CPU的睡眠时间，降低能耗。

### 网卡 PCI Passthrough

通过PCI Passthrough技术将物理网卡直接给虚拟机使用，可以达到几乎和物理网卡一样的性能。

 ![1576330608(1)](resources\1576330608(1).jpg)

**PCI Passthrough配置**

1. 查看网卡设备信息，使用`lspci`命令；或使用`vrish nodedev -list -tree`
2. 使用`virsh nodedev-dumxml pci_0000_04_00_0`命令得到 xml 配置信息
3. 编辑虚拟机 xml 文件，加入PCI设备信息

**应用场景**

PCI Passthrough技术是虚拟化网卡的终极解决方案，能够让虚拟机独占物理网卡，达到最优性能。

### SR-IVO 虚拟化

SR-IVO是将PCI-E设备共享给虚拟机使用的标准，多用在网络设备上。

SR-IVO是一种硬件解决方案，提供一种从硬件上绕过系统和虚拟化层，并且使每个虚拟机能有单独的内存地址、中断、DMA流。简单就是一块物理网卡，开多个VFs(Virtual Functions)实例。有两种功能：

- PFs：拥有全功能PCI-E功能，用于配置管理SR-IOV。
- VFs：只有轻量级的PCI-E功能，包含数据传输必要的资源。但资源可以非常细致配置。

 ![1576479043(1)](resources\1576479043(1).jpg)

**优点**

- 良好性能，没有虚拟化层软件模拟的开销
- 降低成本，减少设备数量，比如网卡、交换机端口、网线

**配置**

1. 加载SR-IOV内核模块

   `modprobe igb`

2. 子网卡使用

   通过网卡独占的方式使用

   ```xml
   <interface type='hostdev' managed='yes'>
   	<source>
   		<address type='pci' domain='0' bus='11' slot='16' function='0'/>
   	</source>
   </interface>    
   ```

### Open vSwitch虚拟化交换

KVM通过Open vSwitch接入网络，相比桥接方式，有更高的灵活性。

**基本概念**

- Bridge：代表一个以太网交换机，一个主机可以创建多个Bridge设备
- Port：相当于物理交换机的端口，每个Port属于一个Bridge
- Interface：连接到Port的接口设备
- Controller：OpenFlow控制器
- datapath：负责数据交换，从接收端口的数据包在流表中进行匹配，并执行匹配到的动作
- Flow table：每个datapath与一个Flow table关联

### 多网卡绑定和建桥

在生产环境中，通过多网卡绑定，可以提高可靠性和带宽。

1. 配置modprobe文件
2. 挂载bonding模块
3. 配置多网卡绑定接口

## 磁盘虚拟化

### QEMU虚拟化

虚拟机在配置磁盘的时候，可以指定IDE、SATA、Virtio、Virtio-SCSI几种磁盘类型。

- IDE、SATA是纯软件模拟磁盘，也称为全虚拟化的磁盘
- Virtio、Virtio-SCSI是半虚拟化的磁盘，通过改造虚拟机系统的驱动来达到提升性能目标

### Virtio磁盘缓存

磁盘I/O从虚拟机到宿主机物理存储的历程。

 ![1576504879(1)](resources\1576504879(1).jpg)

每一台虚拟机的磁盘接口可以配置成writethrough、writeback、none、directsync和unsafe缓存模式。

6种缓存方式：

1. 没有指定缓存方式：默认使用writethrough模式。
2. writethrough模式：宿主机页面缓存在透写模式。虚拟机磁盘驱动告知虚拟机没有回写缓存，所有虚拟机不需要发出刷盘命令以保持数据一致性。存储设备行为就像透过缓存。
3. writeback模式：数据到达宿主机页面缓存就给虚拟机返回写成功报告，页面缓存管理机制会管理数据的合并写入宿主机存储设备。
4. none模式：宿主机的页面缓存被绕过，I/O直接在QEMU-KVM用户空间缓存和宿主机存储设备间发生。
5. unsafe模式：所有的虚拟机刷盘指令会被忽略，使用这个模式意味接受宿主机故障的时候数据丢失的风险换取性能。
6. directsync模式：只有数据被合并写入存储设备才会报告写操作成功。

**缓存模式数据一致性**

1. writethrough，none，directsync模式：比较安全模式，用于保持数据一致性的场景虚拟机可以在需要的时候刷盘。
2. writeback模式：通知虚拟机工作在回写模式，依靠虚拟机在必要时发起刷盘命令保持虚拟机镜像的数据一致性。
3. unsafe模式：同writeback类似，只是忽略虚拟机的刷盘命令，通过刷盘保持数据一致性是无效的。

**应用场景**

- 在线迁移：只有raw、qcow2、qed镜像格式支持在线迁移。如果使用集群文件系统，所有镜像格式支持迁移；如果不是，只有none模式支持在线迁移。
- 缓存方式：none用于虚拟机在线迁移；writethrough用于单机虚拟化，断电或宿主机故障不会数据丢失；writeback用于测试环境；unsafe性能较好，但最不安全。

### 磁盘镜像格式

从存储方式上，第一种存储于文件系统，第二种直接使用裸设备。

文件系统存储镜像格式：raw，cloop，cow，qcow，qcow2，vmdk，vdi等。经常使用的是raw和qcow2。

**应用场景**

在生产环境中，使用原则：

- CPU消耗型虚拟机使用qcow2文件镜像
- 磁盘I/O消耗型虚拟机使用lvm裸设备

### 文件系统块对齐

**块对齐**

第一个分区的扇区是磁盘第63扇区，并且扇区是512Byte。历史原因，硬盘必须将cylinder/head/sector信息报告给BIOS有63个扇区。因此操作系统将第一个分区的开始位置放置到第一个磁盘轨道上，从第63扇区开始。

虚拟机采用对齐方式：

- 512B：虚拟机操作系统使用本地裸设备，并且裸设备使用512bit扇区。
- 4KB：在新的本地硬盘使用4KB；文件系统的存储方式使用4KB物理扇区；基于网络的存储方式使用4KB物理扇区。
- 64KB：在高速网络存储使用，是一些高速网络存储的默认值。
- 1MB：从window 2008开始采用1MB对齐放肆。

虚拟机和宿主机的对齐方式

 ![1576506728(1)](resources\1576506728(1).jpg)

生产环境配置块对齐：

1. Window系统：安装使用winpe引导，通过diskpart命令先划分分区。
2. Linux系统：使用kickstart文件。先在预处理的部分用parted分区，然后即可使用分区。
3. 使用virt-resize命令更改块对齐方式。

### SSD在KVM虚拟机中使用

SSD硬盘有Page和Block，Page大小为4KB，Block大小为512KB。

SSD的write只能写到空的Page上，对于之前写过的Page，必须先进行一次Erase，单位是Block。如果一个Page数据删掉之后，要想再写到Page上，必须经过三步：

1. 将在同一个Block中其他Page读出来
2. 将整个Block擦除
3. 将整个Block的数据写下去

严重降低效率，称为SSD写放大，也叫写惩罚。

解决方法：

1. 预留空间：有一块保留空间，随时都能保证有未使用的空间。在空闲时对删除的块清除。
2. TRIM技术：位于操作系统层，在删除一个Page时，会同时通知SSD这个Page的数据不需要。SSD内部有一个空闲时刻的垃圾收集进程，在空闲时刻将一些空闲的数据集中到一起，然后一起Erase。

## 资源限制

KVM虚拟机的资源限制主要在CGroups配置，Libvirt在CGroups上包一层，通过修改xml文件做虚拟机的资源限制。

CGroups是Linux内核提供一种可以限制，记录，隔离进程组所使用的物理资源(CPU，内存，I/O等)的机制。

### CGroups运行机制

以分组的形式对进程使用系统资源的行为进行管理和控制。用户通过CGroups对所有进程分组，在对该分组整体进行资源的分配和控制。

所有进程都归结到init父进程。我们把Linux进程模式看成一个单一的树型结构，把CGroups模式看成是一个或多个独立的树型结构。

 ![1576585043(1)](resources\1576585043(1).jpg)

需要多个CGroups分级，因为每个分级都会附加到一个或多个子系统中。子系统代表单一资源，比如CPU时间或内存。CGroups提供一种机制，具体的策略通过子系统完成。具体如下：

- cpu子系统：为每个进程组设置一个使用CPU的权重值，以此来管理进程对CPU访问。
- cpuset子系统：对于多核CPU，可以设置进程组只能在指定的核上运行，并且设置进程组在指定的内存节点上申请内存。
- cpuacct子系统：用于生成当前进程组内的进程对CPU的使用报告。
- memory子系统：提供以页面为单位对内存的访问，比如对进程组的设置内存使用上限等。
- blkio子系统：限制每个块设备的输入/输出。通过为每个进程组设置权重来控制块设备对其的I/O时间。也可以限制IOPS。
- devices子系统：限制进程组对设备的访问。
- freezer子系统：使得进程组中的所有进程挂起。
- net-cls子系统：提供对网络带宽的访问限制。
- NameSpaces：名空间子系统。

**子系统，CGroups和进程关系**

1. 规则一

   任何单一子系统最多可附加到一个层级中。结果是，cpu子系统永远无法附加到两个不同的层级。

2. 规则二

   单个层级可附加一个或多个子系统。结果是，cpu和memory子系统都可附加到单一层级中。只要每个子系统不再附加到另一个层级即可。

3. 规则三

   单个进程可以在多个CGroups中，只要每个CGroups都在不同的层级中即可。一个进程永远不能同时位于同一层级的不同CGroups中。结果是，如果cpu和memory子系统都附加到cpu_and_mem层级中，且net_cls子系统附加到名为net的层级中，那么运行的httpd进程可以是cpu_and_mem中任意CGroups成员，同时也是net中任意CGroups的成员。

4. 规则四

   系统中的任意进程都将自己分支创建子进程，该子进程自动成为其父进程所在CGroups的成员。然后可根据需要将该子进程移动到不同的CGroups中，但开始时它总是继承其父进程的CGroups环境。

**CGroups配置方法**

1. 创建挂在条目
2. 创建组群条目
3. 创建层级并附加子系统
4. 在现有层级中附加、删除子系统

### CPU资源限制

QEMU提供对CPU的模拟，展现给虚拟机一定的CPU数目和CPU的特性。每个虚拟机都是一个标准的Linux进程，每个VCPU在宿主机中是QEMU进程派生的一个普通线程。

通过直接修改xml文件的方式绑定VCPU。

`<vcpu placement='static' cpuset='0'>1</vcpu>`

Libvirt也是通过CGroups做限制，但是可以通过virsh命令和虚拟机的xml配置文件进行配置。

### 网络资源限制

限制网络流量方式

- 通过TC限制虚拟机流量，TC是内核中一套限制网络流量的机制
- 通过Libvirt限制虚拟机流量
- 通过iptables限制虚拟机流量

**TC限制**

TC(Trffic Control)是流量控制工具。每个网络接口都有一个队列，用于管理和调度待发的数据。TC工作原理是通过设置不同类型的网络接口队列，从而改变数据包发送的速率和优先级，达到流量控制的目的。

Linux内核支持的队列：

- TBF：令牌桶过滤器，适用于流量整形
- Pfifo_fast：先进先出队列
- CBQ：分类队列，用于实现精细的QoS控制，配置复杂
- SFQ：随机公平队列
- HTB：分层令牌桶，用于实现精细的QoS控制，配置比CBQ简单

TC限流步骤：

1. 建立队列
2. 建立分类
3. 建立过滤器，过滤虚拟机的IP
4. 绑定网卡

**Libvirt限制**

在虚拟机配置文件中的网卡部分加入如下内容：

```xml
<bandwidth>
	<inbound average='100' peak='50' burst='1024'/>
	<outbound average='100' peak='50' burst='1024'/>
</bandwidth>	
```

**iptables限制发包率**

在宿主机上用iptables来限制虚拟机的发包率。

### 磁盘资源限制

1.限制磁盘IOPS：指定CGroups中某设备每秒钟可以执行的读写请求上限。

2.限制磁盘吞吐：配置CGroups参数blkio，

## 桌面虚拟化

桌面虚拟化优势：

- 降低硬件成本
- 降低运维成本，提高可管理性
- 使移动办公变为现实

桌面虚拟化架构：

- 用户终端，可以是PC，客户端，平板电脑
- 远程访问协议，RDP，PCoIP和Spice
- 虚拟化服务器

### Spice协议

架构分为3部分：Spice客户端，Spice服务端及Spice Agent

客户端负责发起连接的，在桌面虚拟化的架构中，Spice客户端运行在用户终端上。

服务端负责处理客户端的连接请求，运行在宿主机上。



















































