---
title: "StowPy dotfile manager 构想"
description: "关于这个项目的初步构思以及功能设想"
pubDate: 2026-08-02
tags: ["个人项目", "StowPy"]
---

## 问题

GNU Stow 将文件和文件夹视为树形递归结构， 并且假设 **源路径的树形结构（Source Tree）与目标路径的树形结构（Target Tree）是 1:1 完全同构的**

形式为：`dotfiles/pkg/A/B/file -> ~/A/B/file`

这种同构假设导致 **stow 无法处理动态结构**，即根据条件选择性映射文件或是在映射过程中改变文件名/路径

例如在配置 ghostty 时，因为 stow 只能链接完整的路径结构，如果要做到不同系统之间采用不同配置只能同时维护两套配置，分别对应不同的系统，无法利用 ghostty 的加载指定文件功能

## 解决方案

设计一个支持选择性映射文件的 CLI symlink manager 用于管理 dotfile，基于 Stow 的设计模式但在映射部分做修改。
支持 MacOS，Linux 双系统，使用 Python 实现全部功能

目前设想通过识别特定后缀实现选择性映射，suffix 初定为：
- MacOS -> "macos"
- Linux -> "linux"

## 最小可用功能集

**1. CLI 命名规范**

> 初定 CLI 调用名为 `stpy`

- 所有命令统一遵循 `stpy <command> [<package>]` 语法：
  - 不指定 `<package>`：对根目录下的所有包进行批处理。
  - 指定 `<package>`：仅对目标包进行局部扫描或操作。

**2. 命令功能明细**

`stpy status [<package>]`
  - 功能：非破坏性的只读检查，用于在执行任何磁盘写入前预览。
  - 执行逻辑：
    - 探测当前系统环境（MacOS / Linux）。递归扫描包内的所有文件，匹配后缀规则，过滤并计算出目标绝对路径。
    检查目标路径在系统中的现状，归类为以下四种状态：
      - 🟢 LINKED：已成功建立软链接且指向当前仓库源文件。
      - 🟡 ABSENT：目标位置为空，准备新建链接。
      - 🟠 WRONG_LINK：目标位置是软链接，但指向别处或属于悬空死链接。
      - 🔴 CONFLICT：目标位置已存在真实文件/目录（非软链接）。
    - 按树状层级在终端格式化输出结果。

`stpy link [<package>]`
  - 功能：根据当前系统环境，将仓库中的配置文件映射至 Home 目录。
  - 执行逻辑：
    - 内部自动先跑一遍 status 诊断。对不同状态采取相应处理：
      - 🟢 LINKED -> Skip，不做重复动作。
      - 🟡 ABSENT -> 自动创建缺失的父级目录（mkdir -p），创建软链接。
      - 🟠 WRONG_LINK -> 自动纠偏：解绑旧/坏软链接，重新绑定至当前仓库源文件，并打印 Log。
      - 🔴 CONFLICT -> 跳过该冲突文件，并在终端打印报警。整个过程绝不盲目覆盖或删除任何真实文件。

`stpy unlink [<package>]`
  - 功能：卸载指定的软链接映射，恢复系统底层状态。
  - 执行逻辑：
    - 内部自动跑一遍 status 诊断，只删除标记为 🟢 LINKED 的 symlink
    - 若删除 symlink 以后当文件夹为空，则删除这个文件夹，直到到达非空目录或 ~ 为止
