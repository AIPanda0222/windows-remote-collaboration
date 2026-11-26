# 一  Muse 与笔记本和台式机互联

先完成 A1，让笔记本和台式机加入同一专网。只想传文件的读者可完成 A1 后直接跳到[双向文件同步](02-syncthing.md)。

## A1 两台 Windows 加入同一个专网

先把两台电脑叫作“笔记本”和“台式机”。Tailscale 给它们各分配一个专网地址。之后输入地址时，使用你自己刚查到的数字。

**1.** 【笔记本】打开文末的 Tailscale 安装指南，下载安装 Windows 客户端。点击任务栏通知区的 Tailscale 图标并登录，按浏览器提示加入你的专网。

**2.** 【台式机】重复安装和登录，选择同一个专网。若两个账号不同，由专网管理员邀请并批准设备。

**3.** 【两台 Windows】分别右键开始菜单，打开“终端”或“Windows PowerShell”。输入下面一行后按回车，把屏幕显示的 IPv4 分别记在自己的笔记里。

```powershell
tailscale ip -4
```

| 要记下的值 | 后续示例中的代称 |
| --- | --- |
| 笔记本的 Tailscale IPv4 | LAPTOP_TS_IP |
| 台式机的 Tailscale IPv4 | DESKTOP_TS_IP |

**4.** 【笔记本】把下行里的 DESKTOP_TS_IP 换成台式机地址再执行。接着在台式机反过来测，把命令末尾换成笔记本地址。

```powershell
tailscale ping DESKTOP_TS_IP
```

> 两边都收到 pong 后继续。看到 direct 或 relay 都表示这次专网探测收到回复。若一端没有回复，先确认两台电脑在线、已登录并位于同一专网。

[Tailscale Windows 安装指南](https://tailscale.com/docs/install/windows)

## A2 确认 Muse 通过哪里访问 Windows

先在 Muse 所在的执行环境确认它用哪一种接入方式。只使用符合自己情况的一行。

| Muse 的环境 | 需要准备什么 |
| --- | --- |
| 能安装并运行 Tailscale | 让该环境加入同一专网，记下它的 Tailscale IPv4。 |
| 平台提供专网代理 | 向平台确认可用的 SSH ProxyCommand 和 Windows 会看到的专网出口 IPv4。保留平台给出的命令和端口。 |

把这次 Windows 会看到的来源 IPv4 记为 MUSE_SOURCE_IP。两台 Windows 的防火墙要按这个值放行。专网有访问策略时，也要允许该接入端访问两台 Windows 的 TCP 22。

> 拿到来源 IP 再配置防火墙。如果平台尚未给出代理命令或出口地址，先向平台确认，也可以先做第二部分的文件同步。

## A3 安装 Windows 的 SSH 服务端

**1.** 【台式机】在机器本地登录。开始菜单搜索“可选功能”，打开设置结果。先搜索已安装的功能，查看“OpenSSH 服务器”是否在列表里。

**2.** 如果没有，点击“查看功能”或“添加可选功能”，搜索 OpenSSH，选中“OpenSSH 服务器”，按提示安装。

**3.** 【笔记本】重复前两步。安装完成后，在两台机的已安装列表中都能查到 OpenSSH 服务器。 若后面找不到 ssh-keygen，可用同样方法安装 OpenSSH 客户端。

下一页会先处理防火墙。安装可能自动创建一条允许访问 TCP 22 的入站规则。请在电脑本地操作，以便遇到配置问题时能恢复。

[微软 Windows 11 OpenSSH 安装步骤](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse?pivots=windows-11)

## A4 把 SSH 入站规则限定到 Muse

【先在台式机做，再在笔记本重复】本地地址填正在操作的 Windows 专网 IP。远程地址填 A2 记录的 Muse 来源 IP。新部署从本地操作，已有业务规则时先记录原设置。

**1.** 按 Win + R，输入 wf.msc 并回车。左侧选“入站规则”。寻找 OpenSSH 服务器规则，打开“属性”，在“协议和端口”核对协议 TCP、本地端口 22。

**2.** 切到“作用域”。上半区“本地 IP 地址”选“这些 IP 地址”，点击“添加”，填这台电脑自己的专网 IP。下半区“远程 IP 地址”同样选“这些 IP 地址”，添加 Muse 来源 IP。

**3.** 在“常规”确认操作为“允许连接”，在“高级”选中适用的网络配置文件。点“应用”。重新打开“作用域”，核对两处 IP。

**4.** 若没有现成规则，在“入站规则”右侧点“新建规则”。规则类型选“自定义”，程序选“所有程序”。按下表走完向导，并命名为“实验 Muse SSH”。

| 向导页面 | 填写内容 |
| --- | --- |
| 协议和端口 | TCP。本地端口选“特定端口”并填 22。远程端口保持任意。 |
| 作用域 | 本地 IP 填这台 Windows 的专网地址。远程 IP 填 Muse 来源。 |
| 操作和配置文件 | 选“允许连接”，勾选实验所需的配置文件，再完成命名。 |

检查防火墙已开启、默认入站阻止，并停用确认属于本实验的宽放行规则。若旧的显式阻止规则也匹配 Muse，先核对并缩小它的阻止范围。显式阻止优先于冲突的允许规则。

> 准备首次本地初始化时，先让这条 SSH 放行规则保持禁用，并确认 TCP 22 暂无其他放行。等 A6 设置好公钥认证，再启用这条规则。

[微软防火墙自定义规则步骤](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/configure)

[微软防火墙规则优先级](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/rules)

## A5 给 Muse 准备公钥并在 Windows 授权

**1.** 【Muse】若平台管理 SSH 密钥，使用平台提供的公钥。若 Muse 有 Linux 终端，可执行以下命令生成实验密钥。按提示设置口令，若选了不同的文件名，后续查看公钥和 A7 的 IdentityFile 也要换成对应路径。

```bash
mkdir -p ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/windows_lab_ed25519
cat ~/.ssh/windows_lab_ed25519.pub
```

最后一行会显示公钥文本。复制从 ssh-ed25519 开始的整行。只在需要授权的两台 Windows 上粘贴该公钥。私钥和口令留在 Muse 一侧。

**2.** 【台式机】用将来允许 Muse 登录的普通 Windows 账号在本机登录一次。打开 PowerShell，执行 whoami，记下显示的本机账号名。若显示“机器名\用户名”，把用户名部分记下来。

**3.** 仍在该账号的 PowerShell 中逐行执行下方命令，准备 .ssh 文件夹并打开授权文件。

```powershell
$keyDir = Join-Path $env:USERPROFILE '.ssh'
New-Item -ItemType Directory -Path $keyDir -Force | Out-Null
notepad (Join-Path $keyDir 'authorized_keys')
```

**4.** 在记事本里把 Muse 公钥粘贴为独立一行，保留原有行。若需要“另存为”，保存类型选“所有文件”，文件名填 authorized_keys，编码用 UTF-8。

**5.** 在资源管理器打开“查看 → 显示 → 文件扩展名”。确认保存的是该账号主目录下的 .ssh\authorized_keys，文件名末尾没有 .txt。然后在笔记本对目标账号重复授权。

> 用管理员账号远程登录时，Windows 默认读取 %ProgramData%\ssh\administrators_authorized_keys。按微软指南设置此文件的权限。入门实验优先使用普通账号。

[微软 Windows OpenSSH 公钥授权指南](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement)

## A6 启用公钥登录并取得两台主机的指纹

**1.** 【台式机本地】按 Win + R，输入 services.msc。找到 OpenSSH SSH Server。首次没有配置文件时，保持 TCP 22 入站阻止，点“启动”生成默认配置和主机密钥，随后点“停止”。

**2.** 用资源管理器打开 %ProgramData%\ssh，把 sshd_config 复制一份备份。以管理员身份运行记事本，用“文件 → 打开”，文件类型选“所有文件”，再打开原 sshd_config。

**3.** 在第一个 Match 段之前，检查下列全局选项，替换同名设置或补上缺少的行。把 WINDOWS_USER 换成 A5 记下的本机用户名，用小写。保存文件。

```sshconfig
PubkeyAuthentication yes
PasswordAuthentication no
AuthenticationMethods publickey
AllowUsers WINDOWS_USER
```

**4.** 打开管理员 PowerShell，先执行配置检查。成功时通常没有输出。若显示某行配置错误，修正后再检查。

```powershell
& "$env:WINDIR\System32\OpenSSH\sshd.exe" -t
```

**5.** 回到 wf.msc，启用 A4 已核对范围的规则。回到服务窗口，启动或重新启动 OpenSSH SSH Server，确认状态为“正在运行”。

**6.** 在管理员 PowerShell 取得本机主机指纹，把以 SHA256: 开头的整段记到自己的笔记里。换到笔记本重复本页，分别记下两台电脑的指纹。

```powershell
$hostPub = "$env:ProgramData\ssh\ssh_host_ed25519_key.pub"
ssh-keygen -lf $hostPub -E sha256
```

> A5 的用户公钥供 Windows 核对 Muse 的身份。这里的主机指纹供 Muse 核对自己连接的是哪台 Windows。

[微软 OpenSSH 服务端配置说明](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)

## A7 在 Muse 填好两台电脑的连接信息

**1.** 【Muse】用执行环境的文件编辑功能，在自己的主目录下新建 windows-lab-ssh.conf。粘贴下面的示例。lab-desktop 和 lab-laptop 是你给两个连接起的简称。

```sshconfig
Host lab-desktop
    HostName DESKTOP_TS_IP
    User DESKTOP_USER
Host lab-laptop
    HostName LAPTOP_TS_IP
    User LAPTOP_USER
Host lab-desktop lab-laptop
    Port 22
    IdentityFile ~/.ssh/windows_lab_ed25519
    IdentitiesOnly yes
    HostKeyAlgorithms ssh-ed25519
    StrictHostKeyChecking ask
    # 需要平台代理时，在这里写入平台给出的 ProxyCommand
```

**2.** 把两个 HostName 分别换成 A1 记录的地址。把两个 User 分别换成 A5 授权的账号。若使用平台托管的身份，按平台说明选中对应密钥或连接入口。

**3.** 如果 Muse 原生加入了 Tailscale，保留示例注释即可。如果 Muse 依赖平台代理，把注释行替换成完整的 ProxyCommand。保存文件。

**4.** 在 Muse 的终端运行下面的命令，核对输出中的 hostname、user、port 和 proxycommand。随后把末尾换成 lab-laptop 再查一次。

```bash
ssh -G -F ~/windows-lab-ssh.conf lab-desktop
```

> 准备直接访问台式机时，这份测试配置省略 ProxyJump。平台接入专网所需的 ProxyCommand 继续保留。原来的笔记本跳板入口可以单独保留。

[OpenSSH 客户端代理配置](https://man.openbsd.org/ssh_config)

## A8 从 Muse 登录并确认连对了电脑

**1.** 【Muse】先连台式机。下面的 -S none 会建立新会话。第一次提示主机指纹时，与 A6 从台式机本地取得的指纹核对。只有一致时才确认。

```bash
ssh -S none -F ~/windows-lab-ssh.conf lab-desktop
```

**2.** 登录后，逐行执行下面的命令。第一行应显示台式机名，第二行应显示授权的 Windows 账号，第三行的第一个地址应与 A2 的 Muse 来源一致。

```text
hostname
whoami
cmd /d /c set SSH_CONNECTION
exit
```

**3.** 回到 Muse，把登录命令的末尾改成 lab-laptop，按相同步骤核对笔记本。两台机器各成功一次，才算完成双目标验收。

**4.** 若要确认台式机路径独立于笔记本，等笔记本方便离线时，从 Muse 重新发起一次台式机连接。再次核对 hostname 和连接来源。

| 看到什么 | 先查哪里 |
| --- | --- |
| 一直等待或超时 | 查专网与代理，然后在目标机确认 SSH 服务正在运行、TCP 22 规则允许 Muse 来源。 |
| Permission denied (publickey) | 查 User、Muse 选择的密钥、目标授权文件及文件权限。 |
| 主机指纹不同 | 先取消连接，回到目标机本地重新核对地址和指纹。 |

要停止实验，在本地禁用自己创建的放行规则。需要恢复原配置时，用 A6 的备份替换实验配置，运行 sshd -t 通过后再重启服务。

[OpenSSH 连接环境变量说明](https://man.openbsd.org/ssh)

## 两台 Windows 之间也需要 SSH 时

在每台电脑上按 A3–A6 配置接收连接的一侧，再从另一台测试。把发起连接的电脑的 Tailscale IP 加入目标机 TCP 22 规则的远程地址，并把该电脑的 SSH 公钥加到目标账号的授权文件。两边各自保管私钥。新建会话时按 A6 核对目标主机指纹，用 A8 的 hostname 和 whoami 确认目标机和账号。只做 B1–B5 文件同步时，可以跳过此项。
