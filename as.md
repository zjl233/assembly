# assembly

## x86

CISC: 硬件复杂，tool-chan 不行。指令集提供近似于高级语言的复杂功能

x86-32 有 8 个 general purpose register(还有其他寄存器,eip:下一条指令的地址,compare)
x86-64 有 16 个 general purpose register

iA-64 不兼容 32 位(失败)
amd64 兼容 32 位

1Byte = 8bit

1Word = 2Byte = 16bit
因为 8086 是 16 位，把这个变量占了

## MIPS

RISC: 常用指令硬件实现，复杂指令软件实现

32 个寄存器

只有 LOAD 和 STORE 指令可以访问存储器的指令系统

## 数据表示

内存的最小分割单位为 Byte
x86-32 的地址间隔为 4(8 \* 4 = 32)
x86-64 的地址间隔为 8(8 \* 8 = 64)

| x86-32 | x86-64 | memo   |
| ------ | ------ | ------ |
| 0x0000 | 0x0000 | 0x0000 |
| ...... | ...... | 0x0001 |
| ...... | ...... | 0x0002 |
| ...... | ...... | 0x0003 |
| 0x0004 | ...... | 0x0004 |
| ...... | ...... | 0x0005 |
| ...... | ...... | 0x0006 |
| ...... | ...... | 0x0007 |
| 0x0008 | 0x0008 | 0x0008 |
| ...... | ...... | 0x0009 |
| ...... | ...... | 0x0010 |

数据 0x01234567
地址 0x100

大端(Big Endian)(ip 地址用大端):
............0x100 0x101 0x102 0x103
|.....|.....|01...|23...|45...|67...|.....|.....|

小端(Little Endian):地位对低地址，高位对高地址
............0x100 0x101 0x102 0x103
|.....|.....|67...|45...|23...|01...|.....|.....|

x86 用小端

浮点数：

(-1)^s*M*2^E

|s|exp.....|frac.......|

exp 域:E
frac[1.0,2.0) 域:M

## register

x86-32(2).png

6 个段寄存器

table[selector]|offset
||
base addr

## c and assembly

gcc -O2 -S .\code.c -m32 -fno-omit-frame-pointer

### 寻址

没有
movl (%eax), (%ebx)
电路实现比较复杂

1. (R) -> Mem[Reg[R]]
2. D(R) -> Mem[Reg[R] + D]

```c
void swap(int *xp, int *yp)
{
    int t0 = *xp;
    int t1 = *yp;
    *xp = t1;
    *yp = t0;
}
```

x86-32

```assembly language
_swap:
; set up
	pushl	%ebp
	movl	%esp, %ebp
	pushl	%ebx
; body
	movl	8(%ebp), %edx
	movl	12(%ebp), %eax
	movl	(%edx), %ecx
	movl	(%eax), %ebx
	movl	%ebx, (%edx)
	movl	%ecx, (%eax)
; finish
	popl	%ebx
	popl	%ebp
	ret
```

x86-64

```
swap:
	pushq	%rbp
	.seh_pushreg	%rbp
	movq	%rsp, %rbp
	.seh_setframe	%rbp, 0
	.seh_endprologue
; body
	movl	(%rcx), %eax
	movl	(%rdx), %r8d
	movl	%r8d, (%rcx)
	movl	%eax, (%rdx)
;
	popq	%rbp
	ret
```

32 位系统用栈来传递参数
64 位系统使用寄存器来传递参数

leal 8(,%eax,4), %eax
只进行数学运算，不去除内存中的值

### control flow

condition codes (CCs)

arithmetic ops set CCs implicitly
t = a + b
cf (carry flag) 用于检测无符号整数运算的溢出
zf (zero flag) if t == 0
sf (sign flag) if t < 0
of (overflowing flag) (a > 0 && b > 0 && t < 0 || a < 0 && b < 0 && t > 0)

comparison
cmpl src2, src1 -> src1 (< > ==) src2
// like computing src1 - src2
cf=1 if carry out from msb(most significant bit)
zf=1 if (src1==src2)
sf=1 if (src1-src2 < 0)
of=1 if two's complement under/overflow

---

x86-64

xorl %eax, %eax
%eax 清零，自动进行零扩展，影响到 %rax

ref:在 64 位体系下，任何 32 位操作都会自动进行 0 拓展

---

分支：

流水线优化：用 cmov(把两个分支都算一遍), 代替 jp
if (cond) state1 else state2
n 条流水线会同时处理 n 条指令的不同部分
防止 cancel 造成的性能损失

### 程序运行栈

---

栈底 addr
| | ⬇
| | ⬇
| | ⬇
| | ⬇
栈顶

---

func(int a, int b, int c)

c 16
b 12
c 8
ret 4
0

esp:栈顶
压栈:esp - 4

call label
将 eip 压栈()

调用者负责保存:%eax %edx %ecx
被调用者负责保存:%ebx %esi %edi
特殊用:%esp %ebp
64 位 %rbp 没有特殊用途

调用过程:

1. 保存 pre %ebp
2. %ebp = %esp
3. 保存必要的寄存器

x86-64
前 6 个参数用寄存器来传
后面的用栈
