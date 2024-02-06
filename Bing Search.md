# Bing Search

### Bing search APIs

+ 搜索内容：Custom search、Entity search、Image search、News search、Video search、Visual search、Web search
+ 其他：Autosuggest、spell check



### Tutorial: Build a Custom Search web page

+ 自定义搜索
  + 设置 bing 搜索的范围，避免搜索到无关内容
  + 应用例：某个网站内嵌了一个 bing 搜索窗口，但希望用户仅能搜到该网站内的内容
+ 先决条件：需要有 Bing Custom Search 的资源
+ 创建门户：https://customsearch.ai/
+ Search Experience：核心部分，设置搜索范围
  + 网络切片概念
    + domain：域
      + 例子：www.microsoft.com
      + bing 将搜索其下所有内容
      + 省略 `www.` 时也会搜索子域
        + 如 `microsoft.com` 也会搜索 `support.microsoft.com`
    + subpage：域下的路径，不超过两个子目录
      + 例子：www.microsoft.com/en-us/windows/
    + webpage：特定网页
      + 例子：https://www.microsoft.com/en-us/p/surface-earbuds/8r9cpq146064?activetab=pivot%3aoverviewtab
      + bing 将仅搜索该网页本身，可以指定是否要包含子页面
  + 范围设置
    + Active：要搜索的切片
      + 可以设置 rank 为 super boost / boost / demote 来控制结果排名
    + Blocked：不搜索的切片
    + Pinned：如果用户输入的搜索内容匹配到 query，则需要置顶的切片
      + 匹配方式：精确、以 query 开头、以 query 结尾、包含 query
      + 不适用于图像和视频搜索
+ Autosuggest：设置搜索建议
  + 注意仅适用于 en-US market
  + 改动可能会需要 24 小时来生效
  + 启用搜索建议
    + 自动 bing 搜索建议：enble Automatic Bing Suggestions 即可
    + 用户自定义搜索建议：在下方添加建议，任何语言皆可
    + 注意要在 Hosted UI 中开启 Autosuggest
+ Hosted UI：设置托管的搜索 UI
  + 可以选择布局、配色、功能选项
  + 此处需要输入 key
    + 注意必须是 Bing Custom Search 的 key
      + 其他服务，比如 Bing Search 或者 Cognitive Service 的 key 是不行的
      + 自动填充可能会选定到 Cognitive Service 的 key，请手动改掉
    + 关于 key 的类型
      + 在 2020/10/30 之前发行的的 key 设置为 Cognitive Service 类型
      + 之后的需要设置为 Bing Search Service 类型
    + 注意如果 key 输入框的下拉框里没找到有效的 key，可以手动输入
+ 使用实例
  + 需要先将实例发布至生产端点
  + 可使用 javascript snippet 或直接内嵌网页



### Web Search REST API

+ 发送 GET 请求
+ 注意 url 总长最大为 2,048，为防止超过长度，query 参数最长为 1,500
+ endpoint：https://api.bing.microsoft.com/v7.0/search
+ request headers
  + 必须项：`Ocp-Apim-Subscription-Key`，key
  + 建议项
    + `User-Agent`：使用的设备（手机 / 电脑），有针对性的响应
    + `X-MSEdge-ClientID`：客户 id，提供连续性
    + `X-MSEdge-ClientIP`：客户端 ip 地址，提供位置信息
    + `X-Search-Location`：描述客户端物理位置，提供位置信息
  + 其他项
    + `Accept`：要求返回的格式，可选 application/json 或 application/ld+json
    + `Accept-Language`：设置语言，可以用逗号分隔多个语言
      + 不要与 setLang 参数同用
      + 使用时需要设置 cc 参数，以此来确定 market
      + 请仅在需要设置多个语言时使用，单个语言请设置 mkt 和 setLang 参数
    + `Pragma`：设为 no-cache 时能阻止 bing 返回缓存的内容
+ 查询参数
  + 必须项：`q`，搜索的 query
  + 设置语言与国家
    + `cc`：2 字符的国家代码，指定搜索结果的来源国家
      + 必须与 headers 的 `Accept-Language` 同用
      + 仅当需要设置多个语言时使用
    + `mkt`：指定 market，格式为 "<语言>-<国家>"，如 "en-US"
    + `setLang`：指定语言，可以使用 2 字符 / 4 字符的语言码
      + 有些语言码不是 2 / 4 字符的，具体见 [Bing supported languages](https://learn.microsoft.com/en-us/bing/search-apis/bing-web-search/reference/market-codes#bing-supported-language-codes)
      + 常用：zh-hans（简体中文）、en（英语）、en-gb（英式英语）
  + 限制搜索结果的个数和位置
    + `count`：设置返回的网页搜索结果的个数，默认为 10
      + 与 `offset` 一并实现了分页效果
      + 注意仅限制网页搜索的结果，对其他搜索（图像、视频等）没有限制
      + 注意有时候即使总结果数量超过 `count`，返回的结果也可能不满 `count` 个
    + `offset`：结果的偏移量，即跳过 `offset` 个结果再开始返回，默认为 0
      + 实际表现不一定会跳过 `offset` 个，可能更多，不清楚会不会更少
      + 仅为 0 时会包含其他搜索种类的结果，否则仅包含网页搜索结果
  + 过滤搜索结果
    + `freshness`：根据时间过滤网页
      + Day（24小时内）、Week（7天内）、Month（30天内）
      + 时间段：`YYYY-MM-DD..YYYY-MM-DD`，如 `2019-02-01..2019-05-30`
      + 具体日期：`YYYY-MM-DD`
    + `safeSearch`：过滤成人内容，默认为 Moderate
      + 可选项：Off、Moderate（仅允许成人文本）、Strict（全部禁止）
  + 搜索类型限制
    + `answerCount`：限制搜索的种类个数
      + 例如：当结果包括网页、图像、视频、相关搜索时，设置为 2 将只返回网页和图像
      + 注意这里是考虑排名的
    + `promote`：需要 promote 的搜索种类，该种类不受 `answerCount` 的限制
      + 可选项：Computation、Entities、Images、News、RelatedSearches、SpellSuggestions、TimeZone、Videos、Webpages
      + 多个种类可逗号分隔
      + 请与 `answerCount` 同用
    + `responseFilter`：需要包含的搜索种类
      + 可选项同 `promote`
      + 如果需要排除某个种类，请在前面加负号
  + 文本设置
    + `textDecorations`：是否显示装饰标记，默认为 false
      + 装饰标记的例子：将搜索结果中匹配到的 query 用高亮符号包裹
    + `textFormat`：文本装饰所用的标记类型，默认为 Raw
      + 可选项：Raw（unicode字符）、HTML