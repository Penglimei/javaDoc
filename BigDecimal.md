# BigDecimal
## 用处
**浮点数之间的等值判断，基本数据类型不能用 == 来比较，包装数据类型不能用 equals 来判断，具体原理和浮点数的编码方式有关。  
应使用 BigDecimal 来定义浮点数的值，再进行浮点数的运算操作。**
### 浮点数的编码方式
>> 在计算机系统理论中，浮点数采用 IEEE 754 标准表示，编码方式是 sign+exponent+fraction。 
>>> ![浮点数编码方式](https://upload-images.jianshu.io/upload_images/1820210-61af804d90504fc0.jpg?imageMogr2/auto-orient/strip|imageView2/2/format/webp)  
>>> 符号位（sign）占用 1 位，用来表示正负数，0 表示正数，1 表示负数  
>>> 指数位（exponent）占用 8 位，用来表示指数，实际要加上偏移量  
>>> 小数位（fraction）占用 23 位，用来表示小数，不足位数补 0  

>>> **指数位决定了大小范围，小数位决定了计算精度，当十进制数值转换为二进制科学表达式后，得到的尾数位数是有可能很长甚至是无限长。
所以当使用浮点格式来存储数字的时候，实际存储的尾数是被截取或执行舍入后的近似值。
这就解释了浮点数计算不准确的问题，因为近似值和原值是有差异的。** 

```
  BigDecimal a = new BigDecimal("1.0");
  BigDecimal b = new BigDecimal("0.9");
  BigDecimal c = new BigDecimal("0.8");

  BigDecimal x = a.subtract(b); 
  BigDecimal y = b.subtract(c); 

  System.out.println(x); /* 0.1 */
  System.out.println(y); /* 0.1 */
  System.out.println(Objects.equals(x, y)); /* true */
```
## 大小比较
a.compareTo(b) : 返回 -1 表示 a 小于 b，0 表示 a 等于 b ， 1表示 a 大于 b。
```
BigDecimal a = new BigDecimal("1.0");
BigDecimal b = new BigDecimal("0.9");
System.out.println(a.compareTo(b));// 1
```
## 保留小数位
通过 setScale() 方法设置保留几位小数以及保留规则。
```
BigDecimal m = new BigDecimal("1.255433");
BigDecimal n = m.setScale(3,BigDecimal.ROUND_HALF_DOWN);
System.out.println(n);// 1.255
```
## 注意事项
***使用BigDecimal时，为了防止精度丢失，推荐使用它的 BigDecimal(String) 构造方法来创建对象。***
## 总结
BigDecimal 主要用来操作（大）浮点数，BigInteger 主要用来操作大整数（超过 long 类型）。  
BigDecimal 的实现利用到了 BigInteger, 所不同的是 BigDecimal 加入了小数位的概念

## 引用自 Snailclimb/JavaGuide 
