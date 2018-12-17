
# 打造基于 `netty` 的高效 `rpc` 异步 `Server` 例子
🍎 基于 `netty4` 构造一个简单的异步 `rpc` 远程调用例子。本例将详细介绍基于 `netty` 的异步请求模型。 本例子不包含远程服务注册与发现、 编解码细则、 动态代理等。
作者考虑打造一个系列，会分开进行分析讲解

# 需要考虑的点

1. 服务端客户端消息接收与发送
1. 编解码器 handler
1. 异步处理思路
1. 心跳机制 
客户端发起心跳，收发消息响应，需要两个handler。服务端只需要回应即可. 如果服务端读超时，也会关闭。服务端不会写。
1. tcp 粘包、拆包  处理
消息头（包含消息长度） + 消息内容 
1. 异常处理
如果服务端抛出异常后，会把当前连接给关闭了。
所以当前连接将不会再处理成功，除非重连。
1. channel 重连机制， add remove. 启动时连接重试
每一次send之前
Done. -> check channel 