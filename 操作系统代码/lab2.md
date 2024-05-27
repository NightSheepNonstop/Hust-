## lab2-1
---
- vmm.c
```cpp
void *user_va_to_pa(pagetable_t page_dir, void *va) {
  // TODO (lab2_1): implement user_va_to_pa to convert a given user virtual address "va"
  // to its corresponding physical address, i.e., "pa". To do it, we need to walk
  // through the page table, starting from its directory "page_dir", to locate the PTE
  // that maps "va". If found, returns the "pa" by using:
  // pa = PYHS_ADDR(PTE) + (va & (1<<PGSHIFT -1))
  // Here, PYHS_ADDR() means retrieving the starting address (4KB aligned), and
  // (va & (1<<PGSHIFT -1)) means computing the offset of "va" inside its page.
  // Also, it is possible that "va" is not mapped at all. in such case, we can find
  // invalid PTE, and should return NULL.
  uint64 value=lookup_pa(page_dir,(uint64)va);
  void* pa=(void*)(value+((uint64)va&((1<<PGSHIFT)-1)));
  return pa;
}
```

## lab 2-2
---
- vmm.c
```cpp
void user_vm_unmap(pagetable_t page_dir, uint64 va, uint64 size, int free) {
  // TODO (lab2_2): implement user_vm_unmap to disable the mapping of the virtual pages
  // in [va, va+size], and free the corresponding physical pages used by the virtual
  // addresses when if 'free' (the last parameter) is not zero.
  // basic idea here is to first locate the PTEs of the virtual pages, and then reclaim
  // (use free_page() defined in pmm.c) the physical pages. lastly, invalidate the PTEs.
  // as naive_free reclaims only one page at a time, you only need to consider one page
  // to make user/app_naive_malloc to behave correctly.
  pte_t*pte=page_walk(page_dir,va,0);
  if(pte==NULL) return;
  void*pa=(void*)lookup_pa(page_dir,va);
  free_page(pa);
  *pte&=~PTE_V;
}
```

## lab2-3
---
- strap.c
```cpp
void handle_user_page_fault(uint64 mcause, uint64 sepc, uint64 stval) {
  sprint("handle_page_fault: %lx\n", stval);
  switch (mcause) {
    case CAUSE_STORE_PAGE_FAULT:
      // TODO (lab2_3): implement the operations that solve the page fault to
      // dynamically increase application stack.
      // hint: first allocate a new physical page, and then, maps the new page to the
      // virtual address that causes the page fault.
      user_vm_map((pagetable_t)current->pagetable,stval-stval%PGSIZE,PGSIZE,(uint64)alloc_page(),prot_to_type(PROT_WRITE | PROT_READ, 1));//注意页面对齐
      
      break;
    default:
      sprint("unknown page fault.\n");
      break;
  }
}
```

## lab2 challenge1 
---
- strap.c(在lab2-3的基础上增加越界判断)
```c
if(stval >= g_ufree_page && stval < g_ufree_page+PGSIZE) {  
	        sprint("this address is not available!\n");  
	        shutdown(-1);  
	        break;  
      }  
```

## lab2 challenge2
---
- syscall.c 修改调用方式
```c
// added @ lab2 challenge2
uint64 sys_user_allocate_page(uint64 n) {
  uint64 addr = user_better_malloc(n);
  return addr;
}
uint64 sys_user_free_page(uint64 va) {
  user_better_free((void*)va);
  return 0;
}
```
- vmm.c 
- 增加头文件
```c
// added @ lab2 challenge2
#include "process.h"
```
- 增加函数
```C
// added@ lab2 challenge2
bool first_malloc = TRUE;
struct MCB_t *MCB_head = NULL;
uint64 cur_mapping = USER_FREE_ADDRESS_START;
void vm_malloc(uint64 n)
{
  uint64 temp = ROUNDUP(cur_mapping, PGSIZE);
  char *new_page;
  for (uint64 i = temp; i < cur_mapping + n; i += PGSIZE)
  {
    new_page = (char *)alloc_page();
    memset(new_page, 0, (size_t)PGSIZE);
    map_pages(current->pagetable, temp, PGSIZE, (uint64)new_page, prot_to_type(PROT_READ | PROT_WRITE, 1));
  }
  cur_mapping += n;
  return;
}
void init_MCBs()
{
  if (first_malloc)
  {
    uint64 va_top = cur_mapping;
    vm_malloc(sizeof(MCB));
    pte_t *pte = page_walk(current->pagetable, USER_FREE_ADDRESS_START, 0);
    MCB_head = (MCB *)PTE2PA(*pte);
    MCB_head->is_occupied = 0;
    MCB_head->offset = *pte + sizeof(MCB);
    MCB_head->size = 0;
    MCB_head->next = NULL;
    first_malloc = FALSE;
    return;
  }
  return;
}
uint64 user_better_malloc(uint64 n)
{
  init_MCBs();
  MCB *cur = (MCB *)MCB_head;
  while (1)
  {
    if (cur->size >= n && cur->is_occupied == 0)
    {
      cur->is_occupied = 1;
      return cur->offset;
    }
    if (cur->next == NULL)
      break;
    cur = cur->next;
  }
  uint64 heap_top = cur_mapping;
  vm_malloc(sizeof(MCB) + n);
  pte_t *pte = page_walk(current->pagetable, heap_top, 0);
  MCB *next_MCB = (MCB *)(PTE2PA(*pte) + (heap_top & 0xfff));
  uint64 align = (uint64)next_MCB % 8;
  next_MCB = (MCB *)((uint64)next_MCB + 8 - align);

  next_MCB->is_occupied = 1;
  next_MCB->offset = heap_top + sizeof(MCB);
  next_MCB->size = n;
  next_MCB->next = cur->next;

  cur->next = next_MCB;
  return next_MCB->offset;
}

void user_better_free(void *addr)
{
  void *real_addr = (void *)((uint64)addr - sizeof(MCB));
  pte_t *pte = page_walk(current->pagetable, (uint64)(real_addr), 0);
  MCB *cur = (MCB *)(PTE2PA(*pte) + ((uint64)real_addr & 0xfff));
  uint64 align = (uint64)cur % 8;
  cur = (MCB *)((uint64)cur + 8 - align);
  cur->is_occupied = 0;
}

```
- vmm.h 增加内存控制块结构
```C
// added @ lab2 challenge2
typedef struct MCB_t {
  int is_occupied;
  uint64 offset;
  uint64 size;
  struct MCB_t *next;
}MCB;//内存控制块
```
- 增加函数声明
```C
// added @ lab2 challenge2
void vm_malloc(uint64 n);
void init_MCBs();
uint64 user_better_malloc(uint64 n);
void user_better_free(void *addr) ;
```

## lab2 challenge3
---
### elf.c
```c
// @lab2 challenge3
void load_bincode_from_host_elf(process *p) {
  arg_buf arg_bug_msg;
 int hartid = read_tp();//读取核id
  // retrieve command line arguements
  size_t argc = parse_args(&arg_bug_msg);
  if (!argc) panic("You need to specify the application program!\n");

  sprint("hartid = %d: Application: %s\n", hartid,arg_bug_msg.argv[hartid]);


  //elf loading. elf_ctx is defined in kernel/elf.h, used to track the loading process.
  elf_ctx elfloader;
  // elf_info is defined above, used to tie the elf file and its corresponding process.
  elf_info info;

  info.f = spike_file_open(arg_bug_msg.argv[hartid], O_RDONLY, 0);
  info.p = p;
  // IS_ERR_VALUE is a macro defined in spike_interface/spike_htif.h
  if (IS_ERR_VALUE(info.f)) panic("Fail on openning the input application program.\n");

  // init elfloader context. elf_init() is defined above.
  if (elf_init(&elfloader, &info) != EL_OK)
    panic("fail to init elfloader.\n");

  // load elf. elf_load() is defined above.
  if (elf_load(&elfloader) != EL_OK) panic("Fail on loading elf.\n");

  // entry (virtual, also physical in lab1_x) address
  p->trapframe->epc = elfloader.ehdr.entry;

  // close the host spike file
  spike_file_close( info.f );

  sprint("hartid = %d: Application program entry point (virtual address): 0x%lx\n",hartid, p->trapframe->epc);
}

```
### kernel.c
- 增加头文件
```C
//@lab2 challenge3
#include "sync_utils.h"
```
- 修改定义
```C
// @lab2 challenge3
process user_app[NCPU];//不同核的应用进程应该区分

```
- 加载
```C
// @lab2 challenge3
void load_user_program(process *proc) {
   int hartid = read_tp();
  sprint("hartid = %d: User application is loading.\n", hartid);
  g_ufree_page[hartid] = USER_FREE_ADDRESS_START;
  // allocate a page to store the trapframe. alloc_page is defined in kernel/pmm.c. added @lab2_1
  proc->trapframe = (trapframe *)alloc_page();
  memset(proc->trapframe, 0, sizeof(trapframe));

  // allocate a page to store page directory. added @lab2_1
  proc->pagetable = (pagetable_t)alloc_page();
  memset((void *)proc->pagetable, 0, PGSIZE);

  // allocate pages to both user-kernel stack and user app itself. added @lab2_1
  proc->kstack = (uint64)alloc_page() + PGSIZE;   //user kernel stack top
  uint64 user_stack = (uint64)alloc_page();       //phisical address of user stack bottom

  // USER_STACK_TOP = 0x7ffff000, defined in kernel/memlayout.h
  proc->trapframe->regs.sp = USER_STACK_TOP;  //virtual address of user stack top
  proc->trapframe->regs.tp = hartid;
  sprint("hartid = %d: user frame 0x%lx, user stack 0x%lx, user kstack 0x%lx \n",hartid, proc->trapframe,
         proc->trapframe->regs.sp, proc->kstack);

  // load_bincode_from_host_elf() is defined in kernel/elf.c
  load_bincode_from_host_elf(proc);

  // populate the page table of user application. added @lab2_1
  // map user stack in userspace, user_vm_map is defined in kernel/vmm.c
  user_vm_map((pagetable_t)proc->pagetable, USER_STACK_TOP - PGSIZE, PGSIZE, user_stack,
         prot_to_type(PROT_WRITE | PROT_READ, 1));

  // map trapframe in user space (direct mapping as in kernel space).
  user_vm_map((pagetable_t)proc->pagetable, (uint64)proc->trapframe, PGSIZE, (uint64)proc->trapframe,
         prot_to_type(PROT_WRITE | PROT_READ, 0));

  // map S-mode trap vector section in user space (direct mapping as in kernel space)
  // here, we assume that the size of usertrap.S is smaller than a page.
  user_vm_map((pagetable_t)proc->pagetable, (uint64)trap_sec_start, PGSIZE, (uint64)trap_sec_start,
         prot_to_type(PROT_READ | PROT_EXEC, 0));
}

```
- S_START
```C
// @lab2 challenge3
static int s_cnt = 0;
int s_start(void) {
  int hartid = read_tp();
  sprint("hartid = %d: Enter supervisor mode...\n",hartid);
  // in the beginning, we use Bare mode (direct) memory mapping as in lab1.
  // but now, we are going to switch to the paging mode @lab2_1.
  // note, the code still works in Bare mode when calling pmm_init() and kern_vm_init().
  write_csr(satp, 0);

  if(hartid == 0){
    // init phisical memory manager
  pmm_init();
  // build the kernel page table
  kern_vm_init();
  }// 初始化一次
  sync_barrier(&s_cnt, NCPU);
  // now, switch to paging mode by turning on paging (SV39)
  enable_paging();
  // the code now formally works in paging mode, meaning the page table is now in use.
  sprint("kernel page table is on \n");

  // the application code (elf) is first loaded into memory, and then put into execution
  load_user_program(&user_app[hartid]);
  sprint("hartid = %d: Switch to user mode...\n",hartid);

  vm_alloc_stage[hartid] = 1;
  // switch_to() is defined in kernel/process.c
  switch_to(&user_app[hartid]);

  // we should never reach here.
  return 0;
}

```
### pmm.c
- 增加头文件
```C
#include "sync_utils.h"//@lab2 challenge3
```
-  分配和回收页使用自旋锁
```C
static int mem_lock;
void free_page(void *pa) {
  spin_lock(&mem_lock);
  if (((uint64)pa % PGSIZE) != 0 || (uint64)pa < free_mem_start_addr || (uint64)pa >= free_mem_end_addr)
    panic("free_page 0x%lx \n", pa);

  // insert a physical page to g_free_mem_list
  list_node *n = (list_node *)pa;
  n->next = g_free_mem_list.next;
  g_free_mem_list.next = n;
  spin_unlock(&mem_lock);
}
//
// takes the first free page from g_free_mem_list, and returns (allocates) it.
// Allocates only ONE page!
//
void *alloc_page(void) {
  spin_lock(&mem_lock);
  list_node *n = g_free_mem_list.next;
  int hartid = read_tp();
  if (vm_alloc_stage[hartid]) {
    sprint("hartid = %ld: alloc page 0x%x\n", hartid, n);
  }
  if (n) g_free_mem_list.next = n->next;
  spin_unlock(&mem_lock);
  return (void *)n;
}
```
### process.c
- 增加头文件
```C
// @lab2 challenge3
process* current[NCPU] = {NULL};
uint64 g_ufree_page[NCPU];
```
- swith_to
```c
 //@lab2 challenge3
  int hartid = read_tp();
  current[hartid] = proc;
```
### process.h
```C
//@lab2 challenge3
extern process* current[NCPU];

// address of the first free page in our simple heap. added @lab2_2
//@lab2 challenge3
extern uint64 g_ufree_page[NCPU];

```
### strap.c
```c
//@lab2 challenge3
static uint64 g_ticks[NCPU] = {0};
//
// added @lab1_3
//
//@ lab2 challenge3 分开管理每个hart上运行应用的tick
void handle_mtimer_trap() {
  int hartid = read_tp();
  sprint("Ticks %d\n", g_ticks[hartid]);
  // TODO (lab1_3): increase g_ticks to record this "tick", and then clear the "SIP"
  // field in sip register.
  // hint: use write_csr to disable the SIP_SSIP bit in sip.
   g_ticks[hartid]++;
 write_csr(sip,0);

}
```

```c
//@lab2 challenge3
void handle_user_page_fault(uint64 mcause, uint64 sepc, uint64 stval) {
  int hartid = read_tp();
  sprint("handle_page_fault: %lx\n", stval);
  switch (mcause) {
    case CAUSE_STORE_PAGE_FAULT:
      // TODO (lab2_3): implement the operations that solve the page fault to
      // dynamically increase application stack.
      // hint: first allocate a new physical page, and then, maps the new page to the
      // virtual address that causes the page fault.
       user_vm_map((pagetable_t)current[hartid]->pagetable,stval-stval%PGSIZE,PGSIZE,(uint64)alloc_page(),prot_to_type(PROT_WRITE | PROT_READ, 1));

      break;
    default:
      sprint("unknown page fault.\n");
      break;
  }
}
```

```c
//@lab2 challenge3
void smode_trap_handler(void) {
  int hartid = read_tp();
  // make sure we are in User mode before entering the trap handling.
  // we will consider other previous case in lab1_3 (interrupt).
  if ((read_csr(sstatus) & SSTATUS_SPP) != 0) panic("usertrap: not from user mode");

  assert(current[hartid]);
  // save user process counter.
  current[hartid]->trapframe->epc = read_csr(sepc);

  // if the cause of trap is syscall from user application.
  // read_csr() and CAUSE_USER_ECALL are macros defined in kernel/riscv.h
  uint64 cause = read_csr(scause);

  // use switch-case instead of if-else, as there are many cases since lab2_3.
  switch (cause) {
    case CAUSE_USER_ECALL:
      handle_syscall(current[hartid]->trapframe);
      break;
    case CAUSE_MTIMER_S_TRAP:
      handle_mtimer_trap();
      break;
    case CAUSE_STORE_PAGE_FAULT:
    case CAUSE_LOAD_PAGE_FAULT:
      // the address of missing page is stored in stval
      // call handle_user_page_fault to process page faults
      handle_user_page_fault(cause, read_csr(sepc), read_csr(stval));
      break;
    default:
      sprint("smode_trap_handler(): unexpected scause %p\n", read_csr(scause));
      sprint("            sepc=%p stval=%p\n", read_csr(sepc), read_csr(stval));
      panic( "unexpected exception happened.\n" );
      break;
  }

  // continue (come back to) the execution of current[hartid] process.
  switch_to(current[hartid]);
}

```

### sync_utils.h
```c
//@lab2 challenge3
static inline void spin_lock(volatile int *lock) {
  int local = 0;
  do {
    asm volatile("amoswap.w %0, %2, (%1)"
                : "=r"(local)
                : "r"(lock), "r"(1)
                : "memory");
  } while(local);
}

static inline void spin_unlock(volatile int *lock) {
  asm volatile("sw zero, (%0)"
              :
              : "r"(lock)
              : "memory");
}

```
### syscall.c
```c
//@lab2 challenge3
#include "sync_utils.h"
```

```c
//@lab2 challenge3
ssize_t sys_user_print(const char* buf, size_t n) {
  // buf is now an address in user space of the given app's user stack,
  // so we have to transfer it into phisical address (kernel is running in direct mapping).
  int hartid = read_tp();
  assert( current[hartid] );
  char* pa = (char*)user_va_to_pa((pagetable_t)(current[hartid]->pagetable), (void*)buf);
  sprint(pa);
  return 0;
}
```

```c
//@lab2 challenge3
static int exit_cnt=0;
ssize_t sys_user_exit(uint64 code) {
  int hartid = read_tp(); 
  sprint("hartid = %d: User exit with code:%d.\n",hartid, code);
  sync_barrier(&exit_cnt, NCPU);//同步至所有核执行完毕
  if(hartid==0){//cpu0负责关闭模拟器
   sprint("hartid = %d: shutdown with code:%d.\n",hartid,code);
  shutdown(code);
  }
  return 0;
}
```

```c
//@lab2 challenge3
uint64 sys_user_allocate_page() {
  int hartid = read_tp();
  void* pa = alloc_page();
  uint64 va = g_ufree_page[hartid];
  g_ufree_page[hartid] += PGSIZE;
  user_vm_map((pagetable_t)current[hartid]->pagetable, va, PGSIZE, (uint64)pa,
         prot_to_type(PROT_WRITE | PROT_READ, 1));
  sprint("hartid = %d: vaddr 0x%x is mapped to paddr 0x%x\n", hartid,va, pa);
  return va;
}
```

```c
//@lab2 challenge3
uint64 sys_user_free_page(uint64 va) {
  int hartid = read_tp();
  user_vm_unmap((pagetable_t)current[hartid]->pagetable, va, PGSIZE, 1);
  return 0;
}
```

### minit.c
```c
#include "kernel/sync_utils.h"//lab2 challenge3
```

```c
// @lab2 challenge3
static int m_cnt = 0;//m态cnt
void m_start(uintptr_t hartid, uintptr_t dtb) {
  if(hartid==0)//spike file interface和HTIF为唯一资源，只初始化一次
 {
   // init the spike file interface (stdin,stdout,stderr)
  // functions with "spike_" prefix are all defined in codes under spike_interface/,
  // sprint is also defined in spike_interface/spike_utils.c
  spike_file_init();
  // init HTIF (Host-Target InterFace) and memory by using the Device Table Blob (DTB)
  // init_dtb() is defined above.
  init_dtb(dtb);
 }
 sync_barrier(&m_cnt,NCPU);//控制同步
 sprint("In m_start, hartid:%d\n", hartid);
 write_tp(hartid);//写入核信息
  // save the address of trap frame for interrupt in M mode to "mscratch". added @lab1_2
  write_csr(mscratch, &g_itrframe);

  // set previous privilege mode to S (Supervisor), and will enter S mode after 'mret'
  // write_csr is a macro defined in kernel/riscv.h
  write_csr(mstatus, ((read_csr(mstatus) & ~MSTATUS_MPP_MASK) | MSTATUS_MPP_S));

  // set M Exception Program Counter to sstart, for mret (requires gcc -mcmodel=medany)
  write_csr(mepc, (uint64)s_start);

  // setup trap handling vector for machine mode. added @lab1_2
  write_csr(mtvec, (uint64)mtrapvec);

  // enable machine-mode interrupts. added @lab1_3
  write_csr(mstatus, read_csr(mstatus) | MSTATUS_MIE);

  // delegate all interrupts and exceptions to supervisor mode.
  // delegate_traps() is defined above.
  delegate_traps();

  // also enables interrupt handling in supervisor mode. added @lab1_3
  write_csr(sie, read_csr(sie) | SIE_SEIE | SIE_STIE | SIE_SSIE);

  // init timing. added @lab1_3
  timerinit(hartid);

  // switch to supervisor mode (S mode) and jump to s_start(), i.e., set pc to mepc
  asm volatile("mret");
}


```