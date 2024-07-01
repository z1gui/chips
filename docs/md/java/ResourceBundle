> #  ResourceBundle读取文件时出现的问题

# 问题产生 

接手的一个运维项目代码写的乱，看到连接数据库信息直接写在class文件里面，不易管理，就是顺手写个wfc_config.properties文件，专门去存储相关的配置信息。整理好信息之后更新到服务器上就报错Can't find bundle for base name，如图所示：

![img](img/20190325132719417.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

# 可能的原因

## 1.可能是文件位置不对（第一次尝试解决，未成功）

ResourceBundle读取的文件位置在项目的src文件下，编译过后在Classes文件下面，但是我的文件就是在classes文件下面啊，所以这种方法不能解决此问题；

## 2.文件编码问题（第二次尝试解决，未成功）

通过查资料，ResourceBundle读取的properties文件的编码应该是ISO-8859-1,在更新的时候我更改过文件内容，可能保存为别的编码模式，然后我就看本地的文件就是ISO-8859-1,在本地改过之后再更新到服务器上面（保证文件的编码不会改变），但是还是没有解决问题，依然是同样的问题，说明这个跟编码没关系；

另注：当时在服务器上的文件编码类型是UTF-8，不太懂UTF-8和ISO-8859-1区别，就去补习了一下区别，简言之，UTF-8兼容ISO-8859-1，所有properties用UTF-8也是没问题（前提是没用中文把？？【存疑】）

（补习链接：[Unicode、UTF－8 和 ISO8859-1到底有什么区别 - 寻水的小鱼 - 博客园](https://www.cnblogs.com/doudou-taste/p/7351278.html)）

## 3.文件内容上的问题（第三次尝试解决，成功）

这个问题应该只是出现在我这个特殊情况上的。

百思不得其解之后求教同事，同事看了输出觉得可能打印的日志里面缺失了异常，建议我先把异常打印出来看看，我打印出异常之后发现了问题的所在：

输出忘记截图了，报出异常：java.lang.IllegalArgumentException: Malformed \uxxxx encoding

properties中不能含有\符号.  只要遇到就会报上面的错误.



![img](img/20190325135149489.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



问题解决！

## 总结

这代码有点早了，当时还没用log，异常被吃了...解决问题时候没想到这个方向，看之前的输出一直以为文件位置或者是文件编码问题，我用logger把异常给打印出来了，解决了，和路径编码没关系，是因为文件中的地址不能用\  要用 \\ 写地址的时候疏忽了

或者 最好使用“/",linux\windows通用

在这个问题上，学到最多的不是文件地址写法问题，而是要多用log输出异常，不能让异常被吃了！！！




