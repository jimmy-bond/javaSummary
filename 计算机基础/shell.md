

创建文件`touch helloworld.sh`

使脚本具有执行权限：`chmod +x helloworld.sh`

~~~shell
#!/bin/bash
#第一个shell小程序,echo 是linux中的输出命令。#是注释的意思
echo  "helloworld!"
~~~

运行脚本:`./helloworld.sh`

字符串是 shell 编程中最常用最有用的数据类型，字符串可以用单引号，也可以用双引号

```bash
#输出引用变量
echo $hello
```

~~~shell
获取字符串长度：
#!/bin/bash
#获取字符串长度
name="SnailClimb"
# 第一种方式
echo ${#name} #输出 10
# 第二种方式
expr length "$name";

~~~

### [关系运算符](https://javaguide.cn/cs-basics/operating-system/shell-intro.html#关系运算符)

关系运算符只支持数字，不支持字符串，除非字符串的值是数字。

![shell关系运算符](https://oss.javaguide.cn/github/javaguide/cs-basics/shell/64391380.jpg)

### [文件相关运算符](https://javaguide.cn/cs-basics/operating-system/shell-intro.html#文件相关运算符)

![文件相关运算符](https://oss.javaguide.cn/github/javaguide/cs-basics/shell/60359774.jpg)

### [有返回值的函数](https://javaguide.cn/cs-basics/operating-system/shell-intro.html#有返回值的函数)

**输入两个数字之后相加并返回结果：**

~~~shell
#!/bin/bash
funWithReturn(){
    echo "输入第一个数字: "
    read aNum
    echo "输入第二个数字: "
    read anotherNum
    echo "两个数字分别为 $aNum 和 $anotherNum !"
    return $(($aNum+$anotherNum))
}
funWithReturn
echo "输入的两个数字之和为 $?"

~~~

