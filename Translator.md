# Translator



## Overview

+ 按字符数计费

  + 对请求体中需要 translator 翻译的正文进行计数
    + 对于 Translate / Transliterate / Dictionary Lookup 为 `Text`
    + 对于 Dictionary Examples 为 `Text` 和 `Translation`
  + 每个 Unicode 码位计为一个字符，注意有些语言的字符会占多个码位
  + 正文的所有字符都会计数，包括 XML 标记等
  + 每次翻译独立计数，一次同时译为多语言也算作一次翻译
  + 注意重复的翻译也会一样地计费

+ 服务限制

  + 翻译请求限制为 50,000 个字符，注意对各语言的翻译请求要加和计算
    + 例子：5,000 个字符翻译为 3 种语言，则共请求 15,000 个字符
    + 另外对请求体的矩阵中最大元素大小、矩阵元素个数也有限制，具体查看[文档](https://learn.microsoft.com/zh-cn/azure/ai-services/Translator/service-limits)
  + 存在每小时字符数限制，注意应平均使用，即应按每分钟可用字符数来计算
    + 例子：F0 层的限制为 2,000,000 / h，则实际应不超过 33,300 / m
  + 标准模型最大延迟为 15s，自定义模型为 120s
  + 文档翻译也存在文档大小、总数等的限制，具体查看[文档](https://learn.microsoft.com/zh-cn/azure/ai-services/Translator/service-limits)

+ Text Translation 阻止翻译：有时部分内容不需要被翻译，如品牌名等

  + HTML 输入

    + 可以将不需要翻译的元素的 `class` 设为 `notranslate`
    + 可以直接为元素添加 `translate="no"`

  + 可以使用动态字典来预先指定特定内容的翻译方式来阻止翻译

  + 可以干脆不传入不需要翻译的部分

  + 可以训练一个自定义翻译器



## Concepts

+ BLEU (Bilingual Evaluation Understudy)：针对机器翻译任务的评分
  + 参考文档：https://zhuanlan.zhihu.com/p/223048748
  + 基本思想：机器译文的 n-gram 命中参考译文的比例
  + 改进：考虑参考译文中的最大词频、对短译文进行惩罚



## V3 Translator Reference

### Base URL

可选择不同的地理位置

| Geography     | Base URL                                  | Datacenters                                                 |
| :------------ | :---------------------------------------- | :---------------------------------------------------------- |
| Global        | api.cognitive.microsofttranslator.com     | 就近选择                                                    |
| Asia Pacific  | api-apc.cognitive.microsofttranslator.com | Korea South, Japan East, Southeast Asia, and Australia East |
| Europe        | api-eur.cognitive.microsofttranslator.com | North Europe, West Europe                                   |
| United States | api-nam.cognitive.microsofttranslator.com | East US, South Central US, West Central US, and West US 2   |



### Authentication

在 headers 中设置

+ 注意地区设置：`Ocp-Apim-Subscription-Region`
  + 在使用 Azure AI multi-service / 区域性资源时，必须
  + 使用全球性的单服务资源时，可选
+ 使用 key：`Ocp-Apim-Subscription-Key`
  + 也可以在 URL 参数 `Subscription-Key` 中设置
  + 此时必须将地区设置到 `Subscription-Region` 参数
+ 使用 token：`Authorization`
  + 注意 Bearer 前缀
  + 获取 token
    + POST 请求
      + 全球：`https://api.cognitive.microsoft.com/sts/v1.0/issueToken`
      + 多服务 / 区域性：把 "api" 改为 "{region}.api"
    + headers 设置 key，payload 为空字符串
+ 使用 Microsoft Entra ID：请查看[文档](https://learn.microsoft.com/en-us/azure/ai-services/translator/reference/v3-0-reference)



### Text Translation REST API

#### Languages

获取支持的语言列表

+ GET 请求：`{base_url}/languages?api-version=3.0`
+ URL 参数
  + `api-version`：必须且必须为 3.0
  + `scope`：限定范围，可选多个，逗号分隔，有效值为 translation / transliteration / dictionary
    + 默认不限定范围，即 `scope=translation,transliteration,dictionary`
+ headers：无必须项
  + `Accept-Language`：语言列表以什么语言显示，默认为英文
  + `X-ClientTraceId`：用于标识本次请求的 GUID，由用户提供



#### Translate

+ POST 请求：`{base_url}/translate?api-version=3.0`

+ URL 参数

  + `api-version`：必须且必须为 3.0
  + `to`：必须，要翻译到哪种语言，可同时翻译为多种语言
    + 例子： `to=de&to=it` 翻译为德语和意大利语

  + `from`：输入文本的语言，未指定时将自动判断语言
    + 如果使用动态字典，则必须指定
  + `textType`：输入文本的格式，可选 plain / html
  + 其他不常用参数请查看[文档](https://learn.microsoft.com/en-us/azure/ai-services/translator/reference/v3-0-translate)

+ headers

  + 授权相关部分，必须，具体见 [authentication](#Authentication)
  + `Content-Type`：必须，payload 类型，必须为 application/json; charset=UTF-8
  + `Content-Length`：必须，请求体长度
  + `X-ClientTraceId`：可选，用户提供的 GUID
    + 也可以在 URL 的 `ClientTranceId` 参数中指定

+ 请求体

  ```json
  [
      {"Text": "I really like you."},
      {"Text": "Do you like me?"}
  ]
  ```

  + JSON 格式，内容为一个矩阵，其元素为仅含 "Text" 属性的对象



#### Detect

检测输入文本所用的语言

+ POST 请求：`{base_url}/detect?api-version=3.0`
+ URL 参数：仅 `api-version`
+ headers：同 translate
+ 请求体：同translate



## Document Translate

### Notes

+ 注意文档翻译的 endpoint 与文本翻译不同
+ 关于 prerequisites
  + 注意需要 storage account
  + 至少需要创建两个容器，并为它们创建 SAS
    1. 源容器 / blob：存放要翻译的文档，SAS 必须有 read 和 list 的权限
    2. 目标容器 / blob：存放翻译结果，SAS 必须有 write 和 list 的权限
    3. （可选）词汇表 blob
  + 必须使用单服务的 Translator 资源，且收费层至少为 S1
+ 支持的文档：pdf、csv、html、md、xlsx、pptx、docx、tsv、txt 等
+ Document Translation 阻止翻译需要使用术语表
  + 注意和 Text Translation 的方法不通用
  + 也可以用字典训练一个自定义模型来阻止翻译



### Document Translation REST API

#### Start translation

+ POST 请求：`{endpoint}/translator/text/batch/v1.1/batches`

+ headers：设置 `Ocp-Apim-Subscription-Key`

+ 请求体

  ```json
  {
      "inputs": [
          {
              "source": {
                  "sourceUrl": <sourceSASUrl>,
                  "storageSource": "AzureBlob",
                  "language": "en"
              },
              "targets": [
                  {
                      "targetUrl": <targetSASUrl>,
                      "storageSource": "AzureBlob",
                      "category": "general",
                      "language": "es"
                  }
              ]
          }
      ]
  }
  ```

  + 具体请参考[文档](https://learn.microsoft.com/en-us/azure/ai-services/translator/document-translation/reference/start-translation)





## Custom Translator

### Concepts

+ 并行文档 Parallel document：对应的两个语言的文档，是训练自定义模型用的数据集
  + 将一方标记为源文档，另一方标记为目标文档，即可训练从源语言到目标语言的翻译
  + 文档内请遵循一句一行的格式
  + 注意训练自定义模型需要至少 10,000 个对齐的句子
  + 注意文档最好都用 UTF-8 编码，否则容易出现乱码问题
+ 句子配对和对齐 Sentence pairing and alignment：句子将被配对以学习映射关系
  + 模型在训练时，会先对并行文档中的句子进行对齐，因此文档本身可以事先未配对好
  + 如果是已经对齐的文档，可以将后缀改为 .align 来指示模型跳过对齐的步骤
+ 字典 Dictionary：指定特定语句应如何翻译的表，应理解为词汇表或术语表
  + 可通过提供将语句映射为自身的字典来阻止翻译
  + 短语字典 Phrase dictionary 区分大小写，进行精准的查找替换
  + 句子字典 Sentence dictionary 不区分大小写，需要句子完整匹配
  + 仅字典的训练：以基线模型为基础，仅训练字典