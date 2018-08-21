# Solution

## 问题描述
```
Daddy, teach me how to use random value in programming!

ssh random@pwnable.kr -p2222 (pw:guest)
```

## 问题分析
查看文件
```
random@ubuntu:~$ ls -l
total 20
-r--r----- 1 random_pwn root     49 Jun 30  2014 flag
-r-sr-x--- 1 random_pwn random 8538 Jun 30  2014 random
-rw-r--r-- 1 root       root    301 Jun 30  2014 random.c

```

查看源代码
```C
random@ubuntu:~$ cat random.c
#include <stdio.h>

int main(){
	unsigned int random;
	random = rand();	// random value!

	unsigned int key=0;
	scanf("%d", &key);

	if( (key ^ random) == 0xdeadbeef ){
		printf("Good!\n");
		system("/bin/cat flag");
		return 0;
	}

	printf("Wrong, maybe you should try 2^32 cases.\n");
	return 0;
}
```

这段代码很简单，`random`接受stl的`rand`函数的返回值，`key`接受标准输入，如果这两个值异或之后是`0xdeadbeef`，我们就可以得到flag。

我们知道，当种子相同时，rand生成的随机数序列是固定的，也就是说每次调用`./random`时变量`random`的值是不会改变的。我们尝试找出这个值。

使用gdb调试main函数的汇编代码
```
   0x00000000004005f4 <+0>:	push   %rbp
   0x00000000004005f5 <+1>:	mov    %rsp,%rbp
=> 0x00000000004005f8 <+4>:	sub    $0x10,%rsp
   0x00000000004005fc <+8>:	mov    $0x0,%eax
   0x0000000000400601 <+13>:	callq  0x400500 <rand@plt>
   0x0000000000400606 <+18>:	mov    %eax,-0x4(%rbp)
   0x0000000000400609 <+21>:	movl   $0x0,-0x8(%rbp)
   0x0000000000400610 <+28>:	mov    $0x400760,%eax
   0x0000000000400615 <+33>:	lea    -0x8(%rbp),%rdx
   0x0000000000400619 <+37>:	mov    %rdx,%rsi
   0x000000000040061c <+40>:	mov    %rax,%rdi
   0x000000000040061f <+43>:	mov    $0x0,%eax
   0x0000000000400624 <+48>:	callq  0x4004f0 <__isoc99_scanf@plt>
   0x0000000000400629 <+53>:	mov    -0x8(%rbp),%eax
   0x000000000040062c <+56>:	xor    -0x4(%rbp),%eax
   0x000000000040062f <+59>:	cmp    $0xdeadbeef,%eax
   0x0000000000400634 <+64>:	jne    0x400656 <main+98>
   0x0000000000400636 <+66>:	mov    $0x400763,%edi
   0x000000000040063b <+71>:	callq  0x4004c0 <puts@plt>
   0x0000000000400640 <+76>:	mov    $0x400769,%edi
   0x0000000000400645 <+81>:	mov    $0x0,%eax
   0x000000000040064a <+86>:	callq  0x4004d0 <system@plt>
---Type <return> to continue, or q <return> to quit---q
```
注意到`<+59>`对应的是源代码的`if( (key ^ random) == 0xdeadbeef )`一句，我们可以看到被异或的两个变量在`$rbp - 4`和`$rbp - 8`。再注意到`<+18>`处的`mov    %eax,-0x4(%rbp)`一句，说明`random`变量的地址是`$rbp - 4`，于是运行到`<+21>`后查看`$rbp - 4`处的内容
```
(gdb) x $rbp -4
0x7ffc811e2adc:	0x6b8b4567
```
于是我们就找到了`random`的值，接下来只要构造`key`使得`key`和`random`异或之后正好是`0xdeadbeef`就好了

## 问题解决
输入如下代码
```
(python -c "print 0xdeadbeef ^ 0x6b8b4567") | ./random 
```