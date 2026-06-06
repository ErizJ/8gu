# Linux 面试知识体系（后端开发校招终极版 · Golang 方向）

> 适用于 Golang 后端开发校招
>
> 学习建议：
>
> - ★★★★★：必会，高频考点
> - ★★★★：常考
> - ★★★：了解即可
>
> 本目录覆盖腾讯、阿里、字节、美团、京东、网易、百度、携程、唯品会等大厂后端开发校招 Linux 面试题。

---

# 目录

- [第一章 Linux 基础](#第一章-linux-基础)
  - [1.1 Linux 操作系统简介 ★★★](#11-linux-操作系统简介-)
  - [1.2 Linux 目录结构 ★★★★](#12-linux-目录结构-)
  - [1.3 文件类型 ★★★★](#13-文件类型-)
    - [inode 与文件存储原理 ★★★★★](#inode-与文件存储原理-)
    - [硬链接 vs 软链接](#硬链接-vs-软链接)
- [第二章 常用命令 ★★★★★](#第二章-常用命令-)
  - [2.1 文件管理命令](#21-文件管理命令)
  - [2.2 文件查看命令 ★★★★★](#22-文件查看命令-)
    - [tail — 文件尾部 ★★★★★](#tail--文件尾部-)
    - [less — 分页查看 ★](#less--分页查看可上下翻推荐)
  - [2.3 文件搜索命令 ★★★★★](#23-文件搜索命令-)
    - [find — 文件搜索 ★★★★★](#find--强大的文件搜索-)
  - [2.4 文本处理命令 ★★★★★](#24-文本处理命令-)
    - [grep ★★★★★](#grep--文本搜索-)
    - [awk ★★★★★](#awk--列处理利器-)
    - [sed ★★★★](#sed--流编辑器-)
    - [sort ★★★★★ / uniq ★★★★★ / wc ★★★★](#sort--排序-)
    - [tee ★★★★ / tr ★★★ / column ★★★](#tee--双向输出-)
    - [jq — JSON 处理 ★★★★★](#jq--json-处理-)
    - [xargs ★★★★★](#xargs--将标准输入转为命令行参数-)
  - [2.5 压缩与解压 ★★★★](#25-压缩与解压-)
    - [tar ★★★★★](#tar--打包解包-)
    - [rsync ★★★★ / scp ★★★★ / md5sum ★★★★](#rsync--增量同步-)
  - [2.6 系统信息与系统管理命令 ★★★★](#26-系统信息与系统管理命令-)
    - [watch ★★★★ / systemd ★★★★★ / crontab ★★★★](#watch--周期性执行命令-)
    - [shutdown / reboot ★★★ / 环境变量](#shutdown--reboot--关机与重启-)
  - [2.7 SSH 基础 ★★★★](#27-ssh-基础-)
- [第三章 用户与权限管理 ★★★★★](#第三章-用户与权限管理-)
  - [3.3 文件权限 ★★★★★ / 3.4 chmod ★★★★★](#33-文件权限-)
  - [ACL（访问控制列表）★★★](#acl访问控制列表更细粒度的权限-)
  - [SUID / SGID / Sticky Bit ★★★★](#特殊权限-suid--sgid--sticky-bit-)
  - [3.5 chown ★★★★ / 3.6 sudo ★★★★](#35-chown--修改所有者-)
- [第四章 进程管理 ★★★★★](#第四章-进程管理-)
  - [进程 vs 线程 vs 协程 ★★★★★（面试必考）](#进程-vs-线程-vs-协程面试必考)
  - [4.2 ps ★★★★★ / pgrep ★★★★](#42-ps-命令-)
  - [4.3 top ★★★★★](#43-top-命令-)
  - [4.6 kill ★★★★★ / nice ★★★ / time ★★★★](#46-kill-命令-)
  - [4.7 后台运行 ★★★★★（nohup / screen / tmux）](#47-后台运行-)
  - [4.8 Signal 信号处理 + Go 优雅关闭 ★★★★★](#48-signal-信号处理--go-优雅关闭-)
  - [4.9 IPC 进程间通信 ★★★★](#49-ipc-进程间通信详解-)
- [第五章 内存管理 ★★★★★](#第五章-内存管理-)
  - [5.0 虚拟内存深度 ★★★★★](#50-虚拟内存深度-)
  - [5.1 free ★★★★★ / 5.2 vmstat ★★★★★](#51-free-命令-)
  - [5.5 OOM ★★★★★ / 内存 overcommit ★★★★](#55-oom-)
- [第六章 磁盘管理 ★★★★](#第六章-磁盘管理-)
  - [6.1 df ★★★★★ / 6.2 du ★★★★★](#61-df-命令-)
  - [6.4 iostat — 磁盘 I/O 分析 ★★★★★](#64-iostat--磁盘-io-分析-)
  - [dd — 磁盘基准测试 ★★★★](#dd--磁盘读写基准测试-)
- [第七章 网络管理 ★★★★★](#第七章-网络管理-)
  - [7.0 TCP 三次握手 & 四次挥手 ★★★★★](#70-tcp-三次握手--四次挥手-)
  - [TCP KeepAlive vs HTTP Keep-Alive](#tcp-keepalive-和-http-keep-alive-的区别)
  - [TCP vs UDP ★★★★★](#tcp-vs-udp-对比-)
  - [7.1 IP 查看与路由管理 ★★★★](#71-ip-查看与路由管理-)
  - [7.2 网络连通性 ★★★★★（ping / curl / nc）](#72-网络连通性-)
  - [7.3 DNS 排查 ★★★★★ / 7.4 端口查看 ★★★★★](#73-dns-排查-)
  - [7.5 lsof ★★★★★ / 7.6 tcpdump ★★★★★](#75-lsof--查看进程打开的文件-)
- [第八章 日志分析 ★★★★★](#第八章-日志分析-)
  - [8.3 awk 日志分析 ★★★★★](#83-awk-日志分析-)
  - [8.4 logrotate — 日志轮转 ★★★★★](#84-logrotate--日志轮转-)
- [第九章 性能分析工具 ★★★★★](#第九章-性能分析工具-)
  - [uptime / vmstat / iostat / sar / pidstat](#91-uptime--系统负载)
  - [上下文切换 — vmstat cs 列深度分析 ★★★★](#上下文切换--vmstat-cs-列深度分析-)
  - [9.6 strace ★★★★★ / 9.7 perf ★★★★★](#96-strace--系统调用跟踪-)
- [第十章 Shell 基础 ★★★★](#第十章-shell-基础-)
- [第十一章 Linux 高并发与服务器开发 ★★★★★](#第十一章-linux-高并发与服务器开发-)
  - [11.1 文件描述符（FD）](#111-文件描述符fd)
  - [11.1.5 五种 I/O 模型 ★★★★★](#1115-五种-io-模型-)
  - [io_uring — 下一代异步 I/O ★★★](#io_uring--下一代异步-io-)
  - [11.4 epoll ★★★★★](#114-epoll--高性能-io-多路复用-)
  - [11.5 Reactor 模型 ★★★★★](#115-reactor-模型-)
  - [TCP_NODELAY ★★★★★ / SO_REUSEADDR ★★★★](#tcp_nodelay--nagle-算法与低延迟-)
  - [11.6 零拷贝 ★★★★★](#116-零拷贝-)
  - [11.8 Nginx 模型 / 11.9 容器底层技术 ★★★★★](#118-nginx-模型)
  - [eBPF 简介 ★★★](#ebpf-简介-)
  - [11.10 高并发内核参数调优 ★★★★★](#1110-高并发内核参数调优-)
- [第十二章 Linux 故障排查 ★★★★★](#第十二章-linux-故障排查-)
  - [场景1: CPU 100% / 场景2: 内存泄漏 / 场景3: 磁盘满](#场景1cpu-100-如何排查)
  - [场景4: 服务器变慢 / 场景5: 端口无法访问 / 场景6: 接口超时](#场景4服务器变慢如何排查)
  - [场景7: 连接数暴涨 / 场景8: DNS 解析失败](#场景7连接数暴涨如何排查)
- [第十三章 Linux 面试高频 Top50 ★★★★★](#第十三章-linux-面试高频-top50-)
- [附录：Go 开发必知必会 Linux 命令速查表](#附录go-开发必知必会-linux-命令速查表)
  - [Go httptrace — HTTP 请求耗时分析](#go-httptracehttp-请求耗时分析)

---

<a id="第一章-linux-基础"></a>
# 第一章 Linux 基础

## 1.1 Linux 操作系统简介 ★★★

### Linux 特点

- **开源**：源代码公开，可自由修改和分发
- **多用户**：多个用户可以同时登录使用
- **多任务**：可同时运行多个进程
- **高稳定性**：服务器可连续运行数年不重启
- **高性能**：内存管理、文件系统、网络协议栈都经过极致优化

### 常见 Linux 发行版

| 发行版 | 包管理器 | 适用场景 |
|--------|---------|---------|
| Ubuntu | apt | 开发环境、个人服务器 |
| CentOS | yum/dnf | 企业服务器（已停止维护） |
| Rocky Linux | dnf | CentOS 替代品 |
| Debian | apt | 稳定版服务器 |
| Alpine | apk | Docker 镜像（超小体积） |

### 面试题

**Q：为什么服务端普遍使用 Linux？**

> ① 开源免费，无版权风险；② 稳定性极高，可长时间运行不重启；③ 性能优秀，内核经过大量优化；④ 安全性好，权限管理严格；⑤ 强大的命令行和脚本能力，自动化运维方便；⑥ 丰富的开发工具链和社区支持。

**Q：Linux 和 Windows 的区别？**

| 对比项 | Linux | Windows |
|--------|-------|---------|
| 源码 | 开源 | 闭源 |
| 用户界面 | 命令行为主 | GUI 为主 |
| 文件系统 | ext4/xfs（单一树） | NTFS（多盘符） |
| 路径分隔符 | `/` | `\` |
| 换行符 | `\n` | `\r\n` |
| 权限模型 | rwx 三位 | ACL |
| 适用场景 | 服务器/嵌入式 | 桌面/企业办公 |

---

## 1.2 Linux 目录结构 ★★★★

### 根目录 `/`

Linux 只有一个根目录 `/`，所有文件都挂载在这棵树下：

```
/
├── bin/      # 基本命令（ls、cp、mv），所有用户可用
├── sbin/     # 系统管理命令（fdisk、iptables），需要 root 权限
├── etc/      # 配置文件（nginx.conf、hosts、systemd）
├── home/     # 普通用户主目录
├── root/     # root 用户主目录
├── var/      # 可变数据（日志文件 /var/log、数据库文件）
├── tmp/      # 临时文件（重启后可能被清空）
├── proc/     # 虚拟文件系统，映射内核和进程状态（不占磁盘）
├── dev/      # 设备文件（磁盘、终端、随机数）
├── usr/      # 用户程序和数据（/usr/bin、/usr/lib）
├── lib/      # 共享库（.so 文件）
├── boot/     # 内核和引导文件
├── opt/      # 第三方软件安装目录
└── mnt/      # 临时挂载点
```

### 面试题

**Q：/proc 目录是什么？**

> `/proc` 是虚拟文件系统，不占用磁盘空间，是内核暴露给用户空间的接口。每个进程在 `/proc/<PID>/` 下有对应的目录，包含进程的详细信息。常用：
>
> ```bash
> cat /proc/cpuinfo          # CPU 信息（型号、核心数）
> cat /proc/meminfo          # 内存详细信息
> cat /proc/loadavg          # 系统负载
> cat /proc/1234/status      # PID 1234 的进程状态（内存、线程数）
> cat /proc/1234/limits      # 进程资源限制
> cat /proc/1234/fd/         # 进程打开的文件描述符
> cat /proc/net/tcp          # TCP 连接状态（十六进制）
> cat /proc/sys/fs/file-max  # 系统级最大文件描述符数
> ```

**Q：/etc 目录存放什么？**

> 所有系统配置文件。关键文件：
>
> ```bash
> /etc/passwd         # 用户账号信息
> /etc/shadow         # 用户密码（加密存储）
> /etc/group          # 用户组信息
> /etc/hosts          # 本地 DNS 解析
> /etc/resolv.conf    # DNS 服务器配置
> /etc/fstab          # 文件系统挂载表
> /etc/sysctl.conf    # 内核参数配置
> /etc/systemd/       # systemd 服务配置
> /etc/nginx/         # Nginx 配置
> ```

---

## 1.3 文件类型 ★★★★

Linux 中一切皆文件。`ls -l` 第一个字符标识文件类型：

| 字符 | 类型 | 说明 |
|------|------|------|
| `-` | 普通文件 | 文本、二进制、图片等 |
| `d` | 目录文件 | 文件夹 |
| `l` | 符号链接 | 软链接（快捷方式） |
| `b` | 块设备 | 磁盘、U盘（随机访问） |
| `c` | 字符设备 | 键盘、鼠标、终端（流式访问） |
| `s` | Socket | 进程间通信 |
| `p` | 管道 | 命名管道（FIFO） |

### inode 与文件存储原理 ★★★★★

Linux 文件系统中，文件由 **inode（索引节点）** 和 **数据块（block）** 两部分组成：

```
inode（元数据）                   数据块（内容）
┌─────────────────┐              ┌──────────────┐
│ 文件类型/权限    │         ┌───▶│ 实际文件内容   │
│ 所有者/组       │         │    └──────────────┘
│ 文件大小        │         │
│ 时间戳          │         │
│ 数据块指针 ─────┼─────────┘
│ 引用计数        │
└─────────────────┘

目录文件内容：文件名 → inode 号的映射表
```

**核心结论**：inode 不存储文件名，文件名只存在于目录中。删除文件 = 删除目录中的映射 + inode 引用计数减 1，计数归零才真正释放数据块。

```bash
# 查看 inode 信息
stat file.txt
# 输出：
#   File: file.txt
#   Size: 1234       Blocks: 8          IO Block: 4096   regular file
#   Device: 801h/2049d   Inode: 123456  Links: 1
#   Access: (0644/-rw-r--r--)  Uid: (1000/user)  Gid: (1000/user)
#   Access: 2025-01-01 10:00:00   # atime 最后访问时间
#   Modify: 2025-01-01 09:00:00   # mtime 最后修改时间
#   Change: 2025-01-01 09:00:00   # ctime 最后状态改变时间

ls -i file.txt      # 查看 inode 号
df -i               # 查看 inode 使用情况（inode 耗尽也无法创建文件）
```

### 硬链接 vs 软链接

```bash
# 硬链接：两个文件名指向同一个 inode
ln original.txt hardlink.txt
# 特点：删除原文件，硬链接仍可访问（引用计数 > 0）
#       不能跨文件系统，不能链接目录

# 软链接（符号链接）：类似快捷方式，存储目标路径
ln -s original.txt softlink.txt
# 特点：删除原文件，软链接失效（悬空链接）
#       可跨文件系统，可链接目录
```

> **面试答法**：inode 是文件元数据结构，存储权限、大小、数据块地址等，不含文件名。硬链接是同一 inode 的多个目录项，引用计数归零才删除数据；软链接是存储目标路径的特殊文件，原文件删除后失效。

---

<a id="第二章-常用命令"></a>
# 第二章 常用命令 ★★★★★

<a id="21-文件管理命令"></a>
## 2.1 文件管理命令

### pwd — 显示当前目录

```bash
pwd                     # /home/user/project
pwd -P                  # 显示物理路径（解析软链接）
```

### ls — 列出目录内容

```bash
ls                      # 列出文件名
ls -l                   # 长格式：权限、大小、时间
ls -a                   # 显示隐藏文件（.开头）
ls -lh                  # 人性化大小（1K、234M、2G）
ls -lt                  # 按修改时间排序（最新的在前）
ls -ltr                 # 按修改时间反序（最旧的在前面）
ls -lS                  # 按文件大小排序
ls -R                   # 递归列出子目录
ls -i                   # 显示 inode 号
ls -d */                # 只列出目录
```

### cd — 切换目录

```bash
cd /path/to/dir         # 绝对路径
cd ../                  # 上级目录
cd -                    # 回到上一次的目录
cd ~                    # 回到 home 目录
cd                      # 同上
```

### mkdir — 创建目录

```bash
mkdir newdir            # 创建目录
mkdir -p a/b/c/d        # 递归创建多级目录（必备参数）
```

### touch — 创建空文件/更新文件时间戳

```bash
touch newfile.txt       # 创建空文件
touch -a file.txt       # 只更新访问时间
touch -m file.txt       # 只更新修改时间
touch -t 202501011200 file.txt  # 设置为指定时间
```

### cp — 复制

```bash
cp source.txt dest.txt              # 复制文件
cp -r source_dir dest_dir           # 递归复制目录
cp -p                                # 保留权限和时间戳
cp -a source_dir dest_dir           # 归档复制（保留所有属性，最常用）
cp -v                                # 显示过程
cp -i                                # 覆盖前确认
```

### mv — 移动/重命名

```bash
mv oldname newname      # 重命名
mv file.txt /path/to/   # 移动文件
mv -i                   # 覆盖前确认
mv -v                   # 显示过程
```

### rm — 删除

```bash
rm file.txt             # 删除文件
rm -r dir/              # 递归删除目录
rm -f file.txt          # 强制删除，不提示
rm -rf dir/             # 强制递归删除（极度危险！）
rm -i                   # 删除前确认
rm -v                   # 显示过程
```

### tree — 树状显示目录

```bash
tree                    # 树状显示
tree -L 2               # 只显示 2 层
tree -d                 # 只显示目录
tree -a                 # 显示隐藏文件
```

### 面试题

**Q：rm -rf 含义是什么？**

> `rm` 删除，`-r` 递归删除（recursive），`-f` 强制删除不提示（force）。组合起来会直接递归删除目录下所有内容，没有任何确认提示。极度危险，生产环境使用前一定要再三确认路径。
>
> **安全建议**：
> ```bash
> # 生产环境中先 ls 确认路径
> ls -la /target/path/
> # 再用更安全的方式
> rm -ri /target/path/   # -i 每次确认
> # 或者在脚本中使用 --preserve-root（防止 rm -rf /）
> ```

---

## 2.2 文件查看命令 ★★★★★

### cat — 查看文件全部内容

```bash
cat file.txt                    # 查看文件
cat -n file.txt                 # 带行号
cat -b file.txt                 # 只对非空行编号
cat file1.txt file2.txt         # 合并查看多个文件
cat > file.txt                  # 从标准输入写入文件（Ctrl+D 结束）

# 实际场景：快速查看配置文件
cat /etc/nginx/nginx.conf | grep -v "^#" | grep -v "^$"  # 过滤注释和空行
```

### more — 分页查看（只能向下翻）

```bash
more large_file.log
# 操作：空格翻页，回车翻一行，q 退出
```

### less — 分页查看（可上下翻，推荐）★

```bash
less large_file.log
# 操作：
#   空格/b 翻页，上下箭头/回车翻行
#   g 跳到开头，G 跳到末尾
#   /keyword 向下搜索，?keyword 向上搜索
#   n 下一个匹配，N 上一个匹配
#   q 退出
#   -N 显示行号
#   -S 不折行显示（长行截断）
#   &pattern 只显示匹配的行

# 实际场景：查看大日志文件
less -N /var/log/app.log
# 输入 /ERROR 搜索错误
# 输入 &ERROR 只显示含 ERROR 的行
```

### head — 查看文件头部

```bash
head file.txt                   # 默认前 10 行
head -n 20 file.txt             # 前 20 行
head -c 100 file.txt            # 前 100 个字节

# 实际场景：查看日志最新格式
head -n 3 app.log               # 看日志格式
```

### tail — 查看文件尾部 ★★★★★

```bash
tail file.txt                   # 默认后 10 行
tail -n 50 file.txt             # 后 50 行
tail -c 100 file.txt            # 后 100 个字节

# 实时查看（最重要的用法）
tail -f app.log                 # 实时追踪文件新增内容（Ctrl+C 退出）
tail -f /var/log/nginx/access.log /var/log/nginx/error.log  # 同时追踪多个文件
tail -F app.log                 # 文件被删除重建后继续追踪（比 -f 更健壮）

# 实际场景：查看日志中特定内容并实时追踪
tail -f app.log | grep "ERROR"
```

### tac — 反向输出（从最后一行开始）

```bash
tac file.txt            # 最后一行变第一行

# 实际场景：反向查看日志（最近的日志在最上面）
tac app.log | less
```

### 面试题

**Q：如何实时查看日志？**

```bash
tail -f app.log                 # 实时追踪
tail -f app.log | grep ERROR    # 实时过滤
tail -n 100 -f app.log          # 先显示最近100行，再实时追踪
```

---

## 2.3 文件搜索命令 ★★★★★

### find — 强大的文件搜索 ★★★★★

```bash
# ====== 按名称搜索 ======
find . -name "*.log"                    # 在当前目录下搜索 .log 文件
find /var/log -name "access*"           # 通配符匹配
find . -iname "README"                  # 忽略大小写

# ====== 按类型搜索 ======
find . -type f                          # 只找普通文件
find . -type d                          # 只找目录
find . -type l                          # 只找符号链接

# ====== 按大小搜索 ======
find . -size +100M                      # 大于 100MB
find . -size -10k                       # 小于 10KB
find . -size +1G -size -5G             # 1G~5G 之间的文件

# ====== 按时间搜索 ======
find . -mtime -7                        # 7 天内修改的文件
find . -mtime +30                       # 30 天前修改的文件
find . -atime -1                        # 1 天内访问的文件
find . -mmin -60                        # 60 分钟内修改的文件

# ====== 按权限搜索 ======
find . -perm 777                        # 权限为 777 的文件
find . -perm /u+x                       # 所有者有执行权限的文件

# ====== 按用户搜索 ======
find . -user nginx                      # nginx 用户的文件
find . -group docker                    # docker 组的文件

# ====== 组合条件 ======
find . -name "*.log" -size +100M        # 大于 100MB 的 log 文件
find . -name "*.tmp" -mtime +7          # 7 天前的临时文件
find . -type f -empty                   # 空文件

# ====== 对搜索结果执行命令 ======
find . -name "*.tmp" -delete            # 删除所有 .tmp 文件
find . -name "*.log" -exec ls -lh {} \;    # 对每个文件执行 ls -lh
find . -name "*.log" -exec grep "ERROR" {} \;  # 在每个 log 文件中搜索 ERROR
find . -mtime +30 -exec rm {} \;        # 删除 30 天前的文件

# ====== 实际场景 ======
# 场景1：磁盘满了，找大文件
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null

# 场景2：清理 30 天前的日志
find /var/log -name "*.log" -mtime +30 -delete

# 场景3：查找所有 Go 文件中的 TODO
find . -name "*.go" -exec grep -n "TODO" {} \;

# 场景4：批量修改文件权限
find . -type f -name "*.sh" -exec chmod +x {} \;

# 场景5：删除空目录
find . -type d -empty -delete
```

### locate — 快速文件搜索（基于数据库）

```bash
locate nginx.conf       # 极快，但不实时
updatedb                # 更新数据库（需要 root）

# 区别：locate 查的是数据库缓存，find 实时扫描磁盘
# locate 快但不实时，find 慢但准确
```

### which — 查找可执行命令的位置

```bash
which go                # /usr/local/go/bin/go
which python3           # /usr/bin/python3
which ls                # /usr/bin/ls
```

### whereis — 查找命令的二进制、源码和 man 手册

```bash
whereis go              # go: /usr/local/go/bin/go /usr/local/go/bin/go1.21
whereis ls              # ls: /usr/bin/ls /usr/share/man/man1/ls.1.gz
```

### 面试题

**Q：find 和 locate 区别？**

> `find` 实时遍历文件系统，准确但慢；`locate` 查询预先建立的数据库（`/var/lib/mlocate/mlocate.db`），极快但不实时（新文件搜不到，直到 `updatedb` 更新数据库）。生产环境找文件用 `find`，日常快速定位用 `locate`。

---

## 2.4 文本处理命令 ★★★★★

### grep — 文本搜索 ★★★★★

```bash
# ====== 基础用法 ======
grep "ERROR" app.log                            # 搜索 ERROR
grep -i "error" app.log                         # 忽略大小写
grep -v "DEBUG" app.log                         # 排除含 DEBUG 的行
grep -n "panic" app.log                         # 显示行号
grep -c "ERROR" app.log                         # 统计匹配行数
grep -r "TODO" .                                # 递归搜索当前目录
grep -r --include="*.go" "context" .            # 只搜索 .go 文件
grep -l "FIXME" *.go                            # 只显示文件名

# ====== 上下文 ======
grep -A 5 "ERROR" app.log                       # 匹配行及后面 5 行
grep -B 5 "ERROR" app.log                       # 匹配行及前面 5 行
grep -C 5 "ERROR" app.log                       # 匹配行及前后各 5 行

# ====== 正则表达式 ======
grep -E "ERROR|WARN|FATAL" app.log              # 多关键字（OR）
grep -E "^2025-01-01" app.log                   # 以日期开头
grep -E "ERROR$" app.log                        # 以 ERROR 结尾
grep -E "[0-9]{3}" app.log                      # 3 位数字
grep -P "\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}"   # IPv4 地址（Perl 正则）

# ====== 实际场景 ======
# 场景1：查看最近 1000 条错误日志
tail -n 1000 app.log | grep ERROR

# 场景2：统计各类型错误数量
grep -oE "(ERROR|WARN|INFO)" app.log | sort | uniq -c

# 场景3：搜索 Go 代码中未处理的 error
grep -rn "err !=" . --include="*.go" | grep -v "_ ="

# 场景4：查看配置文件中非注释行
cat /etc/nginx/nginx.conf | grep -v "^#" | grep -v "^$"

# 场景5：搜索并高亮显示
grep --color=auto "ERROR" app.log
```

### awk — 列处理利器 ★★★★★

```bash
# ====== 基础语法 ======
awk '{print $1}' access.log                     # 打印第 1 列
awk '{print $1, $4, $9}' access.log             # 打印第 1,4,9 列
awk -F: '{print $1, $7}' /etc/passwd            # 以 : 为分隔符
awk -F ',' '{print $2}' data.csv                # 处理 CSV
awk '{print NR, $0}' file.txt                   # 打印行号和内容
awk '{print NF}' file.txt                       # 打印每行的列数
awk '{print $NF}' file.txt                      # 打印最后一列

# ====== 条件过滤 ======
awk '$9 == 500' access.log                      # 状态码为 500
awk '$9 >= 400' access.log                      # 状态码 >= 400（所有错误）
awk '$10 > 1000000' access.log                  # 响应大小 > 1MB
awk '$1 ~ /^192\.168/' access.log               # IP 以 192.168 开头
awk 'NR >= 10 && NR <= 20' file.txt             # 第 10-20 行

# ====== 计算 ======
awk '{sum += $10} END {print sum}' access.log   # 流量总和
awk '{sum += $10} END {print sum/NR}' access.log # 平均流量
awk '{ips[$1]++} END {for(ip in ips) print ip, ips[ip]}' access.log  # 统计各 IP 次数

# ====== 实际场景（Nginx 日志分析）=====
# Nginx 日志格式：$remote_addr - [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"

# 场景1：统计 Top10 访问 IP
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# 场景2：统计各状态码数量
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# 场景3：统计 Top10 请求接口
awk '{print $7}' access.log | sort | uniq -c | sort -rn | head -10

# 场景4：找出响应时间 > 1 秒的慢请求
awk '$NF > 1' access.log

# 场景5：统计每小时请求量
awk '{print $4}' access.log | cut -d: -f 2 | sort | uniq -c

# 场景6：统计各接口平均响应时间
awk '{sum[$7]+=$NF; count[$7]++} END {for(url in sum) print sum[url]/count[url], url}' access.log

# 场景7：统计 5xx 错误最多的接口
awk '$9 ~ /^5/' access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head
```

### sed — 流编辑器 ★★★★

```bash
# ====== 替换 ======
sed 's/old/new/' file.txt               # 每行第一个替换
sed 's/old/new/g' file.txt              # 全局替换
sed 's/old/new/gi' file.txt             # 忽略大小写全局替换
sed -i 's/old/new/g' file.txt           # 直接修改文件（重要！）
sed -i.bak 's/old/new/g' file.txt       # 修改前备份为 .bak
sed 's|/usr/local|/opt|g' file.txt      # 使用 | 作为分隔符（路径含 /）

# ====== 删除 ======
sed '/DEBUG/d' app.log                  # 删除含 DEBUG 的行
sed '/^$/d' file.txt                    # 删除空行
sed '/^#/d' file.txt                    # 删除注释行
sed '10,20d' file.txt                   # 删除第 10-20 行

# ====== 打印 ======
sed -n '10,20p' file.txt                # 只打印第 10-20 行
sed -n '/ERROR/p' app.log               # 打印含 ERROR 的行
sed -n '10p' file.txt                   # 只打印第 10 行

# ====== 插入/追加 ======
sed '1i # This is a header' file.txt    # 第 1 行之前插入
sed '$a # This is a footer' file.txt    # 最后一行之后追加

# ====== 实际场景 ======
# 场景1：修改配置文件（替换 IP 地址）
sed -i 's/192.168.1.100/10.0.0.50/g' /etc/app/config.yaml

# 场景2：清理 Nginx 配置（删除空白和注释）
sed '/^$/d; /^[[:space:]]*#/d' nginx.conf

# 场景3：提取日志中某段时间的记录
sed -n '/2025-01-01 10:00/,/2025-01-01 11:00/p' app.log

# 场景4：批量替换 Go 文件中的 import 路径
sed -i 's|github.com/old/repo|github.com/new/repo|g' $(find . -name "*.go")

# 场景5：在匹配行后添加内容
sed -i '/listen 80/a \    server_name example.com;' nginx.conf
```

### sort — 排序 ★★★★★

```bash
sort file.txt                           # 按字典序排序
sort -n file.txt                        # 按数字排序
sort -rn file.txt                       # 按数字倒序
sort -t: -k3 -n /etc/passwd             # 以 : 分隔，按第 3 列数字排序
sort -u file.txt                        # 排序并去重
sort -h file.txt                        # 人性化大小排序（1K < 2M < 1G）
sort --parallel=4 file.txt              # 并行排序

# 实际场景：找出最大的 10 个文件
du -sh * | sort -rh | head -10
```

### uniq — 去重/统计 ★★★★★

```bash
uniq file.txt                           # 去除连续重复行
uniq -c file.txt                        # 统计连续重复次数
uniq -d file.txt                        # 只显示重复的行
uniq -u file.txt                        # 只显示不重复的行

# 注意：uniq 只对连续行去重，通常配合 sort 使用
sort file.txt | uniq -c | sort -rn      # 统计词频排序
```

### cut — 按列切割 ★★★★

```bash
cut -d: -f1,7 /etc/passwd               # 以 : 分隔，取第 1,7 列
cut -c1-10 file.txt                     # 取每行前 10 个字符
cut -d',' -f2- data.csv                 # 以逗号分隔，取第 2 列到末尾

# 实际场景：提取进程 PID 列表
ps aux | grep nginx | awk '{print $2}'
```

### wc — 统计 ★★★★

```bash
wc file.txt                     # 行数 单词数 字节数
wc -l file.txt                  # 只统计行数
wc -w file.txt                  # 只统计单词数
wc -c file.txt                  # 只统计字节数
wc -m file.txt                  # 只统计字符数

# 实际场景
ps aux | wc -l                  # 当前进程数
ls -1 | wc -l                   # 当前目录文件数
grep -c "ERROR" app.log         # 错误日志行数（等同于 grep | wc -l）
```

### tee — 双向输出 ★★★★

```bash
# tee 将 stdin 同时输出到终端和文件（T 型管道的含义）
command | tee output.txt                # 输出到终端 + 写入文件
command | tee -a output.txt             # 追加而非覆盖
command 2>&1 | tee output.txt           # 同时捕获 stdout 和 stderr
command | tee >(gzip > output.gz)       # 通过进程替换做流式压缩

# 实际场景：编译 Go 项目并同时看输出 + 存日志
go build ./... 2>&1 | tee build.log

# 实际场景：执行脚本时留一份日志审计
./deploy.sh 2>&1 | tee -a /var/log/deploy.log
```

### tr — 字符替换/删除 ★★★

```bash
# tr 按字符做一对一映射（不像 sed 是按行做正则替换）
echo "hello" | tr 'a-z' 'A-Z'          # 小写转大写 → HELLO
echo "hello" | tr '[:lower:]' '[:upper:]'  # 同上，POSIX 字符类
cat file | tr -d '\r'                   # 删除 Windows 换行符的 \r
tr -s '\n'                              # 压缩连续空行为单个
tr ',' '\t' < data.csv                  # 逗号换 Tab

# 实际场景：Windows 脚本转 Unix 格式
tr -d '\r' < win.sh > unix.sh
```

### column — 格式化表格 ★★★

```bash
# 将空格分隔的文本对齐为表格
mount | column -t                       # 挂载信息表格化
kubectl get pods | column -t            # K8s 输出对齐

# 实际场景：按特定分隔符格式化
cat /etc/passwd | column -t -s ':'
```

### jq — JSON 处理 ★★★★★

Go 后端开发中，API 调试和日志分析大量涉及 JSON，`jq` 是必备工具：

```bash
# ====== 基础操作 ======
curl -s http://localhost:8080/api/user | jq .              # 格式化 JSON
curl -s http://localhost:8080/api/user | jq '.name'        # 提取字段
curl -s http://localhost:8080/api/user | jq '.items[0]'    # 数组第一个元素
curl -s http://localhost:8080/api/user | jq '.items[] | {name, age}'  # 重组对象

# ====== 过滤 ======
cat logs.json | jq 'select(.level == "ERROR")'             # 过滤错误日志
cat logs.json | jq 'select(.cost > 1000)'                  # 慢请求（cost 单位 ms）
cat logs.json | jq 'select(.status >= 400)'                # 所有错误响应

# ====== 统计 ======
cat logs.json | jq -s 'group_by(.handler) | map({handler: .[0].handler, count: length})'  # 按接口统计
cat logs.json | jq -s 'map(.cost) | add / length'          # 平均耗时

# ====== 实际场景 ======
# 场景1：提取容器日志中的特定字段
docker logs container 2>&1 | grep "^{" | jq '.msg'

# 场景2：统计各错误类型的数量
cat app.log | grep "ERROR" | jq -r '.error' | sort | uniq -c | sort -rn

# 场景3：查询 K8s 资源
kubectl get pods -o json | jq '.items[] | {name: .metadata.name, status: .status.phase}'

# 场景4：处理 docker inspect 输出
docker inspect nginx | jq '.[0].NetworkSettings.IPAddress'

# 场景5：检查 Go 服务的 pprof 端点
curl -s http://localhost:6060/debug/pprof/ | grep -oP 'href="\K[^"]+' | xargs -I{} echo http://localhost:6060/debug/pprof/{}
```

> **面试提示**：被问到"如何快速分析 JSON 格式日志"时，`jq` + `grep` + `sort | uniq -c` 的组合是最佳答案。

### xargs — 将标准输入转为命令行参数 ★★★★★

```bash
# 为什么需要 xargs：很多命令不接受管道输入，只接受参数
find . -name "*.log" | xargs rm             # 删除所有 .log 文件
find . -name "*.go" | xargs grep "TODO"      # 在所有 .go 文件中搜索
find . -name "*.tmp" | xargs -n 10 rm        # 每次传 10 个参数
find . -name "*.log" | xargs -I {} mv {} backup/  # 用 {} 表示参数位置
ps aux | grep go | awk '{print $2}' | xargs kill  # 批量 kill Go 进程

# 实际场景：删除所有 .tmp 文件（文件名含空格也 OK）
find . -name "*.tmp" -print0 | xargs -0 rm
```

### 面试题

**Q：grep、awk、sed 区别？**

| 工具 | 核心功能 | 适用场景 |
|------|---------|---------|
| grep | 文本搜索/过滤 | 找包含特定模式的行 |
| awk | 列处理/数据统计 | 按列提取、统计、计算 |
| sed | 流编辑/替换 | 文本替换、删除、插入 |

> 三者常组合使用：`grep` 过滤出目标行 → `awk` 提取列 → `sort`/`uniq` 统计。例如：
> ```bash
> grep "ERROR" app.log | awk '{print $5}' | sort | uniq -c | sort -rn
> ```

---

## 2.5 压缩与解压 ★★★★

### tar — 打包/解包 ★★★★★

```bash
# ====== 打包压缩 ======
tar -czf archive.tar.gz dir/        # 打包并 gzip 压缩（.tar.gz）
tar -cjf archive.tar.bz2 dir/       # 打包并 bzip2 压缩（.tar.bz2）
tar -cJf archive.tar.xz dir/        # 打包并 xz 压缩（.tar.xz，压缩率最高）

# ====== 解包 ======
tar -xzf archive.tar.gz             # 解压 .tar.gz
tar -xjf archive.tar.bz2            # 解压 .tar.bz2
tar -xJf archive.tar.xz             # 解压 .tar.xz
tar -xzf archive.tar.gz -C /opt/    # 解压到指定目录

# ====== 查看（不解压）=====
tar -tzf archive.tar.gz             # 查看压缩包内容

# ====== 参数记忆 ======
# c=create 创建  x=extract 解压
# z=gzip  j=bzip2  J=xz
# v=verbose 显示过程  f=file 指定文件名

# ====== 实际场景 ======
# 场景1：打包 Go 项目（排除 vendor 和 .git）
tar -czf project.tar.gz --exclude=vendor --exclude=.git .

# 场景2：备份并显示进度
tar -czf backup.tar.gz /var/log/ 2>&1 | tee backup.log
```

### gzip / gunzip

```bash
gzip file.txt               # 压缩为 file.txt.gz（原文件删除）
gzip -k file.txt            # 保留原文件
gunzip file.txt.gz          # 解压
gzip -9 file.txt            # 最高压缩率（1-9）
```

### zip / unzip

```bash
zip -r archive.zip dir/             # 递归压缩目录
unzip archive.zip                   # 解压
unzip -l archive.zip                # 查看内容
unzip archive.zip -d /opt/          # 解压到指定目录
```

### rsync — 增量同步 ★★★★

```bash
# ====== 基础用法 ======
rsync -av source/ dest/                     # 本地同步（-a 归档模式，-v 显示过程）
rsync -av source/ user@host:/path/          # 推送到远程
rsync -av user@host:/path/ dest/            # 从远程拉取
rsync -avz source/ user@host:/path/         # -z 压缩传输（慢速网络）

# ====== 常用参数 ======
rsync -av --delete source/ dest/            # 删除目标多余文件（保持完全一致）
rsync -av --exclude="*.log" source/ dest/   # 排除特定文件
rsync -av --exclude={'vendor','.git','node_modules'} source/ dest/  # 排除多个
rsync -av --dry-run source/ dest/           # 空跑模式（先看看会同步什么）
rsync -av --progress source/ dest/          # 显示传输进度
rsync -av -e 'ssh -p 2222' source/ user@host:/path/  # 指定 SSH 端口

# ====== 实际场景 ======
# 场景1：增量备份项目（只传输变化部分）
rsync -avz --delete /opt/myapp/ backup:/backup/myapp/

# 场景2：两台服务器间同步静态文件
rsync -avz /var/www/static/ user@cdn-01:/var/www/static/

# 场景3：Go 项目部署后，将二进制发布到目标机器
rsync -avz ./bin/server user@prod-01:/opt/myapp/
```

> **面试答法**：`rsync` 和 `scp` 的区别在于 rsync 支持增量传输（只传差异部分）、断点续传、可以保留文件属性。大文件或大量文件同步用 rsync，小文件快速拷贝用 scp。

### scp — 远程拷贝 ★★★★

```bash
# ====== 基础用法 ======
scp file.txt user@host:/path/              # 本地 → 远程
scp user@host:/path/file.txt ./            # 远程 → 本地
scp -r dir/ user@host:/path/               # 递归拷贝目录
scp -P 2222 file.txt user@host:/path/      # 指定端口（注意是大写 P）

# ====== 实际场景 ======
# 场景1：快速下载远程服务器上的日志
scp prod-01:/var/log/app/error.log ./error_prod.log

# 场景2：上传本地编译好的 Go 二进制到测试服务器
scp ./bin/server test-01:/opt/myapp/
```

### md5sum / sha256sum — 文件完整性校验 ★★★★

```bash
md5sum file.tar.gz                          # 计算 MD5
sha256sum file.tar.gz                       # 计算 SHA256（推荐，更安全）

# 批量校验
sha256sum *.tar.gz > checksums.txt          # 生成校验文件
sha256sum -c checksums.txt                  # 校验所有文件

# 实际场景：下载 Go SDK 后验证完整性
# 从 golang.org 下载 go1.21.0.linux-amd64.tar.gz
# 对比官网提供的 SHA256 值
sha256sum go1.21.0.linux-amd64.tar.gz
```

---

## 2.6 系统信息与系统管理命令 ★★★★

### 系统信息

```bash
uname -a                    # 所有系统信息（内核版本、架构等）
uname -r                    # 内核版本号（如 5.15.0-91-generic）
cat /etc/os-release         # 发行版信息（Ubuntu 22.04.3 LTS）
hostname                    # 主机名
hostname -i                 # 主机 IP 地址
date                        # 当前日期时间
date +%Y%m%d_%H%M%S         # 格式化：20250101_120000
uptime                      # 运行时间 + 系统负载
who                         # 当前登录用户
whoami                      # 当前用户名
id                          # 当前用户 UID/GID 及所有组
last                        # 历史登录记录
last -n 10                  # 最近 10 次登录
```

### watch — 周期性执行命令 ★★★★

```bash
# watch 每隔一定时间执行一次命令，并全屏显示结果
watch -n 2 'ss -antp | wc -l'               # 每 2 秒查看连接数
watch -n 1 'free -h'                        # 每秒查看内存变化
watch -d 'ls -l'                            # -d 高亮变化部分
watch -n 5 'df -h'                          # 每 5 秒查看磁盘使用

# 实际场景：监控服务连接数变化
watch -d -n 1 'ss -antp | grep ESTABLISHED | wc -l'

# 实际场景：观察 Go 进程内存变化趋势
watch -n 2 'ps aux | grep myapp | grep -v grep'

# 实际场景：监控端口监听状态
watch -n 1 'ss -lntp | grep :8080'
```

### systemd 服务管理 ★★★★★

```bash
# systemctl — 管理 systemd 服务
systemctl start nginx           # 启动
systemctl stop nginx            # 停止
systemctl restart nginx         # 重启
systemctl reload nginx          # 重新加载配置（不停服务）
systemctl status nginx          # 查看状态（含最近日志）
systemctl enable nginx          # 开机自启
systemctl disable nginx         # 取消自启
systemctl is-active nginx       # 是否运行中
systemctl is-enabled nginx      # 是否开机自启
systemctl list-units --type=service  # 列出所有服务

# journalctl — 服务日志（生产环境排查必用）
journalctl -u nginx             # 查看服务日志
journalctl -u nginx -f          # 实时追踪日志
journalctl -u nginx --since "1 hour ago"    # 最近 1 小时
journalctl -u nginx --since "2025-01-01"    # 从指定日期
journalctl -u nginx -n 100      # 最近 100 行
journalctl -u nginx -p err      # 只看错误级别
journalctl -u nginx | grep ERROR

# Go 服务部署为 systemd 服务：
# /etc/systemd/system/myapp.service
# [Unit]
# Description=My Go App
# After=network.target
# [Service]
# Type=simple
# User=appuser
# WorkingDirectory=/opt/myapp
# ExecStart=/opt/myapp/server
# Restart=on-failure
# RestartSec=5
# [Install]
# WantedBy=multi-user.target
# systemctl daemon-reload && systemctl enable --now myapp

# ====== Go 服务的最佳实践：Type=notify ======
# Type=simple：systemd 认为 ExecStart 启动后服务就"就绪"了
#                 但 Go 程序可能还在初始化 DB/Redis 连接，还没开始 accept
#                 如果 K8s/docker-compose 在这个窗口内发请求 → 连接失败
#
# Type=notify：服务主动告诉 systemd "我准备好了，可以接流量了"
# Go 代码需要配合 systemd-notify 或 sd_notify 库：
#
# import "github.com/coreos/go-systemd/v22/daemon"
# func main() {
#     // ... 初始化数据库连接池、加载配置 ...
#     http.ListenAndServe(":8080", handler)
#     daemon.SdNotify(false, "READY=1")  // 通知 systemd：服务就绪
#     // ... 处理信号，优雅关闭 ...
# }
#
# systemd unit 改为：
# [Service]
# Type=notify
# WatchdogSec=30  # 健康检查：30 秒内没收到通知则重启
#
# 更多好处：
# - systemctl start 会阻塞到服务真正就绪
# - 配合 WatchdogSec 实现自动重启（Go 中定期 daemon.SdNotify(false, "WATCHDOG=1")）
# - 依赖管理更精确（After= 结合 Type=notify 确保下游服务启动时上游已就绪）
```

### crontab 定时任务 ★★★★

```bash
crontab -e                  # 编辑定时任务
crontab -l                  # 列出定时任务
crontab -r                  # 删除所有定时任务

# crontab 格式：分 时 日 月 周 命令
# ┌─ 分 (0-59)
# │ ┌─ 时 (0-23)
# │ │ ┌─ 日 (1-31)
# │ │ │ ┌─ 月 (1-12)
# │ │ │ │ ┌─ 周 (0-7, 0=周日)
# │ │ │ │ │
# * * * * * /path/to/command

# 常用示例
0 2 * * * /opt/backup.sh                    # 每天凌晨 2 点
*/5 * * * * /opt/health_check.sh            # 每 5 分钟
0 0 * * 0 /opt/weekly_report.sh             # 每周日凌晨
0 3 1 * * /opt/monthly_clean.sh             # 每月 1 号凌晨 3 点
0 9-18 * * 1-5 /opt/business_check.sh       # 工作日 9-18 点每小时

# cron 日志
tail -f /var/log/cron                        # CentOS/RHEL
tail -f /var/log/syslog | grep CRON          # Ubuntu/Debian
```

### shutdown / reboot — 关机与重启 ★★★

```bash
shutdown -h now                 # 立即关机
shutdown -h +10                 # 10 分钟后关机
shutdown -r now                 # 立即重启
shutdown -r 02:00               # 凌晨 2 点重启
shutdown -c                     # 取消已安排的关机/重启
reboot                          # 立即重启（等同于 shutdown -r now）
poweroff                        # 立即关机（等同于 shutdown -h now）
```

### 环境变量管理

```bash
# 设置环境变量
export GOPATH=/home/user/go
export PATH=$PATH:/usr/local/go/bin

# 查看
env                         # 所有环境变量
echo $GOPATH                # 单个

# 配置文件位置
~/.bashrc                   # 每次新终端加载
~/.bash_profile             # 登录 shell 加载
~/.zshrc                    # zsh 用户配置
/etc/profile                # 系统级全局配置
source ~/.bashrc            # 立即生效

# Go 开发常用环境变量
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
export GO111MODULE=on
export GOPROXY=https://goproxy.cn,direct
export GOPRIVATE=github.com/mycompany/*
```

---

## 2.7 SSH 基础 ★★★★

### 密钥管理

```bash
# 生成 SSH 密钥对（推荐 ed25519，比 RSA 更安全更快）
ssh-keygen -t ed25519 -C "your_email@example.com"
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"  # 兼容旧系统

# 将公钥复制到远程服务器（免密码登录）
ssh-copy-id user@host
# 等同手动操作：cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# 测试连接
ssh -T git@github.com                # 测试 GitHub 连接
ssh -v user@host                     # 调试模式（-vvv 更详细）
```

### SSH 客户端配置（跳板机/简化连接）

```bash
# ~/.ssh/config 示例
Host prod-*                          # 通配符匹配所有生产服务器
    User deploy
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    StrictHostKeyChecking no         # 首次连接不确认（CI/CD 环境）

Host prod-app-01
    HostName 10.0.1.101

Host prod-db-01
    HostName 10.0.1.201

# 跳板机配置（通过堡垒机访问内网机器）
Host internal-server
    HostName 192.168.1.50
    User root
    ProxyJump bastion               # 通过 bastion 跳转

Host bastion
    HostName jump.example.com
    User ops
    Port 2222

# 配置好之后直接：
# ssh prod-app-01              → 等价于 ssh deploy@10.0.1.101
# scp file.txt prod-app-01:/opt/  → 直接使用别名

# 权限安全
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519       # 私钥必须 600
chmod 644 ~/.ssh/id_ed25519.pub   # 公钥可以 644
chmod 600 ~/.ssh/config           # 配置文件
chmod 600 ~/.ssh/authorized_keys  # 授权密钥
```

### SSH 端口转发（开发调试利器）

```bash
# 本地端口转发：将远程服务映射到本地
ssh -L 3306:prod-db.internal:3306 user@bastion
# 访问本地 3306 = 访问生产数据库（通过堡垒机）
# Go 中开发调试：./myapp --db-host=127.0.0.1 --db-port=3306

# 远端端口转发：将本地服务暴露给远程（少用，调试用）
ssh -R 8080:localhost:8080 user@remote-host
# 远程机器访问 localhost:8080 = 访问你本机的 8080

# 动态端口转发（SOCKS5 代理）
ssh -D 1080 user@bastion
# 配置浏览器/curl 使用 SOCKS5 代理 127.0.0.1:1080
```

### 面试题

**Q：SSH 免密登录原理？**

> ① 本机 `ssh-keygen` 生成公钥+私钥对；② 公钥放到远程 `~/.ssh/authorized_keys`；③ 连接时远程发随机字符串，本机用私钥签名；④ 远程用公钥验证签名，通过则登录。整个过程私钥始终不离开本机。

**Q：如何安全地管理多台服务器的 SSH 访问？**

> ① 统一使用堡垒机（跳板机），所有服务器只允许堡垒机的 SSH；② `~/.ssh/config` 配置 ProxyJump 简化使用；③ 私钥设置密码保护（passphrase）；④ 生产服务器禁用密码登录，只允许密钥认证。

---

<a id="第三章-用户与权限管理"></a>
# 第三章 用户与权限管理 ★★★★★

## 3.1 用户管理

```bash
useradd username                        # 创建用户
useradd -m -s /bin/bash username       # 创建用户并指定 shell
useradd -u 1500 username               # 指定 UID
useradd -g docker username             # 指定初始组
usermod -aG docker username            # 将用户添加到 docker 组
usermod -s /bin/zsh username           # 修改用户 shell
userdel username                       # 删除用户
userdel -r username                    # 删除用户及其 home 目录
passwd username                        # 修改用户密码
```

## 3.2 用户组管理

```bash
groupadd mygroup                        # 创建组
groupdel mygroup                        # 删除组
groups username                         # 查看用户所属的组
id username                             # 查看用户 UID/GID 及所有组
```

## 3.3 文件权限 ★★★★★

### 权限表示

```bash
-rwxr-xr--  1 user group  4096 Jan 1 10:00 file.txt
│└─┬┘└─┬┘└─┬┘
│  │   │   └── other（其他用户）：r-- = 只读
│  │   └────── group（同组用户）：r-x = 读+执行
│  └────────── owner（所有者）：rwx = 读+写+执行
└───────────── 文件类型（-普通文件 d目录 l软链接）
```

### 权限含义

| 权限 | 文件 | 目录 |
|------|------|------|
| r（读） | 可查看内容 | 可列出目录内容（ls） |
| w（写） | 可修改内容 | 可创建/删除文件 |
| x（执行） | 可执行 | 可进入目录（cd） |

### 数字权限

| 数字 | 权限 | 含义 |
|------|------|------|
| 4 | r-- | 只读 |
| 5 | r-x | 读+执行 |
| 6 | rw- | 读+写 |
| 7 | rwx | 全部权限 |

```
rwx r-x r-x = 755 = 所有者全权限 + 组和其他人读执行
rw- r-- r-- = 644 = 所有者读写 + 组和其他人只读
rwx rwx rwx = 777 = 所有人全权限（危险！）
```

## 3.4 chmod ★★★★★

```bash
# 数字方式
chmod 755 script.sh             # rwxr-xr-x
chmod 644 file.txt              # rw-r--r--
chmod 600 ~/.ssh/id_rsa         # rw-------（私钥安全）
chmod 750 /opt/app              # rwxr-x---

# 符号方式
chmod +x script.sh              # 添加执行权限
chmod -x script.sh              # 移除执行权限
chmod u+x file                  # 所有者加执行
chmod g+w file                  # 组加写权限
chmod o-r file                  # 其他人移除读权限
chmod a+r file                  # 所有人加读权限
chmod u=rwx,g=rx,o= file       # 精确设置：所有者 rwx，组 rx，其他人无

# 递归修改
chmod -R 755 dir/               # 递归修改目录及内容
chmod -R g+w dir/               # 递归添加组写权限

# 实际场景：Go 项目权限设置
chmod 755 /opt/go-app/bin/server           # 可执行文件
chmod 644 /opt/go-app/etc/config.yaml      # 配置文件
chmod 600 /opt/go-app/etc/secrets.yaml     # 敏感文件（仅所有者）
```

### 面试题

**Q：chmod 755 是什么意思？**

> 所有者 rwx（7 = 读+写+执行），同组用户 r-x（5 = 读+执行），其他用户 r-x（5 = 读+执行）。常用于脚本和可执行文件，所有者和 root 可以修改，其他人只能读和执行。

**Q：chmod 777 安全吗？**

> 不安全。777 意味着任何用户都可以读写执行该文件。如果攻击者获得了服务器的低权限账号，可以修改你的文件（植入木马、篡改脚本）。生产环境中应遵循"最小权限原则"：可执行脚本 755，普通文件 644，敏感文件（密钥、证书）600。

### ACL（访问控制列表）— 更细粒度的权限 ★★★

Linux 传统 rwx 权限只能指定 owner/group/others 三类，ACL 可以给任意用户/组单独设置权限：

```bash
# 查看 ACL
getfacl file.txt
# # file: file.txt
# # owner: root
# # group: root
# user::rw-
# user:alice:rw-        # alice 用户有读写权限
# group::r--
# group:dev:rwx         # dev 组有全部权限
# mask::rwx
# other::r--

# 设置 ACL
setfacl -m u:alice:rw file.txt             # 给 alice 用户添加读写权限
setfacl -m g:dev:rwx file.txt              # 给 dev 组添加全部权限
setfacl -m u:alice:- file.txt              # 撤销 alice 的所有权限

# 递归设置
setfacl -R -m g:dev:rwx /opt/project/      # 递归给目录设置

# 删除 ACL
setfacl -x u:alice file.txt                # 删除 alice 的 ACL 条目
setfacl -b file.txt                        # 清空所有 ACL

# 实际场景：Web 服务器目录，给 nginx 用户读权限，但 owner 不变
setfacl -R -m u:nginx:rx /var/www/html/
```

> 注意：设置了 ACL 的文件在 `ls -l` 中权限位末尾会显示 `+`，如 `-rw-rwxr--+`。

### 特殊权限 SUID / SGID / Sticky Bit ★★★★

```bash
# ====== SUID（Set User ID）======
# 以文件所有者的身份执行，而非执行者的身份
# 权限位显示为 s（替代所有者的 x）
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root /usr/bin/passwd
# ↑ 普通用户执行 passwd 时，进程以 root 身份运行（才能修改 /etc/shadow）
chmod u+s /usr/bin/myapp     # 添加 SUID
chmod 4755 /usr/bin/myapp    # 数字方式（4 = SUID）
# 安全提示：SUID 程序如有漏洞，攻击者可提权到 root

# ====== SGID（Set Group ID）======
# 对文件：以文件所属组的身份执行
# 对目录：该目录下新建文件继承目录的所属组（而非创建者的组）
chmod g+s /shared/project/   # 目录设置 SGID
chmod 2755 /shared/project/  # 数字方式（2 = SGID）
# 实际场景：团队共享目录，所有人新建的文件都属于项目组

# ====== Sticky Bit（粘滞位）======
# 对目录：只有文件所有者、目录所有者或 root 才能删除文件
ls -ld /tmp
# drwxrwxrwt 1 root root /tmp   ← t 表示粘滞位
chmod +t /shared/            # 添加 sticky bit
chmod 1777 /shared/          # 数字方式（1 = sticky bit）
# 实际场景：/tmp 目录所有人都可写，但只能删除自己的文件

# ====== 特殊权限数字组合 ======
# SUID=4  SGID=2  Sticky=1  放在权限数字最前面
# chmod 4755 = SUID + rwxr-xr-x
# chmod 2755 = SGID + rwxr-xr-x
# chmod 1777 = Sticky + rwxrwxrwx

# ====== 实际场景：查找有 SUID 权限的文件（安全审计）======
find / -perm -4000 -type f 2>/dev/null    # 查找所有 SUID 文件
find / -perm -2000 -type d 2>/dev/null    # 查找所有 SGID 目录
```

> **面试答法**：SUID 让程序以文件所有者身份执行（passwd 需要 root 权限），SGID 对目录可使新建文件继承目录的组（团队共享目录），Sticky Bit 限制只有所有者能删除（/tmp 就是 1777）。特殊权限数字写在最前面：SUID=4, SGID=2, Sticky=1。

## 3.5 chown — 修改所有者 ★★★★

```bash
chown user file                     # 修改文件所有者
chown user:group file               # 同时修改所有者和组
chown -R user:group dir/            # 递归修改
chown :docker file                  # 只修改组
```

## 3.6 sudo ★★★★

### sudo 原理

> `sudo` 允许普通用户以 root（或其他用户）的身份执行命令。配置在 `/etc/sudoers` 文件中。实际执行时，sudo 会检查 `/etc/sudoers` 中是否允许该用户执行该命令，如果允许则临时提升权限。

```bash
sudo command                        # 以 root 执行
sudo -u nginx command               # 以 nginx 用户执行
sudo -i                             # 切换到 root 的交互式 shell
sudo su -                           # 切换到 root（等同于 sudo -i）
sudo visudo                         # 安全地编辑 /etc/sudoers
```

### 面试题

**Q：sudo 和 su 区别？**

| 对比项 | sudo | su |
|--------|------|-----|
| 含义 | 以其他用户身份执行命令 | 切换用户（switch user） |
| 需要密码 | 当前用户密码 | 目标用户密码 |
| 权限粒度 | 可精确控制允许执行哪些命令 | 获得目标用户的全部权限 |
| 审计 | 记录所有 sudo 命令到日志 | 无详细审计 |
| 使用建议 | 生产环境推荐 sudo | 个人开发环境 |

---

<a id="第四章-进程管理"></a>
# 第四章 进程管理 ★★★★★

## 4.1 什么是进程

- **PID**：进程 ID，唯一标识
- **PPID**：父进程 ID
- **PCB**（Process Control Block）：内核中描述进程的数据结构，包含 PID、状态、优先级、内存信息、打开的文件描述符等

### 进程状态

```
创建(fork) → 就绪(R) ── 获得CPU时间片 ──→ 运行(R)
                                         │
           ←── CPU时间片用完（回到就绪）──┤
                                         │
           ←── 等待I/O/锁/信号 ────→ 睡眠
                                         │可中断(S) / 不可中断(D)
           ←── 收到信号唤醒 ──────────────┘
                                         │
                                        终止
                                         │
                           父进程未回收 → 僵尸(Z)
```

**D 状态（不可中断睡眠）**：进程在等待磁盘 I/O 或内核操作，不响应任何信号（包括 `kill -9`）。D 状态过多通常意味着 I/O 瓶颈。

**僵尸进程（Z）**：进程已退出，但父进程未调用 `wait()` 回收，PCB 残留。大量僵尸进程会耗尽 PID。

### 进程 vs 线程 vs 协程（面试必考）★★★★★

| 对比维度 | 进程 | 线程 | 协程（Go goroutine）|
|---------|------|------|---------------------|
| 资源隔离 | 独立地址空间 | 共享进程地址空间 | 共享线程栈空间 |
| 创建开销 | 大（fork + 新页表） | 中（内核态创建） | 极小（用户态，初始 2KB 栈） |
| 切换开销 | 大（TLB 刷新、页表切换，~1-10µs） | 中（寄存器保存/恢复，~1-2µs） | 极小（只切栈指针，~几十ns） |
| 通信方式 | IPC（管道/消息队列/共享内存） | 共享内存 + 锁 | channel / select |
| 崩溃影响 | 不影响其他进程 | 影响整个进程 | 影响整个进程（panic 可 recover） |
| 调度方式 | 内核调度 | 内核调度 | Go runtime 协作式 + 抢占式 |
| 数量上限 | 数百~数千 | 数千（栈内存开销大） | 数十万~百万 |
| Linux 视角 | 独立 task_struct + mm_struct | task_struct 共享 mm_struct | 用户态，内核无感知 |

**从 Linux 内核视角看**：

```
进程：独立的 task_struct + 独立的 mm_struct（地址空间）
      ┌──────────────────────────────────────┐
      │ task_struct                         │
      │  ├─ pid = 1234        进程 ID       │
      │  ├─ tgid = 1234       线程组 ID     │
      │  ├─ mm_struct → 页表  独立地址空间   │
      │  ├─ files_struct → FD表              │
      │  └─ signal_struct                    │
      └──────────────────────────────────────┘

线程：多个 task_struct 共享同一个 mm_struct
      ┌────────────────┐  ┌────────────────┐
      │ thread A       │  │ thread B       │
      │ task_struct    │  │ task_struct    │
      │  ├─ pid = 1235 │  │  ├─ pid = 1236 │
      │  └─ tgid = 1234│  │  └─ tgid = 1234│
      └──────┬─────────┘  └──────┬─────────┘
             └──── 共享 ────────┘
             mm_struct, files_struct, signal_struct
      → ps 看到的是 tgid（PID），线程属性用 ps -T 或 top -H 看

Goroutine（完全在用户态）：
      Go runtime 管理 goroutine（g 结构体）和 OS 线程（m 结构体）的映射
      内核看不到 goroutine，只能看到几个 OS 线程
      GMP 模型：G(goroutine) → P(处理器) → M(OS线程)
      调度由 Go runtime 在用户态完成，不需陷入内核
```

> **面试答法**：进程是资源分配的基本单位，有独立地址空间；线程是 CPU 调度的基本单位，共享进程地址空间。Go 的 goroutine 是用户态协程，由 Go runtime 调度到 OS 线程上执行。核心区别：① 创建成本 goroutine 极低（2KB 初始栈 vs 线程 ~1MB），Linux 上 10000 个线程基本不可行但 10 万个 goroutine 完全没问题；② 切换成本极低（用户态直接切换 vs 线程需要内核态切换）；③ goroutine 通过 GMP 模型实现高效的 M:N 调度，充分利用多核。

---

## 4.2 ps 命令 ★★★★★

```bash
# ====== 常用组合 ======
ps -ef                          # 显示所有进程完整信息
ps aux                          # 显示所有进程（BSD 风格，含 CPU/内存）
ps -ef | grep nginx             # 查找 nginx 进程
ps aux --sort=-%cpu | head      # CPU 占用最高的进程
ps aux --sort=-%mem | head      # 内存占用最高的进程
ps -e -o pid,ppid,cmd,%cpu,%mem # 自定义输出列
ps -u username                  # 查看指定用户的进程

# ====== 输出列说明（ps aux）=====
# USER   PID  %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
# %CPU = CPU 使用率
# %MEM = 物理内存使用百分比
# VSZ  = 虚拟内存大小（KB）
# RSS  = 物理内存大小（KB）
# STAT = 进程状态（R运行 S睡眠 D不可中断 Z僵尸 T停止）
# START = 启动时间
# TIME = 累计 CPU 时间
```

### 面试题

**Q：ps -ef 和 ps aux 区别？**

> 两者都显示所有进程，只是输出格式不同。`ps -ef` 是 Unix 风格，显示 PPID；`ps aux` 是 BSD 风格，显示 CPU/内存使用率和 VSZ/RSS。实际工作中两者都用，`ps aux | grep xxx` 更常见，因为能看到资源占用。

### pgrep / pidof — 快速获取 PID ★★★★

```bash
# pgrep — 按进程名查找 PID（比 ps | grep 更干净）
pgrep nginx                         # 所有 nginx 进程的 PID
pgrep -f "go run"                   # 按完整命令行匹配
pgrep -u nginx                      # 查看 nginx 用户的所有进程
pgrep -l nginx                      # 同时显示进程名和 PID
pgrep -x nginx                      # 精确匹配（进程名恰好是 nginx）

# pidof — 按进程名直接返回 PID
pidof nginx                         # 列出所有 nginx 的 PID
pidof -s nginx                      # 只返回一个 PID（-s single-shot）

# 实际场景：优雅重启
kill -HUP $(pgrep nginx)            # nginx reload
kill -15 $(pidof myapp)             # 优雅停止 Go 服务
```

## 4.3 top 命令 ★★★★★

```bash
top                             # 启动 top
# 交互命令（在 top 界面内）：
#   P    按 CPU 排序
#   M    按内存排序
#   k    输入 PID 杀死进程
#   q    退出
#   1    显示/隐藏各 CPU 核心
#   c    显示完整命令行
#   E    切换内存单位（KB/MB/GB）
#   u    过滤显示某用户的进程

# 命令行参数
top -u nginx                    # 只看 nginx 用户的进程
top -p 1234                     # 只看 PID 1234 的进程
top -H -p 1234                  # 查看进程的所有线程（重要！）
top -n 3 -d 2                   # 刷新 3 次，间隔 2 秒

# 重要指标解读：
# load average: 0.5, 0.8, 1.0   # 1/5/15 分钟平均负载
# Tasks: 150 total, 1 running    # 总进程数，运行中
# %Cpu(s): 10.0 us, 2.0 sy       # us=用户态 sy=内核态 wa=iowait
# MiB Mem: 16000 total, 2000 free, 8000 used, 6000 buff/cache
```

### 面试题

**Q：top 中哪些指标最重要？**

> ① **load average**（平均负载）：超过 CPU 核数说明过载；② **%Cpu wa**（iowait）：高说明 I/O 瓶颈；③ **%Cpu sy**（内核态 CPU）：高说明系统调用过多；④ **RES**（物理内存）：真实内存占用；⑤ **S**（状态）：R 运行、S 睡眠、D 不可中断（磁盘 I/O 卡住）、Z 僵尸。

## 4.4 htop — 增强版 top ★★★

```bash
htop                            # 彩色界面，支持鼠标，更友好
# F2 进入设置，F3 搜索，F9 杀死进程
```

## 4.5 pstree — 进程树 ★★★★

```bash
pstree                          # 树状显示进程关系
pstree -p                       # 显示 PID
pstree -p $(pgrep nginx)        # 显示 nginx 的进程树
```

## 4.6 kill 命令 ★★★★★

### 常见信号

| 信号编号 | 信号名 | 含义 | 能否捕获/忽略 |
|---------|--------|------|:---:|
| 1 | SIGHUP | 挂起（重新加载配置） | ✓ |
| 2 | SIGINT | Ctrl+C 中断 | ✓ |
| 9 | SIGKILL | 强制终止 | ✗ |
| 15 | SIGTERM | 优雅终止（默认） | ✓ |
| 17 | SIGCHLD | 子进程状态变化 | ✓ |
| 19 | SIGSTOP | 暂停进程 | ✗ |

```bash
kill <PID>                      # 发送 SIGTERM（优雅终止）
kill -15 <PID>                  # 同 kill（SIGTERM）
kill -9 <PID>                   # 强制杀死（SIGKILL，不处理直接终止）
kill -l                         # 列出所有信号
kill -HUP <PID>                 # 发送 SIGHUP（常用于重新加载配置）

killall nginx                   # 杀死所有名为 nginx 的进程
pkill -f "go run main.go"       # 按命令行匹配杀进程
```

### 面试题

**Q：kill -9 和 kill -15 区别？**

| 对比项 | kill -15 (SIGTERM) | kill -9 (SIGKILL) |
|--------|--------------------|--------------------|
| 能否被进程捕获 | 能 | 不能 |
| 进程能否做清理 | 能（关闭连接、写日志、释放资源） | 直接终止 |
| 推荐度 | 首选 | 最后手段 |
| Go 中的表现 | `signal.Notify` 可以捕获 | 无法捕获，直接退出 |

> **实战建议**：先用 `kill -15`，给进程机会做优雅关闭（Go 中 `http.Server.Shutdown()`）。如果进程无响应，再用 `kill -9`。

### nice / renice — 进程优先级 ★★★

```bash
# Linux 优先级范围：-20（最高优先级）到 19（最低优先级）
# nice 值越大，进程越"nice"（谦让），优先级越低

# nice — 以指定优先级启动进程
nice -n 10 ./cpu_heavy_task                # 以较低优先级启动
nice -n -5 ./important_task                # 以较高优先级启动（需 root）

# renice — 修改运行中进程的优先级
renice -n 5 -p <PID>                       # 降低指定进程的优先级
renice -n -10 -p <PID>                     # 提高优先级（需 root）
renice -n 10 -u nginx                      # 修改 nginx 用户所有进程的优先级

# 实际场景：后台跑数据迁移/备份任务，降低优先级避免影响在线服务
nice -n 19 ./data_migration_tool
```

### time — 测量命令执行时间 ★★★★

```bash
# time 命令测量命令执行的实际耗时
time ./data_migration_tool

# 输出解读：
# real    0m5.123s    # 墙上时钟时间（从开始到结束的实际耗时）
# user    0m3.456s    # 用户态 CPU 时间（程序自身代码消耗的 CPU）
# sys     0m0.789s    # 内核态 CPU 时间（系统调用消耗的 CPU）

# 关键判断：
# real ≈ user + sys → CPU 密集型（程序都在跑 CPU）
# real >> user + sys → I/O 密集型或等待网络/磁盘（大部分时间在等待）

# 实际场景：对比优化前后的执行时间
time go build ./...
# 优化前：real 0m15s
# 优化后：real 0m3s

# 精确计时（/usr/bin/time 比 shell 内建 time 更详细）
/usr/bin/time -v ls -la
# 输出包括：内存占用、上下文切换次数、页错误次数、I/O 统计
```

## 4.7 后台运行 ★★★★★

### & — 后台执行

```bash
./app &                         # 后台运行，终端关闭进程也关闭
./app > app.log 2>&1 &          # 后台运行并重定向输出
```

### nohup — 忽略挂断信号 ★★★★★

```bash
nohup ./app &                   # 终端关闭后进程继续运行
nohup ./app > app.log 2>&1 &    # 标准输出和错误都写入文件

# nohup 原理：让进程忽略 SIGHUP 信号。
# 终端关闭时，内核会给该终端的所有进程发 SIGHUP。
# nohup 修改进程的信号处理，使其忽略 SIGHUP，所以进程继续运行。

# 查看 nohup 启动的进程
ps aux | grep app
jobs -l
```

### screen / tmux — 终端复用器 ★★★★

```bash
# screen
screen -S myapp                 # 创建命名会话
screen -ls                      # 列出所有会话
screen -r myapp                 # 重新连接
Ctrl+A D                        # 暂时离开会话

# tmux（推荐，功能更强）
tmux new -s myapp               # 创建会话
tmux ls                         # 列出会话
tmux attach -t myapp            # 重新连接
Ctrl+B D                        # 暂时离开
```

### 面试题

**Q：nohup 原理是什么？**

> `nohup` 修改进程对 SIGHUP 信号的处理为 `SIG_IGN`（忽略）。终端关闭时，内核向该终端关联的所有进程发送 SIGHUP，默认行为是终止进程。nohup 启动的进程收到 SIGHUP 后直接忽略，因此终端关闭不影响进程继续运行。同时 nohup 会自动将输出重定向到 `nohup.out` 文件。

## 4.8 Signal 信号处理 + Go 优雅关闭 ★★★★★

### Linux 信号机制

信号（Signal）是 Linux 进程间的一种异步通知机制。进程可以注册信号处理函数，在信号到达时执行。

```
信号的生命周期：
  内核/进程发送 signal
    → 内核设置目标进程 task_struct 中的 signal 位图
    → 目标进程被调度时检查 pending signals
    → 执行注册的信号处理函数（signal handler）
    → 或执行默认行为（terminate / ignore / core dump）
```

| 信号 | 编号 | 默认行为 | 能否捕获 | 常见场景 |
|------|:---:|---------|:---:|---------|
| SIGHUP | 1 | Terminate | ✓ | 终端关闭、重新加载配置（nginx reload） |
| SIGINT | 2 | Terminate | ✓ | Ctrl+C 中断前台进程 |
| SIGQUIT | 3 | Core dump | ✓ | Ctrl+\ 退出并生成 core dump |
| SIGKILL | 9 | Terminate | ✗ | 强制杀进程（最后手段） |
| SIGTERM | 15 | Terminate | ✓ | kill 命令默认信号，优雅终止 |
| SIGCHLD | 17 | Ignore | ✓ | 子进程状态改变（退出、暂停） |
| SIGSTOP | 19 | Stop | ✗ | 暂停进程（不能被捕获） |
| SIGUSR1 | 10 | Terminate | ✓ | 用户自定义信号 1 |
| SIGUSR2 | 12 | Terminate | ✓ | 用户自定义信号 2 |
| SIGPIPE | 13 | Terminate | ✓ | 向已关闭的管道写数据 |
| SIGSEGV | 11 | Core dump | ✓ | 段错误（非法内存访问） |

### Go 中的信号处理（标准写法）

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    server := &http.Server{
        Addr:    ":8080",
        Handler: nil, // 你的 router
    }

    // 启动服务
    go func() {
        log.Printf("Server starting on :8080")
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server failed: %v", err)
        }
    }()

    // 监听信号（关键代码）
    quit := make(chan os.Signal, 1)
    // 注册关注的信号
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    // SIGINT  = Ctrl+C
    // SIGTERM = kill -15 / docker stop / K8s pod termination

    s := <-quit  // 阻塞等待信号
    log.Printf("Received signal: %v, shutting down...", s)

    // 优雅关闭
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Printf("Server forced to shutdown: %v", err)
    }

    log.Println("Server exited gracefully")
}
```

### 优雅关闭的核心步骤

```
1. 收到 SIGTERM/SIGINT
2. 停止接收新请求（健康检查返回 unhealthy，K8s 停止分发流量）
3. 等待正在处理的请求完成（server.Shutdown 内置此逻辑）
4. 关闭数据库连接池
5. 关闭消息队列消费者
6. 进程退出

K8s pod termination 时序：
  preStop hook → SIGTERM → 等待 terminationGracePeriodSeconds → SIGKILL
  （默认 30s，超过则强制杀）
```

### 信号的高级用法

```go
// SIGHUP 重新加载配置（类似 nginx reload）
func reloadOnSIGHUP() {
    ch := make(chan os.Signal, 1)
    signal.Notify(ch, syscall.SIGHUP)
    go func() {
        for range ch {
            log.Println("Received SIGHUP, reloading config...")
            // loadConfig()
            log.Println("Config reloaded")
        }
    }()
}

// SIGUSR1 触发 pprof 采集（无需暴露 HTTP 端口）
func pprofOnSIGUSR1() {
    ch := make(chan os.Signal, 1)
    signal.Notify(ch, syscall.SIGUSR1)
    go func() {
        for range ch {
            log.Println("Received SIGUSR1, dumping pprof...")
            // f, _ := os.Create("cpu.prof")
            // pprof.StartCPUProfile(f)
            // time.Sleep(30 * time.Second)
            // pprof.StopCPUProfile()
        }
    }()
}
```

### Docker 中的信号陷阱

```dockerfile
# ❌ 错误：PID 1 的 shell 不转发信号！
CMD ./app

# ❌ 错误：exec 形式写在脚本中，信号还是被 shell 吃掉
CMD ["/bin/sh", "-c", "./app"]

# ✅ 正确：exec 形式，进程作为 PID 1 直接接收信号
CMD ["./app"]

# ✅ Go 中正确处理所有信号（包括 PID 1 的特殊情况）
# Go runtime 会自动将收到的信号转发给 signal.Notify 注册的 channel
# 但 SHELL 启动的进程不转发！所以 docker CMD 必须用 exec 形式
```

> **面试答法**：Linux 信号是异步通知机制。Go 中通过 `signal.Notify(ch, os.SIGTERM, os.SIGINT)` 注册关注的信号，收到信号后触发 graceful shutdown：停止 accept 新连接 → 等待现有请求完成 → 关闭 DB/Redis 连接池 → 退出。这是 K8s/Docker 环境中 Go 服务的标准做法。`kill -9` 无法捕获所以要避免直接使用；`kill -15` 和 `docker stop` 发的是 SIGTERM。

## 4.9 IPC 进程间通信详解 ★★★★

Linux 提供多种 IPC（Inter-Process Communication）机制：

```
匿名管道（Pipe）       ── 单向，只能父子/兄弟进程间，shell 中 | 就是管道
命名管道（FIFO）       ── 单向，任意进程间，有文件路径作为入口
消息队列（Message Q）   ── 有结构的消息传递，支持按类型/优先级读取
共享内存（Shared Mem）  ── 最快的 IPC，直接映射同一块物理内存到双方地址空间
信号量（Semaphore）    ── 进程间同步/互斥（不是用来传递数据的！）
信号（Signal）         ── 异步通知，数据量极少
Socket                ── 可跨机器通信，最通用
Unix Domain Socket    ── 同机器通信，比 TCP 快，比管道灵活
```

| IPC 方式 | 速度 | 跨机器 | 边界 | Go 中对应 |
|---------|:---:|:---:|------|---------|
| 匿名管道 | 快 | ✗ | 流式 | `io.Pipe()` |
| 命名管道 | 快 | ✗ | 流式 | `os.OpenFile("/tmp/fifo", ...)` |
| 消息队列 | 中 | ✗ | 有边界 | syscall 包 |
| 共享内存 | **最快** | ✗ | 任意 | `syscall.Mmap()` / `shm_open` |
| Unix Socket | 快 | ✗ | 流式/数据报 | `net.Dial("unix", path)` |
| TCP Socket | 较慢 | ✓ | 流式 | `net.Dial("tcp", addr)` |

**共享内存为什么最快？** 数据不需要经过内核拷贝，两个进程直接映射同一块物理内存，读写就是普通的内存访问，没有系统调用开销。代价是需要自己用信号量做同步。

```bash
# 管道的实际使用（shell 中无处不在）
ps aux | grep nginx | awk '{print $2}'    # 三个进程通过管道串联

# 命名管道
mkfifo /tmp/myfifo
echo "hello" > /tmp/myfifo &              # 写入端（阻塞直到有读取端）
cat /tmp/myfifo                           # 读取端

# 查看/清理 System V IPC 资源
ipcs -a                                   # 查看所有 IPC 资源
ipcs -m                                   # 查看共享内存段
ipcrm -m <shmid>                          # 删除指定共享内存
ipcrm -a                                  # 删除所有 IPC（高危！）

# 查看进程使用的 IPC
lsof -p <PID> | grep -E "pipe|fifo|unix"
```

**Go 中 IPC 选择指南**：
- 同进程 goroutine 之间 → channel / sync 包（不需要 IPC，这是并发的优势）
- 同机器不同进程 → Unix Domain Socket（推荐，性能和 TCP 相当但延迟更低，API 和 TCP 一样方便迁移）
- 跨机器 → gRPC / HTTP / TCP
- 极致性能需求 → 共享内存 `syscall.Mmap()`（参考 Go 的 `arena` 实验性特性）

---

<a id="第五章-内存管理"></a>
# 第五章 内存管理 ★★★★★

## 5.0 虚拟内存深度 ★★★★★

> 虚拟内存是 Linux 内存管理的基石，也是大厂面试的高频问点。理解虚拟内存才能回答 "malloc 背后发生了什么" 这类问题。

### 为什么需要虚拟内存？

```
问题：每个进程直接操作物理内存会怎样？
  - 进程 A 可能非法读取进程 B 的数据（无隔离）
  - 物理内存碎片化，大块连续分配失败
  - 进程地址空间被物理内存大小限制

解决方案：虚拟内存 = 每个进程有独立的虚拟地址空间，通过页表映射到物理内存
```

### 虚拟地址空间布局

```
进程虚拟地址空间（以 64 位为例，实际只用 48 位）：
┌──────────────────────┐ 0x7fffffffffff
│  内核空间（共享）      │  所有进程共享
├──────────────────────┤ 0x00007fffffffffff（典型边界）
│  栈（Stack）          │ 局部变量，函数调用，向下增长
│       ↓              │
│       ...            │
│       ↑              │
│  堆（Heap）           │ malloc/new 分配，向上增长
├──────────────────────┤
│  BSS 段              │ 未初始化的全局变量（默认 0）
├──────────────────────┤
│  Data 段             │ 已初始化的全局变量
├──────────────────────┤
│  Text 段（代码）      │ 程序指令，只读
└──────────────────────┘ 0x00400000

查看 Go 进程的内存映射：
cat /proc/<PID>/maps   # 查看进程的完整地址空间分布
pmap -x <PID>          # 更友好的格式
```

### 页表（Page Table）与地址转换

```
页表是虚拟地址 → 物理地址的映射表。每个进程有自己的页表，切换进程时切换页表。

虚拟地址（48 位）：
│ 9 bits  │ 9 bits │ 9 bits │ 9 bits │ 12 bits   │
│  PGD    │  PUD   │  PMD   │  PTE   │  offset   │

四级页表查找过程：
  1. PGD（Page Global Directory）：第一级目录
  2. PUD（Page Upper Directory）：第二级
  3. PMD（Page Middle Directory）：第三级
  4. PTE（Page Table Entry）：第四级，指向最终物理页
  5. offset：页内偏移

页大小：通常 4KB

关键问题：为什么用多级页表？
  → 如果单级页表，4GB 空间需要 4GB/4KB = 100万 个页表项 × 4字节 = 4MB
  → 每个进程 4MB，多了浪费；多级页表按需分配，省内存
```

### TLB（Translation Lookaside Buffer）

```
TLB 是 CPU 内部的虚拟地址→物理地址翻译缓存（硬件，类似于哈希表）

地址翻译流程：
  1. 先在 TLB 中查找（~1 个时钟周期，极快）
  2. 命中 TLB → 直接获得物理地址
  3. 未命中 TLB → 遍历页表（需要走内存，~数百时钟周期，慢）
  4. 更新 TLB，下次命中

进程切换的代价：
  - 切换页表 CR3 寄存器
  - 旧进程的 TLB 全部失效（地址空间不同）
  - 新进程初始大量 TLB miss → 性能抖动
```

### 缺页中断（Page Fault）

```
缺页中断：访问的虚拟地址对应的物理页不在内存中

三种情况：
  1. Minor Page Fault（小缺页）：
     物理页已分配，但未映射到页表
     例如：malloc 后首次访问、COW 写时复制触发
     开销：更新页表即可

  2. Major Page Fault（大缺页）：
     物理页在磁盘上（Swap / 文件映射）
     必须从磁盘读取 → 极慢（ms 级别）
     开销：磁盘 I/O + 页表更新

  3. Invalid Page Fault：
     访问非法地址 → SIGSEGV（段错误，程序崩溃）
     Go 中表现为 nil pointer dereference

Go 中的表现：
  var p *int
  *p = 42  // panic: runtime error: invalid memory address or nil pointer dereference
```

### malloc 背后到底发生了什么？（面试常问）

```
用户调用 malloc(1024)：

1. 用户态 glibc/jemalloc/tcmalloc：
   - 检查空闲链表是否有合适的内存块
   - 如果有：直接返回（无需内核参与！）
   - 如果没有：调用 brk() 或 mmap() 向内核申请

2. 内核态：
   - brk()：扩展堆（小内存，< 128KB），移动 program break 指针
   - mmap()：映射匿名内存（大内存，≥ 128KB），返回新虚拟地址区域

3. 关键：内核只是"承诺分配"，并不立即分配物理内存！
   - 只更新进程的 vm_area_struct（记录"这块虚拟地址可用"）
   - 物理页在首次访问时才真正分配（缺页中断）

4. 首次读写 malloc 返回的地址：
   - 触发 minor page fault
   - 内核分配一个 4KB 物理页
   - 更新页表，建立虚拟→物理映射
   - 用户进程无感知，继续执行

这叫"惰性分配"（Lazy Allocation / Demand Paging）：
  - malloc 几乎总是立即返回（OOM 推迟到真正使用时才可能触发）
  - 节约物理内存（申请了不一定用）
```

### 写时复制（Copy-On-Write, COW）

```
COW 是 fork() 的实现基础，fork 创建一个子进程但不立即拷贝所有物理页：

1. fork() 调用后：
   - 子进程获得父进程页表的只读副本
   - 父子进程共享相同的物理页
   - 不需要复制数据 → fork 很快

2. 任意一方尝试写入共享页：
   - 触发 page fault（写只读页 → 非法）
   - 内核检测到是 COW 页面
   - 分配新物理页，拷贝原内容
   - 更新页表，标记为可写
   - 恢复进程执行

Go 中：
  对于并发读写大对象，避免 COW 的隐式副本
  比如 slice 共享底层数组时，一个 goroutine append
  可能导致整个底层数组被复制
```

### 查看虚拟内存的命令

```bash
# 查看进程地址空间
cat /proc/<PID>/maps
# 每行一个 VMA（Virtual Memory Area）
# 7f1234000000-7f1234001000 r-xp /usr/lib/libc.so
# └──地址范围──┘ └权限┘ └映射的文件
# r=读 w=写 x=执行 p=私有 s=共享

# 统计缺页
ps -o min_flt,maj_flt -p <PID>  # 累计大/小缺页次数
pidstat -r 1                     # 实时缺页统计

# 系统级缺页统计（vmstat）
vmstat 1
# si/so = swap in/out → si > 0 说明有 major page fault
```

---

## 5.1 free 命令 ★★★★★

```bash
free -h                         # 人性化显示内存使用

# 输出解读：
#               total    used    free    shared    buff/cache    available
# Mem:          15Gi     8Gi     2Gi     500Mi        5Gi           6Gi
# Swap:         2Gi      0Gi     2Gi
#
# total      = 总物理内存
# used       = 已使用（不含 buff/cache）
# free       = 完全未使用
# buff/cache = 内核缓存（Page Cache、目录项缓存等）
# available  = 真正可用的内存（free + buff/cache 中可回收的部分）
# Swap       = 交换空间（磁盘上的虚拟内存）

free -h -s 2                    # 每 2 秒刷新一次
```

### 面试题

**Q：available 和 free 区别？**

> `free` 是完全没有被使用的物理内存。`available` 是系统认为可以分配给新进程的内存，包括 `free` + `buff/cache` 中可回收的部分。Linux 的内存管理策略是尽可能利用空闲内存做缓存（Page Cache），这部分在需要时可以释放给应用。所以判断内存是否充足应该看 `available`，而不是 `free`。

**Q：Page Cache 是什么？**

> Page Cache（页缓存）是 Linux 内核用来缓存文件数据的一块内存。当你读取文件时，数据会被缓存到 Page Cache 中；下次再读同一个文件，直接从内存读取，不经过磁盘。这是 Linux 性能优秀的关键原因之一。如果 `free` 很低但 `available` 很高，说明大量内存被用于 Page Cache，这是正常的，并且可以随时被回收。

## 5.2 vmstat — 虚拟内存统计 ★★★★★

```bash
vmstat 1                        # 每秒更新
vmstat 1 5                      # 每秒更新，共 5 次

# 输出解读：
# procs: r(运行队列) b(不可中断睡眠)
# memory: swpd(交换) free(空闲) buff(缓冲) cache(缓存)
# swap:  si(从磁盘换入) so(换出到磁盘) ← so > 0 说明内存不足在用 swap
# io:    bi(读磁盘) bo(写磁盘)
# system: in(中断) cs(上下文切换)
# cpu:   us(用户态) sy(内核态) id(空闲) wa(等待I/O) st(被虚拟化偷走)

# 关键指标：
# r > CPU 核数 → CPU 瓶颈
# so > 0 → 内存不足，正在 swap
# cs 极高 → 上下文切换过多
# wa > 10% → I/O 等待严重
```

## 5.3 pmap — 查看进程内存映射

```bash
pmap -x <PID>                   # 进程内存映射详情
pmap -x <PID> | sort -rn -k3    # 按 RSS 排序
cat /proc/<PID>/status | grep VmRSS  # 进程物理内存
cat /proc/<PID>/status | grep VmSize # 进程虚拟内存
```

## 5.4 内存泄漏排查

### Go 程序内存泄漏排查

```bash
# 1. 确认内存泄漏
ps aux --sort=-%mem | head           # 找内存持续增长的进程
top -p <PID>                         # 观察 RES 是否持续增长

# 2. Go 程序：导入 pprof
# 代码中添加：
# import _ "net/http/pprof"
# http.ListenAndServe(":6060", nil)

# 3. 采集 heap profile
curl http://localhost:6060/debug/pprof/heap > heap.prof
go tool pprof heap.prof
# pprof> top10            # 查看内存分配热点
# pprof> list FuncName    # 查看具体函数的每行内存分配
# pprof> web              # 生成调用图（需要 graphviz）

# 4. 对比两次 heap profile（找增长点）
curl http://localhost:6060/debug/pprof/heap > heap1.prof   # 第1次
sleep 60
curl http://localhost:6060/debug/pprof/heap > heap2.prof   # 第2次
go tool pprof -base heap1.prof heap2.prof                  # 对比增量

# 5. goroutine 泄漏排查
curl http://localhost:6060/debug/pprof/goroutine > goroutine.prof
go tool pprof goroutine.prof
# pprof> top10            # 哪个函数创建的 goroutine 最多
```

## 5.5 OOM ★★★★★

```bash
# 查看 OOM 历史记录
dmesg -T | grep -i "out of memory"
dmesg -T | grep -i "killed process"

# 典型输出：
# [Mon Jan 1 10:00:00 2025] Out of memory: Kill process 1234 (java) score 800 or sacrifice child
# [Mon Jan 1 10:00:00 2025] Killed process 1234 (java) total-vm:2097152kB, anon-rss:1048576kB

# OOM score：内核根据内存占用计算的分数，越高越容易被杀
cat /proc/<PID>/oom_score              # 查看 OOM 分数
cat /proc/<PID>/oom_score_adj          # 查看/设置 OOM 调整值

# 保护重要进程不被 OOM killer 杀死
echo -1000 > /proc/<PID>/oom_score_adj # 从 OOM 中排除（-1000 ~ 1000，越小越不易被杀）
```

### 面试题

**Q：Linux 为什么会 OOM？**

> 物理内存 + Swap 空间全部耗尽，内核无法为进程分配所需内存时，OOM Killer 被触发。内核根据 OOM score（基于内存占用、运行时间、优先级等计算）选择一个或多个进程强制杀死以释放内存。常见原因：① 内存泄漏导致进程内存持续增长；② 内存申请过大（一次性加载大文件到内存）；③ 并发过高（每个请求占用大量内存）；④ Swap 空间配置不足。

**Q：OOM 时为什么 myapp 总是被杀？**

> 内核通过 OOM score 决定杀哪个进程。查看：`cat /proc/<PID>/oom_score`，分数越高越容易被杀。`oom_score_adj` 可手动调整（-1000 ~ 1000，越小越安全）：
> ```bash
> echo -1000 > /proc/$(pidof myapp)/oom_score_adj  # 保护关键进程（通常放 systemd unit 的 OOMScoreAdjust=-1000）
> ```

### 内存 overcommit（过度分配）机制 ★★★★

> `malloc` 为什么总返回成功但后来 OOM？这与 Linux 的 overcommit 策略有关。

```bash
# 查看当前 overcommit 策略
cat /proc/sys/vm/overcommit_memory
# 0 = 启发式（heuristic）：允许合理过度分配，内核自行判断（默认）
# 1 = 总是过度分配：malloc 几乎永不失败（最宽松，但也最容易 OOM）
# 2 = 严格控制：不超过 swap + 物理内存 * overcommit_ratio% 之和

cat /proc/sys/vm/overcommit_ratio   # 策略 2 时生效，默认 50（物理内存的 50%）
```

```
三种策略的取舍：
  策略 0（默认）- 启发式：
    malloc 大多成功，内核根据自己的逻辑判断
    缺点：规则不透明，可能某个 malloc 失败另一个成功

  策略 1 - 总是 overcommit：
    适合：科学计算、大型数据处理（申请巨大虚拟内存但不一定全用）
    缺点：OOM 风险最高

  策略 2 - 严格限制：
    承诺分配 ≤ Swap + 物理内存 * overcommit_ratio%
    适合：生产数据库（PostgreSQL 文档推荐的设置）
    缺点：可能拒绝合法的 malloc 请求
```

> **Go 开发中的关联**：容器里 overcommit 策略通常是宿主机设置的 0。容器内看到 /proc/meminfo 是宿主机的值（容易误判），实际可用内存受 cgroup 限制。Go 1.19+ 的 GOMEMLIMIT 可以绕过这些问题：设置后 Go GC 会主动在内存接近限制时触发回收，避免走到内核 OOM。

---

<a id="第六章-磁盘管理"></a>
# 第六章 磁盘管理 ★★★★

## 6.1 df 命令 ★★★★★

```bash
df -h                           # 人性化显示磁盘使用
df -h /var/log                  # 查看特定目录所在分区
df -i                           # 查看 inode 使用情况（inode 耗尽也无法创建文件）
df -T                           # 显示文件系统类型（ext4/xfs）
```

## 6.2 du 命令 ★★★★★

```bash
du -sh *                        # 当前目录下每个文件/目录的大小汇总
du -sh /var/log/*               # 查看日志目录各文件大小
du -sh * | sort -rh | head -10  # 找出最大的 10 个目录/文件
du -h --max-depth=1 /var/       # 只显示一层深度的目录大小
du -h --max-depth=1 /var/ | sort -rh

# 实际场景：磁盘满了，快速定位大文件
df -h                           # 先找到哪个分区满了
cd /data                        # 进入满的分区
du -sh * | sort -rh | head -5   # 找到最大的目录
cd 最大的目录
du -sh * | sort -rh | head -5   # 继续深入
# 重复以上步骤直到找到大文件
```

### 面试题

**Q：df 和 du 区别？**

| 对比项 | df | du |
|--------|-----|-----|
| 统计维度 | 文件系统级别 | 文件/目录级别 |
| 原理 | 读取超级块（superblock）信息 | 遍历目录树累加文件大小 |
| 速度 | 快 | 慢（需遍历） |
| 差异原因 | 文件已删除但进程仍持有句柄时，df 会多算 | du 不会统计已删除但未释放的文件 |

> **经典场景**：`df` 显示磁盘满了，但 `du` 统计出来没占那么多。原因：有进程正在向一个已删除的文件写数据（文件已从目录中删除，inode 引用计数还未归零）。解决：`lsof | grep deleted` 找到持有已删除文件的进程，重启该进程释放空间。

## 6.3 fdisk / lsblk — 磁盘分区查看

```bash
fdisk -l                        # 列出所有磁盘和分区
lsblk                           # 树状显示磁盘和分区关系
lsblk -f                        # 显示文件系统类型
```

## 6.4 iostat — 磁盘 I/O 分析 ★★★★★

```bash
iostat -x 1                     # 每秒刷新（扩展信息）
iostat -x 1 5                   # 每秒刷新，共 5 次
iostat -m                       # 以 MB 为单位

# 输出解读：
# r/s, w/s     每秒读写次数（IOPS）
# rkB/s, wkB/s  每秒读写数据量（吞吐量）
# await         平均 I/O 响应时间（ms）→ 反映磁盘性能
# %util         磁盘使用率 → 接近 100% 说明磁盘是瓶颈
# svctm         平均 I/O 服务时间

# 关键判断：
# await > 50ms → 磁盘性能较差
# await > 200ms → 磁盘严重瓶颈
# %util > 80% → 磁盘接近饱和
```

### 面试题

**Q：磁盘 I/O 高如何排查？**

```bash
# 第一步：确认 I/O 瓶颈
top                         # 看 %wa（iowait），> 10% 说明 I/O 有问题
iostat -x 1                 # 看 %util 和 await

# 第二步：找 I/O 大户进程
iotop -o                    # 只看有 I/O 的进程（需 root）
# 或者
pidstat -d 1                # 查看各进程的 I/O 统计

# 第三步：找具体文件
lsof -p <PID>               # 进程打开了哪些文件
lsof -p <PID> | grep REG    # 只看普通文件

# 第四步：Go 程序深入分析
# 使用 pprof 查看哪些函数在大量读写
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof
go tool pprof cpu.prof
```

### dd — 磁盘读写基准测试 ★★★★

```bash
# dd 可以精确控制读写方式，常用于磁盘性能测试和数据备份

# ====== 磁盘写入性能测试 ======
# 测试顺序写吞吐量（1GB 文件）
dd if=/dev/zero of=./testfile bs=1M count=1024 oflag=direct
# 输出：1073741824 bytes (1.1 GB) copied, 2.5 s, 429 MB/s

# 测试随机写（块大小为 4K，模拟数据库写入）
dd if=/dev/zero of=./testfile bs=4k count=100000 oflag=direct

# ====== 磁盘读取性能测试 ======
dd if=./testfile of=/dev/null bs=1M count=1024 iflag=direct

# ====== 参数说明 ======
# bs=1M    块大小（重要的调优参数，影响吞吐量）
# count=   写入的块数
# oflag=direct  绕过 Page Cache，测磁盘真实性能
# iflag=direct  绕过 Page Cache 读取

# ====== 实际场景 ======
# 场景1：测试磁盘性能基线
dd if=/dev/zero of=./bench bs=1M count=1024 oflag=direct 2>&1 | tail -1

# 场景2：创建 swap 文件
dd if=/dev/zero of=/swapfile bs=1M count=4096
mkswap /swapfile && swapon /swapfile

# 场景3：备份磁盘 MBR（前 512 字节）
dd if=/dev/sda of=./mbr.backup bs=512 count=1

# 场景4：快速生成指定大小的测试文件
dd if=/dev/zero of=./1GB.file bs=1M count=1024
```

---

<a id="第七章-网络管理"></a>
# 第七章 网络管理 ★★★★★

## 7.0 TCP 三次握手 & 四次挥手 ★★★★★

> TCP 连接建立和断开是面试必考题，不仅要记住过程，更要理解"为什么"。

### 三次握手（建立连接）

```
Client                                Server
  │                                    │
  │── SYN(seq=x) ───────────────────▶│  LISTEN → SYN_RCVD
  │   客户端：我要连你，我的初始序号是 x   │  服务端：收到了，你的发送能力 OK
  │                                    │
  │◀── SYN+ACK(seq=y, ack=x+1) ──────│
  │   服务端：我收到了，我的初始序号是 y    │
  │                                    │
  │── ACK(ack=y+1) ─────────────────▶│  SYN_RCVD → ESTABLISHED
  │   客户端：我也收到了，确认              │  双方建立连接
ESTABLISHED                            │
```

**为什么三次，不是两次？**

> 两次握手的问题是：服务端无法确认客户端的接收能力正常，也无法防止历史失效连接请求造成的资源浪费。
>
> **场景**：客户端发了一个旧的 SYN（在网络中滞留很久后到达），如果两次握手，服务端收到后就认为连接建立，分配资源。但客户端已经不需要这个连接了，服务端的资源就白白浪费。第三次握手让客户端告诉服务端"这个连接是我真实想要的"。

**为什么不是四次？**

> 第二步 SYN+ACK 可以合并为一次发送，没必要分开（SYN 和 ACK 各发一次），拆成四步没有额外收益。

### 四次挥手（断开连接）

```
Client                                Server
  │                                    │
  │── FIN(seq=u) ──────────────────▶│  ESTABLISHED → CLOSE_WAIT
  │   客户端：我没有数据要发了，关闭       │   服务端：收到，但我的数据可能还没发完
  │                                    │
  │◀── ACK(ack=u+1) ────────────────│
  │   服务端：确认收到你的 FIN            │
FIN_WAIT2                              │  （服务端继续发送剩余数据）
  │                                    │
  │◀── FIN(seq=v) ──────────────────│  CLOSE_WAIT → LAST_ACK
  │   服务端：我也没有数据要发了          │
  │                                    │
  │── ACK(ack=v+1) ─────────────────▶│  LAST_ACK → CLOSED
TIME_WAIT（等待 2MSL，约 60s）          │
  │  2MSL 后 CLOSED                    │
```

**为什么四次挥手？**

> TCP 是全双工的，关闭是单向的：客户端说"我不发了"，服务端说"知道了"，但服务端可能还有数据在发。等双方都不发了，各自独立关闭。所以 FIN 和 ACK 不能合并，必须有四步。

**为什么 TIME_WAIT 要等 2MSL？**

> - **MSL**（Maximum Segment Lifetime）= 报文最大存活时间，通常 30s，2MSL = 60s
> - 原因①：**确保最后的 ACK 能到达服务端**。如果服务端没收到最后的 ACK，会重发 FIN。客户端在 TIME_WAIT 期间收到重发的 FIN 会重发 ACK。
> - 原因②：**让本次连接的所有旧报文在网络中消失**。2MSL 后，所有属于本次连接的报文都已超时消失，新连接不会收到旧连接的数据。

### TIME_WAIT 过多的危害与解决

```bash
# 查看 TIME_WAIT 数量
ss -antp | grep TIME_WAIT | wc -l
ss -s  # 查看统计

# TIME_WAIT 过多的问题：
# - 连接数接近系统端口范围上限时，新连接无法建立
# - 大量 TIME_WAIT 占用内存

# 内核参数调优
# /etc/sysctl.conf 添加：
net.ipv4.tcp_tw_reuse = 1            # 客户端允许复用 TIME_WAIT 连接
net.ipv4.tcp_fin_timeout = 30        # 缩短 TIME_WAIT 等待时间
net.ipv4.tcp_max_tw_buckets = 10000  # 限制 TIME_WAIT 最大数量（超过直接关闭）
sysctl -p                            # 使配置生效
```

### CLOSE_WAIT 过多的含义（重要！）

> **CLOSE_WAIT 过多 = 代码 Bug！** 表示服务端收到了对端的 FIN，但没有调用 `close()` 来关闭自己的 socket。在 Go 中表现为：
> - `resp.Body.Close()` 没有被调用（HTTP 连接无法复用）
> - 数据库/Redis 连接没有 `Close()`
> - 某个 goroutine 卡死，持有的连接没有释放
>
> ```bash
> # 排查 CLOSE_WAIT
> ss -antp | grep CLOSE_WAIT
> ss -antp | awk '{print $NF}' | sort | uniq -c | sort -rn
>
> # 按进程统计 CLOSE_WAIT
> ss -antp | grep CLOSE_WAIT | awk '{print $NF}' | sort | uniq -c | sort -rn
> ```

### tcpdump 抓 TCP 握手挥手

```bash
# 只抓 SYN 包（排查连接问题）
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0' and port 8080

# 只抓 FIN 包
tcpdump -i eth0 'tcp[tcpflags] & (tcp-fin) != 0' and port 8080

# 抓特定 IP 的握手挥手全过程
tcpdump -i eth0 host 10.0.0.1 -nn
```

### 面试题高频追问

**Q：SYN 洪水攻击是什么？** 攻击者发送大量 SYN 包但不完成第三次握手，耗尽服务端的半连接队列（SYN queue），导致正常用户无法连接。防御：增大半连接队列 `tcp_max_syn_backlog`、启用 SYN Cookie `tcp_syncookies = 1`。

**Q：TCP KeepAlive 和 HTTP Keep-Alive 的区别？**

| 对比项 | TCP KeepAlive | HTTP Keep-Alive |
|--------|--------------|----------------|
| 所属层级 | 传输层（TCP 协议栈） | 应用层（HTTP/1.1） |
| 作用 | 检测死连接（对端崩溃但未关闭） | 复用 TCP 连接发送多个 HTTP 请求 |
| 触发时机 | 连接空闲一段时间后自动发探测包 | HTTP 响应结束后不关闭连接 |
| 默认 | 关闭，默认探测间隔 2 小时 | HTTP/1.1 默认开启 |
| Go 设置 | `net.Dialer{KeepAlive: 30s}` | `http.Transport` 默认复用连接 |

> **面试陷阱**：面试官问"KeepAlive 用过吗"，先确认问的是哪个层级。TCP KeepAlive 用于清理死连接；HTTP Keep-Alive 用于减少 TCP 握手开销（HTTP/1.1 默认启用，复用连接发多个请求）。

**Q：TCP KeepAlive 是什么？** TCP 空闲连接保活机制，默认 2 小时后发探测包。Go 中可通过 `net.Dialer` 设置：
```go
d := net.Dialer{
    KeepAlive: 30 * time.Second,  // 30s 发一次保活探测
}
```

**Q：SO_REUSEPORT 有什么作用？** 允许多个 socket 绑定同一个端口。Nginx、Go 的 `net/http` 都利用此特性实现多进程/多线程无锁共享监听，配合 epoll 实现主从 Reactor。

### TCP vs UDP 对比 ★★★★★

| 对比项 | TCP | UDP |
|--------|-----|-----|
| 连接 | 面向连接（三次握手） | 无连接 |
| 可靠性 | 可靠（确认、重传、排序、去重） | 不可靠 |
| 顺序 | 保证顺序 | 不保证 |
| 流量控制 | 有（滑动窗口） | 无 |
| 拥塞控制 | 有（慢启动、拥塞避免、快重传、快恢复） | 无 |
| 头部开销 | 20 字节 | 8 字节 |
| 适用场景 | HTTP、gRPC、MySQL、文件传输 | DNS、视频直播、VoIP、在线游戏 |
| Go 示例 | `net.Dial("tcp", addr)` | `net.Dial("udp", addr)` |

**面试追问**：

**Q：TCP 怎么保证可靠性？**
> ① 确认应答（ACK）+ 超时重传；
> ② 序列号保证顺序和去重；
> ③ 滑动窗口做流量控制；
> ④ 拥塞控制（慢启动、拥塞避免、快重传、快恢复）

**Q：DNS 为什么用 UDP？**
> DNS 查询通常一个请求一个响应，数据量小（< 512 字节）；UDP 无连接开销，查询更快；丢包客户端直接重试即可。但 DNS 区域传输（Zone Transfer）用 TCP，因为数据量大且需要可靠传输。

**Q：TCP 拥塞控制四大算法？**
```
慢启动（Slow Start）：cwnd 从 1 MSS 开始，每 RTT 翻倍
拥塞避免（Congestion Avoidance）：达到 ssthresh 后线性增长
快重传（Fast Retransmit）：收到 3 个重复 ACK 立即重传
快恢复（Fast Recovery）：不降到慢启动，降到新 ssthresh 继续
```

**Q：什么场景选 UDP？**
> ① 实时性 > 可靠性（直播、视频通话）；② 可容忍少量丢包；③ 广播/多播场景；④ 极低延迟要求（在线游戏 FPS）。

---

## 7.1 IP 查看与路由管理 ★★★★

```bash
# ====== 查看网络接口 ======
ip addr                         # 所有接口的 IP（替代 ifconfig）
ip addr show eth0               # 查看特定网卡
ip -s link show eth0            # 查看网卡流量统计（RX/TX 字节数/包数/错误数）
ip link show                    # 查看链路状态

# ====== 管理链路状态 ======
ip link set eth0 up             # 启用网卡
ip link set eth0 down           # 停用网卡

# ====== 路由管理 ======
ip route                        # 查看路由表
ip route show default           # 只看默认路由
ip route get 8.8.8.8            # 查看到 8.8.8.8 会走哪个网卡/网关
# 输出：8.8.8.8 via 192.168.1.1 dev eth0 src 192.168.1.100
#         ↑ 目标    ↑ 网关           ↑ 网卡   ↑ 本机 IP

# 添加/删除路由
ip route add 10.0.0.0/8 via 192.168.1.1    # 添加到内网网段的路由
ip route add default via 192.168.1.1        # 添加默认网关
ip route del 10.0.0.0/8                    # 删除路由

# ====== ARP 缓存（IP ↔ MAC 地址映射）======
ip neigh                        # 查看 ARP 表（替代 arp -a）
ip neigh | grep 192.168.1.1     # 查看网关的 MAC 地址
ip neigh add 10.0.0.100 lladdr aa:bb:cc:dd:ee:ff dev eth0  # 添加静态 ARP
ip neigh del 10.0.0.100 dev eth0           # 删除 ARP 条目
ip neigh flush dev eth0                    # 清除 eth0 的全部 ARP 缓存

# ====== 网络命名空间（容器网络调试）======
ip netns list                   # 列出所有 net namespace
ip netns exec <ns> ip addr      # 在指定 namespace 中执行命令
# 实际场景：不进入容器也能看容器内的网络配置
# docker inspect <id> | jq '.[0].State.Pid'  → 找到 PID
# nsenter -t <PID> -n ip addr               → 看容器内 IP

ifconfig                        # 旧命令（已废弃，推荐 ip）
route -n                        # 旧命令（已废弃，推荐 ip route）
arp -a                          # 旧命令（已废弃，推荐 ip neigh）
```

## 7.2 网络连通性 ★★★★★

### ping — 测试连通性

```bash
ping -c 4 google.com            # 发送 4 个包
ping -i 0.5 google.com          # 每 0.5 秒发送一个
ping -s 1000 google.com         # 指定包大小
```

### traceroute — 路由追踪

```bash
traceroute google.com           # 追踪到达目标经过的路由
traceroute -n google.com        # 不解析域名（更快）
```

### mtr — 综合诊断工具（ping + traceroute）

```bash
mtr google.com                  # 实时显示各跳的丢包率和延迟
```

### curl — HTTP 测试 ★★★★★

```bash
curl http://localhost:8080/health           # GET 请求
curl -v http://localhost:8080/health        # 显示详细信息（含请求头响应头）
curl -X POST http://localhost:8080/api/user -H "Content-Type: application/json" -d '{"name":"test"}'
curl -o /dev/null -s -w "%{http_code}\n" http://localhost:8080/health  # 只输出状态码
curl -o /dev/null -s -w "%{time_total}\n" http://localhost:8080/api     # 测量响应时间

# 实际场景：测试接口响应时间
curl -o /dev/null -s -w "\
HTTP Status: %{http_code}\n\
Time Total: %{time_total}s\n\
Time Connect: %{time_connect}s\n\
Time First Byte: %{time_starttransfer}s\n" http://localhost:8080/api/health
```

### 面试题

**Q：ping 原理是什么？**

> `ping` 使用 ICMP 协议（Internet Control Message Protocol）。向目标发送 ICMP Echo Request 包，目标收到后回复 ICMP Echo Reply 包。通过往返时间（RTT）判断网络延迟，通过丢包率判断网络质量。需要注意的是：防火墙可能禁止 ICMP 导致 ping 不通，但 TCP 连接可能是正常的。

### nc（netcat）— TCP/UDP 多功能网络工具 ★★★★★

```bash
# ====== 端口连通性测试 ======
nc -zv 192.168.1.100 8080                      # TCP 端口是否开放（-z 不发送数据，-v 详细信息）
nc -zv -w 3 192.168.1.100 8080                 # 超时 3 秒
nc -zv 10.0.0.1 1-65535 2>&1 | grep succeeded  # 扫描端口范围

# ====== 简易 TCP Server / Client ======
# 启动一个简单的 TCP 监听
nc -l 8080                                      # 监听 8080 端口
# 另一个终端连接并发送数据
echo "hello" | nc localhost 8080

# ====== 简易 HTTP 测试 ======
printf "GET /health HTTP/1.1\r\nHost: localhost\r\n\r\n" | nc localhost 8080

# ====== 文件传输 ======
# 接收端
nc -l 1234 > received_file.tar.gz
# 发送端
nc 192.168.1.100 1234 < file_to_send.tar.gz

# ====== UDP 测试 ======
nc -u -l 9090                                  # UDP 监听
echo "test" | nc -u localhost 9090

# ====== 实际场景 ======
# 场景1：快速验证服务端口是否可达（比 telnet 更通用）
nc -zv prod-db.internal 3306

# 场景2：服务启动前等待依赖就绪（wait-for-it 原理）
while ! nc -z mysql 3306; do sleep 1; echo "Waiting for MySQL..."; done
./myapp

# 场景3：快速启动一个 HTTP health check mock
while true; do echo -e "HTTP/1.1 200 OK\r\n\r\nOK" | nc -l -p 8080 -q 1; done
```

> **面试提示**：被问到"如何快速测试端口是否通"时，`nc -zv` 是最佳答案。它比 `telnet` 更适合脚本化（不依赖交互），比 `ping` 更准确（测试 TCP 层面，不受 ICMP 防火墙影响）。

### 面试题

## 7.3 DNS 排查 ★★★★★

```bash
# nslookup — 查询 DNS
nslookup example.com                    # 查询 A 记录
nslookup example.com 8.8.8.8           # 指定 DNS 服务器

# dig — 更详细的 DNS 查询
dig example.com                         # 查询 A 记录
dig example.com A                       # 明确查询 A 记录
dig example.com MX                      # 查询邮件记录
dig example.com +short                  # 简洁输出
dig @8.8.8.8 example.com               # 指定 DNS 服务器
dig -x 8.8.8.8                          # 反向查询

# host — 简洁的 DNS 查询
host example.com
host -t MX example.com                  # 查询特定记录类型

# 查看 DNS 配置
cat /etc/resolv.conf                    # DNS 服务器配置
cat /etc/hosts                          # 本地 hosts 文件
```

### 面试题

**Q：DNS 解析失败如何排查？**

```bash
# 1. 检查 DNS 服务器配置
cat /etc/resolv.conf

# 2. 检查本地 hosts 是否有错误映射
cat /etc/hosts | grep example.com

# 3. 测试 DNS 服务器连通性
ping 8.8.8.8

# 4. 直接查询 DNS
dig @8.8.8.8 example.com
nslookup example.com

# 5. Go 程序中使用自定义 DNS 解析器排查
# 检查 /etc/resolv.conf 是否被正确读取
# 检查 /etc/nsswitch.conf 中的 hosts 配置顺序
```

## 7.4 端口查看 ★★★★★

### netstat — 网络连接查看

```bash
netstat -antp                   # 所有 TCP 连接及进程
# -a 所有连接（含监听）
# -n 不解析域名（更快）
# -t TCP
# -p 显示进程名/PID
# -u UDP
# -l 只显示监听

netstat -antp | grep LISTEN     # 只看监听端口
netstat -antp | grep ESTABLISHED | wc -l  # 统计活跃连接数
netstat -s                      # 网络统计信息（丢包、重传）
```

### ss — 更快的 netstat ★★★★★

```bash
ss -lntp                        # 所有监听 TCP 端口及进程
ss -antp                        # 所有 TCP 连接
ss -s                           # 连接数量统计汇总
ss -antp | grep TIME_WAIT | wc -l  # TIME_WAIT 连接数
ss state time-wait | wc -l      # 同上
ss -o state established '( sport = :8080 )'  # 特定端口的 ESTABLISHED 连接
```

### 面试题

**Q：netstat 和 ss 区别？**

> `ss` 是 `netstat` 的现代替代品。`ss` 直接从内核获取 socket 信息（通过 netlink），速度更快，信息更详细；`netstat` 读取 `/proc/net/*` 文件，数据量大时性能差。查看连接时优先用 `ss`。

## 7.5 lsof — 查看进程打开的文件 ★★★★★

```bash
lsof -i :8080                           # 查看占用 8080 端口的进程
lsof -i tcp:8080                        # 只看 TCP
lsof -p <PID>                           # 查看进程打开的所有文件
lsof -p <PID> | grep REG                # 进程打开的普通文件
lsof -p <PID> | grep TCP                # 进程打开的网络连接
lsof -u nginx                           # nginx 用户打开的文件
lsof +D /var/log                        # 打开了 /var/log 下文件的进程
lsof | grep deleted                     # 查找已删除但仍占用的文件（磁盘空间未释放）
```

### 面试题

**Q：如何查看端口被哪个进程占用？**

```bash
# 方法1：ss（推荐）
ss -lntp | grep :8080

# 方法2：lsof
lsof -i :8080

# 方法3：netstat
netstat -lntp | grep :8080

# 方法4：fuser
fuser 8080/tcp
```

## 7.6 tcpdump — 抓包利器 ★★★★★

```bash
# ====== 基础抓包 ======
tcpdump -i eth0                             # 抓取 eth0 网卡的所有包
tcpdump -i any port 8080                    # 抓取 8080 端口的包
tcpdump -i eth0 host 192.168.1.100          # 抓取和特定 IP 的包
tcpdump -i eth0 port 80 or port 443         # 抓取 80 和 443 端口
tcpdump -i eth0 src host 10.0.0.1           # 只抓源 IP
tcpdump -i eth0 dst host 10.0.0.1           # 只抓目标 IP

# ====== 保存和读取 ======
tcpdump -i eth0 port 8080 -w capture.pcap   # 保存到文件
tcpdump -r capture.pcap                     # 读取文件
tcpdump -r capture.pcap -nn -A              # 以 ASCII 显示内容

# ====== 常用参数 ======
# -n    不解析域名（关键！否则可能引发新的 DNS 请求）
# -nn   域名和端口都不解析
# -v   详细信息
# -vv  更详细信息
# -X   显示包内容（Hex + ASCII）
# -A   以 ASCII 显示
# -c   抓取包数量
# -s   抓包长度（0=全包）

# ====== 实际场景 ======
# 场景1：抓取 HTTP 请求
tcpdump -i eth0 -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

# 场景2：抓取特定主机间的 MySQL 流量
tcpdump -i eth0 -s 0 host 192.168.1.100 and port 3306 -w mysql.pcap

# 场景3：抓取 DNS 请求
tcpdump -i eth0 -n port 53

# 场景4：抓取特定端口的 SYN 包（排查连接问题）
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0' and port 8080

# 场景5：抓取 Go 服务间的 gRPC 通信
tcpdump -i eth0 port 9090 -w grpc.pcap
```

### 面试题

**Q：tcpdump 如何抓取 HTTP 请求？**

```bash
tcpdump -i eth0 -A -s 0 'tcp port 80'
# -A 以 ASCII 格式显示内容（可以看到 HTTP 头和 body）
# -s 0 抓取完整数据包（不截断）
```

---

<a id="第八章-日志分析"></a>
# 第八章 日志分析 ★★★★★

## 8.1 实时查看日志

```bash
tail -f app.log                                     # 实时追踪
tail -n 100 -f app.log                              # 先显示 100 行，再追踪
tail -f app.log | grep -E "ERROR|WARN"               # 过滤错误和警告
tail -f app.log | grep --line-buffered "ERROR"       # 行缓冲模式
tail -f /var/log/app/*.log                          # 追踪目录下所有日志
```

## 8.2 grep 日志分析

```bash
# 搜索错误
grep "ERROR" app.log | tail -20
grep -n "panic" app.log                             # 显示行号
grep -C 10 "panic" app.log                          # 显示上下文
grep "2025-01-01 10:" app.log | grep "ERROR"        # 特定时间段的错误
grep -v "DEBUG" app.log | grep "ERROR"              # 排除 DEBUG 行
```

## 8.3 awk 日志分析 ★★★★★

### 实际场景：Nginx Access Log 分析

Nginx 日志格式：
```
192.168.1.1 - - [01/Jan/2025:10:00:00 +0800] "GET /api/user HTTP/1.1" 200 1234 "-" "curl/7.68.0" 0.005
$1=IP  $4=时间  $6=方法  $7=URL  $9=状态码  $10=响应大小  $NF=响应时间(最后字段)
```

```bash
# 场景1：统计 TOP10 访问 IP
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# 场景2：统计 TOP10 接口
awk '{print $7}' access.log | sort | uniq -c | sort -rn | head -10

# 场景3：统计各状态码数量
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# 场景4：统计 5xx 错误最多的接口
awk '$9 ~ /^5/' access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head

# 场景5：统计慢请求（响应时间 > 1 秒）
awk '$NF > 1 {print $NF, $7}' access.log | sort -rn | head -20

# 场景6：统计各接口平均响应时间
awk '{sum[$7]+=$NF; count[$7]++} END {for(url in sum) printf "%.3f %d %s\n", sum[url]/count[url], count[url], url}' access.log | sort -rn | head

# 场景7：统计每小时请求量
awk '{print substr($4,2,14)}' access.log | cut -d: -f2 | sort | uniq -c

# 场景8：统计 UV（独立 IP 数）
awk '{print $1}' access.log | sort -u | wc -l

# 场景9：统计 PV（页面访问量）
wc -l access.log

# 场景10：统计流量消耗
awk '{sum += $10} END {printf "%.2f GB\n", sum/1024/1024/1024}' access.log
```

### 实际场景：Go 应用日志分析

```bash
# Go 日志格式示例：
# 2025-01-01T10:00:00.123Z [INFO] user.go:42 login success user_id=12345 cost=5ms

# 统计各级别日志数量
awk '{print $3}' app.log | sort | uniq -c

# 统计最慢的接口
grep "cost=" app.log | grep -oP 'handler=\K\S+' | sort | uniq -c
grep "cost=" app.log | grep -oP 'cost=\K[0-9]+' | sort -rn | head

# 统计错误最多的文件
grep "ERROR" app.log | grep -oP '\S+\.go:\d+' | sort | uniq -c | sort -rn | head
```

### 面试题

**Q：如何统计访问量最高的 IP？**

```bash
# 方法1：awk + sort + uniq（最常用）
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# 方法2：纯 awk
awk '{ips[$1]++} END {for(ip in ips) print ips[ip], ip}' access.log | sort -rn | head -10

# 方法3：超大日志文件（先过滤再统计）
grep "GET /api/" access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
```

## 8.4 logrotate — 日志轮转 ★★★★★

> 生产环境必须配置日志轮转，否则日志文件会无限膨胀，撑满磁盘。

### 配置文件

```bash
# 主配置：/etc/logrotate.conf
# 各服务配置：/etc/logrotate.d/ 下的独立文件

# 示例：/etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily                       # 每天轮转
    rotate 30                   # 保留 30 份历史日志
    missingok                   # 日志文件不存在也不报错
    notifempty                  # 空文件不轮转
    compress                    # 压缩历史日志（gzip）
    delaycompress               # 昨天的日志今天才压缩（给 tail -f 留一天缓冲）
    dateext                     # 用日期而非数字命名（如 app.log-20250101.gz）
    dateformat -%Y%m%d
    copytruncate                # 复制后截断原文件（适合不可重载的程序）
    # create 640 appuser appuser  # 创建新文件并设权限（或 copytruncate 二选一）
    maxsize 100M                # 超过 100M 也触发轮转
    postrotate
        # 轮转后执行（如通知进程重新打开日志文件）
        systemctl reload myapp || true
    endscript
}
```

### copytruncate vs create

> - **`copytruncate`**：复制原文件 → 清空原文件（inode 不变，进程无需重新打开）。适合不能发送信号的进程，但可能丢失轮转瞬间的日志。
> - **`create`**：创建新文件，旧文件改名。需要 `postrotate` 中通知进程重新打开文件（发信号或用 `systemctl reload`）。Go 程序如果收到 SIGHUP 重新打开日志文件则用此方式。

### 手动触发与调试

```bash
logrotate -d /etc/logrotate.d/myapp    # debug 模式（不真正执行）
logrotate -f /etc/logrotate.d/myapp    # 强制执行（忽略时间条件）
logrotate -v /etc/logrotate.d/myapp    # 显示详细过程
cat /var/lib/logrotate/status          # 查看各日志上次轮转时间
```

---

<a id="第九章-性能分析工具"></a>
# 第九章 性能分析工具 ★★★★★

## 9.1 uptime — 系统负载

```bash
uptime
# 输出：10:00:00 up 30 days, 2:00, 5 users, load average: 0.52, 0.78, 1.05

# load average：1/5/15 分钟的平均活跃进程数(R状态 + D状态)
# > CPU 核数 → 系统过载
# 例如 4 核 CPU，load average 为 6 → 有 2 个进程在排队等待 CPU
```

### 面试题

**Q：Load Average 是什么？CPU 使用率和 Load 的区别？**

> **Load Average** 是 1/5/15 分钟内的**平均活跃进程数**，包括正在使用 CPU（R 状态）和等待 I/O（D 状态）的进程。**CPU 使用率**是 CPU 繁忙程度的百分比（0-100%）。
>
> - CPU 使用率高 + Load 不高 → CPU 密集型任务，系统能处理
> - Load 高 + CPU 使用率不高 → I/O 瓶颈（大量进程等待磁盘/网络）
> - 两者都高 → CPU 瓶颈，需要扩容或优化
>
> **经验值**：Load Average 持续超过 CPU 核数，说明系统已过载，有进程在排队。超过核数的 1.5 倍，需要立即关注。

## 9.2 vmstat — 虚拟内存统计

```bash
vmstat 1                         # 实时监控
# 关注：r(运行队列长度)、so(swap out)、wa(iowait)、cs(上下文切换)
```

## 9.3 iostat — 磁盘 I/O 分析

```bash
iostat -x 1                      # 重点关注 %util 和 await
iostat -x 1 | awk '$1=="sda"'    # 只看特定磁盘
```

## 9.4 sar — 历史性能分析

```bash
sar -u                          # CPU 历史
sar -r                          # 内存历史
sar -n DEV                      # 网络历史
sar -q                          # 负载历史
sar -f /var/log/sa/sa01         # 查看特定日期的数据
```

## 9.5 pidstat — 进程性能分析

```bash
pidstat 1                       # 各进程 CPU 使用
pidstat -r 1                    # 各进程内存使用
pidstat -d 1                    # 各进程 I/O 统计
pidstat -t -p <PID> 1           # 特定进程的各线程 CPU 使用
```

### 上下文切换 — vmstat cs 列深度分析 ★★★★

> 上下文切换（Context Switch）是操作系统中最基础的概念之一，也是性能分析的关键指标。

**什么是上下文切换？**

```
上下文切换：CPU 从执行一个进程/线程切换到另一个进程/线程

切换时需要保存/恢复的"上下文"：
  - CPU 寄存器（通用寄存器、PC 程序计数器、SP 栈指针）
  - 页表基址寄存器（CR3）→ 地址空间切换（进程切换独有）
  - FPU/SSE 寄存器（浮点寄存器，延迟保存）
  - TLB 刷新（进程切换才需要，代价最大）

三种切换：
  1. 进程切换：切换地址空间 → 最贵（~1-10µs）
  2. 线程切换：共享地址空间 → 中等（~1-2µs）
  3. 协程切换：用户态 → 极快（~几十ns，如 goroutine 切换）
```

**vmstat 中的 cs 列**

```bash
vmstat 1
# cs (context switch) = 每秒上下文切换次数

# 经验值：
# cs < 10000/s  → 正常
# cs 10000-50000/s → 偏高，需关注
# cs > 50000/s → 过高，性能问题

# 实际场景：查看各进程的上下文切换
pidstat -w 1
# cswch/s  = 自愿上下文切换（voluntary）
# nvcswch/s = 非自愿上下文切换（involuntary）
```

**什么时候触发上下文切换？**

```
自愿切换（voluntary ctx switch）：
  - 进程主动调用 sleep()、wait()、yield()
  - 等待 I/O（read/write 阻塞）
  - 等待锁（mutex lock）
  - 等待管道/信号量
  → 进程自己让出 CPU，正常行为

非自愿切换（involuntary ctx switch）：
  - 时间片用完（CFS 调度器强制切换）
  - 更高优先级进程就绪（抢占）
  - 中断处理唤醒高优先级进程
  → 被内核抢走 CPU，说明 CPU 时间片不够用
```

**上下文切换过多的排查**

```bash
# 1. 看 cs 是否过高
vmstat 1 5
# cs 列 > 50000 → 需要排查

# 2. 找切换最多的进程
pidstat -w 1
# 关注 nvcswch/s 高的进程

# 3. Go 程序中常见原因
# a) 大量 goroutine 被阻塞在同一个锁上（mutex 竞争）
curl http://localhost:6060/debug/pprof/mutex > mutex.prof
go tool pprof mutex.prof

# b) channel 操作过多，runtime 频繁调度 goroutine
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof
go tool pprof cpu.prof
# 看 runtime.schedule 函数占比

# c) 系统调用过多（strace -c -p <PID> 查看）
# d) GOMAXPROCS 设置不合理（过多 P 导致频繁抢占）
```

**Go 中上下文切换的特殊性**

```
Goroutine 切换 vs OS 线程切换：
  1 个 OS 线程切换 = ~µs 级别
  1 个 goroutine 切换 = ~几十 ns（用户态，只切几个寄存器）

Go 的 GMP 模型减少 OS 线程切换：
  M（OS 线程）绑定到 P（处理器）后，
  在用户态切换 G（goroutine），
  不需要陷入内核 → 极快

但 goroutine 过多也会带来问题：
  - 调度开销（Go scheduler 需要时间去选下一个 G）
  - 内存开销（每个 G 有栈）
  - 建议：GOMAXPROCS = CPU 核数，不要过大
```

---

## 9.6 strace — 系统调用跟踪 ★★★★★

```bash
strace -p <PID>                                 # 跟踪进程的系统调用
strace -c -p <PID>                              # 统计各系统调用的次数和时间
strace -e trace=read,write -p <PID>             # 只看 read/write 调用
strace -e trace=network -p <PID>                # 只看网络相关调用
strace -e trace=file -p <PID>                   # 只看文件相关调用
strace -T -p <PID>                              # 显示每次调用耗时
strace -f -p <PID>                              # 跟踪子进程

# 实际场景：排查 Go 进程卡死
strace -p <PID>
# 如果卡在 futex() → 锁竞争/等待
# 如果卡在 read() → 等待数据
# 如果卡在 epoll_wait() → 正常等待连接
```

### 面试题

**Q：strace 有什么作用？**

> `strace` 跟踪进程发出的系统调用（System Call），是排查进程异常的最底层工具。典型场景：① 进程卡死不响应（看卡在哪个系统调用上）；② 进程启动失败但无日志（看打开哪个文件失败）；③ 数据库操作慢（看 send/recv 的耗时）；④ Go 程序无法用 pprof 时的兜底排查手段。注意：strace 有性能开销，生产环境谨慎使用。

## 9.7 perf — CPU 热点分析 ★★★★★

```bash
# 采样 CPU 性能数据
perf top                                        # 实时查看 CPU 热点
perf top -p <PID>                               # 只看特定进程
perf record -p <PID> -g --sleep 30              # 录制 30 秒（-g 记录调用栈）
perf report                                     # 分析录制结果

# 生成火焰图（需要 Brendan Gregg 的脚本）
perf record -p <PID> -g --sleep 30
perf script > out.perf
stackcollapse-perf.pl out.perf > out.folded
flamegraph.pl out.folded > flamegraph.svg

# Go 程序推荐直接用 pprof 而不是 perf
# pprof 是 Go 原生工具，能看到函数名而不是汇编地址
```

### 面试题

**Q：perf 如何定位 CPU 瓶颈？**

> 1. `perf top` 查看实时热点函数；2. `perf record -p <PID> -g --sleep 30` 采集 30 秒；3. `perf report` 分析结果，看哪些函数消耗 CPU 最多；4. 生成火焰图直观定位。但 Go 程序推荐用 pprof（`go tool pprof`），因为 perf 看到的是编译后的汇编，pprof 能看到 Go 函数名和调用栈，更直观。

---

<a id="第十章-shell-基础"></a>
# 第十章 Shell 基础 ★★★★

## 10.1 Shell 变量

```bash
#!/bin/bash

# 定义变量
NAME="world"
readonly PI=3.14            # 只读变量
export GOPATH=/home/user/go # 环境变量

# 使用变量
echo "Hello, ${NAME}"
echo "${NAME}_suffix"       # 变量名边界需要用 {}
echo '${NAME}'              # 单引号不解析变量，输出 ${NAME}

# 特殊变量
echo $0                     # 脚本名
echo $1 $2 $3              # 脚本参数
echo $#                     # 参数个数
echo $@                     # 所有参数（分开的字符串）
echo $*                     # 所有参数（一个字符串）
echo $?                     # 上一条命令的退出码（0=成功）
echo $$                     # 当前 shell 的 PID
echo $!                     # 最后一个后台进程的 PID
```

### 实用 Shell 小工具

```bash
# ====== basename — 提取文件名 ======
basename /path/to/file.txt          # file.txt
basename /path/to/file.txt .txt     # file（去掉后缀）

# ====== dirname — 提取目录 ======
dirname /path/to/file.txt           # /path/to

# ====== mktemp — 安全创建临时文件/目录 ======
mktemp                              # /tmp/tmp.XXXXXX（随机名，避免竞态）
mktemp -d                           # 创建临时目录
tempfile=$(mktemp /tmp/myscript.XXXXXX)  # 指定前缀
trap "rm -f $tempfile" EXIT         # 脚本退出时自动清理

# ====== 实际场景：脚本中安全使用临时文件 ======
#!/bin/bash
OUTPUT=$(mktemp /tmp/process.XXXXXX)
trap "rm -f $OUTPUT" EXIT
grep "ERROR" /var/log/app.log > "$OUTPUT"
# ... 处理 $OUTPUT ...

# 用 basename/dirname 处理路径
SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd)  # 获取脚本所在目录（常见写法）
CONFIG="$SCRIPT_DIR/../config/app.yaml"
```

## 10.2 条件判断

```bash
# if 语句
if [ -f "/etc/nginx/nginx.conf" ]; then
    echo "Nginx config exists"
elif [ -d "/etc/nginx/conf.d" ]; then
    echo "Nginx conf.d exists"
else
    echo "Not found"
fi

# 文件测试
[ -f file ]     # 是否为普通文件
[ -d dir ]      # 是否为目录
[ -e path ]     # 是否存在
[ -r file ]     # 是否可读
[ -w file ]     # 是否可写
[ -x file ]     # 是否可执行
[ -s file ]     # 是否存在且非空
[ -L file ]     # 是否为符号链接

# 字符串测试（推荐用 [[ ]]，更安全）
[[ -z "$STR" ]]         # 字符串为空
[[ -n "$STR" ]]         # 字符串非空
[[ "$A" == "$B" ]]      # 字符串相等
[[ "$A" != "$B" ]]      # 字符串不等

# 数字测试
[ $a -eq $b ]           # 等于
[ $a -ne $b ]           # 不等于
[ $a -lt $b ]           # 小于
[ $a -gt $b ]           # 大于
[ $a -le $b ]           # 小于等于
[ $a -ge $b ]           # 大于等于

# case 语句
case "$ACTION" in
    start)
        echo "Starting..."
        ;;
    stop)
        echo "Stopping..."
        ;;
    *)
        echo "Usage: $0 {start|stop}"
        exit 1
        ;;
esac
```

## 10.3 循环

```bash
# for 循环
for i in {1..5}; do
    echo $i
done

for file in /var/log/*.log; do
    echo "Processing: $file"
    gzip "$file"
done

# 遍历命令输出
for pid in $(ps aux | grep nginx | awk '{print $2}'); do
    echo "Nginx PID: $pid"
done

# while 循环
COUNT=0
while [ $COUNT -lt 5 ]; do
    echo $COUNT
    COUNT=$((COUNT+1))
done

# 逐行读取文件
while IFS= read -r line; do
    echo "$line"
done < file.txt
```

## 10.4 函数

```bash
# 定义函数
log_info() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO] $1"
}

log_error() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [ERROR] $1" >&2
}

# 带返回值的函数
check_port() {
    local port=$1
    if ss -lntp | grep -q ":$port "; then
        return 0    # 端口被占用
    else
        return 1    # 端口空闲
    fi
}

# 带输出返回值的函数
get_process_count() {
    local name=$1
    local count=$(ps aux | grep "$name" | grep -v grep | wc -l)
    echo $count
}

# 使用函数
log_info "Starting deployment..."
if check_port 8080; then
    log_error "Port 8080 is already in use"
    exit 1
fi
```

## 10.5 常见 Shell 脚本题

### 脚本1：批量重命名

```bash
#!/bin/bash
# 批量将 .txt 重命名为 .md
for file in *.txt; do
    mv "$file" "${file%.txt}.md"
done
```

### 脚本2：日志清理

```bash
#!/bin/bash
# 清理 30 天前的日志文件
LOG_DIR="/var/log/app"
find "$LOG_DIR" -name "*.log" -mtime +30 -exec rm -f {} \;
echo "Cleaned logs older than 30 days from $LOG_DIR"
```

### 脚本3：服务健康检查

```bash
#!/bin/bash
# 检查服务是否正常，异常则重启
SERVICE="myapp"
URL="http://localhost:8080/health"
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "$URL")

if [ "$HTTP_CODE" != "200" ]; then
    echo "[$(date)] Service $SERVICE returned $HTTP_CODE, restarting..."
    systemctl restart "$SERVICE"
fi
```

### 脚本4：自动备份

```bash
#!/bin/bash
BACKUP_DIR="/backup"
APP_DIR="/opt/myapp"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/myapp_$DATE.tar.gz"

tar -czf "$BACKUP_FILE" "$APP_DIR"
echo "Backup created: $BACKUP_FILE"

# 只保留最近 7 天的备份
find "$BACKUP_DIR" -name "myapp_*.tar.gz" -mtime +7 -delete
```

---

<a id="第十一章-linux-高并发与服务器开发"></a>
# 第十一章 Linux 高并发与服务器开发 ★★★★★

## 11.1 文件描述符（FD）

### 什么是 FD

> 文件描述符（File Descriptor）是一个整数，是内核为每个进程维护的**打开文件句柄表**的索引。0=stdin, 1=stdout, 2=stderr。在 Linux 中一切皆文件，网络连接（socket）也占用 FD。

```bash
# 查看 FD 限制
ulimit -n                       # 当前用户最大打开文件数
ulimit -n 65535                 # 临时修改

# 永久修改：/etc/security/limits.conf 添加：
# *    soft    nofile    65535
# *    hard    nofile    65535

# 查看进程的 FD 使用
ls /proc/<PID>/fd/ | wc -l      # 进程打开的 FD 数量
ls -la /proc/<PID>/fd/           # 查看具体打开了什么

# 系统级最大 FD 数
cat /proc/sys/fs/file-max
```

### 面试题

**Q：文件描述符是什么？高并发服务为什么要调整 FD 限制？**

> FD 是进程打开的文件的整数句柄。默认限制通常为 1024，但对于高并发服务器（如 Nginx、Go HTTP 服务），每个客户端连接就是一个 FD。如果有 10000 个并发连接，就需要 10000+ 个 FD。超过限制会导致 `accept()` 返回 `EMFILE`（too many open files），新连接无法建立。

### Go 中设置 FD 限制

```go
import "syscall"

func main() {
    var rLimit syscall.Rlimit
    err := syscall.Getrlimit(syscall.RLIMIT_NOFILE, &rLimit)
    if err != nil {
        panic(err)
    }
    rLimit.Cur = 65535
    rLimit.Max = 65535
    err = syscall.Setrlimit(syscall.RLIMIT_NOFILE, &rLimit)
    if err != nil {
        panic(err)
    }
}
```

## 11.1.5 五种 I/O 模型 ★★★★★

> 理解五种 I/O 模型是理解为什么需要 epoll 的前提。整个过程分两个阶段：① 等待数据到达内核缓冲区；② 将数据从内核缓冲区拷贝到用户空间。

```
1. 阻塞 I/O（Blocking I/O）
   用户进程: recv() ────阻塞等待─────────────▶ 返回
   内核:              等待数据 → 数据到达 → 拷贝到用户空间
   特点：两个阶段都阻塞，一个线程只能处理一个连接
   Go 中 conn.Read() 看起来是阻塞的，但底层 runtime 用 epoll 做了异步化

2. 非阻塞 I/O（Non-blocking I/O）
   用户进程: recv() → EAGAIN → recv() → EAGAIN → ... → recv() → 就绪 → 返回
   特点：第一阶段不阻塞但需要不断轮询，空耗 CPU
   不推荐使用，已被 I/O 多路复用替代

3. I/O 多路复用（I/O Multiplexing）← 最重要的模型
   用户进程: epoll_wait(fd集合) ─阻塞等待─▶ 某fd就绪 → recv() → 返回
   特点：一个线程监听多个 fd，有数据才处理，不浪费 CPU
   Nginx、Redis、Go netpoller 都基于此模型

4. 信号驱动 I/O（Signal-driven I/O）
   用户进程: 注册 SIGIO → 继续执行 → 收到信号 → recv()
   特点：第一阶段异步，第二阶段仍同步拷贝
   实际很少使用（信号处理复杂，性能不如 epoll）

5. 异步 I/O（Asynchronous I/O, AIO）
   用户进程: aio_read() → 立即返回 → 收到通知（数据已在用户缓冲区）
   特点：两个阶段全异步，真正的异步
   Linux 传统 AIO 支持有限，io_uring（Linux 5.1+）才是真正的异步 I/O
```

**五种 I/O 模型对比**：

| 模型 | 第一阶段 | 第二阶段 | 代表技术 |
|------|:---:|:---:|------|
| 阻塞 I/O | 阻塞 | 阻塞 | 传统 socket（一个连接一个线程）|
| 非阻塞 I/O | 轮询 | 阻塞 | 已过时 |
| I/O 多路复用 | 阻塞（监听多 fd） | 阻塞 | epoll（Nginx/Redis/Go）|
| 信号驱动 I/O | 异步 | 阻塞 | SIGIO（极少使用）|
| 异步 I/O | 异步 | 异步 | io_uring（Linux 5.1+）|

### io_uring — 下一代异步 I/O ★★★

> io_uring 是 Linux 5.1 引入的真正异步 I/O 接口，解决 AIO 的局限性。epoll 解决了"等数据"阶段，但读数据仍需同步系统调用；io_uring 通过**共享环形缓冲区**实现完全的零系统调用 I/O。

```
io_uring 核心原理：
  用户态和内核态共享两个环形缓冲区：
  
  SQ（Submission Queue，提交队列）
    用户进程将 I/O 请求放入 SQ（无需系统调用！）
    内核从 SQ 读取请求并执行
  
  CQ（Completion Queue，完成队列）
    内核将完成结果放入 CQ
    用户进程从 CQ 读取结果（无需系统调用！）

  关键优势：
    - 批量提交：一次放多个 I/O 请求，一次处理多个完成事件
    - 零系统调用：通过 mmap 共享内存，不走 read/write/futex
    - 真正的异步：不像 epoll 只是"等到数据来了告诉我"，而是"帮我把数据搬到我指定的地方"
    - 支持所有 I/O 类型：文件、网络、splice、sendfile 等
```

> **Go 中的 io_uring**：目前 Go 标准库尚未使用 io_uring（因为需要广泛的生态系统兼容性测试），但已有实验性项目如 `iceber/iouring-go`。如果未来 Go netpoller 迁移到 io_uring，网络 I/O 性能将再次显著提升。校招面试了解概念即可，能说出"这是 Linux 新一代异步 I/O，用共享环形缓冲区实现零系统调用"就很加分。

> **面试答法**：Linux 有五种 I/O 模型。生产环境统一用 I/O 多路复用（epoll）。Go 的 netpoller 底层用 epoll 实现，但对开发者暴露同步阻塞 API（`conn.Read()`），runtime 将阻塞的 goroutine 挂起，fd 就绪后再唤醒——"同步写法，异步执行"。这也是 Go 网络编程的核心优势。Linux 5.1+ 的 io_uring 是下一代方案，通过共享环形缓冲区实现零系统调用的真正异步 I/O。

---

## 11.2 select — I/O 多路复用（旧方案）

```c
// select 工作流程：
// 1. 用户进程构造 fd_set 位图
// 2. 调用 select()，位图拷贝到内核
// 3. 内核遍历所有 fd，检查是否就绪
// 4. 有就绪或超时则返回
// 5. 用户进程遍历所有 fd 找到就绪的
```

**缺点**：
- FD 数量限制：默认 1024（`FD_SETSIZE`）
- 每次调用需要拷贝整个 fd_set 到内核
- O(n) 遍历，连接数多时性能差
- 每次调用返回后需要遍历所有 fd

## 11.3 poll — select 的改进版

```c
// poll 使用 pollfd 数组代替位图
// 优势：无 1024 限制（用链表实现）
// 缺点：仍然是 O(n) 遍历，连接数多时性能差
```

## 11.4 epoll — 高性能 I/O 多路复用 ★★★★★

### 三个核心系统调用

```
epoll_create(size)     → 创建 epoll 实例
                         内核创建：红黑树（存所有监听 fd）+ 就绪链表

epoll_ctl(epfd, op, fd, event)  → 注册/修改/删除监听的 fd
                                   将 fd 加入红黑树

epoll_wait(epfd, events, maxevents, timeout)  → 等待就绪事件
                                                 从就绪链表取结果，O(1)
```

### epoll 内部数据结构

```
epoll 实例
┌───────────────────────────────┐
│ 红黑树（rbtree）              │  ← 存储所有监听的 fd
│    fd1 → (事件, 回调函数)     │     epoll_ctl 操作
│    fd2 → (事件, 回调函数)     │
│    fd3 → (事件, 回调函数)     │
│    ...                        │
│                               │
│ 就绪链表（rdllist）           │  ← fd 就绪时加入
│    fd2 → fd5                  │     epoll_wait 从这里取
└───────────────────────────────┘

事件驱动流程：
  数据到达网卡 → 内核处理 → fd 变为可读
  → 内核执行该 fd 在红黑树中的回调函数
  → 回调将该 fd 加入就绪链表
  → epoll_wait 被唤醒，从链表取结果
```

### LT（水平触发）vs ET（边缘触发）

| 对比项 | LT（Level Triggered） | ET（Edge Triggered） |
|--------|----------------------|---------------------|
| 通知时机 | fd 有数据就通知 | fd 状态变化时通知一次 |
| 数据未读完 | 下次 epoll_wait 继续通知 | 不再通知直到新数据到达 |
| 编程复杂度 | 简单 | 复杂（必须循环读到 EAGAIN） |
| 适用场景 | 通用场景 | 高性能场景（Nginx） |
| 默认 | 是 | 否（需设置 EPOLLET） |

```c
// ET 模式的标准写法：循环读取直到返回 EAGAIN
while (1) {
    n = read(fd, buf, sizeof(buf));
    if (n == -1) {
        if (errno == EAGAIN) break;  // 已无数据
        // 处理错误
        break;
    }
    if (n == 0) {
        // 对端关闭连接
        break;
    }
    // 处理数据
}
```

### 面试题（必考）

**Q：select、poll、epoll 区别？**

| 对比项 | select | poll | epoll |
|--------|--------|------|-------|
| FD 数量限制 | 1024 | 无 | 无 |
| 数据结构 | 位图 | 链表 | 红黑树 + 就绪链表 |
| 时间复杂度 | O(n) | O(n) | O(1) |
| FD 拷贝 | 每次调用都拷贝 | 每次调用都拷贝 | 只注册时拷贝一次 |
| 就绪通知 | 遍历全部 | 遍历全部 | 只返回就绪 fd |
| 适用场景 | 连接少 | 连接少 | 高并发 |

**Q：epoll 为什么快？**

> ① **数据结构**：用红黑树管理 fd，增删查 O(log n)；② **事件驱动**：fd 就绪时内核通过回调直接加入就绪链表，epoll_wait 只需从链表取结果，O(1)；③ **减少拷贝**：fd 只在 epoll_ctl 注册时拷贝一次到内核，不像 select/poll 每次调用都拷贝；④ **增量返回**：只返回就绪的 fd，用户不需要遍历全部 fd。

## 11.5 Reactor 模型 ★★★★★

```
单 Reactor 单线程：
  Reactor + Acceptor + Handler 都在一个线程
  代表：Redis（6.0 前）
  优势：简单，无锁
  劣势：不能利用多核

单 Reactor 多线程：
  Reactor + Acceptor 在主线程
  Handler（业务处理）在线程池
  优势：业务处理并发
  劣势：Reactor 可能成为瓶颈

主从 Reactor（多 Reactor 多线程）：
  Main Reactor（1个）：处理 accept，分配连接
  Sub Reactor（N个）：每个处理一个连接集合的 I/O
  代表：Nginx、Netty、Go netpoll
  优势：全并发，性能最高
```

### Go 的 netpoller（基于 epoll）

Go runtime 在 Linux 上使用 epoll 实现网络 I/O 多路复用。当 goroutine 执行网络 I/O 阻塞时，runtime 将其挂起，把 fd 注册到 epoll；fd 就绪后，runtime 唤醒对应 goroutine。整个过程对开发者透明，开发者写的是同步代码，运行时用异步 I/O 执行。

```go
// 这段代码看起来是同步的，但 Go runtime 底层用 epoll 实现
// 当 conn.Read 阻塞时，runtime 将 goroutine 挂起，不会阻塞 OS 线程
func handleConn(conn net.Conn) {
    buf := make([]byte, 1024)
    for {
        n, err := conn.Read(buf)   // 看起来是阻塞读，实际底层是用 epoll 的异步 I/O
        if err != nil {
            return
        }
        // 处理数据...
    }
}
```

### TCP_NODELAY — Nagle 算法与低延迟 ★★★★★

> Nagle 算法是 TCP 的默认行为：发送端会尽量把小数据包合并成一个大的 MSS 包再发送，以减少网络上的小包数量。但这会导致高延迟——对于延迟敏感的交互式应用（如游戏、实时通信、HTTP API）是灾难。

```
Nagle 算法的工作方式：
  发了一个小包 → 等 ACK 回来 → 把积累的小数据和新数据打包成一个包发出去
  如果 ACK 一直不回来 → 下一个包最多等 200ms（Nagle 超时）

典型场景：HTTP 请求头刚好超过 MSS，剩一个字节在第二个包
  → 必须等第一个包的 ACK 回来或 200ms → 凭空增加 40-200ms 延迟
```

```go
// Go 中禁用 Nagle 算法（设置 TCP_NODELAY）
import "net"

func dialWithNoDelay() {
    d := net.Dialer{
        Timeout:   5 * time.Second,
        KeepAlive: 30 * time.Second,
    }
    conn, _ := d.Dial("tcp", "example.com:8080")
    
    // 对 *net.TCPConn 设置 TCP_NODELAY
    if tcpConn, ok := conn.(*net.TCPConn); ok {
        tcpConn.SetNoDelay(true)  // 禁用 Nagle → 立即发送，不等待合并
    }
}

// 或者直接用 tcp 包：
// tcpAddr, _ := net.ResolveTCPAddr("tcp", "example.com:8080")
// conn, _ := net.DialTCP("tcp", nil, tcpAddr)
// conn.SetNoDelay(true)
```

> **面试答法**：Nagle 算法为了减少小包打包发送，但会增加延迟。Go 默认不修改 Nagle 设置（依赖操作系统默认，通常开启）。对于 HTTP API、gRPC 等延迟敏感场景，应该 `SetNoDelay(true)`。注意 `net/http` 的 Transport 对 localhost 连接默认禁用 Nagle，但对外部地址不保证。

### SO_REUSEADDR — 快速重启绑定端口 ★★★★

> `SO_REUSEADDR` 允许在 TIME_WAIT 状态下重新绑定端口，是实现**平滑重启/热升级**的关键。

```
场景：Go 服务重启时，旧进程进入 TIME_WAIT（60s），新进程无法绑定同一端口
  → SO_REUSEADDR 让新进程立即绑定，跳过 TIME_WAIT 等待
  
与 SO_REUSEPORT 的区别：
  SO_REUSEADDR：允许绑定 TIME_WAIT 中的端口（重启用）
  SO_REUSEPORT：允许多个进程同时绑定同一端口并监听（负载均衡用，如 Nginx 多 worker）
```

```go
// Go 中设置 SO_REUSEADDR（配合 ListenerConfig）
import (
    "context"
    "net"
    "syscall"
    
    "golang.org/x/sys/unix"
)

func listenWithReuseAddr() (net.Listener, error) {
    lc := net.ListenConfig{
        Control: func(network, address string, c syscall.RawConn) error {
            return c.Control(func(fd uintptr) {
                unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEADDR, 1)
            })
        },
    }
    return lc.Listen(context.Background(), "tcp", ":8080")
}
```

> **Go 注意事项**：`net.Listen` 和 `http.ListenAndServe` 内部**已经默认启用 SO_REUSEADDR**（Go 1.11+）。直接用标准库监听不需要手动设置。只有使用自定义 `ListenConfig` 覆盖了 Control 时才需要显式添加。

## 11.6 零拷贝 ★★★★★

### 传统文件发送（4 次拷贝，4 次上下文切换）

```
磁盘 →(DMA)→ 内核缓冲区 →(CPU)→ 用户缓冲区 →(CPU)→ Socket 缓冲区 →(DMA)→ 网卡
      ↑拷贝1              ↑拷贝2            ↑拷贝3              ↑拷贝4
```

### sendfile 零拷贝（2 次拷贝，2 次上下文切换）

```
磁盘 →(DMA)→ 内核缓冲区 →(DMA gather)→ 网卡
      ↑拷贝1              ↑拷贝2（gather DMA，不经过 CPU）
```

### mmap + write（减少一次拷贝）

```
用户缓冲区映射到内核缓冲区 → 不拷贝数据到用户空间
```

### 零拷贝在 Go 中的应用

```go
// io.Copy 在 Linux 上使用 sendfile 系统调用
// 当 src 是 *os.File，dst 是 *net.TCPConn 时自动使用 sendfile
http.HandleFunc("/download", func(w http.ResponseWriter, r *http.Request) {
    file, _ := os.Open("large_file.zip")
    defer file.Close()
    io.Copy(w, file)  // 底层使用 sendfile，零拷贝！
})
```

### 面试题

**Q：什么是零拷贝？Kafka 为什么快？**

> 零拷贝指数据在传输过程中避免 CPU 将数据从内核空间拷贝到用户空间再拷贝回去。传统发送文件需要 4 次拷贝和 4 次上下文切换；使用 `sendfile` 零拷贝后只需要 2 次拷贝和 2 次上下文切换，DMA 引擎直接将数据从内核缓冲区搬移到网卡。
>
> **Kafka** 使用零拷贝技术将磁盘上的日志文件直接发送给消费者：生产者写入 → 磁盘（顺序写） → 消费者读取时用 `sendfile` 直接将数据从 Page Cache 发送到网卡，不经过用户空间，极大减少 CPU 开销。这也是 Kafka 能支撑百万级吞吐的核心原因。

## 11.7 C10K 问题

> C10K（Client 10000）即在单台服务器上同时处理 10000+ 个客户端连接的问题。解决方案：
> - **I/O 模型**：必须用 I/O 多路复用（epoll），不能用阻塞 I/O
> - **线程模型**：不能每个连接一个线程 → 使用协程/事件循环
> - **Go 的天然优势**：goroutine 轻量（2KB 初始栈）+ netpoller 底层使用 epoll = 天然支持高并发

## 11.8 Nginx 模型

```
Nginx 进程架构：
  Master 进程（1 个）：管理 Worker，不处理请求
  Worker 进程（CPU 核数个）：每个 Worker 独立运行一个事件循环（epoll）

  ┌──────────────┐
  │    Master    │  root 运行，绑定 80/443 端口
  └──┬───┬───┬───┘
     │   │   │
     v   v   v
  ┌───┐┌───┐┌───┐
  │ W1││ W2││ W3│  nginx 用户运行，每个独立 epoll 循环
  └───┘└───┘└───┘
    共享监听 socket（SO_REUSEPORT）

  特点：
  - 多进程，非多线程（稳定性高，一个 Worker 挂了不影响其他）
  - 每个 Worker 独立 epoll 事件循环（主从 Reactor）
  - 无锁设计（Worker 之间不共享数据）
```

## 11.9 容器底层技术 ★★★★★

> 容器本质上是 Linux Namespace + Cgroups 的组合。现在几乎所有 Go 服务都跑在容器里，校招面试中容器底层原理是高频考点。

### Namespace — 资源隔离

Linux Namespace 让进程只能看到"自己世界"里的资源，实现进程间隔离。

```
共 7 种 Namespace（面试重点记前 6 种）：

1. PID Namespace
   隔离：进程 ID 编号
   容器内 PID=1 是容器的 init 进程，在宿主机上有不同的真实 PID
   效果：docker exec 进去只能看到容器内的进程树，看不到宿主机的

2. Network Namespace
   隔离：网络设备、IP 地址、端口、路由表、iptables 规则
   每个容器有独立的网络栈
   veth pair：连接容器 namespace 与宿主机 bridge（docker0）

3. Mount Namespace
   隔离：文件系统挂载点
   容器有独立的根文件系统视图
   结合 chroot/pivot_root 实现容器镜像的文件系统

4. UTS Namespace
   隔离：主机名（hostname）和 NIS 域名
   每个容器可以有独立的主机名

5. IPC Namespace
   隔离：System V IPC（消息队列、信号量、共享内存）
   容器间 IPC 互不影响

6. User Namespace
   隔离：UID/GID 映射
   容器内的 root(UID=0) 可以映射为宿主机的普通用户
   提供安全隔离（rootless container）

7. Cgroup Namespace（Linux 4.6+ 加入）
   隔离：cgroup 层级视图
   容器内看到的 /sys/fs/cgroup 不会包含宿主机的完整信息
```

**验证 Namespace 的命令**：

```bash
# 查看容器在宿主机上的真实 PID
docker inspect <容器ID> | jq '.[0].State.Pid'

# 查看该进程所属的 namespace
ls -la /proc/<PID>/ns/
# 输出：
# lrwxrwxrwx net -> net:[4026532824]
# lrwxrwxrwx pid -> pid:[4026532825]
# lrwxrwxrwx mnt -> mnt:[4026532822]
# ...
# 方括号中的数字是 namespace ID，同一容器内进程共享相同 ID

# 进入容器的 network namespace（宿主机操作）
nsenter -t <PID> -n ip addr    # 等价于 docker exec 看到的网络
nsenter -t <PID> -n ss -lntp   # 看容器内的端口监听
```

### Cgroups — 资源限制

```bash
# Cgroups v1：每个 subsystem 独立
ls /sys/fs/cgroup/
# cpu/  cpuacct/  memory/  blkio/  devices/  ...
# 每个都是独立的文件系统

# Cgroups v2：统一层级（K8s 1.25+ 默认）
# 所有控制器在 /sys/fs/cgroup/ 下
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2

# 查看容器的 cgroup 限制
docker inspect <容器ID> | jq '.[0].HostConfig.Memory'
# 输出：1073741824 (= 1GB)

# 从宿主机查看容器的内存使用
cat /sys/fs/cgroup/memory/docker/<容器ID>/memory.usage_in_bytes
cat /sys/fs/cgroup/memory/docker/<容器ID>/memory.limit_in_bytes
```

**Cgroups 四大子系统**：

```
1. cpu   子系统：限制 CPU 使用量
   cpu.cfs_quota_us / cpu.cfs_period_us = CPU 核心数
   例如：quota=150000, period=100000 → 1.5 核 CPU

2. memory 子系统：限制内存使用
   memory.limit_in_bytes：硬限制（超过会 OOM kill）
   容器中常见：超过限制后容器直接被 kill

3. blkio 子系统：限制磁盘 I/O
   限制设备的读写 IOPS 和吞吐量

4. devices 子系统：控制设备访问权限
   容器中通常只允许访问有限设备（/dev/null, /dev/random 等）
```

### cgroups v1 vs v2

| 对比项 | cgroups v1 | cgroups v2 |
|--------|-----------|-----------|
| 层级结构 | 每个子系统独立树 | 统一层级 |
| 路径 | `/sys/fs/cgroup/cpu/`, `/sys/fs/cgroup/memory/`... | `/sys/fs/cgroup/` |
| 灵活性 | 可独立挂载不同子系统 | 必须统一挂载 |
| 内核支持 | Linux 2.6.24+ | Linux 4.5+（K8s 1.25+ 默认） |
| 特点 | 成熟稳定 | 支持 PSI（资源压力监控）、配置更一致 |

```bash
# 查看 cgroup 版本
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2 → v2
# 或
stat -fc %T /sys/fs/cgroup
```

### eBPF 简介 ★★★

eBPF（extended Berkeley Packet Filter）允许在**不修改内核源码或加载内核模块**的情况下，在内核中安全运行沙箱程序。

```
典型应用：
  - Cilium：基于 eBPF 的 K8s CNI 网络插件（替代 kube-proxy）
  - Falco：容器运行时安全监控和入侵检测
  - bpftrace / BCC：性能分析和追踪工具
  - Go 中通过 cilium/ebpf 库操作 eBPF 程序
```

> 校招了解概念即可，极少深入提问。

### Go 程序创建 Namespace 示例

```go
// Go 中创建新的 Namespace（底层调用 clone 系统调用）
// 这是容器运行时的核心操作
package main

import (
    "os"
    "os/exec"
    "syscall"
)

func main() {
    cmd := exec.Command("/bin/sh")
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    // 设置 Namespace 隔离
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUTS |   // UTS: 独立主机名
                    syscall.CLONE_NEWPID |   // PID: 独立进程树
                    syscall.CLONE_NEWNS |    // Mount: 独立文件系统
                    syscall.CLONE_NEWNET |   // Network: 独立网络栈
                    syscall.CLONE_NEWIPC,    // IPC: 隔离 IPC 资源
    }

    cmd.Run()
    // 运行后你会看到隔离的进程空间，ps 只显示容器内的进程
}
```

> 这就是 `docker run` 底层做的事情——通过 clone 系统调用创建进程并加入各个 Namespace，再用 cgroups 限制资源。

### Go 与容器：常见问题与解决

```go
// 问题1：容器内获取的 CPU 核数是宿主机的
runtime.NumCPU()  // 返回 64（宿主机），但容器限制 2 核
// 解决：Go 1.9+ 传入 GOMAXPROCS=2 或使用 uber-go/automaxprocs
import _ "go.uber.org/automaxprocs"  // 自动读取 cgroup 限制

// 问题2：容器 OOMKilled 无日志
// 原因：超过 memory.limit 后内核直接 SIGKILL（杀 PID 1）
// 排查：dmesg -T | grep -i oom（宿主机上查看）
// 解决：设置 GOMEMLIMIT（Go 1.19+）主动限制 GC 触发

// ====== GOMEMLIMIT 详解 ======
// Go 1.19+ 引入的软内存限制，是 Go 程序在容器中运行的"必配项"
import "runtime/debug"

// 方式1：代码中设置
debug.SetMemoryLimit(2.5 * 1024 * 1024 * 1024)  // 2.5GB 硬限制

// 方式2：环境变量（推荐，无需改代码）
// GOMEMLIMIT=2500MiB ./app
// 或 K8s Deployment 中：
// env:
//   - name: GOMEMLIMIT
//     valueFrom:
//       resourceFieldRef:
//         resource: limits.memory   # 自动读取 K8s 容器内存限制

// GOMEMLIMIT 工作原理：
//   Go GC 会尽量将堆内存控制在 GOMEMLIMIT 以下
//   如果堆内存接近限制 → GC 频率主动增加，释放内存
//   如果堆内存持续超过限制 → GC 不会阻止（这是"软限制"）
//   只有 cgroup memory.limit 才是硬限制（超过 → OOM Kill）

// 关键：GOMEMLIMIT 不替代 cgroup 限制，而是补充
// 正确设置 = 容器内存限制 * 80%（预留 20% 给 GC 自身、stack、off-heap）
// 例如：K8s limits.memory = 2Gi → GOMEMLIMIT = 1600MiB

// 常见错误：
// ❌ GOMEMLIMIT = 容器限制 → GC 来不及释放就 OOM
// ❌ 不设 GOMEMLIMIT → Go 默认 GC target = 100% 容器限制 → 危险
// ✅ GOMEMLIMIT = 容器限制 * 0.8 → 留 20% 缓冲

// 问题3：优雅关闭时间不够
// K8s terminationGracePeriodSeconds 默认 30s
// 必须：① server.Shutdown 超时 < terminationGracePeriodSeconds
// ② preStop hook 优先处理流量摘除
// ③ 确保 server.ListenAndServe 和 server.Shutdown 协调
```

> **面试答法**：容器由 Linux Namespace（隔离）+ Cgroups（限制）实现。Namespace 让容器看到独立的进程树、网络栈、文件系统；Cgroups 限制 CPU/内存/IO 用量。Go 服务跑容器需要注意：① 用 automaxprocs 正确识别 CPU 限制；② 设置 GOMEMLIMIT 防止 OOM；③ 优雅关闭时间要小于 K8s terminationGracePeriodSeconds。

## 11.10 高并发内核参数调优 ★★★★★

> 每个高并发 Go 服务的生产环境都需要调整这些内核参数。

### /etc/sysctl.conf 完整配置及说明

```bash
# ====== TCP 三次握手相关 ======
# 半连接队列（SYN queue）：未完成三次握手的连接
# sysctl net.ipv4.tcp_max_syn_backlog=8192
# → 短时大量 SYN 请求时来不及处理的缓冲池大小

# 全连接队列（Accept queue）：已完成三次握手，等待 accept() 的连接
# sysctl net.core.somaxconn=65535
# → Go 1.11+ 中 net.Listen 会自动读取 /proc/sys/net/core/somaxconn 的值作为 backlog
# → 所以只需要设置 sysctl somaxconn，Go 程序会自动使用该值
# → 早期版本或手动指定 backlog 时：
#   listener, _ := net.ListenConfig{Backlog: 65535}.Listen(ctx, "tcp", ":8080")
# → 注意：somaxconn 只是上限，实际 backlog = min(应用指定值, somaxconn)

# ====== TCP 连接复用与 TIME_WAIT ======
net.ipv4.tcp_tw_reuse = 1
# 客户端允许复用 TIME_WAIT 连接（关键！）
# 高并发短连接场景（如 HTTP 调用下游服务）必须开启

net.ipv4.tcp_tw_recycle = 0
# 严禁开启！在 NAT 环境下会导致连接失败
# Linux 4.12+ 已移除此选项

net.ipv4.tcp_fin_timeout = 30
# 默认 60s，调小可加速 FIN_WAIT2 → TIME_WAIT 转换
# TIME_WAIT 持续时间是 2*MSL（~60s），不能通过 tcp_fin_timeout 直接改

net.ipv4.tcp_max_tw_buckets = 10000
# 系统中同时存在的 TIME_WAIT 最大数量
# 超过后新产生的 TIME_WAIT 直接被关闭
# 防御性参数，防止 TIME_WAIT 耗尽内存

# ====== 端口范围 ======
net.ipv4.ip_local_port_range = 1024 65535
# 客户端连接时可用的本地端口范围
# 默认 32768-60999（约 28000 个），调大到 1024-65535
# 避免 "Cannot assign requested address" 错误

# ====== TCP 缓冲区调优 ======
# 三个值：min / default / max（单位：字节）
net.ipv4.tcp_rmem = 4096 131072 6291456
# TCP 读缓冲区：最小 4KB / 默认 128KB / 最大 6MB
# → 大吞吐下载场景（文件服务/CDN）可调大 max

net.ipv4.tcp_wmem = 4096 65536 4194304
# TCP 写缓冲区：最小 4KB / 默认 64KB / 最大 4MB
# → 大文件上传场景可调大 max

net.ipv4.tcp_mem = 786432 1048576 1572864
# 系统级 TCP 内存总量（单位：page，通常 4KB/page）
# → 值 = 页面数，分别对应低/压力/上限三个水位线
# → 高并发场景需要调大，否则内核会限制新连接

# 场景：10 万长连接（如 WebSocket/消息推送）
# 每个连接 16KB 接收缓冲区 = 1.6GB
# tcp_mem max 需 > 1.6GB / 4KB = 409600 pages

# ====== Go 代码中的 Socket 缓冲区设置 ======
// 对大文件传输优化 socket 缓冲区
func setLargeBuffer(network, address string, c syscall.RawConn) error {
    return c.Control(func(fd uintptr) {
        // 设置为 256KB
        unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_RCVBUF, 256*1024)
        unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_SNDBUF, 256*1024)
    })
}
# 注意：内核会 double 设置的值（为了开销），setsockopt 设 256KB 实际得到 512KB

# ====== TCP KeepAlive ======
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3
# 600s 无数据后开始探测，每 30s 一次，3 次失败则关闭
# 用于清理死连接（对端崩溃但未正常关闭）

# ====== 文件描述符 ======
# /etc/security/limits.conf
*    soft    nofile    65535
*    hard    nofile    65535
# 每个进程最大打开文件数
# 高并发服务必须调，默认 1024 远不够

# ====== 内存 ======
vm.swappiness = 10
# 取值范围 0-100，默认 60
# 值越大越倾向使用 swap；越小越倾向回收 Page Cache
# 服务器设为 10，尽量少用 swap
# 数据库服务器设为 1 或 0

# ====== 参数查看方法 ======
# sysctl -a | grep tcp_tw_reuse
# 使配置生效：sysctl -p
```

### Go 代码中对应的设置

```go
import (
    "net"
    "syscall"
    "golang.org/x/sys/unix"
)

// 设置 socket 选项
func setSocketOptions(network, address string, c syscall.RawConn) error {
    return c.Control(func(fd uintptr) {
        // 设置 SO_REUSEPORT（多进程共享监听）
        unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)

        // 设置 TCP keepalive
        unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_KEEPALIVE, 1)

        // 设置 TCP Fast Open（TFO），减少一次 RTT
        // unix.SetsockoptInt(int(fd), unix.IPPROTO_TCP, unix.TCP_FASTOPEN, 1)
    })
}

// 使用自定义 Listener
lc := net.ListenConfig{
    Control: setSocketOptions,
}
ln, _ := lc.Listen(context.Background(), "tcp", ":8080")
```

### 面试高频：全连接队列溢出怎么发现？

```bash
# 看是否有溢出（Overflow）
netstat -s | grep -i "listen"
# 关键字：times the listen queue of a socket overflowed

# 实时监控（出现 overflow 说明 accept 不够快）
watch -n 1 'netstat -s | grep overflowed'

# Go 中优化：
# 1. 提高 somaxconn → sysctl net.core.somaxconn=65535
# 2. 多 goroutine 并行 accept（但 Go 标准库是单 goroutine accept）
# 3. 用 fasthttp 替代 net/http（更高的 accept 吞吐）
```

---

<a id="第十二章-linux-故障排查"></a>
# 第十二章 Linux 故障排查 ★★★★★

## 场景1：CPU 100% 如何排查？

```bash
# 步骤1：找出高 CPU 进程
top
# 按 P 排序，记录高 CPU 进程的 PID

# 步骤2：找出该进程内的高 CPU 线程
top -H -p <PID>
# 记录高 CPU 线程的 TID

# 步骤3：Go 程序 → 使用 pprof
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof
go tool pprof cpu.prof
# pprof> top10          # 查看 CPU 热点函数
# pprof> list FuncName  # 查看具体代码行
# pprof> web            # 生成火焰图

# 步骤4：非 Go 程序 → 使用 perf
perf top -p <PID>
perf record -p <PID> -g --sleep 30
perf report

# 步骤5：看系统调用
strace -c -p <PID>      # 统计系统调用次数和时间
```

## 场景2：内存持续上涨如何排查？

```bash
# 步骤1：确认内存增长
top -p <PID>                # 观察 RES 列
ps aux | grep app           # 观察 %MEM 和 RSS

# 步骤2：查看系统内存概况
free -h
vmstat 1                    # 看 si/so（swap in/out）

# 步骤3：Go 程序内存分析
curl http://localhost:6060/debug/pprof/heap > heap.prof
go tool pprof heap.prof
# pprof> top10              # 哪个函数分配内存最多
# pprof> list FuncName      # 具体代码行的内存分配

# 步骤4：对比两次 heap（找到增长点）
curl http://localhost:6060/debug/pprof/heap > heap1.prof
sleep 60
curl http://localhost:6060/debug/pprof/heap > heap2.prof
go tool pprof -base heap1.prof heap2.prof
# pprof> top10              # 这段时间内增长最多的内存分配

# 步骤5：检查 goroutine 泄漏
curl http://localhost:6060/debug/pprof/goroutine?debug=1 | head
# 或
curl http://localhost:6060/debug/pprof/goroutine > goroutine.prof
go tool pprof goroutine.prof
# pprof> top10              # 哪个函数创建的 goroutine 最多

# 步骤6：查看进程内存详情
pmap -x <PID> | sort -rn -k3  # 按 RSS 排序
cat /proc/<PID>/status | grep Vm  # 各种内存指标
```

## 场景3：磁盘满了如何排查？

```bash
# 步骤1：确认哪个分区满了
df -h

# 步骤2：找到大目录
cd /data            # 进入满的分区
du -sh * | sort -rh | head -5

# 步骤3：逐层深入
cd 最大目录
du -sh * | sort -rh | head -5
# 重复此步骤直到找到具体的大文件

# 步骤4：检查是否有大日志文件
du -sh /var/log/* | sort -rh | head -10

# 步骤5：检查是否有进程占用已删除的文件
lsof | grep deleted | sort -rn -k 7 | head
# SIZE 大的进程 → 重启该进程释放空间
# 原理：文件虽已从目录删除，但进程仍持有 fd，空间未释放

# 步骤6：查找并清理
find /data -name "*.log" -size +100M -mtime +30    # 30天前的大日志文件
find /data -name "*.tmp" -delete                    # 删除临时文件
find /data -name "*.log" -mtime +30 -exec gzip {} \;  # 压缩旧日志
```

## 场景4：服务器变慢如何排查？

```bash
# 整体诊断（5 分钟内定位问题）

# 1. 先看负载
uptime
# load average 高 → 继续排查

# 2. 看 CPU
top
# us 高 → CPU 密集型任务（找高 CPU 进程）
# sy 高 → 系统调用过多（strace 排查）
# wa 高 → I/O 等待（排查磁盘）

# 3. 看内存
free -h
# available 低 → 内存不足
# Swap used > 0 → 内存紧张

# 4. 看磁盘 I/O
iostat -x 1
# %util 高 → 磁盘瓶颈

# 5. 看网络
ss -s                       # 连接数统计
# TIME_WAIT 过多 → 可能存在短连接问题
```

## 场景5：端口无法访问如何排查？

```bash
# 步骤1：确认服务是否在监听
ss -lntp | grep <端口>
# 如果没有输出 → 服务没启动或没监听该端口
# 检查服务是否只监听 127.0.0.1（本地）而非 0.0.0.0（所有网卡）

# 步骤2：确认进程是否在运行
ps aux | grep <进程名>

# 步骤3：本地测试
curl -v http://localhost:<端口>/health
curl -v http://127.0.0.1:<端口>/health

# 步骤4：检查防火墙
iptables -L -n          # iptables 规则
firewall-cmd --list-all # firewalld 规则

# 步骤5：远程测试
telnet <远程IP> <端口>
nc -zv <远程IP> <端口>

# 步骤6：查看进程是否被占用
lsof -i :<端口>
```

## 场景6：线上接口超时如何排查？

```bash
# 步骤1：确认问题范围
# 是单个接口慢还是所有接口都慢？
# 是偶发还是一直慢？

# 步骤2：看服务状态
top -p <PID>                    # CPU 使用
ss -antp | grep :8080 | wc -l  # 当前连接数

# 步骤3：看慢在哪一层
# 可以用 curl 测量各阶段时间
curl -o /dev/null -s -w "\
DNS解析: %{time_namelookup}s\n\
TCP连接: %{time_connect}s\n\
SSL握手: %{time_appconnect}s\n\
首字节: %{time_starttransfer}s\n\
总时间:  %{time_total}s\n" http://localhost:8080/api/test

# 步骤4：Go 程序用 pprof trace 看阻塞点
curl http://localhost:6060/debug/pprof/trace?seconds=5 > trace.out
go tool trace trace.out

# 步骤5：看数据库/缓存
strace -p <PID> -e trace=read,write,connect,sendto,recvfrom
# 或
tcpdump -i eth0 port 3306 -w mysql.pcap  # 抓 MySQL 流量
```

## 场景7：连接数暴涨如何排查？

```bash
# 步骤1：确认连接数
ss -s                                       # 连接数统计
ss -antp | wc -l                            # 总连接数
ss -antp | grep ESTABLISHED | wc -l         # 活跃连接数
ss -antp | grep TIME_WAIT | wc -l           # TIME_WAIT 数量

# 步骤2：按 IP 统计连接数（找刷流量的 IP）
ss -antp | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -20

# 步骤3：按进程统计连接数
ss -antp | awk '{print $NF}' | sort | uniq -c | sort -rn | head

# 步骤4：排查 TIME_WAIT 过多
# 检查是否是短连接设计问题
# 内核参数优化（谨慎）：
# net.ipv4.tcp_tw_reuse = 1
# net.ipv4.tcp_fin_timeout = 30
# net.ipv4.tcp_max_tw_buckets = 5000

# 步骤5：Go 程序检查连接池配置
# 检查 http.Client 是否共享 Transport
# 检查数据库连接池 MaxOpenConns / MaxIdleConns 设置
```

## 场景8：DNS 解析失败如何排查？

```bash
# 步骤1：查看 DNS 配置
cat /etc/resolv.conf

# 步骤2：测试 DNS 服务器连通性
ping -c 2 8.8.8.8

# 步骤3：手动 DNS 查询
nslookup example.com
dig example.com
dig @8.8.8.8 example.com     # 指定 DNS 服务器

# 步骤4：查看本地 hosts
cat /etc/hosts | grep example.com

# 步骤5：Go 程序 DNS 解析问题
# Go 默认使用系统 DNS 解析器，但可以通过 GODEBUG 切换
# GODEBUG=netdns=go     强制使用 Go 内置解析器
# GODEBUG=netdns=cgo    强制使用系统解析器（cgo）
# 排查 DNS 缓存问题：检查 /etc/nsswitch.conf 的 hosts 顺序
```

---

<a id="第十三章-linux-面试高频-top50"></a>
# 第十三章 Linux 面试高频 Top50 ★★★★★

## 基础命令

**1. ls 和 ll 区别？**

> `ll` 是 `ls -l` 的别名（大部分发行版默认设置）。`ls` 只列文件名，`ll` 显示详细信息（权限、大小、时间等）。

**2. grep 常用参数？**

```bash
grep -i "error"         # 忽略大小写
grep -v "debug"         # 排除匹配行
grep -n "panic"         # 显示行号
grep -r "TODO" .        # 递归搜索
grep -C 3 "error"       # 显示上下文
grep -E "err|warn"      # 正则匹配
grep -l "pattern" *.go  # 只显示文件名
grep -c "pattern" file  # 统计匹配行数
```

**3. awk 常用用法？**

```bash
awk '{print $1, $NF}'               # 打印第1列和最后1列
awk -F: '{print $1}' /etc/passwd    # 指定分隔符
awk '$3 > 100 {print}' file         # 条件过滤
awk '{sum+=$1} END {print sum}'     # 求和
awk '{arr[$1]++} END {for(k in arr) print arr[k], k}'  # 统计频次
```

**4. sed 常用用法？**

```bash
sed 's/old/new/g' file              # 替换（输出到终端）
sed -i 's/old/new/g' file           # 替换（直接修改文件）
sed '/pattern/d' file               # 删除匹配行
sed -n '10,20p' file                # 打印指定行范围
sed '/^$/d; /^#/d' nginx.conf       # 删除空行和注释行
```

**5. find 常用参数？**

```bash
find . -name "*.go"                 # 按名称
find . -type f -size +100M          # 按大小
find . -mtime -7                    # 按修改时间
find . -name "*.log" -exec rm {} \; # 执行命令
find . -type d -name "vendor" -prune -o -print  # 排除目录
```

---

## 权限

**6. chmod 755 含义？**

> 所有者 rwx（7=读+写+执行），同组 r-x（5=读+执行），其他 r-x（5=读+执行）。常用于可执行文件和目录。

**7. chmod 777 风险？**

> 任何用户可读写执行该文件，攻击者可篡改。生产环境禁止使用 777，遵循最小权限原则。

**8. sudo 和 su 区别？**

> `su` 完全切换到目标用户（需要目标用户密码），`sudo` 以其他用户身份执行单条命令（需要当前用户密码），可精细控制权限。

---

## 进程

**9. ps 和 top 区别？**

> `ps` 是快照（一瞬间的进程状态），`top` 是实时动态监控。排查持续性问题用 `top`，查看进程树/特定条件过滤用 `ps`。

**10. kill -9 和 kill -15 区别？**

> `-15`（SIGTERM）是优雅终止，进程可以捕获并做清理（Go `signal.Notify` 可捕获）；`-9`（SIGKILL）强制终止，进程无法处理，直接退出。优先用 `-15`。

**11. nohup 原理？**

> 修改进程对 SIGHUP 信号的响应为忽略（SIG_IGN）。终端关闭时内核发 SIGHUP 给进程，nohup 启动的进程忽略该信号，继续运行。

---

## 内存

**12. free 和 available 区别？**

> `free` = 完全未使用的内存。`available` = 可分配给新进程的内存 = free + buff/cache 中可回收部分。判断内存是否够用看 `available`。

**13. Linux 为什么 OOM？**

> 物理内存 + Swap 全部耗尽时，OOM Killer 根据 OOM score 选择进程杀死释放内存。常见原因：内存泄漏、大内存申请、高并发。

**14. Page Cache 是什么？**

> 内核用于缓存文件数据的物理内存。读写文件时数据先经过 Page Cache，后续访问命中缓存则无需磁盘 I/O。这就是 Linux 内存管理中"free 很低但系统不慢"的原因——内存被用来做缓存了。

---

## 磁盘

**15. df 和 du 区别？**

> `df` 查看文件系统级磁盘使用（读超级块，快）。`du` 遍历目录树累加文件大小（遍历，慢）。差异原因：文件已删除但进程占用时，df 会计入，du 不计入。

**16. iostat 怎么看？**

```bash
iostat -x 1
# await = 平均 I/O 响应时间（>50ms 性能较差，>200ms 瓶颈）
# %util = 磁盘使用率（>80% 接近饱和）
# r/s, w/s = 每秒读写次数（IOPS）
```

---

## 网络

**17. ping 原理？**

> 发送 ICMP Echo Request，目标回复 ICMP Echo Reply。通过 RTT 判断延迟，丢包率判断网络质量。防火墙可能禁止 ICMP 导致 ping 不通。

**18. traceroute 原理？**

> 利用 IP 报文的 TTL 字段。发送 TTL=1 的包，第一个路由器收到后 TTL 变为 0，返回 ICMP Time Exceeded；再发 TTL=2 的包……以此类推，追踪到目标经过的每一跳路由。

**19. netstat 和 ss 区别？**

> `ss` 是 `netstat` 的现代替代品，直接从内核获取 socket 信息（netlink），更快更详细。大量连接时 ss 明显更快，推荐使用 ss。

**20. lsof 如何查端口？**

```bash
lsof -i :8080           # 查看占用 8080 端口的进程
lsof -i tcp:8080        # 只看 TCP
lsof -p <PID> | grep TCP  # 查看进程的所有网络连接
```

---

## 日志

**21. 如何实时查看日志？**

```bash
tail -f app.log                     # 实时追踪
tail -f app.log | grep ERROR        # 实时过滤错误
tail -n 100 -f app.log              # 先显示最近 100 行再追踪
```

**22. 如何统计 Top10 访问 IP？**

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
```

**23. 如何统计错误日志？**

```bash
grep -c "ERROR" app.log             # 错误总数
grep "ERROR" app.log | awk '{print $5}' | sort | uniq -c | sort -rn  # 按错误类型统计
```

---

## 性能分析

**24. Load Average 是什么？**

> 1/5/15 分钟的平均活跃进程数（R 状态 + D 状态）。超过 CPU 核数说明有进程在排队。与 CPU 使用率不同：Load 高但 CPU 低 = I/O 瓶颈；两者都高 = CPU 瓶颈。

**25. CPU 100% 如何排查？**

```bash
top → top -H -p <PID> → Go: pprof / 其他: perf → strace
```

**26. 内存泄漏如何排查？**

```bash
top → ps aux --sort=-%mem → Go: pprof heap → 对比两次 heap 找增长点
```

**27. 磁盘 I/O 高如何排查？**

```bash
iostat -x 1 → iotop → lsof -p <PID>
```

---

## 网络编程

**28. 文件描述符（FD）是什么？**

> 进程打开文件的整数句柄，内核通过它定位文件的具体信息。0=stdin, 1=stdout, 2=stderr。高并发服务器需调大 `ulimit -n`，否则会报 `too many open files`。

**29. select 原理？**

> 传入 fd_set 位图，内核遍历全部 fd 检查就绪状态。限制 1024 个 fd，O(n) 遍历，每次调用需拷贝整个 fd_set。适用于连接少的场景。

**30. poll 原理？**

> 用 pollfd 链表替代位图，无 1024 限制。但仍然是 O(n) 遍历，连接多时性能差。

**31. epoll 原理？**

> 内核中维护红黑树（存所有 fd）+ 就绪链表（存就绪 fd）。fd 就绪时通过回调加入就绪链表，epoll_wait 直接从链表取结果，O(1)。fd 只在注册时拷贝一次，不随连接数增长性能下降。

**32. LT 和 ET 区别？**

> LT（水平触发）：fd 有数据就通知，数据未读完下次继续通知。ET（边缘触发）：fd 状态变化时只通知一次，必须一次读完所有数据（循环 read 直到 EAGAIN），否则数据丢失。LT 简单，ET 高性能。

---

## 高并发

**33. Reactor 模型？**

> 事件驱动的 I/O 处理模型。单 Reactor 单线程（Redis 6.0 前），单 Reactor 多线程，主从 Reactor（Nginx、Netty、Go netpoller）。Go 中每个 goroutine 处理一个连接，底层 netpoller 用 epoll 实现，开发者写同步代码，运行时异步执行。

**34. C10K 问题？**

> 单机处理 10000+ 并发连接的问题。解决方案：I/O 多路复用（epoll）+ 事件驱动 + 轻量级协程。Go 的 goroutine + netpoller 天然解决此问题。

**35. 零拷贝原理？**

> 避免 CPU 在内核空间和用户空间之间拷贝数据。`sendfile` 系统调用将数据直接从内核缓冲区通过 DMA 发送到网卡。传统方式 4 次拷贝 → 零拷贝 2 次拷贝。

**36. sendfile 原理？**

> `sendfile(out_fd, in_fd, offset, count)` 将数据从 in_fd 对应的 Page Cache 直接传输到 out_fd 对应的 Socket Buffer，跳过用户空间的拷贝。Linux 2.4+ 支持 gather DMA，数据直接从 Page Cache 发送到网卡。Go 中 `io.Copy` 在 `*os.File` → `*net.TCPConn` 时自动使用 sendfile。

---

## 综合场景

**37. 服务无法启动怎么办？**

> ① 看启动日志；② `ss -lntp | grep <端口>` 确认端口是否被占用；③ `lsof -i :<端口>` 找到占用端口的进程；④ 看磁盘空间 `df -h`（空间满无法写日志）；⑤ 看权限（非 root 无法绑定 1024 以下端口）；⑥ `strace -f ./app` 看启动时卡在哪个系统调用。

**38. 服务器突然变慢怎么办？**

> `top` 看 CPU/内存 → `iostat -x 1` 看 I/O → `ss -s` 看连接数 → `vmstat 1` 看上下文切换和 swap。快速定位是 CPU/内存/磁盘/网络哪个维度的瓶颈。

**39. 线上接口 RT 升高怎么办？**

> ① 看 QPS 是否增加（`ss -antp | wc -l`）；② 看 CPU/内存是否打满；③ 看数据库慢查询；④ 看下游依赖是否超时；⑤ Go 程序用 pprof trace 看具体阻塞点；⑥ 用 `tcpdump` 抓包分析网络耗时。

**40. 数据库连接打满怎么办？**

> ① `ss -antp | grep 3306 | wc -l` 看实际连接数；② 检查应用连接池配置（Go 中 `SetMaxOpenConns`、`SetMaxIdleConns`）；③ 检查是否有连接泄漏（`lsof -p <PID> | grep TCP | wc -l`）；④ 看数据库端是否有慢查询导致连接积压。

**41. 连接数暴涨怎么办？**

> `ss -antp | awk '{print $5}' | sort | uniq -c | sort -rn | head` 按来源 IP 统计，找异常流量源。检查是否有 TIME_WAIT 堆积（短连接问题）。

**42. CPU 持续 100% 怎么办？**

> `top` → `top -H -p <PID>` → Go: pprof CPU profile → 找热点函数 → `list FuncName` 看具体代码行 → 优化或加缓存。

**43. 内存持续上涨怎么办？**

> Go: `pprof heap` → 对比两次 heap 找增长点 → 检查 map 是否无限增长 / goroutine 是否泄漏 / channel 是否阻塞。

**44. 磁盘空间不足怎么办？**

> `df -h` → `du -sh *` 逐层深入 → `lsof | grep deleted` 找已删除但占用空间的文件 → 清理旧日志 `find /var/log -name "*.log" -mtime +30 -delete`。

**45. 端口被占用怎么办？**

> `lsof -i :<端口>` 或 `ss -lntp | grep :<端口>` 找到占用进程 → 如果是需要的进程则换端口，如果是不需要的则 `kill <PID>`。

**46. DNS 解析失败怎么办？**

> ① `cat /etc/resolv.conf` 查 DNS 配置；② `dig @8.8.8.8 example.com` 手动测试；③ `cat /etc/hosts` 查本地 hosts；④ `ping 8.8.8.8` 测试网络连通性。

**47. 服务频繁 OOM 怎么办？**

> ① `dmesg -T | grep -i oom` 看 OOM 日志确认是哪个进程被杀；② `free -h` 看系统内存；③ Go 程序 `pprof heap` 分析内存分配热点；④ 考虑调大 Swap 或加内存；⑤ 调整 `oom_score_adj` 保护关键进程。

**48. 如何抓包分析问题？**

> `tcpdump -i eth0 port 8080 -w capture.pcap` 抓包 → Wireshark 分析。命令行快速查看：`tcpdump -i eth0 -A port 8080`。

**49. 如何分析慢请求？**

> ① Nginx access log 分析 `$request_time` 最大的请求；② curl 测量各阶段时间；③ Go pprof trace 看阻塞和调度延迟；④ 检查慢查询和慢调用链。

**50. 如何定位线上故障？**

> **黄金 5 分钟法则**：① `uptime` 看负载 → ② `top` 看 CPU/内存 → ③ `free -h` 看内存 → ④ `iostat -x 1` 看 I/O → ⑤ `ss -s` 看网络连接 → ⑥ 看业务日志 → ⑦ Go pprof 深入分析。先整体后局部，先粗后细。

---

<a id="附录go-开发必知必会-linux-命令速查表"></a>
# 附录：Go 开发必知必会 Linux 命令速查表

```bash
# ====== 日常开发 ======
go build -o app ./cmd/server/          # 编译
GODEBUG=gctrace=1 ./app                # 查看 GC 日志
GOMAXPROCS=4 ./app                     # 限制 CPU 使用
GOGC=200 ./app                         # 调整 GC 触发百分比

# ====== 进程管理 ======
ps aux | grep <app>                    # 找 Go 进程
kill -15 <PID>                         # 优雅停止 Go 服务（触发 graceful shutdown）
lsof -p <PID>                          # 进程打开的文件和连接

# ====== pprof 性能分析 ======
# CPU profile
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof
go tool pprof cpu.prof

# Heap profile（内存）
curl http://localhost:6060/debug/pprof/heap > heap.prof
go tool pprof heap.prof

# Goroutine profile
curl http://localhost:6060/debug/pprof/goroutine > gr.prof
go tool pprof gr.prof

# Trace（调度分析）
curl http://localhost:6060/debug/pprof/trace?seconds=5 > trace.out
go tool trace trace.out

# 对比两次 profile 找增长点
go tool pprof -base heap1.prof heap2.prof

# ====== HTTP 请求耗时分析 ======
# Go httptrace：精确定位 HTTP 请求各阶段耗时
# 面试加分项：能说出"用 httptrace 排查慢请求"比"用 curl -w"更专业

import (
    "net/http/httptrace"
    "time"
)

func traceHTTPRequest() {
    var dnsStart, connectStart, tlsStart time.Time
    
    trace := &httptrace.ClientTrace{
        DNSStart: func(_ httptrace.DNSStartInfo) { dnsStart = time.Now() },
        DNSDone:  func(_ httptrace.DNSDoneInfo) {
            log.Printf("DNS lookup: %v", time.Since(dnsStart))
        },
        ConnectStart: func(_, _ string) { connectStart = time.Now() },
        ConnectDone: func(_, _ string, _ error) {
            log.Printf("TCP connect: %v", time.Since(connectStart))
        },
        TLSHandshakeStart: func() { tlsStart = time.Now() },
        TLSHandshakeDone: func(_ tls.ConnectionState, _ error) {
            log.Printf("TLS handshake: %v", time.Since(tlsStart))
        },
        GotFirstResponseByte: func() {
            log.Printf("Time to first byte: %v", time.Since(dnsStart))
        },
    }
    
    req, _ := http.NewRequest("GET", "https://api.example.com/data", nil)
    req = req.WithContext(httptrace.WithClientTrace(req.Context(), trace))
    http.DefaultClient.Do(req)
}

// 输出示例：
// DNS lookup: 3ms
// TCP connect: 15ms
// TLS handshake: 45ms
// Time to first byte: 120ms
// → 说明 TLS 和服务器处理占了大部分时间

# ====== 网络调试 ======
ss -lntp | grep <端口>                 # 确认端口监听
ss -antp | grep <端口> | wc -l         # 连接数
tcpdump -i eth0 port <端口> -w capture.pcap  # 抓包

# ====== 日志分析 ======
tail -f app.log | grep ERROR           # 实时错误
grep -C 5 "panic" app.log              # panic 上下文

# ====== 系统资源 ======
top -p <PID>                           # 看 CPU/内存
free -h                                # 系统内存
ulimit -n                              # FD 限制
cat /proc/<PID>/limits                 # 进程资源限制
```

---

> **总结**：Linux 面试核心考点是 **I/O 模型**（epoll 原理是必考）、**进程与线程**（goroutine 对比）、**TCP 握手挥手**、**文件系统（inode）**、**零拷贝原理**，以及**能不能用命令排查线上问题**。Go 方向额外关注：pprof 性能分析、goroutine 泄漏排查、netpoller 原理。掌握这些方向，加上能完整说出一个故障排查流程，Linux 面试基本没问题。
