---
name: remote-machine
description: 需要操作另一台 Windows 机器时读这份 —— 部署代码过去、看它的日志、在它上面跑命令或跑要密钥的作业、排查「连不上」。也适用于任何「我在这台改、那台跑」的双机分工。给出正确的 SSH 姿势、两个必踩的陷阱（SSH 会话读不到凭据库、部署≠换文件），以及连不上时该往哪查、不该往哪查。
---

# 操作另一台 Windows

**用工具，别手敲 ssh**：

```bash
python D:/dev/remote-machine/remote.py check     # 连得上吗、是谁、代码到哪一版、任务活着没
```

| 想干什么 | 命令 |
|---|---|
| 看状态 | `remote.py check` |
| 跑一条只读命令 | `remote.py run "<PowerShell>"` |
| 看日志尾部 | `remote.py logs [名字] [行数]` |
| 上线（拉代码 + 重启服务） | `remote.py deploy` |
| 触发一个已注册的计划任务 | `remote.py task <名字>` |
| 传个脚本过去 | `remote.py push <本地文件> <远端相对路径>` |
| **跑要密钥/要花钱的作业** | `remote.py job "<PowerShell>"`（先 `job --install` 一次） |
| 操作另一台（不是默认那台） | 任何命令加 `--machine <名字>` |

机器配置在 `~/.remote-machine/machines.toml`。**在哪个项目目录里跑，就自动选中那台机器**
（按配置里的 `local` 匹配），所以平时不用加 `--machine`。

工具把三个用血换来的细节包好了：**用户名是账号不是计算机名**、
**中文要套 UTF-8 外壳**、**连不上时把「该往哪查、不该往哪查」直接打出来**。

## ⚠ 陷阱 1：SSH 会话里读不到 Windows 凭据库 —— 要密钥的作业必须走 `job`

**`run` 走的是网络登录会话**，凭据管理器在这种会话里整个打不开
（真话是 `CredRead: 指定的登录会话不存在`，常被上层的静默降级盖住）。
所以从 `run` 发起的、任何要读密钥的作业**会在第一次用到密钥时才废** ——
跑到一半才废，前面花掉的时间和额度全白费。

**`job` 通道没有这个限制**（已实测）：它触发一个 `LogonType=Interactive` 的计划任务，
那个会话跑在 `SessionId=1`，凭据库正常可读。

```bash
python D:/dev/remote-machine/remote.py job --install        # 每台机器只做一次
python D:/dev/remote-machine/remote.py job "<PowerShell>"   # 对面用自己的身份跑
python D:/dev/remote-machine/remote.py job "<...>" --async  # 不等它；之后 --tail 看进展
```

| 要干的事 | 走哪条 |
|---|---|
| 拉代码 / 装包 / 跑测试 / 离线检查 / 读日志读数据 | `run`（更快，一条 ssh 就完） |
| 任何要密钥的、调付费 API 的、写外部账号的 | **`job`** |

⚠ 这条通道**一次只跑一个作业**。长作业一律 `--async` 发（连不上是常态，不是异常）。
⚠ 花钱的大批量作业**先问用户** —— 那是钱的量级问题，不是能力问题。

## ⚠ 陷阱 2：`git pull` 之后，对面跑的还是旧代码

**部署不等于把文件换掉。** 常驻进程（服务、面板、守护任务）在磁盘代码被换掉之后
**照跑旧的，而且没有任何迹象表明它是旧的**。真咬过一次：更新完打开网页面板，
新加的设置项根本不显示 —— 老进程占着端口跑着旧代码，新起的进程绑不上端口直接死了，
「关掉窗口再打开」也没用。

所以：**用 `remote.py deploy`**（它跑项目自己的上线脚本，只有那个脚本知道该重启谁），
别自己拼 `git pull`。手工停进程的话：

```powershell
$p=(Get-NetTCPConnection -LocalPort <端口> -State Listen).OwningProcess; Stop-Process -Id $p -Force
```

计划任务托管的服务用 `Stop-ScheduledTask` + `Start-ScheduledTask`，
但注意**孙子进程**未必跟着换掉，重启后核对一次日志里的启动时间。

## 「连不上」怎么查（这是最费时间的一类）

先跑一次 `remote.py check`，它会按错误码给出该查的方向。几条硬判据：

1. **`ping` 不通什么都不能证明** —— 很多机器的防火墙不回 ICMP。实测过：
   ping 收 0 个回包的同一秒，SSH 连得好好的。要探活只能用 22 端口本身。
2. **`Permission denied (publickey,...)` 对「账号不存在」和「公钥不对」是同一句**，
   在这边分不出来。决定性证据只在对面的 sshd 日志里：
   `Get-WinEvent -LogName OpenSSH/Operational -MaxEvents 12`；写着 `Invalid user` = 账号错。
3. **`Connection closed by ...`（sshd 还没打招呼）跟账号密钥全无关**：
   要么对面在睡，要么**本机的代理/VPN 接管了这个连接**。
   分辨判据：连一个**确定不存在**的 IP 的 22 端口，它要是也「连上」了，就是代理。
4. **超时 ≠ 关机。** 断的往往是**路**不是机器：一台多网卡的机器，
   **不同的路通往不同的它** —— 无线断了但有线还活着时，它照常干活，只是你走的那条路没了。
   要断言它睡了，拿证据：`Get-WinEvent -ProviderName Microsoft-Windows-Kernel-Power`，
   没有电源事件就说明它压根没睡过。
5. **唤醒包（`wake`）不是可靠退路** —— 实测发出去了对面没醒（无线网卡睡眠后基本不响应）。
   该常开的机器就别让它睡：`powercfg /change standby-timeout-ac 0`。

## 编码：中文会在四个地方分别烂掉

顶上那层 UTF-8 外壳只管**输出**，管不到另外三处。工具内部都处理好了，
自己写命令时要记得：

| 在哪 | 怎么治 |
|---|---|
| 控制台输出 | `$OutputEncoding` / `[Console]::OutputEncoding` 设成 UTF8 |
| Python 子进程 | `$env:PYTHONIOENCODING='utf-8'` |
| **读文件**（`Get-Content`） | 加 `-Encoding utf8`，少了就是 `[蹇冭烦]` 这种乱码 |
| **读 .ps1 脚本自身** | 文件必须**带 BOM**，否则中文在执行前就烂了 |

复杂脚本**别硬拼引号**（要穿过 ssh → PowerShell 两层解析）：`push` 传过去再执行，
跑完删掉临时文件。

## 给不懂编程的人用时

凡是要他在那台机器跟前做的事，做成**双击就能跑的 .bat**或界面上的一个按钮，
别给他要敲的命令。
