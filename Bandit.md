# Bandit Writeup

> OverTheWire Bandit 通关记录。
> 每关固定记录四项：**目标**、**命令**、**踩的坑**、**方法论**；报错原文可选，但只要卡过就值得原样留着。
> 密码统一放在 `password.md`，该文件已写进 `.gitignore`，不会被推送到 GitHub。

---

## Level 0 → 1

**目标**：家目录下有个叫 `readme` 的文件，密码在里面。

**命令**：

```bash
ls
cat readme
```

**方法论**：先 `ls` 看清目录里有什么，再动手。这一步看着多余，但后面几关的坑，根源全是"没先看清"。

---

## Level 1 → 2

**目标**：密码在一个名叫 `-` 的文件里（名字就是一个短横线）。

**命令**：

```bash
ls
cat ./-
```

**踩的坑**：直接 `cat -` 没有任何反应，像卡住了。

**原因**：在 Unix 里 `-` 是一个**约定**，表示"标准输入"。所以 `cat -` 的意思是"从键盘读数据"，它会一直等你输入，直到你按 `Ctrl+D`。它**不是**"识别不出来 `-` 是什么"，而是有一个明确的含义。

**方法论**：以 `-` 开头的参数会被程序当成"选项"。对付它有两个标准武器：

- `./文件名` —— 加个前缀，让它不再以 `-` 开头
- `--` —— 告诉程序"选项到此为止，后面都是文件名"

---

## Level 2 → 3

**目标**：密码文件名叫 `--spaces in this filename--` —— 首尾各两个短横线，中间是空格。

**命令**：

```bash
ls
cat -- *space*
```

**报错原文**（这条值得原样留着）：

```text
error: unexpected argument '--spaces in this filename--' found

  tip: to pass '--spaces in this filename--' as a value, use '-- --spaces in this filename--'
```

**踩的坑**：前三次尝试全部失败，原因只有一个 —— **凭印象打文件名**。我打的是 `spaces in this filename`，漏掉了首尾那四个短横线。

**方法论**：

1. **出问题的第一步永远是 `ls`**，而不是猜。一条 `ls` 就能看见那四个短横线，能省掉那三轮。
2. **报错里的 `tip:` / `hint:` / `try:` 常常直接给出答案** —— 上面那条报错已经把解法写出来了，只是当时没读。

---

## Level 3 → 4

**目标**：密码在 `inhere` 目录下的一个**隐藏文件**里。

**命令**：

```bash
cd inhere
ls -a
cat <文件名>
```

> 当时是用通配符 `...*Hiding*` 匹配到它的，说明真实文件名里含 `Hiding`。
> **这个真实文件名应该补上** —— 平台上的名字和常见教程不一样，这正是本关踩出来的独有收获。

**踩的坑**：`ls` 什么也不显示；试了 `ls -b`、`ls -C` 也没用。

**原因**：Unix 里的"隐藏"没有任何特殊属性，**规则只有一条：名字以 `.` 开头就不显示**。

- `-b` 改的是"不可见字符怎么显示"
- `-C` 改的是"输出排几列"
- 这两个改的都是**怎么显示**，而你需要的是 `-a`，它改的是**显示哪些**

**方法论**：目录看起来是空的，第一个动作就是 `ls -a`。读 man 手册时，先分清一个选项属于"列出哪些"还是"怎么排版"。

---

## Level 4 → 5

**目标**：`inhere` 下有一堆文件，密码在**唯一一个人类可读**的文件里。

**命令**：

```bash
cd inhere
file ./*
cat ./-file07
```

一条命令直接筛出来：

```bash
file ./* | grep text
```

**踩的坑**：我一个一个 `cat` 过去，虽然最后找到了，但二进制文件把终端刷出一片乱码。

**方法论**：

1. **用工具筛选，而不是人工遍历。** 十个文件还能手试，真实场景里可能是几千个。
2. **别随便 `cat` 二进制文件** —— 里面的转义序列真的会搞乱终端。终端花了就敲 `reset` 恢复。

---

## Level 5 → 6

**目标**：`inhere` 下有 20 个子目录，密码文件同时满足三个条件：人类可读、**1033 字节**、不可执行。

**命令**：

```bash
cd inhere
find . -type f -size 1033c
find . -type f -size 1033c -not -executable
cat <找到的路径>
```

**踩的坑**：一开始用 `du` 看大小，数字对不上，怎么都找不到 1033。

**原因**：`du` 和"文件有多少字节"是两件事。

- `du` 的默认单位是 **1024 字节的块**，所以输出 `72` 是 72 KiB，不是 72 字节；
- 更关键的是，`du` 算的是**占用了多少磁盘空间**，不是内容的字节数。磁盘按块分配，1 字节的文件通常也占 4 KiB。

| 工具 | 给你什么 |
|---|---|
| `du` | 磁盘占用（默认 KiB 块） |
| `du -b` | 表观大小，以字节计 |
| `ls -l` | 单个文件的字节数 |
| `stat -c %s 文件` | 单个文件的精确字节数 |
| `find -size Nc` | **按字节数筛选文件** ← 这一关要的 |

**方法论**：**把题目的自然语言条件，翻译成命令的参数。**

| 题目条件 | 翻译成 |
|---|---|
| 「1033 字节」 | `-size 1033c`（`c` = 字节） |
| 「不可执行」 | `-not -executable` |
| 「人类可读」 | 找到后用 `file` 验证 |

这是 `find` 最核心的用法，也是真实排查里最常用的技能。

---

## Level 6 → 7

**目标**：密码文件同时满足三个条件 —— 属于用户 `bandit7`、属于组 `bandit6`、大小 **33 字节**。而且位置是 "somewhere on the server"，不在某个目录里。

**命令**：

```bash
find / -user bandit7 -group bandit6 -size 33c 2>/dev/null
cat /var/lib/dpkg/info/bandit7.password
```

**踩的坑**：拿到 `find` 给的路径之后，我连着四次给路径"加工" —— 加 `~/`、加引号、用通配符 —— 结果全部报 `No such file or directory`。

**原因**：`~` 是家目录的简写（`/home/bandit6`）。在一个绝对路径前面加 `~/`，等于让系统去家目录里找 `/var/...`，当然找不到。**`find` 输出的路径已经是完整的，原样用。**

**报错原文**（注意报错里已经把它实际去找的路径打出来了）：

```text
bandit6@bandit:~$ cat ~/var/lib/dpkg/info/bandit7.password
cat: /home/bandit6/var/lib/dpkg/info/bandit7.password: No such file or directory
```

**方法论**：

1. **把自然语言条件逐条翻译成参数**：`owned by user X` → `-user X`，`owned by group Y` → `-group Y`，`33 bytes` → `-size 33c`。
2. **搜索起点由题目的范围决定**：`under the inhere directory` → `find .`；`somewhere on the server` → `find /`。
3. **从 `/` 搜索会刷出大量 `Permission denied`**，用 `2>/dev/null` 把错误流丢掉。
4. **工具输出的路径原样用，不要加工。**

---

## Level 7 → 8

**目标**：密码在 `data.txt` 里，紧挨着单词 `millionth`。

**命令**：

```bash
grep millionth data.txt
```

**方法论**：`grep` 的作用是**在文件里按内容找行**。它和上一关的 `find` 正好配成一对：

| 工具 | 在哪找 | 找什么 |
|---|---|---|
| `find` | **目录**里 | 文件（按名字、大小、属主） |
| `grep` | **文件**里 | 内容（按关键词） |

以后看到"在哪找什么"，先想这两个。

---

## Level 8 → 9

**目标**：`data.txt` 里有一行**只出现一次**，密码就是它。

**命令**：

```bash
sort data.txt | uniq -u
```

**踩的坑**：一开始想用 `grep` 解决，但 `grep` 只能按内容匹配，没法判断"出现了几次"。

**方法论**：

1. **管道 `|`**：把左边命令的输出，当作右边命令的输入。一个命令做不完的事，用管道把多个命令接起来。
2. **`uniq` 只比较相邻的行**，所以必须先 `sort` —— 这就是 `sort | uniq` 永远成对出现的原因。
3. 常用选项：`-u` 只留出现一次的、`-d` 只留重复的、`-c` 附带出现次数。

---

## Level 9 → 10

**目标**：密码是 `data.txt` 里的一段人类可读字符串，前面有若干个 `=`。

**命令**：

```bash
strings data.txt | grep '==='
```

**踩的坑**：第一次写成 `strings data.txt '='`，多给了一个参数 —— `strings` 后面跟的都是**文件名**，于是它去找一个叫 `=` 的文件：

```text
strings: '=': No such file
```

> 那次输出里其实已经能看到答案了，但靠眼睛在几百行里翻是不可靠的。

**方法论**：

1. `strings` 从二进制文件里**提取可打印字符串**。
2. **一个工具的输出有几百行时，用另一个工具去筛，而不是用眼睛** —— 这正是管道存在的理由。
3. **参数位置要看清楚**：`strings` 只接文件名，筛选要靠管道。

---

## Level 10 → 11

**目标**：`data.txt` 里是 base64 编码的数据。

**命令**：

```bash
base64 -d data.txt
```

**方法论**：

1. **base64 是编码，不是加密** —— 任何人都能还原，没有任何保密作用。这个区分在安全领域非常重要。
2. `-d` = 解码；不加就是编码。
3. **识别特征**：只含 `A-Za-z0-9+/`，末尾可能有 `=`，**长度是 4 的倍数**，比原文长约三分之一。

---

## Level 11 → 12

**目标**：`data.txt` 里所有大小写字母都被"**往后挪了 13 位**" —— 这就是 **ROT13**，凯撒密码的一种。

**命令**：

```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

**原理**：

- `tr` 是**字符级替换**工具：第一个集合的第 N 个字符，换成第二个集合的第 N 个字符。
- `N-ZA-Mn-za-m` 就是字母表挪 13 位之后的样子 —— 大写的 `N-Z` 接 `A-M`，小写的 `n-z` 接 `a-m`。
- 挪 13 位正好是字母表的一半，所以 **ROT13 是自逆的：加密和解密是同一个操作**，做两次就还原。

**踩的坑**：`tr` **不接收文件名**，它只从标准输入读。所以 `tr 'abc' 'xyz' data.txt` 是错的，必须用管道或 `<` 喂进去：

```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
tr 'A-Za-z' 'N-ZA-Mn-za-m' < data.txt
```

**方法论**：**遇到"字符替换"类问题先想 `tr`**；`tr 'A-Za-z' 'N-ZA-Mn-za-m'` 这一段可以当固定片段记住，看到 ROT13 就用它。

---

## Level 12 → 13

**目标**：`data.txt` 是一个**十六进制转储**，把它还原出来的文件被**反复压缩**过（gzip / bzip2 / tar 层层套）。题面还提示要在 `/tmp` 下建一个工作目录。

**命令**：

```bash
mkdir -p /tmp/work && cd /tmp/work   # 家目录不可写，必须在 /tmp 干活
cp ~/data.txt .
head data.txt                        # 先看是哪种十六进制格式
xxd -r data.txt > data.bin           # 还原成二进制（注意重定向！）
file data.bin                        # 问它是什么
```

然后进入循环：**`file` → 按结果解压 → `file` → 解压 → ……**，直到 `file` 说是 `ASCII text`。

| `file` 说是 | 用什么解 |
|---|---|
| `gzip compressed data` | `gzip -dc 输入 > 输出` |
| `bzip2 compressed data` | `bzip2 -dc 输入 > 输出` |
| `POSIX tar archive` | `tar xf 输入` |
| `ASCII text` | 到站了，`cat` 它 |

**踩的坑**（这一关坑最多，逐条记）：

1. **`xxd -r` 没重定向**，二进制直接打到终端，屏幕刷出一片乱码。二进制必须 `> 文件`。（终端花了敲 `reset` 恢复。）
2. **家目录写不了**：`-bash: data.bin: Permission denied`。这是权限设计，不是故障 —— 去 `/tmp` 干活。
3. **概念错误**：我想把二进制"转成十进制"。二进制**文件**不是二进制**数**，没有进制可换，它只能被解压。
4. **`xxd -r -p` 用错了格式**：`-p` 对应"没有地址列的纯十六进制"，用在普通 hexdump 上会让整份数据错位，`file` 于是报 `data`（认不出来）。
5. **`gunzip data.gz` 之后原文件消失了**：解压类命令默认"就地替换"，`x.gz` → `x`。想保留用 `-k` 或 `-dc`。
6. **顺序反了**：应该"先改名、再解压"，我有几次是先解压、后改名，晚了一步。
7. **`bunzip2 data` 自己起了个名字 `data.out`** —— 其实提示里写得很清楚：

```text
bunzip2: Can't guess original name for data -- using data.out
```

8. **把已经解压过的文件又拿去解压**：`data.out` 是解压后的**结果**，再 `bunzip2` 就报 `not a bzip2 file`。**改名前必须先 `file`。**
9. **`Output file data already exists`**：解压工具默认不覆盖已有文件，这是保护机制，不是故障。用 `-c` 指定输出名就能绕开。

**方法论**：

1. **`file` 读的是文件内容（开头的魔数），不是扩展名。** 扩展名只是给人的提示 —— 所以 **`mv` 之前必须先 `file`**。
2. **`file *` 是最快的"现状体检"**：一条命令看清目录里所有文件的真实格式，再决定动哪个。
3. **报错和提示信息里常常直接写着答案**（`using data.out` 那句就是），值得逐字读。
4. 用 `-dc` + 重定向，可以**完全绕开"改扩展名"这一步**，也不会误删原文件。
5. **不知道下一步做什么时，问 `file`，不要猜。**

---

## Level 13 → 14

**目标**：家目录里有一个**私钥文件** `sshkey.private`，用它登录下一关即可。题目提示：`localhost` 指"你正在操作的这台机器"。

**命令**（注意：要在**你自己的电脑**上执行，不能在 bandit 会话里连 localhost）：

```powershell
scp -P 2220 bandit13@bandit.labs.overthewire.org:sshkey.private bandit14.key
ssh -i bandit14.key bandit14@bandit.labs.overthewire.org -p 2220
```

登进去之后：

```bash
cat /etc/bandit_pass/bandit14
```

**踩的坑**（这一关踩了四层）：

1. **忘了 `-p 2220`**，连到默认的 22 端口，服务器的横幅直接告诉你端口不对。
2. **补上端口后，被"禁止从 localhost 登录"挡住**：

```text
!!! You are trying to log into this SSH server from localhost.
!!! Connecting from/to localhost is blocked to conserve resources.
!!! Please log out and log in again, directly from your client machine.
```

**网上所有老教程写的 `ssh bandit14@localhost` 现在都失效了** —— 服务器自己给出了出路：从你自己的电脑直连。

3. **家目录不可写**，于是有这两句警告（**无害**，只是没法把对方指纹写进 `known_hosts`）：

```text
Could not create directory '/home/bandit13/.ssh' (Permission denied).
Failed to add the host to the list of known hosts (...)
```

4. **以 bandit13 的身份去 `cat /etc/bandit_pass/bandit14` 会 Permission denied —— 这是设计，不是错误。** 正是它逼你换成 bandit14 的身份。

**方法论**：

1. **SSH 密钥登录 = 用文件代替密码**，`-i` 指定用哪把私钥。私钥权限必须是 `600`，ssh 会主动检查（这是安全设计，不是刁难）。
2. **`scp` 用大写的 `-P` 指定端口，`ssh` 用小写的 `-p`。** 一个字母的大小写，错了就连不上。
3. **教程会过时，服务器不会骗你。** 所有老教程都说用 `localhost`，但服务器明确拒绝并且自己说明了该怎么办 —— 冲突时以实际返回为准。

---

## Level 14 → 15

**目标**：把当前密码提交给 **localhost 的 30000 端口**，它会返回下一关的密码（明文 TCP，未加密）。

**命令**：先登进 bandit14（在你自己的电脑上）：

```bash
ssh bandit14@bandit.labs.overthewire.org -p 2220
```

进去之后，在 bandit14 的会话里，把密码发给本机的 30000 端口：

```bash
echo <bandit14的密码> | nc localhost 30000
```

**踩的坑**：我在自己的电脑上敲了 `ssh -p 30000 bandit14@bandit.labs.overthewire.org`，结果：

```text
ssh: connect to host bandit.labs.overthewire.org port 30000: Connection timed out
```

错了两处：

- **位置错了**：这个服务只监听游戏服务器自己的 `127.0.0.1`，公网根本看不到它。
- **工具错了**：30000 端口后面不是 SSH 服务，是裸 TCP 服务。用 `ssh` 去连，等于拿螺丝刀去拧一字螺丝。

题目里那个 **"on localhost"** 就是在提示你：要先登进去，在那个会话里连。

**方法论**：

1. **"localhost 上的服务" = 只有那台机器自己能连。** 数据库、管理接口通常都这么部署 —— 这是真实世界最常见的攻击面之一。
2. **端口后面跑什么服务，决定你用什么工具。**

| 端口后面是什么 | 你用什么 |
|---|---|
| SSH 服务 | `ssh` |
| Web 服务 | 浏览器 / `curl` |
| 数据库 | 对应的客户端 |
| **不知道是什么** | **先用 `nc` 手动试** |

3. 遇到陌生端口，第一反应不是"我该用什么工具"，而是"**先 `nc` 上去看看它会说什么**"。

---

## Level 15 → 16

**目标**：把当前密码提交给 **localhost 的 30001 端口**，这次要求 **SSL/TLS 加密**。

**命令**：

```bash
echo <bandit15的密码> | openssl s_client -connect localhost:30001 -quiet
```

**踩的坑**：只写了主机名，没写端口：

```text
BIO_connect:Connection refused... connect:errno=111
```

`-connect` 的格式是 **`主机:端口`**。只给主机，OpenSSL 就用了它的默认端口 **443** —— localhost 的 443 上没有服务，于是被拒绝。

**方法论**：

1. **`-connect` 必须写成 `主机:端口`。** 省略端口时，工具会用它自己的默认值（OpenSSL 是 443），结果和你的预期毫无关系。
2. **`Connection refused` 和 `Connection timed out` 是两种不同的失败，含义完全不一样：**

| 报错 | 含义 | 说明你 |
|---|---|---|
| `Connection refused`（errno 111） | 包**到达了机器，但那个端口没人监听**，被明确拒绝 | **端口写错了** |
| `Connection timed out` | 包**没得到任何回应**，被丢掉了 | **地址写错了**，或被防火墙挡了 |

**看是哪种错，能立刻把排查范围缩小一半。**

3. `s_client` 连上之后有一小套自己的交互命令（`DONE`、`RENEGOTIATING`、`KEYUPDATE`）。响应里出现这些词，说明你的输入被当成了 s_client 的命令、而不是发给服务器的数据 —— 题面那句提示说的就是这个坑。`-quiet` 用来关掉证书那一大堆噪音。

---

## 反复出现的几个模式

通关十几关下来，错的种类其实很少，翻来覆去就这几种：

1. **报错和提示信息里常常直接写着答案。** `tip: use '-- ...'`、`using data.out`、`port 22 is not intended`、`Connecting from localhost is blocked` —— 全都明说了。**逐字读报错，比查教程快。**
2. **先看真相，再动手。** `ls` 看有什么、`file` 看是什么格式、`file *` 做现状体检。凭印象猜文件名、猜格式、猜路径，是绝大部分失败的来源。
3. **工具输出的东西原样用。** `find` 给的路径、`file` 给的文件名 —— 加 `~/`、加引号、换扩展名，都是在给自己挖坑。
4. **端口后面是什么服务，决定用什么工具。** SSH / HTTP / 裸 TCP / TLS 各有各的客户端；不知道是什么就先 `nc`。
5. **教程会过时，服务器不会骗你。** 老教程的 `localhost` 写法已被服务器明确禁止 —— 两者冲突时，以实际返回为准。

---

**进度**：Level 0 → 16 已完成。
