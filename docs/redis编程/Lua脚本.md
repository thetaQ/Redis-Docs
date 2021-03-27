# EVAL script numkeys key [key ...] arg [arg ...]

> **2.6.0版本开始支持**
>
> **时间复杂度**：取决于执行的脚本

### EVAL简介

从2.6.0版本开始，[EVAL](https://redis.io/commands/eval)和[EVALSHA](https://redis.io/commands/evalsha)命令用于使用Redis内置的Lua解释器来执行脚本。

[EVAL](https://redis.io/commands/eval)命令的第一个参数是Lua 5.1脚本。脚本不需要（也不应该）去定义Lua函数，它仅仅是一个在Redis服务器上下文中运行的Lua程序。

[EVAL](https://redis.io/commands/eval)命令的第二个参数是脚本后面跟着的Redis键参数的数量（从三个参数开始）。Lua脚本可以使用全局变量`KEYS`去获取这些参数（例如`KEYS[1]`，`KEYS[2]`······，下标从1开始）。

其余的参数不应该代表键名称，Lua脚本可以使用`ARGV`全局变量获取，类似于`KYES`（例如`ARGV[1]`，`ARGV[2]`······，下标从1开始）。

下面的例子应该清楚的阐释了上述内容：

```shell
> eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
1) "key1"
2) "key2"
3) "first"
4) "second"
```

注意：正如你所见，Lua数组是以Redis多块响应(multi bulk replies)的形式返回，这种返回类型在你的客户端库会转换成你用的编程语言中的数组形式。

Lua脚本可以使用两个不同的Lua函数去调用Redis命令：

- redis.call()
- redis.pcall()

`redis.call()`和`redis.pcall()`类似，唯一的区别就是，如果Redis命令调用产生错误，`redis.call()`会抛出一个Lua错误，然后强制[EVAL](https://redis.io/commands/eval)把错误返回给命令调用者；而`redis.pcall()`将会捕获错误，然后返回一个表示错误的Lua table结构。

`redis.call()`和`redis.pcall()`函数的参数就是Redis命令的所有的参数：

```shell
> eval "return redis.call('set','foo','bar')" 0
OK
```

上述的脚本将键`foo`的值设置成字符串`bar`。但是这样使用违反了[EVAL](https://redis.io/commands/eval)命令的语义，因为所有脚本中使用的键都应该使用`KEYS`数组传递：

```shell
> eval "return redis.call('set',KEYS[1],'bar')" 1 foo
OK
```

在执行之前必须分析所有的Redis命令，以确定该命令将会在哪些键上进行操作。为了正确使用[EVAL](https://redis.io/commands/eval)，所有的键都要显示的传递。这在很多场景是很有帮助的，尤其是确保Redis集群可以把请求转发到正确的节点上。

注意，这个规则没有强制的原因，是为了给用户提供使用Redis单机实例配置的机会，但这要以编写出不兼容Redis集群模式的脚本为代价。

通过一些列转换规则，Lua脚本返回的值会从Lua类型转换成Redis协议的值。

### Lua和Redis之间数据类型转换

当Lua脚本使用`call()`或者`pcall()`调用Redis命令时，Redis的返回值将会被转换成Lua数据类型。类似的，当调用Redis命令并且Lua脚本返回值时，Lua数据类型将会转换成Redis协议，以便脚本可以控制[EVAL](https://redis.io/commands/eval)命令返回给客户端的值。

这种数据类型之间的转换被设计成这种形式：如果Redis类型转换成Lua数据类型，然后结果又被转换为Redis类型，那么结果将会和初始值相同。

换句话说，Lua和Redis类型的转换是一对一的。下表展示了所有的转换规则：

**Redis到Lua**的转换表：

- Redis 整型响应 -> Lua number
- Redis 块响应(bulk reply) -> Lua字符串
- Redis 多块响应(multi bulk reply ) -> Lua table (可能嵌套了其他Redis数据类型)
- Redis 状态响应 -> Lua table，包含`ok`字段
- Redis 错误响应 -> Lua table，包含`err`字段
- Redis Nil块响应或Nil多块响应 -> Lua布尔类型false

**Lua到Redis**的转换表：

- Lua number -> Redis 整型响应 (the number is converted into an integer)
- Lua字符串-> Redis 块响应(bulk reply)
- Lua table（array） -> Redis 多块响应(multi bulk reply )  (截止至Lua数组的第一个nil)
- 仅包含`ok`字段的Lua table -> Redis 状态响应
- 仅包含`err`字段的Lua table -> Redis 错误响应
- Lua布尔类型false -> Redis Nil块响应

还有一个额外的Lua到Redis的转换规则，但是没有对应的Redis到Lua转换的规则：

- Lua布尔类型true -> Redis整型响应，值为1

最后还需要注意三个规则：

- Lua只有一个数字类型Lua number。整型和浮点类型之间没有区别。所以从Lua number转换成Redis整型响应时，只会转换成整数，小数部分会去掉。**如果你想从Lua脚本返回浮点数，你应该使用字符串类型**，Redis自身也是这么做的（可以参考[ZSCORE](https://redis.io/commands/zscore)命令）
-  [Lua数组中是没有nil值的](http://www.lua.org/pil/19.1.html)（译者注：Lua数组是以nil结尾，类似于C语言中字符串是以\0结尾，字符串中不能有\0这个字符），这是Lua table语法决定的。所以当从Lua数组转换成Redis协议数据时，当发现nil时转换终止。
- 当Lua table包含一些键值对，转换成Redis响应时并不会包含这些字段。

**RESP3 模式的转换规则**：注意，Lua引擎可以在使用新Redis 6协议的RESP3模式下工作。因此还有一些额外的转换规则，并且与RESP2模式相比，有些规则稍有变动。详情请参考文档的RESP3章节。

下面是一些转换的例子：

```shell
> eval "return 10" 0
(integer) 10

> eval "return {1,2,{3,'Hello World!'}}" 0
1) (integer) 1
2) (integer) 2
3) 1) (integer) 3
   2) "Hello World!"

> eval "return redis.call('get','foo')" 0
"bar"
```

最后一个例子展示了，如果是Lua直接调用命令`redis.call()`或`redis.pcall()`，是如何获取到返回值的。

下面的例子我们可以看到浮点型，以及包含nil值和键的数组是如何处理的：

```shell
> eval "return {1,2,3.3333,somekey='somevalue','foo',nil,'bar'}" 0
1) (integer) 1
2) (integer) 2
3) (integer) 3
4) "foo"
```

我们可以看到3.333被转换成了3，*somekey*被排除掉，*bar*字符串由于前面有nil值，所以永远也不会被返回。

### 返回Redis类型的辅助函数

有两个辅助函数可以从Lua中返回Redis类型。

- `redis.error_reply(error_string)`返回一个错误响应。这个函数返回一个只包含`err`字段的Lua table。字段的值被设置指定的字符串。
- `redis.status_reply(status_string)`返回一个状态响应。这个函数返回一个只包含`ok`字段的Lua table。字段的值被设置指定的字符串。

使用辅助函数和直接返回特定格式的table是没有任何区别的。所以下面两种形式结果是相同的：

```
return {err="My Error"}
return redis.error_reply("My Error")
```

### 脚本的原子性

Redis所有的命令都使用同一个Lua解释器。Redis也保证脚本被原子的执行：当一个脚本执行时，不会执行其他任何脚本或者Redis命令。这个语法和[MULTI](https://redis.io/commands/multi)/[EXEC](https://redis.io/commands/exec)包围的事务类似。在其他的客户端看来，脚本的执行结果要么是不可见的（not visible)，要么是已经完成的（already completed）。

然而这也意味着，执行一个慢脚本并不明智。写一个运行很快的脚本并不难，因为脚本运行的开销是很小的。但是如果你准备执行一个慢脚本，那么你要注意当脚本运行时，所有的客户端都不能执行命令。

### 错误处理

前面介绍过，`redis.call()` 和`redis.pcall()`的唯一区别在于它们对错误处理的不同。当`redis.call()`在执行命令过程中发生错误时，脚本会停止执行，并返回一个脚本错误，错误的输出信息会说明产生错误的原因：

```shell
> del foo
(integer) 1
> lpush foo a
(integer) 1
> eval "return redis.call('get','foo')" 0
(error) ERR Error running script (call to f_6b1bf486c81ceb7edf3c093f4c48582e38c0e791): ERR Operation against a key holding the wrong kind of value
```

使用`redis.pcall()`，将不会引发错误，而是使用特定的格式（仅包含err字段的Lua table）返回一个错误对象。脚本可以把`redis.pcall()`产生的错误对象返回给用户。

```shell
> eval "return redis.pcall('get','foo')" 0
(error) WRONGTYPE Operation against a key holding the wrong kind of value
```

### 带宽和EVALSHA

[EVAL](https://redis.io/commands/eval)命令需要你每次都要传输脚本内容。Redis虽然有内部的缓存机制，不需要每次都重新编译脚本，但是在很多场合，付出无谓的贷带宽去传输脚本内容并不是最佳选择。

**[TODO]**

另一方面，使用特殊的命令或者通过redis.conf配置也会有一些问题：

- 不同的实例可能使用不同的命令实现
- 确保所有的实例都有指定的命令，这样部署起来很困难，尤其是分布式环境下。

为了减少带宽的消耗，同时避免上述问题，Redis实现了[EVALSHA](https://redis.io/commands/evalsha)命令。

[EVALSHA](https://redis.io/commands/evalsha)类似于[EVAL](https://redis.io/commands/eval)，但是第一个参数不是脚本主体，而是脚本的SHA1摘要。命令的表现为：

- 如果服务器仍然保存一个匹配该SHA1摘要的脚本，那么这个脚本就会被执行。
- 如果服务器没有匹配该SHA1摘要的脚本，那么将返回一个错误给客户端，提醒其其使用[EVAL](https://redis.io/commands/eval)。

示例：

```shell
> set foo bar
OK
> eval "return redis.call('get','foo')" 0
"bar"
> evalsha 6b1bf486c81ceb7edf3c093f4c48582e38c0e791 0
"bar"
> evalsha ffffffffffffffffffffffffffffffffffffffff 0
(error) `NOSCRIPT` No matching script. Please use [EVAL](/commands/eval).
```

客户端库实现[EVAL](https://redis.io/commands/eval)命令时底层可以实际发送[EVALSHA](https://redis.io/commands/evalsha)去优化，我们期望服务器已经有了该脚本的缓存。如果返回了`NOSCRIPT`错误，再使用 [EVAL](https://redis.io/commands/eval)重新发送脚本。

执行[EVAL](https://redis.io/commands/eval)命令时，前面提到的使用参数的方式传递键名和附加参数，在这里也起到了重要作用：可以让脚本内容固定不变，以便于Redis缓存脚本内容。

### 脚本缓存

Redis保证所有被运行过的脚本都会被永久保存在缓存中。这就意味着，如果[EVAL](https://redis.io/commands/eval)命令在一个Redis实例上成功执行某个脚本后，后续针对这个脚本的所有的[EVALSHA](https://redis.io/commands/evalsha)命令都会被成功执行。

脚本可以被长时间缓存的原因是，一个优秀的应用程序的编写，不应该会有这么多不同的脚本，导致内存问题。每一个脚本从概念上讲都是一个新的命令，即使是一个很大的应用，一般最只有几百个脚本。即使应用程序被修改很多次，很多脚本变化了，内存的占用也是微不足道的。

刷新脚本缓存的唯一方法是使用[SCRIPT FLUSH](https://redis.io/commands/script-flush)命令，它会清空之前运行过的所有脚本的缓存。

通常只有在云环境下，该Redis实例重新分配给其他的客户或者应用程序使用时，才会执行该命令。

另外，之前也提到过，脚本缓存不是持久化的，在重启Redis实例时也会被刷新。然而从客户端的角度来看，只有两个方法去判断Redis实例没有被重启：

- 与服务器的连接仍然持续，没有被关闭。
- 客户端明确检查[INFO](https://redis.io/commands/info)命令的`runid`字段，确保服务器没有重启，进程没有改变。

实际上来说，对客户端而言最简单的办法就是假设，在给定连接的上下文中脚本缓存保证存在，除非系统管理员明确调用[SCRIPT FLUSH](https://redis.io/commands/script-flush)命令。

事实上，用户会发现在pipelining的场景下，Redis不清除脚本缓存是一个很好的主意。例如，对于一个和Redis保持持久链接的应用程序来说，它可以确信，执行过一次的脚本会一直保存在内存中。因此它可以在流水线中使用[EVALSHA](https://redis.io/commands/evalsha)命令而不用担心找不到脚本的错误（后面会详细介绍这个问题）。

通常的模式是，调用[SCRIPT LOAD](https://redis.io/commands/script-load)去加载流水线中将要用到的所有脚本，然后直接在流水线中使用[EVALSHA](https://redis.io/commands/evalsha)，而不需要去检查脚本哈希值不存在的错误。

## SCRIPT 命令

Redis提供了`SCRIPT `命令，用来对脚本子系统进行控制。当前支持三个命令：

- [SCRIPT FLUSH](https://redis.io/commands/script-flush)

  该命令是清空Redis脚本缓存的唯一方法。在云环境下，将同一个实例分配给其他不同用户时很有用。在测试客户端库脚本特性的实现时也很有帮助。

- SCRIPT EXISTS sha1 sha2 ... shaN

  给定一些列SHA1摘要作为参数，该命令返回一个包含0和1值的数组。1表示指定的SHA1摘要在缓存中存在，0则表示该脚本从未被执行过（或者最后一次SCRIPT FLUSH之后未执行过）。

- SCRIPT LOAD script

  该命令将指定的脚本加载到Redis脚本缓存中。如果我们不想先通过实际运行脚本去确保[EVALSHA](https://redis.io/commands/evalsha)不失败（比如pipline或者`MULTI/EXEC `事务操作），这个命令很有效。

- [SCRIPT KILL](https://redis.io/commands/script-kill)

  该命令是终止一个已经运行了超过最大运行时间的脚本的唯一办法。`SCRIPT KILL`只能应用于运行期间不会改变数据库内容的脚本（因为终止一个只读的脚本，并不违反脚本引擎原子性的保证）。更多关于长时间运行的脚本的内容，可以参考下一节。

### 纯函数脚本

**[TODO]**

*注意：从Redis 5开始，脚本是使用复制而不是逐字发送脚本。因此接下来的内容适合Redis 4及以前的版本*

编写脚本一个非常重要的部分就是编写纯函数脚本。默认情况下，Redis实例执行的脚本都是通过发送脚本本身（而不是生成的命令）传播到其他Redis实例或者AOF文件中。

主要原因是将脚本发送到另一个Redis实例通常比发送脚本生成的多个命令要快得多。因此如果客户端发送多个脚本到主节点，然后转换成多个独立的命令传给复制节点/AOF文件

