linux常用命令

# 端口占用

```
netstat -tlpn
```


# ubuntu免密码

```
sudo vi  /etc/sudoers

```

// 添加代码

ubuntu  ALL=(ALL) NOPASSWD: ALL

# key.pem　登入

```shell

sudo chmod 600 key.pem 

ssh -i key.pem root@ip

```