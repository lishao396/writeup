# Forensics Writeup

> 分类：Forensics
> 日期：2026-10-18
> 工具：`file`、Python 3、Wireshark 4.6.6、`tshark`、`base64`

---

## 1. Binary Digits

**目标**：把全是 `0`/`1` 的 ASCII 文本还原成 JPEG，打开图片读取 flag。

### 关键判断

先用三条命令确认它到底是什么：

```bash
file digits.bin
head -c 80 digits.bin
wc -c digits.bin
```

- `file` 说是 `ASCII text`；
- 内容只有字符 `0` 和 `1`；
- 总长度是 71728，能被 8 整除；
- 71728 ÷ 8 = 8966，正好是一个小型 JPEG 的大小。

这里要分清“字符”和“字节”：

- `digits.bin` 里每个 `'0'` 或 `'1'` 是一个**字符**，在磁盘上占 1 个字节（ASCII `0x30` / `0x31`）；
- 每 8 个字符表达的是 8 个**比特**，还原成 1 个真正的字节；
- 所以原文件是 71728 字节，还原后是 8966 字节，正好缩小 8 倍。

### 还原脚本

```python
# 输入：全是 '0'/'1' 字符的文本文件
src = "digits.bin"

# 读成一个大字符串，strip() 去掉首尾空白
data = open(src).read().strip()

# 只保留 '0' 和 '1'，去掉换行、空格等
data = "".join(ch for ch in data if ch in "01")

# 检查长度：71728 能被 8 整除
print(len(data), len(data) % 8)

# 每 8 个字符切一组
chunks = [data[i:i + 8] for i in range(0, len(data), 8)]

# 每组按二进制解析成 0~255 的整数
nums = [int(c, 2) for c in chunks if len(c) == 8]

# 整数列表 -> 字节串
blob = bytes(nums)

# 以二进制模式写入；b 不能省
with open("out.bin", "wb") as f:
    f.write(blob)

print("written:", len(blob), "bytes")
```

运行并验证：

```bash
python3 restore.py
file out.bin
```

`file out.bin` 输出 `JPEG image data`，再把文件改名或直接用图片查看器打开即可：

```bash
mv out.bin out.jpg
```

### 容易混的地方

- `src`、`data`、`chunks`、`nums`、`blob` 都是**自己起的变量名**；
- `int()`、`bytes()`、`len()`、`range()`、`print()` 是 Python **内置函数**；
- `for` 是**关键字**，不是函数；
- `chunks` 不是内置函数，拼写是 `chunks`，不要写成 `chuncks`；
- `int(c, 2)` 里的 `2` 表示“按二进制解析”，不是长度；
- `if len(c) == 8` 会悄悄丢掉最后不足 8 位的残尾。这道题 71728 能被 8 整除，所以安全；换一道题最好先检查或直接报错。

**小结**：文件类型由字节内容和魔数决定，不由扩展名决定。二进制文本、十六进制、Base64 都只是字节的不同表示方式；看到“只有 `0`/`1` 且长度能被 8 整除”，就应该想到“这可能是某个文件的二进制转写”。

---

## 2. Ph4nt0m 1ntrud3r

**目标**：分析 pcap，找出藏在 TCP SYN 载荷里的 Base64 片段，按时间顺序还原 flag。

### 先看全局

用 Wireshark 打开 `myNetworkTraffic.pcap`：

1. `Statistics > Protocol Hierarchy`：22 个包全是 TCP，没有 HTTP、DNS 等其他协议；
2. `Statistics > Conversations`，切到 `TCP` 标签：只有一条会话 `10.200.0.2:20 ↔ 192.168.1.2:80`，共 22 个包、约 1 kB。

这说明不用再找别的 IP 或端口，异常就在这一条流里。

### 找异常

正常情况下，TCP 的 `SYN` 包只用于三次握手，**不该携带应用层数据**。但这里 22 个包全是 `SYN`，四元组相同，`Seq=0`，而且每个包都带了 4~12 字节的载荷。

Wireshark 会在 Info 列显示 `[TCP Retransmission]`，因为所有包的序号都是 0。因此不要直接用 `Follow > TCP Stream` 下结论：它按 TCP 序号重组，而这里的序号全是 0，拼出来的顺序不可信。

### 按 `tcp.len` 分组

在包列表里加一列 `tcp.len`：

1. 点一个包；
2. 展开 `Transmission Control Protocol`；
3. 右键 `TCP Segment Len` → `Apply as Column`。

会看到载荷长度只有三种：

| `tcp.len` | 包数 | 每段 Base64 解出 | 解出后可打印的字节 | 判断 |
|---|---:|---:|---:|---|
| 8 | 15 | 5 字节 | 24 / 75（约 32%） | 掩护/噪声 |
| 12 | 6 | 7 字节 | 42 / 42（100%） | 真数据主体 |
| 4 | 1 | 1 字节 | 1 / 1（100%） | 结尾字符 |

注意 `tcp.len` 是 TCP 载荷长度，不是包列表里的 `Length`。整帧长度是这样组成的：

- 8 字节载荷：20 IP 头 + 20 TCP 头 + 8 = 48；
- 12 字节载荷：20 + 20 + 12 = 52；
- 4 字节载荷：20 + 20 + 4 = 44。

8 字节的包虽然最多，但解出来大部分是不可打印字节，像掩护流量；12 字节和 4 字节的包解出来是可读文本。**信号是少数，噪声是多数**，不要因为 8 字节的包多就以为它们才是真的。

### 提取与解码

先按 `Time` 列排序。包列表默认是文件顺序，不是时间顺序；

过滤出有效载荷：

```text
tcp.len == 12 || tcp.len == 4
```

状态栏应显示 7 个包（22 个里的 31.8%）。

**图形界面方法**（个人觉得这个更快一点🥴）：

1. 点一个包，展开 `Transmission Control Protocol`；
2. 点 `Data (12 bytes)`，看底部状态栏确认选中的是 `Data`，不是 `Checksum`；
3. `Analyze > Show Packet Bytes`；
4. `Decode as` 选 `Base64`，`Show as` 选 `ASCII`；
5. 每个包单独解码，再按 `Time` 顺序把解码后的文本接起来。

**命令行方法**：

```bash
cd ~/ctf/pico        ##放置文件的目录

tshark -r myNetworkTraffic.pcap \
  -Y "tcp.len == 12 || tcp.len == 4" \
  -T fields -e frame.time_epoch -e tcp.payload \
  | sort -n \
  | cut -f2 \
  | while read -r hex_payload; do
      printf '%s\n' "$hex_payload" | xxd -r -p
      printf '\n'
    done > payloads.txt
```

这条命令做的是：

- `-Y`：显示过滤器，只保留 12/4 字节的包；
- `-T fields`：只输出指定字段；
- `-e frame.time_epoch`：时间戳，用于排序；
- `-e tcp.payload`：TCP 载荷，以十六进制输出；
- `sort -n`：按时间戳从小到大排序；
- `cut -f2`：只保留十六进制载荷；
- `xxd -r -p`：把十六进制还原成字符，也就是 Base64 文本；
- `> payloads.txt`：每段 Base64 存一行。

然后逐段解码：

```bash
while read -r b64; do
  printf '%s' "$b64" | base64 -d
done < payloads.txt
```

`while read` 每次读一行；`printf '%s'` 不额外加换行；`base64 -d` 只解码当前这一段。**不要先把所有 Base64 字符串拼成一条长串再解码**：每段自带 `=` 填充，直接拼会在中间出现填充符，解码器会报错。

**小结**：先看 `tcp.len` 分布，再抽样解码判断可打印性，最后按时间顺序逐段解码并拼接。换一道题，长度不一定是 12/4，但“分组 → 抽样 → 判断可读性 → 重排 → 逐段解码”的方法是通用的。
