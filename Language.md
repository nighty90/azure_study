# Language





## QnA Maker

### Overview

+ 2025/03/31 起将被彻底弃用，新版将作为 Azure AI Language 的一部分
  + 现已无法创建新的 QnA Maker 资源
+ 云端 NLP 服务，主要用于根据问答的数据库来回答问题
+ 分层排名过程
  + 数据存储在 Azure search，由它给出第一层结果
  + top 结果由 QnA Maker 的 NLP 重排模型处理后给出最终结果及置信度（第二层）
+ 可以通过多轮对话的方式改进问答库





### Quickstart

已弃用，这里仅梳理流程

1. 创建资源；注意已无法创建 QnA Maker 资源，应改为 Language 资源

2. 创建知识库；也可在这里创建资源

   + 导入知识可用 URL，支持 pdf 和 docx

   + 多轮默认文本：当文档中提取不到答案，但有层级组织时的提示文本
   + 闲聊：选择 QnA 的闲聊风格

3. 添加问答对，注意回答是 markdown 格式的，且需要手动写入换行符

4. 保存并训练

5. 测试，Inspect 给出更多细节

6. 发布知识库，进入生产环境

7. 创建问答机器人



## Language Service - Question answering

### Notes

+ 关于收费
  + 文本记录：1,000 个字符的文本计数单元
  + Free tier 中，所有 Language 的功能每月共享 5,000 条文本记录
+ 规划相关资源
  + QA 同时需要 Language 资源和 Search 资源
  + 总体考虑：知识库的语言、应用区域、每个主题域中有多少文档
  + 定价层考虑
    + 吞吐量
      + QA 的管理和预测 API 限制为 10 记录/s
      + Search 方面可能需要多个副本来扩容
    + 知识库大小 / 数量
      + 单语言：可用知识库数量为总索引数 - 1
      + 多语言：可用知识库数量为总索引数 / 2
    + 知识库中的文档数量：QA 中没有限制，主要考虑 Search
+ 应用规划：机器人风格、对话流、协作创作、客户端集成、主动学习……
+ QnA -> QA
  + 在 QnA 的 [portal](https://www.qnamaker.ai/) 中开始迁移，填入信息
  + 由于 QA 的验证更严格，会有因字符问题迁移失败的知识库，可以一键替换非法字符
  + 注意若有同名知识库则会覆盖
  + 迁移只会复制知识库的测试实例，故迁移完成后需要再手动部署



### Quickstart

0. 创建 Language 资源，注意需要启用自定义问题回答
   + 注意启用问题回答会自动创建 Azure Cognitive Search 资源并连接
1. 创建项目
   + 可以设置未找到答案时的默认回答
2. 添加数据来源，可以来自 URL、文件、闲聊
   + 将自动从文档中分析出问题和相应的解答
   + 之后可以手动添加新的问答对
   + 注意如果 quota 不够的话会添加失败
3. 在**编辑知识库**页面可测试问答情况 / 编辑问答对
   + 短回应：在回复完整答案之前，先给出一个简短的回答
   + inspect：可查看问题可能对应的各个回答及其置信度，可以从中选择 / 编辑
     + 对于选中的回答，需要将该问题保存为它的可选问题
     + 之后再遇到相同 / 类似问题时，置信度会发生变化
     + 注意任何编辑之后需要手动保存
   + metadata：可以对问答对添加元数据来便于请求时过滤结果
     + 例如可以添加一个 topic 键，用于反映该问答对的话题
4. 部署项目
   + 部署后，项目内容将从 Azure Search 的 test 索引转移至 prod 索引
   + 部署后生成 endpoint，可对其发送 POST 请求来获取回答
     + 详细参考[文档](https://learn.microsoft.com/en-us/rest/api/language/question-answering/get-answers?view=rest-language-2023-04-01&tabs=HTTP)
   + 部署好的 QA 可以创建 bot
     + 注意会创建 4 个资源，全部创建完需要一定时间



### QA REST API 

#### Get Answers

+ POST 请求

  ```
  {Endpoint}/language/:query-knowledgebases
  ```

  + 必须参数：`projectName`、`deploymentName`、`api-version`（=2023-04-01）

+ Headers

  + `Ocp-Apim-Subscription-Key`

+ 请求体示例

  ```json
  {
    "question": "Where can I find the downloads?",
    "top": 3,
    "confidenceScoreThreshold": 0.2,
    "answerSpanRequest": {
      "enable": true,
      "confidenceScoreThreshold": 0.2,
      "topAnswersWithSpan": 1
    },
    "includeUnstructuredSources": true
  }
  ```

  + `question`：需要提问的问题，唯一的必须项
  + `top`：限制返回的答案个数
  + `confidenceScoreThreshold`：置信度阈值
  + `includeUnstructuredSources`：是否包含非结构化的源
  + `answerSpanRequest`：配置回答定位
    + `confidenceScoreThreshold`
    + `enable`：是否启用
    + `topAnswersWithSpan`：带定位信息的回答个数
  + 其他详细见[文档](https://learn.microsoft.com/zh-cn/rest/api/language/question-answering/get-answers?view=rest-language-2023-04-01&tabs=HTTP)





## Language Understanding (LUIS)

### Overview

+ 从 2025/10/01 起停用，建议迁移到 CLU
  + 从 2023/04/01 开始无法创建新的 LUIS 资源
+ 基于云服务的对话式 AI 服务，用于从用户对话中推测含义及提炼信息
+ 应用：聊天机器人、语音助理



### Concepts

+ 意向（intent）：用户表达的目的，如订票、招呼、确认天气等
  + 所有 app 都带有 None 意向，表示与当前域不相干的话题
  + 对于正面和反面的意向可以分别创建
  + 如果示例话语包含正则，则请启用模式匹配，使用正则实体
  + 各意向（分类）之间样本应当数量均衡，注意 None 应占约 10%
+ 实体（entity）：与意向相关的元素，是需要从对话中提取的信息
  + 列表实体
    + 枚举类型，区分大小写且精确匹配
    + 可用于处理同义词 / 近义词，以此规范输出
  + 正则实体
    + 使用正则表达式匹配提取，不区分大小写
    + 用于处理结构化或格式特定的信息
  + 预建实体
    + LUIS 已自带的实体，无需额外构筑，但不同区域可用的预建实体不同
    + 其行为是预先训练的，无法修改
  + Pattern.Any 实体
    + 长度可变的占位符，模板式语言，类似通配符的方式
    + 可以理解为包含实体的一串匹配用的模板
  + ML 实体
    + 以机器学习算法根据上下文提取实体，需要自行打标签定制
    + ML 实体可以层级结构化，即包含其他子实体
  + 作为特征的实体
    + 即，将别的实体提取的信息用作 ML 实体的特征
    + 注意 ML 实体还有其他类型的特征，而不是只能以实体为特征
+ 话语（utterance）：用户输入，可以认为是训练集
  + 尽量多收集意思相同的不同构造的句子
    + 考虑长度、标点、用词、时态、词序
  + 收集话语的考虑点
    + 用户输入不一定格式正确，可以考虑拼写检查，或训练能应对错误格式的模型
    + 尽量收集用户会有的表达，注意这和专家、从业者的表达很可能不同
    + 尽量多收集同义词
  + 每个意向至少需要 15 个示例话语
  + 迭代时，请考虑仅添加 15 个话语来训练、发布、测试
  + 话语规范化：考虑单词形式、音调、标点
+ 模式（pattern）
  + 模式能够提高话语相似时，对意向识别的准确性
  + 注意对实体的识别没有帮助
  + 若模式中的多个实体在上下文中相关，则将使用实体角色来提取上下文信息
+ 特征（feature）
  + 特征文本以 token 为单位
  + 特征类型
    + 短语列表：描述一个概念的词 / 短语表，用于 token 级别的大小写不敏感的匹配
      + 在需要 app 提高泛化能力，识别该概念的新物品时使用
      + 类似于一个领域内特有的词汇表
    + 模型（意向 / 实体）用作特征
  + 必须特征：如果确信该特征必须存在于用户输入，则可设为必须
  + 全局特征：将特征设置为所有模型（意向 / 实体）可见



### Quickstart

0. 登录门户，创建 LUIS 创作（authoring）资源
1. 创建新的 app：设置名称、文化（语言）、描述、预测资源
   + 注意之后文化无法再修改
2. 添加域
   + 预建域：已经设置好了意向、实体和语句
   + 自定义：需要自行增加意向、ML 实体、语句，并打标签
     + 实体调色盘：选中 `@` 图标 -> 选择实体 -> 划上对应的文本
     + 内联标签：划上文本 -> 选择实体
3. 确认意向和实体
4. 训练并测试 LUIS app
5. 此时已完成 app 的创作，需要为其创建预测（prediction）资源
   + portal -> manage -> azure resource -> add prediction resource
6. 发布到生产环境：portal -> publish -> production slot
7. 通过 REST API 使用



### LUIS Prediction REST API

#### 从发布的 slot 中获取预测结果

+ GET 请求

  ```
  https://{endpoint}/luis/prediction/v3.0/apps/{appId}/slots/{slotName}/predict?query={query}[&verbose][&log][&show-all-intents]
  ```

  + `verbose`：是否返回所有元数据
  + `log`：是否将 query 记录
  + `show-all-intents`：是否显示所有的意向及其置信度

+ headers：设置 `Ocp-Apim-Subscription-Key`



## conversational language understanding (CLU)

### Overview

+ 关于收费
  + 注意，请求中每 1,000 个字符计为 1 条文本记录
  + 推理
    + 免费层：每月 5,000 条文本记录免费
    + 标准层： $2 / 1,000 条文本记录
  + 训练
    + 免费层：标准训练免费，高级训练仅能训练 1 小时，endpoint 托管免费
    + 标准层：标准训练免费，高级训练 $3 / 1h，endpoint 托管免费



### Migrate from LUIS to CLU

+ CLU 的优势：准确性、多语言支持、便于与 QA 集成
+ CLU 与 LUIS 的主要区别
  + CLU 的单个实体由多个实体组件构成
    + 已学习（类似于 ML 实体）、列表、预建、正则
  + 不再支持 Pattern.Any 实体
  + 不再需要实体角色、规范化设置、正则和列表特征
  + 可用单一语言训练而支持多语言
  + 需要添加测试数据集
  + 两种训练模型：标准（仅英语） / 高级（其他语言或多语言）



### Quickstart

以 [Language Studio](https://language.cognitive.azure.com/) 的操作为例

0. 需要 Language 资源
1. 登录 Language Studio，创建 CLU 项目
   + 本教程中，可直接导入已配置好的 JSON 文件
2. 为项目添加意图、实体、话语
   + 本教程中已通过 JSON 直接导入，可以跳过
   + 可以手动添加测试集
3. 发起训练任务：设置模型名称、训练模式、数据分割方式
   + 若手动添加了测试集，这里可以选择手动分割方式
   + 训练配置有多个版本，可以在模型名称那里找到更改版本的按钮
   + 一个项目一次只能有一个训练作业
4. 部署模型
   + 注意部署有过期时间，通常是训练配置过期时间的 12 个月之后
5. 测试模型
   + 注意选择语言，如果自动检测语言会产生额外费用
   + 可能提示已过时，找不到自动检测的选项



### CLU prediction REST API

#### 发送预测请求

+ POST 请求

  ```
  {ENDPOINT}/language/:analyze-conversations?api-version={API-VERSION}
  ```

+ headers：设置 `Ocp-Apim-Subscription-Key`

+ 请求体格式

  ```json
  {
    "kind": "Conversation",
    "analysisInput": {
      "conversationItem": {
        "id": "1",
        "participantId": "1",
        "text": "Text 1"
      }
    },
    "parameters": {
      "projectName": "{PROJECT-NAME}",
      "deploymentName": "{DEPLOYMENT-NAME}",
      "stringIndexType": "TextElement_V8"
    }
  }
  ```

  + 重点为 `analysisInput.conversationItem.text`，这是实际需要分析的文本

