# lab1-report

*计71 刘欣 2017011463*

### 练习一

- 练习1.1

  生成ucore.img的代码为：

  ```
  $(UCOREIMG): $(kernel) $(bootblock)
  	$(V)dd if=/dev/zero of=$@ count=10000  //生成10000个块的文件
  	$(V)dd if=$(bootblock) of=$@ conv=notrunc  //把bootblock写到第一个块
  	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc //之后为kernel
  ```

  说明生成相关文件依赖先生成的bootblock、kernel。

  生成bootblock的代码：

  ```
  $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
  		@echo + ld $@
  		$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ \
  			-o $(call toobj,bootblock)
  		@$(OBJDUMP) -S $(call objfile,bootblock) > \
  			$(call asmfile,bootblock)
  		@$(OBJCOPY) -S -O binary $(call objfile,bootblock) \
  			$(call outfile,bootblock)
  		@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
  ```

  需要先生成bootasm.o、bootmain.o、sign。

  生成bootasm.o,bootmain.o的代码为：

  ```
  bootfiles = $(call listf_cc,boot) 
  $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
  ```

  依赖于bootasm.S生成bootasm.o，命令为：

  ```
  gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
  ```

  这里添加了很多参数，指定了环境位数、禁用标准C库、不生成缓冲区溢出检测的代码、优化信息等。

  由bootmain.o生成bootmain.c：

  ```
  gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
  ```

  生成sign：

  ```
  $(call add_files_host,tools/sign.c,sign,sign)
  $(call create_target_host,sign,sign)
  ```

  生成bootblock：

  ```
  ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
  objcopy -S -O binary obj/bootblock.o obj/bootblock.out
  bin/sign obj/bootblock.out bin/bootblock
  ```

  生成kernel：

  ```
  $(kernel): tools/kernel.ld
  	 $(kernel): $(KOBJS)
  	 	@echo + ld $@
  	 	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
  	 	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
  	 	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
  ```

  相应命令为：

  ```
  ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/picirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o obj/libs/printfmt.o obj/libs/string.o
  ```

  其中依赖的.o文件由.c文件编译得到，相关命令为：

  ```
  	gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
  ```

- 练习1.2

  磁盘主引导扇区为512字节，第510个为0x55,511个为0xAA。
  
###练习2

- 练习2.1

  在gdbinit文件中有gdb的配置信息，需要补充```set architecture i8086```指定cpu信息。

  执行```make debug```进入调试界面，用```si```命令单步跟踪BIOS。

- 练习2.2

  修改gdbinit文件，用```b *address```设置断点，用```x /2i $pc```查看eip寄存器的值。

  结果为：

  ```
  Breakpoint 2, 0x00007c00 in ?? ()
  	=> 0x7c00:      cli    
  	   0x7c01:      cld    
  	   0x7c02:      xor    %eax,%eax
  	   0x7c04:      mov    %eax,%ds
  	   0x7c06:      mov    %eax,%es
  	   0x7c08:      mov    %eax,%ss 
  	   0x7c0a:      in     $0x64,%al
  	   0x7c0c:      test   $0x2,%al
  	   0x7c0e:      jne    0x7c0a
  	   0x7c10:      mov    $0xd1,%al
  ```

- 练习2.3

  通过设置断点，可以得到结果。三者相同。

  ```
  	----------------
  	IN: 
  	0x00007c00:  cli    
  	
  	----------------
  	IN: 
  	0x00007c01:  cld    
  	0x00007c02:  xor    %ax,%ax
  	0x00007c04:  mov    %ax,%ds
  	0x00007c06:  mov    %ax,%es
  	0x00007c08:  mov    %ax,%ss
  	
  	----------------
  	IN: 
  	0x00007c0a:  in     $0x64,%al
  	
  	----------------
  	IN: 
  	0x00007c0c:  test   $0x2,%al
  	0x00007c0e:  jne    0x7c0a
  	
  	----------------
  	IN: 
  	0x00007c10:  mov    $0xd1,%al
  	0x00007c12:  out    %al,$0x64
  	0x00007c14:  in     $0x64,%al
  	0x00007c16:  test   $0x2,%al
  	0x00007c18:  jne    0x7c14
  	
  	----------------
  	IN: 
  	0x00007c1a:  mov    $0xdf,%al
  	0x00007c1c:  out    %al,$0x60
  	0x00007c1e:  lgdtw  0x7c6c
  	0x00007c23:  mov    %cr0,%eax
  	0x00007c26:  or     $0x1,%eax
  	0x00007c2a:  mov    %eax,%cr0
  	
  	----------------
  	IN: 
  	0x00007c2d:  ljmp   $0x8,$0x7c32
  	
  	----------------
  	IN: 
  	0x00007c32:  mov    $0x10,%ax
  	0x00007c36:  mov    %eax,%ds
  	
  	----------------
  	IN: 
  	0x00007c38:  mov    %eax,%es
  	
  	----------------
  	IN: 
  	0x00007c3a:  mov    %eax,%fs
  	0x00007c3c:  mov    %eax,%gs
  	0x00007c3e:  mov    %eax,%ss
  	
  	----------------
  	IN: 
  	0x00007c40:  mov    $0x0,%ebp
  	
  	----------------
  	IN: 
  	0x00007c45:  mov    $0x7c00,%esp
  	0x00007c4a:  call   0x7d0d
  	
  	----------------
  	IN: 
  	0x00007d0d:  push   %ebp
  ```

  

- 练习2.4

  修改gdbinit，设置断点并查看结果：

  ```
  	b *0x8888
  	c
  	x /10i $pc
  ```

  ![1583635562365](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1583635562365.png)

### 练习3

​	进入地址为```%cs=0 $pc=0x7c00```。

​	设置flag和段寄存器：

```
	    cli
	    cld
	    xorw %ax, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %ss
```

​	设置A20：

```
	seta20.1:               
	    inb $0x64, %al      
	    testb $0x2, %al     
	    jnz seta20.1        
	    movb $0xd1, %al     
	    outb %al, $0x64     
	
	seta20.1:               
	    inb $0x64, %al       
	    testb $0x2, %al     
	    jnz seta20.1        
	    movb $0xdf, %al     
	    outb %al, $0x60      
```

​	初始化GDT：

```
	    lgdt gdtdesc
```

​	更新cs并建立堆栈：

```
	 	ljmp $PROT_MODE_CSEG, $protcseg
		movw $PROT_MODE_DSEG, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %fs
	    movw %ax, %gs
	    movw %ax, %ss
	    movl $0x0, %ebp
	    movl $start, %esp
```

​	最后转到保护模式进入boot：

```
	    movl %cr0, %eax
	    orl $CR0_PE_ON, %eax
	    movl %eax, %cr0
	    call bootmain
```

### 练习4

通过readsect函数读取硬盘扇区。

```
	static void
	readsect(void *dst, uint32_t secno) {
	    waitdisk();
	
	    outb(0x1F2, 1);                         // 设置读取扇区的数目为1
	    outb(0x1F3, secno & 0xFF);
	    outb(0x1F4, (secno >> 8) & 0xFF);		//按字节读入，读四次得到32位地址
	    outb(0x1F5, (secno >> 16) & 0xFF);
	    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
	        // 上面四条指令联合制定了扇区号
	        // 在这4个字节线联合构成的32位参数中
	        //   29-31位强制设为1
	        //   28位(=0)表示访问"Disk 0"
	        //   0-27位是28位的偏移量
	    outb(0x1F7, 0x20);                      // 0x20命令，读取扇区
	
	    waitdisk();

	    insl(0x1F0, dst, SECTSIZE / 4);         // 读取到dst位置，
	}
```

bootmain中使用对readsect封装的readseg函数。

```
	void
	bootmain(void) {
	    // 首先读取ELF的头部
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	
	    // 通过储存在头部的幻数判断是否是合法的ELF文件
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }
	
	    struct proghdr *ph, *eph;
	
	    // ELF头部有描述ELF文件应加载到内存什么位置的描述表，
	    // 先将描述表的头地址存在ph
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	    eph = ph + ELFHDR->e_phnum;
	
	    // 按照描述表将ELF文件中数据载入内存
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	    // ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
	    // ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000

	    // 根据ELF头部储存的入口信息，找到内核的入口
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
	bad:
	    outw(0x8A00, 0x8A00);
	    outw(0x8A00, 0x8E00);
	    while (1);
	}
```

### 练习5

ebp指向caller的ebp，eip为当前指令地址，由此不断更新ebp遍历可以得到整个堆栈。

具体实现见代码。

### 练习6

- 练习6.1

  8个字节，2-3位是段选择字，剩下拼起来是偏移量。

- 练习6.2

- 练习6.3

### 练习7

需要修改```T_SWITCH_TOU``` ```T_SWITCH_TOK```两个中断的行为，并在```lab1_switch_to_user```和```lab1_switch_to_kernel```中调用中断。具体实现见代码。