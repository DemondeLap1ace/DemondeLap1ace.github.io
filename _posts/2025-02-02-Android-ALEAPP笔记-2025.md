---
layout:     post                       
title:         Android-ALEAPP笔记
subtitle:   
date:       2025-02-02              
author:     Closure                         
header-img: img/20250202-1.png
catalog: true                         
tags:                               
    - Android
    - 取证
---

> 整体描述：在一起谋杀案的调查中，警方获取了受害者的手机作为关键证据。通过与证人、受害者朋友及家属访谈，了解到受害者最近将所有钱都投入到一个投资交易类应用上并因此欠下债务，且与债主产生冲突。
> 朋友证词：“当我们在一起时，我的朋友接到了几个电话并让他回避。他说他欠来电者很多钱，但现在无力偿还”。

> 家属陈述：2023 年 9 月 20 日受害者离开了住处，没有告知目的地。事后从酒店得知受害者入住了 10 天，并在随后安排了航班。调查员推测行程信息可能保存在手机上。

我的分析顺序：明确核心，是涉及财产纠纷&行程去向&社交关系➡️通览ALEAPP报告和分类，看哪些数据模块有用➡️分析嫌疑应用的权限➡️通过通话/短信/应用使用/文件修改/浏览器访问/位置信息的时间戳做个时间线

### ALEAPP

用到的ALEAPP是个将 Android 系统内常见或重要的数据切片后分门别类地提取出来的软件。

https://github.com/abrignoni/ALEAPP

[https://abrignoni.blogspot.com/2020/02/aleapp-android-logs-events-and-protobuf.html](https://abrignoni.blogspot.com/2020/02/aleapp-android-logs-events-and-protobuf.html)

```
python aleapp.py -t <zip | tar | fs | gz> -i <path\_to\_extraction> -o <path\_for\_report\_output>
```

![ ](/img/20250202-2.png)
![ ](/img/20250202-3.png)
![ ](/img/20250202-4.png)

**Report Home**

这是 ALEAPP 生成的报告首页或汇总入口。

**ADB Hosts**

Android Debug Bridge主机信息文件记录了曾通过 USB 或网络连接到该手机的计算机或者主机信息。

**Call logs**

这里包含了手机的呼入、呼出、未接来电等通话信息，包括时间戳、持续时长、电话号码等。

**CHROMIUM**

在 Android 设备上，Chrome 等基于 Chromium 的浏览器会将浏览记录、缓存、Cookie 等信息存放在一些数据库中。包含了Cookies、Keyword Search Terms、Network Action Predictor、Search Terms、Web History、Web Visits。

**Contacts**

用户手机的联系人信息。

**DEVICE INFO**

包括了多个与设备本身设置和信息相关的文件或数据表：

**Partner Settings**

手机厂商或 Google 预设的系统设置或配置信息，用于记录手机固件、定制功能的状态等。

**SIM\_info\_0**

设备当前插入 SIM 卡的信息，如手机号码、运营商、SIM 卡序列号（ICCID）、IMSI 等。

**Settings\_Secure\_0**

设备的系统安全设置数据库。

**Discord Chats**

提取自 Discord 应用的聊天记录、用户信息、消息时间。

**EMULATED STORAGE METADATA**

这是关于手机“内部存储”的文件元数据信息，Android 会有一个 Emulated 存储区（虚拟 SD 卡），里面的文件、文件夹属性、大小、修改时间等元数据会被提取出来。

**FCM-Dump-com.discord**

Firebase Cloud Messaging 是 Google 提供的推送服务，这里显示 Discord 应用接收推送时所记录或缓存的一些数据。

**Image Manager Cache**

系统或应用在加载图片时会产生缓存（缩略图或临时文件）。

**App Icons**

提取已安装应用的图标。

**Google Play Links for Apps**

记录每个已安装 App 对应的 Google Play 商店链接（包名等），方便查看应用来源或在商店里的信息。

**Installed Apps (GMS) for user 0**

列举用户 0（即主用户）所安装的 Google Mobile Services 相关应用或其他应用。包括版本号、安装时间等。

**Packages**

详细列出了手机上所有安装的package，包含应用的包名、版本、签名信息。

**PERMISSIONS**

这部分集中展示了与应用权限相关的各类数据库或 XML 文件：

**Appops.xml**

Android 的 AppOps 机制，用来细分应用实际使用的敏感操作（如是否访问传感器、定位等）。

**Package and Shared User**

展示包与用户 ID、共享用户 ID 的对应关系。

**Permission Trees / Permissions / Runtime Permissions\_0**

Android 系统中权限管理的细分项目：哪些权限是普通权限、哪些是危险权限，哪些在运行时需要用户同意等。对隐私或取证来说非常关键，比如哪款应用能够访问通讯录、相机、定位等等。

**Last Boot Time**

记录手机上次重启或开机的时间，对确认设备的使用轨迹、上线下时间很有帮助。

**Recent Activity\_0**

可能是一些用户最近活动的记录，如应用打开时间、切换情况、活动面板（Activity）。

**SMS messages**

短信与彩信内容及元数据，包括发信人、收信人、时间、文本内容等。在取证或运营商分析中都非常关键。

**OS Version**

系统版本信息。

**UsageStats\_0**

Android 系统自带的 UsageStatsService，用来记录用户在一段时间里对应用的使用情况（启动次数、前台时间等）。

**Wi-Fi Hotspot / Wi-Fi Profiles / Wifi Configuration Store Combined - 0**

这些文件记录了手机连接过的 Wi-Fi 网络、配置参数（SSID、密码加密方式、上次连接时间）、以及个人热点配置等，用于分析用户可能去过哪些地点或场所（通过 Wi-Fi 名称，也叫做 SSID），或判断是否使用过手机热点分享网络。

**Factory Reset**

这是关于手机是否重置为出厂设置的操作信息。

### 分析嫌疑应用

![ ](/img/20250202-5.png)
![ ](/img/20250202-6.png)

Tick.olymptrade 谷歌搜索得出是金融交易软件

在Android的权限体系中，permission带有 android:protectionLevel 属性，用来表示该权限的保护级别，常见取值及意义如下：

* ​normal (0):普通权限，对用户影响较低。例如访问网络、设置闹钟等。系统在安装时自动授予，无需用户特别确认。
* ​dangerous (1): 危险权限，涉及用户隐私或可能造成费用。如 READ\_CONTACTS、READ\_SMS、ACCESS\_FINE\_LOCATION等。
* signature (2)：只有与声明此权限的应用拥有相同签名的其他应用才可获得该权限（可以理解为“同一开发者的应用内部共享”）。
* signatureOrSystem (3)​: 早期系统中存在的组合级别，仅当应用与系统签名相同或安装在系统分区（system app）时才能被授予。这在后续 Android 版本中逐渐弱化或被废弃。
  
  ![ ](/img/20250202-7.png)

如图，olymptrade的Protection= 2，表示该权限为 signature 级别，仅限同一签名才能调用。而且只看到自定义权限 DYNAMIC\_RECEIVER\_NOT\_EXPORTED\_PERMISSION 和在 UsageStats 中实际用到的 WAKE\_LOCK、READ/WRITE\_EXTERNAL\_STORAGE，但是没有看到 READ\_SMS、READ\_CONTACTS 等更敏感权限。发现了该 App 在 2023-09-19（晚上 19:57 左右）和 2023-09-20（晚上 20:59 到 21:02 左右）都有使用痕迹，吻合了题目的酒店时间。

### “当我们在一起时，我的朋友接到了几个电话并让他回避。他说他欠来电者很多钱，但现在无力偿还”

![ ](/img/20250202-8.png)
![ ](/img/20250202-9.png)

看通话记录和联系人直接得出债主联系方式和名字。
+201172137258 Shaday wahab

### 9.20日行为轨迹

受害者家属陈述为2023 年 9 月 20 日受害者离开了住处，没有告知目的地。事后从酒店得知受害者入住了 10 天，并在随后安排了航班。

首先查看wifi连接记录，发现没有可用的信息。

![ ](/img/20250202-10.png)

**补充**：连接记录能帮助我们推断用户曾在哪里出现或​何时到达某个地点​。先找到​Wi-Fi 配置或历史，因为在 ALEAPP 可以看到 Wi-Fi Profiles, Wi-Fi Hotspot, Wifi Configuration Store Combined - 0 等，调查时关注字段：SSID、BSSID（MAC）、加密方式、最近连接时间、上次自动连接时间等。如果 SSID 名字包含 “HotelXYZ” 或 “StarbucksABC”，能直接联想到地理场所，如果仅有 BSSID，可通过在线数据库（Wigle ([https://wigle.net/](https://wigle.net/)  *这是一个全球性的 Wi-Fi AP 数据库，用户可上传收集到的 Wi-Fi 信息（BSSID、坐标等），形成庞大的众包地图，取证后可在 Wigle 上搜索BSSID，看是否有人在某地扫到过它*)，无法直接获取 GPS 定位或基站信息的情况下，Wi-Fi 数据往往能提供室内或特定场所的轨迹。

​寻找“LastConnectedTime”或类似字段​，在 ALEAPP 输出的 Wi-Fi 模块（如 Wi-Fi Profiles, Wifi Configuration Store Combined）中，查看每个网络配置是否附带“最后连接时间（lastConnectedTime）”或“Connect time”。

有时候只能看到设备曾保存或自动连接过某 Wi-Fi，但是没有确切时间戳，这种情况可以尝试​交叉比对：在 UsageStats\_0 或 Event logs 等模块中查找当时的网络切换事件（Android 可能记录 “network\_state” 的变动时间），或在 Logcat/events.log 中找到 Wi-Fi 连接事件。如果 ALEAPP 未提取到，也要考虑单独解析原始文件。​最后排序所有可用时间戳​，标注地点或可能场景。

### RecentActivity: Recent Tasks, Snapshots & Images

![ ](/img/20250202-11.png)
![ ](/img/20250202-12.png)
![ ](/img/20250202-13.png)

​**RecentActivity**​（或“Recent Tasks”）是安卓系统用于记录近期运行应用的列表，包含：

* **Task\_ID**
* ​**Last\_Time\_Moved**​（最后一次切换到前台或更新任务时间）
* ​**Component / Real\_Activity**​（启动的具体组件/Activity）
* ​**Snapshot\_Image**​（最近任务列表中的屏幕截图/预览图）

从截图可以看到，多条记录分别对应三个应用：

1. **Discord**
2. **Google Maps**
3. **Chrome**

每个应用都包含以下信息：

* ​**Task\_ID**​: 标识系统中的任务（Task）ID
* ​**Effective\_UID**​: 应用运行时使用的用户 ID（Android 系统内部机制）
* ​**Affinity**​: 任务的 “Task Affinity”，通常与应用包名一致
* ​**Real\_Activity**​: 实际启动的活动（Activity），如 com.discord/.main.MainActivity
* ​**Last\_Time\_Moved**​: 最近一次把任务移到前台（或最近一次使用该任务）的时间
* ​**Calling\_Package**​: 谁调用/启动了这个应用（常见是 com.google.android.apps.nexuslauncher，即桌面启动器）
* ​**User\_ID**​: 用户标识（0 表示主用户）
* ​**Action**​: android.intent.action.MAIN（表示主入口）
* ​**Component**​: 组件路径，包括包名 + 具体 Activity
* ​**Snapshot\_Image / Recent\_Image**​: 代表系统在最近任务视图中截取的应用“快照”或“最近使用”界面缩略图。

通常 Android 会在 “最近使用的应用” 列表中为每个 Task 保存一个Snapshot以便在多任务切换时显示预览，如上图，谷歌搜索后得出在。

*Discord:Last\_Time\_Moved = 2023-09-20 23:50:24*

表示在这个时间里 ，用户将 Discord 的 Activity 切换至前台，或者刚刚离开该界面以切换到其他应用。

*Google Maps:Last\_Time\_Moved = 2023-09-20 23:50:29*

与Discord 时间非常接近，仅相隔 5 秒，这意味着用户快速地从 Discord 切换到 Google Maps。

*Chrome:Last\_Time\_Moved = 2023-09-20 23:50:55*

又过了约 26 秒，用户从 Google Maps 切换到 Chrome。

如果把这些时间点连起来，可以大致描述出一个切换顺序：

23:50:24 - Discord 活跃/离开→23:50:29 - 切换到 Google Maps→23:50:55 - 切换到 Chrome

用户先后在使用聊天、地图和Chrome，在 Discord 收到信息后，迅速查看地图位置，然后用 Chrome 查询其他资讯。

**补充**：给的sample并不完善，在实际取证分析的时候应该​对应外部线索。​如果在 Discord记录中也看到 23:50 左右有一段通信，且内容与“出门地点”或“聚会地点”相关，就可印证受害者当时在敲定行程。

后续能在 Google Maps 缓存或者 com.google.android.apps.maps 的历史记录中（找到 9 月 20 日晚的查询地点的话，便能推断用户想要去的具体位置。同理，在 Chrome’s Web History (via ALEAPP) 查到 23:50:55 以后浏览过哪些网站？是某家酒店预订页面，还是航班查询？

### Discord

![ ](/img/20250202-14.png)

从截图可见有两条聊天记录，均发生在同一个频道（​**Channel ID = 1153848030269804606**​）：

附加字段（Timestamp, Channel ID, ID, Username, Content, Mentions, Mention Roles, Pinned, Avatar, Edited Timestamp）是 Discord 提取的常规信息：

* Timestamp：消息的 UTC 时间
* Channel ID：对应的频道标识
* ID：该条消息的唯一标识
* Username：发送者
* Content：具体聊天内容
* Mentions/Mention Roles：是否@了某个用户或角色
* Pinned：消息是否置顶
* Avatar：用户头像哈希
* Edited Timestamp：消息是否编辑、何时编辑（此处均为空或缺省）

​第一条：2023-09-20 00:57:26 UTC，​第二条：2023-09-20 20:46:02 UTC。Discord 存储默认是 UTC 时间戳，可以推断“infern0\_o”在9月20日的凌晨（UTC）发消息，“rob1ns0n.”在同日稍晚时候才回复。

infern0\_o 提到：“Some changes have occurred in the plan.” “I have booked my travel ticket for 01/10 at 9:00 AM.”，表明之前有计划旅游，他重新订了票，​日期是 01/10，时间 9:00 AM。

**补充**：在部分国家/地区，“01/10” 可能表示 “10 月 1 日”；在美国常见格式中也可能是 “1 月 10 日”。结合前后脉络（案发 9 月 20 日），大概率这里是​10 月 1 日（一个多星期后）。

“Where am I supposed to meet you?”：询问见面地点，​rob1ns0n.​回复在The Mob Museum。结合上下文，infern0\_o 改变了原定出发日期或行程，重新预订了10月1日早上 9 点的航班；rob1ns0n告知将在拉斯维加斯的 The Mob Museum 见面。

### 谷歌

![ ](/img/20250202-15.png)
![ ](/img/20250202-16.png)

从 Visit Timestamp 与 Search Term 可以看出大部分访问集中在 ​2023-09-19（18:07 \~ 21:54左右），以及 ​2023-09-20（20:12 ~ 23:51）。主要内容为Gmail / Facebook / Amazon、YouTube和加密货币/交易平台，有相关搜索，如 “What is cryptocurrency trading?”、 “What are the best platforms for cryptocurrency trading?”还有旅游相关搜索，如 “What is the easiest country to travel to from Egypt?”以及航班变更/机票改签相关搜索，如 “How to reschedule an already booked flight?”

页面内容是阿拉伯语界面，可以暂且认为此设备/账号常在中东地区使用？

* 佐证对加密货币/在线交易平台的“近期兴趣”(被诈骗)
* 正探索“快速获利或出国”路径
* 已经或即将进行机票改签（或至少在搜索如何操作了）

### 时间线

9/19 18:07 - 21:54：浏览器大量访问电商、邮件、社交平台及加密货币交易信息；发现第一次使用 OlympTrade 投资 App 。

↓

9/20 00:57（Discord）：受害者（infern0\_o）通知重新订票至 10 月 1 日，询问见面地点。

↓

9/20 白天​：离家（家属证词），可能前往酒店；朋友证词期间还在接“债主”电话。

↓

9/20 20:46（Discord）：对方（rob1ns0n.）回覆说将在“Mob Museum”碰面。

↓

9/20 20:59 \~ 21:02​：再次使用 OlympTrade。

↓

9/20 23:50：通过 Discord → Google Maps → Chrome 的快速切换，进行定位 / 查询操作。

↓

9/20 深夜 / 9/21​：继续搜索机票改签方式，疑似在为后续离境做准备。

债主来电具体呼叫时段：与 9/19 \~ 9/20 间的通话日志相结合
OlympTrade / 投资软件，在 9/19 19:57 与 9/20 20:59 两次集中使用
Discord 约定地点，10 月 1 日早班机 → The Mob Museum (Las Vegas)




