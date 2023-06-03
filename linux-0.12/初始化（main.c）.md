---
title: linux源码分析（一）初始化程序
date: 2023-05-07 19:00:00
tags: 
	- linux内核
	- init
categories: linux-0.12
---
系统在执行完boot/head.s程序后就会将执行权交给main.c。该程序位于内核源代码的init目录，完成了内核初始化的所有工作。

### 初始化流程

### 源码

```c
void main(void)		/* This really IS void, no error here. */
{			/* The startup routine assumes (well, ...) this */
/*
 * Interrupts are still disabled. Do necessary setups, then
 * enable them
 */
 	ROOT_DEV = ORIG_ROOT_DEV;
 	SWAP_DEV = ORIG_SWAP_DEV;
	sprintf(term, "TERM=con%dx%d", CON_COLS, CON_ROWS);
	envp[1] = term;
	envp_rc[1] = term;
 	drive_info = DRIVE_INFO;
	memory_end = (1<<20) + (EXT_MEM_K<<10);
	memory_end &= 0xfffff000;
	if (memory_end > 16*1024*1024)
		memory_end = 16*1024*1024;		// 最多管理16M内存
	if (memory_end > 12*1024*1024) 		// 物理内存大于12M，则设置主内存从4M开始
		buffer_memory_end = 4*1024*1024;
	else if (memory_end > 6*1024*1024)	// 物理内存大于6M，小于12M，则设置主内存从2M开始
		buffer_memory_end = 2*1024*1024;
	else								// 物理内存小于6M，则设置主内存从1M开始
		buffer_memory_end = 1*1024*1024;
	main_memory_start = buffer_memory_end;
#ifdef RAMDISK
	main_memory_start += rd_init(main_memory_start, RAMDISK*1024);
#endif
	mem_init(main_memory_start,memory_end);	// 初始化主内存区
	trap_init();
	blk_dev_init();
	chr_dev_init();
	tty_init();
	time_init();
	sched_init();
	buffer_init(buffer_memory_end);		// 初始化高速缓冲区
	hd_init();
	floppy_init();
	sti();	// 开启中断
	move_to_user_mode();
	if (!fork()) {		/* we count on this going ok */
		init();
	}
/*
 *   NOTE!!   For any other task 'pause()' would mean we have to get a
 * signal to awaken, but task0 is the sole exception (see 'schedule()')
 * as task 0 gets activated at every idle moment (when no other tasks
 * can run). For task0 'pause()' just means we go check if some other
 * task can run, and if not we return here.
 */
	for(;;)
		__asm__("int $0x80"::"a" (__NR_pause):"ax");
}
```
