---
title: "android SELinux权限配置"
date:  2023-12-06T10:36:27+08:00
draft: true
categories: [android]
tags:
  - android
  - selinux
---

## SELinux policy介绍

​	SELinux中，每种东西都会被赋予一个安全属性，就是SContext一个字符串，主要由三部分组成。

### 进程的SContext

​	进程的SContext可通过ps -Z命令来查看，如下图所示：

<img src="./../img/ps-z.png" alt="img"  />

​	以第一个进程/init的SContext为例，其值是u:r:init:s0,其中：

1. u为user的意思，一个SELinux用户
2. r为role的意思
3. init，代表该进程所属的domain为init
4. s0为进程的级别

### 文件的SContext

​	文件的SContext可通过ls -Z来查看，如下图所示：

![img](./../img/ls-z.png)

​	以最常见到的audioserver为例，其信息为u:object_r:audioserver_exec:s0:

1. u为user的意思，一个SELinux用户
2. object_r:文件的角色
3. audioserver_exec：代表该进程所属的domain为audioserver_exec，而exec表示该文件是可执行文件
4. s0为进程的级别

**根据SELinux规范，完整的SContext字符串为：user:role:type[:range]；注意方括号中的内容表示可选项。**

### android源码中的SELinux定义

**在android平台定制的SELinux，是通过编译sepolicy；（android7.0开始目录为system/sepolicy，而在device目录下有不同厂商的定义自己的sepolicy文件夹）**

1. ​	system/sepolicy：提供了Android平台中的安全策略源文件。同时，该目录下的tools还提供了诸如m4,checkpolicy等编译安全策略文件的工具。注意，这些工具运行于主机（即不是提供给Android系统使用的）
2. external/libselinux：提供了Android平台中的libselinux，供Android系统使用。
3. external/libsepol：提供了供安全策略文件编译时使用的一个工具checkcon。

**对我们而言，最重要的还是external/sepolicy。通过如下make命令查看执行情况： mmm system/sepolicy --just-print**

1. sepolicy的重头工作是编译sepolicy安全策略文件。这个文件来源于众多的te文件，初始化相关的文件（initial_sid,initial_sid_context,users,roles,fs_context等）。
2. file_context：该文件记载了不同目录的初始化SContext，所以它和死货打标签有关。
3. seapp_context：和Android中的应用程序打标签有关。
4. property_contexts：和Android系统中的属性服务（property_service）有关，它为各种不同的属性打标签。

**SELinux分为两种模式，android 5.0后所有进程都是使用enforcing mode**

```
enforcing mode ： 限制访问
permissive mode： 只审查权限，不限制
```

**SELinux Policy文件路径**

```
# Google原生目录
external/sepolicy (android7.0开始目录为system/sepolicy)

#厂家目录,高通将mediatek 换为 qcom
alps\device\mediatek\common\sepolicy
alps\device\mediatek\<platform>\sepolicy
```

## 问题分析

​	当在android系统执行过程中SELinux权限缺失时，会报如下错误，我们可以根据错误内容进行对应的权限添加；

​	` 	type=1400 audit(862271.000:14): avc: denied { read write } for pid=666 comm="cookoosvc" name="ttyHSL3" dev="tmpfs" ino=14643 scontext=u:r:cookoosvc:s0 tcontext=u:object_r:quec_device:s0 tclass=chr_file permissive=0`

1. 缺少什么权限： denied { read write }
2. 访问的目标文件或者进程名称：name=ttyHSL3
3. 源SContext标签：scontext=u:r:cookoosvc:s0
4. 目标SContext标签：tcontext=u:object_r:quec_device:s0
5. 目标进程或者文件的类型：tclass=chr_file

然后我们就可以在自己定义的cookoosvc.te文件下添加对应的权限，添加案例如下：

```
allow cookoosvc quec_device:chr_file rw_file_perms;
```

allow是允许，cookoosvc服务或者文件访问quec_device类型的设备节点，目标进程或者文件的类型是chr_file，并且给予读写权限

## SELinux Sepolicy中添加权限

修改相应源.te文件（基本以源进程名命名），以cookoosvc.te为例，添加如下：

```
type cookoosvc, domain;
type cookoosvc_exec, exec_type, file_type;

init_daemon_domain(cookoosvc)

binder_use(cookoosvc)

# using binder call

binder_call(cookoosvc, platform_app)
binder_call(cookoosvc, system_app)
binder_call(cookoosvc, servicemanager);
binder_call(cookoosvc, system_server)

binder_service(cookoosvc)

allow cookoosvc cookoo_service:service_manager {add find};

#Allow mediacodec to access proc_net files
allow cookoosvc proc_net:file r_file_perms;
allow cookoosvc rootfs:lnk_file { getattr };

allow cookoosvc appops_service:service_manager find;
allow cookoosvc audioserver_service:service_manager find;
allow cookoosvc batterystats_service:service_manager find;
allow cookoosvc cameraproxy_service:service_manager find;
allow cookoosvc mediaserver_service:service_manager find;
allow cookoosvc processinfo_service:service_manager find;
allow cookoosvc scheduling_policy_service:service_manager find;
allow cookoosvc surfaceflinger_service:service_manager find;

allow cookoosvc system_file:file { execute_no_trans rx_file_perms };
allow cookoosvc system_file:dir r_dir_perms;
allow cookoosvc quec_device:chr_file rw_file_perms;
```

***通常不在Google default的policy下修改，推荐更新厂商相关的policy***

### 文件配置相关的

​	如果进行文件相关的policy配置，需在file_contexts中绑定对应的file，以system/bin/cookoosvc为例，如下:

```
/system/bin/cookoosvc                                                u:object_r:cookoosvc_exec:s0

在相应源.te文件添加文件type的安全类型，如下：
type cookoosvc_exec, exec_type, file_type;
这是定义cookoosvc_exec类型的文件是文件类型和可执行文件类型
```

如果是设备节点类型需在file_contexts中定义device类型，以ttyHSL3为例，如下：

```
/dev/ttyHSL3                                    u:object_r:quec_device:s0
```

如果需要访问的某类型文件的权限，需对对应的申请方式如下：

```
#格式
allow 源类型 目标类型:访问类型  {操作权限};//注意分号

allow cookoosvc system_file:file { execute_no_trans rx_file_perms };
allow cookoosvc system_file:dir r_dir_perms;
allow cookoosvc quec_device:chr_file rw_file_perms;

chr_file - 字符设备 file - 普通文件 dir - 目录
```

### 添加property

添加一个自定义的system property: perisist.dome,并为system_app设置读写权限

1. 在property.te中定义system property类型
   ```
   type demo_prop,property_type
   ```

2. 在property_context中绑定system property的安全上下文

   ```
   persist.demo  u:object_r:demo_prop:s0
   ```

3. 在system_app.te中新增写权限，可以使用set_prop宏

   ```
   set_prop(system_app,demo_prop)
   ```

4. 在system_app.te中新增读权限，可以使用get_prop宏

   ```
   get_prop(system_app,demo_prop)
   ```

###   添加系统服务

​	有一个扩展的系统服务，需system_app进行bind绑定；以cookoosvc为例

1. 在service_context中绑定service

   ```
   cookoo.uart.service                            u:object_r:cookoo_service:s0

   # cookoo.uart.service是bind通信中ServiceManager绑定的名称
   ```

2. 在service.te中对service类型定义

   ```
   type cookoo_service,              service_manager_type;

   # 这个意思是将cookoo_service类型的服务绑定service_manager_type类型
   app_api_service - 应用的api类型服务 system_api_service - 系统的api类型服务
   system_server_service - 系统类型服务 service_manager_type - ServiceManager标签类型
   ```

3. 给system_app进行绑定bind通信权限

   ```
   在相应源.te文件添加服务绑定的安全类型，如下
   binder_call(cookoosvc, system_app)

   并在system_app.te下添加service_manager的add和find权限
   allow system_app cookoo_service:service_manager { add find };
   ```

### 使用Local socket

​	一个native service通过init创建一个socket并绑定在/dev/socket/demo，而且允许某些process访问。

1. 在file.te中定义socket的类型

   ```
   type demo_socket, file_type;
   ```

2. 在file_context中绑定socket的类型

   ```
   /dev/socket/demo u:object_r:demo_socket:s0
   ```

3.  允许所有的process访问

   ```
   #使用宏unix_socket_connect(clientdomain,sockrt,serverdomain)
   unix_socket_connect(appdomain,demo_socket,demo)
   ```

### init fork 新进程

​	在init.rc启动service时，需添加selinux，以cookoosvc为例，如下：

1. SELinux Policy文件路径下创建一个相应源.te文件（基本以源进程名命名），如《文件配置相关的》栏

2. 在cookoosvc.te中定义cookoosvc类型，init启动service时类型转换。根据cookoosvc需要访问的文件权限以及设备，定义其它权限在cookoosvc.te中。

   ```
   type cookoosvc, domain;
   type cookoosvc_exec, exec_type, file_type;

   init_daemon_domain(cookoosvc)
   ```

3.  绑定执行档file_context类型

   ```
   /system/bin/cookoosvc                                                u:object_r:cookoosvc_exec:s0
   ```

4. 创建cookoosvc的入口执行档cookoosvc_exec、并配置相应的权限（如果遇到问题需添加权限，可以根据日志进行相应的分析）；

## 抓取SELinux log

1. 抓kernel log, adb shell dmesg
2. 抓kernel log, 使用命令，可以直接提出avc的log：adb shell "cat /proc/kmsg | grep avc" > avc_log.txt
3. adb logcat -b events,搜索关键字：avc: denied

实际的项目使用中应根据log分析缺少的权限，来对应添加；再次强调，***通常不在Google default的policy下修改，推荐更新厂商相关的policy***



