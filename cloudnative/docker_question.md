# Docker常见问题汇总

## Go环境依赖问题

```Dockerfile

# 启用go module
ENV GO111MODULE=on \
    GOPROXY=https://goproxy.cn,direct

```

[go项目dockerfile最佳实践](https://www.cnblogs.com/baoshu/p/13399780.html)


## docker-compose网桥问题

