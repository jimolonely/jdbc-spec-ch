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
    * 和数据库操作相关的错误，不需要在程序员的控制中。比如，未预知的连接断开，数据源名称找不到，无法处理事务，在处理时内存分配错误等。
        必须是 DatabaseError的子类
* IntegrityError
    * 完整性错误。比如外键检测失败。必须是 DatabaseError的子类
* InternalError
    * 数据库内部错误。比如 游标不再合法。必须是DatabaseError
* ProgrammingError
    * 程序错误。例如，表找不到或已经存在，语法错误，参数个数错误。必须是DatabaseError的子类
* NotSupportedError
    * 使用了数据库不支持的API。比如在不支持事务的数据库调用 `a.rollback()`. 必须是 DatabaseError的子类

下面是异常继承结构：

```shell script
StandardError
|__Warning
|__Error
   |__InterfaceError
   |__DatabaseError
      |__DataError
      |__OperationalError
      |__IntegrityError
      |__InternalError
      |__ProgrammingError
      |__NotSupportedError
```

> 注意：这些异常的值都没定义，他们应该给用户一个清洗的描述到底哪错了

# 3.连接对象

连接对象应该遵循下面的方法

## 3.1 连接方法

* close()
    * 现在就关闭连接（而不是无论何时调用 `.__del__()`）
    * 关闭之后连接应该不可用，否则应该抛出 [Error](##2.3异常)（或子类） 异常。如果在提交修改之前关闭连接需要回滚。
* commit()
    * 提交任何挂起的事务到数据库
    * 注意是否数据库支持自动提交，如果是，那么默认应该关闭。同事应该提供一个设置是否自动提交标志的方法
    * 不支持事务的数据库应该实现一个空方法
* rollback()
    * 该方法可选，因为不是所有数据库都支持事务
    * 回滚就是回到事务提交的开始状态    
* cursor()
    * 返回一个[Cursor](#4.游标对象)对象
    * 如果数据库不支持游标对象，那么实现需要模拟游标，通常需要采用其他方法还是扩展这个规范

# 4.游标对象

游标用来管理获取数据的上下文。同一个Connection创建的游标不是相互隔离的。
例如，任何通过游标对数据库的修改都立即对其他游标可见。不同connection创建的游标可能隔离，取决于事务是如何实现的。

游标需要支持以下方法和属性。

## 4.1 游标属性

* .description
    * 可读属性：由7个元素组成的序列
    * 每个序列包含以下信息来描述结果列
        * name
        * type_code
        * display_size
        * internal_size
        * precision
        * scale
        * null_ok
    * 开头2个是强制的，后面5个是可选的，没有就设成None
    * 如果没有返回结果rows或者游标还没有调用 [execute*()]()方法，那么这个属性为None
    * `type_code`对应下面的[类型对象](#5.类型对象和构造器)
* .rowcount
    * 可读属性，表示执行返回的行数，由 `execute*()`产生（对于DQL语句如SELECT）或者修改
    （DML语句如UPDATE，INSERT）
    * 在还没执行或者最后一个操作不能确定结果时，属性值为-1
    > 注意：未来版本的API可能重新定义为返回None而不是`-1`
                                                                                                                                         >
                                                                                                                                         
## 4.2 游标方法

* `.callproc(procname[,parameters])`
    * 可选，因为不是所有数据库都提供存储过程
    * 通过名称调用存储过程。参数必须和存储过程对应的映射。结果返回输入序列的一个副本。
    * 如果返回结果集，必须遵循标准的 `.featch()` 方法
* `.close()`
    * 立即关闭游标（而不是任何时候调用 `__del__`）
    * 关闭后不再可用，否则抛出Error
* `.execute(operation[,parameters])`
    * 准备和执行数据库操作（查询或命令）
    * 参数和操作的映射应该是对应的，参考上面[paramstyle](##2.2全局模块)
    * 这个操作的引用被游标保留，如果同样的操作多次使用，那么可以重复使用
    * 为了最大化的重用操作，最好使用 `.setinputsizes()` 来事先指定参数类型和大小。
    * 参数同样可以是tuple列表。例如，插入很多行数据，但是这种用户被淘汰了，使用 `.executemany()`方法
    * 返回值未定义

* `.executemany(operation[,seq_of_parameters])`
    * 准备一个数据库操作（查询或命令），然后根据一系列参数执行
    * 可以自由的通过调用多次 `.execute()` 来实现，或者使用数组操作来让数据库一次性处理
    * 这个方法的结果集可能会产生未知的行为，实现允许（但不强求）抛出异常。
    * `.execute()`方法的定义同样适用于该方法
    * 返回值未定义
    
* `.fetchone()`
    * 获取查询结果集的下一行数据，返回一个序列或者没有数据时返回None
    * 当 `.execute*()`方法没有返回结果时抛出异常

*  `.fetchmany([size=cursor.arraysize])`
    * 返回结果集的下一批数据，返回序列的列表（例如：tuple list）。当没有数据时返回空序列
    * 参数size可以指定返回的条数，如果没有给定，就是游标的数组长度。
    * 当 `.execute*()`方法没有返回结果时抛出异常
    * 注意出于性能考虑，最好是返回游标的数组长度这么多数据

* `.fetchall()`
    * 获取所有（剩余的）行数结果集。
    * 当 `.execute*()`方法没有返回结果时抛出异常

* `.nextset`
    * (该方法可选，因为不是所有数据库都支持多结果集)
    * 该方法将使游标移动到下一可用的结果集，抛弃现在剩下的结果行
    * 如果没有更多的集合，返回None。否则返回 true，然后可以调用 `.fetch*()`方法获取结果集
    * 当 `.execute*()`方法没有返回结果时抛出Error异常

* `.arraysize`
    * 读写属性，指定 `.fetchmany()`一次性获取的数据行数。默认是1，表示一次获取一行
    * 这个参数同样可用于 `.executemany()`

* `.setinputsizes(sizes)`
    * 可以用来在执行 `.execute*()` 方法前预定义内存区域
    * `sizes`是一个序列，每个元素对应一个输入参数，应该是一个 类型对象，或者一个整数代表
        字符串参数的最大长度。如果 元素为None，那么没有预定义的内存区域为这个列保留
    * 这个方法应该在 `.execute*()` 之前调用
    * 实现可以自由的实现为什么都不做，让用户自由的使用 

* `.setoutputsize(size[,column])`
    * 设置列的缓存区大小，比如对于大的列（LONG，BLOB等）。column可以指定为结果集的索引。
    * 这个方法应该在 `.execute*()` 之前调用
    * 实现可以自由的实现为什么都不做，让用户自由的使用 
    
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
 
