# Vision



## Note

+ 如果 http 请求需要附带 JSON 格式的数据，则 headers 中需要设置 `Content-Type: application/json`
+ headers 中必须设置 `Ocp-Apim-Subscription-Key: <key>`
+ 区分各种 Read 和 OCR
  + 参考资料：https://stackoverflow.com/questions/70657936/microsoft-computer-vision-ocr-read-api-charged-as-s3-transaction-instead-of-s2
  + Vision v3.2 - OCR：从图像中识别印刷体文字
    + POST 请求：`{endpoint}/vision/v3.2/ocr`
    + 同步操作，会直接返回识别的结果
    + 基本已经弃用的 API，不建议使用
    + Pricing Tier 为 Vision 的 S2 Tier
  + Vision v3.2 - Read：从图像中识别印刷体或手写体的文字，能应对多页、多语言的文档
    + POST 请求：`{endpoint}/vision/v3.2/read/analyze`
    + GET 请求：`{endpoint}/vision/v3.2/read/analyzeResults/{operationId}`
    + 异步操作，发出 POST 请求调用模型后，需要再发送 GET 请求来获取分析结果
    + 是 tutorial 中调用的 API
    + Pricing Tier 为 Vision 的 S3 Tier
  + Vision v4.0 - Read：从图像中识别文字，更适用于海报、路牌等文字量不密集的图像
    + POST 请求
      + `{endpoint}/computervision/imageanalysis:analyze?api-version=2022-10-12-preview]`
      + 注意 4.0 的 Vision API 中，所有功能都在该 API 下，需要通过 `feature` 参数来指定具体使用的功能
  + Document Intelligence - Read：从图像和文档中识别文字，在图像方面更适合分析文件的照片等
  + 虽然文档中经常都管它们叫 OCR (Read)，但要注意其实是好几个不同的模型



## Vision - Overview - What is Azure AI Vision?

+ 包含的服务
  + OCR：从图像识别文字，相比 Document Intelligence 更适合识别非文档的图像
  + Image Analysis：从图像提取视觉特征，包括物体、人脸、成人内容等，并自动生成说明
  + Face：检测 / 识别 / 分析人脸
  + Spatial Analysis：从视频输入分析人物是否出现或移动
+ 图像要求
  + 格式：JPEG、PNG、GIF、BMP
  + 文件大小：< 4M
  + 尺寸：> 50 x 50（对于 OCR，还需要 < 10,000 x 10,000）



## Vision - OCR - Quickstart: Azure AI Vision v3.2 GA Read

以 REST API 为例

### Notes

+ 确定要使用的服务
  + 文档（PDF、扫描件等）建议使用 Document Intelligence 
  + 图像（海报、照片等）建议使用 OCR for images
+ 需要创建 Azure AI Vision 资源



### Read printed and handwritten text

1. 调用 read API 分析图像

   + POST 请求：`{endpoint}/vision/v3.2/read/analyze`

     + `model-version`：指定模型版本

   + request body

     ```json
     {"url": "<url_to_image_containing_text>"}
     ```

   + response 的 header 中，`Operation-Location` 记录了追踪用的 url

2. 获取 read 结果

   + GET 请求，访问追踪用的 url
   + 识别结果包括文字内容、位置、样式等



## Vision - Face - Quickstart: Use the Face service

以 REST API 为例



### Notes

+ 需要创建 Face 资源，注意该服务也是访问受限的，需要申请 access



### Identify and verify faces

1. 在源图像上调用 detect，得到需要识别的源人脸

   + POST 请求：`{endpoint}/face/v1.0/detect`

     + `returnFaceId`：是否返回人脸 id，默认为 true
     + 注意需要 access
     + `returnFaceLandmarks`：是否返回人脸特征点，默认为 false
     + `returnFaceAttributes`：需要提取的人脸属性，多属性之间逗号分隔
       + 可选项: headPose, galsses, occlusion, accessories, blur, exposure, noise, mask,  qualityForRecognition
       + 注意不是所有 detection 模型都支持这些属性
         + 例如 detection_03 仅支持 headpose、mask、qualityforrecognition
       + 注意分隔仅需要逗号，不要空格
     + `recognitionModel`：所使用的 recognition 模型
       + 可选项：recognition_01 到 04，默认为 01，推荐 04
     + `returnRecognitionModel`：是否返回所使用的模型，默认为 false
     + `detectionModel`：所使用的 detection 模型
       + 可选项：detection_01 到 03，默认为 01
     + `faceIdTimeToLive`：人脸 id 缓存的时间，单位为 s，范围为 60 - 86400（24h），默认最大

   + request body

     ```json
     {"url": "<url_to_image_containing_faces_to_be_recognized>"}
     ```

   + 返回的内容：人脸 id、所在区域、各人脸属性

   + 注意记录返回的人脸 id

2. 创建 LargePersonGroup，该对象用于存储多个人的聚合人脸数据

   + PUT 请求：`{endpoint}/face/v1.0/largepersongroups/{GroupId}`

     + `GroupId` 由用户设置，仅由小写字母、数字、下划线、hypen组成，最大长度为 64

   + request body

     ```json
     {
         "name": "<group_name>",
         "userData": "<user_provided_data_attached_to_the_group>",
         "recognitionModel": "<recognition_model>"
     }
     ```

     + 注意 `recognitionModel` 需要与检测源人脸时的 recognition 模型一致
       + 或者反过来说，检测原人脸时，应使用对应组所用的 recognition 模型

   + 注意记录自己设置的组 id

3. 为创建的组再创建人员对象

   + POST 请求：`{endpoint}/face/v1.0/largepersongroups/{GroupId}/persons`

     + 这里 `GroupId` 为上一步设置的组 id

   + request body

     ```json
     {
         "name": "<person_name>",
         "userData": "<user_provided_data_attached_to_the_person>"
     }
     ```

   + 注意记录创建的人员 id

4. 为已创建的人员关联人脸数据

   + POST 请求

     ```
     {endpoint}/face/v1.0/largepersongroups/{GroupId}/persons/{personId}/persistedfaces
     ```

     + `detectionModel`：设置所用的 detection 模型，默认为 detection_01

   + request body

     ```json
     {"url": "<url_to_face_image_of_the_person>"}
     ```

5. 使用关联好的人脸数据训练该 LargePersonGroup，令模型学会将人脸与人员对应

   + POST 请求：`{endpoint}/face/v1.0/largepersongroups/{GroupId}/train`

6. 使用训练好的组来识别源人脸

   + POST 请求：`{endpoint}/face/v1.0/identify`

   + request body

     ```json
     {
         "largePersonGroupId": "<INSERT_PERSONGROUP_ID>",
         "faceIds": [
                 "<INSERT_SOURCE_FACE_ID>"
         ],  
         "maxNumOfCandidatesReturned": 1,
         "confidenceThreshold": 0.5
     }
     ```

     + 注意这里的 `largePersonGroupId` 是组 id 而不是组名称

   + 返回的是该人脸 id 对应的人员 id

   + 注意如果检测人脸和创建组时用的 recognition model 不一致，会报错

     + 400，BadArgument，'recognitionModel' is incompatible.

7. 进行人脸验证

   + POST 请求：`{endpoint}/face/v1.0/verify`

   + request body

     ```json
     {
         "faceId": "<INSERT_SOURCE_FACE_ID>",
         "personId": "<INSERT_PERSON_ID>",
         "largePersonGroupId": "<INSERT_PERSONGROUP_ID>"
     }"
     ```

   + 判断人脸 id 与人员 id 是否能对应上，返回布尔值及其置信度



## Vision - Face

Face 的 base_url 为 `{endpoint}/face/v1.0`

### Find similar faces

1. 检测人脸，获取所有人脸的 id

   + POST请求：`{base_url}/detect`
     + 参数主要要设置 `detectionModel`
   + 要获取源人脸（query）和用于比较的人脸（database）

2. 查找相似人脸

   + POST请求：`{base_url}/findsimilars`

   + request body

     ```json
     {
         "faceId": "<source_face_id>",
         "faceIds": [<ids_of_faces_to_be_compared>],
         "maxNumOfCandidatesReturned": 10,
         "mode": "matchPerson"
     }
     ```

   + 返回匹配到的人脸 id



### Notes

+ detection 模型比较

  |            | **detection_01**                    | **detection_02**               | detection_03                                 |
  | ---------- | :---------------------------------- | :----------------------------- | :------------------------------------------- |
  | 历史       | 最早的模型，默认项                  | 2019/05 发布                   | 2021/02 发布                                 |
  | 针对性优化 | 基础                                | 小尺寸人脸、侧视人脸、模糊人脸 | 基础准确度、人脸较小（64x64 像素）、人脸转动 |
  | 可用属性   | 头部姿势、年龄、表情等 + 人脸特征点 | /                              | 面罩、头部姿势 + 人脸特征点                  |





## Vision - Image Analysis

### Overview

+ 4.0 版（预览）的适用区域：East US、France Central、Korea Central、North Europe、Southeast Asia、West Europe、West US
  + 注意不适用于 East Asia
+ 图像分析的可用功能
  + 自定义（仅4.0）
  + 读取文本（仅4.0）
  + 检测人物（仅4.0）：返回人物的边界框、置信度
  + 为图像生成文字描述
    + 4.0 版支持生成密集的描述文字，即为各个物体生成描述
  + 检测物体：返回物体的边界框、tag、置信度
  + 标记视觉特征：对图像打 tag
  + 智能裁剪
  + 检测品牌（仅3.2）
  + 图像分类（仅3.2）：按层次结构进行标识和分类
  + 检测人脸（仅3.2）：返回人脸的坐标、矩形、性别、年龄
  + 检测图像类型（仅3.2）：检测图像是否是素描、剪贴画等类型
  + 检测特定领域（仅3.2）：针对性检测名人、地标等
  + 检测配色（仅3.2）：检测黑白 / 彩色，对彩色可确定主色和调性色
  + 审查（仅3.2）：检测成人内容，返回各类别的置信度
  + 产品识别（仅4.0）：分析货架上商品的位置
  + 多模态 embedding（仅4.0）：对图像和文本的 query 进行编码，可根据文本 query 搜索图像
  + 背景移除（仅4.0）：可返回前景或其蒙版