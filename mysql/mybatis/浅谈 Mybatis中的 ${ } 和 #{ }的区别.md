# [浅谈 Mybatis中的 ${ } 和 #{ }的区别](https://www.cnblogs.com/dato/p/7027949.html)

好了，真正做开发也差不多一年了。一直都是看别人的博客，自己懒得写，而且也不会写博客，今天就开始慢慢的练习一下写博客吧。前段时间刚好在公司遇到这样的问题。

**一、举例说明**

```mysql
1 select * from user where name = "dato"; 
2 
3 select * from user where name = #{name}; 
4 
5 select * from user where name = '${name}'; 
```

一般情况下，我们都不会注意到这里面有什么不一样的地方。因为这些sql都可以达到我们的目的，去查询名字叫dato的用户。

**二、区别**

动态 SQL 是 mybatis 的强大特性之一，也是它优于其他 ORM 框架的一个重要原因。mybatis 在对 sql 语句进行预编译之前，会对 sql 进行动态解析，解析为一个 BoundSql 对象，也是在此处对动态 SQL 进行处理的。在动态 SQL 解析阶段， #{ } 和 ${ } 会有不同的表现

```
select * from user where name = #{name}; 
```

\#{} 在动态解析的时候， 会解析成一个参数标记符。就是解析之后的语句是：

```
select * from user where name = ？; 
```

 

 

那么我们使用 ${}的时候

```mysql
select * from user where name = '${name}'; 
```

${}在动态解析的时候，会将我们传入的参数当做String字符串填充到我们的语句中，就会变成下面的语句

```mysql
select * from user where name = "dato"; 
```

预编译之前的 SQL 语句已经不包含变量了，完全已经是常量数据了。相当于我们普通没有变量的sql了。

综上所得， ${ } 变量的替换阶段是在动态 SQL 解析阶段，而 #{ }变量的替换是在 DBMS 中。

这是 #{} 和 ${} 我们能看到的主要的区别，除此之外，还有以下区别：

- \#方式能够很大程度防止sql注入。
- $方式无法防止Sql注入。
- $方式一般用于传入数据库对象，例如传入表名.
- **一般能用#的就别用$.**

**所以我们在使用mybatis的时候，尽量的使用#方式！！！这是大家要注意的地方**