_斜体_

*斜体*

**加粗**

__加粗__

<u>有下划线</u>

- 无序列表
- 无序列表
-
// - + * 的效果都是一样的

- 循环嵌套列表
  + 列表儿
  + 列表三
    * 列表四


1. 有序列表


```Java
public class StaticMethodTest {
    public static void main(String[] args) {
        Message message1 = new Message();
        Message message2 = new Message();

        for (int i = 0; i < 3; i++) {
            new Thread(new SynchronizedThreadE(message1)).start();
        }
        for (int i = 0; i < 3; i++) {
            new Thread(new SynchronizedThreadE(message2)).start();
        }
    }
}
```

| Tables        | Are           | Cool  |
| ------------- |:-------------:| -----:|
| col 3 is      | right-aligned | $1600 |
| col 2 is      | centered      |   $12 |
| zebra stripes | are neat      |    $1 |


项目     | 价格
-------- | ---
Computer | $1600
Phone    | $12
Pipe     | $1
<span style="border-bottom:2px dashed black;">下划线</span>

<font face="黑体">我是黑体字</font>

<font face="微软雅黑">我是微软雅黑</font>

<font face="STCAIYUN">我是华文彩云</font>

<font color=#0099ff size=12 face="黑体">黑体</font>

<font color=#00ffff size=3>null</font>

<font color=gray size=5>gray</font>

快捷键 `Ctrl + D` 来收藏本页
