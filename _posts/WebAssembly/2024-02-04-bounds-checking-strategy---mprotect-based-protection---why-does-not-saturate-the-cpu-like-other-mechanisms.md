Bounds checking strategy - mprotect()-based protection - Does not saturate the CPU like other mechanisms
===


# Source
<mark>[Szewczyk, R., Stonehouse, K., Barbalace, A., & Spink, T. (2022). Leaps and bounds: Analyzing WebAssembly’s performance with a focus on bounds checking.  *2022 IEEE International Symposium on Workload Characterization (IISWC)*, 256–268. [https://doi.org/10.1109/IISWC55918.2022.00030]](https://doi.org/10.1109/IISWC55918.2022.00030)</mark>

## abstract from paper

+ At the same time, there are several different Wasm runtimes available to choose from, with a diverse set of designs and implementations, potentially each with a unique approach to bounds checking and code generation. Like many implementation details, the choice of bounds checking strategy can introduce significant overhead, and ultimately impact application execution time. While other issues such as register allocation and dealing with inlining are encountered by many language virtual machines, the bounds checking for all memory accesses is Wasm-specific. Despite diversity, ==to the best of our knowledge, most of the adopted Wasm runtimes implement bounds checking with mprotect() on POSIX-compliant operating systems (OS)==, even if several mechanisms have been made available recently and therefore need evaluation. In this paper, we will focus on POSIX OSes, specifically on Linux due to its wide adoption in data centers.
+ ==Using mprotect() on linux to dynamically adjust size of WebAssembly memory causes poor multithreaded scaling==: lowering maximum CPU utilization by up to 25% on shortrunning benchmarks.
+ Another major difference, also visible in the context switch graphs, is the ==poor scaling of mprotect()-based memory protection to multiple threads==    . Especially visible in the PolybenchC benchmarks, which cause frequent allocation and deallocation of memory as they execute in a short span of time, this stresses the virtual memory management subsystem in the Linux kernel for the host process, causing excessive locking and pausing of thread execution.
+ In case of multithreaded scaling of the tested runtimes, which might for example be used to quickly scale up serverless instances for a single function without the overhead of spawning new processes, ==the default approach taken by WAVM, Wasmtime and V8 of using the mprotect() syscall to resize memory can cause excessive locking in the Linux kernel==. This can be mitigated by using simpler, lockfree data structures for managing page permissions, which we were able to implement using Linux’s recent userfaultfd mechanism for handling page faults in userspace.
+ One big difference visible here between the bounds checking strategies is that ==mprotect()-based protection does not saturate the CPU like other mechanisms, as earlier discusses in Section 4.1.1. This is due to a mutex in the Linux kernel protecting the process’ virtual memory areas tree [13], when WebAssembly resizes its memory to allocate or run the next iteration that mutex is acquired for significant periods of time== that we confirmed by capturing stack traces of threads in a waiting state via bpftools. Software bounds checking requires less virtual memory manipulation, hence the effect there is not visible. ==The UFFD mechanism in the kernel does not acquire an exclusive lock over that structure==, so the userspace code is able to use lockfree structures to manage its memory – in our implementation we use an atomic integer variable controlling the size of each memory arena, and a hazard pointer [18]-style implementation for adding and removing memory arenas, avoiding the need for locks most of the time.

# [Description of mprotect()](https://man7.org/linux/man-pages/man2/mprotect.2.html)

chang the access protections for the calling process's memory pages containing any part of the address range in the interval [addr, addr + len + 1]. Addr must be aligned to a page boundary.

If the calling process tries to access memory in a manner that violates the protections, then the kernel generates a SIGSEGV signal for the process.

`prot` is a combination of the following access flags: PROT_NONE or a bitwise OR of the other values.

## examples 

This example demonstrates the use of mprotect() The program allocates four pages of memory, makes the third of these pages read=only, and then executes a loop that walks upward through the allocated region modifying bytes.

```cpp
#include <malloc.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <unistd.h>

#define handle_error(msg) \
    do { perror(msg); exit(EXIT_FAILURE); } while (0)

static char *buffer;

static void
handler(int sig, siginfo_t *si, void *unused)
{
    /* Note: calling printf() from a signal handler is not safe
        (and should not be done in production programs), since
        printf() is not async-signal-safe; see signal-safety(7).
        Nevertheless, we use printf() here as a simple way of
        showing that the handler was called. */

    printf("Got SIGSEGV at address: %p\n", si->si_addr);
    exit(EXIT_FAILURE);
}

int
main(void)
{
    int               pagesize;
    struct sigaction  sa;

    sa.sa_flags = SA_SIGINFO;
    sigemptyset(&sa.sa_mask);
    sa.sa_sigaction = handler;
    if (sigaction(SIGSEGV, &sa, NULL) == -1)
        handle_error("sigaction");

    pagesize = sysconf(_SC_PAGE_SIZE);
    if (pagesize == -1)
        handle_error("sysconf");

    /* Allocate a buffer aligned on a page boundary;
        initial protection is PROT_READ | PROT_WRITE. */

    buffer = memalign(pagesize, 4 * pagesize);
    if (buffer == NULL)
        handle_error("memalign");

    printf("Start of region:        %p\n", buffer);

    if (mprotect(buffer + pagesize * 2, pagesize,
                PROT_READ) == -1)
        handle_error("mprotect");

    for (char *p = buffer ; ; )
        *(p++) = 'a';

    printf("Loop completed\n");     /* Should never happen */
    exit(EXIT_SUCCESS);
}
```
And the run results:

```cmd
$ ./a.out
Start of region:        0x804c000
Got SIGSEGV at address: 0x804e000
```
# Explain where causes excessive locking in the Linux

The excessive locking in the Linux kernel, ==is primarily associated with the mprotect() function's need to modify the process's virtual memory areas (VMAs). When mprotect() changes the protection levels of memory pages, it must update the corresponding page table entries.== This operation requires acquiring a mutex to ensure consistency and prevent concurrent modifications to the VMAs, which can lead to significant locking overhead, especially in scenarios where memory protections are frequently modified. The described mutex protects the VMAs tree, a critical structure, thus serializing access to it and potentially leading to performance bottlenecks in high-concurrency situations.

```cpp
/*
 *  linux/mm/mprotect.c
 *
 *  (C) Copyright 1994 Linus Torvalds
 */
#include <linux/stat.h>
#include <linux/sched.h>
#include <linux/kernel.h>
#include <linux/mm.h>
#include <linux/shm.h>
#include <linux/errno.h>
#include <linux/mman.h>
#include <linux/string.h>
#include <linux/malloc.h>

#include <asm/segment.h>
#include <asm/system.h>
#include <asm/pgtable.h>
// 修改虚拟地址address到address+size的页表项内容
static inline void change_pte_range(pmd_t * pmd, unsigned long address,
  unsigned long size, pgprot_t newprot)
{
  pte_t * pte;
  unsigned long end;

  if (pmd_none(*pmd))
    return;
  if (pmd_bad(*pmd)) {
    printk("change_pte_range: bad pmd (%08lx)\n", pmd_val(*pmd));
    pmd_clear(pmd);
    return;
  }
  // 获取一项页表项地址
  pte = pte_offset(pmd, address);
  // 屏蔽低位
  address &= ~PMD_MASK;
  // 结束地址
  end = address + size;
  // 不能超过该目录项管理的地址范围
  if (end > PMD_SIZE)
    end = PMD_SIZE;
  do {
    pte_t entry = *pte;
    if (pte_present(entry))
      // 更新页表项内容
      *pte = pte_modify(entry, newprot);
    // 下一个待处理的虚拟地址
    address += PAGE_SIZE;
    pte++;
  } while (address < end);
}
// 修改虚拟地址address到address+size区间的页目录项、页表项内容
static inline void change_pmd_range(pgd_t * pgd, unsigned long address,
  unsigned long size, pgprot_t newprot)
{
  pmd_t * pmd;
  unsigned long end;

  if (pgd_none(*pgd))
    return;
  if (pgd_bad(*pgd)) {
    printk("change_pmd_range: bad pgd (%08lx)\n", pgd_val(*pgd));
    pgd_clear(pgd);
    return;
  }
  // 某个页目录项
  pmd = pmd_offset(pgd, address);
  address &= ~PGDIR_MASK;
  end = address + size;
  if (end > PGDIR_SIZE)
    end = PGDIR_SIZE;
  do {
    change_pte_range(pmd, address, end - address, newprot);
    address = (address + PMD_SIZE) & PMD_MASK;
    pmd++;
  } while (address < end);
}
// 修改当前进程虚拟地址start到start+end区间的页目录和页表项内容
static void change_protection(unsigned long start, unsigned long end, pgprot_t newprot)
{
  pgd_t *dir;
  // 返回某页目录项地址 
  dir = pgd_offset(current, start);
  while (start < end) {
    // 修改某页目录项对应的页表内容
    change_pmd_range(dir, start, end - start, newprot);
    start = (start + PGDIR_SIZE) & PGDIR_MASK;
    dir++;
  }
  // 刷新快表
  invalidate();
  return;
}
// 设置vma的读写属性和映射方式
static inline int mprotect_fixup_all(struct vm_area_struct * vma,
  int newflags, pgprot_t prot)
{
  // 用户层面的属性
  vma->vm_flags = newflags;
  // 页的属性，和vm_flags存在映射关系
  vma->vm_page_prot = prot;
  return 0;
}
// 修改开始地址为vma->start，结束地址为end的内存属性
static inline int mprotect_fixup_start(struct vm_area_struct * vma,
  unsigned long end,
  int newflags, pgprot_t prot)
{
  struct vm_area_struct * n;
  // vma的flag和prot是是针对整个vma的，所以这里要切分成两个vma
  n = (struct vm_area_struct *) kmalloc(sizeof(struct vm_area_struct), GFP_KERNEL);
  if (!n)
    return -ENOMEM;
  // 复制原vma结构体内容
  *n = *vma;
  // 修改原vma的start为end，即一分为二
  vma->vm_start = end;
  // 新vma的start不变，end改成切分边界的值
  n->vm_end = end;
  // 重新计算偏移，可能超过end
  vma->vm_offset += vma->vm_start - n->vm_start;
  // 只需要设置新块的标记
  n->vm_flags = newflags;
  n->vm_page_prot = prot;
  // 多了一个vma引用文件
  if (n->vm_inode)
    n->vm_inode->i_count++;
  if (n->vm_ops && n->vm_ops->open)
    n->vm_ops->open(n);
  // 插入进程的vma结构
  insert_vm_struct(current, n);
  return 0;
}
// 设置开始地址为start结束地址为vma的end这片内存的属性
static inline int mprotect_fixup_end(struct vm_area_struct * vma,
  unsigned long start,
  int newflags, pgprot_t prot)
{
  struct vm_area_struct * n;
  // 一分为二，申请一块新的vma
  n = (struct vm_area_struct *) kmalloc(sizeof(struct vm_area_struct), GFP_KERNEL);
  if (!n)
    return -ENOMEM;
  *n = *vma;
  // start为切分边界，修改原vma的end为start
  vma->vm_end = start;
  // 新vma的start为start
  n->vm_start = start;
  // 相当于vm_offset = vm_offset - vma->start + n->vm_start,新地址加上相对偏移
  n->vm_offset += n->vm_start - vma->vm_start;
  // 只需设置新块的属性
  n->vm_flags = newflags;
  n->vm_page_prot = prot;
  // 多了一个vma引用文件
  if (n->vm_inode)
    n->vm_inode->i_count++;
  if (n->vm_ops && n->vm_ops->open)
    n->vm_ops->open(n);
  // 插入进程vma结构
  insert_vm_struct(current, n);
  return 0;
}
// 设置开始地址为start结束地址为end这片内存的属性
static inline int mprotect_fixup_middle(struct vm_area_struct * vma,
  unsigned long start, unsigned long end,
  int newflags, pgprot_t prot)
{
  struct vm_area_struct * left, * right;
  // 一分为三
  left = (struct vm_area_struct *) kmalloc(sizeof(struct vm_area_struct), GFP_KERNEL);
  if (!left)
    return -ENOMEM;
  right = (struct vm_area_struct *) kmalloc(sizeof(struct vm_area_struct), GFP_KERNEL);
  if (!right) {
    kfree(left);
    return -ENOMEM;
  }
  // 复制得到默认值
  *left = *vma;
  *right = *vma;
  // 一块的结束地址是start
  left->vm_end = start;
  // 第二块的开始地址是start，结束地址是end，start和end是用户修改属性的内存范围
  vma->vm_start = start;
  vma->vm_end = end;
  // 第三块的start是end
  right->vm_start = end;
  // 第一块不需要更新offset，第二、第三块需要更新offset，都是新开始地址+之前的相对偏移
  vma->vm_offset += vma->vm_start - left->vm_start;
  right->vm_offset += right->vm_start - left->vm_start;
  // 只需要设置第二块的属性
  vma->vm_flags = newflags;
  vma->vm_page_prot = prot;
  // 多了两个vma引用文件
  if (vma->vm_inode)
    vma->vm_inode->i_count += 2;
  if (vma->vm_ops && vma->vm_ops->open) {
    vma->vm_ops->open(left);
    vma->vm_ops->open(right);
  }
  // 插入两个vma
  insert_vm_struct(current, left);
  insert_vm_struct(current, right);
  return 0;
}
// 修改一个vma某个内存区间的属性
static int mprotect_fixup(struct vm_area_struct * vma, 
  unsigned long start, unsigned long end, unsigned int newflags)
{
  pgprot_t newprot;
  int error;
  // 不变
  if (newflags == vma->vm_flags)
    return 0;
  // 见mmap.c的protection_map，把用户层的标记转成页表项格式的值，第四位表示是否共享
  newprot = protection_map[newflags & 0xf];
  if (start == vma->vm_start)
    if (end == vma->vm_end)
      // 地址完全重合则直接覆盖vma的设置
      error = mprotect_fixup_all(vma, newflags, newprot);
    else
      // start重合则修改start到end的设置
      error = mprotect_fixup_start(vma, end, newflags, newprot);
  // 结束地址重合
  else if (end == vma->vm_end)
    error = mprotect_fixup_end(vma, start, newflags, newprot);
  else
    // 中间部分重合
    error = mprotect_fixup_middle(vma, start, end, newflags, newprot);

  if (error)
    return error;
  // 修改页目录、页表的内容
  change_protection(start, end, newprot);
  return 0;
}
// 设置start开始，大小是len的这片内存的属性为prot
asmlinkage int sys_mprotect(unsigned long start, size_t len, unsigned long prot)
{
  unsigned long nstart, end, tmp;
  struct vm_area_struct * vma, * next;
  int error;
  // 低12位不为0，没有页对齐，报错
  if (start & ~PAGE_MASK)
    return -EINVAL;
  // 长度是页大小的整数倍，~PAGE_MASK表示不够一页则补足一页
  len = (len + ~PAGE_MASK) & PAGE_MASK;
  // 修改的末地址
  end = start + len;
  if (end < start)
    return -EINVAL;
  // 只能设置这三个标记
  if (prot & ~(PROT_READ | PROT_WRITE | PROT_EXEC))
    return -EINVAL;
  // 没有内存需要修改
  if (end == start)
    return 0;
  // 找出地址start对应vma
  vma = find_vma(current, start);
  // 地址无效
  if (!vma || vma->vm_start > start)
    return -EFAULT;
  // 循环处理
  for (nstart = start ; ; ) {
    unsigned int newflags;

    /* Here we know that  vma->vm_start <= nstart < vma->vm_end. */
    /*
      (vma->vm_flags & ~(PROT_READ | PROT_WRITE | PROT_EXEC))表示清掉读写执行三个标记，
      保留其他的标记，然后再与prot。即重新设置读写执行位
    */
    newflags = prot | (vma->vm_flags & ~(PROT_READ | PROT_WRITE | PROT_EXEC));
    /*
      flag的取值见mm.h
      高四位是标记对应的属性是否可以设置。从低到高分别是可读、可写、可执行
      newflags右移四位把高四位移到第四位，四位中，置一的位说明可以设置，所以不需要校验，
      只需要校验为0的位，所以取反，置0的位变成1，如果最后与的时候非0，说明用户设置了这一位，
      则不合法。(最后&0xf说明只关注低四位。)
    */
    if ((newflags & ~(newflags >> 4)) & 0xf) {
      error = -EACCES;
      break;
    }
    // 成立的话说明用户设置的内存区间落在一个vma里，直接修改就行，否则需要修改多个vma，见下面
    if (vma->vm_end >= end) {
      error = mprotect_fixup(vma, nstart, end, newflags);
      break;
    }
    // 用户设置的end大于vma的end，所以需要设置多次
    tmp = vma->vm_end;
    // 下一个vma
    next = vma->vm_next;
    // 设置第一个vma的属性，下一轮修改下一个vma的属性
    error = mprotect_fixup(vma, nstart, tmp, newflags);
    if (error)
      break;
    // 重新设置start的值，为当前vma的end，而不是下一个vma的开始地址
    nstart = tmp;
    vma = next;
    // 下一块的start不等于nstart，即不等于上一块的end，说明不连续，用户设置的范围不合法，报错
    if (!vma || vma->vm_start != nstart) {
      error = -EFAULT;
      break;
    }
  }
  // 处理avl树
  merge_segments(current, start, end);
  return error;
}
```