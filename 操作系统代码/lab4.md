## lab4-1
---
- rfs.c
```cpp
free_dinode->size=0;
free_dinode->type=R_FILE;
free_dinode->nlinks=1;
free_dinode->blocks=0;
```

## lab4-2
---
- rfs.c
```cpp
int rfs_readdir(struct vinode *dir_vinode, struct dir *dir, int *offset) {
  int total_direntrys = dir_vinode->size / sizeof(struct rfs_direntry);
  int one_block_direntrys = RFS_BLKSIZE / sizeof(struct rfs_direntry);

  int direntry_index = *offset;
  if (direntry_index >= total_direntrys) {
    // no more direntry
    return -1;
  }

  // reads a directory entry from the directory cache stored in vfs inode.
  struct rfs_dir_cache *dir_cache =
      (struct rfs_dir_cache *)dir_vinode->i_fs_info;
  struct rfs_direntry *p_direntry = dir_cache->dir_base_addr + direntry_index;

  // TODO (lab4_2): implement the code to read a directory entry.
  // hint: in the above code, we had found the directory entry that located at the
  // *offset, and used p_direntry to point it.
  // in the remaining processing, we need to return our discovery.
  // the method of returning is to popular proper members of "dir", more specifically,
  // dir->name and dir->inum.
  // note: DO NOT DELETE CODE BELOW PANIC.
  strcpy(dir->name,p_direntry->name);
  dir->inum=p_direntry->inum;
  // DO NOT DELETE CODE BELOW.
  (*offset)++;
  return 0;
}
```

## lab4-3
---
- rfs.c
```cpp
int rfs_link(struct vinode *parent, struct dentry *sub_dentry, struct vinode *link_node) {
  // TODO (lab4_3): we now need to establish a hard link to an existing file whose vfs
  // inode is "link_node". To do that, we need first to know the name of the new (link)
  // file, and then, we need to increase the link count of the existing file. Lastly, 
  // we need to make the changes persistent to disk. To know the name of the new (link)
  // file, you need to stuty the structure of dentry, that contains the name member;
  // To incease the link count of the existing file, you need to study the structure of
  // vfs inode, since it contains the inode information of the existing file.
  //
  // hint: to accomplish this experiment, you need to:
  // 1) increase the link count of the file to be hard-linked;
  // 2) append the new (link) file as a dentry to its parent directory; you can use 
  //    rfs_add_direntry here.
  // 3) persistent the changes to disk. you can use rfs_write_back_vinode here.
  //
  link_node->nlinks++;
  rfs_add_direntry(parent,sub_dentry->name,link_node->inum);
  rfs_write_back_vinode(link_node);
  return 0;
}
```
## lab4 challenge1
---
- process.c
```cpp
void getAbsolutePath(char *relative_path, char *abspath)
{
  struct dentry *p = current->pfiles->cwd;
  char *tokens;
  tokens = strtok(relative_path, "/");
  if (strcmp(tokens, "..") == 0)
    p = p->parent;
  while (p)
  {
    while (p)
    {
      char tmp[MAX_DENTRY_NAME_LEN];
      memset(tmp, '\0', MAX_DENTRY_NAME_LEN);
      strcpy(tmp, abspath);
      memset(abspath, '\0', MAX_DENTRY_NAME_LEN);
      strcpy(abspath, p->name);
      if (p->parent != NULL)
      {
        strcat(abspath, "/");
      }
      strcat(abspath, tmp);
      p = p->parent;
    }
  }
  if (strcmp(tokens, "..") == 0) 
    strcat(abspath, relative_path + 3);
  else if (strcmp(tokens, ".") == 0)
    strcat(abspath, relative_path + 2);
  else
    strcat(abspath, relative_path);
}

```
- process.h
```cpp
void getAbsolutePath(char* relative_path,char* abspath);
```
- syscall.c
```cpp
ssize_t sys_user_open(char *pathva, int flags)
{
  char *pathpa = (char *)user_va_to_pa((pagetable_t)(current->pagetable), pathva);
  //added @ lab4 challenge1
  char abs_path[MAX_DENTRY_NAME_LEN];
  memset(abs_path, '\0', MAX_DENTRY_NAME_LEN);
  getAbsolutePath(pathpa,abs_path);
  return do_open(abs_path, flags);
}

```

```cpp
// added @ lab4 challenge1
ssize_t sys_user_rcwd(char* pathva) {
    char *pathpa = (char *)user_va_to_pa((pagetable_t)(current->pagetable), (void *)pathva);
    struct dentry *p = current->pfiles->cwd;
    if (p->parent == NULL) {
        strcpy(pathpa, "/");
    } else {
        while (p) {
            char tmp[MAX_DENTRY_NAME_LEN];
            memset(tmp, '\0', MAX_DENTRY_NAME_LEN);
            strcpy(tmp, pathpa); 
            memset(pathpa, '\0', MAX_DENTRY_NAME_LEN);
            strcpy(pathpa, p->name); 
            if (p->parent != NULL) {
                strcat(pathpa, "/"); 
            }
            strcat(pathpa, tmp);
            p = p->parent;
        }
      while (pathpa[strlen(pathpa) - 1]!='/')
      {
        pathpa[strlen(pathpa) - 1] = '\0';
      }
      pathpa[strlen(pathpa) - 1] = '\0';
        
    }
     return 0;
}
ssize_t sys_user_ccwd(const char *pathva)
{
    char *pathpa = (char *)user_va_to_pa((pagetable_t)(current->pagetable), (void*)pathva);
    struct dentry *current_directory = current->pfiles->cwd;
    char abs_path[MAX_DEVICE_NAME_LEN];
    memset(abs_path, '\0', MAX_DENTRY_NAME_LEN);
    getAbsolutePath(pathpa, abs_path);
    int fd = do_opendir(abs_path);
    current->pfiles->cwd = current->pfiles->opened_files[fd].f_dentry;
    do_closedir(fd);
    return 0;
}
```
dosyscall增加case
```cpp
 case SYS_user_rcwd:
    return sys_user_rcwd((char *)a1);
  case SYS_user_ccwd:
    return sys_user_ccwd((const char *)a1);
```
- syscall.h
```cpp
#define SYS_user_rcwd (SYS_user_base + 30)
#define SYS_user_ccwd (SYS_user_base + 31)
```

## lab4 challenge2
---
### elf.c
- elf_fpread
```c
// @lab4 challenge2
static uint64 elf_fpread(elf_ctx *ctx, void *dest, uint64 nb, uint64 offset) {
  elf_info *msg = (elf_info *)ctx->info;
  // call spike file utility to load the content of elf file into memory.
  // spike_file_pread will read the elf file (msg->f) from offset to memory (indicated by
  // *dest) for nb bytes.
  vfs_lseek(msg->f, offset, SEEK_SET);
  return vfs_read(msg->f, dest, nb);
}
```
- load_bincode_from_host_elf
	- 使用vfs文件系统
```c
//@lab4 challenge2

  info.f = vfs_open(arg_bug_msg.argv[0], O_RDONLY);
```

```c
 //@lab4 challenge2
  vfs_close(info.f);
```

- elf_reload
```c
//@lab4 challenge2
elf_status elf_reload(elf_ctx *ctx, elf_info *info) {
    for (int i = 0; i < ctx->ehdr.phnum; ++i) {
      // 在每次迭代中，读取一个程序头信息
        elf_prog_header ph_addr;
        if (elf_fpread(ctx, (void *)&ph_addr, sizeof(ph_addr), ctx->ehdr.phoff + i * sizeof(ph_addr)) != sizeof(ph_addr))
            return EL_EIO;

        if (ph_addr.type != ELF_PROG_LOAD) continue;
        if (ph_addr.memsz < ph_addr.filesz || ph_addr.vaddr + ph_addr.memsz < ph_addr.vaddr) return EL_ERR;

        bool is_executable = (ph_addr.flags == (SEGMENT_READABLE | SEGMENT_EXECUTABLE));
        bool is_writable = (ph_addr.flags == (SEGMENT_READABLE | SEGMENT_WRITABLE));
        bool get_data_segment = FALSE;
        for (int j = 0; j < PGSIZE / sizeof(mapped_region); ++j) {
        if(is_writable&&info->p->mapped_info[j].seg_type == DATA_SEGMENT) get_data_segment=TRUE;
      // 检查是否已经有相同类型的段，如果有，则解除映射
      // 然后分配内存并读取段内容到分配的内存中，最后更新映射信息列表
            if ((is_executable && info->p->mapped_info[j].seg_type == CODE_SEGMENT) ||
                (is_writable && info->p->mapped_info[j].seg_type == DATA_SEGMENT)) {
                sprint("%s_SEGMENT added at mapped info offset:%d\n", is_executable ? "CODE" : "DATA", j);
                user_vm_unmap(info->p->pagetable, info->p->mapped_info[j].va, PGSIZE, 1);
                void *dest = elf_alloc_mb(ctx, ph_addr.vaddr, ph_addr.vaddr, ph_addr.memsz);
                if (elf_fpread(ctx, dest, ph_addr.memsz, ph_addr.off) != ph_addr.memsz)
                    return EL_EIO;
                info->p->mapped_info[j].va = ph_addr.vaddr;
                info->p->mapped_info[j].npages = 1;
                info->p->mapped_info[j].seg_type = is_executable ? CODE_SEGMENT : DATA_SEGMENT;
                break;
            }
        }

        if (is_writable&&!get_data_segment) {
            void *dest = elf_alloc_mb(ctx, ph_addr.vaddr, ph_addr.vaddr, ph_addr.memsz);
            if (elf_fpread(ctx, dest, ph_addr.memsz, ph_addr.off) != ph_addr.memsz) return EL_EIO;
            for (int j = 0; j < PGSIZE / sizeof(mapped_region); ++j) {
                if (info->p->mapped_info[j].va == 0) {
                    sprint("DATA_SEGMENT added at mapped info offset:%d\n", j);
                    info->p->mapped_info[j].va = ph_addr.vaddr;
                    info->p->mapped_info[j].npages = 1;
                    info->p->mapped_info[j].seg_type = DATA_SEGMENT;
                    ++info->p->total_mapped_region;
                    break;
                }
            }
        }
    }
    return EL_OK;
}

```

### elf.h
```c
//@lab4 challenge2
typedef struct elf_info_t {
  struct file *f;
  process *p;
} elf_info;
elf_status elf_reload(elf_ctx *ctx, elf_info *info);
```
### process.c
```c
//@ lab4 challenge2
int do_exec(char* pathname){
  struct file * file = vfs_open(pathname,O_RDONLY);
    elf_ctx elfloader;
    elf_info info;
    info.f = file;
    info.p = current;
    if(elf_init(&elfloader, &info) != EL_OK) {
        return -1;
    }
    if(elf_reload(&elfloader, &info) != EL_OK){
        return -1;
    }
    current->trapframe->epc = elfloader.ehdr.entry;
    vfs_close(file);
    sprint("Application program entry point (virtual address): 0x%lx\n", current->trapframe->epc);
    return 0;
}
```
### process.h
```c
// @lab4 challenge2
int do_exec(char* pathname);
```
### syscall.c
```c
//@lab4 challenge2
ssize_t sys_user_exec(char * pathname){
  char *pa = (char *)user_va_to_pa((pagetable_t)(current->pagetable), (void *)pathname);
  return do_exec(pa);
}
```

```c
 //@lab4 challenge2
    case SYS_user_exec:
      return sys_user_exec((char *)a1);
```

### syscall.h
```c
// @lab4 challenge2
#define SYS_user_exec (SYS_user_base + 30)
```

### user_lib.c
```c
// @lab4 challenge2
int exec(const char* pathname){
  return do_user_call(SYS_user_exec,(uint64)pathname,0,0,0,0,0,0);
}
```
### user_lib.h
```c
// @lab4 challenge2
int exec(const char *path);
```