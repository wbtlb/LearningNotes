---
title: "IO体系"
date: 2018-07-24T19:25:43+08:00
draft: false
---

### 1. IO体系结构

计算机之所以可以正常工作，提供数据通路，是因为在CPU、RAM和IO设备之间有一条总线。这条总线连接大部分内部硬件设备。一条典型的总线叫PCI，同时还有其他类型的总线，例如，ISA、SCSI和USB等。前端总线将CPU连接到RAM控制器上，后端总线将CPU直接连接到外部硬件的高速缓存上。主机上的桥将系统总线和前端总线连接在一起。

CPU和IO设备之间的通路叫IO总线。80x86微处理器使用16位的地址总线对IO设备寻址。每个IO设备依次连接到IO总线上，这种连接使用了包含3个元素的硬件组织层次：IO端口、接口和设备控制器。




### 2. 块设备驱动程序

#### 2.1 基本概念

块设备是具有一定结构的随机存取设备，对这种设备的读写是按块进行的，他使用缓冲区来存放暂时数据，待条件成熟后，从缓存中一次性写到设备或从设备一次性读到缓冲区。可以随机访问，块设备的访问位置必须能够在介质的不同扇区间前后移动。

##### 块设备相关属性

* 扇区（Sectors）：任何块设备硬件对数据处理的基本单位。通常一个扇区大小为512byte。
* 块（Blocks）：扇区是硬件设备传送数据的基本单位，而块是VFS和文件系统传送数据的基本单位。一个块对应磁盘上一个或多个相邻扇区，VFS将其看成一个单一的数据单元。在Linux中，块大小必须是2的幂，且不能超过一个页框，它必须是扇区大小的整数倍，因为每个块必须包含整个扇区。块设备大小不是唯一的，管理员可以自定义合适的大小。每个块都需要自己的块缓冲区，他是内核用来存放块内容的RAM内存区。当从磁盘读出一个块时，就用从硬件设备中所获得的值来填充相应的块缓冲区；同样块缓冲区实际值来更新块设备。块缓冲区大小通常要与相应的块大小相匹配。
* 段（Segments）：由若干个相邻的块组成。是Linux内存管理机制中一个内存页或者内存页的一部分。

#### 2.2 通用块层

通用块层是一个内核组件，用来处理来自系统中所有块设备发出的请求。内核可以通过该层完成以下功能：

* 将数据缓冲区放在高端内存，仅当CPU访问数据时，才将页框映射为内核中的限行地址空间，并完成访问后取消映射。
* 通过附加的手段，实现“零——复制”，将磁盘数据直接存放在用户态地址空间而不是首先复制到内核内存区；事实上，内核为IO数据传输使用的缓冲区所在的页框就应设在进程的用户态线性地址空间中。
* 管理逻辑卷。
* 发挥大部分新磁盘控制器的高级特性。

#### 2.3 bio 结构

通用块层核心数据结构是一个称为bio的描述符，它描述了块设备的IO操作。每个bio结构都包含了一个磁盘存储区标识符和一个或多个描述与IO操作相关的内存区的段。bio数据结构如下：

```
struct bio {
	sector_t		bi_sector;	/* device address in 512 byte
						   sectors */
	struct bio		*bi_next;	/* request queue link */
	struct block_device	*bi_bdev;
	unsigned long		bi_flags;	/* status, command, etc */
	unsigned long		bi_rw;		/* bottom bits READ/WRITE,
						 * top bits priority
						 */

	unsigned short		bi_vcnt;	/* how many bio_vec's */
	unsigned short		bi_idx;		/* current index into bvl_vec */

	/* Number of segments in this BIO after
	 * physical address coalescing is performed.
	 */
	unsigned int		bi_phys_segments;

	unsigned int		bi_size;	/* residual I/O count */

	/*
	 * To keep track of the max segment size, we account for the
	 * sizes of the first and last mergeable segments in this bio.
	 */
	unsigned int		bi_seg_front_size;
	unsigned int		bi_seg_back_size;

	bio_end_io_t		*bi_end_io;

	void			*bi_private;
#ifdef CONFIG_BLK_CGROUP
	/*
	 * Optional ioc and css associated with this bio.  Put on bio
	 * release.  Read comment on top of bio_associate_current().
	 */
	struct io_context	*bi_ioc;
	struct cgroup_subsys_state *bi_css;
#endif
#if defined(CONFIG_BLK_DEV_INTEGRITY)
	struct bio_integrity_payload *bi_integrity;  /* data integrity */
#endif

	/*
	 * Everything starting with bi_max_vecs will be preserved by bio_reset()
	 */

	unsigned int		bi_max_vecs;	/* max bvl_vecs we can hold */

	atomic_t		bi_cnt;		/* pin count */

	struct bio_vec		*bi_io_vec;	/* the actual vec list */

	struct bio_set		*bi_pool;

	/* FOR RH USE ONLY
	 *
	 * The following padding has been replaced to allow extending
	 * the structure, using struct bio_aux, while preserving ABI.
	 */
	RH_KABI_REPLACE(void *rh_reserved1, struct bio_aux *bio_aux)

	/*
	 * We can inline a number of vecs at the end of the bio, to avoid
	 * double allocations for a small number of bio_vecs. This member
	 * MUST obviously be kept at the very end of the bio.
	 */
	struct bio_vec		bi_inline_vecs[0];
};

```

