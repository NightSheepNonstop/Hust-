## lab1-1
---
- strap.c
```c
static void handle_syscall(trapframe *tf) {
  // tf->epc points to the address that our computer will jump to after the trap handling.
  // for a syscall, we should return to the NEXT instruction after its handling.
  // in RV64G, each instruction occupies exactly 32 bits (i.e., 4 Bytes)
  tf->epc += 4;

  // TODO (lab1_1): remove the panic call below, and call do_syscall (defined in
  // kernel/syscall.c) to conduct real operations of the kernel side for a syscall.
  // IMPORTANT: return value should be returned to user app, or else, you will encounter
  // problems in later experiments!
   tf->regs.a0=do_syscall(tf->regs.a0,tf->regs.a1,tf->regs.a2,tf->regs.a3,tf->regs.a4,tf->regs.a5,tf->regs.a6,tf->regs.a7);
}
```

## lab1-2
---
- mtrap.c
```c
void handle_mtrap() {
  uint64 mcause = read_csr(mcause);
  switch (mcause) {
    case CAUSE_MTIMER:
      handle_timer();
      break;
    case CAUSE_FETCH_ACCESS:
      handle_instruction_access_fault();
      break;
    case CAUSE_LOAD_ACCESS:
      handle_load_access_fault();
    case CAUSE_STORE_ACCESS:
      handle_store_access_fault();
      break;
    case CAUSE_ILLEGAL_INSTRUCTION:
      // TODO (lab1_2): call handle_illegal_instruction to implement illegal instruction
      // interception, and finish lab1_2.
     handle_illegal_instruction();

      break;
    case CAUSE_MISALIGNED_LOAD:
      handle_misaligned_load();
      break;
    case CAUSE_MISALIGNED_STORE:
      handle_misaligned_store();
      break;

    default:
      sprint("machine trap(): unexpected mscause %p\n", mcause);
      sprint("            mepc=%p mtval=%p\n", read_csr(mepc), read_csr(mtval));
      panic( "unexpected exception happened in M-mode.\n" );
      break;
  }
}
```

## lab1-3
---
- strap.c
```c
void handle_mtimer_trap() {
  sprint("Ticks %d\n", g_ticks);
  // TODO (lab1_3): increase g_ticks to record this "tick", and then clear the "SIP"
  // field in sip register.
  // hint: use write_csr to disable the SIP_SSIP bit in sip.
 g_ticks++;
 write_csr(sip,0);
}
```

## lab1 challenge1
---
### elf.c
- 第12行，定义
```c
uint64 symtab_size;
uint64 strtab_size;
char strtab[STRTAB_SIZE];
elf_sym symtab[SYMTAB_SIZE];
```
- elf_load函数,增加
```C
elf_sect_header shdr;
int shentsize = ctx->ehdr.shentsize;
// 计算shstr表头的偏移
off = ctx->ehdr.shoff + ctx->ehdr.shstrndx * shentsize;
//加载shstr表头
if (elf_fpread(ctx, (void*)&shdr, shentsize, off) != shentsize)
    return EL_EIO;
char shstr[shdr.size];
//加载shstr表
if (elf_fpread(ctx, shstr, shdr.size, shdr.offset) != shdr.size)
    return EL_EIO;
// 遍历查找每个节表的名字并匹配，从而加载symtab和strtab
off = ctx->ehdr.shoff ;
for (i = 0; i < ctx->ehdr.shnum; i++, off += shentsize) {
    if (elf_fpread(ctx, &shdr, shentsize, off) != shentsize)
        return EL_EIO;
    char* section_name = shstr + shdr.name;
    if (!strcmp(section_name, ".symtab")) {
        if (elf_fpread(ctx, symtab, shdr.size, shdr.offset) != shdr.size)
            return EL_EIO;
        symtab_size = shdr.size / sizeof(elf_sym);
    }
    if (!strcmp(section_name, ".strtab")) {
        if (elf_fpread(ctx, strtab, shdr.size, shdr.offset) != shdr.size)
            return EL_EIO;
        strtab_size = shdr.size / sizeof(char);
    }
}
```
- 增加函数
```C
uint64 elf_user_print_backtrace(int depth){
   uint64 *fp = (uint64 *)current->trapframe->regs.s0;
   //当未到达调用栈底部时循环调用
   for(; *(fp+1) && depth; depth--){
    //判断当前在那个函数中
    for(int i = 0; i < symtab_size; i++){
      if (*(fp+1)- symtab[i].st_value <= symtab[i].st_size)
      {
         sprint("%s\n", strtab + symtab[i].st_name);
         break;
      }
    }
     //回到上一个函数
     fp = (uint64 *)(*fp) - 2;
   }
 return EL_OK;
}
```

### elf.h
- 第8行
```C
#define SYMTAB_SIZE 2000
#define STRTAB_SIZE 2000
```
- 增加函数声明
```C
uint64 elf_user_print_backtrace(int depth);
```
- 增加结构体
```c
typedef struct elf_sect_header_t{  
  uint32 name;  
	uint32 type;  
	uint64 flags;  
	uint64 addr; 
	uint64 offset;
	uint64 size;  
	uint32 link;  
	uint32 info;  
	uint64 addralign;  
	uint64 entsize;  
} elf_sect_header;  
typedef struct elf_sym {  
	uint32 st_name; // symbol name, the index of string table  
    uint64 st_value; // symbol value, the virtual address  
	uint32 st_size;  
	uint8 st_info;  
	uint8 st_other;  
	uint16 st_shndx; // 符号相关节的节索引  
} elf_sym;  

```
## syscall.c
- 包含头文件
```c
#include "elf.h"
```
- 增加函数
```C
ssize_t sys_user_print_backtrace(int depth) {  
// elf_user_print_backtrace is defined in elf.c  
  elf_user_print_backtrace(depth);  
  return 0;  
} 
```
- do_syscall增加case
```C
 case SYS_user_print_backtrace:
      return sys_user_print_backtrace(a1);
```

## syscall.h

- 增加定义
```c
#define SYS_user_print_backtrace (SYS_user_base + 32)
```

## user_lib.c

- 增加函数
```c
int print_backtrace(int depth){
  return do_user_call(SYS_user_print_backtrace,depth,0,0,0,0,0,0);
}
```

## user_lib.h

- 增加声明
```c
int print_backtrace(int depth);
```

## lab1 challenge2
---
- elf.c
```cpp
//added @ lab1 challenge2
    char  elf_debug_line[8765];
```

```cpp
elf_status elf_load(elf_ctx *ctx) {
  // elf_prog_header structure is defined in kernel/elf.h
  elf_prog_header ph_addr;
  int i, off;

  // traverse the elf program segment headers
  for (i = 0, off = ctx->ehdr.phoff; i < ctx->ehdr.phnum; i++, off += sizeof(ph_addr)) {
    // read segment headers
    if (elf_fpread(ctx, (void *)&ph_addr, sizeof(ph_addr), off) != sizeof(ph_addr)) return EL_EIO;

    if (ph_addr.type != ELF_PROG_LOAD) continue;
    if (ph_addr.memsz < ph_addr.filesz) return EL_ERR;
    if (ph_addr.vaddr + ph_addr.memsz < ph_addr.vaddr) return EL_ERR;

    // allocate memory block before elf loading
    void *dest = elf_alloc_mb(ctx, ph_addr.vaddr, ph_addr.vaddr, ph_addr.memsz);

    // actual loading
    if (elf_fpread(ctx, dest, ph_addr.memsz, ph_addr.off) != ph_addr.memsz)
      return EL_EIO;
  }
  //added @ lab1 challenge2
 elf_sect_header shdr;
int shentsize = ctx->ehdr.shentsize;
// 计算shstr表头的偏移
off = ctx->ehdr.shoff + ctx->ehdr.shstrndx * shentsize;
//加载shstr表头
if (elf_fpread(ctx, (void*)&shdr, shentsize, off) != shentsize)
    return EL_EIO;
char shstr[shdr.size];
//加载shstr表
if (elf_fpread(ctx, shstr, shdr.size, shdr.offset) != shdr.size)
    return EL_EIO;
// 遍历查找每个节表的名字并匹配，从而加载.debug_line
off = ctx->ehdr.shoff ;
for (i = 0; i < ctx->ehdr.shnum; i++, off += shentsize) {
    if (elf_fpread(ctx, &shdr, shentsize, off) != shentsize)
        return EL_EIO;
    char* section_name = shstr + shdr.name;
    if (!strcmp(section_name, ".debug_line")) {
        if (elf_fpread(ctx, elf_debug_line, shdr.size, shdr.offset) != shdr.size)
            return EL_EIO;
        break;
    }
}
make_addr_line(ctx,elf_debug_line,shdr.size);
return EL_OK;
}
```
- mtrap.c
```cpp
// added @lab1 challenge2
#include "string.h"
void print_error()
{
  uint64 error_addr = read_csr(mepc);
  char path[100];
  for (int i = 0; i < current->line_ind; i++)
  {
    if (error_addr == current->line[i].addr)
    {
      //获取完整路径
      addr_line *illegal_addr_line = current->line + i ;
      size_t dir_len = strlen((char *)current->dir[current->file[illegal_addr_line->file].dir]);
      size_t file_name_len = strlen((char *)current->file[illegal_addr_line->file].file);
      size_t path_len = dir_len + file_name_len + 1;
      strcpy(path, current->dir[current->file[illegal_addr_line->file].dir]);
      path[dir_len] = '/';
      strcpy(path + dir_len + 1, current->file[illegal_addr_line->file].file);
      path[path_len] = '\0';

      //读文件
      struct stat f_stat;
      spike_file_t *f = spike_file_open(path, O_RDONLY, 0);
      spike_file_stat(f, &f_stat);
      char buffer[f_stat.st_size + 1];
      spike_file_read(f, buffer, f_stat.st_size);
      spike_file_close(f);

      //读到对应行处
      int offset = 0, line_num = 1;
      while (offset < f_stat.st_size)
      {
        int temp = offset;
        while (temp < f_stat.st_size && buffer[temp] != '\n')
          temp++; 
        if (line_num == illegal_addr_line->line )
        {
          buffer[temp] = '\0';
          sprint("Runtime error at %s:%d\n%s\n",path, illegal_addr_line->line, buffer + offset);
          break;
        }
        else
        {
          line_num++;
          offset = temp + 1;
        }
      }
      break;
    }
  }
}
```

```cpp
static void handle_instruction_access_fault()
{ // added @ lab1 challenge2
  print_error();
  panic("Instruction access fault!");
}

static void handle_load_access_fault()
{ // added @ lab1 challenge2
  print_error();
  panic("Load access fault!");
}

static void handle_store_access_fault()
{ // added @ lab1 challenge2
  print_error();
  panic("Store/AMO access fault!");
}

static void handle_illegal_instruction()
{
  // added @ lab1 challenge2
  print_error();
  panic("Illegal instruction!");
}

static void handle_misaligned_load()
{ // added @ lab1 challenge2
  print_error();
  panic("Misaligned Load!");
}

static void handle_misaligned_store()
{ // added @ lab1 challenge2
  print_error();
  panic("Misaligned AMO!");
}
```

## lab1 challenge3
---
### tips
1. spike模拟器通过HTIF与host机器的物理设备进行交互，在pke启动时，会在M mode对spike模拟器的一些虚拟设备和HTIF接口进行初始化。这个初始化的过程只能被执行一次，而非每个核都执行一次。并且在spike和HTIF初始化完成之前，需要用**同步机制**保证每个核都不会访问他们相应的资源。//m_start
2. 每个核的时钟周期和时钟中断应该是独立的，不能受其他核的干扰。即你需要分开管理每个hart上运行应用的`tick`。//g_ticks
3. 单核实验中，pke会从命令行第一个参数加载应用程序，但是本次实验需要从命令行前两个参数加载两个程序。你需要阅读并修改`elf.c`来让pke能从命令行加载第二个app。
4. 之前的实验在单核的基础上开展，且没有内存管理机制，所以user app使用到的一些内存地址是固定的（见`kernel/config.h`）。即使实验基础代码在elf文件中为两个app指定了不同的起始地址，两个核同时加载并运行的它们时，其部分内存地址也会重叠并出错。你应当想办法分离两个app的内存。//修改config.h
5. 在之前的单核实验中，有一个全局的`current`变量来指示当前正在执行的进程，以便处理中断。然而在多核环境下，每个核的中断是独立的，在处理好中断后，各个核应当恢复到其原本执行的用户app。你需要让每个核能知道他们在执行什么app。//将current变为数组
6. 在单核环境下，一个核的用户进程调用`exit`退出时，会立即关闭模拟器；而在多核环境下，这会强使其他核也停止工作。正确的退出方式是，同步等到所有核执行完毕之后再关闭模拟器。本次实验中，你应当让CPU0负责关闭模拟器。


### elf.c
```c
// fixed @lab1 challenge3
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

  info.f = spike_file_open(arg_bug_msg.argv[hartid], O_RDONLY, 0);//打开当前核的文件
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
- 将user_app变量定义为进程数组
```c
// @lab1 challenge3
process user_app[NCPU];//不同核的应用进程应该区分
```
- load_user_program
```c
//@lab1 challenge3
void load_user_program(process *proc) {
  int hartid = read_tp();
  // USER_TRAP_FRAME is a physical address defined in kernel/config.h
  proc->trapframe = (trapframe *)(uint64)USER_TRAP_FRAME(hartid);
  memset(proc->trapframe, 0, sizeof(trapframe));
  // USER_KSTACK is also a physical address defined in kernel/config.h
  proc->kstack = USER_KSTACK(hartid);
  proc->trapframe->regs.sp = USER_STACK(hartid);
  proc->trapframe->regs.tp = hartid;
  // load_bincode_from_host_elf() is defined in kernel/elf.c
  load_bincode_from_host_elf(proc);
}
```
-  更改s_start函数
```C
// @lab1 challenge3
int s_start(void) {
  int hartid = read_tp();
  sprint("hartid = %d: Enter supervisor mode...\n",hartid);
  // Note: we use direct (i.e., Bare mode) for memory mapping in lab1.
  // which means: Virtual Address = Physical Address
  // therefore, we need to set satp to be 0 for now. we will enable paging in lab2_x.
  // 
  // write_csr is a macro defined in kernel/riscv.h
  write_csr(satp, 0);

  // the application code (elf) is first loaded into memory, and then put into execution
  load_user_program(&user_app[hartid]);
  sprint("hartid = %d: Switch to user mode...\n",hartid);
  // switch_to() is defined in kernel/process.c
  switch_to(&user_app[hartid]);

  // we should never reach here.
  return 0;
}

```
### minit.c

- 增加头文件
```C
#include "kernel/sync_utils.h"//lab1 challenge3
```
- m_start
```C
// @lab1 challenge3
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
## process.c
- 将current变为数组
```C
// @lab1 challenge3
process* current[NCPU] = {NULL};
```
- switch_to函数
```C
//@lab1 challenge3
  int hartid = read_tp();
  current[hartid] = proc;
```
### process.h
```c
//@lab1 challenge3
extern process* current[NCPU];
```
### strap.c
```c
//@lab1 challenge3
static uint64 g_ticks[NCPU] = {0};
//
// added @lab1_3
//
//@ lab1 challenge3 分开管理每个hart上运行应用的tick
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
//@lab1 challenge3
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

  // we need to handle the timer trap @lab1_3.
  if (cause == CAUSE_USER_ECALL) {
    handle_syscall(current[hartid]->trapframe);
  } else if (cause == CAUSE_MTIMER_S_TRAP) {  //soft trap generated by timer interrupt in M mode
    handle_mtimer_trap();
  } else {
    sprint("smode_trap_handler(): unexpected scause %p\n", read_csr(scause));
    sprint("            sepc=%p stval=%p\n", read_csr(sepc), read_csr(stval));
    panic( "unexpected exception happened.\n" );
  }

  // continue (come back to) the execution of current[hartid] process.
  switch_to(current[hartid]);
}

```
### syscall.c
- 增加头文件
```c
//@lab1 challenge3
#include "sync_utils.h"
```
- print
```c
//@lab1 challenge3
ssize_t sys_user_print(const char* buf, size_t n) {
  int hartid = read_tp();
  sprint("hartid = %d: %s\n", hartid,buf);
  return 0;
}
```
- exit
```c
//lab1 challenge3
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
### config.h
```c
#ifndef _CONFIG_H_
#define _CONFIG_H_

// we use only one HART (cpu) in fundamental experiments
#define NCPU 2

//interval of timer interrupt. added @lab1_3
#define TIMER_INTERVAL 1000000

#define DRAM_BASE 0x80000000
//@lab1 challenge3 分离两个app的内存
#define APP_BASE 0x81000000u
#define APP_SIZE 0x4000000u
#define STACK_SIZE 0x100000u

/* we use fixed physical (also logical) addresses for the stacks and trap frames as in
 Bare memory-mapping mode */
// user stack top
#define USER_STACK(i) (APP_BASE + APP_SIZE * NCPU + STACK_SIZE * 3 * i + STACK_SIZE)

// the stack used by PKE kernel when a syscall happens
#define USER_KSTACK(i) (APP_BASE + APP_SIZE * NCPU + STACK_SIZE * 3 * i + STACK_SIZE * 2)

// the trap frame used to assemble the user "process"
#define USER_TRAP_FRAME(i) (APP_BASE + APP_SIZE * NCPU + STACK_SIZE * 3 * i + STACK_SIZE * 3)

#endif

```