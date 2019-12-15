---
title: ETW 理解
date: 2019-12-10 11:18:10
tags: 
- tracing system
categories: 科研
---
# 背景介绍
软件系统越来越复杂，要解释软件的所有执行状态是不可能的，应用在运行时不可避免地会出现意料之外的行为。软件和硬件的大量耦合以及workload的不同特性使得软件问题诊断变得十分困难。使用插桩能够改善软件可靠性和可管理性。  
插桩具有以下作用：
- 注入错误，使软件执行快速失败，从而减少debug时间。
- 解决性能问题，获取软件性能退化时的trace数据，定位出问题组件、服务或者瓶颈。
- 根据transaction的trace数据统计资源使用，进行容量规划和趋势分析。
<!-- more -->

# ETW
## ETW简介
ETW的全称是Event Tracing for Windows，是windows提供的一个通用和高速的tracing工具。ETW的buffering和logging机制在内核中实现，用户态应用程序和内核态设备驱动程序利用ETW提供的机制来产生事件。ETW能够动态enable或者disable logging，logging的数据先存在per-processor buffer中，然后由一个异步的线程写回到硬盘中。  
ETW首先在windows 2000引入，之后它就成了windows平台上重要的插桩技术。ETW被抽象到了windows preprocessor software tracing technology中。在windows Vista上，ETW经历了一次大的升级，重要的变化之一是统一事件生产者模型和API的引入。这个统一的API将记录trace和写到事件查看器结合起来，成为对事件生产者一致且易用的机制。  

## ETW architecture
![ETW architecture](/images/etw_arch.gif)
ETW的架构包含4个主要的组件：
### 事件生产者
逻辑上存在的实体，把事件写到ETW会话。任何重要的可记录的活动都可以一个事件。事件生产者可以是用户态的应用程序，管理应用程序，驱动或者其他软件实体，唯一时的要求是事件生产者必须通过注册API注册一个生产者ID。生产者先注册到ETW，然后在代码中的各个位置调用ETW的logging API产生事件。生产者受控制器的动态控制，控制器决定生产者是否启动，产生的事件被发送哪个trace会话。事件生产者发送的事件由一个包含事件元数据的固定头部和额外的用户上下文数据组成。当事件被记录到会话中时，ETW会在其中加入一些额外的数据，包括时间戳，进程和线程ID，处理器编号，logging线程的CPU使用，这些数据记录在事件头部中并传送给事件消费者。
### 控制器
控制器开始或停止ETW会话，启动会话的事件生产者。在一些场景中，例如debugging和diagnosis，需要调用控制器工具来收集traces。
### 事件消费者
事件消费者是一个应用程序，读取log文件或者监听一个会话来获取实时事件并处理它们。事件的消费是基于回调的，消费者注册一个事件回调，每次有一个事件，ETW调用这个回调。事件按照时间顺序被递交给消费者。下面展示了内核生产者记录的“Process”事件的xml dump。
```
<Event xmlns="http://schemas.microsoft.com/win/2004/08/events/event">
    <System>
        <Provider Guid="{9e814aad-3204-11d2-9a82-006008a86939}" />
        <EventID>0</EventID>
        <Version>2</Version>
        <Level>0</Level>
        <Task>0</Task>
        <Opcode>1</Opcode>
        <Keywords>0x0</Keywords>
        <TimeCreated SystemTime="2006-12-18T12:26:27.887309500Z" />
        <Correlation 
            ActivityID="{00000000-0000-0000-0000-000000000000}" />
        <Execution ProcessID="3396" ThreadID="3260" ProcessorID="0" 
            KernelTime="390" UserTime="195" />
        <Channel />
        <Computer />
    </System>
    <EventData>
        <Data Name="UniqueProcessKey">0xFFFFFA800143FA80</Data>
        <Data Name="ProcessId">0x10EC</Data>
        <Data Name="ParentId">0xD44</Data>
        <Data Name="SessionId">1</Data>
        <Data Name="ExitStatus">0</Data>
        <Data Name="UserSID">guest</Data>
        <Data Name="ImageFileName">notepad.exe</Data>
        <Data Name="CommandLine">notepad</Data>
    </EventData>
    <RenderingInfo Culture="en-US">
        <Opcode>Start</Opcode>
        <Provider>MSNT_SystemTrace</Provider>
        <EventName xmlns=
            "http://schemas.microsoft.com/win/2004/08/events/trace">
            Process</EventName>
        </RenderingInfo>
        <ExtendedTracingInfo xmlns="
            http://schemas.microsoft.com/win/2004/08/events/trace">
        <EventGuid>{3d6fa8d0-fe05-11d0-9dda-00c04fd7ba7c}</EventGuid>
        </ExtendedTracingInfo>
</Event>
```
生产者使用新API产生事件时，需要提供一个称为事件清单的XML文件，这个文件定义了所有事件以及它们的布局。一个通用的消费者应用程序使用Tracer Data Helper这个API提取事件元数据，解码事件，然后展示出来。
### 事件trace会话
负责buffering和logging，接收事件，创建trace文件。有多种logging模式。每个会话有一个独立的写者线程会把事件刷新到文件中,或者发给实时消费者应用程序。使用pre-processor buffering使得在logging时消除了使用锁的需要，从而获得更高性能。   
   
tracing意味着从感兴趣的生产者收集事件。一个事件trace会话会绑定到一个或多个生产者。ETW允许更加动态和灵活的trace和事件管理，控制器动态启动或停止ETW会话，启用会话的生产者。

## 统一事件生产者模型和API
1. 组件先使用EventRegister API注册成为生产者，要有一个GUID来唯一标识，这个GUID称为providerId。
2. 在旧的模型中，生产者必须对enable和disable通知提供一个回调。控制器启用生产者时，回调函数被调用。回调函数的一个参数是会话的句柄。在收到enable的回调之后，生产者设置一个全局变量来指示追踪是否开启，保存会话的句柄，这个句柄在使用logging API时需要用到。在新的模型中，ETW代表生产者记住enable的设置，生产者直接注册和调用logging API，而不用检查它们是否被启动了。在使用logging API时，生产者快速查看enable的setting，如果被启用了，则把事件发送给会话，否则就丢弃这个logging调用。
3. 在新的模型中，logging API使用的是注册句柄，而不是会话句柄，这样enable的setting就变得对生产者透明。ETW也能够多播一个事件，意思是一个生产者能被enable给多个会话。
4. 用户能够在事件中添加上下文数据，logging API使用scatter/gather机制来操作具体事件的数据。调用者通过创建数据描述符的数组来传递额外的事件数据。对每一个需要记录的数据，用户添加一个数据描述符。ETW在logging时将这些用户提供的内容复制到会话的buffer中。
5. 在事件清单中，事件的布局必须通过一个\<Template\>标签来声明。Template描述了一个事件包含的用户指定的上下文数据。Template中能定义事件的布局，事件的布局会包含单独的数据域。事件不一定需要Template，没有指定Template被认为没有包含用户数据。当消费者遇到事件时，它先找到事件的Template，然后再根据这个Template解码出各个数据。

## 端到端追踪
端到端的请求追踪时，一个请求会经过应用程序的多个部件，多台机器，需要用一个唯一的ID来标识每一个请求。有了这个ID，在后续的消费过程中就可以把这些事件关联起来。  
ETW在事件中引入了ActivityId，ActivityId存储在当前的执行线程中，事件被记录时，这个ActivityId就被使用。ActivityId能够伴随请求传播到各个组件。有时由于协议或设计的限制，ActivityId无法传播，这种情况下可以用EventWriteTransfer API产生一个传输事件，表示ActivityId的传输。

## 事件
每一个事件都会打上一个生产者ID，并被赋予一个事件描述符。事件描述符定义了事件中的标准信息，提供更多的标志和语义信息。在插桩时，开发者要决定插桩点的事件描述符，在事件清单中写入对应的条目。Message Compiler根据给定的事件清单生成事件描述符，保存在头文件中，之后include这个头文件就能在源码中使用。  
事件ID用来在生产中中唯一标识一个事件。消费者收到一个事件时，使用它的生产者ID和事件ID赖在事件清单中查找事件的定义。可以用version来标识事件的改动，version加上事件ID和生产者ID唯一地标识一个事件。

## 通道
通道定义了发给目标接收者的一组事件。通道是下列四种类型之一：admin，operational，analytic，debug。  
- 放入admin通道的事件是可操作事件，接收到这种事件后，管理员马上知道事件为什么发生以及怎么处理。
- 放入operational通道的事件目标接收者是监控工具和support staff，它们提供了更详细的上下文信息。
- Analytic通道是为了将传统的trace数据提供给专家级的支持人员或者详细诊断或者troubleshooting工具。
- debug通道用于debug信息，包含的事件将给开发者消费。  

## 级别和关键字
启用一个生产者时，控制器指定级别和关键字。级别用来根据事件的严重性和冗长程度进行过滤，关键字用来指示生产者中的子部件。例如，开发者能把事件划分为信息性事件和关键错误事件，对应用中的不同部件赋予不同的关键字，有选择地启用不同的级别和关键字，控制器就可以使生产者只记录从某个部件产生的错误事件，或者是某些部件产生的所有事件。当控制器启用一个级别时，该级别之上的级别也会启动。

## 任务和操作码
任务和操作码给事件添加额外的信息。任务指定了被插桩的component或task，这里的task通常表示component为达成目标所采用的关键步骤。操作码表示事件被写下时发生的具体操作。例如，windows内核生产者将所有文件IO操作事件归类为“FileIOP”任务，操作码表示是什么操作：create、open、read、write。

## 事件清单
事件描述符和布局在事件清单中说明。开发者在设计插桩的时候写好事件清单，事件清单是一个XML文件，用户定义的通道，任务，操作码，级别和关键字在这个文件中用相应的标签声明。XML中的事件标签下声明了事件的元数据，这个标签与事件ID唯一关联。下面是事件清单的一个样例片段。
```
<provider name="Microsoft-Windows-Kernel-Registry" 
    guid="{70eb4f03-c1de-4f73-a051-33d13d5413bd}" 
    symbol="RegistryProvGuid"
    resourceFileName="%SystemRoot%\System32\advapi32.dll" 
    messageFileName="%SystemRoot%\System32\advapi32.dll">
  <channels>
    <channel name="Microsoft-Windows-Kernel-Registry/Analytic" 
        chid="RegistryEvents" symbol="REG_Events" type="Analytic" 
        isolation="System">This channel contains registry 
        events.</channel>
  </channels>
  <opcodes>
    <opcode value="32" name="CreateKey" symbol="" />

    ...

  </opcodes>
  <keywords>
    <keyword name="CreateKey" symbol="" mask="0x1000" />

    ...

  </keywords>
  <templates>
    <template tid="tid_RegOpenCreate">
      <data name="BaseObject" inType="win:Pointer" 
         outType="win:HexInt64" />
      <data name="KeyObject" inType="win:Pointer" 
          outType="win:HexInt64" />
      <data name="Status" inType="win:UInt32" 
          outType="win:HexInt32" />
      <data name="Disposition" inType="win:UInt32" />
      <data name="BaseName" inType="win:UnicodeString" 
          outType="xs:string" />
      <data name="RelativeName" inType="win:UnicodeString" 
          outType="xs:string" />
    </template>

    ...

  </templates>
  <events>
    <event value="1" symbol="ETW_REGISTRY_EVENT_CREATE_KEY" 
        template="tid_RegOpenCreate" opcode="CreateKey" 
        channel="RegistryEvents" level="win:Informational" 
        keywords="CreateKey" 
        message="$(string.Registry.RegOpenCreate)"/>

        ...

      </events>
    </provider>

      ...

  <localization>
    <resources culture="en-US">
      <stringTable>
        <string id="Registry.RegOpenCreate" 
            value="Registry key %6 was created with status %3." />
      </stringTable>
    </resources>
  </localization>
```

# 总结
总的来看，ETW的设计跟现有的端到端追踪系统有很多共同之处，它的生产者相当与追踪系统广泛使用的agent，会话相当于collector，事件查看器相当于webUI，区别在于概念上和实现方式上。ETW做的也是源代码插桩，而且产生的事件要预先定义，这表示对于突发事件，ETW可能无法处理，这是它的一个局限性。ETW设计了一个控制器，能够启用或停用生产者，使得它具有动态控制的能力。ETW对于事件的产生和发送的控制粒度是比较细的，这体现在生产者产生的事件被发送到不同的通道，生产者又与会话绑定，事件可以根据级别和关键字来过滤。这样的做法有助于集中观察某些事件或某些组件，更快地定位出问题。另外ETW的tracing还考虑到了权限和安全，这貌似是现有的大多数追踪系统所不具备的。  

---

来源：[Improve Debugging And Performance Tuning With ETW](https://docs.microsoft.com/zh-cn/archive/msdn-magazine/2007/april/event-tracing-improve-debugging-and-performance-tuning-with-etw)
