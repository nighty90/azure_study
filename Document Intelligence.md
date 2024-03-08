# Document Intelligence

文档：[Document Intelligence documentation - Quickstarts, Tutorials, API Reference - Azure AI services | Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/?view=doc-intel-4.0.0)

示例数据：[azure-sdk-for-python/sdk/formrecognizer/azure-ai-formrecognizer/samples at azure-ai-formrecognizer_3.3.2 · Azure/azure-sdk-for-python · GitHub](https://github.com/Azure/azure-sdk-for-python/tree/azure-ai-formrecognizer_3.3.2/sdk/formrecognizer/azure-ai-formrecognizer/samples)



## Notes

+ 曾用名为 Azure Form Recognizer，以下简称为 FR
+ 有诸多版本，注意对照文档查看
+ 注意客户很可能仍使用 FR 来称呼该服务



## Overview

### Models

调用时，所有非自定义的模型 ID基本遵循 `prebuilt-<lower case model name>` 的格式，如 `prebuilt-layout`

+ 文档分析
  + General document：提取文本、表格、结构、键值对
    + 模型 ID 为 `prebuilt-document`
  + Layout：提取文本、表格、勾选框、结构信息
    + 输入数据为文档（PDF / TIFF）或图像（JPG / PNG / BMP）
  + Read：提取文本行、词语、两者的位置、所用的语言、手写风格
    + 输入数据为文档（PDF / TIFF）或图像（JPG / PNG / BMP）
+ 预建模型：Invoice、Receipt、Health insurance card、W-2、ID document、Business card
+ 自定义模型
  + 自定义提取模型：从表格或文档中提取信息，最少仅需标注 5 个 sample
  + 自定义分类模型：文档类型分类器，最少两类，每类各 5 个sample



## 自定义模型

### Notes

+ 训练集需要存储在 storage account 里

  + 注意允许匿名访问 container
  + 需要配置 CORS，允许 FR 的操作

  + 可以事先上传到 container 里，也可以创建 project 之后直接在 studio 里上传
    + 注意路径默认从 container 的子文件夹开始，不要以 "/" 开头，不需要路径已存在
    + 如果直接在 studio 里上传，会传到设置的 container 下的路径里
    + 如果已经上传，应该将路径设置到对应的文件夹里
  + 对于分类模型
    + 如果要创建分类模型，可以创建子目录来放置各类别的样本，studio 将自动据此打标签
    + 理论上，对于分类模型，还应该提供文件对应的布局信息（运行 Layout 获得）
    + 不过实际上对于没有布局信息的文件，studio 也会自动分析并生成



### 训练自定义提取模型

+ 总体步骤
  + 选择 Custom extraction model，创建新 project
    + 配置 subscription、resource 和 storage container
  + 对 sample 文档打标签，至少要 5 篇文档
    + 如果是与预建模型能处理的文档类型相近的，可以尝试 auto label 先自动打标签
    + 通常直接使用 Layout 自动检测布局并绘制区域，再手动打标签
    + 如果 Layout 检测不准确，可以再手动绘制区域，但注意需要先运行过 Layout
  + 点击 Train，设置模型名称和底模，训练模型，会自动跳转至 model 页面
    + 底模可选 neural / template，推荐使用 neural
  + 训练完成后可以切换到 test 页面查看模型效果
+ 关于打标签
  + 
  + 打标签为表格

    1. 手动在右侧边栏添加 Table 类型的标签，选择表格类型（Dynamic / Fixed），起名
       + 动态表格：行数不定，列固定，通常没有行名
       + 固定表格：行列固定，通常行名列名都有，即交叉表的形式
    2. 按需求增加行 / 列
    3. 选择文档中的文本填入每一格
  + 打标签为签名

    + 签名区域常为留白，检测不到，需要手动绘制区域捕捉
    + 注意 neural 模型不支持 Signature 类型的标签





## Python SDK

以 python 为例，注意需要安装 python 包 `azure-ai-formrecognizer`



### 基本使用

  ```python
  # 导入 DocumentAnalysisClient 类和 AzureKeyCredential 类
  from azure.ai.formrecognizer import DocumentAnalysisClient
  from azure.core.credentials import AzureKeyCredential
  
  # 创建 Client
  ENDPOINT = os.getenv("MY_AI_SERVICE_ENDPOINT")
  KEY = os.getenv("MY_AI_SERVICE_KEY")
  document_analysis_client = DocumentAnalysisClient(
      endpoint=ENDPOINT, 
      credential=AzureKeyCredential(KEY)
  )
  
  # 创建分析文档作业
  docUrl = "https://raw.githubusercontent.com/Azure-Samples/cognitive-services-REST-api-samples/master/curl/form-recognizer/sample-layout.pdf"
  poller = document_analysis_client.begin_analyze_document_from_url(
      "prebuilt-document", docUrl
  )
  
  # 获取结果
  result = poller.result()
  ```



### 分析结果为 `AnalyzeResult` 类

+ 常用属性（参考 `prebuilt-document` ）
  + `styles`
  + `key_value_pairs`：`key`、`value`
  + `pages`：`lines`、`words`、`select_marks`
  + `tables`：`cells`
  + 这些属性大多都有 `content` 和 `bounding_regions` 属性
    + `BoundingRegion`：`page_number`、`polygon`

+ `prebuilt-layout`
  + `DocumentLine` 有 `get_words()` 方法，返回该行对应的词的 list

+  `prebuilt-invoice`
  + 注意结果的处理主要集中在 `documents` 属性，是 `AnalyzedDocument` 的 list
    + 主要访问 `AnalyzedDocument` 的 `fields` 属性，是 `DocumentField` 的 dict
      + `DocumentField` 类有 `value` 和 `confidence` 属性



## Disaster recovery

为了应对故障，可以在多个不同区域部署同样的 Document Intelligence，为此需要复制操作

### Copy Steps

1. 向 target resource 发出复制授权请求，收到新模型（占位）的 url

   ```http
   POST https://{TARGET_FORM_RECOGNIZER_RESOURCE_ENDPOINT}/formrecognizer/documentModels:authorizeCopy?api-version=2023-07-31
   ```

   + header 中设置 `Ocp-Apim-Subscription-Key` 为 target source 的 key
   + request body 包含 `modelId` 和 `description`
   + 收到 200，保存 response 用于下一步

2. 将复制授权请求和新模型的 url 发给 source resource，收到追踪进度用的 url

   ```http
   POST {{source-endpoint}}formrecognizer/documentModels/{model-to-be-copied}:copyTo?api-version=2023-07-31
   ```

   + 同样，将 source resource 的 key 放入 header
   + 将上一步的 response 作为 request body
   + 收到 202，其中 Operation-Location 为进度 url，注意保存

3. 以 source resource 的 credentials 查询进度直到操作完成

   ```bash
   GET https://{source-resource}.cognitiveservices.azure.com/formrecognizer/operations/{operation-id}?api-version=2023-07-31
   ```

   + 注意这里的 key 是 source resource 的
   + 也可以通过 get model API 来查询目标模型，以此追踪状态