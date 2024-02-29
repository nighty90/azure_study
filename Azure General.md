# Azure General



## REST API

+ 如果 http 请求需要附带 JSON 格式的数据，则 headers 中需要设置 `Content-Type: application/json`
  + 使用 python 的各种库时，一般可以传入 `json` 参数，此时将自动为 headers 设置
  + postman 中设置 request body 为 json 格式，也会对应设置
  + 本质是在有请求体时需要指明请求体格式，比如请求体为音频时需要指明为音频编码格式



## Authentication

+ 一般将 key 在 headers 中设为 `Ocp-Apim-Subscription-Key`
+ 有些服务可以申请临时的 token，在 headers 中设为 `Authorization`
  + 比如 speech
+ AOAI 的 key 在 headers 里设为 `api-key` 



## Container

[在本地使用 Azure AI 服务容器 - Azure AI services | Microsoft Learn](https://learn.microsoft.com/zh-cn/azure/ai-services/cognitive-services-container-support)

```bash
# 以 TTS container 为例
# 联网使用
docker run --rm -it -p 5000:5000 --memory 12g --cpus 6 \
mcr.microsoft.com/azure-cognitive-services/speechservices/neural-text-to-speech \
Eula=accept \
Billing={ENDPOINT_URI} \
ApiKey={API_KEY}

# 断网使用

```

+ 仅部分服务提供
+ 联网使用 container
  + 不要设置 license 相关的参数，否则会变为断网使用
+ 断网使用 container
  + 需要设置 license 相关的参数，且需要申请权限并购买对应承诺层，否则无法下载 / 更新 license
  + 不能完全断网，例如下载及更新 license 的时候需要联网



## APIM

+ 本质是反向代理，由它接收客户的请求并转发到对应的服务器
+ 可以记录流量等信息
+ 结构
  + client -> SLB -> APIM 实例 -> 后端
+ 概念
  + API：有唯一的 URL，
  + operation：对应一个 http 请求，有唯一的 url suffix，即 operation id
  + capacity：指示 APIM 实例的工作负载，是 CPU、内存、网络队列长度等信息的综合指标
+ kusto
  + database: ProxyRequest