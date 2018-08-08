# Solution

## 地址
    ssh col@pwnable.kr -p2222 (pw:guest)

## 文件
```shell
ls -l
total 16
-r-sr-x--- 1 col_pwn col     7341 Jun 11  2014 col
-rw-r--r-- 1 root    root     555 Jun 12  2014 col.c
-r--r----- 1 col_pwn col_pwn   52 Jun 11  2014 flag
```

## 题目分析
`cat col.c`
```C
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
	int* ip = (int*)p;
	int i;
	int res=0;
	for(i=0; i<5; i++){
		res += ip[i];
	}
	return res;
}

int main(int argc, char* argv[]){
	if(argc<2){
		printf("usage : %s [passcode]\n", argv[0]);
		return 0;
	}
	if(strlen(argv[1]) != 20){
		printf("passcode length should be 20 bytes\n");
		return 0;
	}

	if(hashcode == check_password( argv[1] )){
		system("/bin/cat flag");
		return 0;
	}
	else
		printf("wrong passcode.\n");
	return 0;
}
```
题目要求输入一个参数，这个参数是一个长度为20的字符串，四个一组对应一个32位整数，五个整数加起来要等于`0x21DD09EC`

## 问题解决
只要构造出的字符在`1`到`255`之间就可以了(不能有`0`是因为它是字符串结束的标志)

使用python来解决这个问题

首先在`/tmp/`目录下建立`.py`文件: `vim /tmp/col.py`
解题脚本如下
```python
import subprocess

def i32toS(x):
        mask = 0xff
        return chr(x & mask) + chr((x >> 8) & mask ) + chr((x >> 16) & mask) + chr((x >>24) & mask)

hashcode = 0x21DD09EC

a = 0x01010101
b = hashcode - 4 * a

target = subprocess.Popen(args=['/home/col/col',i32toS(a) * 4 + i32toS(b)])
```
其中`subprocess.Popen`是启动一个进程，这里启动的是`./col`并传入了相关参数

最后`python /tmp/col.py`就可以了