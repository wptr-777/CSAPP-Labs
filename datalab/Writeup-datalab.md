# Writeup

## bitsXor

> 按位异或

只能使用～ &

德摩根律划公式

```c
int bitXor(int x, int y) { 
  return ~(~((~x)&y)&(~(x&(~y))));
}
```

## tmin

> 返回最小整数的补码

32位整数，直接1<<31,最高位为1

```c
int tmin(void){
	return 1<<31;
}
```

## allOddBits

> 判断一个数是否奇数位全为0，从第0位开始算

0xaa的二进制正好是1010的形式，我们人为构造一个0xaaaaaaaa，再去比较既可。

```c
int allOddBits(int x) {
int y = (0xAA<<24)+(0xAA<<16)+(0xAA<<8)+(0xAA);
return !(~x&y);
}
```

## negate

> 返回一个数的相反数

补码直接取反+1

```c
int negate(int x) {
  return ~x+1;
}
```

## isAsciiDigit

> 判断是否为Ascii字符

判断x的值 0x30<=x<=0x39,使用减法进行判断，取反+1，运算结束后判断符号位

```c
int isAsciiDigit(int x) {          //0x39 >= x >= 0x30
  int flag1 = !((x+(~0x30+1)>>31));  //compare with 0x30 
  int flag2 = (x+(~0x3a+1))>>31;   //compare with 0x39
  return (flag1 & flag2); 
}
```

## conditional

> 实现一个类似c语言的三目运算符，x ? y : z

需要实现分支判断

先根据x值的真假，返回全0或全1的二进制数，最后返回的结果就是-1或0。

最后根据-1/0的真假实现分支

-1 & x  =  x 

0  &  x  = 0 

```c
int conditional(int x, int y, int z) {
  int tmp = !x+(~1+1); // x=0-->0  else --> -1
  return ((tmp)&y) | (~tmp&z);
}
```

## isLessOrEqual

> 判断是否小于等于

在进行减法判断之前，可能会在符号位相同时，存在溢出的问题

所以先判断符号位，当符号位不同时直接判断

符号位相同时再进行相减

```c
int isLessOrEqual(int x, int y) {
    int sx=(x>>31)&1; // sign for x
    int sy=(y>>31)&1; // sign for y
    int flag =((y+~x+1)>>31)&1;  // sign for y-x
    return (!(sx^sy)&!flag) | ((sx^sy)&sx);
}
```

## logicalNeg

> 实现!运算符

若x为0，则x与其相反数中符号位都为0，若x不为0，则x与-x中必有一个符号位为1

```c
int logicalNeg(int x) { 
  int y = x^(~x+1);
  return  ~(y>>31);
}
```

## howManyBits

> 求解需要多少个比特位才能表示该数字

先判断符号位，若为负数，则进行取反（方便查找最高位的位置）

思路类似于二分查找

先查找高16位是否有1,高8位,4,2,1。

```c
int howManyBits(int x) {
  int x16,x8,x4,x2,x1,x0;
  int sign=x>>31;
  x = (sign&~x)|(~sign&x); // ~ depend on sign
  /* binary search */
  x16 = !!(x>>16)<<4;      // search in highest 16 bits
  x = x>>x16;              //ensure next area to search    
  x8 = !!(x>>8)<<3;        //search in highest 8bit of area
  x = x>>x8;
  x4 = !!(x>>4)<<2;
  x = x>>x4;
  x2 = !!(x>>2)<<1;
  x = x>>x2;
  x1 = !!(x>>1);
  x = x>>x1;
  x0 = x;
  return x16+x8+x4+x2+x1+x0+1;
}
```

## floatScale2

> 求出2*f的值,f为浮点数

在本题中，浮点数以unsigned int的形式传入，按照IEEE754标准解析一下即可

排除特例,∞小,0,∞大,非数值 

阶码的值分别为:0,0,255,255

前两种情况直接<<1,不过会丢失符号位,需要补上去

后两种情况直接返回参数

一般情况就阶码++

最后恢复符号位和尾数既可。

```c
unsigned floatScale2(unsigned uf) {
  int x = (uf&0x7f800000)>>23; // get the step code
  int sign = uf&(1<<31); // uf<0 0x80000000  uf>=0x00000000
  if(x==0) return uf<<1|sign;
  if(x==255) return uf;
  x++;
  if(x==255) return 0x7f800000|sign;
  return (x<<23)|(uf&0x807fffff);//recover the float
}
```

## floatFloat2Int

> 将浮点数转化为整数

将符号位，阶码，尾数部分都取出来，取尾数时需要在最前面补个1

阶码-127，算出指数位，因为尾数位在第23位的位置，所以判断一下指数大小再进行移位

最后判断结果，若结果与符号位相同，直接返回

若与符号位不同，则观察移位后的符号位，若为1，则说明存在溢出现象，返回最小整数

若为0，则说明，原来的数小于0，直接取反+1即可

```c
int floatFloat2Int(unsigned uf) {
  int sign   = uf>>31;
  int exp  = ((uf&0x7f800000)>>23)-127; // step code - 127
  int frac = (uf&0x007fffff)|0x00800000;// frac and add 1
  if(!(uf&0x7fffffff)) return 0; // check sign

  if(exp > 31) return 0x80000000; // overflow
  if(exp < 0) return 0;

  if(exp > 23) frac = frac<<(exp-23); //exp > 127
  else frac = frac>> (23-exp);        //exp <127

  if(!((frac>>31)^sign)) return frac; //sign equal
  else if(frac>>31) return 0x80000000;//ans < 0 overflow
  else return ~frac+1;//ans >=0
}
```

## floatPower2

> 计算2的x次方

阶码=x+127，直接将阶码移位，尾数部分和符号位都为0

```c
unsigned floatPower2(int x) {
  int exp = x + 127;
  int ans;
  if (exp <= 0) return 0;
  if (exp >= 255) return 0x7f800000;
  return exp<<23;
}
```

