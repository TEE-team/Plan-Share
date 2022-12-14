# A Hardware-Software Co-design for Efficient Intra-Enclave Isolation
## 背景
为了保证用户数据、密钥等敏感信息不被恶意程序攻击，intel使用SGX架构，将应用程序放到称为enclave的可信执行环境中隔离执行。

但是，要想应用程序支持SGX架构，需要应用程序开发者使用SGX提供的库来开发，这也就意味着之前开发的应用程序需要进行重构。为了尽量减少代码重构工作，目前流行的做法是调用在enclave中的第三方库或者库操作系统来运行整个应用程序。但是这会带来两个问题

## 问题

+ LibOSes有很多漏洞，容易遭受攻击

+ SGX enclave采用了整体模型使得Trusted Computing Base(可信计算基)十分臃肿，攻击面增加的同时性能还有所下降。例如，一旦将包含漏洞的第三方库导入到enclave中，攻击者可能会利用这些漏洞来篡改enclave的完整性甚至机密性

+ 创建多个的enclave会产生巨大的跨边界开销
## 研究点

保证SGX安全性的同时提高性能，还能方便开发人员编程

## 提出解决方法(任务)

克服障碍将Intel MPK技术与enclave结合，利用MPK地址空间划分的硬件特性实现enclave内隔离

理由如下：

+ MPK在内存访问验证方面几乎没有额外的开销
+ MPK与SGX都是针对用户级的程序，彼此交互较为方便
+ 可以将地址空间划分到多个内存域，可以很自然的实现enclave内部的隔离

### 实现上述任务所要克服的困难

+ 安全问题：
  + MPK是完全受制于底层操作系统的，这就造成了操作系统可以任意操作MPK来违反隔离
  + MPK提供用户级指令WRPKRU，用于修改域访问权限，攻击者可以利用这一点绕过enclave的隔离措施

## 任务
+ 允许在一个SGX enclave中建立多个轻量级灵活的隔离
+ 扩展了SGX，以解决SGX与MPK信任模型之间的冲突
+ 提供兼容现有开发流程的友好编程模型
+ 针对以上要求，提出了LIGHTEBCLAVE，是一种软硬件协同设计
## LE的优点
+ enclave隔离之间高效安全的通信
	+ 高效：隔离之间的数据通信通过轻量级的lightenclave-gates而不是退出在重进enclave
	+ 安全：数据使用安全的共享域进行共享，而不是在不受保护的内存通过加密来共享
+ 实施内存隔离的开销大大减少
	+ 使用硬件强制的域检查，相较于边界检查，开销几乎为零，并且不要求连续域区域
+ 更灵活
	+ 软硬件协同机制，硬件层面LE提供对enclave的划分机制；软件层面将lightenclave进行抽象：可以是互相不信任的，也可以是等级分明的
## 效果
+ 对于两个最先进的SGX LibOSes：Graphene-SGX和Occlum
	+ 对于Graphene-SGX，与不用lightenclave相比：
		+ 为CPU密集型工作负载实现了10.5倍的加速比
		+ 为多任务密集型工作负载实现了46.5倍的加速比
	+ 对于Occlum，与不用lightenclave相比：
		+ 为CPU密集型工作负载实现了1.49倍的加速比
		+ 为多任务密集型工作负载实现了1.28倍的加速比
+ 除了提供加速，lightenclave还提供更高的安全性
	+ 可以将服务应用程序分解为一些组件来隔离不受信任的应用程序
+ 对于隐私保护的无服务器应用程序，LE将启动延迟减少了50%-77%
## 基础知识
+ IntelSGX
	+ Secure memory
		+ SGX架构将一部分DRAM分出来作为secure memory，称作EPC(Enclave Page Cache),EPC由EPCM(EPCMap)来进行管理，EPCM同样存储在这部分DRAM中。EPCM为每个EPC page创建一个entry，保存着对应page的读、写以及执行权限、映射虚拟地址，page的所有者enclave等。对于Secure memory，MMU(Memory Management Unit)不仅会检查页表信息，也会检查EPCM信息。
	+ Enclave创建
		+ Enclave是由OS发送创建指令*ECRWATE*生成的，在创建的时候CPU会生成一个measurement来保证对enclave的控制，在外界想访问enclave时，会首先请求这个measurement并且使用认证服务来检查enclave的内部结构？
		+ 使用EADD来将空的EPC page添加到enclave中，使用PAGEINFO这个参数结构来将page的权限传递到EPCM中
		+ 使用EINIT作为enclave完成的标志
	+ Enclave执行过程
		+ 一个thread通过非特权指令EENTER和EEXIT来进入和退出enclave。当进入enclave时，该thread独占一个enclave TCS(一个epc页面)，这个页面会向thread指定一个固定的入口点和状态保存区(State Save Area, SSA)来保证thread可以获得enclave的执行结果
		+ Enclave的执行过程可以被异常或者中断中断，在中断之后，CPU执行异步enclave exit(AEX)来保存执行的状态到SSA中；若想再次恢复执行状态，可使用ERESUME指令
+ Intel MPK
	+ MPK和内存域(memory domains)
		+ MPK允许一个应用程序将其所管理的虚拟地址空间划分为16个不同的内存域。同时，生成长度为4位的域id，将其存储在内存页相对应的页表项的前四位预留位置中。
	+ MPK寄存器和指令
		+ 针对CPU的每个核心，MPK中都有一个叫做PKRU的寄存器，它为核心指定对不同域的访问权限。寄存器大小为32位，每两位代表对不同域的访问权限
		+ 使用WRPKRU和RDPKRU两个指令来修改和读取PKRU
		+ MMU可以强制执行MPK检查
	+ MPK与SGX的interaction(互动)
		+ 除了原有的SGX内存权限检查之外，MMU还支持将MPK域权限检查应用于enclave内存访问
		+ PKRU寄存器也可以在AEX期间自动保存由ERESUME恢复
## Light-enclave架构细节
### 内容概述
+ 在enclave内部使用MPK技术进行程序隔离。+ 
+ 每个程序享有一个只能自己访问的内存域，该技术是依靠LE在enclave表项张标记域ID(domain-id)来实现的。
+ 默认le之间是互不信任的，不能访问之间的内容，但是LE也提供了权限机制，允许一个le比另一个le的特权高。
+ LE可以根据需要将内存划分domain，并用domain-id来标记，其中domain-0存放所有程序的代码，domain1-n分别存放程序1-n的数据。
### 组件
+ Secure monitor
	+ 被无条件信任，可以访问所有内存区域，代码和数据都在domain-0中
	+ 负责创建新的le和动态内存管理
+ Usage model
	+ 这种抽象方便程序员将其所写的程序进行分离，将敏感的信息及代码放到一个le中，将其他库放到另一个le中，并且声明彼此之间交互的接口。而LE则会在这些le之间创建gate，简称leg，gate用于传递两个le之间的控制流以及切换身份。
### 威胁模型
+ LE继承了SGX的威胁模型，即假设对手可以完全控制除SGX enclave之外的所有软件，包括操作系统。
+ 除此之外，LE还假设enclave内部有一些代码是不可信的，因为不排除有一些第三方库存在漏洞等问题。
+ 对于enclave中的le来说，它需要信任SGX硬件、安全监视器以及比它自己的特权等级更高的le(如果存在的话)。除此之外不需要信任其他任何软件。
+ LE假定每个le不会暴露其秘密，也不会采取任何措施来消除这类bug；同时也不会考虑硬件上的攻击，例如硬件bug和侧信道攻击等。
## 架构扩展
简单的将MPK和SGX结合并不能够有效抵挡攻击。关键原因是，MPK依靠操作系统修改domain-id来发挥作用，但是SGX并不信任操作系统，认为攻击者有可能完全操控操作系统。因此简单的将MPK引入SGX中会造成非常大的安全隐患，需要进行硬件扩展。
### 硬件扩展三原则
+ 创建enclave时，对每一个domain-id都应进行验证(远程用户的权利)。
+ 在运行过程中，不再允许操作系统修改domain-id，操作系统只能执行添加指令。
+ MPK不能在le创建时被禁用。
### 将domain-id视为安全元数据
要安全地使用MPK的特性，就必须将domain-id视为安全元数据。将数据全都放到enclave中进行管理。

具体来讲，使用PAGEINFO结构来作为添加domain-id的参数结构，PAGEINFO包含四个字段：指定元数据、enclave页的数据，第三个字段指向另一个结构，名为SECINFO，该结构包含enclave页的信息包括读写以及执行权限等。同时使用此结构来存储domain-id。
### 验证domian-id
+ 验证时间：在根据EPCM进行地址转换时。
+ 验证者：远程用户、MMU
+ 验证方式：远程用户可以查看当前le的domain-id。MMU会验证le访问内存的合法性。同时，进一步检查，即将从TLB中检索的domain-id与ECPM中的domian-id是否相等，相等才能合法访问内存。通过这一步检查，即使攻击者可以修改页表中的le页的domain-id，但是其无法修改EPCM中的domain-id(**为什么**，**因为EPCM在单独的domain中，即使le中因为有漏洞被攻击者控制，攻击者也不能通过篡改domain-id来达到攻击目的**)，因此两者会不相等，MMU会检测到，从而防止了修改domain-id产生的攻击。
### 在EPC页交换期间保存和恢复域id
在EPC页被换出时，针对domain-id生成MAC值，在恢复时，校验mac值来保证domain-id不被修改。
### 检查le迁移过程中MPK是否一直工作
在SECS中添加一位(PKE)位，用于指定le是否据需要进行MPK域的权限检查。同时，LE还设置了EENTER指令，该指令用于检查在SECS中设置PKE时是否还设置了CR4.PKE。如果没有，将拒绝让这一个le执行。在le执行时，操作系统无法清除CR4.PKE，因为CPU以le模式运行，此时操作系统没有被运行。

## 动态地管理encalve页
SGXv2支持将enclave页动态地加入到正在运行中的enclave中，操作系统可以使用EAUG指令实现上述功能，参数结构与EADD差不多，都包含PAGEINFO。enclave可以发送EACCEPT或者EACCEPTCOPY指令来接收enclave页，从而使其生效。

LE依然可以扩展上述三条指令来使enclave依赖不受信任的操作系统来添加页面。

同时，SGXv2还提供了EMODPE指令，用于扩展页面的访问权限。EMODPE使用SECINFO来调整EPCM的内容。

威胁：

1. **攻击者可以使用EAUG指令将恶意代码和TCS后门注入到le中，如果被一个遭到破坏的le接收并且执行，则会造成危害。** 

2. **一个恶意的le可以通过执行EMODPE并且要求OS修改页面表中填写的domain-id来修改其他le的domain-id，然后它就可以访问任何敏感数据了。**
### 应对方法1：在le之间实现权限分离
设立一个特权表，里面保存着每个le的权限，这些权限决定了le可以执行哪些指令。总共有三种特权，三种特权之间使用页表结构进行连接。第一层只有一个页面，地址在SECS中指定，拥有最高特权。第一级页面可以指向512个二级页面，同样，一个二级页面可以指向512个三级页面。在三级页面，每个条目存储与le基地址的相对偏移量。
+ PKRU值被分成四部分
	+ 前两个部分(每个部分取9位)被用作定位下一级页面的索引；第三部分定位最后一级页面中的1字节许可；第四部分都是1，标识不能访问domain-0
+ SECS中有超过3000个保留字节，足以存储能力表的基本偏移量。
+ 能力表由SECS中的一个控制位来控制开启与否。
+ 当le执行敏感指令（例如动态分页）时，cpu会检查能力表，这会涉及到多次内存访问。但是，敏感指令通常不会很多，因此性能几乎不会下降。
## 硬件扩展总结
1. 引入MPK内存隔离机制，实现enclave内le之间的隔离
   + 但是原生的MPK收到不信任的操作系统的控制，因此需要进行改进。
   + LE通过在enclave初始化时记录domian-id，以及让MMU在enclave运行时自动检查内存访问内存信息，从而剥夺了不信任的操作系统修改domain-id这一能力。
2. SGX支持切换enclave页面，因此需要在页面交换期间保持domain-id不变，在删除和重新加载enclave页面时需要保存和恢复domain-id。
3. 为了支持动态分页和domain-id修改，扩展了EAUG、EMODPE和accept指令来指定enclave页面的domain-ID。
4. 增加了安全检查，防止不受信任的操作系统在进入硬件enclave时禁用MPK；并允许在执行四个敏感的ENCLU指令时根据每个enclave能力表进行细粒度的特权分离。
5. 硬件扩展使用微码（microcode）技术可以集成到现有的SGX硬件设计中。

# 软件设计
LE的软件部分分为enclave内部和外部。

内部代码是基于SGX的扩展，负责生成程序员指定的enclave内存布局和一些用于简化le开发的代码库。

外部代码主要是用来设计安全监视器和le的门。

## 编程模型
+ LE开发人员仍可以使用与enclave相同的IDL来定义le与不受信任的应用程序之间的接口。
  + 接口分为两类：公共的和专用的
+ LE允许构建互相不信任的le，两个le可以通过共享内存域直接交换数据，而除这两个了le之外的任何le或者操作系统都无法访问共享内存。
+ LE允许分层构建le，一个le内部可以有多个le，他们之间的内存域访问规则与上述一致。
+ LE保存了与enclave相同的IDL，同时还给开发人员提供了声明指令执行权限的能力。默认情况下，le不能执行敏感指令。
+ LE会在初始化时自动设置domain-id、自动生成le之间交互的le门、自动配置能力表还有内存分区划分。

## PKRU绑定
上述变成模型成立的一个关键是domain-id的设定，而domain-id存储在PKRU寄存器中。因此必须保证该寄存器的安全，否则，攻击者可以通过修改寄存器中的内容来访问其他内存或执行不允许的指令来提升特权。

四种恶意修改PKRU的方式：
+ 攻击者在执行EENTER指令时不改变PKRU值，以便le可以继承在SGX enclave外部配置的PKRU值。
+ 异步enclave退出程序会自动将PKRU值保存到SSA（State Save Area）中，在执行ERESUME时恢复。若SSA被攻破，PKRU值依然会被修改。
+ 使用WRPKRU指令来修改PKRU值。
+ XRSTOR/XRSTORS指令可以恢复处理器的扩展状态，包括PKRU寄存器的内容。

相应的防范措施：
+ LE将PKRU配置时间提前到控制流转移到le之前，这样EENTER指令只负责将控制流传给le即可，PKRU的值由LE管理。
+ 将所有的SSA页驻留在domian-0内存区域，如此，SSA只能被安全监视器访问。
+ LE通过扫描和重写来保证le中没有WRPKRU指令。
  + 逐字节扫描来定位指令，若存在，将其替换为语义相同的指令，这样可以防止面向返回编程的攻击。
+ 与上一方法类似，使用二进制检查来保证le中没有类似的指令。

LE通过上述措施可以确保le在执行过程中始终绑定到指定的PKRU上。

## 安全监视器
+ 每一个硬件enclave包含一个安全监视器，用来确保每一条程序安全执行。是可信的。可以访问所有的内存域并且执行所有与SGX兼容的指令。
+ 初始化时有两个职责：远程认证和在le中填充随机数作为标记
  + 远程认证：想远程用户报告整个硬件enclave的监控结果
+ 运行时提供两种服务：
  + 动态的enclave页面管理，为不同的le分配所需的domain-id的enclave内存页
  + 动态管理le，包括构建新的le，回收现有的le，添加新的le门以及分配新的共享内存域，同时还更新enclave能力表。

# 实现细节
LE是基于SGX实现的，因此需要扩展以及修改原本的SGX库。

主要修改的库有以下：
+ Occlum：支持在其上运行多个任务。修改后可以利用le的抽象来运行不同的任务。
+ Graphene-SGX，支持运行未经修改的Linux程序，通过创建多个硬件enclave来运行多进程应用程序。修改后可以创建le，而不是硬件enclave。

此外，文章还讨论了LE实现的硬件开销、性能开销以及内存开销。结论是
+ 微码技术使得SGX易于扩展，引入LE的硬件开销大大降低。
+ enclave交换页时的性能开销几乎可以忽略不计，只有在交换表的时候才会显著增加cpu开销，但是，如果安全监视器只监控敏感指令，cpu开销就又可以变得可以忽略不计。
+ LE扩展的指令大部分是利用的SGX中保留字节，因此内存开销也可以忽略不计。

# 安全分析
## 不受信任的操作系统不能对le发动攻击
因为le是运行在SGX enclave中，这也就意味着不受信任的软件包括操作系统不能访问le的内存和运行状态。

## 恶意的le不能对其他le发动攻击
一个le a可能与另一个同样在SGX enclave中的恶意le b一同运行。此时恶意le有四种攻击方式：
+ b直接访问a的内存页
  + 不可行。因为MPK技术将a与b的内存页隔离，无法访问。
+ b尝试访问a在SSA中的执行状态
  + 不可行。因为LE讲SSA存放在domain-0域中，le无法访问。
+ b尝试修改决定域访问权限的PKRU寄存器
  + 不可行。因为LE包装了PKRU寄存器，le无法访问。
+ b可能会故意触发异常，导致整个enclave崩溃，从而导致DoS攻击。
  + 不可行，因为LE在设计上要求操作系统将异常报告给安全监视器，由安全监视器使用SSA定位b。从而精准处理异常，避免整个enclave崩溃。当操作系统不配合时，可能会发生DoS。但在没有LE的情况下，操作系统本身就可以发动DoS攻击。
# 性能表现
LE设计的初衷就是既能增加安全性，又能提高性能，因此本章从四个方面讨论LE给SGX带来的哪些性能优化。
## LE在le创建和le之间的通信方面有多快
## LE在enclave内进行隔离对应用程序产生了多少开销
## LE能给现有的LibOS带来多少性能优化
## LE能给FaaS场景带来多少性能优化



## 不足
+ 没有设计出enclave内存访问检查(因为模拟器没有模拟它)