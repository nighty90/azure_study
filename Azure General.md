# Azure General



### REST API

+ 如果 http 请求需要附带 JSON 格式的数据，则 headers 中需要设置 `Content-Type: application/json`
  + 使用 python 的各种库时，一般可以传入 `json` 参数，此时将自动为 headers 设置



### Authentication

+ 一般将 key 在 headers 中设为 `Ocp-Apim-Subscription-Key`
+ 有些服务可以申请临时的 token，在 headers 中设为 `Authorization`
  + 比如 speech
+ AOAI 的 key 在 headers 里设为 `api-key` 



### Container

+ 仅部分服务提供
+ 在线 container
  + 不要设置 license 相关的参数
+ 离线 container
  + 需要申请权限并购买对应承诺层
  + 不能完全断网，例如下载及更新 license 的时候需要联网
  + 需要设置 license 相关的参数