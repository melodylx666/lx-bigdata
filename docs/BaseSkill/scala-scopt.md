# Scopt库

## 快速入门

### 定义

scopt是一个scala 解析与处理命令行参数的工具。它提供一个简灵活的接口来解析和定义命令行选项。其使用DSL来定义命令行选项，重点是学会用法而不是底层原理。

### 用法

第一步，定义配置类

```java
case class Config(
    input: String = "",
    output: String = "",
    verbose: Boolean = false,
    count: Int = 0
)

```

配置类用于存储解析命令行参数之后的结果，包含默认值的字段是可选的。

第二步，定义解析器(返回值就是parser)

```java
val builder = OParser.builder[Config]
val parser = {
  import builder._
  OParser.sequence(
    programName("my-program"),
    head("my-program", "1.0"),

    opt[String]('i', "input")
      .valueName("<file>")
      .action((x, c) => c.copy(input = x))
      .text("Input file path"),

    opt[String]('o', "output")
      .valueName("<file>")
      .action((x, c) => c.copy(output = x))
      .text("Output file path"),

    opt[Unit]('v', "verbose")
      .action((_, c) => c.copy(verbose = true))
      .text("Enable verbose mode"),

    opt[Int]("count")
      .valueName("<number>")
      .action((x, c) => c.copy(count = x))
      .text("An integer value")
  )
}

```

其中

- `opt[String](i,"input")`定义一个字符串类型的可选参数(短选项或者长选项)：-i，或者是--input
- `valueName("<file>")`是显示在帮助信息的参数占位符
- `action((x,c) => c.copy(input = x))` 当参数x出现的时候，配置c实例应该如何更新
- `text("Input file path")`帮助信息中对该参数的描述
- 如果此参数是必须的，则需要加上`.required()`
- 可以通过如下方式进行校验：`.validate(x => if (x.nonEmpty) success else failure("Value must not be empty"))`


第三步，传入解析器，命令行参数，空配置实例，得到解析完毕的实际配置实例

> 这里`Oparser.parse(parser,args,config)`，或者是`parser.parse(args,config)`均可

```java
    OParser.parse(parser, args, Config()) match {
      case Some(config) =>
        if (config.verbose) {
          println(s"Input: ${config.input}")
          println(s"Output: ${config.output}")
          println(s"Count: ${config.count}")
        }
      case _ =>
        // 错误发生或 --help 被调用
    }
```
