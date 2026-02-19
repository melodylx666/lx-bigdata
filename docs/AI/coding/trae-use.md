# AI Coding工具的使用姿势

上大学以来，我一直积极探索与使用`AI Coding`工具。在使用过程中发现其产品形态主要为以下几种：

* 传统官网的chatbot模模式，比如文心一言。
* 传统IDE插件，比如通义灵码，对原有编码方式的入侵程度较低。
* AI IDE，比如Trae IDE，对于编码方式来说有了新的选择。

AI Coding工具的产品形态总是对旧形态兼容，这意味着只需要本文只需要从最新的AI IDE出发介绍即可，这里以`Trae IDE`为例

## 基本模式

Trae是一款端到端的AI Coding工具。包含IDE和SOLO两种使用模式。

### IDE模式

在IDE模式下，Trae IDE提供了与传统IDE相似的编码环境。我们可以在Trae IDE中编写代码，同时利用Trae IDE的智能提示和自动补全功能，使用`ctrl + u`可以打开对话框，选择不同的Agent。

对话框中已经内置了基于`需求分析 -> 代码学习 -> 方案实施 -> 变更`的workflow所开发的Agent，从简单到复杂为：`Chat，Builder，Builder with MCP`。Chat适合简单对话，实现需求常用Builder，其可以调用命令行工具以及内置的MCP，实现较为复杂的需求。Trae在执行命令行命令的时候，会基于当前的操作系统，启动对应的沙箱环境，防止意外的命令执行效果对用户造成影响。 

### Solo模式

Solo模式是Trae提供的独立AI对话界面，无需项目上下文，专注于快速AI交互。支持三种工作模式：

- **直接对话模式**：即时问答。
- **Plan模式**：输入`/plan`启动，AI先制定执行计划，经用户确认后分步实施。适合模块级别开发。
- **Spec模式**：输入`/spec`启动，AI先输出详细技术规格文档(spec.md + tasks.md + checklist.md)。适合应用级别开发。


## 核心概念

### Context

通过`#`可以维护上下文，上下文可以是本地`File`或者远程`Web`资源。

### MCP & Tool

MCP全称为Model Context Protocol，即模型上下文协议。其核心组件分为如下部分：

* MCP Client: MCP Client为应用级别的，其可以对接多个MCP Server。其通常接收LLM生成的结构化指令，将其转换为符合MCP协议的指令后路由到MCP Server，最终将相应结果返回给LLM。
* MCP Server：一个MCP Server中维护一组Tools。每个Tool对应一个API/Command。MCP Server接收请求，并将请求转发给相应的Tool，并返回结果。
* MCP Protocol：MCP Protocol定义了MCP Server和MCP Client之间的通信协议。

通常MCP Client维护多个MCP Server的连接，这些连接之间可以为不同的通信模式。通信模式的设置通常为Server级别的，其中`stdio`使用于调用本地服务，而`SSE`(Client通过Get请求建立一次性连接，之后Server通过SSE多次推送实时结果数据)适用于调用远程服务，返回结果可以通过Pattern-Match进行对应操作。而`Streamable HTTP`可以复用连接，允许并发请求，并分chunk返回结果。

一个MCP Servers的配置例子如下：
```json
{
  "mcpServers": {
    "tools-group-1": {
      "command": "sql-gen",
      "args": [
        "execute-mode",
        "content"
      ]
    }
  }
}
```

### Skill

通常，我们写一个agent会有如下的Prompt：
```
## 角色


## 技能


## 限制
```

但是这样的Prompt有3个痛点：

* 无法复用。每次使用只能复制粘贴。
* 没有版本控制。无法知道变更历史。
* 逻辑组织差。一段文本中需要组织多种逻辑，比较难以维护和扩展。


而Skill通过引入文件系统，来完美解决上述痛点。一个`skill`通过一个核心skill.md + yaml格式 + 其他辅助文件来维护，最终需要将元数据统一注册，方便LLM调用。

* 针对无法复用的痛点，Skill可以维护项目级别或者全局来解决。
* 针对版本控制的痛点，引入filesystem + git自然可以解决。
* 针对逻辑组织差的痛点，一个Skill-name目录下可以包含skill.md,examples,templates等内容。

### Rule

Rule和Skill的思路完全一致。
















