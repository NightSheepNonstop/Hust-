## lab3-1
---
- process.c
```cpp
int do_fork( process* parent)
{
  sprint( "will fork a child from parent %d.\n", parent->pid );
  process* child = alloc_process();

  for( int i=0; i<parent->total_mapped_region; i++ ){
    // browse parent's vm space, and copy its trapframe and data segments,
    // map its code segment.
    switch( parent->mapped_info[i].seg_type ){
      case CONTEXT_SEGMENT:
        *child->trapframe = *parent->trapframe;
        break;
      case STACK_SEGMENT:
        memcpy( (void*)lookup_pa(child->pagetable, child->mapped_info[STACK_SEGMENT].va),
          (void*)lookup_pa(parent->pagetable, parent->mapped_info[i].va), PGSIZE );
        break;
      case HEAP_SEGMENT:
        // build a same heap for child process.

        // convert free_pages_address into a filter to skip reclaimed blocks in the heap
        // when mapping the heap blocks
        int free_block_filter[MAX_HEAP_PAGES];
        memset(free_block_filter, 0, MAX_HEAP_PAGES);
        uint64 heap_bottom = parent->user_heap.heap_bottom;
        for (int i = 0; i < parent->user_heap.free_pages_count; i++) {
          int index = (parent->user_heap.free_pages_address[i] - heap_bottom) / PGSIZE;
          free_block_filter[index] = 1;
        }

        // copy and map the heap blocks
        for (uint64 heap_block = current->user_heap.heap_bottom;
             heap_block < current->user_heap.heap_top; heap_block += PGSIZE) {
          if (free_block_filter[(heap_block - heap_bottom) / PGSIZE])  // skip free blocks
            continue;

          void* child_pa = alloc_page();
          memcpy(child_pa, (void*)lookup_pa(parent->pagetable, heap_block), PGSIZE);
          user_vm_map((pagetable_t)child->pagetable, heap_block, PGSIZE, (uint64)child_pa,
                      prot_to_type(PROT_WRITE | PROT_READ, 1));
        }

        child->mapped_info[HEAP_SEGMENT].npages = parent->mapped_info[HEAP_SEGMENT].npages;

        // copy the heap manager from parent to child
        memcpy((void*)&child->user_heap, (void*)&parent->user_heap, sizeof(parent->user_heap));
        break;
      case CODE_SEGMENT:
        // TODO (lab3_1): implment the mapping of child code segment to parent's
        // code segment.
        // hint: the virtual address mapping of code segment is tracked in mapped_info
        // page of parent's process structure. use the information in mapped_info to
        // retrieve the virtual to physical mapping of code segment.
        // after having the mapping information, just map the corresponding virtual
        // address region of child to the physical pages that actually store the code
        // segment of parent process.
        // DO NOT COPY THE PHYSICAL PAGES, JUST MAP THEM.
       user_vm_map(child->pagetable,parent->mapped_info[i].va,parent->mapped_info[i].npages,lookup_pa(parent->pagetable,parent->mapped_info[i].va),prot_to_type(PROT_EXEC | PROT_READ, 1));

        // after mapping, register the vm region (do not delete codes below!)
        child->mapped_info[child->total_mapped_region].va = parent->mapped_info[i].va;
        child->mapped_info[child->total_mapped_region].npages =
          parent->mapped_info[i].npages;
        child->mapped_info[child->total_mapped_region].seg_type = CODE_SEGMENT;
        child->total_mapped_region++;
        break;
    }
  }

  child->status = READY;
  child->trapframe->regs.a0 = 0;
  child->parent = parent;
  insert_to_ready_queue( child );

  return child->pid;
}

```
## lab3-2
---
- syscall.c
```cpp
ssize_t sys_user_yield() {
  // TODO (lab3_2): implment the syscall of yield.
  // hint: the functionality of yield is to give up the processor. therefore,
  // we should set the status of currently running process to READY, insert it in
  // the rear of ready queue, and finally, schedule a READY process to run.
  current->status=READY;
  insert_to_ready_queue(current);
  schedule();
  return 0;
}

```
## lab3-3
---
- strap.c
```cpp
void rrsched() {
  // TODO (lab3_3): implements round-robin scheduling.
  // hint: increase the tick_count member of current process by one, if it is bigger than
  // TIME_SLICE_LEN (means it has consumed its time slice), change its status into READY,
  // place it in the rear of ready queue, and finally schedule next process to run.
  if(current->tick_count+1>=TIME_SLICE_LEN){
    current->tick_count=0;
    current->status=READY;
    insert_to_ready_queue(current);
    schedule();
  }
  else{
    current->tick_count++;
  }
}
```

## lab3 challenge 1
---
### process.c

- do_fork函数
```c
case DATA_SEGMENT:
        {void * child_page = alloc_page();
        memcpy(child_page, (void*) lookup_pa(parent->pagetable, parent->mapped_info[i].va), PGSIZE);  
        user_vm_map(child->pagetable, parent->mapped_info[i].va, PGSIZE, (uint64)child_page,prot_to_type(PROT_READ | PROT_WRITE, 1)); 
        // register the vm region
        child->mapped_info[child->total_mapped_region].va = parent->mapped_info[i].va;
        child->mapped_info[child->total_mapped_region].npages =
          parent->mapped_info[i].npages;
        child->mapped_info[child->total_mapped_region].seg_type = DATA_SEGMENT;
        child->total_mapped_region++;
        break;}
```
- 增加函数
```c
bool is_parent(const int pid_a, const int pid_b) {
  return (pid_a==procs[pid_b].parent->pid);
}
```

### process.h

- 增加函数声明
```C
bool is_parent(const int pid_a, const int pid_b);
```

### syscall.c

- 引用外部变量
```c
extern process procs[NPROC];
```
- 修改exit
```c
ssize_t sys_user_exit(uint64 code)
{
  sprint("User exit with code:%d.\n", code);
  // added @ lab3 challenge1
  if (current->parent)
  {
    int parent_pid = current->parent->pid;
    if (procs[parent_pid].status == BLOCKED)
    {
      procs[parent_pid].status = READY;
      // return the child's pid
      procs[parent_pid].trapframe->regs.a0 = current->pid;
      insert_to_ready_queue(&procs[parent_pid]);
    }
  }
  // reclaim the current process, and reschedule. added @lab3_1
  free_process(current);
  schedule();
  return 0;
}
```
- wait函数
```c
ssize_t sys_user_wait(int pid)
{
 if (pid == -1) {
    current->status = BLOCKED;
    schedule();//转换为任意子进程
    return current->pid;
  }
  else if (pid > 0 && pid <= NPROC && is_parent(current->pid,pid))
  {
    current->status = BLOCKED;
    do
    {
      schedule();
    } while (current->pid!=pid);
    return current->pid;
  }
  else
    return -1;
}
```
- do_syscall增加case
```c
case SYS_user_wait:
    return sys_user_wait(a1);
```

### syscall.h

- 增加定义
```c
#define SYS_user_wait (SYS_user_base + 33)
```

### user_lib.c

- 增加函数
```c
void wait(int pid){
   do_user_call(SYS_user_wait,pid,0,0,0,0,0,0);
}
```

### user_lib.h

- 增加声明
```c
void wait(int pid);
```

## lab3 challenge2
---
### kernel.c
- s_start函数增加对信号量的初始化
```c
// added @lab3 challenge2
  init_sem();
```
### process.c
```c
// added @ lab3 challenge2
sem sems[NPROC];
void init_sem()
{
  memset(sems, 0, sizeof(sem) * NPROC);
  for (int i = 0; i < NPROC; i++)
  {
    sems[i].value = 0;
    sems[i].head = 0;
    sems[i].tail = 0;
    sems[i].is_occupied = 0;
  }
}
```
### process.h
- 增加数据结构
```c
// added @ lab3 challenge2
typedef struct sem_t{
  int value;
  int is_occupied;
  process* head,*tail;//当前信号量控制的进程队列的头尾
}sem;
```
- 增加函数声明
```c
// added @ lab3 challenge2
void init_sem();
```
### syscall.c
- 外部变量引用
```c
// added @lab3 challenge2
extern sem sems[NPROC];
```
- 增加函数
```c
// added@lab3 challenge2
int sys_user_sem_new(int value)
{
  for (int i = 0; i < NPROC; i++)
  {
    if (sems[i].is_occupied == 0)
    {
      sems[i].value = value;
      sems[i].is_occupied = 1;
      return i;
    }
  }
  return -1;
}
int sys_user_sem_P(int sem)
{
  if (sem < 0 || sem >= NPROC)
    return -1;
  sems[sem].value--;
  if (sems[sem].value < 0)
  {
    if (sems[sem].head == 0)
    {
      sems[sem].head = sems[sem].tail = current;
      current->queue_next = 0;
    }
    else
    {
      sems[sem].tail->queue_next = current->queue_next;
      sems[sem].tail = current;
    }
    current->status = BLOCKED; // 阻塞当前进程转进程调度
    schedule();
  }
  return 0;
}
int sys_user_sem_V(int sem)
{
  if (sem < 0 || sem >= NPROC)
    return -1;
  sems[sem].value++;
  if (sems[sem].head)
  {
    process *t = sems[sem].head;
    sems[sem].head = t->queue_next;
    if (t->queue_next == 0)
      sems[sem].tail = 0;
    insert_to_ready_queue(t);
  }
  return 0;
}
```
- do_syscall增加case
```c
// added @lab3 challenge2
  case SYS_user_sem_new:
    return sys_user_sem_new(a1);
  case SYS_user_sem_P:
    return sys_user_sem_P(a1);
  case SYS_user_sem_V:
    return sys_user_sem_V(a1);
```
### syscall.h
- 增加定义
```c
 // added @lab3 challenge2
#define SYS_user_sem_new (SYS_user_base + 6)
#define SYS_user_sem_P (SYS_user_base + 7)
#define SYS_user_sem_V (SYS_user_base + 8)
```
### user_lib.c
-  增加用户函数定义
```c
// added @lab3 challenge2
int sem_new(int value){
  return do_user_call(SYS_user_sem_new,value,0,0,0,0,0,0);
}
int sem_P(int sem){
  return do_user_call(SYS_user_sem_P,sem,0,0,0,0,0,0);
}
int sem_V(int sem){
  return do_user_call(SYS_user_sem_V,sem,0,0,0,0,0,0);
}
```
### userj_lib.h
- 增加用户函数声明
```c
// added @lab3 challenge2
int sem_new(int value);
int sem_P(int sem);
int sem_V(int sem);
```

## lab3 challenge3
---
### process.c
- do fork,HEAP_SEGMENT
```c
 case HEAP_SEGMENT:
        // build a same heap for child process.

        // convert free_pages_address into a filter to skip reclaimed blocks in the heap
        // when mapping the heap blocks
        {
          int free_block_filter[MAX_HEAP_PAGES];
          memset(free_block_filter, 0, MAX_HEAP_PAGES);
          uint64 heap_bottom = parent->user_heap.heap_bottom;
          for (int i = 0; i < parent->user_heap.free_pages_count; i++) {
            int index = (parent->user_heap.free_pages_address[i] - heap_bottom) / PGSIZE;
            free_block_filter[index] = 1;
          }
          //@lab3 challenge3
          //just map the heap blocks
          for (uint64 heap_block = current->user_heap.heap_bottom;
              heap_block < current->user_heap.heap_top; heap_block += PGSIZE) {
            if (free_block_filter[(heap_block - heap_bottom) / PGSIZE])  // skip free blocks
              continue;
            user_vm_map((pagetable_t)child->pagetable, heap_block, PGSIZE, lookup_pa(parent->pagetable,heap_block),
                        prot_to_type(PROT_READ, 1));
            pte_t *pte = page_walk(child->pagetable, heap_block, 0);
            *pte |= PTE_C;
            pte = page_walk(parent->pagetable,heap_block,0);
            *pte &= ~PTE_W;
            *pte |= PTE_C_P;
          }

          child->mapped_info[HEAP_SEGMENT].npages = parent->mapped_info[HEAP_SEGMENT].npages;

          // copy the heap manager from parent to child
          memcpy((void*)&child->user_heap, (void*)&parent->user_heap, sizeof(parent->user_heap));
          parent->user_heap.ref_count++;
          break;
        }
```

```c
//@lab3 challenge3
void copy_on_write(process *child, process *parent,uint64 va)
{
    user_vm_unmap(child->pagetable, va, PGSIZE, 0);
    void *child_pa = alloc_page();
    memcpy(child_pa, (void *)lookup_pa(parent->pagetable, va), PGSIZE);
    user_vm_map((pagetable_t)child->pagetable,va, PGSIZE, (uint64)child_pa,
                prot_to_type(PROT_WRITE | PROT_READ, 1));
    
  parent->user_heap.ref_count--;
}
void copy_to_sons(process *parent,uint64 va)
{
  for (int i = 0; i < NPROC; i++)
  {
    if (procs[i].status != FREE && procs[i].status != ZOMBIE && procs[i].parent == parent)
    {
      copy_on_write(&procs[i], parent,va);
    }
  }
}
```

### process.h
- process_heap_manager
```c
//@lab3 challenge3
  int ref_count;//引用计数
```

```c
//@lab3 challenge3
void copy_on_write(process* child,process* parent,uint64 va);
void copy_to_sons(process* parent,uint64 va);
```

### riscv.h
```c
//@lab3 challenge3
#define PTE_C (1L << 8) //1 -> copy-on-write
#define PTE_C_P (1L << 9)//1 -> copy-on-write's parent
```

### syscall.c
```c
//@lab3 challenge3
uint64 sys_user_free_page(uint64 va) {
   if(va >= current->user_heap.heap_bottom && va < current->user_heap.heap_top) {
    copy_to_sons(current);
  }
  user_vm_unmap((pagetable_t)current->pagetable, va, PGSIZE, 1);
  // add the reclaimed page to the free page list
  current->user_heap.free_pages_address[current->user_heap.free_pages_count++] = va;
  return 0;
}

```

### strap.c
```c
//@lab3 challenge3
void handle_user_page_fault(uint64 mcause, uint64 sepc, uint64 stval)
{
  sprint("handle_page_fault: %lx\n", stval);
  switch (mcause)
  {
  case CAUSE_STORE_PAGE_FAULT:
    pte_t *pte = page_walk(current->pagetable, stval, 0);
    if (pte == NULL)
    { // TODO (lab2_3): implement the operations that solve the page fault to
      // dynamically increase application stack.
      // hint: first allocate a new physical page, and then, maps the new page to the
      // virtual address that causes the page fault.
      user_vm_map((pagetable_t)current->pagetable, stval - stval % PGSIZE, PGSIZE, (uint64)alloc_page(), prot_to_type(PROT_WRITE | PROT_READ, 1));
    }
    else if (*pte & PTE_C)
    {
      copy_on_write(current,current->parent,ROUNDDOWN(stval, PGSIZE));
    }
    else if (*pte & PTE_C_P)
    {
      copy_to_sons(current,ROUNDDOWN(stval, PGSIZE));
      *pte &= ~PTE_C_P;
      *pte |= PTE_W;
    }


    break;
  default:
    sprint("unknown page fault.\n");
    break;
  }
}
```

### vmm.c
```c
//@lab3 challenge3
void user_vm_unmap(pagetable_t page_dir, uint64 va, uint64 size, int free) {
  // TODO (lab2_2): implement user_vm_unmap to disable the mapping of the virtual pages
  // in [va, va+size], and free the corresponding physical pages used by the virtual
  // addresses when if 'free' (the last parameter) is not zero.
  // basic idea here is to first locate the PTEs of the virtual pages, and then reclaim
  // (use free_page() defined in pmm.c) the physical pages. lastly, invalidate the PTEs.
  // as naive_free reclaims only one page at a time, you only need to consider one page
  // to make user/app_naive_malloc to behave correctly.
  pagetable_t pt = page_dir;
  pte_t *pte;
  for(int level = 2; level >= 0; level--) {
    pte = pt + PX(level, va);
    if(((*pte) & PTE_V) == 0) {
      return;
    }
    pt = (pagetable_t)PTE2PA(*pte);
  }
  *pte &= ~PTE_V;
  if(free) {
    free_page((void *)PTE2PA(*pte));
  }
}

```