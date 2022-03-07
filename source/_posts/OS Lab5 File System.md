---
title: OS文件系统与驱动设计（个人OS课设笔记节选）
subtitle: 不知道为什么阅读量这么高……
date: 2021-07-06 10:00:00
---

# OS课程设计 文件系统的实现与驱动设计

实验的目的在于：

1. 了解文件系统的基本概念和作用
2. 了解普通磁盘的基本结构和读写方式
3. 了解实现设备驱动的方法
4. **掌握并实现文件系统服务的基本操作**
5. 了解微内核的基本设计思想和结构

为了避免同志们坐享其成，所有代码均取自[login学长的开源代码]([login256/BUAA-OS-2019: 北航OS课课设代码 (github.com)](https://github.com/login256/buaa-os-2019/))，为方便理解，做少量注释，可以理解该篇为Lab 5任务导读（x

[TOC]

## 什么是文件系统？

文件系统的出现是为了更好地管理在不易失存储介质上存放的数据，而通常来说这种外部不易失的存储设备就是磁盘。文件系统将磁盘上的数据抽象化，使得用户能够很方便地访问数据而无需关心具体和磁盘之间的交互。

注意，**文件系统是高度抽象性的**，它是管理数据的抽象的界面，而背后实际的数据存储形式是对用户来说不可见的。因此，文件系统一方面需要面向复杂多样的数据存储媒介，一方面需要面向用户提供简洁统一的接口。

同时，**文件系统也可以不仅是文件系统**，诸如proc这样的虚拟文件系统还可以实现Windows中任务管理器的功能，这取决于你如何定义一个文件是什么。

### 有关本次实验的具体问题

**本次实验中提到的文件系统既是指磁盘文件系统，又是指操作系统上的文件系统**。注意，磁盘文件系统是在磁盘驱动器上而言的，而操作系统的文件系统是针对操作系统而言的，两者的**结构即使一致，其在磁盘和内存上的表示也会有一定差异**，需要注意区分。

本次我们需要分三步实现文件系统：

1. 实现磁盘的用户态驱动程序
2. 实现磁盘上和操作系统上的文件系统结构，并在调用驱动程序的基础上实现文件系统操作相关函数
3. 提供接口和机制使得用户态下也能够使用文件系统

### 一些基本概念的补充

为什么说操作系统和磁盘上文件系统不同也能进行正常使用：举例，Linux使用的是VFS文件系统，但是可以与Ext4等多种文件系统的磁盘驱动器正常通讯。理论上，磁盘文件系统不同的磁盘上，数据的组织方式不同，按统一的方式去访问肯定不行。但是Linux提出的VFS（Virtual Filesystem Switch）是一个虚拟的文件系统，其将其他不同文件系统分别进行解释，然后以统一的方式进行管理，由此实现了对于用户来说完全一致的效果。**这恰恰反映了文件系统的抽象性**。

磁盘驱动程序：**位于操作系统中的一段代码**，与操作系统高度相关，描述了对应磁盘驱动器的信息和提供操作接口。操作系统需要通过驱动程序才能与磁盘驱动器通信。

磁盘与磁盘驱动器：磁盘是一个物理结构，用于存储信息，而磁盘驱动器是用于控制磁盘旋转、读取的机构。操作系统需要通过驱动程序才能和磁盘驱动器交流，而磁盘驱动器再去磁盘上寻找对应信息。

## 三点几嘞，写个磁盘驱动先啦

本次实验中我们使用**内存映射I/O技术（MMIO）**来实现一个IDE磁盘的驱动。IDE具体的意思是Integrated Driver Electronics，字面意思指这种磁盘的控制器和盘体集合在一起，但是SATA磁盘也是这样的结构，二者主要区别在于接口串行和并行。不过这和我们的实验没有什么关系。

另外需要说明的一点是，本次的驱动程序**完全运行在用户态下**，这是需要两个新的系统调用`sys_write_dev`和`sys_read_dev`支持的，它们实现了用户态下对设备的读写。

#### sys_write_dev

```c
// lib/syscall_all.c
int sys_write_dev(int sysno, u_int va, u_int dev, u_int len)
{
	int cnt_dev = 3; // 支持三个设备
	u_int dev_start_addr[] = {0x10000000, 0x13000000, 0x15000000}; // 每个设备对应的物理地址
	u_int dev_length[] = {0x20, 0x4200, 0x200}; // 每个设备对应的空间长度
	u_int target_addr = dev + 0xa0000000; // 计算出对应的内核虚拟地址
	int i;
	int checked = 0;

	//do check:
	if (va >= ULIM)
	{
		return -E_INVAL;
	}
	for (i = 0; i < cnt_dev; i++)
	{
		if (dev_start_addr[i] <= dev && dev + len - 1 < dev_start_addr[i] + dev_length[i])
		{
			checked = 1; // 表示确认了往这个地址写是在允许范围内的
			break;
		}
	}
	if (checked == 0)
	{
		return -E_INVAL;
	}

	//do copy
	bcopy((void *)va, (void *)target_addr, len); // 从va处向目标处复制，完成写入
	return 0;
}

```

#### sys_read_dev

```c
// lib/syscall_lib.c
int sys_read_dev(int sysno, u_int va, u_int dev, u_int len)
{
	int cnt_dev = 3;
	u_int dev_start_addr[] = {0x10000000, 0x13000000, 0x15000000};
	u_int dev_length[] = {0x20, 0x4200, 0x200};
	u_int target_addr = dev + 0xa0000000;
	int i;
	int checked = 0;

	//do check:
	if (va >= ULIM)
	{
		return -E_INVAL;
	}
	for (i = 0; i < cnt_dev; i++)
	{
		if (dev_start_addr[i] <= dev && dev + len - 1 < dev_start_addr[i] + dev_length[i])
		{
			checked = 1;
			break;
		}
	}
	if (checked == 0)
	{
		return -E_INVAL;
	}

	//do copy
	bcopy((void *)target_addr, (void *)va, len); // 和sys_write_dev 唯一的不同
	return 0;
}

```



### 内存映射I/O MMIO

硬件设备上具有一些寄存器，CPU通过读写这些寄存器来和硬件设备进行通信，因此这些寄存器被称为**I/O端口**。而这些寄存器并不是直接以寄存器的方式展现给CPU的，而是**映射到内存的某个位置**。当CPU读写这块内存的时候，实际上读写了相应的I/O端口。而操作系统怎么知道不同的外设映射到具体哪个位置呢？实际上这需要在系统启动后，由BIOS告知。

在MIPS结构中，这种机制更为简单。其在kseg0和kseg1段里从硬件的层次可预知地实现了物理地址到内核虚拟地址的转换，这使得所有I/O设备都可以存放在这段空间里，并通过确定的映射关系计算出对应的物理地址。而我们**用kseg1来进行转换，而不用kseg0**，因为kseg0需要经过cache缓存，导致不可预知的问题。

进一步，在我们的实验中，模拟器中**I/O设备的物理地址是完全固定的**，我们的驱动程序就只需要对特定内核虚拟地址进行读写即可。

### 驱动程序编写

由于驱动程序的编写实际上就是对特定地址进行读写，我们需要清楚两个主要问题：

1. 往哪里写？从哪里读？
2. 读/写对应的数据的意义是什么？

这两个是和硬件有关的，所幸的是指导书中已经给了出来，Gxemul中IDE磁盘的基址是0x13000000（注意这是物理地址）。

| 偏移量          | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| 0x0000          | 向这个地址写入的32位u_int，将指定下一次读/写操作相对于磁盘起始地址的偏移量的**低32位**（以字节计） |
| 0x0008          | 向这个地址写入的32位u_int，将指定下一次读/写操作相对于磁盘起始地址的偏移量的**高32位**（以字节计） |
| 0x0010          | 向这个地址写入的32位u_int，将指定具体的磁盘ID（本实验中这个值始终为0） |
| 0x0020          | 向这个地址写入的8位u_char，将指定需要进行的操作类型，0为读，1为写 |
| 0x0030          | 从这个位置读取的8位u_char，将反映上一次操作的执行状态，0表示失败，否则为成功 |
| 0x4000 - 0x41ff | 进行读操作时，当读取成功后，从这个区间读出的512个字节，将反映从指定位置读出的数据；进行写操作时，当写入成功，这512个字节将被写入指定位置 |

当对其进行操作时，正如我们上文提到的，需要通过**kseg1区进行地址转换**，由此我们**需要访问0xB3000000**，从而利用kseg1区的地址转换机构成功访问0x13000000。由此我们成功解释了如何通过MIPS指令就能操作磁盘，接下来我们需要以此编写具体的操作函数。

#### ide_write

```c
void
ide_write(u_int diskno, u_int secno, void *src, u_int nsecs)
{
	u_int offset_begin = secno * 0x200; // 根据起始扇区号计算出起始偏移量，一个扇区512个字节
	u_int offset_end = offset_begin + nsecs * 0x200; // 根据读取扇区数量计算出终止偏移量
	u_int offset = 0; // 初始化循环递增量
	u_int dev_addr = 0x13000000;
	u_char status = 0;
	u_char write_value = 1;

	writef("diskno: %d\n", diskno);

	while (offset_begin + offset < offset_end) {
        // 每个循环操作512个字节，即一个扇区
		u_int now_offset = offset_begin + offset;
		if (syscall_write_dev((u_int)&diskno, dev_addr + 0x10, 4) < 0)
		{
			user_panic("ide_write error!");
		}
		if (syscall_write_dev((u_int)&now_offset, dev_addr + 0x0, 4) < 0)
		{
			user_panic("ide_write error!");
		}
		if (syscall_write_dev((u_int)(src + offset), dev_addr + 0x4000, 0x200) < 0)
		{
			user_panic("ide_write error!");
		}
		if (syscall_write_dev((u_int)&write_value, dev_addr + 0x20, 1) < 0)
		{
			user_panic("ide_write error!");
		}
		status = 0;
		if (syscall_read_dev((u_int)&status, dev_addr + 0x30, 1) < 0)
		{
			user_panic("ide_write error!");
		}
		if (status == 0)
		{
			user_panic("ide write faild!");
		}
		offset += 0x200;
	}
	//writef("ide_write %x %s\n", offset_begin, src);
}
```

#### ide_read

```c
void
ide_read(u_int diskno, u_int secno, void *dst, u_int nsecs)
{
	// 0x200: the size of a sector: 512 bytes.
    // 除了读写行为不同，其他逻辑和ide_write一致
	u_int offset_begin = secno * 0x200;
	u_int offset_end = offset_begin + nsecs * 0x200;
	u_int offset = 0;
	u_int dev_addr = 0x13000000;
	u_char status = 0;
	u_char read_value = 0;

	while (offset_begin + offset < offset_end) {
		u_int now_offset = offset_begin + offset;
		if (syscall_write_dev((u_int)&diskno, dev_addr + 0x10, 4) < 0)
		{
			user_panic("ide_read error!");
		}
		if (syscall_write_dev((u_int)&now_offset, dev_addr + 0x0, 4) < 0)
		{
			user_panic("ide_read error!");
		}
		if (syscall_write_dev((u_int)&read_value, dev_addr + 0x20, 1) < 0)
		{
			user_panic("ide_read error!");
		}
		status = 0;
		if (syscall_read_dev((u_int)&status, dev_addr + 0x30, 1) < 0)
		{
			user_panic("ide_read error!");
		}
		if (status == 0)
		{
			user_panic("ide read faild!");
		}
		if (syscall_read_dev((u_int)(dst + offset), dev_addr + 0x4000, 0x200) < 0)
		{
			user_panic("ide_read error!");
		}
		offset += 0x200;
	}
	//writef("ide_read %x %s\n", offset_begin, dst);
}
```

## 呐，文件系统始まる！

文件系统，从根本上来说是一种规范。我们实现的磁盘驱动只是往磁盘读写特定的二进制数据，而不管这些数据是如何组织的，也不管这些数据的意义是什么。而文件系统就是一套说明这些数据组织的逻辑，更实际一点，就是如何划分和解释磁盘的空间，这也是**文件系统结构**的根本意义。

我们的MOS的文件系统将磁盘按下面这种方式进行划分：

![磁盘划分情况](img/blog/2028226-20210523220512737-1714373405.png)

我们将磁盘描述为若干个**磁盘块（Block），每个Block大小为4KB**。

对于N个磁盘块的磁盘，其**第一个磁盘块用于启动和存放分区表**，第二个磁盘块整个都被分给了**超级块（super block）**。它超级在哪里呢？其**负责存放整个文件系统的基本信息：磁盘大小、根目录位置等**。

### 相关机制和数据结构设计

**这里课程组有一个锅，其指导书混淆了磁盘块（Block，4KB）、扇区（512B）和文件控制块（sizeof(struct File) = 256B）。如果你是学弟学妹的话，可以留意一下指导书有没有改**。~~我已经向助教提出了这个问题，学弟学妹们可以注意一下~~。

#### 超级块 Super

我们使用一个数据结构来描述超级块：

```c
struct Super {
  u_int s_magic; // Magic number: FS_MAGIC，用于识别文件系统类型
  u_int s_nblocks; // 总磁盘块数量，MOS中为1024
  struct File s_root; // 一个文件控制块，表示根目录位置
};
```

注意到虽然超级块的数据结构并不大，但是磁盘中整整分配给它了一整个磁盘块（4KB大小）。

#### 文件控制块 File

File就是我们定义的文件控制块：

```c
struct File {
  u_char f_name[MAXNAMELEN]; // filename
  u_int f_size; // file size in bytes
  u_int f_type; // file type，分为FILE_REG（普通文件）和FILE_DIR（目录文件）
  u_int f_direct[NDIRECT]; // 文件的直接指针，其数值表示磁盘中特定磁盘块号，NDIRECT = 10，即可代表至多10个磁盘块，共40KB的文件大小
  u_int f_indirect; // 表示一个间接磁盘块的块号，其中存储了指向文件内容的磁盘块的直接指针（此处我们规定不使用间接磁盘块中的前10个直接指针）
  struct File *f_dir; // 指向文件所属的目录文件
  u_char f_pad[BY2FILE - MAXNAMELEN - 4 - 4 - NDIRECT * 4 - 4 - 4]; // 占位，为了使得一个struct File恰好占据256字节（BY2FILE = 256）
};
```

两个File结构体恰好占一个扇区。

#### 磁盘块管理位图 nbitblock 

我们在Lab 5中使用位图法来管理磁盘块，这有别于使用链表法进行管理。在本实验中，**相应位为1表示空闲，反之为占用**。注意，我们在这里使用位图法，也就是每一位都是对应一个磁盘块的。而实际上`nbitblock`是一个字节数组，其中每个元素都是8位，称之为一个**位图块**。因此在初始化时，注意**有的位图块（8位）的后面几个位不一定有实际的磁盘块与之对应**，所以不能将其初始化为空闲。

#### 文件系统进程空间 与 块缓存机制

我们的**文件系统是在用户空间内的一个进程**，其拥有4GB的进程空间，而这个空间是我们实现和磁盘数据交流的一个重要中介。我们将所有磁盘块都按一定规则映射到这各进程空间内，而当我们需要往磁盘写数据时，就从这个进程空间取数据，而读数据时就往这个进程空间存放数据。

注意这个**文件系统进程空间和传统的进程空间不同**，其将DISKMAP\~DISKMAP+DISKMAX（0x10000000\~0x4fffffff）这一大段空间作为缓冲区，当对硬盘上特定磁盘块Block[id]进行读写时，其唯一对应于这块缓冲区中一块512字节的空间，需要写入的数据会存放在这块空间里等候写入，读出来的信息也会放在这块空间里等候发送给用户进程。这就是**块缓存**。

而这个进程空间本身又是由虚拟内存管理来实现的，每个block在没有被触发时只是在理论上占有一个虚拟空间，而当使用时会动态分配一个物理页给它（我们会发现一个磁盘块大小恰为一个物理页）。

由此我们的文件系统实际上与三个数据结构有较大联系：

1. 磁盘块管理位图 nbitblock
2. 磁盘块中的文件控制块 File
3. 文件系统进程空间映射关系（实际上是管理其的页表结构）

### 访问文件的中经历了什么？

访问一个文件，首先要找到其对应的文件控制块结构。该过程首先需要经过从根目录文件的逐级查找。而找到对应文件控制块后，则可以通过其中的指针找到对应的磁盘块，从而利用驱动程序访问到指定数据。

而**文件系统在我们的操作系统中作为独立的进程**存在，其通过进程间IPC的方式来服务于用户进程，其为服务所开放的函数中存储在fs/serv.c中。用户进程需要调用user/file.c中的函数实现文件系统操作，而其底层调用user/fsipc.c中的函数来实现和文件系统进程的IPC。

#### 代码的分布和调用逻辑

![文件系统代码调用逻辑](img/blog/2028226-20210523220445682-418539770.png)

以打开一个文件并获取其数据为例，用户进程需要调用file.c中的`open`函数，其中调用了fsipc.c中的`fsipc_open`和`fsipc_map`。

用户进程调用`fsipc_open`来将文件路径path和打开方式mode打包进一个特殊的数据结构中，然后通过IPC发送给文件系统，文件系统返回一个用于描述该文件的文件描述块（struct Fd，其中包含对应的文件控制块id、文件大小等）。

用户进程调用`fsipc_map`来通过指定文件控制块fileid和偏移量（以字节计），来获取指定位置的磁盘块中的数据。其打包发送给文件系统进程后，文件系统通过fileid找到对应文件，再通过offset找到对应磁盘块。磁盘块数据恰好1页大小，正好能够通过我们的IPC机制通讯传送回到用户进程。有趣的是这里面磁盘块数据涉及多次映射，一次是从磁盘中映射到文件系统进程的指定位置，一次是在IPC过程中从文件系统映射回用户进程。

#### 文件系统进程中的函数

##### fs.c

fs.c文件定义了有关磁盘块和文件的一系列操作，主要包含两个大方面：

###### 磁盘块管理：

![磁盘块管理相关函数关系（部分调用省略）](img/blog/2028226-20210523220348907-176254116.png)

上图中表明了磁盘块管理相关函数的调用关系：

- 绿色框内为基础检查/映射函数，不参与实际管理，仅被其他函数调用，故忽略调用关系
- 红色为系统调用
- 亮紫色为驱动程序

1. 磁盘块位图的管理：

   - `alloc_block_sum`:遍历bitmap，找到的第一个空闲的磁盘块，然后**将其写入磁盘**（DEBUG:没看懂login为什么这么写，我得写个函数调用图），然后返回磁盘块号
   - `alloc_block`:调用`alloc_block_sum`并为获得的新的磁盘块分配一个物理页面
   - `free_block`：在位图中标记一个磁盘块为空
   - `block_is_free`：检查位图来判断是否是空闲的

2. 磁盘块在文件系统进程空间的映射管理

   - `diskaddr`：实现从磁盘块号到对应虚拟地址的映射

   - `map_block`：调用`syscall_mem_alloc`为该磁盘块分配对应的物理页面并添加进入页表
   - `unmap_block`：检查是否需要将该磁盘块内容写入磁盘（取决于是否dirty），再调用`syscall_mem_unmap`释放物理页面
   - `va/block_is_mapped/dirty`：通过查询页表来检查其是否已经被分配了物理页面/因更改而变dirty，**通过检查权限位实现**

3. 磁盘块对应虚拟地址空间到磁盘的通过驱动程序支持的读写管理：
   - `read_block`:从磁盘中读出指定磁盘块对应的数据并存放在对应的虚拟空间里
   - `write_block`:从虚拟空间中读出需要写入磁盘的数据并写入

###### 文件控制块管理：

文件控制块管理囊括大量的文件操作函数，在这里不一一细讲。大部分函数都由课程组实现了，同学们一定要自己看一下。

- `file_block_walk`:通过一个文件控制块指针和一个整数filebno，去找到该文件中第filebno个4KB起始位置对应的磁盘块号。这个函数涉及直接指针和间接指针，注意查找逻辑。

- `file_map_block`:将`file_block_walk`进行包装，当alloc==1时，如果没找到对应的磁盘块号，则调用`alloc_block`新建一个。

- `file_clear_block`:将某个磁盘块从文件中移除

- `file_get_block`:读取文件特定磁盘块的信息

- `file_dirty`:将该文件特定磁盘块设置为dirty

- `dir_lookup`：根据一个指向目录文件的文件控制块指针dir，找到特定名字的文件对应的文件控制块。

  ……

其中`dir_lookup`函数需要我们自己实现：

```c
int
dir_lookup(struct File *dir, char *name, struct File **file)
{
	int r;
	u_int i, j, nblock;
	void *blk;
	struct File *f;

	nblock = ROUND(dir->f_size, BY2BLK) / BY2BLK; // 根据目录大小计算出内部磁盘块的数量

	for (i = 0; i < nblock; i++) {
		r = file_get_block(dir, i, &blk); // 读出该目录第i个磁盘块的信息并保存在blk中
		if (r < 0)
		{
			return r;
		}
	
		for (j = 0; j < FILE2BLK; j++) {
            // 遍历该磁盘块中所有的文件控制块
			f = ((struct File *)blk) + j;
			if (strcmp(f->f_name, name) == 0)
			{
                //如果找到目标文件，就返回
				f->f_dir = dir;
				*file = f;
				return 0;
			}
		}
	}
	return -E_NOT_FOUND; // 否则报异常
}
```

##### fsformat.c

该文件中存放了文件系统格式化相关的函数，包括磁盘的初始化、文件的创建、写入，主要以定义在普通文件和文件目录上的操作为主，可以看做是较为高级的文件操作集合。同样地，源码大部分已经被课程组实现了，就不再细说。

以下讲一下`create_file`函数：

这个函数从一个目录文件出发，目的是**寻找第一个能够放下新的文件控制块的位置**。当它找到一个指向已经被删除了的文件的文件控制块指针时，它直接返回，以求后续操作将这个空间覆盖掉。而当没有找到时，其直接进行拓展一个Block，并返回这个新的空白空间的起始地址。这里我们需要注意到，**一个未被占用的空间被解释为文件控制块指针时，其行为和一个指向已经被删除了的文件的文件控制块指针一致**，因此能够被统一处理。

```c
struct File *create_file(struct File *dirf) {
    struct File *dirblk;
    int i, bno, j;
    int nblk = dirf->f_size / BY2BLK; // 计算出该目录文件下有多少磁盘块
  
	for (i = 0; i < nblk; i++)
	{
        // 遍历所有磁盘块
		if (i < NDIRECT)
		{
			bno = dirf->f_direct[i];
		}
		else
		{
			bno = ((int *)(disk[dirf->f_indirect].data))[i];
		}
        // 根据直接指针或间接指针获得第i个磁盘块的块号
		dirblk = (struct File *)(disk[bno].data);
        // 得到第i个磁盘块起始位置起算的文件控制块数组的基址
		for (j = 0; j < FILE2BLK; j++)
		{
            // 遍历该磁盘块中所有文件控制块
			if (dirblk[j].f_name[0] == '\0')
			{
                // 如果发现有的文件控制块名称为终止符，说明这个文件已经被删除了/这个文件控制块的位置还没被占用，将该文件控制块起始地址返回
				return &dirblk[j];
			}
		}
	}
	// 遍历了所有的Block后都没找到一个空的能放文件控制块的地方
	bno = make_link_block(dirf, nblk);
    //直接拓展dirf的大小，并使第nblk个磁盘块链接到一个新的磁盘块（块号bno）
	return (struct File *)disk[bno].data;
    // 返回这个新的磁盘块内的起始地址
}
```

#### 用户接口

用户进程虽然通过IPC机制与文件系统通信，但是其在基本的通信逻辑上又封装了一些更为简洁的接口，存放在user/file.c中。**这一部分代码基本就是对文件系统服务的封装，在这里就不一一细说**。

同时用户接口中封装了一套专门用于描述文件的数据结构和逻辑，称为**文件描述符**，对应数据结构为struct Fd和struct Filefd。

##### 文件描述符

```c
struct Fd {
        u_int fd_dev_id; // 指示了该文件所处的设备
        u_int fd_offset; // 指示了当前用户进程对该文件进行操作的指针偏移位置（从文件开头起）
        u_int fd_omode; // 指示了文件的访问权限
};
struct Filefd {
        struct Fd f_fd;
        u_int f_fileid;
        struct File f_file;
};
```

注意struct Fd.fd_offset，其描述了用户进程当前在文件操作中的指针位置，该值会在`read`、`write`和`seek`时被修改（定义在user/fd.c中）。

可以看出，Filefd实际上是Fd和File的组合，并包含了其文件控制块id。你甚至可以直接让Fd*指向Filefd类型，因为Filefd类型的内存上的前一部分存放的正是一个Fd类型的数据。

##### fd.c

描述了write\read\close等用户使用的接口，其基于文件描述符去寻找所需的进行的操作。

##### file.c

定义了一系列基于文件控制块的函数，通过对文件控制块的解析和调用IPC来实现功能。

##### fsipc.c

定义了一系列IPC相关的函数，主要功能在于打包所要传递的参数和发送\接收。