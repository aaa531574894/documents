### 二进制的操作

#### 1.运算符

基于二进制使用的运算符主要有：

##### `&`   

与    

##### `| `    

或 

##### `~ `    

非   

所有位取反

```
 ~       00000000 00000000 00000000 00000010
         -------------------------------------
         11111111 11111111 11111111 11111101
```

##### `^ `    

异或      

二元操作符，操作两个二进制数据；两个二进制数最低位对齐，只有当两个对位数字不同时为1，相同时为0

```
eg:
        00000000 00000000 00000000 00000011
     ^  00000000 00000000 00000000 00000010
        ------------------------------------
        00000000 00000000 00000000 00000001 
```

##### `<< `    

左移运算符 

向左移动的过程中，如果移动至符号位，会侵占符号位，低位全补0

二进制数向左移动多少位
m<<n   等价于  m 乘 2的n次方

```
运算：3 << 2 
            00000000 00000000 00000000 00000011
    << 2    
    -------------------------------------------
            00000000 00000000 00000000 00001100  
            //该补码对应十进制为12 ,即 3 ×  2 的 2次
```

##### `>>`  

右移运算符 (带符号，正数右移高位补0；负数右移高位补1)

二进制数向右移动多少位
m >> n  等价于 m除以 2的n次方， 结果向上取整。

```
3.案例：
    int a = 3 >>> 2 ; 
    System.out.println(a); //结果为 0 
4.分析：
    3的二进制补码表示为：
        00000000 00000000 00000000 00000011
    运算：
        00000000 00000000 00000000 00000011
    >>>2
    ------------------------------------------
        00000000 00000000 00000000 00000000 
```

##### `>>>  ` 

无符号右移运算符 

（无论是正数还是负数，高位统统补0）

\>> 与 >>> 对正数右移没有影响，因为高位都是补0

但是对负数却有影响。



> 注：java的二进制运算都是基于补码进行的；且 byte、char、short位移时会升级为int类型位移（4个字节）；int long类型位移时不会再提升。



#### 2.二进制相关的用法

##### 一、计算某个正数的二进制表示法中 1 的个数

```java
//求解正数的二进制表示法中的 1 的位数
 private static int countBit(int num){
     int count = 0;
     for(; num > 0; count++) {
         num &= (num - 1);
     }
     return count;
 }
```

算法思路：每次for循环，都将num的二进制中最右边的 1 清除。

为什么n &= (n – 1)能清除最右边的1呢？因为从二进制的角度讲，n相当于在n - 1的最低位加上1。举个例子，8（1000）= 7（0111）+ 1（0001），所以8 & 7 = （1000）&（0111）= 0（0000），清除了8最右边的1（其实就是最高位的1，因为8的二进制中只有一个1）。再比如7（0111）= 6（0110）+ 1（0001），所以7 & 6 =（0111）&（0110）= 6（0110），清除了7的二进制表示中最右边的1（也就是最低位的1）。



##### 二、获取某个数的第 i 位（判断某个数的第 i 位是0 还是 1？）

```java
//获取 整数 num 的第 i 位的值
private static boolean getBit(int num, int i){
    return ((num & (1 << i)) != 0);//true 表示第i位为1,否则为0
}
```

##### 三、将第 i 位设置为1

```java
public static int setSpecificBitToTrue(int number ,int i ){
    return  ( number | 1 << i );
}
```



##### 四、将第 i 位设置为0（清0）

```java
public static int setSpecificBitToFalse(int number ,int i ){
    int mask = ~(1<<i);
    return number & mask;
}
```

