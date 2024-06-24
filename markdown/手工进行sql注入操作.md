# 手工SQL注入示例

## 一、联合查询注入（回显注入）

是一种结合数据库原始报错信息和union查询的注入方式

使用场景：数据库中查询的结果能够直接在前端页面中展示出来

UNION 操作符用于将两个或多个 SELECT 语句执行的结果合并为一个结果集输出,以下是演示示例：

![](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240614141754405.png)

![](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240614142053846.png)

![image-20240614141950605](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240614141950605.png)

### 了解MySQL数据库中几个特殊的数据库

sys

mysql

performanc_schema

**information_schema**  (是MySQL的元数据信息数据库，存储了关于数据库、表及其字段、用户等的元数据,MySQL5.0及以后的版本才有这个库，也是进行SQL注入时重点关注的一个库)

### 查询时使用的关键字段信息（这个不重要）

![image-20240614143004688](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240614143004688.png)

### 操作步骤

#### （一）判断注入点

##### 判断注入点的数据类型

（1）一般情况下，看默认的输入的值与输入点的功能即可看出输入点的数据类型（默认输入时数字，则可能是数字型；默认输入是字符，则一定是字符型）。

(2)判断数字型，使用四则运算判断（如果id=n和id=(n+1)-1产生的结果一致，则可能是数字型，再尝试添加闭合符号，时sql执行出错，就能证明注入点的数据类型为数字型）

（3）判断字符型（先是报错再闭合），如果有原始数据库报错信息，就一一枚举可能得闭合符号。

（4）判断注入点是否存在，在疑似注入点后加上 and 1=1 和and 1=2,如果两次执行的页面发生了改变（也就是拼接的sql语句在数据库中执行的时候出了问题，页面就会有所改变），说明漏洞存在

#### （二）判断表中的列数

利用order by 结合折半查找的方式确定表中的列数。

#### （三）判断数据回显的位置

利用union并将原始查询置空（置空的目的，是让union前面的select查询语句查不到结果，union前面的select语句就不会输出内容，使整个sql语句只输出union后面select语句的查询内容（也就是我们想要看到的内容））来判断数据回显的位置。

#### （四）将待查询的内容替换到数据回显的位置

### 示例（以sqlli_labs第3关为例）

#### （一）判断注入点

（1）判断注入点的数据类型

![image-20240619164432384](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240619164432384.png)

![image-20240619164502843](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240619164502843.png)

可以看到输入参数id=1和id=2时返回的结果不一致，表明注入点的数据类型为**字符型**。

---

（2）判断注入点使用的闭合符号

一般情况下，输入点的闭合符号大概为**英文状态下的单引号、双引号和小括号**。在判断闭合符号时，遵循“先报错，再闭合”原则，“报错”是指让界面显示数据库**原始报错信息**。然后再使用闭合符号使sql语句形成闭合状态。

![image-20240620163513956](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620163513956.png)

此时出现了数据库原始报错信息，接着我们按照“先报错，再闭合”的思路进去进行下一步“闭合操作”

![image-20240620163609154](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620163609154.png)

可以看出，当我在末尾加上  --+   用来注释掉sql语句后面多余的部分，还是出现了数据库原始报错信息。（这里为什么使用  --+  我会在后面进行解释）。出现这种情况，说明此处数据的闭合符号不单单是一个**单引号**，必定还有其他符号和这个单引号一起组成了闭合符号。

借来继续尝试   '  号与其他符号组合，让前端界面出现数据原始报错信息。

![image-20240619175554502](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240619175554502.png)

可以看到，在加上   ')   闭合符号后，页面也出现了数据库原始报错信息。

```java
其实当我在注入点数据后加上 ')  后，在后端文件中，sql语句已经被拼接成了以下语句
SELECT * FROM `users` where id=('1')') LIMIT 0,1;
```

可以看出，拼接后的sql语句多了一组闭合符号，也就是  ')   这两个符号，此时sql语句已经不再是一条正确的sql查询语句，所以当后端文件在拼接完成sql语句并带入到数据库中执行时，必然会出错。并且报错信息在前端界面中展示了出来。

为了更直观的展示拼接后的sql语句在数据中执行的效果，我直接在数据库中进行演示。

![image-20240620140830141](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620140830141.png)

![image-20240620141019249](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620141019249.png)

从以上两张图是不是很直观的看出问题出在了哪？首先，sql语句中的1，就是我在前端中输入的参数，参数1后面的  ')   这组闭合符号，也是我在前端中通过不断尝试凑出的闭合符号。而后面青蓝色框框住的这一组   ')  闭合符号，其实是在后端文件中预设好的。以下是第3关的原代码

![image-20240620142026738](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620142026738.png)

从上图可以看出，在源代码中，确实已经有了一组  ')  闭合符号，而我在前端中又凑了一组闭合符号，而多出来的这一组闭合符号，就是让sql语句报错的根本原因。

至于为什么要在末尾加上  --+  这几个符号，这就涉及到MySQL中的注释了。在MySQL的查询语句中，除了使用 #  号来做注释意外，是不是还可以使用   --    来做注释(注意：--符号后面有一个空格)。在末尾加上  --+  符号，就是为了将sql语句中后面多余的内容给注释掉。下面我将在数据库中做演示。

![image-20240620143546756](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620143546756.png)

 为什么在URL中空格要换成  +  号，大致解释可以参考该文章https://blog.csdn.net/dongheng123/article/details/130416855

---

#### （二）判断表中的列数

一般情况下，数据库中表的字段数（列数）不会很大，一般不会超过20 ，但也存在特殊情况。通常情况下从10或者20开始，一半一半往下降，很快就能确定表中的列数。

order by 10

![image-20240620144810426](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620144810426.png)

order by 5

![image-20240620144830165](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620144830165.png)

order by 3

![image-20240620144851716](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620144851716.png)

order by 4

![image-20240620144910301](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620144910301.png)

从上面四张图可以看出，我使用order by 进行排序，当我尝试到order by 3时，前端界面展示出了数据，而尝试到order by 4时，数据有没了，这就说明表中的字段只有三个。![image-20240620145305050](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620145305050.png)

看一下数据库，表中确实只有三个字段id，username，password

---

#### （四）判断数据回显位置

表中有既然有3个字段信息，就使用union select 1,2,3  来观察，哪些数据会在前端界面中展示出来。（注意：在判断回显位置时，要将原始查询置空。）

![image-20240620150002344](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620150002344.png)

从上图可以看出，2和3出现的位置就是数据回显的位置，此时我们只需要把我们要查询的内容给替换到这两个位置的任一位置就行。比如获取当前数据库和数据库角色。

![image-20240620150518658](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620150518658.png)

可以看出当前用户为本地root用户，当前数据库为security。

---

#### （五）爆数据

爆数据是在知道库、表、列的情况下，才能获得最终数据。也就是你得知道数据库服务器中有哪些数据库，数据库中有哪些表，表中有哪些字段（列），你才能获得明确的数据。这里就设计到MySQL数据库中有一个特殊的数据库——information_schema数据库![image-20240620153024277](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620153024277.png)

这个数据库中，有三个特殊的表:schemata、tables、columns，这三个表的有哪些特殊的点呢？

##### schemata表

该表中schema_name字段，记录了该数据库服务器中所有数据库的信息。

![image-20240620153702474](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620153702474.png)

##### tables表

该表中的table_name字段，记录了数据库中所有表的信息。

![image-20240620154255224](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620154255224.png)

##### columns表

该表中的column_name字段，记录了数据库中所有字段（列）的信息。

![image-20240620154529681](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620154529681.png)

##### （1）爆库

就是我们首先得知道这个数据库服务器中有哪些数据库，只有获取到数据库的信息，才能继续往下操作。

之前说过，information_schema数据库的schemata表的schema_name字段，记录了数据库中所有数据库的信息，所以此时我们要获取所有数据库的信息，就得在information_schema库的schemata表中来操作。

![image-20240620152154748](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620152154748.png)

从上图可以看出，sql语句执行后它只列出了一个数据库名  information_schema，很显然列出的数据库不全。如何让它显示全部的数据库信息，这里就要用到MySQL中的聚合函数group_concat()，该聚合函数的作用，就是把查询出的多行数据，合并成一个字符串进行输出。![image-20240620155150063](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620155150063.png)

以下是数据库管理软件中的截图

![image-20240620155246967](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620155246967.png)

结合两张图，确实只有六个数据库，且完全对应得上。

##### （2）爆表

在知道有哪些数据库以后，我们得根据需求，去查指定数据库中有哪些表。之前说过，information_schema数据库的table表的table_name字段，记录了数据库中所有表的信息，所以要获取表的信息，就得在information_schema数据库的table表中进行操作。另外，还得根据上一步确定的数据库的信息，指定获取哪个数据库中的表

![image-20240620160158458](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620160158458.png)

##### （3）爆字段(列)

```java
http://127.0.0.1/sqli-labs-master/Less-3/?id=-1%27)%20union%20select%201,2,group_concat(column_name)%20from%20information_schema.columns%20where%20table_schema%20=%20%27security%27%20and%20table_name%20=%20%27users%27--+
```

![image-20240620160925463](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620160925463.png)

从上图可以看出，表中的字段有id，username，password

##### (4)爆数据

```java
http://127.0.0.1/sqli-labs-master/Less-3/?id=-1%27)%20union%20select%201,2,group_concat(id,username,password)%20from%20users--+
```

![image-20240620161445162](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620161445162.png)

只查询用户名和密码

![image-20240620161623085](https://raw.githubusercontent.com/XDDA11/xiaomingshuibuzhao/main/markdown/image-20240620161623085.png)

以上，就是手工进行联合查询注入的步骤及操作。

## 后续还有报错注入，布尔盲注，时间盲注，堆叠注入、宽字节注入等常见注入方式讲解，后面会继续完善。

































































































































































