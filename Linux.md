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

