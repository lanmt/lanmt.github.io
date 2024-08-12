---
title: copy on write lab
date: 2024-08-12
description: copy on write lab
toc: 
comment: 
math:
categories:
    - 6.s081
    
---

#### forward
只是大概思路，没有全部的代码
## Implement copy-on-write
#### 给每个物理页计数
```c
struct{
  struct spinlock lock;
  int cnt[PHYSTOP/PGSIZE+10];
}C;

void setC(uint64 pa, int num){
  acquire(&C.lock);
  C.cnt[pa/PGSIZE] = num;
  release(&C.lock);
}
void addC(uint64 pa){
  acquire(&C.lock);
  C.cnt[pa/PGSIZE]++;
  release(&C.lock);
}
void subC(uint64 pa){
  acquire(&C.lock);
  C.cnt[pa/PGSIZE]--;
  release(&C.lock);
}
int getC(uint64 pa){
  int rtn;
  acquire(&C.lock);
  rtn = C.cnt[pa/PGSIZE];
  release(&C.lock);
  return rtn;
}
```
```c
void
kfree(void *pa)
{
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");
  //free一个物理页的时候，只有当这个物理页的map数量为1才正在释放
  if(getC((uint64)pa) != 1){
    subC((uint64)pa);
    return;
  } 
  subC((uint64)pa);
  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);
  struct run *r;
  r = (struct run*)pa;
  
  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);
  if(r){
    memset((char*)r, 5, PGSIZE); // fill with junk
    setC((uint64)r, 1); // 初始化map的数量
  }
  return (void*)r;
}
```
#### Modify uvmcopy()
我们使用RSW (reserved for software) bits标记COW的页面
```c
#define COW (1L<<8)
```
```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    int f = PTE_W & (*pte);
    // i一定是对齐的，所以pa也一定是对齐的，也就是offset为0
    pa = PTE2PA(*pte);
    // 可写的页面就将COW置1，PTE_W置0
    if(f) {
      (*pte) ^= PTE_W;  
      (*pte) |= COW;
      addC((uint64)pa);
    }
    if(mappages(new, i, PGSIZE, pa, PTE_FLAGS(*pte)) != 0){
      goto err;
    }
  }
  return 0;
 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}

int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  a = PGROUNDDOWN(va);
  last = PGROUNDDOWN(va + size - 1);
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    // 如果两个虚拟地址和同一个物理地址map的话，会报remap
    // 但如果我们要设置的flag的COW位为1，也是可以的
    if((*pte & PTE_V) && !(perm & COW))
      panic("remap");
    *pte = PA2PTE(pa) | perm | PTE_V;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
```
#### Modify usertrap() to recognize page faults.
```c
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  }else if(r_scause() == 0xf || r_scause() == 0xd){

    uint64 va = PGROUNDDOWN(r_stval());
    pte_t *pte;
    if(va >= p->sz || (pte = walk(p->pagetable, va, 0)) == 0 || (*pte & PTE_V) == 0){
      p->killed = 1;
      goto err;
    }
    // 如果发生trap的这个页面被标记成COW
    if(*pte & COW){
      uint64 pa = walkaddr(p->pagetable, va);
      // 现在只有一个虚拟地址与物理地址pa map，就直接更新这个pte
      if(getC(pa) == 1){
        *pte |= PTE_W;
        *pte ^= COW;
        goto err;
      }
      char *mem;
      if((mem = kalloc()) == 0){
        p->killed = 1;
        goto err;
      }
      uint flags = PTE_FLAGS(*pte);
      flags |= PTE_W;
      flags ^= COW;

      kfree((char*)pa);
      memmove(mem, (char*)pa, PGSIZE);
      if(mappages(p->pagetable, va, PGSIZE, (uint64)mem, flags) != 0){
        p->killed = 1;
        kfree(mem);
        goto err;
      }
    }else{
      p->killed = 1;
      goto err;
    }

  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

err:;
  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```
#### Modify copyout() for a COW page
```c
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    

    pte_t *pte;
    if((pte = walk(pagetable, va0, 0)) == 0){
      panic("copyout");
      return -1;
    }
    
    if(*pte & COW){
      if(getC(pa0) == 1){
        *pte |= PTE_W;
        *pte ^= COW;
        memmove((void *)(pa0 + (dstva - va0)), src, n);
        goto err;
      }
      char *mem;
      if((mem = kalloc()) == 0){
        return -1;
      }
      uint flags = PTE_FLAGS(*pte);
      flags |= PTE_W;
      flags ^= COW;
      kfree((char*)pa0);
      memmove(mem, (char*)pa0, PGSIZE);
      if(mappages(pagetable, va0, PGSIZE, (uint64)mem, flags)!=0){
        kfree(mem);
        return -1;
      }
      memmove((void *)((uint64)mem + (dstva - va0)), src, n);
    }
    err:;
    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```
## 总结
好几个地方都少加了memmove，实验拖太久了，每次都需要重新把整个过程想一遍，希望下次可以一遍过。进度太慢，要赶紧赶一赶，现在不能在这一个实验上耗这么长时间，希望能有个学上。。