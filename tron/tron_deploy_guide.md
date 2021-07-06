# Tron 部署教程

## docker 部署

```bash

sudo docker run -it -d -p 8080:8080 -p 8090:8090 -p 18888:18888 -p 50051:50051 \
           -v /data/data0/tron/docker/conf:/java-tron/conf \
           -v /data/data0/tron/docker/datadir:/java-tron/data \
           tronprotocol/java-tron \
           -jvm "{-Xmx10g -Xms10g}" \
           -c /java-tron/conf/config-localtest.conf \
           -d /java-tron/data \
           -w 

```

注意：后面的路径确保在 docker 挂载的目录下面。