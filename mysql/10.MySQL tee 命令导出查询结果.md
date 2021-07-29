## [mysql tee 命令](https://www.cnblogs.com/JiangLe/p/5413353.html)

tee 命令说明：

　　用过mysql的应该都会知道mysql有一个叫show 的命令，这个命令应该是SQL标准之外的一个扩展；和这个类似mysql还扩展了一个叫tee的命令。

　　tee的功能是把你的所有输入和输出都记录到日志文件中去，话说回来记到那个日志文件中去是general_log吗？当然不是！！请看tee的用法

　　mysql>tee <log_file>

 

tee 用法举例

　　<img src=".\images\643807-20160420163703945-118300480.png" alt="img" style="zoom:85%;" />

 

查看记录下来的日志：

<img src=".\images\643807-20160420163950195-1695881260.png" alt="img" style="zoom:85%;" />