---
title: xv6-book Chapter 5:Interrupts and device drivers
date: 2024-03-28
description: Interrupts and device drivers
toc: 
comment: 
math:
categories:
    - 6.s081
    
---
## Device driver
设备驱动程序是一段驻留在操作系统的代码，用于管理特定的硬件设备。驱动程序负责设置好这些硬件设备，告诉设备要执行什么动作，处理设备产生的中断，并与正在等待该设备的上层用户进行交互等。\
通常情况下，大多数的设备驱动程序，都可以看成一个分上下部分的结构：顶部top half运行在内核空间中，通常由某一个进程的内核线程来运行，而底部bottom half则在中断产生时执行，大体上就是Interrupt handler（比如uartintr）。
## 同步和异步
##### [同步和异步](https://zhuanlan.zhihu.com/p/270428703)
### 同步
```c
funcA() {
    // 等待函数funcB执行完成
    funcB();

    // 继续接下来的流程
}
```

程序按顺序执行，像这种同步调用，funcA和funcB运行在同一线程中，这是最为常见的情况。\
但假如用户函数进行磁盘读取操作，会使用系统调用read，这时用户函数和被调函数运行在不同的线程中。\
所以说，**同步调用和函数与被调函数是否运行在同一个线程是没有关系的。**
### 异步
控制台输出字符就是这种例子

## Console input
控制台Console是与用户进行交互的硬件设备。键盘通过连接到RISC-V的UART串行端口传输（UART Serial-port），控制台驱动程序会接收到这些输入。控制台驱动程序处理一些特殊字符，直到完整的一行。用户程序（e.g. shell）可以使用read系统调用读取它们。\
从UART0(0x10000000)以下的地址都是device，这里有一些偏移和状态的定义
```c
#define Reg(reg) ((volatile unsigned char *)(UART0 + reg))

#define RHR 0                 // receive holding register (for input bytes)
#define THR 0                 // transmit holding register (for output bytes)
#define IER 1                 // interrupt enable register
#define IER_TX_ENABLE (1<<0)
#define IER_RX_ENABLE (1<<1)
#define FCR 2                 // FIFO control register
#define FCR_FIFO_ENABLE (1<<0)
#define FCR_FIFO_CLEAR (3<<1) // clear the content of the two FIFOs
#define ISR 2                 // interrupt status register
#define LCR 3                 // line control register
#define LCR_EIGHT_BITS (3<<0)
#define LCR_BAUD_LATCH (1<<7) // special mode to set baud rate
//咦？为啥不用4呢
#define LSR 5                 // line status register
#define LSR_RX_READY (1<<0)   // input is waiting to be read from RHR
#define LSR_TX_IDLE (1<<5)    // THR can accept another character to send

#define ReadReg(reg) (*(Reg(reg)))
#define WriteReg(reg, v) (*(Reg(reg)) = (v))
```
### 用户键盘输入到shell读出的整个过程
用户键盘输入一个字符 **->** UART在RHR中接收到该字符，发出中断 **->** xv6接收到中断，陷入trap **->** trap handler发现是外部设备中断，设备是UART **->** 调用uartintr **->** 发现RHR中有字符可读，调用consoleintr **->** 将输入字符缓冲到cons.buf中，如果读到'\n'或'ctrl+D'，说明用户输入满足一行，就唤醒consoleread **->** 读出一整行的用户输入，拷贝到用户空间中

### 1、初始化UART
xv6在kernel/main.c的main函数中调用consoleinit
consoleinit调用uartinit
```c
void
uartinit(void)
{
  // disable interrupts.
  WriteReg(IER, 0x00);

  // special mode to set baud rate.
  WriteReg(LCR, LCR_BAUD_LATCH);

  // LSB for baud rate of 38.4K.
  WriteReg(0, 0x03);

  // MSB for baud rate of 38.4K.
  WriteReg(1, 0x00);

  // leave set-baud mode,
  // and set word length to 8 bits, no parity.
  WriteReg(LCR, LCR_EIGHT_BITS);

  // reset and enable FIFOs.
  WriteReg(FCR, FCR_FIFO_ENABLE | FCR_FIFO_CLEAR);

  // enable transmit and receive interrupts.
  WriteReg(IER, IER_TX_ENABLE | IER_RX_ENABLE);

  initlock(&uart_tx_lock, "uart");
}
```
uartinit的主要工作是写入相关的控制寄存器，配置好传输的波特率，重置FIFO缓冲区，最后开启接收中断和发送完成中断。
### 2、输入字符产生中断
当用户键下键盘输入一个字符，且UART的RHR接收到时，UART就会向CPU发出中断（receive interrupt），然后执行trap handler。发现是设备中断后，trap handler会跳转到devintr。
```c
// check if it's an external interrupt or software interrupt,
// and handle it.
// returns 2 if timer interrupt,
// 1 if other device,
// 0 if not recognized.
int
devintr()
{
  uint64 scause = r_scause();

  if((scause & 0x8000000000000000L) &&
     (scause & 0xff) == 9){
    // this is a supervisor external interrupt, via PLIC.

    // irq indicates which device interrupted.
    int irq = plic_claim();

    if(irq == UART0_IRQ){
      uartintr();
    } else if(irq == VIRTIO0_IRQ){
      virtio_disk_intr();
    } else if(irq){
      printf("unexpected interrupt irq=%d\n", irq);
    }

    // the PLIC allows each device to raise at most one
    // interrupt at a time; tell the PLIC the device is
    // now allowed to interrupt again.
    if(irq)
      plic_complete(irq);

    return 1;
  } else if(scause == 0x8000000000000001L){
    // software interrupt from a machine-mode timer interrupt,
    // forwarded by timervec in kernelvec.S.

    if(cpuid() == 0){
      clockintr();
    }
    
    // acknowledge the software interrupt by clearing
    // the SSIP bit in sip.
    w_sip(r_sip() & ~2);

    return 2;
  } else {
    return 0;
  }
}
```
devintr通过scause寄存器判断发生中断的设备，由PLIC告诉CPU，发现是UART之后，devintr就会调转到uartintr。（这个就是一个interrupt handler）

### 3、uartintr
```c
// handle a uart interrupt, raised because input has
// arrived, or the uart is ready for more output, or
// both. called from trap.c.
void
uartintr(void)
{
  // read and process incoming characters.
  while(1){
    int c = uartgetc();
    if(c == -1)
      break;
    consoleintr(c);
  }

  // send buffered characters.
  acquire(&uart_tx_lock);
  uartstart();
  release(&uart_tx_lock);
}

// read one input character from the UART.
// return -1 if none is waiting.
int
uartgetc(void)
{
  if(ReadReg(LSR) & 0x01){
    // input data is ready.
    return ReadReg(RHR);
  } else {
    return -1;
  }
}
```
它首先尝试从其控制寄存器RHR中读出一个字符（我们说过RHR保持着UART接收的输入），如果有就执行控制台驱动程序bottom half，即consoleintr来处理。 


### 4、控制台接收字符
有输入字符到达控制台时会产生中断，因此跳转到控制台的中断处理程序consoleintr。
```c
struct {
  struct spinlock lock;
  
  // input
#define INPUT_BUF 128
  char buf[INPUT_BUF];
  uint r;  // Read index
  uint w;  // Write index
  uint e;  // Edit index
} cons;

// the console input interrupt handler.
// uartintr() calls this for input character.
// do erase/kill processing, append to cons.buf,
// wake up consoleread() if a whole line has arrived.
//
void
consoleintr(int c)
{
  acquire(&cons.lock);

  switch(c){
  case C('P'):  // Print process list.
    procdump();
    break;
  case C('U'):  // Kill line.
    while(cons.e != cons.w &&
          cons.buf[(cons.e-1) % INPUT_BUF] != '\n'){
      cons.e--;
      consputc(BACKSPACE);
    }
    break;
  case C('H'): // Backspace
  case '\x7f':
    if(cons.e != cons.w){
      cons.e--;
      consputc(BACKSPACE);
    }
    break;
  default:
    if(c != 0 && cons.e-cons.r < INPUT_BUF){
      c = (c == '\r') ? '\n' : c;

      // echo back to the user.
      consputc(c);

      // store for consumption by consoleread().
      cons.buf[cons.e++ % INPUT_BUF] = c;

      if(c == '\n' || c == C('D') || cons.e == cons.r+INPUT_BUF){
        // wake up consoleread() if a whole line (or end-of-file)
        // has arrived.
        cons.w = cons.e;
        wakeup(&cons.r);
      }
    }
    break;
  }
  
  release(&cons.lock);
}
```
它会将字符缓冲到cons.buf，直到cons.buf积累了一整行的输入，consoleintr才会唤醒一个用户进程的consoleread。
### 5、用户进程的consoleread
用户程序（比如shell）发出read系统调用请求后，最终将导向控制台驱动程序的top half，并执行consoleread(kernel/console.c)。
```c
// user read()s from the console go here.
// copy (up to) a whole input line to dst.
// user_dist indicates whether dst is a user
// or kernel address.
//
int
consoleread(int user_dst, uint64 dst, int n)
{
  uint target;
  int c;
  char cbuf;

  target = n;
  acquire(&cons.lock);
  while(n > 0){
    // wait until interrupt handler has put some
    // input into cons.buffer.
    while(cons.r == cons.w){
      if(myproc()->killed){
        release(&cons.lock);
        return -1;
      }
      sleep(&cons.r, &cons.lock);
    }

    c = cons.buf[cons.r++ % INPUT_BUF];

    if(c == C('D')){  // end-of-file
      if(n < target){
        // Save ^D for next time, to make sure
        // caller gets a 0-byte result.
        cons.r--;
      }
      break;
    }

    // copy the input byte to the user-space buffer.
    cbuf = c;
    if(either_copyout(user_dst, dst, &cbuf, 1) == -1)
      break;

    dst++;
    --n;

    if(c == '\n'){
      // a whole line has arrived, return to
      // the user-level read().
      break;
    }
  }
  release(&cons.lock);

  return target - n;
}
```
consoleread主要从缓冲区con.buf中读取整行的输入到用户空间中，如果cons.buf中没有字符可读，或者读完了缓冲区中的所有字符该行还未结束，shell会睡眠。
## Console output 
用户进程请求的write系统调用，最终将导向到UART的驱动程序的top half，并执行uartputc(kernel/uart.c)，如果发送区的缓存已满，驱动程序会将这个进程挂起睡眠。
```c
// add a character to the output buffer and tell the
// UART to start sending if it isn't already.
// blocks if the output buffer is full.
// because it may block, it can't be called
// from interrupts; it's only suitable for use
// by write().
void
uartputc(int c)
{
  acquire(&uart_tx_lock);

  if(panicked){
    for(;;)
      ;
  }

  while(1){
    if(((uart_tx_w + 1) % UART_TX_BUF_SIZE) == uart_tx_r){
      // buffer is full.
      // wait for uartstart() to open up space in the buffer.
      sleep(&uart_tx_r, &uart_tx_lock);
    } else {
      uart_tx_buf[uart_tx_w] = c;
      uart_tx_w = (uart_tx_w + 1) % UART_TX_BUF_SIZE;
      uartstart();
      release(&uart_tx_lock);
      return;
    }
  }
}
```
然后执行uartstart，uartstart的工作是尝试发送一个字符到THR中，如果缓冲区是空的或者发送已经就绪（THR有值），会直接返回。之后UART就会发送THR中的字符，同时唤醒一个正在睡眠的uartputc字符，表明可以继续往缓冲区写入字符。
```c
// if the UART is idle, and a character is waiting
// in the transmit buffer, send it.
// caller must hold uart_tx_lock.
// called from both the top- and bottom-half.
void
uartstart()
{
  while(1){
    if(uart_tx_w == uart_tx_r){
      // transmit buffer is empty.
      return;
    }
    
    if((ReadReg(LSR) & LSR_TX_IDLE) == 0){
      // the UART transmit holding register is full,
      // so we cannot give it another byte.
      // it will interrupt when it's ready for a new byte.
      return;
    }
    
    int c = uart_tx_buf[uart_tx_r];
    uart_tx_r = (uart_tx_r + 1) % UART_TX_BUF_SIZE;
    
    // maybe uartputc() is waiting for space in the buffer.
    wakeup(&uart_tx_r);
    
    WriteReg(THR, c);
  }
}
```
所以，第一个字符是uartputc调用uartstart发送的，每当UART发送完一个字符，就会产生transmit complete interrupt，然后跟之前一样的一系列操作，跳转到uartintr，uartintr调用uartstart，以此类推，所以后续字符都是transmit complete interrup发送的，直到缓冲区满。\
uartstart和uartputs都会很快返回，不会等待什么，所以UART暴露给用户程序或内核的uartputc接口是**异步**的。\
uartputc也提供了**同步**的版本，或者说**阻塞**的版本--uartputc_sync。
```c
// alternate version of uartputc() that doesn't 
// use interrupts, for use by kernel printf() and
// to echo characters. it spins waiting for the uart's
// output register to be empty.
void
uartputc_sync(int c)
{
  push_off();

  if(panicked){
    for(;;)
      ;
  }

  // wait for Transmit Holding Empty to be set in LSR.
  while((ReadReg(LSR) & LSR_TX_IDLE) == 0)
    ;
  WriteReg(THR, c);

  pop_off();
}
```
## Concurrency in drivers
### three concurrency dangers
1. 两个在不同CPU的线程可能同时会调用consoleread
2. 正在执行consoleread的时候，硬件可能会要求这个CPU响应console(really UART) interrupt
3. 正在执行consoleread的时候，硬件可能要求另一个CPU响应console interrupt\
还有一种需要注意的情况是，一个进程可能正在等待设备工作的完成，但是当该设备的工作完成并产生中断时，正在运行的是另一个进程。因为这种原因，中断处理程序不应该认为当前运行的进程就是它所要交付工作的进程。例如，中断处理程序简单地使用当前进程的页表来调用copyout是不安全的。因此，更好的方式是，中断处理程序只做很小一部分工作，例如将数据拷贝到缓冲区中，然后在top half中，唤醒特定的进程来完成剩下的工作。
## Timer interrupts
计时器中断产生时，首先在机器模式下产生一个软件中断，提醒内核及时处理计时器这一软件中断。在后续内核执行user trap或kernel trap的过程中，如果中断开放，就能注意到这一软件中断，从而执行计时器到时的响应逻辑事件（如调度）。\
计时器中断是在**机器模式**下处理的，内核不能运行在机器模式下，所以xv6对计时器中断的方式，和用户空间下的trap、内核空间下的trap不同。\
在xv6启动的时候，机器模式下进行了计时器的初始化。\
主要的工作有，对CLINT（core-local interruptor）进行编程，使其在一定时延后产生中断；然后设置一个类似trapframe的容器scratch，用于在机器模式下保存寄存器和CLINT的地址；最后，设置mtvec，使得机器模式的trap跳转到timervec，并且开放计时器中断。
```c
// set up to receive timer interrupts in machine mode,
// which arrive at timervec in kernelvec.S,
// which turns them into software interrupts for
// devintr() in trap.c.
void
timerinit()
{
  // each CPU has a separate source of timer interrupts.
  int id = r_mhartid();

  // ask the CLINT for a timer interrupt.
  int interval = 1000000; // cycles; about 1/10th second in qemu.
  *(uint64*)CLINT_MTIMECMP(id) = *(uint64*)CLINT_MTIME + interval;

  // prepare information in scratch[] for timervec.
  // scratch[0..3] : space for timervec to save registers.
  // scratch[4] : address of CLINT MTIMECMP register.
  // scratch[5] : desired interval (in cycles) between timer interrupts.
  uint64 *scratch = &mscratch0[32 * id];
  scratch[4] = CLINT_MTIMECMP(id);
  scratch[5] = interval;
  w_mscratch((uint64)scratch);

  // set the machine-mode trap handler.
  w_mtvec((uint64)timervec);

  // enable machine-mode interrupts.
  w_mstatus(r_mstatus() | MSTATUS_MIE);

  // enable machine-mode timer interrupts.
  w_mie(r_mie() | MIE_MTIE);
}
```
计时器中断可以在**任何时刻**发生，不管是在执行用户或内核代码。即使内核正在**临界区**中执行，也不**应该关闭计时器中断**。\
计时器的trap handler应该在一个不会干扰被中断的内核代码的地方运行，因此计时器trap handler要从机器模式下开始。计时器到时，xv6会陷入机器模式，陷入我们预先在寄存器mtvec中设置好的中断向量timervec中，timervec主要重置计时器，并且发出软中断software interrupt，然后立刻返回。通过发出软中断，CPU将处理计时器中断的任务交付给了内核，现在内核可以按照与普通中断相同的trap机制来处理该中断，显然内核也可以屏蔽该软件中断（例如当执行类似acquire的原子操作时，要关闭中断）。\
定时器中断逻辑有两部分。一部分是timervec的代码，这部分是在机器模式下运行的。另一部分是发出软件中断后，内核对定时器到时这个事件做出具体相应的中断，这部分在devintr中，可以被内核屏蔽掉，关闭中断就不会相应这个软中断。处理该软件中断的流程在devintr中。
## Real World
xv6的设备中断和计时器中断，可以发生在内核空间或用户空间下。即使是在内核空间下，计时器中断也会造成线程的切换（通过yield）。现在的操作系统中设备驱动程序的代码量，要远大于内核的代码量。\
UART驱动程序每次从控制寄存器RHR中读出一个字节大小的字符，这种模式称为编程I/O（Programmed I/O），因为我们通过软件来驱动数据传输。编程I/O的方式比较简单，但是很难用于高速率传输的场合。一种更为人熟知的方式是直接存储器访问DMA（Direct Memory Access），DMA硬件会直接从RAM中读出数据，或将数据直接写入RAM中。现代磁盘和网络设备基本都使用DMA，因为它们对传输速率的要求比较高。DMA中的驱动程序，会在RAM中准备好相应的数据后，通过写入控制寄存器来通知相应设备处理这些数据。\
设备在无法预知的时间点完成工作，并不时地让CPU的注意到它，我们使用了中断这种机制。但中断有较高的CPU开销，因此高速设备如网络、磁盘控制器等要设法减少中断的次数。一种方式是，对于一大串的输入输出请求，我们执行一次中断；另一种方式是，禁用中断，而让设备驱动程序周期性地检查设备工作是否完成，这种技术称为轮询Polling。如果设备的工作很快，轮询是高效的，但如果设备经常处于闲置状态，轮询又会浪费CPU资源。一些驱动程序采用中断和轮询相交替的方式，取决于当前设备的工作负载。\
UART驱动程序将数据先拷贝到内核的缓冲区中，再拷贝到用户空间下。如果数据传输速率低，这么做是可以的，但是两次的复制显然不太高效。因此，一些操作系统可以直接在用户缓冲区和设备之间拷贝数据，这通常结合DMA来实现。