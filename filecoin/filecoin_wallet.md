# Filecoin 钱包功能总结

## 钱包类型

Filecoin 有两种地址类型，分别是　f1 (secp256k1) or f3 (BLS)。

## 命令行工具

在全节点的机器执行，创建　bls　类型钱包

```bash

lotus wallet new bls

```

创建　secp256k1　类型钱包

```bash

lotus wallet new

```

##　JSON-RPC

go　语言使用demo

```go

import (
	"context"
	"fmt"
	"log"
	"net/http"

	jsonrpc "github.com/filecoin-project/go-jsonrpc"
	lotusapi "github.com/filecoin-project/lotus/api"
)

func main() {
	authToken := "<value found in ~/.lotus/token>"
	headers := http.Header{"Authorization": []string{"Bearer " + authToken}}
	addr := "127.0.0.1:1234"

	var api lotusapi.FullNodeStruct
	closer, err := jsonrpc.NewMergeClient(context.Background(), "ws://"+addr+"/rpc/v0", "Filecoin", []interface{}{&api.Internal, &api.CommonStruct.Internal}, headers)
	if err != nil {
		log.Fatalf("connecting with lotus failed: %s", err)
	}
	defer closer()

       // Now you can call any API you're interested in.
	tipset, err := api.ChainHead(context.Background())
	if err != nil {
		log.Fatalf("calling chain head: %s", err)
	}
	fmt.Printf("Current chain head is: %s", tipset.String())
}

```


参考：

[Filecoin API 文档官方](https://docs.filecoin.io/reference/lotus-api/)

[Filecoin RPC API文档](http://cw.hubwiz.com/card/c/filecoin-lotus-rpc/)

[Lotus: send and receive FIL](https://docs.filecoin.io/get-started/lotus/send-and-receive-fil/#about-wallet-addresses)