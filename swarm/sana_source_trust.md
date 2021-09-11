# Sana 源码解读：Trust协议

在上篇文章有讲过，Sana的挖矿启动流程。其中，启动挖矿的时候，会创建Trust协议。协议定义方法如下：

**pkg/mine/trust/trust.go**

```go
func (s *Service) Protocol() p2p.ProtocolSpec {
	return p2p.ProtocolSpec{
		Name:    protocolName,
		Version: protocolVersion,
		StreamSpecs: []p2p.StreamSpec{
			{
				Name:    streamSign,
				Handler: s.handlerSign,
			},
			{
				Name:    streamSignRet,
				Handler: s.handlerSignRet,
			},
			{
				Name:    streamRollCall,
				Handler: s.handlerRollCall,
			},
			{
				Name:    streamRollCallSign,
				Handler: s.handlerRollCallSign,
			},
		},
	}
}
```

