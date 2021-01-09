

# 网易易盾的防反编译、加固等技术实现

[TOC]

## 一、`Android`客户端安全攻防分析

### 1.1 反编译安全威胁

#### 1.1.1 攻击方式

* 文件反编译：`DEX`文件、`SDK`文件、`SO`文件、资源文件；
* 代码分析：`Java`代码、`C/C++`代码、`JS/HTML`代码；
* 逆向破解：调试、抓包、`HOOK`注入、绕过签名校验等。

#### 1.1.2 安全威胁

* 逆向分析：代码调试，漏洞挖掘，协议分析；
* 二次打包：`APP`盗版仿冒，插入广告、病毒木马，修改资源等；
* 功能破解：`VIP`，会员，内购破解，去广告等。

### 1.2 `Android`应用加固

#### 1.2.1 加固清单

| 功能         | 说明                                                      |
| ------------ | --------------------------------------------------------- |
| `DEX`加固    | 对`DEX`文件进行加壳保护，防止被静态反编译工具破解获取源码 |
| 防二次打包   | 应用在被非法二次打包后不能正常运行                        |
| 防调试器     | 防止通过调试工具对应用进行非法破解                        |
| 内存防`dump` | 防止运行时在内存中`dump`数据                              |
| 资源文件保护 | 加密资源文件，防止`APK`资源文件被破解                     |
| `H5`文件混淆 | 对`JS/HTML`代码文件进行保护，防止破解分析                 |
| `SDK`加固    | 对`jar/AAR`文件进行保护，防止反编译获取源码               |
| `SO`加密保护 | 对`SO`进行加壳，保护`native`代码不被逆向分析              |

#### 1.2.2 网易易盾解决方案

`Android`加固：

> 1. `DEX`加固;
> 2. `SDK`加固；
> 3. 资源加固；
> 4. `SO`加固；
> 5. `H5`保护。

#### 1.2.3 `Android`加固主要功能

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/guard_function.png)

## 二、加固技术实现

### 2.1 `DEX`加固

* **防反编译：**基础的做法是针对市面上出现的反编译工具做对抗，找漏洞，使得这些工具无法反编译。后来升级到`VMP`加固和`java2c`的方案之后，防反编译已经不是主要问题了，因为被网易易盾加固后，即使反编译出来的内容也不是原来的代码了，仅仅是一些无用的外壳代码。
* **`VMP`加固：**主要是把`dex`中的函数指令运行在壳的`VMP`环境中，极大地提高了破解的门槛，让破解变得不可能。
* **`Java2c`：**通过把`java`代码在加固阶段就转换为`Native`层的`c`代码，破解分析完全不可逆，在性能上相较于`VMP`加固也有很大改进。

#### 2.1.1 保护`classes.dex`

* 加固前的`classes.dex`和`clases2.dex`直接暴露给了分析者；
* 加固后只有一个`classes.dex`，用`baksmali.jar/jeb`等工具查看，无法查看原`app`代码逻辑，同时保护`classes.dex`与`classes2.dex`。

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/protect_classes_dex.png)

 防止`classes.dex`被逆向分析

* 加固前很容易反编译出`jar`包文件查看源码；
* 加固后让反编译失败，并无从查看源码。

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/protect_classes_dex2.png)

 代码逻辑混淆替换

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/code_rpoguard.png)

#### 2.1.2 `VMP`保护效果

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/vmp_result.png)

`VMP`指令替换：

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/vmp_command_replace.png)

`VMP`执行流程：

加固指令加密替换 --> `APP`运行 --> 外壳执行 --> 加载加密`DEX`指令 --> 易盾虚拟机解析指令

#### 2.1.3 `Java2c`保护效果

方法级，`Java`代码彻底转换为`Native`代码指令。与`VMP`相比，用空间换时间。

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/java2c_result.png)

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/java2c_result2.png)

`java2c`原理：

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/java2c_theory.png)

#### 2.1.4 防止动态调试

加固后`APK`一旦被调试，`APP`异常退出，阻断调试。使用`IDA Pro`调试`APP`自动退出。

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/prevent_dynamic_debug.png)

### 2.2 资源加固

#### 2.2.1 `assets`资源透明加解密

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/assets_proguard.png)

#### 2.2.2 `res`资源混淆保护

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/res_proguard.png)

#### 2.2.3 `H5`保护效果--`js`保护

保护方式：

* 字符串加密；
* 混淆/去`log`/变量名混淆/函数名混淆；
* 压缩；
* 游戏保护/平台识别；
* 防篡改/防加速。

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/js_proguard.png)

#### 2.2.4 `H5`保护效果--`html`保护

保护方式：

* 加密后内容分块；
* 乱序插入干扰信息；
* 压缩处理，去除注释无用属性等；
* 平台识别，检测到非移动平台，不显示网页。

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/html_proguard.png)

### 2.3 `SDK/SO`加固

#### 2.3.1 `SDK`加固

从代码安全、文件安全等方面对`SDK`进行保护。对抗反编译手段，防止恶意篡改`SDK`、窃取用户隐私信息等，有效提升`SDK`保护的强度。

* `SDK`接口保持不变，对接入者透明；
* 支持`JAR`包、`AAR`包；
* 可灵活指定加固保护的类和函数；
* 加固体积增量几乎无影响。

**`Java`代码保护**

对`JAR`包的保护，我们采取**将重要逻辑代码抽离保护**的方案，在运行时再动态修复回去，这样接入者在开发阶段仍然可以依赖`JAR`包作为库文件直接引用接口函数，保持了接入者开发的透明。但是实际代码又是抽离的，我们测试时只选取了`xxxx.jar`中类`com.test.app.view.ExampleShell`的接口进行处理，那么接入者在打开`JAR`包时只能看到如下的效果：

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/java_shell_proguard.png)

函数的指令代码被隐藏保护了，相应的，其它类和函数也可以依此方案处理。

由于代码逻辑被隐藏掉了，接入者看不到代码逻辑，也就无法分析甚至阉割`SDK`的功能了。

#### 2.3.2 `SO`加固

`SO`加固可以阻断`IDA`分析`so`及`so`代码加密。`SO`加固后`IDA`将无法正常打开`SO`进行分析，`IDA`会显示如下报错：

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/so_proguard_error.png)

`SO`加固后与加固前已没有任何相似性，代码合字符串都被加密（左边为原始`SO`，右边为加固后的`SO`）:

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/so_proguard_result.png)

**`SO`导出函数隐藏**：

`SO`调用接口常常以`Java_xxx_xxx_xxx`形式导出，从接口名字即可大致看出该函数的功能。`SO`加固可将该函数完全隐藏，此处将反`IDA`静态分析功能去掉做示例对比：

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/so_proguard_theory.png)

**`SO`加固反调试**：

`SO`加固反调试效果，`IDA attach`到进程上后，`IDA`面板都为空，获取不到指令和模块信息：

![image](https://github.com/tianyalu/NeProgramGuardDecompilation/raw/master/show/so_prevent_debug.png)

**`SO`加固保护优点**：

| 优点             | 介绍                                                         |
| ---------------- | ------------------------------------------------------------ |
| 无需源码         | 只需要编译好的`so`即可加保护，相比于有些厂商提交源码的`so`保护方式，更为安全，有效保护厂商的技术机密 |
| 关键函数动态加密 | 一些关键函数运行完后，对其进行动态加密，提升安全性           |
| 全平台支持       | 支持`armeabi`、`armeabi-v7a`、`x86`、`arm64-v8a`、`x86_64`五大平台 |
| 防系统`API HOOK` | `Hook`一些系统`API`，比如`strcmp`、`strcpy`等字符串操作函数就可以获取到很多关键信息。加保护后，这些`hook`将获取不到任何信息 |
| 无法脱壳         | 直接对`ELF`结构进行改造，仅在运行时存在可运行的代码，其它结构全部重构，任何时刻都不存在原生`so`的内存 |

