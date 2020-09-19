# Python 数据库 API 规范 2.0

[https://www.python.org/dev/peps/pep-0249/](https://www.python.org/dev/peps/pep-0249/)

# 1.介绍

更多资源：[https://wiki.python.org/moin/DatabaseProgramming](https://wiki.python.org/moin/DatabaseProgramming)

1.0版本规范：[https://www.python.org/dev/peps/pep-0248](https://www.python.org/dev/peps/pep-0248)


# 2.模块接口

## 2.1 构造器

通过 connection 对象访问数据库。模块必须提供下面的构造：

* connect(parameters...)
    * 创建一个数据库连接
    * 返回一个[Connection](#3.连接对象)对象，参数是数据库需要的

## 2.2 全局模块

下面的模块必须是全局的：

* apiLevel
    * 字符串常量，声明支持的DB API版本
    * 目前仅允许 `1.0`,`2.0`, 如果没给，默认假设是 `1.0`

* threadsafety
    * 整数，声明线程安全的级别，可能的值为：

        | threadsafety | 含义 |
        | --- | --- |
        | 0 | 线程不能共享这个模块 |
        | 1 | 线程能共享这个模块，但不能共享连接 |
        | 2 | 线程可以共享这个模块、连接 |
        | 3 | 线程可以共享这个模块、连接和游标 |
        
    * 注意线程安全和资源互斥    

* paramstyle 
    * 字符串常量：拼接参数的格式化类型
    
    | paramstyle | 含义 |
    | --- | --- |
    | qmark | 问号，例如：WHERE name=? |
    | numeric | 数字位置，例如：WHERE name=:1 |
    | named | 命名格式，例如：WHERE name=:name |
    | format | C语言printf格式，例如：WHERE name=%s |
    | pyformat | Python扩展的格式代码，例如：WHERE name=%(name)s |
    
    
## 2.3 异常

通过下面的异常及其子类，模块应该展示所有的错误信息：

* Warning
    * 重要的警告如插入数据时被截断，必须是Python `StandardError`的子类
* Error
    * 所有错误异常的基类，可以用一个except捕获所有错误。警告不是错误，所以不该继承Error。
     它必须是Python `StandardError`的子类
* InterfaceError
    * 和数据库接口相关的错误而不是数据库本身的错，必须是 Error的子类
* DatabaseError
    * 和数据库相关的错，必须是 `Error` 的子类
* DataError
    * 处理数据的过程中发生错误，比如除零，数字越界等。必须是 `DatabaseError`的子类
* OperationalError
    *  

# 3.连接对象

## 3.1 连接方法

# 4.游标对象

## 4.1 游标属性

## 4.2 游标方法

# 5.类型对象和构造器

# 6.模块作者的实现提示

# 7.可选的DB API扩展

# 8.可选的错误处理扩展

# 9.可选的2阶段提交扩展

## 9.1 TPC事务ID

## 9.2 TPC连接方法

# 10.FAQ

# 11.从1.0到2.0的主要变化

# 12.开放问题
 

