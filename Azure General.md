# Azure General



## REST API

+ 如果 http 请求需要附带 JSON 格式的数据，则 headers 中需要设置 `Content-Type: application/json`
  + 使用 python 的各种库时，一般可以传入 `json` 参数，此时将自动为 headers 设置
  + postman 中设置 request body 为 json 格式，也会对应设置
  + 本质是在有请求体时需要指明请求体格式，比如请求体为音频时需要指明为音频编码格式



## Authentication

[Authentication in Azure AI services - Azure AI services | Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/authentication)

所有服务在调用时都需要认证，主要有三种认证方式

+ 使用 resource key：在 headers 中设置 `Ocp-Apim-Subscription-Key: <key>`

  + 注意 AOAI 的 key 是在 headers 里设为 `api-key` 
  + 对于 multi 资源，需要注意地区设置
    + 对于大多数服务，向特定地区发送请求，URL 为 `<region>.api.cognitive.microsoft.com`
    + 对于 Translator，需要在 headers 中设置 `Ocp-Apim-Subscription-Region: <region>`

+ 使用 access token：在 headers 中设置 `Authorization：Bearer <token>`

  + 目前仅用于 3 个 API： Translator 的 Text Translation、Speech 的 STT 和 TTS

    + 未来可能变化，尽量参考具体服务的文档

  + 申请 token：每个 token 的有效期为 10min，考虑到网络延迟等，建议同一 token 仅使用 9min

    ```bash
    curl -v -X POST \
    "https://YOUR-REGION.api.cognitive.microsoft.com/sts/v1.0/issueToken" \
    -H "Content-type: application/x-www-form-urlencoded" \
    -H "Content-length: 0" \
    -H "Ocp-Apim-Subscription-Key: YOUR_SUBSCRIPTION_KEY"
    ```

+ 使用 Microsoft Entra ID：需要与 custom subdomain 一起使用，很罕见



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