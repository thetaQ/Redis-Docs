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

注意：正如你所见，Lua数组是以Redis多条批量恢复的形式返回，这种返回类型在你的客户端库会转换成你用的编程语言中的数组形式。

Lua脚本可以使用两个不同的Lua函数去调用Redis命令：

- redis.call()
- redis.pcall()

`redis.call()`和`redis.pcall()`类似，唯一的区别就是，如果Redis命令调用产生错误，`redis.call()`会抛出一个Lua错误，然后强制[EVAL](https://redis.io/commands/eval)把错误返回给命令调用者；而`redis.pcall()`将会捕获错误，然后返回一个表示错误的Lua表结构。

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





