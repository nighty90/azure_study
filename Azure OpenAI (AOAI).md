# Azure OpenAI (AOAI)

[Azure OpenAI Service - Documentation, quickstarts, API reference - Azure AI services | Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/openai/)

## Notes

+ 注意实际输入给 GPT 模型的 prompt 是包含一定量的上下文的
+ `max_token` 指示了向模型申请的计算资源的大小，因此是计费的依据



## Overview

### Basics

+ 基本概念：提供 OpenAI 语言模型的 API（REST API、Python SDK、Azure OpenAI Studio）
+ responsible AI：为防止 AI 生成错误及有害内容，Microsoft 采取了一系列措施
  + 要求申请人展示妥善定义的用例
  + 融入 Microsoft 的 responsible AI 使用原则
  + 生成内容筛选器（Content Filter，以下简称 CF）以支持客户
  + 向已加入的新客户提供 responsible AI 实施指导
+ 访问 Azure OpenAI：因需求量大而限制访问中，需要申请（https://aka.ms/oai/access）
+ Azure OpenAI 与 OpenAI
  + Azure OpenAI 为 Open AI 的模型额外提供安全性和企业承诺
  + Azure OpenAI 提供：专用网络、区域可用性、Responsible AI 内容筛选



### Key concepts

+ Prompts 与 completions
+ Tokens：Azure OpenAI 将文本分割为 token 来处理
  + 可以是完整的单词，但通常仅是字符块（比如一个音节）
  + 处理的 token 总数取决于 [ 输入、输出、请求参数 ] 的长度
  + token 数量会影响响应延迟与通量
+ Resources：如同其他 Azure 产品，Azure OpenAI 需要创建 resource 来使用
+ Deployments：创建 resource 后，需要先将模型部署，再开始调用 API
+ Prompt engineering：GPT 系列模型基于 prompt，故需要调整 prompt 以更好地使用
+ models：Azure OpenAI 提供了一系列模型的接口
  + 文本输入：GPT 系列（文本生成）、Embeddings（文本编码）、DALL-E（文生图）
  + 音频输入：Whisper（语音转录）



### Content Filter（CF）

+ 分类模型的集合，是与核心模型分开的一套系统
+ 用户的输入和模型的输出都会经过 CF 进行检查
+ 创建修改过的 CF 的权限需要额外申请
  - 申请表：[Azure OpenAI Limited Access Review: Modified Content Filtering (microsoft.com)](https://customervoice.microsoft.com/Pages/ResponsePage.aspx?id=v4j5cvGGr0GRqy180BHbR7en2Ais5pxKtso_Pz4b1_xUMlBQNkZMR0lFRldORTdVQzQ0TEI5Q1ExOSQlQCN0PWcu)
  - 创建时将 Filters and annotations 设置为 off，可将该 CF 设置为关闭状态
  - 对于大概 2023/12/06 之后创建 off CF，其效果等同于此前的 Microsoft.Nil CF
    - 完全不经过 CF，没有过滤造成的延迟，响应中没有过滤相关内容
    - 在这之前创建的 off CF 则是仍经过 CF，但不过滤内容，响应中依旧有过滤相关内容



## GPT prompt engineering

### Basics

+ GPT 系列模型基于 prompt，对其敏感，故需要 prompt engineering
+ 可以将 prompt 的作用理解为配置模型的权重以适应特定任务的需要
+ 类比人类，GPT 的行为类似于“给定 prompt，回复第一件据此联想到的事情”，本质词语接龙
  + 所以知名语段的续写会很准确
  + 注意模型给出的始终是“最可能跟在 prompt 后出现的文本”，而非真正的理解并回复



### Prompt components

+ 注意点
  + Completion 与 Chat Completion 的不同
    + Completion：prompt 的各部分之间没有区别
    + Chat Completion：prompt 由一系列不同角色的消息组成（system / user / assistant）
  + 注意，以下将 prompt 分解为 components 更多是为了帮助构建 prompt，对于模型它们没有本质差别
+ instructions：指示模型应该做什么
+ primary content：需要模型处理的文本
  + 例子：需要翻译、总结、润色等处理时，提供给模型的原文
  + GPT 也可处理结构化文本，如 tsv 表格
+ examples：提供给模型参考的示例
  + 通常给模型提供 1 个或几个例子，称为 one-shot / few-shot learning
    + 对于 Chat Completion，few-shot 通常以 user-assistant 交互的形式输入示例
  + 对应地，不提供例子时称为 zero-shot learning
+ cue：帮助模型输出的提示，通常为提供给模型续写的前缀
+ supporting content：帮助模型构筑输出的其他信息
  + 例子：当前日期、用户偏好等上下文信息
  + 注意与 primary content 不同，supporting content 本身不是模型需要直接处理的文本



### Best practices

+ Be Specific：减少歧义，限制可能的输出空间
+ Be Descriptive：使用类比
+ Double Down：有时需要重复提供给模型相近的内容，如反复强调 instruction
+ Order Matters：输入的顺序会影响输出（recency bias）
+ Give the model an “out”：有时任务无法完成可能，最好给模型提供应对该情况的做法
  + 避免模型为了强行完成而胡编乱造



### Space efficiency

+ 模型能处理的 token 数量有大小限制（具体查文档），需要注意输入的大小
+ 注意有时候 token 的分割比较反直觉
  + `October 18 2022` -> `[October] [ 18] [ 2022]`，3 个 token
  + `10-18-2022` -> `[10] [-] [18] [-] [20] [22]`，6 个 token
+ 技巧
  + 使用表格：与需要为每个值指明名称的格式（如 JSON）相比，表格更省空间
  + 注意空格：连续的空格会被单独分割为各个 token，而单词前的空格会与单词分割在一起



## REST API

### Authentication

将 API Key 或 Microsoft Entra token 设置到 HTTP header 中

+ 通过 API Key：设置 `api-key` 参数

+ 通过 Microsoft Entra token

  ```bash
  # 保存 user 信息
  export user=$(az account show --query "user.name" -o tsv)
  
  # 将自己的角色设置为 Cognitive Services User
  # assignment 需要约 5min 生效
  export resourceId=$(az group show -g $RG --query "id" -o tsv)
  az role assignment create --role "Cognitive Services User" --assignee $user --scope $resourceId
  
  # 请求 Microsoft Entra token 
  # 注意 1h 后过期，之后需要再次请求
  export accessToken=$(az account get-access-token --resource https://cognitiveservices.azure.com --query "accessToken" -o tsv)
  
  # 调用 API
  curl $AZURE_OPENAI_ENDPOINT/openai/deployments/$DEPLOYMENT_NAME/completions?api-version=2023-05-15 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $accessToken" \  # 在这里包含，注意有固定前缀
  -d '{ "prompt": "Once upon a time" }'
  ```



### Completions

```http
POST https://{your-resource-name}.openai.azure.com/openai/deployments/{deployment-id}/completions?api-version={api-version}
```

+ URL 路径要素

  + `your-resource-name`：AOAI 资源的名称
  + `deployment-id`：部署了对应模型的 deployment 的名称

+ URL 参数

  + `api-version`：API 的版本，具体见文档

+ 请求体

  + `prompt`：string / array，输入模型的 prompt
    + 默认值为 `<\|endoftext\|>`，此时模型认为要为空白的新文档生成文本
  + `max_tokens`：integer，补全的最大 token 数
    + 本质上请求了需要占用的计算资源，是实际计算使用的 token 数的依据
    + 当该值较大时，模型延迟也会较大
    + 注意 [ prompt 长度 + max_tokens ] 不能超过模型 context 长度
    + context 长度通常为2048，新模型为 4096
    + 默认
  + `temperature`：number，采样温度，范围为 0 - 2，默认为 1
    + 数值越低，模型就越会选择概率高的 token，为 0 时模型只输出概率最大的 token
    + 不推荐同时改变该参数与 `top_p`
  + `top_p`：number，核心采样（nucleus sampling）的阈值，默认为 1
    + 将概率降序排序，模型会在前缀和第一次大于 top_p 的 token 之间进行选择
    + 不推荐同时改变该参数与 `temperature`
  + `logit_bias`：map，修改补全中特定 token 的概率，默认为 null
    + 可接收 json 对象，将 token 映射为对应偏置值
    + 这里 token 具体来说指的是 GPT tokenizer 给出的 token ID
    + 偏置值范围为 -100 - 100，在采样前被加到模型生成的 logits 上
  + `user`：string，终端用户的标识符，可用于监测滥用情况，默认为空
  + `n`：integer，各 prompt 生成的补全数量，默认为 1
    + 注意会大量消耗 token 配额，请确保已正确设置 `max_tokens` 和 `stop`
  + `stream`：boolean，是否以数据流的形式返回，默认为 False
    + 一旦 token 可用便会以仅含数据的 event 的形式发送，直到遇到 `data: [DONE]`
  + `logprobs`：integer，设置每个位置返回的最可能的 token 个数，默认为 null
    + 仅当不为 null 时启用，返回的 response 里也包含每个 token 的对数概率
    + 该参数不适用于 `gpt-35-turbo`
  + `suffix`：string，补全的后缀，默认为 null
  + `echo`：boolean，是否也输出 prompt，默认为 False
    + 该参数不适用于 `gpt-35-turbo`
  + `stop`：string / array，设置终止信号（不包含在输出中），长度最多为 4，默认为 null
  + `presence_penalty`：number，根据是否已出现惩罚 token，范围为 -2.0 - 2.0，默认为 0
    + 注意正值为惩罚，负值为鼓励
  + `frequency_penalty`：number，根据频率惩罚 token，范围为 -2.0 - 2.0，默认为 0
    + 注意正值为惩罚，负值为鼓励
  + `best_of`：integer，设置服务器端生成的补全个数，默认为 1
    + 注意仅返回最佳补全，即平均 token 对数概率最低的补全
    + 无法与 `stream` 同用
    + 与 n 同用时，注意 `best_of` 一定要大于 n
    + 注意 token 配额的消耗
    + 该参数不适用于 `gpt-35-turbo`

  

### Embeddings

```http
POST https://{your-resource-name}.openai.azure.com/openai/deployments/{deployment-id}/embeddings?api-version={api-version}
```

+ URL 路径要素及参数：同 Completions
+ 请求体
  + `input`：string / array，必须参数，需要编码的文本
    + 支持的 token 数因模型而异
    + 仅 `text-embedding-ada-002 (Version 2)` 支持 array 输入
  + `user`：同 Completions，注意不要传递 PII 标识符，请使用伪匿名化值，如 GUID



### Chat completions

```http
POST https://{your-resource-name}.openai.azure.com/openai/deployments/{deployment-id}/chat/completions?api-version={api-version}
```

+ 路径参数：同 Completions

+ request 内容：基本同 Completions，以下列出不同点

  + 没有 `prompt`，而是使用 `messages`
    + array，元素为 [chat message](#ChatMessage)，必须，输入给模型的一系列上下文
  + `max_tokens` 默认为 inf，此时实际上能返回的最多 token 数 = 4069 - prompt 的 token数
  + `function_call`：非必须，控制模型对函数的调用情况
    + "none"：模型不调用函数，回复终端用户，未提供 functions 参数时该值为默认值
    + "auto"：模型可回复终端用户或调用函数，提供了 functions 参数时该值为默认值
    + 以 `{"name": "my_function"}` 的形式指定函数：要求模型必须调用该函数
    + 需要 API 版本为 2023-07-01-preview，且非所有模型都支持
  + `functions`：FunctionDefinition[]，非必须，一系列可供调用的函数，其输入由模型生成
    + 需要 API 版本为 2023-07-01-preview
  + 其他没有的参数：suffix、echo、best_of

+ 相关数据类型（JSON 格式）

  + **ChatMessage**

    | Name          | Type         | Required | Description                                                  |
    | :------------ | :----------- | -------- | :----------------------------------------------------------- |
    | content       | string       | required | 该 message 的内容                                            |
    | function_call | FunctionCall | optional | 需要调用的函数及其参数                                       |
    | name          | string       | optional | 该 message 作者的名字； <br>如果 role 为 function 则必须提供，值为对应的函数名； <br>最长 64 字符 |
    | role          | ChatRole     | required | 该 message 的 role                                           |

  + **ChatRole**（四选一）

    + assistant：提供回复
    + function：提供函数的调用结果
    + system：指示或设置 assistant 的行为
    + user：提供 prompt 输入

  + **FunctionCall**（注意需要 API version 为 2023-07-01-preview）

    | Name      | Type   | Description                                                  |
    | :-------- | :----- | :----------------------------------------------------------- |
    | arguments | string | 调用函数所需的参数，由模型以 JSON 格式生成； <br>注意模型不一定能生成合法参数，调用前需要注意检查 |
    | name      | string | 需要调用的函数名                                             |

  + **FunctionDefinition**（注意需要 API version 为 2023-07-01-preview）

    | Name        | Type   | Description                                  |
    | :---------- | :----- | :------------------------------------------- |
    | description | string | 描述函数功能，是模型选择函数及生成参数的依据 |
    | name        | string | 函数名                                       |
    | parameters  |        | 函数的参数，以 JSON 格式提供                 |



#### 



## Python SDK

本质上是对 REST API 的调用做了包装，因此还是以参考 REST API 为主



## Add your data

mongodb



