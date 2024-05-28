---
layout: post
title: Shell Comands
categories: tools
---

### sed

#### 替换

替换目标字符串所在的一整行：
```
sed -i "s/^.*str1.*$/str2/" file
```

#### 删除
```
sed -i '1d' largeFile.txt   # 删除首行
sed -i '1,10d' largeFile.txt # 删除1-10行
sed -i '10,$d' largeFile.txt # 删除10行之后的所有行
```

### awk

```
awk '{sum+=$1} END {print sum}' file
```