---
layout: post
title: "Understanding Android Obfuscation Techniques: A Large-Scale Investigation in the Wild"
date: 2019-09-27 00:02:33 -0400
comments: true
categories: 
---
作者：Shuaike Dong, Menghao Li, Wenrui Diao, Xiangyu Liu, Jian Wei Liu, Zhou Li, Fenghao Xu, Kai Chen, Xiaofeng Wang, Kehuan Zhang

单位：The Chinese University of Hong Kong, Institute of Information Engineering, Jinan University, Alibaba, Indiana University Bloomington

出处：SecureComm 2018

论文：[Paper](https://arxiv.org/pdf/1801.01633.pdf)
<hr/>
# Abstract
在Android程序中，由于Java程序具有易于逆向的特性，所以其代码保护尤为重要。代码混淆技术被常规的应用程序开发人员和恶意软件作者广泛使用，为逆向分析增加了许多难度。尽管在混淆技术上已经有很多研究，但是对现实世界开发人员如何使用混淆技术的了解仍然有限。在本文中，作者通过大规模的野外调查来展现真实世界中Android混淆技术的使用情况。在文中作者重点关注4种流行的混淆方法：标识符重命名，字符串加密，Java反射和加壳。作者的APK数据集来自于Google Play，多个第三方市场和恶意软件数据库中。作者发现恶意软件作者更频繁地使用字符串加密，而加壳技术的应用在第三方市场中比Google Play更普遍。
<!--more-->

截止2017年三月，仅仅在Google Play上已经有2.8m个app可供用户下载。同样混淆技术也无处不在，合法的软件公司使用混淆技术防止程序被轻易逆向或者被抄袭代码，恶意软件作者使用混淆技术规避杀软和自动化分析。有许多混淆器被开发并公开，有些混淆器的作者声称他们的工具被超过30万的app所使用。这一现象吸引了大量研究人员分析app混淆技术，但是到目前为止，所有研究的关注点在于使用了什么样的混淆技术，某种混淆的具体实现是什么，这种混淆如何抵御最先进的分析工具，以及如何自动化地反混淆这种混淆技术。然而本文作者的重点在于消除混淆技术分析领域存在的gap：这些混淆技术在现实世界中是如何被众多的开发者所使用的。

作者认为了解混淆技术的使用可以更好地帮助设计代码分析工具，并优先处理最需要解决的挑战。

作者的主要贡献如下：

- 系统研究： 系统地研究了应用程序开发人员使用的当前主流Android混淆技术。
- 新技术：作者提出了几种准确检测不同混淆技术的技术，如基于n-gram的重命名检测模型和基于反向切片的反射检测算法。
- 大规模评估： 作者进行了大规模的实验，从三个不同来源收集的超过10万的APK文件。 并根据分析结果对混淆代码的深入分析提供了解释。

# 2 BACKGROUND

###2.1 APK File Structure

众所周知APK包含如下文件：

- res：Android 资源文件，根据ID隐射至.R文件 
- assets：存放静态文件
- lib：native lib文件
- META-INF：签名信息文件
- AndroidManifest.xml
- classes.dex：Dalvik二进制文件
- resources.arsc：APK资源索引表文件

### 2.2 Android Obfuscation Characterization

作者重点关注四种混淆技术，并在检测中着重针对这四种混淆技术进行检测

**Identifier Renaming **在程序开发中，代码标识符的名称通常是有意义的，尽管开发人员可能遵循不同的命名规则（如驼峰命名法，匈牙利命名法）。 但是，这些有意义的名称也便于逆向工程师理解代码逻辑并快速定位目标函数。为了减少潜在的信息泄漏，标识符的名称使用无意义的字符串替换。 以下代码段给出了一个示例，其中重命名了Account类中的所有标识符。

![1533990482850](/images/2019-09-27/1533990482850.png)

**String Encryption **在混淆的应用程序中，原始明文被随机字符串替换并在运行时使用解密函数解密恢复。字符串加密可以有效地阻碍硬编码静态扫描。 示例如下：

![1533991251565](/images/2019-09-27/1533991251565.png)

![1533991269077](/images/2019-09-27/1533991269077.png)

**Java Reflection **反射是Java的一个高级功能，它为开发人员提供了一种与程序交互的灵活方法，可以用来创建新的对象实例和动态调用方法。  以下代码块提供了一个调用隐藏API batteryinfo的反射示例：

![1533991341986](/images/2019-09-27/1533991341986.png)

作为一种混淆技术，反射是隐藏程序行为的一个很好的选择，因为它可以隐式地将控制转移到某个特定的函数，也是静态分析工具无法很好地处理的一种混淆方式。 因此，恶意软件开发人员通常会大量使用反射来隐藏恶意操作。

**Packing **加壳是一种广泛使用的代码保护技术。 加壳的APK文件由加密的原始APK和壳APK组成。 当用户启动APK时，壳将首先运行，解密原始APK并将其加载到内存中，然后把执行交给解密的APK。 由于加密过程在运行时尽心，通过静态分析很难获得原始代码。 作者认为加壳是广义上的混淆技巧，因为它的目标是阻碍逆向分析。

# 3 SYSTEM DESIGN

### 3.1 System Overview

![1533991825484](/images/2019-09-27/1533991825484.png)

为了检测混淆技术的使用，作者提出了一种自动分析APK文件的架构，如上所示。从几个渠道收集的APK文件存储在服务器中，检测框架将尝试解压它们进行主要测试。

某些损坏的APK文件未通过此步骤将被丢弃。然后该框架应用四种有针对性的检测方法来识别混淆的Smali代码块。这些检测方法可以分为两类：基于签名和基于机器学习。

对于具有特定特征的混淆技术，作者在Smali代码中搜索相应的代码逻辑签名以确定是否存在。例如，可以通过搜索序列模式[Class.forName()→getMethod()→invoke()]来定位隐式调用另一个函数的反射调用。

然而，对于某些混淆技术（例如加密字符串）难以提取固定特征，因此作者利用机器学习进行检测。

### 3.2 APK Dataset

![1533992087417](/images/2019-09-27/1533992087417.png)

作者在实验中使用了三个代表性的APK数据集：Google Play（26,614个样本），第三方市场（65,666个样本）和恶意软件（22,280个样本） 。这些样本是在2016年和2017年期间收集的。实验数据集包含114,560个样本，大小约为1.521TB。表1中给出了更多详细信息。作为Android的官方应用商店，Google Play是主要的Android应用分发渠道。因此，其样本集可以反映主流开发人员使用的混淆的部署状态。此外，由于政策限制，在某些国家/地区（例如中国），Google Play无法使用，用户必须从第三方市场安装应用。因此，在第二个数据集中，作者选择了来自中国的六个热门应用市场（Anzhi，Xiaomi，Wandoujia，360，Huawei和AppChina）,并排除来自不同市场的相同样本。最后，作者还对恶意软件作者是否大量使用混淆技术好奇。因此，最后一个数据集包含来自VirusShare和VirusTotal的恶意软件样本。

#4 OBFUSCATION DETECTIONS AND LARGE-SCALE INVESTIGATION

### 4.1 Identifier Renaming

标识符重命名操作可以在APK文件打包的不同层面进行，例如ProGuard和Allatori工作在源代码层面，可以根据开发者的配置对标识符进行重命名，而像DashO，DexProtector和Shield4J可以直接修改APK文件中的.dex和.class文件。

**Identifier Renaming Detection **检测标识符重命名通过结合可计算linguistics和机器学习技术来实现的。主要分为以下三步：

1. 数据预处理：因为开发者会使用很多第三方的lib来避免重复开发，为了反映开发者使用混淆技术的真实情况，需要对APK中的第三方库进行移除。作者使用`Li et al`方法一共移除了12000个第三方库。
2. 特征生成：作者使用n-gram算法扩建3-gram^2算法构建通用表达式来生成记录连续单词的特征向量。
3. 分类：训练集基于F-Droid，然后作者使用不同的混淆器对Android应用源码混洗生成APK，最后使用支持向量机作为分类算法。

**Large-scale Investigation and Findings **

作者研究的目的是揭示在野的Android应用混淆情况，所以最后使用训练模型与分别来自Google Play，第三方应用商店和恶意软件样本进行检测。作者检测结果显示：

- 与Google Play商店相比，第三方应用市场使用了更多的重命名混淆方式
- 超过三分之一的恶意软件没有使用重命名混淆
- 但是恶意软件会使用更加复杂的重命名策略

具体检测结果数据如下图所示：

![1534001981368](/images/2019-09-27/1534001981368.png)



### 4.2 String Encryption

.dex文件中的字符串加密通常是会在混淆后，存储一个加密后的字符串，然后再运行时调用解密函数解密。作者使用如下两个层面分析字符串加密混淆：

- 检测APK是否使用字符串加密
- 检测APK使用了哪种加解密函数进行的字符串加解密

**String Encryption Detection **

仅仅检测部分的方案使用的是和标识符重命名检测类似的方案，旧瓶装新酒。只不过训练集是使用4.1中混淆工具的字符串加密功能生成的字符串当做数据样本。

**Cryptographic Function Analysis **

对于加解密字符串所使用的具体的算法，作者是基于前任在二进制代码加密算法检测方面的工作上添加了一些额外的特征检测：

- 比特和循环操作比
- Java密码学扩展API的调用
- 对于str和char类型变量的操作
- 加密字符串作为函数参数的使用频率

**Large-scale Investigation and Findings **作者检测结论如下：

- 几乎所有的合法App都没有使用字符串加密
- 字符串加密在恶意软件中更常见
- 大约有17.6%的字符串加密恶意软件实现了多种加密算法
- 加解密函数所使用的秘钥可以是静态硬编码的也可能是动态生成的

具体分布如下：

![1534004033481](/images/2019-09-27/1534004033481.png)

### 4.3 Reflection

因为反射是程序中较为常见的程序逻辑，所以并不是所有包含反射的代码就一定都是有混淆目的的，作者探讨了关于反射在混淆中的两个问题：

1. 在野应用中反射的使用有多广泛？
2. 这些使用了反射的应用，有多少是处于混淆目的的？

作者使用如下这种函数调用逻辑来匹配使用反射的程序片断：`[Class.forName() → getMethod() → invoke()]`，这种使用特定序列模式检测反射的方法由Li et al`提出。

**Reflection Detection **具体的检测如下：

1. 首先通过扫描代码，定位到包含`Class.forName()`和`getMethod()`的代码，参数注册会被设置为切片标准（slicing criterion）
2. 然后会跟踪定位到的代码处，分析每条指令找打欧对饮更低slices
3. 最后工具会解析并在slices中模拟每条指令

**Large-scale Investigation and Findings ** 作者检测结论如下：

- 开发者在恶意软件和合法软件中使用反射的请款差不多
- 大部分反射的目的在于实现应用的兼容性
- 和常规软件相比，恶意软件会更多地使用复杂的反射去实现隐藏的操作

具体分布如下：

![1534005487093](/images/2019-09-27/1534005487093.png)

###4.4 Packing

加壳不同于上诉三种混淆，是针对整个应用逻辑的保护。加壳的检测就比较简单，作者罗列了6中不同的壳所具备的特征，然后根据这些特征为指纹去检测应用：

![1534005617203](/images/2019-09-27/1534005617203.png)

**Large-scale Investigation and Findings ** 作者检测结论如下：

- 第三方应用市场（国内应用市场）和恶意软件会更多地使用加壳

具体分布如下：

![1534005674158](/images/2019-09-27/1534005674158.png)

# 5 DISCUSSION

作者讨论了本文存在的限制和问题，例如没有考虑控制流混淆和native lib混淆。作者认为控制流混淆不够通用，只有小部分混淆器支持，而且并没有提供如这些混淆器所说那么强的控制流混淆效果。而native lib的混淆对于开发者来说需要很高的C/C++开发能力，也同样不是Android应用混淆的主流，而且作者认为native lib的混淆或者保护应当属于一个独立的研究领域，不在本文的讨论范围之内。


