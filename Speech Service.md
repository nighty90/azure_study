# Speech Service



## Notes

+ 



## Overview



## Speech to text (STT)

### STT REST API for short audio

#### Notes

+ 注意这里是专门给短音频用的 STT REST API，而不是 STT REST API，这两者不一样
+ 用处有限，请尽量使用 SDKs 或 STT REST API
  + 直接传输音频的请求最多只能包含 60 秒的音频
  + 仅能处理 WAV 和 OGG 格式的音频
  + 仅返回最终结果，不提供部分结果
  + 不支持语音翻译、批量转录、自定义STT



#### Usage

+ POST 请求

  ```
  https://<REGION>.stt.speech.microsoft.com/speech/recognition/conversation/cognitiveservices/v1
  ```

+ Headers

  + `Ocp-Apim-Subscription-Key`：必须，key，与 `Authorization` 二选一即可

  + `Authorization`：必须，带上 "Bearer" 前缀的 token，与 `Ocp-Apim-Subscription-Key` 二选一即可

  + `Content-type`：必须，提供的音频格式及编解码器

    + WAV 格式：audio/wav; codecs=audio/pcm; samplerate=16000
    + OGG 格式：audio/ogg; codecs=opus

  + `Pronunciation-Assessment`：指定如何评估发音，是 Base64 编码的 JSON

    + python 中进行 base64 编码的例子

      ```python
      import base64
      import json
      
      ParamsDict = {
          "ReferenceText": "Good morning.",
          "GradingSystem": "HundredMark",
          "Dimension": "Comprehensive"
      }
      ParamsJson = json.dumps(ParamsDict)
      ParamsBase64 = base64.b64encode(bytes(ParamsJson, "utf-8"))
      Params = str(ParamsBase64, "utf-8")
      ```

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

+ url 参数

  + `language`：必须，需要识别的语言
    `format`：指定结果格式，可选 simple / detailed，默认simple
    + simple 内容：RecognitionStatus、DisplayText、Offset 和 Duration
    + detailed 内容：包括显示文本的四种不同的表示形式
  + `profanity`：不雅内容的处理方式，可选 masked / removed / raw，默认为 masked
  + `cid`：自定义模型的 id

+ 申请 token

  + POST 请求

    ```
    https://<REGION>.api.cognitive.microsoft.com/sts/v1.0/issueToken
    ```

  + headers

    + `Ocp-Apim-Subscription-Key`：<YOUR_SUBSCRIPTION_KEY>
    + `Content-type`：application/x-www-form-urlencoded
    + `Content-Length`：0

  + 每个 token 的有效期为 10min，考虑到网络延迟等，建议同一 token 仅使用 9min



### STT Python SDK

#### Notes

+ Python SDK：`azure-cognitiveservices-speech`

+ 导入：`import azure.cognitiveservices.speech as speechsdk`

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
  auto_detect_source_language_config = \
  	speechsdk.languageconfig.AutoDetectSourceLanguageConfig(
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

+ 



#### Quickstart: Recognize and convert speech to text

以从麦克风中识别为例

1. 设置 speech：必须设置 key、region 和语言

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

1. 在本例中，使用音频文件作为输入

2. 设置全局变量用于之后退出循环：`done = False`

3. 构造回调函数用以停止识别

   ```python
   def stop_cb(evt):
       print('CLOSING on {}'.format(evt))
       speech_recognizer.stop_continuous_recognition()
       nonlocal done
       done = True
   ```

   + 必须调用 `stop_continuous_recognition()`
   + 必须改变全局变量来退出循环

4. 将事件信号（识别器的多个属性）与回调函数相连接

   ```python
   speech_recognizer.recognizing.connect(
       lambda evt: print('RECOGNIZING: {}'.format(evt))
   )
   speech_recognizer.recognized.connect(
       lambda evt: print('RECOGNIZED: {}'.format(evt))
   )
   speech_recognizer.session_started.connect(
       lambda evt: print('SESSION STARTED: {}'.format(evt))
   )
   speech_recognizer.session_stopped.connect(
       lambda evt: print('SESSION STOPPED {}'.format(evt))
   )
   speech_recognizer.canceled.connect(
       lambda evt: print('CANCELED {}'.format(evt))
   )
   
   speech_recognizer.session_stopped.connect(stop_cb)
   speech_recognizer.canceled.connect(stop_cb)
   ```

   + 这里作为示例，多个信号的回调函数为打印收到的信号
   + 注意同一个信号可以连接多个回调函数

5. 开始识别

   ```python
   speech_recognizer.start_continuous_recognition()
   while not done:
       time.sleep(0.5)
   ```

   



#### Important Classes

+ `SpeechConfig`：语音设置

  + 构造函数

    ```python
    SpeechConfig(subscription: str | None = None, region: str | None = None, endpoint: str | None = None, host: str | None = None, auth_token: str | None = None, speech_recognition_language: str | None = None)
    ```

    + 可以有 4 种初始化方式
      1. 从 subscription：使用 `subscription` 和 `region`
      2. 从 endpoint：使用 `endpoint`，此时 `subscription` 和 `auth_token` 可选
         + 通常用于使用训练好的自定义模型
      3. 从 host：使用 `host`，此时 `subscription` 和 `auth_token` 可选
      4. 从 token：使用 `auth_token` 和 `region`
    + 可以在这里设置 `speech_recognition_language` 来指定识别器的语言

+ `audio.AudioConfig`：音频设置

  + 构造函数

    ```python
    AudioConfig(use_default_microphone: bool = False, filename: str = None, stream: AudioInputStream = None, device_name: str = None)
    ```

    + `use_default_microphone`：是否使用默认系统麦克风进行音频输入
    + `device_name`：指定要使用的音频设备的 ID
    + `filename`：指定音频输入文件
    + `stream`：指定使用的输入流，为 `AudioInputStream` 对象

+ `SpeechRecognizer`：识别器

  + 构造函数

    ```python
    SpeechRecognizer(speech_config: SpeechConfig, audio_config: AudioConfig = None, language: str = None, source_language_config: SourceLanguageConfig = None, auto_detect_source_language_config: AutoDetectSourceLanguageConfig = None)
    ```

    + 仅 `speech_config` 为必须参数
    + 语言相关的参数请三选一
      + `language`：手动指定语言，仅指定语言
      + `source_language_config`：手动指定语言，但同时可以指定 endpoint
      + `auto_detect_source_language_config`：自动判断

  + 方法

    + 以下方法都有 `<function_name>_async()` 的异步版本，返回 `ResultFuture` 对象
    + 用 `start_<suffix>()` 开始识别后，用对应的 `stop_<suffix>()` 来停止识别
    + 识别结果为 `SpeechRecognitionResult` 对象
    + `recognize_once()`：仅识别一次，根据静音 / 超时（15s）来判断结束
    + `start_continuous_recognition()`：开始连续识别
      + 产生的识别信号由对应的 EventSignal 对象接收
      + 需要用 `connect()` 方法为其设置回调函数以处理识别结果
    + `start_keyword_recognition(model)`：调用 keyword 模型识别，此时识别器在传入 keyword 后开始识别

+ `EventSignal`：处理事件信号

  + `connect()`：将事件信号与回调函数连接
    + 注意同一个信号可以连接多个回调函数，调用的顺序与连接时的顺序一致

+ `RecognitionEventArgs`：事件信号

  + 属性：session_id、offset、result（`SpeechRecognitionResult` 对象）

+ `SpeechRecognitionResult`：识别结果