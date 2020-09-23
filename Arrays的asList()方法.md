# Arrays.asList()
## 作用
将数组转为List集合
```java
String[] myArray = {"Apple", "Banana", "Orange"};
List<String> myList = Arrays.asList(myArray);
//上面两个语句等价于下面一条语句
List<String> myList = Arrays.asList("Apple","Banana", "Orange");
```
## 注意事项
> **Arrays.asList()将数组转换为集合后,底层其实还是数组。**  
> ![Arrays.asList()底层](https://camo.githubusercontent.com/26b4048f6dd0109fcbb839ab6be16a088427a16d/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545392539382542462545392538372538432545352542372542342545352542372542344a6176612545352542432538302545352538462539312545362538392538422d4172726179732e61734c69737428292545362539362542392545362542332539352e706e67)  
```java
public class Main {
    public static void main(String[] args) {
        String[] str = {"a1","a2","a3"};
        List<String> list = Arrays.asList(str);
        list.add(0,"b1");
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    }
}

/**
 *  运行结果：
 *  
 *  Exception in thread "main" java.lang.UnsupportedOperationException
 * 	    at java.util.AbstractList.add(AbstractList.java:148)
 * 	    at com.plm.test.Main.main(Main.java:14)
 */
```
```java
public class Main {
    public static void main(String[] args) {
        String[] str = {"a1","a2","a3"};
        List<String> list = Arrays.asList(str);
        str[0] = "b1";
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    }
}

/**
 *  运行结果：
 *  b1
 *  a2
 *  a3
 */
```

> **使用集合的修改方法:add()、remove()、clear()会抛出异常。**
```java
public class Main {
    public static void main(String[] args) {
        Integer[] str = {1,2,3};
        List list = Arrays.asList(str);
        list.add(0);
        list.remove(1);
        list.clear();
    }
}

/**
 *  运行结果：
 *  Exception in thread "main" java.lang.UnsupportedOperationException
 * 	at java.util.AbstractList.add(AbstractList.java:148)
 * 	at java.util.AbstractList.add(AbstractList.java:108)
 * 	at com.plm.test.Main.main(Main.java:12)
 *
 * 	at java.util.AbstractList.remove(AbstractList.java:161)
 * 	at com.plm.test.Main.main(Main.java:13)
 * 	
 * 	at java.util.AbstractList.remove(AbstractList.java:161)
 * 	at java.util.AbstractList$Itr.remove(AbstractList.java:374)
 * 	at java.util.AbstractList.removeRange(AbstractList.java:571)
 * 	at java.util.AbstractList.clear(AbstractList.java:234)
 * 	at com.plm.test.Main.main(Main.java:14)
 */
```

> **Arrays.asList()是泛型方法，传入的对象必须是对象数组。所以，传递的数组必须是对象数组，而不是基本类型。**  
当传入一个原生数据类型数组时，Arrays.asList() 的真正得到的参数就不是数组中的元素，而是数组对象本身！此时List 的唯一元素就是这个数组
``` java
public class Main {
    public static void main(String[] args) {
        int[] str = {1,2,3};
        List list = Arrays.asList(str);
        System.out.println(list.size());
        System.out.println(list.get(0));
        System.out.println(list.get(1));
    }
}

/**
 *  运行结果：
 * 1
 * [I@266474c2
 * Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: 1
 * 	at java.util.Arrays$ArrayList.get(Arrays.java:3841)
 * 	at com.plm.test.Main.main(Main.java:14)
 */

```
```java
public class Main {
    public static void main(String[] args) {
        Integer[] str = {1,2,3};
        List list = Arrays.asList(str);
        System.out.println(list.size());
        System.out.println(list.get(0));
        System.out.println(list.get(1));
    }
}

/**
 *  运行结果：
 *  3
 *  1
 *  2
 */

```
## 如何正确的将数组转换为ArrayList
> 最推荐的方法
`List list = new ArrayList<>(Arrays.asList("a", "b", "c"))` 
```java
public class Main {
    public static void main(String[] args) {
        Integer[] str = {1,2,3};
        List list = new ArrayList(Arrays.asList(str));
        list.add(0);
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    }
}

/**
 *  运行结果：
 *  1
 *  2
 *  3
 *  0
 */

```

## 引用自Snailclimb/JavaGuide
