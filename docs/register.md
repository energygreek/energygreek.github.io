---
date: 2020-08-03
tags: [c, cpp, asm]
---

# c 中 register 关键字

register 关键字是c/c++ 里面唯一能操作寄存器的指令，强制将数据存放在寄存器里，而不是放在堆栈里，这样读写速度最快。

而汇编语言强大之处是能直接操作寄存器，且可以指定使用哪些寄存器。 

通过汇编一段代码来演示register的作用  

先是c语言代码
```c
int main(){

	register int i;

	for(i = 0 ;i<10;i++){
		int j = i;
	}

}
```
对应的汇编
```asm
main:
.LFB0:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        pushq   %rbx
        .cfi_offset 3, -24
        movl    $0, %ebx
        jmp     .L2
.L3:
        movl    %ebx, -12(%rbp)
        addl    $1, %ebx
.L2:
        cmpl    $9, %ebx
        jle     .L3
        movl    $0, %eax
        movq    -8(%rbp), %rbx
        leave
        .cfi_def_cfa 7, 8
        ret

```

没有register关键字的时候 
```c
int main(){

	int i;

	for(i = 0 ;i<10;i++){
		int j = i;
	}

}
```

```asm
main:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	movl	$0, -8(%rbp)
	jmp	.L2
.L3:
	movl	-8(%rbp), %eax
	movl	%eax, -4(%rbp)
	addl	$1, -8(%rbp)
.L2:
	cmpl	$9, -8(%rbp)
	jle	.L3
	movl	$0, %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret

```

可见，当register修饰int i时， i的值存放在%ebx里，当没有register修饰时，i的值放在栈里`%rbp -8`的位置
`movl    %ebx, -12(%rbp)` 可见当register时， j的值复用了ebx的高32位。 此时的内存如下

```
--- 栈底
旧的rbp 64位
---
旧的rbx 高32位
---  j
旧的rbx 低32位
---栈顶
```

rbp表示基地址指针，指向的是函数栈的起始地址，这个起始地址只是对于当前函数而言的  
rsp表示栈顶指针，指向当前的栈顶  
每次函数调用时(main 函数被内核`_start_main__`调用):
1. 将调用者的函数栈基地址保存起来   
2. 参数压栈, 可能由调用者压栈，可能由被调用者压栈，影响的是当前的栈顶，继而栈顶影响rbp
3. 将被调用函数当前的栈顶复制给基地址寄存器。新的rbp == rsp即当前的栈顶。相当于构造了新的函数运行环境

## rbp和rsp
实例代码，main函数中调用foo函数。
```c
int foo(){
  printf("hello");
  return 1;
}

int main(){
  foo();
  return 0;
}
```

```
(gdb) disassemble 
Dump of assembler code for function main:
   0x0000555555555158 <+0>:	push   %rbp
   0x0000555555555159 <+1>:	mov    %rsp,%rbp
   0x000055555555515c <+4>:	mov    $0x0,%eax
=> 0x0000555555555161 <+9>:	call   0x555555555139 <foo>
   0x0000555555555166 <+14>:	mov    $0x0,%eax
   0x000055555555516b <+19>:	pop    %rbp
   0x000055555555516c <+20>:	ret
```

1. main 函数> `push %rbp` 将栈底寄存器的值保存到堆栈中
2. main 函数> `mov %rsp,%rbp` 将当前的栈顶寄存器的值复制到栈底寄存器
3. main 函数> `mov $0x0,%eax` 将foo函数的返回值初始化成0


进入到foo函数中
```
(gdb) disassemble 
Dump of assembler code for function foo:
   0x0000555555555139 <+0>:	push   %rbp
   0x000055555555513a <+1>:	mov    %rsp,%rbp
=> 0x000055555555513d <+4>:	lea    0xec0(%rip),%rax        # 0x555555556004
   0x0000555555555144 <+11>:	mov    %rax,%rdi
   0x0000555555555147 <+14>:	mov    $0x0,%eax
   0x000055555555514c <+19>:	call   0x555555555030 <printf@plt>
   0x0000555555555151 <+24>:	mov    $0x1,%eax
   0x0000555555555156 <+29>:	pop    %rbp
   0x0000555555555157 <+30>:	ret
End of assembler dump.
```


附录
```
General-Purpose Registers

The 64-bit versions of the 'original' x86 registers are named:

    rax - register a extended
    rbx - register b extended
    rcx - register c extended
    rdx - register d extended
    rbp - register base pointer (start of stack)
    rsp - register stack pointer (current location in stack, growing downwards)
    rsi - register source index (source for data copies)
    rdi - register destination index (destination for data copies)

The registers added for 64-bit mode are named:

    r8 - register 8
    r9 - register 9
    r10 - register 10
    r11 - register 11
    r12 - register 12
    r13 - register 13
    r14 - register 14
    r15 - register 15

These may be accessed as:

    64-bit registers using the 'r' prefix: rax, r15
    32-bit registers using the 'e' prefix (original registers: e_x) or 'd' suffix (added registers: r__d): eax, r15d
    16-bit registers using no prefix (original registers: _x) or a 'w' suffix (added registers: r__w): ax, r15w
    8-bit registers using 'h' ("high byte" of 16 bits) suffix (original registers - bits 8-15: _h): ah, bh
    8-bit registers using 'l' ("low byte" of 16 bits) suffix (original registers - bits 0-7: _l) or 'b' suffix (added registers: r__b): al, bl, r15b

Usage during syscall/function call:

    First six arguments are in rdi, rsi, rdx, rcx, r8d, r9d; remaining arguments are on the stack.
    For syscalls, the syscall number is in rax.
    Return value is in rax.
    The called routine is expected to preserve rsp,rbp, rbx, r12, r13, r14, and r15 but may trample any other registers.
```



## 参考
https://shikaan.github.io/assembly/x86/guide/2024/09/08/x86-64-introduction-hello.html