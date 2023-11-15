---
layout: default
title: datalab-notes
parent: cs-startup
nav_order: 2
has_children: true
---

作者：汪晨旸

本次实验由题目组成，实验报告将主要记录做题思路。  
1.bitNor 
```
/* 
 * bitNor - ~(x|y) using only ~ and & 
 *   Example: bitNor(0x6, 0x5) = 0xFFFFFFF8
 *   Legal ops: ~ &
 *   Max ops: 8
 *   Rating: 1
 */
int bitNor(int x, int y) {
  return (~x)&(~y);
}
```
德摩根律。

2.copyLSB
```
/* 
 * copyLSB - set all bits of result to least significant bit of x
 *   Example: copyLSB(5) = 0xFFFFFFFF, copyLSB(6) = 0x00000000
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 5
 *   Rating: 2
 */
int copyLSB(int x) {
  return ~(1&x)+1;
}
```
首先取最低位，然后根据1的补码每一位都是1，0的补码每一位都是0性质完成需求。

3.isEqual 
```
/* 
 * isEqual - return 1 if x == y, and 0 otherwise 
 *   Examples: isEqual(5,5) = 1, isEqual(4,5) = 0
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 5
 *   Rating: 2
 */
int isEqual(int x, int y) {
  return !(x^y);
}
```
异或的定义。

4.bitMask 
```
/* 
 * bitMask - Generate a mask consisting of all 1's 
 *   lowbit and highbit
 *   Examples: bitMask(5,3) = 0x38 00111000
 *   Assume 0 <= lowbit <= 31, and 0 <= highbit <= 31
 *   If lowbit > highbit, then mask should be all 0's
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 16
 *   Rating: 3
 */
int bitMask(int highbit, int lowbit) {
	int lowmask=(1<<lowbit)+(~0);
	int highmask=(1<<highbit<<1)+(~0);
    return (lowmask^highmask)&highmask;
}
```
(1<<lowbit)+~0是为了构造一个lowbit-1位到0位都是1的mask，(1<<highbit<<1)+(~0)构造highbit到0位都是1的mask，两个mask异或一下就可以得到答案了。
  
1<<highbit<<1而不是1<<(highbit+1)是防止highbit=31时，1<<32=1  
  
最后&highmask是为了防止highbit<lowbit,如果highbit<lowbit,由于现在异或的结果是lowbit-1位到highbit+1位都是1，&highmask正好为0,反之不会产生影响。  

5.bitCount 
```
/*
 * bitCount - returns count of number of 1's in word
 *   Examples: bitCount(5) = 2, bitCount(7) = 3
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 40
 *   Rating: 4
 */
int bitCount(int x) {
	//0b mask1=0101```,mask2=00110011,mask3=00001111 0000000011111111
	
    int mask1=0x55;
	int mask2=0x33;
    int mask3=0x0f;
	int mask4=0xff;
	int mask5=~0+(1<<16);

	mask1=(mask1<<8)|mask1;
	mask1=(mask1<<16)|mask1;
	mask2=(mask2<<8)|mask2;
	mask2=(mask2<<16)|mask2;
	mask3=(mask3<<8)|mask3;
	mask3=(mask3<<16)|mask3;
	mask4=(mask4<<16)|mask4;

	x=(x&mask1)+((x>>1)&mask1);
	x=(x&mask2)+((x>>2)&mask2);
	x=(x&mask3)+((x>>4)&mask3);
	x=(x&mask4)+((x>>8)&mask4);
	x=(x&mask5)+((x>>16)&mask5);
	
	return x;
}
```
观察一下每个二进制数。例如0b 11101100这个二进制数，每一位上的位不仅表示一个数值，并且表示了这一位数上的数字是否为1。所以bitCount的实现方法就是把每一位加起来即可。   
为了使代码优雅一点，我们采取分治的思想。首先，每两个数一组，相加并为一组，得到（还是以上面那个二进制数为例）10 01 10 00 ，然后每两组之间再相加并为一组，依次类推，直到32位都并为一组，就是最后的答案了。   
由于两个长为x的二进制数相加结果一定能用长为2*x的二进制数表示，所以不必考虑进位的问题。

6.TMax 
```
/* 
 * TMax - return maximum two's complement integer 
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 4
 *   Rating: 1
 */
int tmax(void) {
  return ~0+(1<<31);
}
```
造0x7FFFFFFF就好。

7.isNonNegative 
```
/* 
 * isNonNegative - return 1 if x >= 0, return 0 otherwise 
 *   Example: isNonNegative(-1) = 0.  isNonNegative(0) = 1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 6
 *   Rating: 3
 */
int isNonNegative(int x) {
  return !(x>>31);
}
```
把符号位提取出来取反即可。

8.addOK 
```
/* 
 * addOK - Determine if can compute x+y without overflow
 *   Example: addOK(0x80000000,0x80000000) = 0,
 *            addOK(0x80000000,0x70000000) = 1, 
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 20
 *   Rating: 3
 */
int addOK(int x, int y) {
  return (!(((x+y)^x)>>31))|(((x^y)>>31)&1);
}
```
首先，溢出的先决条件是x，y同号，即若（sign）x^y=1，则肯定不溢出。  
之后，在已知x，y同号的条件下算一下x+y是否与x（或y）异号，异号的话就发生了溢出。  

9.rempwr2 
```
/* 
 * rempwr2 - Compute x%(2^n), for 0 <= n <= 30
 *   Negative arguments should yield negative remainders
 *   Examples: rempwr2(15,2) = 3, rempwr2(-35,3) = -3
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 20
 *   Rating: 3
 */
int rempwr2(int x, int n) {
	int mask=(~0)<<n;
	int ans=x&(~mask);
	return ans+(((x&(~ans+1))>>31)&mask);	
}
```
这个题目对负数的要求很奇怪，按照我的测试题目要求应该是负数直接取低 n 位，前面补 1（低n位正好为0的话前面补0）。所以这题卡了我很久。

首先明确一个概念，对2^n取余，就相当于取低n位数，然后高位补上符号位（取余结果为0则无论正负，都返回0）。
所以先造出一个mask，低n位为0，其余位为1。之后造出一个ans取x的低n位数。
最后，由于负数取余还有一个特殊条件，就是如果取余后结果正好为0，则前面补0，即返回0，所以利用所有正数的算术补码都是负数，符号位为1，0的补码是0的性质，用x&(~ans+1)让取余结果为0的负数ans前面也补0.

事实上，在整理完后我发现，这一题遇到负数先把负数变成正数，然后取余，最后再变回负数跑出来也是满分的。你们可以试试。    


10.isLess 
```
/* 
 * isLess - if x < y  then return 1, else return 0 
 *   Example: isLess(4,5) = 1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 24
 *   Rating: 3
 */

/*x<y y-x>0 te pan fu hao*/
int isLess(int x, int y) {
	int s_x=(x>>31)&1;
	int s_y=(y>>31)&1;
	int diff=y+~x+1;
  return !!diff&(!(s_y&!s_x))&((!s_y&s_x)|(!((diff>>31)&1)));
}
```
判断x<y，即判断y-x>0，即计算y-x判断符号位。  
首先，特判y-x=0的情况，出现代表y=x，返回0；  
之后，注意加法溢出的情况，注意到本题只有x，y一正一负的时候才会溢出，故特判x正y负返回0，x负y正返回1； 
最后判断符号位，为0返回1，为1返回0。

(写完发现自己有点问题，其实可以判断x-y<0，这样符号位就不用特判0了，符号位为1直接是一个完整的情况)

11.absVal 
```
/* 
 * absVal - absolute value of x
 *   Example: absVal(-1) = 1.
 *   You may assume -TMax <= x <= TMax
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 10
 *   Rating: 4
 */
int absVal(int x) {
  return (x^(x>>31))+!!(x>>31);
}
```
题目本质上是如果x是正数就不变，x为负数取反加1（取补码）。  
注意到任何位与0异或等于它本身，与1异或代表取反，故x^(x>>31)满足取反的要求。  
!!(x>>31)就是满足根据符号位加一的需求了。  

12.isPower2
```
/*
 * isPower2 - returns 1 if x is a power of 2, and 0 otherwise
 *   Examples: isPower2(5) = 0, isPower2(8) = 1, isPower2(0) = 0
 *   Note that no negative number is a power of 2.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 20
 *   Rating: 4
 */
int isPower2(int x) {
  return !!x&!(x>>31)&(!((x&(~x+1))^x));
}
```
观察发现如果一个正数是2的幂，则这个数的2进值位只有一个1。负数按照题目要求，都不是2的幂。    
故首先判断这个数是否为负（主要是0x8000000在捣乱），之后判断这个数与它的lowbit（取最低位1，一个数与上它的算术补码就是取最低位运算）是否相等，是的话就返回一。值得注意的是，0也与它的lowbit相等，故需特判。  

13.float_neg
```
/* 
 * float_neg - Return bit-level equivalent of expression -f for
 *   floating point argument f.
 *   Both the argument and result are passed as unsigned int's, but
 *   they are to be interpreted as the bit-level representations of
 *   single-precision floating point values.
 *   When argument is NaN, return argument.
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 10
 *   Rating: 2
 */
unsigned float_neg(unsigned uf) {
  unsigned exp=(uf<<1)>>24;
  unsigned frac=(uf<<9)>>9;
  if(exp==255 && frac)  return uf;
  return uf^(1<<31);
}
```
根据要求，NaN返回自身，故特判一下。    
之后把符号位取反就行了。    

14.float_half
```
/* 
 * float_half - Return bit-level equivalent of expression 0.5*f for
 *   floating point argument f.
 *   Both the argument and result are passed as unsigned int's, but
 *   they are to be interpreted as the bit-level representation of
 *   single-precision floating point values.
 *   When argument is NaN, return argument
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
unsigned float_half(unsigned uf) {
	unsigned sign=uf&(1<<31);
	unsigned exp=(uf<<1)>>24;
	unsigned frac=(uf<<9)>>9;
	unsigned round=!((uf&3)^3);
	if(exp==255)  return uf;
	if(exp>1){
		exp-=1;
	}
	else if(exp==1){
		exp=0;
		frac=((frac>>1)|(1<<22))+round;
	}else{
		frac=(frac>>1)+round;
	}
  return sign|(exp<<23)|frac;
}
```
首先为了避免麻烦（其实主要是不在开头定义变量会报错），先判断进位。由于除二至多会丢弃一位frac，根据Round-To-Even原则，只有粘滞位为1且近似位为1的时候，才会加一。所以直接判断可能会产生Round的末两位是不是11，是的话round=1。    

之后，判断uf是否为NaN或无穷，是的话直接丢回去就好了。  

最后，根据/2之后的结果为denormal、不/2的时候是否为denormal分成三类，然后对相应的情况进行右移就好了。  

15.float_i2f 
```
/* 
 * float_i2f - Return bit-level equivalent of expression (float) x
 *   Result is returned as unsigned int, but
 *   it is to be interpreted as the bit-level representation of a
 *   single-precision floating point values.
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
unsigned float_i2f(int x) {
	unsigned sign=x&(1<<31);
	unsigned exp=0;
	unsigned frac=0;
    unsigned y= x>0?x:-x;
	unsigned yy=y; 
	
	int index=0;
	
	if(!x) return 0;

	while(y>0){
		y=y>>1;
		index++;
	}
	exp=index-1+127;
	yy= yy<<(32-index);
	frac= (yy>>8)& 0x7fffff;

	yy= yy&0xff;
	if(yy>128||(yy==128&&(frac&1))){
		frac+=1;
	}
	if(frac>>23){
		frac-=(1<<23);
		exp+=1;
	}
  return sign|(exp<<23)|frac;
}
```
首先确定x的正负，然后取x的绝对值，并转为unsigned方便右移操作。  
之后特判一下0，因为0会使exp变为126，而正确的指数表达应该指数位为0。  
然后判断x的二进制最高位是第几位，借此确定exp（注意要减一和加bias）。   
再然后为了统一，将x的最高有效位移到最高，分离出要丢弃的粘滞位（八位，因为frac是23位，再加上一个隐藏的1（已经确定x不为0，所以肯定有这个隐藏1）,32-24=8）和保留的位。  
最后根据Round-To-Even原则与round-nearest原则，看是否需要进位。值得注意的是如果frac因为进位大于23位，需要将溢出的那一位加到指数上。  
（frac= (yy>>8)& 0x7fffff; 事实上不用& 0x7fffff，它本身就是无符号数）  

欢迎关注我的csdn博客某汪922，后期会继续更新有关内容。  
原文链接：http://t.csdnimg.cn/OHr5Q