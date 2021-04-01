# boom lab

直接开IDA分析那个boom程序,配合gdb动调

## phase1

判断字符串是否相等

```bash
Border relations with Canada have never been better.
```

## phase2

输入6个数第一个数必须是1，然后下一个数是上一个数的两倍

```bash
1 2 4 8 16 32
```

## phase3

需要输入的第二个数与第一个数相匹配，在ida里能看到匹配

```bash
1 311
```

## phase4 

在ida中分析逻辑，第一个输入只需要取(0,14)中间的数字就可以，第二个输入0

```
7 0 
```

## phase5

需要输入字符串的ascii码&0xf得到数组的index来拼接字符串，最后得到字符串"flyers"

```bash
ionefg
```

## phase6

有个结构体链表在里面，先读入数字，都要小于等于6，读入的数字会被处理

```c
n = 7 - n;
```

node1-6会按照处理后的值进行排序，最后会检测排序的值是否为升序

经过调试，最后答案为

```bash
4 3 2 1 6 5
```

## sercet phase

在第四关多输入一个DrEvil就可以开启隐藏关卡

建议在gdb调试一下

最后结果调出来是22

