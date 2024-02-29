# Speech



## Speech to text (STT)

### STT REST API for short audio

#### Notes

+ 注意这里是专门给短音频用的 STT REST API，用处有限，请尽量使用 SDKs
  + 直接传输音频的请求最多只能包含 60 秒的音频
  + 仅能处理 WAV 和 OGG 格式的音频
  + 仅返回最终结果，不提供部分结果
  + 不支持语音翻译、批量转录、自定义STT
+ 另有用于 batch transcription 和 custom speech 的 STT REST API



#### 短音频 STT

```http
POST https://<REGION>.stt.speech.microsoft.com/speech/recognition/conversation/cognitiveservices/v1
```

##### url 参数

+ `language`：必须，需要识别的语言
  `format`：指定结果格式，可选 simple / detailed，默认simple
  + simple 内容：RecognitionStatus、DisplayText、Offset 和 Duration
  + detailed 内容：包括显示文本的四种不同的表示形式
+ `profanity`：不雅内容的处理方式，可选 masked / removed / raw，默认为 masked
+ `cid`：自定义模型的 id



##### Headers

+ `Ocp-Apim-Subscription-Key`：必须，key，与 `Authorization` 二选一即可

+ `Authorization`：必须，带上 "Bearer" 前缀的 token，与 `Ocp-Apim-Subscription-Key` 二选一即可

+ `Content-type`：必须，提供的音频格式及编解码器；SDK 支持的格式更多
  + WAV 格式：audio/wav; codecs=audio/pcm; samplerate=16000
  + OGG 格式：audio/ogg; codecs=opus
  
+ `Pronunciation-Assessment`：指定如何评估发音，是 Base64 编码的 JSON 格式文本
  + `ReferenceText`：必须，参考文本
  
  + `GradingSystem`：分数规格，可选 FivePoint / HundredMark
  
  + `Granularity`：评估粒度，可选 Phoneme / Word / FullText
  
    + Phoneme 给出全文、单词、音素的得分，而 FullText 只给出全文得分
  
  + `Dimension`：输出内容，可选 Basic / Comprehensive
  
  + `EnableMiscue`：启用误读计算，布尔值，默认为 False
  
  + `ScenarioId`：GUID，表示自定义分数系统
  
+ `Transfer-Encoding`：指定要发送的数据分块，仅当要分块时使用

  + 必须与 `Expect` 同用
  + 建议使用分块传输，能减小延迟

+ `Expect`：分块传输时需启用，SS 将确认初始请求并等待附加的数据

  + 值固定，为 Expect: 100-continue

+ `Accept`：SS 返回的结果格式，必须是 application/json

  + 为避免某些框架默认的该项参数不正确，尽量手动设置一下



#### 申请 token

每个 token 的有效期为 10min，考虑到网络延迟等，建议同一 token 仅使用 9min

```http
POST https://<REGION>.api.cognitive.microsoft.com/sts/v1.0/issueToken
```

##### headers

+ `Ocp-Apim-Subscription-Key`：<YOUR_SUBSCRIPTION_KEY>
+ `Content-type`：application/x-www-form-urlencoded
+ `Content-Length`：0





### STT Python SDK

#### Notes

+ offset 与 duration：识别时，语句的开始点与时长

  + 以 10^-7^ s = 100 纳秒为单位
  + 若需要获得单词粒度的时间，可以调用 `SpeechConfig.request_word_level_timestamps()`

+ phrase list：事先为识别器提供短语列表，可以使识别更准确

  ```python
  phrase_list_grammar = speechsdk.PhraseListGrammar.from_recognizer(recognizer)
  phrase_list_grammar.addPhrase("Contoso")
  phrase_list_grammar.addPhrase("Jessie")
  phrase_list_grammar.addPhrase("Rehaan")
  ```

  + 通常添加名字、专有名词、缩写称呼等

+ 自动识别语言

  ```python
  auto_detect_source_language_config = speechsdk.languageconfig.AutoDetectSourceLanguageConfig(
      	languages=["en-US", "de-DE"]
  )
  speech_recognizer = speechsdk.SpeechRecognizer(
          speech_config=speech_config, 
          auto_detect_source_language_config=auto_detect_source_language_config, 
          audio_config=audio_config
  )
  result = speech_recognizer.recognize_once()
  auto_detect_source_language_result = speechsdk.AutoDetectSourceLanguageResult(result)
  detected_language = auto_detect_source_language_result.language
  ```

+ 启用日志

  ```python
  speech_config.set_property(speechsdk.PropertyId.Speech_LogFilename, "LogfilePathAndName")
  ```

+ 指定语言

  + 可以在 `SpeechConfig` 中设置 `speech_recognition_language`，然后传入 `SpeechRecognizer` 的 `speech_config`
  + 可以创建 `languageconfig.SourceLanguageConfig`，然后传入 `SpeechRecognizer` 的 `source_language_config`
  + 可以直接在 `SpeechRecognizer` 中设置 `language`
  + 对于英语，似乎被指定成别的语言也依旧可以识别？

+ 文件输入的限制

  + 只支持 wav 格式？
  + 目前测下来大概不支持奇怪的声道？比如 8 声道文件




#### Basic Usage

1. 设置 speech

   ```python
   speech_config = speechsdk.SpeechConfig(subscription=key, region=region)
   speech_config.speech_recognition_language="en-US"
   ```

2. 音频设置

   ```python
   audio_config = speechsdk.audio.AudioConfig(use_default_microphone=True)
   ```

   + 这里设置了使用麦克风
   + 如果是文件输入，可以 `AudioConfig(filename="your_file_name.wav")`

3. 创建识别器

   ```python
   speech_recognizer = speechsdk.SpeechRecognizer(
       speech_config=speech_config, audio_config=audio_config
   )
   ```

4. 调用方法识别语音

   ```python
   result = speech_recognizer.recognize_once_async().get()
   ```

5. 处理识别结果

   ```python
   # 成功识别
   if result.reason == speechsdk.ResultReason.RecognizedSpeech:
       print(f"Recognized: {result.text}")
   # 有语音输入，但识别不出
   elif result.reason == speechsdk.ResultReason.NoMatch:
       print(f"No speech could be recognized: {result.no_match_details}")
   # 操作被取消
   elif result.reason == speechsdk.ResultReason.Canceled:
       details = result.cancellation_details
       print(f"Speech Recognition canceled: {details.reason}")
       if details.reason == speechsdk.CancellationReason.Error:
           print(f"Error details: {details.error_details}")
           print("Did you set the speech resource key and region values?")
   ```

   

#### Use continuous recognition

1. 设置全局变量用于之后退出循环：`done = False`

2. 构造回调函数用以停止识别

   ```python
   def stop_cb(evt):
       print('CLOSING on {}'.format(evt))
       speech_recognizer.stop_continuous_recognition()
       nonlocal done
       done = True
   ```

   + 必须调用 `stop_continuous_recognition()`
   + 必须改变全局变量来退出循环

3. 将事件信号（识别器的多个属性）与回调函数相连接

   + 这里作为示例，多个信号的回调函数为打印收到的信号
   + 注意同一个信号可以连接多个回调函数

4. 开始识别

   ```python
   speech_recognizer.start_continuous_recognition()
   while not done:
       time.sleep(0.5)
   ```

   





## Text To Speech (TTS)

### Notes

#### 关于人声

+ 旧版人声（非神经人声）正在逐渐停用，请迁移到神经人声（neural voice）
  + 自定义人声 custom voice：从 2021/03/01 到 2024/02/29 逐渐停用
  + 内置人声 prebuilt voice： 从 2021/09/01 到 2024/08/31 逐渐停用
+ [可用人声列表](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/language-support?tabs=tts)及[浏览](https://speech.microsoft.com/portal/voicegallery)，注意各区域可用的人声不同
+ 所有人声都为双语，可以说自己的语言和英语（带口音）
+ AOAI 也出了 TTS 声音
  + 可以通过 Speech 的 SDK 使用
    + 推荐，只需要改一下使用的声音名就行
  + 也可以通过 AOAI 的 REST API 调用
    + 注意请求体里的 model 属性与 deployment 不是一个东西
    + 也许需要注意 api_version？目前测试 2024-02-15-preview 可用

+ 人声有在更新，经测试 3.0 版本与 2.0 版本合成结果波形不同



#### SSML 格式

Simple example

```xml
<speak version="1.0" xmlns="http://www.w3.org/2001/10/synthesis" xml:lang="en-US">
    <voice name="en-US-JennyNeural">
        Welcome <break time="750ms" /> to text to speech.
    </voice>
</speak>
```



Basic structure

```xml
<speak version="1.0" xmlns="http://www.w3.org/2001/10/synthesis" xmlns:mstts="https://www.w3.org/2001/mstts" xml:lang="string">
    <mstts:backgroundaudio src="string" volume="string" fadein="string" fadeout="string"/>
    <voice name="string" effect="string">
        <audio src="string"></audio>
        <bookmark mark="string"/>
        <break strength="string" time="string" />
        <emphasis level="value"></emphasis>
        <lang xml:lang="string"></lang>
        <lexicon uri="string"/>
        <math xmlns="http://www.w3.org/1998/Math/MathML"></math>
        <mstts:audioduration value="string"/>
        <mstts:express-as style="string" styledegree="value" role="string"></mstts:express-as>
        <mstts:silence type="string" value="string"/>
        <mstts:viseme type="string"/>
        <p></p>
        <phoneme alphabet="string" ph="string"></phoneme>
        <prosody pitch="value" contour="value" range="value" rate="value" volume="value"></prosody>
        <s></s>
        <say-as interpret-as="string" format="string" detail="string"></say-as>
        <sub alias="string"></sub>
    </voice>
</speak>
```



##### Notes

+ 基本上是一种 XML

+ 关于收费

  + 注意语音合成是按字符数收费的（包括标点）
  + 不会对 SSML 本身直接收费，而是计算其中的收费字符
    + 实际应用 TTS 的文本
    + 除 `speak` 和 `voice` 以外的所有元素，注意构成元素的所有字符都计费
    + 字母、标点、空格、制表符、标记、空白字符
    + Unicode 定义的所有码位

+ Speech 的 SSML 是基于 W3C SSML 1.0 版本的，但支持的元素有所不同

+ 作为一种 XML，需要注意特殊字符的转义

  + 常见：`&` -> `&amp;`、`<` -> `&lt;`、`>` -> `&gt;`
  + 其他详细见 [Extensible Markup Language (XML) 1.0: Appendix D](https://www.w3.org/TR/xml/#sec-entexpand)



##### 常用元素及其属性

+ 划分结构
  + `speak`：根元素，至少包含一个 `voice` 元素
    + `version`：SSML 版本，当前为 "1.0"
    + `xml:lang`：文档语言，注意不是语音的语言
    + `xmlns`：一个 URI，指向定义 SSML 标记词汇的文档
  + `break`：添加暂停，无包含，可用于词之间
    + `strength`：暂停的相对时间
      + 可选 x-weak / week / medium（默认） / strong / x-strong
    + `time`：暂停的绝对时间，优先级高于 `strength`，单位可以是 s 或 ms
  + `mstts:silence`：添加静音，只能用于句子之间
    + 应用于当前 `voice` 的全部文本
    + `type`：指明静音方式和位置，可选项很多，具体见文档
    + `value`：静音时长，单位为 s 或 ms，范围为 0 - 5s
  + `p` 与 `s`：指明段落和句子，缺少时 SS 将自动推断
    + 主要用于提升 SSML 的可读性？
  + `bookmark`：标记特定位置，无包含，合成时产生 `BookmarkReached` 事件
+ 控制语音
  + `voice`：用于 TTS 的文本
    + `name`：必须，使用的人声名称
    + `effect`：音效，可选 eq_car / eq_telecomhp8k
  + `mstts:express-as`：控制人声风格和角色
    + `style`：人声风格，通常为情绪或身份
    + `styledegree`：风格强度，范围为 0.01 - 2，默认为 1，最小单位为 0.01
    + `role`：角色，用于使人声模拟不同年龄、性别的声调
  + `lang`：转变语言
    + 注意多数神经人声不支持在句子 / 单词水平设置语言
    + 注意 `speak` 中的 `xml:lang` 设置的是默认语言
    + `xml:lang`：需要使用的语言
  + `prosody`：韵律，调整音高、语调、范围、速率和音量
    + 设置较为复杂，建议参考文档
  + `emphasis`：设置强调
    + 仅适用于 `en-US-GuyNeural`、`en-US-DavisNeural`、`en-US-JaneNeural`
    + `level`：强度，可选 reduced / none / moderate（默认） / strong
  + `audio`：预录制的音频
    + `src`：音频文件的 URI，可以放在同 Azure 区域的 Blob 中以减少延迟
  + `mstts:audioduration`：设置输出的合成音频的持续时间
    + `value`：持续时间，以 s / ms 为单位，范围为原音频的一半到两倍长
  + `mstts:backgroundaudio`：设置背景音，每个 SSML 文档仅允许存在一个
    + `src`：URI
    + `volume`：音量，范围为 0 - 100，默认为 1
    + `fadein` / `fadeout`：淡入淡出，单位为 ms，范围为 0 - 10,000
+ 控制音节发音：较不常用，请参考文档





### TTS REST API

https://learn.microsoft.com/en-us/azure/ai-services/speech-service/rest-text-to-speech?tabs=streaming#convert-text-to-speech

```http
POST https://{REGION}.tts.speech.microsoft.com/cognitiveservices/v1
```

#### headers

注意以下 4 项全为必须项

+ `Ocp-Apim-Subscription-Key`：key
+ `Content-Type`：application/ssml+xml
+ `X-Microsoft-OutputFormat`：输出格式
  + 例子：audio-16khz-128kbitrate-mono-mp3
+ `User-Agent`：发出请求的应用名，最长255字符，由用户设置

+ 请求体

  + 自定义神经人声：纯文本格式，ASCII / UTF-8
  + 其他：SSML格式



### TTS Python SDK

https://learn.microsoft.com/en-us/python/api/azure-cognitiveservices-speech/?view=azure-python

#### Example

```python
# 导包
import os
import azure.cognitiveservices.speech as speechsdk

# speech 设置
# 设置 key 和 region
speech_config = speechsdk.SpeechConfig(
    subscription=os.environ.get('SPEECH_KEY'), 
    region=os.environ.get('SPEECH_REGION')
)
# 设置人声
speech_config.speech_synthesis_voice_name = 'en-US-JennyNeural'

# audio 设置
audio_config = speechsdk.audio.AudioOutputConfig(
    use_default_speaker=True  # 直接使用扬声器
)

# 创建合成器
synthesizer = speechsdk.SpeechSynthesizer(
    speech_config=speech_config, 
    audio_config=audio_config
)

# 调用
print("Enter some text that you want to speak >")
text = input()
result = synthesizer.speak_text_async(text).get()

# 结果处理
if result.reason == speechsdk.ResultReason.SynthesizingAudioCompleted:
    print("Speech synthesized for text [{}]".format(text))
elif result.reason == speechsdk.ResultReason.Canceled:
    cancellation_details = result.cancellation_details
    print("Speech synthesis canceled: {}".format(cancellation_details.reason))
    if cancellation_details.reason == speechsdk.CancellationReason.Error:
        if cancellation_details.error_details:
            print("Error details: {}".format(cancellation_details.error_details))
            print("Did you set the speech resource key and region values?")
```



#### Notes

+ `SpeechConfig` 类的人声设置
  + 设置 `speech_synthesis_voice_name` / `speech_synthesis_language` 属性
  + 优先级
    1. SSML 中指定的人声
    2. `speech_synthesis_voice_name` 指定的人声
    3. `speech_synthesis_language`，此时使用该语言的默认人声
    4. en-US 的默认人声
+ `SpeechConfig` 类的音频格式设置
  + 调用 `set_speech_synthesis_output_format()`
  + 该函数的输入为 `SpeechSynthesisOutputFormat` 枚举类的属性
  + 查看[支持的格式](https://learn.microsoft.com/en-us/python/api/azure-cognitiveservices-speech/azure.cognitiveservices.speech.speechsynthesisoutputformat)，注意 "Raw" 前缀的格式不包含音频的 headers
+ 注意音频设置使用的是 `audio.AudioOutputConfig` 类
  + 注意不是 STT 中用的 `audio.AudioConfig` 类
  + 关键字参数：`use_default_speaker`、`filename`、`stream`、`device_name`
  + 关于数据流输出（储存在内存中）
    + `SpeechSynthesizer` 对象的 `audio_config` 参数应设为 None
    + 此时合成结果为数据流，可以用 `AudioDataStream` 类来辅助处理
+ `SpeechSynthesizer` 类的方法
  + 注意点
    + 以下方法都有 `<function>_async()` 的异步版本
    + 注意异步版本返回的是 `ResultFuture` 对象
  
  + `get_voices_async()`：获取可用人声
  + `speak_ssml()` / `speak_text()`：以 SSML / 纯文本为输入合成
  + `start_speaking_ssml()` / `start_speaking_text()`：连续合成
  + `stop_speaking()`：结束连续合成
  
+ `SpeechSynthesizer` 类的事件信号使用 `connect()` 方法连接回调函数
+ `ResultFuture` 类使用 `get()` 方法等待操作完成并返回结果



### Batch synthesis REST API (Preview) for text to speech

以下 base_url 为 `https://{region}.customvoice.api.speech.microsoft.com/api`

#### Workflow

![Diagram of the Batch Synthesis API workflow.](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/media/long-audio-api/long-audio-api-workflow.png)



#### Notes

+ API 的 url

  ```
  https://{region}.customvoice.api.speech.microsoft.com/api/texttospeech/3.1-preview1/batchsynthesis
  ```

+ 创建批量合成：POST 请求

  + 请求体示例

    ```json
    {
        "displayName": "batch synthesis sample",
        "description": "my ssml test",
        "textType": "SSML",
        "inputs": [
            {
                "text": "<speak version='\''1.0'\'' xml:lang='\''en-US'\''>
    				<voice name='\''en-US-JennyNeural'\''>
    					The rainbow has seven colors.
    				</voice>
    			</speak>",
            },
        ],
        "properties": {
            "outputFormat": "riff-24khz-16bit-mono-pcm",
            "wordBoundaryEnabled": false,
            "sentenceBoundaryEnabled": false,
            "concatenateResult": false,
            "decompressOutputFiles": false
        },
    }
    ```

  + 注意设置 `textType` 为 PlainText 时，需要设置 `synthesisConfig` 的 `voice` 属性

+ 获取合成结果：对合成的 id 发送 GET 请求

  + `status` 表明当前状态（NotStarted / Running / Succeeded / Failed）
  + `outputs.result` 储存了合成结果的 url，对其发起 GET 请求来获取
    + 注意 headers 中要设置 key

+ 列出批量合成：GET 请求

  + url 参数包含 `skip`（跳过结果数） 和 `top`（返回结果个数）

+ 删除批量合成：对合成的 id 发送 DELETE 请求





## Speech Translation

### Notes

+ 本质是 STT + translator，很久以前的曾经是单独的功能



## Speech Python SDK

### Notes

+ 似乎实质上是在调用 C 语言的底层代码？
+ 有很多异步操作，但并不是以 python 的方式实现的，因此会需要自行实现类似 await 的机制



### 设置

#### SpeechConfig

```python
SpeechConfig(
    subscription: str | None = None, 
    region: str | None = None, 
    endpoint: str | None = None, 
    host: str | None = None, 
    auth_token: str | None = None, 
    speech_recognition_language: str | None = None
)
```

+ 主要用于设置认证信息，也可以对具体服务进行一些设置
+ 对象构造
  + 可以有 4 种初始化方式
    1. 从 subscription：使用 `subscription` 和 `region`
    2. 从 endpoint：使用 `endpoint`，此时 `subscription` 和 `auth_token` 可选
       + 通常用于使用训练好的自定义模型，在这里给定模型的 endpoint
    3. 从 host：使用 `host`，此时 `subscription` 和 `auth_token` 可选
    4. 从 token：使用 `auth_token` 和 `region`
  + 可以在这里设置 `speech_recognition_language` 来指定识别器的语言
+ 常用属性
  + speech_recognition_language：指定 STT 时的语言
  + speech_synthesis_voice_name：指定 TTS 时使用的声音
  + endpoint_id：指定要使用的 CNV 模型



#### audio.AudioConfig

```python
AudioConfig(
    use_default_microphone: bool = False, 
    filename: str = None, 
    stream: AudioInputStream = None, 
    device_name: str = None
)
```

+ 音频输入设置，用于 STT 和 Speech Translation
+ 对象构造
  + `use_default_microphone`：是否使用默认系统麦克风进行音频输入
  + `device_name`：指定要使用的音频设备的 ID
  + `filename`：指定音频输入文件
    + 文件不能太大？以及仅支持 wav 和 pcm？
  + `stream`：指定使用的输入流，为 `AudioInputStream` 对象
    + 原生只支持构建 pcm，其他格式需要依赖 gstreamer 软件来构建，有点难用



### 调用服务

#### SpeechRecognizer

```python
SpeechRecognizer(
    speech_config: SpeechConfig, 
    audio_config: AudioConfig = None, 
    language: str = None, 
    source_language_config: SourceLanguageConfig = None, 
    auto_detect_source_language_config: AutoDetectSourceLanguageConfig = None
)
```

+ 识别器，用于 STT
+ 对象构造
  + 仅 `speech_config` 为必须参数
  + 语言相关的参数请三选一
    + `language`：手动指定语言，仅指定语言
    + `source_language_config`：手动指定语言，但同时可以指定 endpoint
    + `auto_detect_source_language_config`：自动判断

+ 常用方法
  + 注意点
    + 以下方法都有 `<function_name>_async()` 的异步版本，返回 `ResultFuture` 对象
    + 用 `start_<suffix>()` 开始识别后，用对应的 `stop_<suffix>()` 来停止识别
    + 识别结果为 `SpeechRecognitionResult` 对象
  + `recognize_once()`：仅识别一次，根据静音 / 超时（15s）来判断结束
  + `start_continuous_recognition()`：开始连续识别
    + 产生的识别信号由对应的 EventSignal 对象属性接收
    + 需要用 `connect()` 方法为识别器的 EventSignal 对象属性设置回调函数，以处理识别结果
  + `stop_continuous_recognition()`：停止连续识别
  + `start_keyword_recognition(model)`：调用 keyword 模型识别，此时识别器在传入 keyword 后开始识别
+ 关于连续识别
  + 需要为 SpeechRecognizer 的各种 EventSignal 类的属性设置回调函数来处理事件信号
    + session_started, session_stopped
    + speech_start_detected, speech_end_detected,
      + 很奇怪地，start 后马上就会接收到 end
    + recognizing, recognized
      + 基本上识别完一句后，会产生一个 recognized 信号
    + canceled
  + 调用 `stop_continuous_recognition()` 后的表现
    1. 将当前识别的句子识别完，产生 recognized 信号
    2. 关闭当前会话，产生 session_stopped



### 事件信号

#### EventSignal

+ 接收并处理事件信号，通常作为 SpeechRecognizer 的属性存在
+ 常用方法
  + `connect()`：将事件信号与回调函数连接
    + 注意同一个信号可以连接多个回调函数，调用的顺序与连接时的顺序一致



#### RecognitionEventArgs：事件信号

+ recognizing
+ 属性：session_id、offset、result（`SpeechRecognitionResult` 对象）



#### SpeechRecognitionResult：识别结果





