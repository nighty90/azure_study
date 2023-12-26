# Azure OpenAI (AOAI)

## Notes

+ 价目：https://azure.microsoft.com/zh-cn/pricing/details/cognitive-services/openai-service/#pricing
+ 注意实际输入给 GPT 模型的 prompt 是包含一定量的上下文的
+ `max_token` 指示了向模型申请的计算资源的大小，因此是计费的依据



## Overview

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
  + 注意会有客户问 Azure OpenAI 和 OpenAI



### Key concepts

+ Prompts 与 completions
  + completions endpoint 是 Azure OpenAI API 的核心组件
  + 该 API 提供了模型的文本输入和文本输出的交互界面
+ Tokens：Azure OpenAI 将文本分割为 token 来处理
  + 可以是完整的单词，也可以仅是字符块（比如一个音节）
  + 处理的 token 总数取决于 [ 输入、输出、请求参数 ] 的长度
  + token 数量会影响响应延迟与通量
+ Resources：如同其他 Azure 产品，Azure OpenAI 需要创建 resource 来使用
+ Deployments：创建 resource 后，需要先将模型部署后，再开始调用 API
+ Prompt engineering：GPT 系列模型基于 prompt，故需要调整 prompt 以更好地使用
+ models：Azure OpenAI 提供了一系列模型的接口
  + 文本输入：GPT 系列（文本生成）、Embeddings（文本编码）、DALL-E（文生图）
  + 音频输入：Whisper（语音转录）



## How-to: Create and deploy an Azure OpenAI Service resource

以 Portal 为例

### Prerequisites

+ Azure subscription
+ Azure OpenAI 的 access（提交表格申请）
+ 创建 Azure OpenAI resource 以及部署模型的权限

### Steps

1. 创建 Azure OpenAI resource
   1. 登录 Portal，创建新的 resource，选择 Azure OpenAI
   2. 在创建页面中提供所需信息
      + subscription（注意需要已经申请到 Azure OpenAI 的 access）
      + resource group（可以新建 / 使用已有）
      + region（会影响延迟）
      + name（最好能清楚描述该 Azure OpenAI resource）
      + pricing tier（目前仅 Standard tier 可使用 Azure OpenAI）
   3. 进行网络配置：以下三选一
      + 允许所有网络：默认选项
      + 允许特定网络
        + 必须项：设置允许访问的 virtual network 和 subnets
        + 可选项：通过防火墙设置允许访问的 IP 范围
      + 禁止所有：设置 private endpoint 来访问 resource
   4. 确认 resource 设置并创建
2. 部署需要的模型
   1. 登录 Azure OpenAI Studio，选择要使用的 subscription 和 Azure OpenAI resource
   2. 选择 **management** > **deployments**
   3. 创建新的 deployment：要部署的模型、deployment 的名字、其他高级选项



## Quickstart: Get started generating text using Azure OpenAI Service

以 Python 为例



### Prerequisites

+ Azure subscription
+ Azure OpenAI 的 access（提交表格申请）
+ Python 3.7.1 及以上版本
+ Python库：`os`、`requests`、`json`（仅 `requests` 需要额外安装）
+ 部署了 `gpt-35-turbo-instruct` 的Azure OpenAI resource



### Steps

1. 获取 key 和 endpoint

   + ENDPOINT
     + 获取：查看 resource 的 **Keys & Endpoint**
       + 或者 **Azure OpenAI Studio** > **Playground** > **Code View**
     + 例子：https://docs-test-001.openai.azure.com/
   + API-KEY：同样在 **Keys & Endpoint**，KEY1 和 KEY2 任选其一
   + DEPLOYMENT-NAME：查看 **Resource Management** > **Deployments** 

2. 将 key 和 endpoint 保存至环境变量 `AZURE_OPENAI_KEY` 和 `AZURE_OPENAI_ENDPOINT`

   + 变量名没有严格规定，请尽量起一个能清楚描述内容的名字

3. 编写 Python 脚本

   1. 导入：os（获取环境变量）、openai（与模型交互）

      + 给的例子里另外导入了 requests 和 json，但暂时没有用上

   2. openai 库配置

      ```python
      openai.api_key = os.getenv("AZURE_OPENAI_KEY")
      openai.api_base = os.getenv("AZURE_OPENAI_ENDPOINT")
      openai.api_type = "azure"
      openai.api_version = "2023-07-01-preview"
      ```

   3. 发送 completion job

      + `openai.Completion.create(engine, prompt)`

        + 似乎不适用于 `gpt-35-turbo-16k`，应该可用于 `gpt-35-turbo-instruct`
        + engine：deployment name，指定所用的部署（模型）
        + prompt：用户输入

        + max_tokens：补全的最大 token 数，若该值小于模型给的输出则会发生截断
      + `openai.ChatCompletion.create(engine, message)`

        + engine：同上
        + message：发送给模型的输入，是 dict 构成的 list；注意该参数不叫 prompt
          + 例子：`[{"role": "user", "content": "Hi!"}]`

        + max_tokens：同上

   4. 处理 response

4. 清理 resource



## Add your data

mongodb

![image-20231226135804224](https://raw.githubusercontent.com/nighty90/ImgRepository/main/img/202312261358314.png)

