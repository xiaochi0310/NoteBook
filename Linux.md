### Linux学习

------

#### 1、常用命令

查看文件的第几行

```shell
# 查看main.go的第5行
sed -n '5p' main.go 
# 查看main.go的第5到10行
sed -n '5,10p' main.go
```

字符串替换:全文替换将str1替换为str2

```
:%s/str1/str2/g
```

压缩解压

```
tar -zcvf pack.tar.gz pack/  #打包压缩为一个.gz格式的压缩包
tar -zxvf pack.tar.gz /pack  #解包解压.gz格式的压缩包到pack文件夹
```

