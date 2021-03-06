线性地址的保护
================================================================================

在Intel IA-32架构中，进程保护体现在对进程内存空间的保护，进程内存空间的保护是由线性地址保护，
物理地址保护实现的。
现代计算机基本上都沿用冯.诺依曼体系。这个体系中，指令，数据都存储在同样的内存中。内存设计为随机
访问存储器（RAM），也就是说可以在内存空间任意读、写数据或指令。在没有保护模式之前，从物理内存的
角度看，不同用户程序的代码和数据没有物理上的差异，都由一连串的0、1组成。至于什么地方可以读写，
什么地方不可以读写，并没有明确的物理限制，没有什么机制能阻拦不同用户程序间的相互干扰。
支持实时多任务首先遇到的问题就是，如何保证每个进程在运行过程中能够与其他进程互不干扰，也就是
保证进程之间不能相互访问代码、数据，更不能相互覆盖代码、数据。

进程线性地址空间的格局
--------------------------------------------------------------------------------

Intel IA-32 CPU架构设计为，只要打开PE、PG，所有在计算机中运行的程序使用的只能是线性地址，
然后将线性地址转换到具体的物理地址，转换是由CPU中的MMU根据页目录表、页表、页的设定，
由硬件自动实现的。
线性地址就是CPU可以寻址的地址。在IA-32体系架构下，32位地址总线的线性地址空间范围是0～4GB。
为了在线性地址层面分隔进程的内存空间，Linux 0.11采取的总体策略是将4 GB的线性地址空间分成
互不重叠的64等份，每份64 MB，每个进程一份，最多同时开启64个进程。要求进程无论怎样执行，
都不能跨越起点和终点。这样进程的线性地址空间彼此不重叠，实现在线性地址层面对进程内存空间的保护。
这是整个Linux 0.11中线性地址空间设计的最大格局，所有针对线性地址空间方面的设计都服从于这个格局。
task[64]是这个格局的基点，所有进程登记、注销都只由它统一管理。每个进程只有在task[64]中定位后，
才能在线性地址空间中进行安排。操作系统根据task[64]的项号nr在GDT中找到对应的LDT。task[64]起到了
控制进程总量，关联进程与GDT中的LDT、TSS的关键作用。虽然在操作系统代码中规划出了64等分
4 GB线性地址空间的格局，但仅此能否对进程跨越64 MB线性地址空间的访问做出有效的阻拦？
也就是说，能否确保进程的线性地址空间彼此不重叠？操作系统内核虽然做出了进程的线性地址空间的格局，
但却无法仅仅依靠算法、控制逻辑阻拦进程的跨界访问。原因是CPU一次只能执行一条指令，执行进程的指令
就不能执行内核的指令。所以，不论内核有多么漂亮的控制越界算法，当进程执行跨界访问时，内核的控制
越界算法都不在执行状态，无法控制进程的越界行为。软件的方法无效，那只能依靠硬件方法。
Intel IA-32架构专门设计了基于CPU硬件的控制进程访问越界的方法。

段基址、段限长、GDT、LDT、特权级
--------------------------------------------------------------------------------

Intel IA-32架构对进程线性地址空间的保护是基于段的。历史上，由于函数（子程序）链接的需要，
发明了在内存中划分段的方法，所有程序的设计都是基于段的。线性地址空间是一维的，所以只要
看住一段线性地址空间的两头，让程序在段空间里执行，别越界，它就不会干扰到其他段，也就不会
干扰到其他进程。Intel早期的CPU为了降低成本，只设计了看住段起始位置的段头寄存器，并没有
设计看住段结束位置的段尾寄存器。为了兼容早期CPU，Intel IA-32架构在段（头）寄存器中设计了段限长，
等效于设计了段尾寄存器，用一个寄存器巧妙地起到了两个寄存器的作用。Linux 0.11操作系统利用
Intel IA-32 CPU架构提供的段基址、段限长有效地阻拦了段内跳转中有意无意的越界行为。
比如，jmp X，如果这个X很大，超过段限长，硬件会阻止类似指令的执行，并立即报出GP错误。
对于进程代码中跨越段边界的ljmp，段基址、段限长不能阻拦.

Linux 0.11是用什么方法拦截非法跨越进程边界的访问动作的呢？非法跨越进程边界有两种情况：
* 一种是从一个进程非法跨越到另一个进程;
* 另一种是从一个进程非法跨越到内核。

### 从一个进程非法跨越到另一个进程

从一个进程用ljmp指令非法跨越到另一个进程，从IA-32架构的角度看，两个进程的代码段都是3特权级，
Linux 0.11的所有进程都安排在一个4 GB的线性地址空间，所以允许这个ljmp指令的执行，
段基址、段限长此时起不到阻拦非法越界的作用。Linux 0.11采用的是通过LDT的设计，阻拦非法ljmp指令
的执行。Linux 0.11的GDT、LDT的设计: 64个进程，每个进程占用GDT的两项，一项是TSS，另一项就是LDT。
所有进程的LDT段的设计是完全一样的，每个LDT都有3项，都是第一项为空，第二项是进程代码段，第三项是
进程数据段。当一个进程的代码中有非法的跨进程跳转的指令时，比如，ljmp指令执行时，该指令后面的
操作数是“段内偏移段选择子”。代码段的段选择子存储在CS里面。仔细考察一下，可以看出Linux0.11中所有
进程的CS的内容都是一样的，用二进制表示的形式都是0000000000001111。CPU硬件无法识别是哪一个进程的
CS，也就无法选择段描述符，只能默认使用当前LDT中提供的段描述符，所以类似ljmp这样的段间跳转指令，
无论后面操作数怎么写，都无法跨越当前进程的代码段，也就无法进行段间跳转，最终只能是执行到本段。
由此可见，Linux 0.11的LDT的设计看似重复，其实颇具匠心。
试想一下，如果Linux 0.11不是这样的设计，而是将所有进程的代码段描述符都直接写到GDT中。对所有进程
共用一个4GB线性地址空间的Linux 0.11而言，进程代码中的非法跨越进程的跳转指令就可以不受阻拦地执行。
按照这个思路，可以发现，Linux 0.11在防止非法跨越进程的长跳转指令方面，略显粗糙。TSS段、LDT段的
段限长是一样的，都是104B。这个段限长对TSS来说是合适的，对LDT来说就太长了。LDT只有3项，每项8字节，
一共只有24 B。从进程0的INIT_TASK可以看出LDT后面紧跟着TSS，这个数据结构在父子进程创建机制创建进程
时会向后遗传，所有进程的task_struct里面的LDT、TSS都是一样的。如果进程代码中有这样的代码：

```
ljmp 偏移,CS（CS的值是0000000000111111，即3特权级，LDT表中的第8项）
```

这样的指令仍然会在段内执行，而从LDT基址往后偏移到“第8项”的数据内容不可预知，出现的错误也不可预知。
但无论是哪些错误，都无法跨越进程的边界，也无法改变LDT。反过来看这个问题，就算非法的进程跨越的
长跳转指令能够执行，也只是代码跳转过去，数据段、栈段都没跟着变换过去。代码在一个进程的段中执行，
数据和栈却在另一个进程中，在这种条件下，代码通常会执行死了。从这个反向角度，我们可以更深刻地领悟
到为什么正常的进程切换，是用TSS将代码段、数据段、栈全部变换过去的。要想进行正确的进程切换，
必须将进程的执行状态成套、完整地保存，并成套、完整地切换到另一个进程。以上讲解的是用ljmp
从一个进程非法跨越到另一个进程的情况，下面讨论用ljmp从一个进程非法跨越到内核的情况。

### 从一个进程非法跨越到内核

用户进程代码段的特权级都是3，内核的特权级是0，IntelIA-32架构禁止代码跨越特权级长跳转，3特权级
长跳转到0特权级是禁止的，0特权级长跳转到3特权级同样是禁止的。所以这样的非法长跳转指令会被
CPU硬件有效阻拦，进程与内核的边界得到有效的保护。0特权级的内核代码可以访问3特权级的进程数据，
3特权级的进程代码不能访问0特权级的进程代码。这些禁止都是非常刚性的硬件禁止。从上面的讲解可以看出，
Linux对GDT、LDT的设置，有效地阻止了非法跨越进程边界的访问。用户进程是否可以设置GDT、LDT，以使
自己写的非法跨越边界的指令能够执行？答案是否定的，因为Linux 0.11将GDT、LDT这两个数据结构设置
在内核数据区，是0特权级的，只有0特权级的代码才能修改设置GDT、LDT。那么，用户进程是否可以在
自己的数据段按照自己的意愿重新做一套GDT、LDT呢？如果仅仅是形式上做一套和GDT、LDT一样的数据结构，
没有什么不可以，但起不到真正的GDT、LDT的作用。真正起作用的GDT、LDT是CPU硬件认定的。这两个数据
结构的首地址必须挂接在CPU中的GDTR、LDTR上，运行时，CPU只认GDTR、LDTR指向的数据结构，其他数据结构
就算起名字叫GDT、LDT，CPU也一概不认。
Linux 0.11内核在进程的初始化阶段，就将GDT、LDT挂接到了CPU中的GDTR、LDTR上了。用户进程能否也将
自己制作的GDT、LDT挂接到GDTR、LDTR上？答案是否定的，因为对GDTR、LDTR进行设置的指令LGDT、LLDT只能
在0特权级下执行。到此为止，我们可以看清楚Linux操作系统依托Intel IA-32架构设计的段基址、段限长、
GDT、LDT、特权级这一整套硬件保护机制，在线性地址层面建立了牢固的进程间、进程与内核间的边界，
有效地防止了非法跨越边界的操作。进程间合理的跨越边界的数据沟通如何解决？进程间切换及进程需要
跨越边界获得操作系统内核合理的支持应该如何操作呢？
Linux 0.11中的进程间切换是在schedule()中完成的，其技术路线很像任务门（但没有用任务门），
是在0特权级下，用ljmp指令直接跳转到要切换的进程的TSS（指令跳转到数据，似乎有些奇怪，实际上
后面有CPU硬件做了大量的工作，最终跳转到目标进程的代码段），实现进程切换的。进程要想获得内核
的支持（如读盘），Intel IA-32架构提供了中断门技术，支持由3特权级代码段翻转到0特权级代码段执行。
注意，这个翻转不是普通的跳转，需要经过CPU的硬件中断机制，不同于平坦的、普通的内存寻址跳转。获得
内核的支持后，再由iret指令经过CPU硬件，从0特权级的内核代码段翻转到3特权级的进程代码段继续执行。
