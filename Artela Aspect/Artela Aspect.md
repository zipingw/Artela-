# Artela Aspect

Aspect引入了一种在区块链平台上实现系统级功能的动态机制。本质上，Aspect类似于一个系统扩展，它以WebAssembly（WASM）形式执行，可以绑定到任何智能合约上。这种能力提供了灵活性，可以增强目标合约的功能或监控其活动，在满足预定条件时触发特定操作。

## Aspect是什么？

Aspect的强大之处源于WebAssembly（WASM）。在Artela上，我们开发了一个专门的WASM运行时环境，名为Aspect-Runtime，专门用于在平台上执行Aspects。Aspect Runtime通过一系列预定义的主机API，实现了Aspect与区块链核心模块之间的交互。这些API赋予了Aspect多种能力，例如读取区块链状态、调用智能合约、管理自身状态等等。

对于有Web开发背景的人来说，"中间件"这个概念可能比较熟悉。对于新手，让我们进一步探讨这个概念。

### Web框架中的Interceptor

"Interceptor"这个概念的中文翻译如下：

Interceptor（拦截器）是可以集成到网络服务器中以扩展其功能的代码。想象一下，使用拦截器为网络服务器添加身份验证或日志记录功能，如下所示：

![](.\Interceptor.svg)

当收到HTTP请求时，网络服务器的拦截器（在某些框架中可能称为中间件）机制允许开发人员创建模块，在请求的主要处理之前或之后对其进行处理。例如，身份验证拦截器可能会在请求到达核心逻辑之前验证用户的凭据。这种设计模式使身份验证和日志记录等功能去中心化，确保了更模块化的方法。

这种拦截器设置还支持共享上下文，实现了不同中间件或路由处理程序之间的通信。例如，在验证用户身份后，相关拦截器可以在共享上下文中存储用户数据。随后的拦截器或路由处理程序就可以访问这些数据，而无需重新加载。

### Aspect in Artela

类比来说，Aspect可以被视为智能合约的拦截器。想象一下在Artela上开发一个去中心化交易所并集成Aspects：

![](.\aspect.svg)

就像拦截器增强Web框架一样，Aspects也可以增强您的去中心化应用（dApps）。它们实现了模块化，并通过共享上下文促进模块间的通信。

## Aspect是如何工作的?

### 扩展层

在Artela中，我们提供了一个内置的扩展层，它位于应用层（由智能合约组成）和基础层（区块链的核心模块，例如共识、内存池、P2P网络）之间。这个扩展层允许开发者开发链级扩展，为特定的智能合约定制交易处理工作流程。

![](.\Extension layer.svg)

### Aspect Core

Aspect Core是一个系统模块，用于管理Aspects的生命周期，包括部署、执行、绑定、升级和销毁。它同时也作为一个系统合约暴露在`0x0000000000000000000000000000000000A27E14`地址，可以通过EVM合约调用来调用它。Aspect Core的实现是用原生代码编写的，以减少虚拟机启动的开销。

![](.\Aspect Core.svg)

### Aspect Lifecycle

Aspect的生命周期由Aspect Core管理。它包含以下阶段：

- **编译**：Aspect被Aspect编译器编译成WASM字节码。
- **部署**：编译后的WASM字节码可以部署到区块链上，并用特定属性进行初始化。
- **绑定**：为了在连接点被触发，Aspect必须与智能合约绑定。这个绑定过程是通过调用Aspect Core的`bind`方法完成的。
- **执行**：Aspect的执行可以在预定义的连接点触发，也可以通过直接调用Aspect的`operation`接口触发。

关于Aspect生命周期的更多详细信息，请参考[相关文档](https://docs.artela.network/develop/core-concepts/lifecycle)。

## Aspect能够做什么？

### 插入"Just-in-Time"调用

即时调用（Just-in-Time，JIT）是一种可以在合约调用级别连接点处插入到当前EVM调用栈的调用类型。有关JIT调用概念的详细说明，请参考[相关文档](https://docs.artela.network/develop/core-concepts/jit-call)。

### 与其他Aspects和智能合约共享信息

尽管Aspect和智能合约的执行环境是不同的（WASM vs. EVM），但它们并非完全隔离。信息可以通过**Aspect上下文**在Aspects和智能合约之间共享。Aspect上下文本质上是一个临时存储，其生命周期仅限于当前交易。Aspect上下文可用于实现Aspect和智能合约之间的双向通信。

关于这个主题的更多详细信息，请阅读[相关文档](https://docs.artela.network/develop/core-concepts/communication#aspect-context)。

### 从EVM智能合约查询

Aspect可以使用最新状态对EVM智能合约进行只读调用（目前不允许查询历史状态）。在某些情况下，比如您正在构建价格反馈预言机Aspect时，您可以利用这个功能从预言机合约查询最新价格，并将其保存到Aspect上下文中，然后与智能合约共享。

关于这个主题的更多详细信息，请阅读[相关文档](https://docs.artela.network/develop/core-concepts/communication#evm-static-caller)。

### 状态变更追踪

Aspects可以追踪智能合约状态的变化。例如，Aspect可以根据输入参数检查智能合约的提交状态是否符合预期。

这种追踪是通过ASOLC编译器生成的额外操作码和IR方法来实现的：

![](.\state-tracing.svg)

关于这个主题的更多信息，请参考这里的ASOLC[文档](https://docs.artela.network/develop/core-concepts/asolc)。

### 像智能合约一样维护状态

与智能合约类似，Aspect也是有状态的。每个Aspect都维护着自己的状态，这些状态会被持久化到世界状态中。目前Aspect只提供简单的键值存储，更好的存储绑定功能将在后续版本中提供。要将数据持久化到区块链世界状态中，您可以使用以下方式：

```javascript
// read & write state
let state = sys.aspect.mutableState.get<string>('state-key');
let unwrapped = state.unwrap();
state.set('state-value');
```

### 处理调用

Aspect也可以独立运行。它可以通过其字节输入-字节输出的`operation`方法处理外部交易或调用：

```javascript
class AspectTest implements IAspectOperation {
  operation(input: OperationInput): Uint8Array {
    // handle incoming request and echo it back
    return data;
  }
}
```

> Note
> 请注意，operation方法仅是一个字节输入-字节输出的入口点，如果Aspect中有任何敏感数据，请确保您正确进行身份验证。

## 了解更多

Artela Aspect的初始版本是使用[AssemblyScript](https://www.assemblyscript.org/)（TypeScript的一个子集，具有严格的类型系统）构建的。为了让开发Aspects变得更容易，我们还创建了Aspect工具集（[Aspect Tooling](https://github.com/artela-network/aspect-tooling)），这是一个库和工具集，供开发者与我们的底层主机API进行交互。更多详细信息，您可以查看我们的代码仓库。
