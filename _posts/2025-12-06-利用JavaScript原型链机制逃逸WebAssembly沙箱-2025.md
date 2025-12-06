---
layout:     post                       
title:    利用JavaScript原型链机制逃逸WebAssembly沙箱
subtitle:   
date:       2025-12-06 
author:     Closure                         
header-img: img/fmqwq.jpg
catalog: true                         
tags:                               
    - 安全
    - 笔记
---

https://webassembly.github.io/spec/js-api/#read-the-imports
https://wasi.dev/
https://phrack.org/issues/72/10_md#article

WebAssembly被认为是在Web&Node.js上运行原生代码的安全方式的假设是WASM虚拟机是一个封闭的黑盒，它的线性内存与宿主环境的内存是完全隔离的，唯一能与外部世界交互的渠道，是宿主在实例化时显式传入的importObject。

如果宿主执行`WebAssembly.instantiate(code, {})`，理论上这个WASM模块就变成了纯计算单元，无法发起网络请求、无法读取Cookie、也无法执行eval()。

但是WASM规范在定义模块链接阶段的行为时，引入了一个攻击面：WASM运行时与JS对象模型是耦合的。

WASM是一种静态类型的二进制格式，JS是动态的，当WASM引擎试图解析一个Import时，宿主环境并不是只检查importObject是否拥有env这个Own Property，它调用的是JS标准的Get操作，查找过程遵循标准的原型链遍历。

即使importObject是一个空对象 {}，WASM仍然看见了Object.prototype上的所有内容，以上就构成了WASM沙箱逃逸：如果攻击者能够利用JS侧的Object.prototype 上存在危险的Gadget，就能越过空的importObject来直接引用并执行这些特权函数。

### 突破口

突破口是原型链劫持，虽然WASM定义了怎么解析importObject，但是实际上它依赖了宿主环境的标准属性查找机制。

当WASM模块请求导时，JS引擎会执行类 `importObject["mod"]["func"] `的查找，即便开发者传入一个空对象`const importObject = {}`，这个对象仍然继承自Object.prototype，所以importObject拥有toString、constructor、valueOf这些属性。

攻击路径是在WASM模块中定义特殊的导入名称，诱导JS引擎沿着原型链向上查找来获取全局构造器，来导入importObject.toString.constructor，这实际上就是 JavaScript 的全局 Function 构造器。

Function 构造器类似于 eval()，它允许通过字符串创建并执行新的函数。

难题有两个，构造任意字符串和字符生成

#### 构造任意字符串

拿到Function构造器后需要传入字符串代码来生成恶意函数，但是在WASM中只有数字类型，所以没法直接创建JS字符串对象，所以需要在JS侧拼凑出字符串。

作者给出的方法是利用String.fromCharCode将WASM中的数字转换为字符，然后再将字符拼接。

- 获取String： 通过访问某个函数的length属性的构造器，间接拿到Number构造器，再通过相关属性跳转最终拿到String构造器及其静态方法fromCharCode和raw。
- 字符生成：Object.groupBy调用String.fromCharCode。
- 输入：WASM 提供的 ASCII 码数字。
- 输出：单字符字符串对象。

生成的字符被暂时存储在Object.groupBy返回对象的Keys中, 使用Object.keys() 将这些散落的字符提取出来形成一个数组。

利用String.raw方法将上述提取出的字符数组缝合成一个完整的Payload 。

通过反复调用这些原生Gadgets，作者在JS堆内存中凭空制造出了恶意代码字符串来绕过WASM无法直接创建字符串的限制。

#### 执行链

即使我们成功用Function构造器创建了一个恶意函数，也没有办法直接调用这个f。因WASM的call_indirect指令需要通过Table来调用函数，而动态生成的JS函数并不WASM的Table中，所以无法直接说“执行在这个变量里的JS函数”。

作者再次利用Object.groupBy并且发现 Object.groupBy(items, callback) 是一个完美的代理执行器，机制是Object.groupBy遍历 items并对每个元素执行callback函数。

准备一个只有一项的数组，将我们生成的恶意函数 f 作为 callback 参数传入。

调用：Object.groupBy([0], f)。

触发：groupBy 内部会执行 f(0)。

结果：恶意代码被执行（弹窗弹出）。

这个Exploit完全没有利用内存破坏或者别的逻辑漏洞

最关键的还是Object.groupBy

![ ](/img/20251206-0.png)

正常来说是JS原生静态方法，把将任何可迭代对象中的元素进行分组。
Object.groupBy这里最主要的利用点是能把攻击者控制的数据值转换成对象的键名，所以能让它成为绕过特别是针对JSON.parse的防御的辅助工具。

### 为什么这里要用Object.groupBy？

在JavaScript标准库中攻击寻找的是静态方法，且该方法必须接受一个函数作为参数并执行它。

对比Array.prototype.map，它需要先有一个数组实例，而且需要通过原型链访问，步骤繁琐。但是Object.groupBy是 Object 构造函数的直接属性，通过原型链污染很容易拿到全局 Object 构造器，并且拿到 Object 后直接读取 Object.groupBy 即可使用不用实例化任何对象。

文章中的攻击流程主要分为三步，Object.groupBy在最后一步负责触发。

### 获取 Object 构造器

前文提到的WASM模块通过导入对象的漏洞，访问 constructor 属性获取宿主环境的全局 Object。

(Payload)

利用刚才获取到的 Object 再次访问 constructor 拿到 Function 构造器后生成一个恶意函数。

目标函数：`const evilFunc = new Function("alert(1)");`

如上述，在WASM的线性内存和类型系统中evilFun只是一个Refer，WASM指令集受到类型签名的严格限制，所以无法直接像JS那样随意调用这个动态生成的函数。

接下里就是Object.groupBy被作为代理调用者的时候

```
const dummyArray = [1]; 
const evilFunc = new Function("return alert('XSS')");
Object.groupBy(dummyArray, evilFunc);
```

执行过程是Object.groupBy开始遍历dummyArray → 必须决定如何对元素1进行分组 → 为了得到分组的 Key强制调用传入的第二个参数 → evilFunc 被执行 alert('XSS') 弹出。

### 复现&扩展

https://github.com/ThomasRinsma/wasm_js_escape/blob/main/payload.wat

![ ](/img/20251206-1.png)

如图，一个基础的弹窗。

⬆️，只要能执行alert(1)就意味着已经突破了WASM的线性内存限制，接触到了JS的全局作用域，在这个基础上可以扩展出所有标准JS能做的事情，像xss 篡改dom 内网扫描 csrf 服务端rce都是可行的。

像比较特色的扩展插件也有支持WASM的，比如Figma, VS Code Web和一些CMS中，利用这个漏洞的恶意插件包含无害胶水代码和被编译在二进制中的.wasm文件，可以是图像滤镜或者一个加密库。

当宿主环境将importObject传递给WASM内部的初始化函数的时候会立即执行原型链遍历，利用Object.groupBy触发Payload。

和NPM/浏览器插件投毒比较下好处在于理论上不需要申请任何敏感权限，却能获得最高权限，并且代码是二进制指令，如果不进行逆向很难看出它在遍历原型链。而且防御难度高，只要宿主环境在加载WASM时泄露了Object且没有补丁，任何静态扫描都无法阻止代码执行。

#### 扩展

之后可能会用到，写了网络侦察和持久化访问功能。
![ ](/img/20251206-2.png)

感觉利用写VS Code插件加载内置WASM模块再原型链逃逸有点慢了，而且目标大概没什么价值

可以到那种在线代码编辑器的技术社区求助，发个帖说自己遇到一个奇怪的WASM内存泄漏问题，只能在特定环境下复现，附上链接，然后让社区热心人点击链接。

许多在线IDE会在项目加载时自动运行npm install或者实时预览，所以我们的WASM模块也会在受害者的浏平台或者服务端渲染容器中启动。如果是云端容器，利用原型链逃逸读取容器的环境变量，如果是浏览器端就尝试读取IndexedDB中的敏感Session。

像Remix/Fork/Clone这些SaaS平台允许基于社区模板快速创建项目，但是如果不考虑和GitHub深度集成的平台（指面向非技术人员x），比如Webflow和Figma就支持一键导入复制。

所以先把自己的模版SEO毒化，再在深层的utils/helper.wasm或者npm依赖中植入自己的，这样受害者点击复制的时候平台自动拉取代码并运行构建脚本，就可以完成了（



