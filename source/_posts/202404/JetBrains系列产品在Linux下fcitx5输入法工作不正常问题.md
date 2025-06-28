---
title: JetBrains系列产品在Linux下fcitx5输入法工作不正常问题
date: 2024-04-04 22:02:17
category:
- 实践
tags: 
- fcitx
- 输入法
---
### JetBrains系列产品在Linux下fcitx5输入法工作不正常问题
##### 本机环境
1. 使用wayland
2. 使用fcitx5
3. 使用plasma6

##### 问题
1. 输入法不显示
2. 输入法显示但是候选框一直在左下角

##### 解决
**一、输入法不显示**

执行fcitx5自带的检查命令

```bash
> fcitx5-diagnose
```
发现有红色的提示，以下环境变量没有配置

```bash
# 可以直接加入/etc/enviroment
export XMODIFIERS=@im=fcitx
export QT_IM_MODULE=fcitx
export GTK_IM_MODULE=fcitx
```
加入后重启idea，发现可以显示输入法了，*但是电脑开始启动是会提示wayland不需要配置这些环境变量*

**二、输入法候选框位置没有跟随光标**

youtrack上反馈此问题，属于openJDK的bug，JetBrains修改了openJDK发了新的jbr。

下载连接：
[Wrong position of input window and no input preview with fcitx and ubuntu 13.04](https://youtrack.jetbrains.com/issue/JBR-2460/Wrong-position-of-input-window-and-no-input-preview-with-fcitx-and-ubuntu-13.04)


**三、其他问题解决**

因为我启动idea的bash脚本设置了“IDEA_JDK”变量导致第二步设置的jbr被忽略了，所以重新修改启动脚本，最终如下

```bash
#!/bin/sh

# 退出的适合取消环境变量
trap 'unset XMODIFIERS; unset QT_IM_MODULE; unset GTK_IM_MODULE' EXIT

# 这些设置变量可以去掉，反正没有
if [ -z "$IDEA_JDK" ] ; then
  IDEA_JDK="/usr/lib/jvm/java-21-openjdk/"
fi
# open-jfx location that should match the JDK version
if [ -z "$IDEA_JFX" ] ; then
  IDEA_JFX="/usr/lib/jvm/java-21-openjfx/"
fi
# classpath according to defined JDK/JFX
if [ -z "$IDEA_CLASSPATH" ] ; then
  IDEA_CLASSPATH="${IDEA_JDK}/lib/*:${IDEA_JFX}/lib/*"
fi

# 单独为idea设置输入法需要的环境变量
export XMODIFIERS=@im=fcitx
export QT_IM_MODULE=fcitx
export GTK_IM_MODULE=fcitx

# 自定了修复输入法的jbr，所以这里不要覆盖了
# exec env IDEA_JDK="$IDEA_JDK" IDEA_CLASSPATH="$IDEA_CLASSPATH" /usr/share/idea/bin/idea "$@"
exec env /usr/share/idea/bin/idea "$@"

# vim: ts=2 sw=2 et:
```
效果截图：

![输入法截图](/images/202404/输入法截图.png)
