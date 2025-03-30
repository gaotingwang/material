MCP（Model Context Protocol）协议，即[模型上下文协议](https://spec.modelcontextprotocol.io/specification/2024-11-05/)。该协议是一种开放协议，支持大模型应用程序与外部数据源和工具之间的无缝集成，提供了一种标准化的方式来连接 LLMs 需要的上下文。

![](https://gtw.oss-cn-shanghai.aliyuncs.com/AI/mcp.webp)



## 通信方式

传输层处理客户端和服务器之间的实际通信，MCP 支持多种传输机制，并且所有传输都使用 JSON-RPC 2.0 来交换消息。

- **标准输入/输出 （stdio）**

  stdio 传输支持通过标准输入和输出流进行通信。这对于本地集成和命令行工具特别有用。以下情况使用：

  - 构建命令行工具
  - 本地集成
  - 简单的通信
  - 使用 shell 脚本

- **SSE**

  SSE 传输支持使用 HTTP POST 请求进行服务器到客户端流式处理，以实现客户端到服务器的通信。以下情况使用：

  - 需要服务端到客户端的流式传输
  - 网络受限

- **自定义传输协议**

  MCP 可以根据特定需求自定义传输，只需要实现 Transport 接口即可。一般在自定义网络协议、性能优化等场景下自定义。

MCP 使用 [JSON-RPC](https://link.juejin.cn/?target=https%3A%2F%2Fwww.jsonrpc.org%2F) 2.0 作为其传输格式。传输层负责将 MCP 协议消息转换为 JSON-RPC 格式进行传输，并将收到的 JSON-RPC 消息转换回 MCP 协议消息。 

```json
{
  jsonrpc: "2.0",
  id: number | string,
  method: string,
  params?: object
}
```



## 消息类型

- 请求消息

  ```typescript
  interface Request { 
      method: string;    // 请求方式
      params?: { ... };  // 请求参数
  }
  ```

- 响应消息：对请求消息的成功响应

  ```typescript
  interface Result { 
      [key: string]: unknown;  // 响应内容
  }
  ```

- 错误消息：当请求 MCP Server 发生错误时，则返回错误消息

  ```typescript
  interface Error {
      code: number; //  错误码
      message: string; // 错误信息
      data?: unknown; // 返回数据
  }
  ```

- 通知消息：通知消息是不需要请求的单向消息，一般是由于服务端的工具、资源或者提示词有变更，需要通知到客户端的消息。

  ```typescript
  interface Notification {
    method: string;
    params?: { ... };
  }
  ```



## 核心能力

### 工具 Tools

工具是模型上下文协议 （MCP） 中一个强大的原语，它使服务器能够向客户端公开可执行功能。通过工具，LLMs可以与外部系统交互、执行计算并在现实世界中执行作。

> 工具设计为 model-controlled，这意味着工具从服务器公开给客户端，然后由 AI 模型决策调用哪个工具（在循环任务中需要授权批准）。

MCP 对于工具的定义规范如下；

```json
{
  name: string;          // 工具的唯一标识
  description?: string;  // 工具描述，清晰、易懂
  inputSchema: {         // 工具参数的 JSON Schema 描述
    type: "object",  // 参数类型
    properties: { ... }  // 工具的参数属性描述
  }
}
```

### 资源 Resources

资源是模型上下文协议 （MCP） 中的核心基元，它允许服务器公开可由客户端读取并用作LLM交互上下文的数据和内容。

> 资源被设计为 application-controlled。这就意味着客户端应用程序可以决定如何以及何时使用它们。不同的客户端对资源的处理采用不同的方式。

MCP Server 提供不同类型的数据给 MCP Client，每种资源都由唯一的 URI 标识，并且可以包含文本或者二进制数据。可能包括；文件/数据库/API 响应/实时系统数据/日志文件/.....

#### 1. 资源 URI

定义资源 URI 标准格式：

```
 [protocol]://[host]/[path]    协议://域名/路径
```

例如：`postgres://database/customers/schema` 协议和路径结构由 MCP 服务器实现定义，服务器可以自定义 URI 方案

#### 2. 资源类型

- 文本资源，包含 UTF-8 编码的文本数据，适应于：
  - 源代码
  - 配置文件
  - 日志文件
  - Json/xml 数据
  - 纯文本
- 二进制资源，包含 base64 编码的原始二进制数据，适应于：
  - 图像/图片
  - PDF 文件
  - 音视频文件
  - 其它非文本格式

#### 3. 资源发现

资源发现主要发生在 MCP Client，主要通过两种方式发现可用资源：

- 直接资源，服务器通过 resources/list 端点公开具体资源的列表，每个资源包含

  ```json
  {
    uri: string;           // 资源唯一标识
    name: string;          // 资源名称，必须人类可读，可理解
    description?: string;  // 资源描述（可选）
    mimeType?: string;     // 资源的 MIME 类型 （可选）
  }
  ```

- 资源模板，针对于动态资源，服务器可以公开 URI 模板，客户端可以根据模板构造有效的资源 URI

  ```json
  {
    uriTemplate: string;   // URI 模版，遵循 RFC 6570标准
    name: string;          // URI 模版名称 必须人类可读，可理解
    description?: string;  // 资源描述（可选）
    mimeType?: string;     // 资源的 MIME 类型 （可选）
  }
  ```

#### 4. 读取资源

MCP Client 通过使用资源 URI发出 **resources/read** 读取资源，服务器使用资源内容列表进行响应

```json
{
  contents: [
    {
      uri: string;        // 资源的 URI
      mimeType?: string;  // 资源 MIME 类型 （可选）

      // One of: 根据不同的资源二选一
      text?: string;      // 资源文本内容
      blob?: string;      // 资源的 Base64 编码的二进制文件
    }
  ]
}
```

#### 5. 资源更新

- 资源列表变更

  当 MCP Client 端的可用资源列表发生变更时，MCP Server 可以通过 /notifications/resources/list_changed 通知 MCP Client。

- 资源内容变更

  MCP Client 可以订阅特定资源的更新：

  - 客户端使用资源 URI 发送 resources/subscribe 订阅。
  - 服务器在资源变更时，发送 notifications/resources/updated。
  - 客户端使用 resources/read 获取最新的内容。
  - 客户端使用 resources/unsubscribe 取消订阅。

### 提示词 Prompts

创建可重用的提示词模板和工作流（服务器可预定一些提示词模板、将更为复杂的多步任务封装为一个工作流程的 Prompt）。这样提供提示词模板更加方便用户使用，与大模型交互返回的结果更加稳定和标准。

> 提示词设计为 user-controlled，这意味着提示词由服务器向客户端公开，由用户自由选择使用。

#### 1. 提示词结构

在 MCP Server 定义提示词模板时，必须遵循如下的标准结构：

```json
{
  name: string;              // 提示词的唯一标识
  description?: string;      // 提示词的描述，必须清晰、可理解
  arguments?: [              // 参数 （可选）
    {
      name: string;          // 参数名称
      description?: string;  // 参数的描述，也必须清晰、可理解
      required?: boolean;    // 参数是否必填
    }
  ]
}
```

#### 2. 获取提示词

客户端可以通过 prompts/list 端点获取可用的提示词模板：

```json
// 请求结构
{
  method: "prompts/list"
}

// 返回结构
{
  prompts: [
    {
      name: "analyze-code",
      description: "Analyze code for potential improvements",
      arguments: [
        {
          name: "language",
          description: "Programming language",
          required: true
        }
      ]
    }
  ]
}
```

#### 3. 使用提示词

客户端可以通过 prompts/get 请求获取提示词模板。请求以及响应如下；

```json
// 请求
{
  method: "prompts/get", // 请求url
  params: {
    name: "analyze-code", // 提示词模板名称
    arguments: {          // 参数
      language: "python"
    }
  }
}

// 返回提示词模板
{
  description: "Analyze Python code for potential improvements", // 描述，MCP Server定义
  // 消息
  messages: [  
    {
      role: "user", // 消息类型
      // 内容
      content: { 
        type: "text", // 内容类型
        text: "Please analyze the following Python code for potential improvements:\n\n```python\ndef calculate_sum(numbers):\n    total = 0\n    for num in numbers:\n        total = total + num\n    return total\n\nresult = calculate_sum([1, 2, 3, 4, 5])\nprint(result)\n```"
      }
    }
  ]
}
```

#### 4. 动态提示词

提示词不仅是静态的，而且也可以设置为动态的，比如可以嵌入用户的一些上下文信息（嵌入资源上下文、多步骤工作流）。动态提示词模板可以动态填入变量，比如代码片段、错误信息，生成针对性的输出。在实际的业务场景下使用也是非常多的。



## 根 Roots

Roots 是 MCP 中的一个概念，它定义了服务器在文件系统中可以操作的边界，使它们能够了解它们可以访问哪些目录和文件。



