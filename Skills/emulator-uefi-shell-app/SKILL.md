---
name: emulator-uefi-shell-app
description: |
  基于 EmulatorPkg 的 UEFI Shell 应用程序开发技能。当用户需要在 UEFI Shell 环境下开发应用程序、在 EmulatorPkg 模拟器中运行和调试时使用。覆盖 Shell App 创建、INF 文件配置、常用协议（GOP、SimpleTextIn）、编译部署、中文编码问题处理、以及模拟器调试方法。支持 Windows（VS2019）和 Linux（GCC）平台。
---

# EmulatorPkg UEFI Shell 应用程序开发指南

本技能提供在 EmulatorPkg 模拟器环境下开发 UEFI Shell 应用程序的完整流程，适用于 Windows（VS2019）和 Linux（GCC）平台。

## 目录

1. [环境搭建](#1-环境搭建)
2. [编译与运行模拟器](#2-编译与运行模拟器)
3. [创建 Shell 应用程序](#3-创建-shell-应用程序)
4. [部署到模拟器](#4-部署到模拟器)
5. [编译错误处理](#5-编译错误处理)
6. [模拟器调试方法](#6-模拟器调试方法)
7. [常用协议与库](#7-常用协议与库)

---

## 1. 环境搭建

### 1.1 Windows + Visual Studio 2019

**前置条件：**
- Visual Studio 2019（社区版/专业版/企业版）
- Windows SDK 10
- Git for Windows

**环境变量设置：**

```batch
:: 设置 VS2019 路径
set VS2019_PREFIX=C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\
set WINSDK10_PREFIX=C:\Program Files (x86)\Windows Kits\10\bin\10.0.19041.0\

:: 设置 EDK2 工作区
set WORKSPACE=<你的edk2路径>
set EDK_TOOLS_PATH=%WORKSPACE%\BaseTools
set PACKAGES_PATH=%WORKSPACE%
```

**初始化环境：**

```batch
cd %WORKSPACE%
edksetup.bat
```

### 1.2 Linux + GCC

**前置条件：**
- GCC 5+ 或 GCC 9+（推荐）
- GNU Make、Python 3.x
- libuuid-dev、nasm

```bash
# Ubuntu/Debian
sudo apt-get install build-essential uuid-dev iasl nasm python3

# 设置环境变量
export WORKSPACE=/path/to/edk2
export EDK_TOOLS_PATH=$WORKSPACE/BaseTools
export PACKAGES_PATH=$WORKSPACE

# 初始化
cd $WORKSPACE
source edksetup.sh
make -C BaseTools
```

### 1.3 获取 EDK2 源码

```bash
git clone https://github.com/tianocore/edk2.git
cd edk2
git submodule update --init
```

---

## 2. 编译与运行模拟器

### 2.1 配置 target.txt

编辑 `Conf/target.txt`：

```ini
ACTIVE_PLATFORM       = EmulatorPkg/EmulatorPkg.dsc
TARGET                = DEBUG
TARGET_ARCH           = X64          # 或 IA32
TOOL_CHAIN_TAG        = VS2019       # Windows
# TOOL_CHAIN_TAG      = GCC5         # Linux
```

### 2.2 编译命令

**Windows：**
```batch
build -p EmulatorPkg\EmulatorPkg.dsc -a X64 -t VS2019 -b DEBUG
```

**Linux：**
```bash
build -p EmulatorPkg/EmulatorPkg.dsc -a X64 -t GCC5 -b DEBUG
```

**输出文件：**
```
Build/Emulator/DEBUG_VS2019/X64/WinHost.exe    # Windows 模拟器
Build/Emulator/DEBUG_GCC5/X64/Host             # Linux 模拟器
```

### 2.3 运行模拟器

**Windows：**
```batch
cd Build\Emulator\DEBUG_VS2019\X64
WinHost.exe
```

**Linux：**
```bash
cd Build/Emulator/DEBUG_GCC5/X64
./Host
```

模拟器启动后进入 UEFI Shell 界面。

---

## 3. 创建 Shell 应用程序

### 3.1 目录结构

```
EmulatorPkg/Application/MyShellApp/
├── MyShellApp.inf     # 模块定义文件
├── main.c             # 主源文件
└── types.h            # 头文件（可选）
```

### 3.2 INF 文件模板

```ini
[Defines]
  INF_VERSION                    = 0x00010005
  BASE_NAME                      = MyShellApp
  FILE_GUID                      = 12345678-90ab-cdef-1234-567890abcdef
  MODULE_TYPE                    = UEFI_APPLICATION
  VERSION_STRING                 = 1.0
  ENTRY_POINT                    = UefiMain

[BuildOptions]
  MSFT:*_*_*_LINK_FLAGS = /SUBSYSTEM:EFI_APPLICATION

[Sources]
  main.c

[Packages]
  MdePkg/MdePkg.dec

[LibraryClasses]
  UefiLib
  UefiApplicationEntryPoint
  BaseLib
  BaseMemoryLib
```

**如需使用 Shell 库函数：**

```ini
[Packages]
  MdePkg/MdePkg.dec
  ShellPkg/ShellPkg.dec

[LibraryClasses]
  UefiLib
  UefiApplicationEntryPoint
  ShellLib
  BaseLib
```

### 3.3 基础代码模板

```c
#include <Uefi.h>
#include <Library/UefiLib.h>
#include <Library/UefiBootServicesTableLib.h>

EFI_STATUS
EFIAPI
UefiMain (
  IN EFI_HANDLE        ImageHandle,
  IN EFI_SYSTEM_TABLE  *SystemTable
  )
{
  Print(L"Hello, UEFI Shell!\n");
  return EFI_SUCCESS;
}
```

### 3.4 GOP 图形应用模板

```c
#include <Uefi.h>
#include <Protocol/GraphicsOutput.h>
#include <Library/UefiLib.h>
#include <Library/UefiBootServicesTableLib.h>
#include <Library/MemoryAllocationLib.h>

EFI_STATUS
EFIAPI
UefiMain (
  IN EFI_HANDLE        ImageHandle,
  IN EFI_SYSTEM_TABLE  *SystemTable
  )
{
  EFI_STATUS Status;
  EFI_GRAPHICS_OUTPUT_PROTOCOL *Gop;
  UINTN Mode;
  EFI_GRAPHICS_OUTPUT_MODE_INFORMATION *Info;
  UINTN InfoSize;

  // 定位 GOP
  Status = gBS->LocateProtocol(
    &gEfiGraphicsOutputProtocolGuid,
    NULL,
    (VOID **)&Gop
  );
  if (EFI_ERROR(Status)) {
    Print(L"Locate GOP failed: %r\n", Status);
    return Status;
  }

  // 查询并设置 1024x768 模式
  for (Mode = 0; Mode < Gop->Mode->MaxMode; Mode++) {
    Status = Gop->QueryMode(Gop, Mode, &InfoSize, &Info);
    if (EFI_ERROR(Status)) continue;
    if (Info->HorizontalResolution == 1024 &&
        Info->VerticalResolution == 768) {
      Gop->SetMode(Gop, Mode);
      break;
    }
  }

  // 填充屏幕为蓝色
  EFI_GRAPHICS_OUTPUT_BLT_PIXEL Blue = {255, 0, 0, 0};  // BGRA
  Gop->Blt(Gop, &Blue, EfiBltVideoFill,
           0, 0, 0, 0, 1024, 768, 0);

  return EFI_SUCCESS;
}
```

---

## 4. 部署到模拟器

### 4.1 在 DSC 文件中注册

编辑 `EmulatorPkg/EmulatorPkg.dsc`，在 `[Components]` 节添加：

```ini
[Components]
  # ... 已有组件 ...

  EmulatorPkg/Application/MyShellApp/MyShellApp.inf
```

### 4.2 在 FDF 文件中包含

编辑 `EmulatorPkg/EmulatorPkg.fdf`，在 `[FV.FvRecovery]` 节添加：

```ini
[FV.FvRecovery]
  # ... 已有条目 ...

  INF EmulatorPkg/Application/MyShellApp/MyShellApp.inf
```

### 4.3 重新编译

```batch
build clean
build -p EmulatorPkg\EmulatorPkg.dsc -a X64 -t VS2019 -b DEBUG
```

### 4.4 运行应用

1. 启动模拟器：`WinHost.exe`
2. 在 Shell 中输入应用名称：

```
Shell> MyShellApp
```

---

## 5. 编译错误处理

### 5.1 中文字符编码错误

**问题现象：**
```
warning C4819: The file contains a character that cannot be represented in the current code page (936)
error C2001: newline in constant
```

**解决方案一：在 INF 中添加编译标志**

```ini
[BuildOptions]
  MSFT:*_*_*_CC_FLAGS = /wd4819 /source-charset:utf-8
  MSFT:*_*_*_LINK_FLAGS = /SUBSYSTEM:EFI_APPLICATION
```

**解决方案二：在 DSC 中添加全局选项**

```ini
[BuildOptions]
  MSFT:*_*_*_CC_FLAGS = /wd4819 /source-charset:utf-8
```

**解决方案三：文件保存为 UTF-8 with BOM**

VS2019: 文件 → 高级保存选项 → 编码: Unicode (UTF-8 with signature)

### 5.2 常见警告抑制

```ini
[BuildOptions]
  MSFT:*_*_*_CC_FLAGS = /wd4819 /wd4100 /wd4204 /wd4245
```

| 参数 | 说明 |
|------|------|
| /wd4819 | 字符编码问题 |
| /wd4100 | 未引用的参数 |
| /wd4204 | 非常量聚合初始化 |
| /wd4245 | 有符号/无符号不匹配 |

---

## 6. 模拟器调试方法

### 6.1 Print 调试输出

```c
#include <Library/UefiLib.h>

Print(L"值: %d, 状态: %r\n", value, Status);
```

**格式说明符：**

| 格式 | 说明 |
|------|------|
| `%d` | 十进制整数 |
| `%x` | 十六进制 |
| `%lx` | 64位十六进制 |
| `%r` | EFI_STATUS |
| `%s` | Unicode 字符串 |
| `%a` | ASCII 字符串 |

### 6.2 DebugLib 调试

```c
#include <Library/DebugLib.h>

DEBUG((DEBUG_INFO, "信息消息\n"));
DEBUG((DEBUG_ERROR, "错误: %r\n", Status));

ASSERT(Status == EFI_SUCCESS);
```

### 6.3 配置调试级别

在 DSC 文件中：

```ini
[PcdsFixedAtBuild]
  # 启用所有调试输出
  gEfiMdePkgTokenSpaceGuid.PcdDebugPrintErrorLevel|0xFFFFFFFF

  # 启用调试属性
  gEfiMdePkgTokenSpaceGuid.PcdDebugPropertyMask|0x1f
```

### 6.4 VS2019 调试器

1. 打开 VS2019
2. 文件 → 打开 → 项目
3. 选择 `Build\Emulator\DEBUG_VS2019\X64\WinHost.exe`
4. 设置断点后按 F5 开始调试

---

## 7. 常用协议与库

### 7.1 GOP 图形协议

**头文件：** `MdePkg/Include/Protocol/GraphicsOutput.h`

```c
// 查询模式
Gop->QueryMode(Gop, ModeNumber, &InfoSize, &Info);

// 设置模式
Gop->SetMode(Gop, ModeNumber);

// 块传输
Gop->Blt(Gop, Buffer, Operation, SrcX, SrcY, DstX, DstY, Width, Height, Delta);
```

**Blt 操作：**
- `EfiBltVideoFill` - 填充
- `EfiBltVideoToBltBuffer` - 读屏幕
- `EfiBltBufferToVideo` - 写屏幕

### 7.2 文本输入协议

**头文件：** `MdePkg/Include/Protocol/SimpleTextIn.h`

```c
EFI_INPUT_KEY Key;

// 读取按键
gST->ConIn->ReadKeyStroke(gST->ConIn, &Key);

// 等待按键
gBS->WaitForEvent(1, &gST->ConIn->WaitForKey, &Index);
```

**扫描码：**

| 常量 | 值 | 说明 |
|------|-----|------|
| SCAN_UP | 0x0001 | 上箭头 |
| SCAN_DOWN | 0x0002 | 下箭头 |
| SCAN_LEFT | 0x0004 | 左箭头 |
| SCAN_RIGHT | 0x0003 | 右箭头 |
| SCAN_ESC | 0x0017 | ESC |

### 7.3 常用库

| 库 | 头文件 | 用途 |
|-----|--------|------|
| UefiLib | Library/UefiLib.h | Print 函数 |
| BaseMemoryLib | Library/BaseMemoryLib.h | 内存操作 |
| MemoryAllocationLib | Library/MemoryAllocationLib.h | 内存分配 |
| DebugLib | Library/DebugLib.h | 调试输出 |

### 7.4 内存分配

```c
#include <Library/MemoryAllocationLib.h>

VOID *Buffer = AllocatePool(Size);
VOID *Buffer = AllocateZeroPool(Size);
FreePool(Buffer);

VOID *Pages = AllocatePages(PageCount);
FreePages(Pages, PageCount);
```

---

## 快速参考

| 任务 | 命令 |
|------|------|
| 初始化环境 | `edksetup.bat` |
| 编译模拟器 | `build -p EmulatorPkg\EmulatorPkg.dsc -a X64 -t VS2019 -b DEBUG` |
| 运行模拟器 | `Build\Emulator\DEBUG_VS2019\X64\WinHost.exe` |
| 清理重编 | `build clean && build` |

---

## 故障排除

| 问题 | 解决方案 |
|------|----------|
| 找不到包 | 检查 PACKAGES_PATH 环境变量 |
| 应用不在 Shell 中 | 确认 DSC 和 FDF 都已添加 INF |
| 中文编码错误 | 添加 `/wd4819 /source-charset:utf-8` |
| GOP 定位失败 | 确保模拟器启动完成后再运行 |
| Print 无输出 | 检查控制台重定向设置 |