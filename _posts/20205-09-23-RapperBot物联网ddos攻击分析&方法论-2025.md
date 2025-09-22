---
layout:     post                       
title:     RapperBot物联网ddos攻击分析&方法论
subtitle:   
date:       2025-09-22       
author:     Closure                         
header-img: img/fm4.jpg
catalog: true                         
tags:                               
    - 笔记
    - 安全
---

从RapperBot整个攻击链里面学到的针对iot设备的技巧太多了，遂记录()

[部分代码详情-RapperBot: From Infection to DDoS in a Split Second](https://www.bitsight.com/blog/rapperbot-infection-ddos-split-second "部分代码详情-RapperBot: From Infection to DDoS in a Split Second")

[XLAB-僵尸永远不死：RapperBot僵尸网络近况分析](https://blog.xlab.qianxin.com/rapperbot/ "XLAB-僵尸永远不死：RapperBot僵尸网络近况分析")

[ioc](https://github.com/bitsight-research/threat_research/blob/main/rapperbot/rapperbot-iocs.json "ioc")

前情提要一下，RapperBot是基于IoT恶意软件Mirai变种的僵尸网络，通过感染并控制海量的互联网边缘设备来发起大规模的DDoS攻击。RapperBot是基于Mirai的，整体攻击杀伤链与Mirai都是遵循着大规模扫描-利用弱点-植入恶意软件-C2控制-发起DDoS的模式 。RapperBot可以被视为Mirai演化树上的一个重要分支，没有颠覆性创新而是在一个已被证明成功的模型基础上通过引入新的TTPs来适应和利用一个略有差异的生态位。

![ ](/img/20250923-1.png)

提炼了RapperBot能借鉴的针对iot设备攻击的几个面，第一是战略上的简单性，它没有建立在利用day上面，相反的它核心优势是攻击模型的极致简单和可大规模复制的特性，RapperBot抓住了当前iot设备生态的弱点：数量庞大 直接暴露于公网 缺乏安全维护

第二是一次性资产模型，跟web那些寻求在受害设备上长期潜伏的恶意软件不同，他们摒弃了传统的持久化机制，不试图在设备重启后继续存活，而是选择依赖其持续不断的 高强度的互联网扫描和再感染能力来动态维持和补充其僵尸网络的规模。

第三是精准的攻击向量，虽然RapperBot的侦察阶段表现为广撒网式的扫描，但在漏洞利用阶段可以看出他们对特定目标设备的了解，所以没有喷射各种漏洞利用代码，而是针对已识别的设备类型采用多步骤的攻击链。

### 侦察阶段

![ ](/img/20250923-0.png)

要利用的漏洞包括但不限于⬆️多数漏洞有两个共同且关键的特点，一个是rce一个是弱认证。

这些漏洞通常出现在设备的web/CGI 接口、固件升级接口和解析功能中，如果厂商在接受用户输入时没有过滤或转义特殊字符，攻击者就能把shell注入进去然后在系统上执行任意命令。并且许多漏洞是pre-auth，扫描器只要能访问到设备的公网端口就能直接利用，省去了暴力破解或社工步骤。

iot一般都部署在边缘不易维护，而且很多固件很少更新或根本停更，厂商也不会强制用户修改默认密码。还有厂商为了功能或兼容性使用了很多不同的服务，攻击向量也因此多样。

RapperBot定制了SSH暴力破解扫描器，在恶意软件样本初始化后这个是扫描器首批被调用的，执行流程：main函数调用resolve_host来解析目标，scanner_init进行初始化，最后由scanner_start启动实际的扫描和攻击循环。

看了奇安信lab的逆向分析，他们没有使用任何开源的SSH库，而是选择从头编写一个轻量级的SSH客户端。首先就避免使用通用库而产生的可被轻易识别的特征，可以绕过了许多基于签名的ID。

通过定制还可以精确地只实现暴力破解所需的最基本功能，来剔除所有不必要的复杂性，让最终生成的二进制文件体积更小 运行效率更高。（满足资源极其受限的嵌入式设备上运行的恶意软件特征

这个扫描器的工作模式是在一个循环中迭代一个硬编码在二进制文件内部的用户名和密码组合列表，通过分析这份凭证列表可以发现列表中不仅包含了Linux服务器和网络设备的常见默认凭证，还可能包含特定IoT平台或应用程序的默认登录信息。

### 权限提升与据点巩固

脚本除了优先尝试使用curl、wget、ftpget这些在绝大多数基于Linux的系统中预装的标准化工具之外，还有一个可以运用到之后实战的技巧。他们用了NFS来允许一个系统通过网络挂载远程服务器上的目录，并像访问本地文件一样访问它们。

如果发现目标设备上没有wget等工具，但是开放了NFS客户端功能就可以将恶意软件放在自己控制的NFS服务器上，然后命令受害设备挂载这个远程目录，最后直接从挂载点复制并执行恶意程序。

NFS挂载和本地文件复制操作比通过HTTP或FTP下载文件更不寻常，可以完全绕过基于网络流量特征的检测。

衍生想了一下，还可以尝试下用I/O重定向与网络工具，目标上可能没有wget但是几乎所有的BusyBox工具箱中都包含了nc和dd。这种最底层的管道式文件传输几乎无法被基于应用层协议特征的检测工具发现，我们可以在自己的服务器上监听一个端口并提供恶意文件，然后在目标设备上用nc连接该端口，并将其输出通过管道直接重定向给dd工具，由dd将接收到的二进制流写入文件。

### 防御规避

RapperBot执行后的首要动作之一就是进行自我伪装躲避系统管理员的初步审查，比如在一个繁忙的Linux系统中，一个名为rapperbot.arm7的进程会立即引起警觉。但是如果一个进程显示为 [kworker/u1:1] 或 (systemd)，就能轻易地混入众多合法的内核工作线程和系统守护进程中。

⬆️这种欺骗是通过调用Linux 特有的 prctl() 系统调用来实现的，RapperBot使用 PR_SET_NAME 选项来修改其调用线程的名称，这个名称被记录在 /proc/[pid]/comm 文件中，并被多种系统监控工具读取和显示。

```cpp
#include <stdio.h>
#include <string.h>
#include <sys/prctl.h>
#include <unistd.h>

/**
 * @brief 将当前进程的名称修改为一个不起眼的系统进程名以规避检测。
 * 
 * 该函数使用 prctl() 系统调用和 PR_SET_NAME 选项来更改调用线程的
 * “comm”字段。选择的名称，如“[kworker/u1:1]”，旨在模仿合法的
 * Linux 内核线程，使其在进程列表（如 top 或 ps 的输出）中不那么显眼。
 * 
 * @return 0 表示成功, -1 表示失败。
 */
int masquerade_process() {
    // 选择一个常见的、看起来合法的进程名。
    // 名称长度必须小于 16 个字节（包括结尾的 null 终止符）。
    const char* new_name = "[kworker/u1:1]";

    // 调用 prctl() 来设置进程名。
    // PR_SET_NAME 是第一个参数，指定要执行的操作。
    // 第二个参数是新名称字符串的地址。
    // 其余参数在此操作中未使用，应设置为 0。
    if (prctl(PR_SET_NAME, (unsigned long)new_name, 0, 0, 0) == -1) {
        perror("prctl(PR_SET_NAME) failed");
        return -1;
    }

    printf("Process name changed to: %s\n", new_name);
    
    // 进程将在此处继续执行其他恶意活动，但其名称已被伪装。
    // 例如，可以进入一个无限循环以保持进程存活。
    // sleep(60); 

    return 0;
}

int main() {
    printf("Original process PID: %d\n", getpid());
    
    if (masquerade_process()!= 0) {
        fprintf(stderr, "Failed to masquerade process.\n");
        return 1;
    }

    // 在这里，如果用 "ps -eo comm,pid | grep <PID>" 查看，
    // 进程名将显示为 "[kworker/u1:1]"。
    
    // 模拟恶意软件的持续运行状态
    printf("Masquerading successful. Process is now running in disguise.\n");
    while (1) {
        sleep(10);
    }

    return 0;
}
```

在完成初步伪装后RapperBot会激活杀手模块，这个模块的设计目标非常明确，继承并强化了Mirai的核心理念：确保对受感染设备的绝对和独占控制。它通过两个层面的清洗动作来实现这一目标：

- 消除竞争对手：该模块会扫描系统中的所有活动进程，并终止属于其他已知僵尸网络家族的进程，例如 QBOT 和 Zollard 。通过这种方式，RapperBot 确保了对主机 CPU、内存和网络带宽等宝贵资源的独占权，避免了与其他恶意软件的资源争夺。
- 阻止应急响应：主动终止合法的远程管理服务，特别是 sshd 和 telnetd。这一行为的直接后果是将设备的合法所有者和系统管理员锁在门外，使其无法通过标准远程方式登录系统进行故障排查、日志分析或恶意软件清除。这极大地增加了应急响应的难度和成本，为 RapperBot的长期驻留创造了有利条件。

这个c代码是一个简化的杀手模块，通过遍历 /proc 目录来识别系统中的所有正在运行的进程然后读取每个进程的名称，并与一个预定义的必杀列表进行比对。一旦匹配成功就用 kill() 系统调用发送SIGKILL信号终止目标进程。

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dirent.h>
#include <ctype.h>
#include <signal.h>
#include <unistd.h>

// 预定义的“必杀”列表
const char* target_processes = {
    "sshd",
    "telnetd",
    "zollard", // 竞争性僵尸网络
    "qbot",    // 竞争性僵尸网络
    "mirai",   // 竞争性 Mirai 变种
    "httpd"    // Web 管理服务
};
const int num_targets = sizeof(target_processes) / sizeof(target_processes);

/**
 * @brief 遍历 /proc 目录，查找并终止在 target_processes 列表中的进程。
 *
 * 该模块实现了两个核心目标：
 * 1. 消除竞争：终止其他僵尸网络的进程，以独占系统资源。
 * 2. 锁定访问：终止 sshd 和 telnetd 等远程管理服务，阻止管理员进行修复。
 */
void execute_killer_module() {
    DIR *proc_dir;
    struct dirent *entry;

    // 打开 /proc 目录
    if ((proc_dir = opendir("/proc")) == NULL) {
        perror("opendir(/proc) failed");
        return;
    }

    // 遍历 /proc 目录中的所有条目
    while ((entry = readdir(proc_dir))!= NULL) {
        // 检查条目名是否为纯数字，这代表它是一个进程ID (PID)
        int is_pid = 1;
        for (char *c = entry->d_name; *c; c++) {
            if (!isdigit(*c)) {
                is_pid = 0;
                break;
            }
        }

        if (is_pid) {
            char comm_path;
            FILE *comm_file;
            char process_name = {0};
            pid_t pid = atoi(entry->d_name);

            // 构建指向 /proc/[pid]/comm 文件的路径
            // 这个文件包含了进程的名称
            snprintf(comm_path, sizeof(comm_path), "/proc/%s/comm", entry->d_name);

            if ((comm_file = fopen(comm_path, "r"))!= NULL) {
                if (fgets(process_name, sizeof(process_name), comm_file)!= NULL) {
                    // 移除末尾的换行符
                    process_name[strcspn(process_name, "\n")] = 0;
                    
                    // 检查进程名是否在我们的目标列表中
                    for (int i = 0; i < num_targets; i++) {
                        if (strcmp(process_name, target_processes[i]) == 0) {
                            printf("Target found: %s (PID: %d). Terminating...\n", process_name, pid);
                            
                            // 发送 SIGKILL 信号 (信号编号 9) 来强制终止进程
                            // 这是一个无法被捕获或忽略的信号
                            if (kill(pid, SIGKILL) == 0) {
                                printf("Process %s (PID: %d) terminated successfully.\n", process_name, pid);
                            } else {
                                perror("kill failed");
                            }
                            break; // 找到并处理后，跳出内层循环
                        }
                    }
                }
                fclose(comm_file);
            }
        }
    }

    closedir(proc_dir);
}

int main() {
    printf("Executing killer module to eliminate rivals and lock down the environment...\n");
    execute_killer_module();
    printf("Killer module execution finished.\n");
    return 0;
}
```

虽然RapperBot的主要感染向量是通过 SSH 暴力破解实现的但是杀手模块却会终止sshd 服务。因为RapperBot 在成功入侵后，会通过修改 ~/.ssh/authorized_keys 文件来植入自己的SS公钥，建立一个永久性的无需密码的后门 。在建立了这个更为隐蔽和可靠的后门之后，再终止 sshd 服务，可以一举多得，阻止了其他攻击者利用相同的SSH暴力破解方法再次入侵该设备。

### C2通信

这个C2基础设施的建立过程是客户端首先会尝试通过一组硬编码在程序中的DNS服务器IP地址，来解析一组同样硬编码的域名，这里的C2服务器的真实IP地址不是直接存储在A记录中，而是被编码在这些域名的DNS TXT记录里。Bot客户端会查询TXT记录并从中提取出C2服务器的IP地址列表，这样可以随时更换后端的C2服务器，只需修改DNS TXT记录就可以重定向到新的服务器。

在与C2服务器建立连接后通信采用了一种自定义的加密数据包格式，关键特征是数据包的长度是可变的并且会使用随机数据进行填充。以及简单XOR加密和数据包的头部包含一个checksum，这里的细节设计是校验和的计算范围仅限于包头本身，并不覆盖加密的载荷部分。

为什么只用单字节异或不做复杂密钥交换，这样不会很容易被检测识别吗？我看到溯源之后第一反应是这个，后面结合了iot环境来看，它不能阻挡逆向分析但是对于大量自动化感染和在物联网环境下，需要快速部署&短生命周期的恶意软件来说，这种简单混淆可以绕过最基础的签名匹配检测并且实现成本低且性能开销小。

IoT 设备通常没有安装主机型杀毒软件。所以攻击者更依赖网络侧/基于签名的检测器被动失效来获得存活空间，许多网络入侵检测或防火墙只做简单的字符串/正则匹配，或者只关注常见端口流量，轻量混淆加上使用常见服务端口和预认证 RCE 的直接执行路径能让自动化利用链条更容易成功。

### 无持久化

RapperBot战术选择是明确放弃了在受感染设备上建立持久化机制的策略，在一般通过恶意软件里这属于是一个技术缺陷，但是在RapperBot的运营背景下这是一个非常合适的选择。

这种策略极大程度上增加了隐蔽性来有效规避了检测。持久化机制里的修改系统启动脚本和cron jobs是HIDS和EDR的重点监控，通过完全在内存中运行并放弃任何形式的文件系统修改顺便显著减少了在受害设备上留下的指纹。一旦设备因断电或者重启就会被从内存中彻底清除，一是难以捕获和分析样本，二是也增加了对其进行特征化和签名的难度。

因为最终目的是ddos，所以完全不需要永久性地占有某一个设备，当一个被感染的节点因为重启而丢失时，扫描模块可以在几分钟内找到并感染另一个脆弱节点。

### iot攻击方法论

看了如此多新闻，物联网生态系统的脆弱性很明显是组件和通信协议的多样性。

物联网设备的是嵌入式硬件以及连接不同网络协议的物联网网关，这些硬件往往为了控制成本而牺牲了安全性能，比如缺少安全启动机制或物理防篡改设计。运行在设备上的嵌入式软件是控制设备行为的关键，像如AWS IoT Core这些云平台提供了设备管理服务，像固件代码中的漏洞和不安全的更新机制以及配置不当都是很常见的攻击向量。  

iot这些协议根据距离、功耗和带宽需求可分为：短距离无线协议，Wi-Fi、低功耗蓝牙、Zigbee和NFC

长距离无线协议，4G/5G LPWAN LoRaWAN和Sigfox，适用于城市广域应用。消息传递协议有MQTT和CoAP，是专为资源受限设备设计的轻量级协议用于设备与云端之间的通信 。  

上述每一种硬件 固件 云平台和通信协议都有潜在漏洞......

单独列物联网的漏洞是肯定没用的，得理解这些漏洞为什么普遍且难以根除。首先是物联网设备在设计和运行上与传统IT设备存在基本差异，绝大多数物联网设备都是资源受限的，它们通常配备低功耗的处理器和有限的内存以及存储空间，并依赖电池供电。这种硬件限制使得实现安全功能在计算上变得成本过高。

iot物联网生态系统的构建是一个由无数种不同制造商生产的设备 专有操作系统和通信协议组成的巴别塔，这种极度的异构性和标准化的缺乏让开发通用的安全解决方案和管理工具都非常困难，无法像在Win那样用统一的工具来管理。

还有为iot设备打补丁比为服务器打补丁复杂，设备数量大且地理位置分散，手动更新不切实际，并且许多设备缺乏可靠的空中下载更新机制，以及iot设备的生命周期通常很长但制造商的安全支持周期却很短。

像之前提到的Mirai，成功的地方就是极致简单性和高效的传播，在互联网上大规模随机地扫描开放着Telnet服务端口，只是依赖于一个很基础的默认凭证，它的代码中硬编码了一个包含约62组用户名和密码组合的字典，这些组合都是物联网设备最常见的出厂默认凭证。一旦扫描到响应Telnet的设备Mirai就会尝试使用这个字典进行暴力破解登录。

IoT攻击不需要太多0day，我们是在一个混乱无序且防御薄弱的环境中利用普遍存在的 意料之中的设计缺陷。先作一个宏观环境分析，那种售价便宜的个人级IoT设备最有可能在安全上做出妥协。

测绘的时候可以参考下Mirai，就是一部自动化扫描器在整个IPv4空间中持续寻找开放了Telnet端口且使用约60组常见默认口令的设备。关键词就是默认服务端口 特定产品指纹和默认凭证。

绘制完整攻击面关注设备是否可能被物理接触，像公共场所的摄像头可以物理接触意味着可以利用UART调试接口直接提取固件。

还有设备/固件层，设备运行了哪些软件？版本号是多少？设备与App、云端之间使用什么协议？数据是否加密？即使加密，证书校验是否严格？

云端/移动应用层是分析与设备交互的手机App和云API，这些接口是否存在弱密码策略 未授权访问等Web应用的常见漏洞。

初步入侵的时候先暴力破解，用包含数百个常见IoT设备默认用户名/密码的字典自动化地对第一阶段发现的目标进行登录尝试。

[字典1](https://github.com/govolution/betterdefaultpasslist "字典1")

[Passwords/Default-Credentials/目录中，可以找到mirai-passwords.txt 直接复现的Mirai僵尸网络使用的字典。](https://github.com/danielmiessler/SecLists/tree/master/Passwords/Default-Credentials "Passwords/Default-Credentials/目录中，可以找到mirai-passwords.txt 直接复现的Mirai僵尸网络使用的字典。")

以及扫描目标设备开放的服务来对照CVE寻找已公开但未修复的RCE洞，像过时的mini_httpd服务器、存在漏洞的UPnP库、旧版本的OpenSSL都在考虑范围。

还可以将自己置于设备与路由器之间，比如通过ARP欺骗or加入同一wifi，如果通信未加密可以直接嗅探到敏感数据，如果使用了TLS但实现错误则可以发动中间人攻击。

反正如果能接触到设备就拆开外壳找UART调试口，连接后直接获得root shell。。。

