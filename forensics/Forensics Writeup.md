---
















---
---
## 1.Binary Digits

目标：利用python把全是0/1的一个ASCII text转化为.jpg 根据图像识别flag

**步骤**：

(1)`file`发现格式为ASCII text 但是文本内容全部是01 长度又能被 8 整除，就可以怀疑：“这会不会是某个文件的二进制转写“

(2)尝试:
```
data = open(src).read().strip()
```

因为全是10 直接尝试每八个字节为一个字符串(提前得知总数字是八的数):

```
chunks = [data[i:i + 8] for i in range(0, len(data), 8)]
```

(3)把字符串按【二进制】解析成整数:
```
nums = [int(c, 2) for c in chunks if len(c) == 8]
blob = bytes(nums)
```

(4)创建文件导入:
```
with open("out.bin", "wb") as f:
	f.write(blob)
```

**小结**：主要在于全是01要转化 以及python的应用 其中`int（）` `bytes()` `for` `chuncks`要会用

---

## 2.Ph4nt0m 1ntrud3r

**目标**：将数据用base64解码 从而找到flag

**步骤**：

（1）**前戏**：
 用wireshark打开文件 
 将文件按时间排序

（2）**观察**：
 经过观察发现tcp.len长等于12/4右下角ASCII区域有一部分数据与base64编码结果很相似（len长为8的解码不出正常结果）
 
（3）**解码**：
  选取右下角ASCII特殊区域 点击“analyze”中的“show packet bytes”
  *Decode as设为base64*
  *show as设为ASCII*
  将每一个包的该区域按时间顺序解码拼接即可得到flag

**小结**：要寻找到特殊包 并且对base64解码有一定的敏感度 


