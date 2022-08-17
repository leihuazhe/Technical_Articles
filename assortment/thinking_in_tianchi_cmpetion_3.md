# 进阶版作品思路

[https://tianchi.aliyun.com/competition/entrance/531985/information](https://tianchi.aliyun.com/competition/entrance/531985/information)

1. 基于 Serverless 云服务的多租隔离能力，将示例扩展为多租的 IDE SaaS 服务。多租隔离包括但不限于：不同租户的WebIde Server函数实例隔离、不同租户代码存储到OSS的数据安全保证、WebIde Server的身份认证保请求是对应租户的。
2. 优化数据保存和恢复策略，即使用户不小心关闭了 IDE 页面，或者运行 IDE 的实例发生故障，用户仍可以快速恢复到之前的状态。
3. 优化 Web IDE 实例的启动性能，随时随地，秒开 IDE ！
4. 优化成本。Open VSCode 基于Websocket协议，这种长连接在 Serverless 下有持续计费问题。通过管控侧或者对VSCode Server代码改造，优化资源利用率，降低成本。

注：参赛方案语言不限制。

***针对以上4点我的优化思路。***

- 针对第1点，从不同用户的请求 request header 中获取用户标志，然后根据用户标志，拼装新的 oss_key，将每个用户的workspace 和 history 等操作信息以不同的 key 保存到 aliyun OSS 上。而 fc 本身配置支持多实例并发，因此不同用户调用，通过用户标志，即路由到不同实例上，租户之间做到了隔离。

- 针对第2点，代码里对 fc 函数 添加 pre-freeze 生命周期，设置60s超时时间，当60s内没有请求时，将会冻结实例，冻结实例时保存 vs_code 各用户的 user 和 workspace 信息。
- 针对第3点，采用 fc 异步调用方式，取代同步调用方式，见文档说明：[https://help.aliyun.com/document_detail/193056.html](https://help.aliyun.com/document_detail/193056.html)
- 对第4点，在 构造 `ReverseProxy` 时，增加自定义参数信息，缩小 `IdleConnTimeout` 的超时时间，一段时间没有 client 的响应后，对实例进行 freeze。(目前git旗下 gitpod 即这么做的)。

## 实践问题

1.我目前采用阿里云提供的 go sdk 进行代码编写后，有各种意外的报错问题，比如最典型的 FC_RUNTIME_API  设置问题找不到文档，其对应的函数 fc 触发的 trigger 又没有明细说明。

该 go sdk 部分代码和 aws 的 go sdk 重复，目测可能是从 aws 的 lambda go sdk clone 改造而来。

2.在参赛页面，提交记录后，一直无法出结果，已经过去了3天左右，仍然没有出结果，同时又不能进行新的提交。

3.我加入了钉钉比赛群后，发现比赛群里群聊消息寥寥无几。

综上：结合以上诸多问题，感觉举办方心有余而力不足，重视程度不够，目前整体体验较差，可能咱们只能熟悉一下，并提出一些解题思路，无法真正的去完成比赛了。

1.添加 pre-freeze 生命周期。

2.采取异步调用函数进行数据的快速响应。

vscode 采用 websocket 时，使用 Custom Container 处理。