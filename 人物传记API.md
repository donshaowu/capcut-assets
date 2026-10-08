---
name: capcut-clip
description: 根据视频、标题、字幕和音频自动创建并保存剪映草稿，支持连续时间线、历史纪录片风格转场、标题、字幕、背景音乐和结果记录。
---

# 剪映小助手自动剪辑规范

## 适用范围

用于将多段视频自动组合成剪映草稿，并完成视频排序、时间线、转场、标题、字幕、背景音乐和草稿保存。

如果用户在提供本模板的同时，另外提出具体的项目要求、参数或提示词，应以用户当前明确提出的要求为最高优先级；本模板仅作为通用流程和默认参数参考，未被用户明确修改的部分继续按本模板执行。

官方文档：<https://docs.jcaigc.cn/>

API 基础地址：

```text
https://capcut-mate.jcaigc.cn/openapi/capcut-mate/v1
```

## 一、标准执行顺序

```text
create_draft
→ add_videos
→ add_captions（标题）
→ add_captions（分段字幕）
→ add_audios
→ save_draft
```

所有接口都使用上一步返回的 `draft_url`。

## 二、输入要求

- 视频 URL：按文件编号或用户指定顺序提供；
- 标题：一条，覆盖整个视频；
- 分段字幕：每段视频对应一条；
- 音频 URL：必须是接口可访问的公开直链；
- 画布默认使用 `720×1280`，即9:16竖屏720P；
- 视频数量必须与字幕数量一致；
- 所有时间参数单位均为微秒，1秒等于1,000,000微秒。

## 三、创建草稿：create_draft

请求参数：

```json
{
  "width": 720,
  "height": 1280
}
```

保存接口返回的 `draft_url`，作为后续操作的目标草稿。

## 四、添加视频、时间线和转场：add_videos

接口：

```text
POST /add_videos
```

`video_infos` 必须传入 JSON 字符串。示例：

```json
{
  "draft_url": "DRAFT_URL",
  "video_infos": "[{\"video_url\":\"VIDEO_01_URL\",\"width\":720,\"height\":1280,\"start\":0,\"end\":5000000,\"duration\":5000000,\"volume\":0},{\"video_url\":\"VIDEO_02_URL\",\"width\":720,\"height\":1280,\"start\":5000000,\"end\":10000000,\"duration\":5000000,\"transition\":\"叠化\",\"transition_duration\":400000,\"volume\":0}]"
}
```

视频字段：

- `video_url`：视频公开直链，必填；
- `width`、`height`：素材尺寸；
- `start`、`end`：时间线起止时间，必填；
- `duration`：素材原始总时长，可由接口自动获取；如果接口无法自动获取，应使用实际素材时长；
- `volume`：视频原声，添加背景音乐时通常设为 `0`；
- `transition`：转场名称；
- `transition_duration`：转场时长，单位微秒。

时间线规则：

- 第一段从 `0` 开始；
- 后一段的 `start` 必须等于前一段的 `end`；
- 不得出现重叠或空白；
- 视频按编号顺序添加；
- 第1段通常不设置转场；
- 第2段及之后根据相邻内容添加转场；
- 根据前后视频内容选择官方支持的转场名称和时长；不得把某一种转场或固定时长作为所有项目的强制要求；
- 转场时长范围为 `100000—2500000` 微秒；
- 转场名称必须使用官方支持列表中的准确名称。

## 五、添加标题：第一次调用 add_captions

标题覆盖整个视频时长，放置在画面上方。

```json
{
  "draft_url": "DRAFT_URL",
  "captions": "[{\"start\":0,\"end\":TOTAL_DURATION,\"text\":\"标题文字\"}]",
  "border_color": "#000000",
  "font": "系统",
  "font_size": 14,
  "line_spacing": 10,
  "text_color": "#F2E8C9",
  "transform_y": 1000,
  "alignment": 1
}
```

## 六、添加分段字幕：第二次调用 add_captions

字幕必须与对应视频片段一一对应，时间不得重叠或错位。

```json
{
  "draft_url": "DRAFT_URL",
  "captions": "[{\"start\":0,\"end\":5000000,\"text\":\"第1段字幕\"},{\"start\":5000000,\"end\":10000000,\"text\":\"第2段字幕\"}]",
  "border_color": "#000000",
  "font": "系统",
  "font_size": 12,
  "line_spacing": 10,
  "text_color": "#ffde00",
  "transform_y": -900,
  "alignment": 1
}
```

字幕文本框要求：文本框左右边界与画布左右边界对齐，宽度与画布同宽；字幕文字在文本框内居中对齐。若当前 API 未提供独立的文本框宽度字段，不要虚构参数，应在剪映客户端中将文本框左右边缘分别拖至画布边缘。

## 七、添加背景音频：add_audios

默认音乐链接：

```text
https://karo-1412224486.cos.ap-shanghai.myqcloud.com/mini/08/now.mp3
```

接口：

```text
POST /add_audios
```

`audio_infos` 必须传入 JSON 字符串：

```json
{
  "draft_url": "DRAFT_URL",
  "audio_infos": "[{\"audio_url\":\"https://karo-1412224486.cos.ap-shanghai.myqcloud.com/mini/08/now.mp3\",\"start\":0,\"end\":TOTAL_DURATION,\"duration\":AUDIO_DURATION,\"volume\":0.8}]"
}
```

规则：

- `start` 通常为 `0`；
- `end` 必须等于视频总时长；
- `duration` 为音频文件总时长；
- 背景音乐建议音量为 `0.6—0.8`；
- 音频 URL 必须可以被服务器直接访问；
- 音频过长时，将播放时间裁切到视频总时长；
- 音频过短时，应更换足够长的音频；只有在接口明确支持循环时才允许循环添加，不得虚构循环参数；
- 无特殊需求时不添加 `audio_effect`。

## 八、保存草稿：save_draft

接口：

```text
POST /save_draft
```

请求参数：

```json
{
  "draft_url": "DRAFT_URL"
}
```

只有在视频、转场、标题、字幕和音频全部添加成功后，才调用 `save_draft`。

## 九、自动执行规则

1. 从当前文件夹和已有记录读取视频 URL；
2. 按编号排序并检查视频数量；
3. 检查字幕数量是否与视频数量一致；
4. 创建 `720×1280` 草稿；
5. 按实际视频时长生成连续时间线；
6. 添加视频和合适的历史纪录片风格转场；
7. 分两次添加标题和分段字幕；
8. 添加覆盖全片的背景音乐；
9. 保存草稿；
10. 保存全部接口响应、时间线和最终草稿链接。

## 补充：历史人物视频草稿任务模板

当用户要求将多段历史人物视频制作成剪映草稿时，按以下规则执行：

1. 使用 `create_draft` 创建草稿；
2. 默认画布使用本规范规定的 `720×1280`、9:16竖屏尺寸；如用户明确指定其他尺寸，可在不影响其他参数的情况下覆盖画布尺寸；
3. 使用 `add_videos` 按视频编号顺序添加历史人物视频；
4. 根据每段视频的实际素材时长设置独立的 `start`、`end` 和 `duration`；如果接口能够自动获取素材时长，应优先使用实际获取结果，避免手动填写错误；
5. 各段视频必须首尾连续排列，后一段的 `start` 等于前一段的 `end`，这样视频之间不会出现空白，也不会发生意外时间重叠；
6. 根据每段视频的实际播放时长计算时间线，不固定视频段数，也不固定每段时长；
7. 总时长等于所有视频实际播放时长之和；
8. 从第1段开始按顺序累计计算每段的 `start` 和 `end`，不得使用固定段数或固定时长；
9. 在相邻视频之间添加符合内容的转场，转场属于 `add_videos` 中的视频参数；转场不得造成额外空白或意外重叠，转场时长必须按照接口规定写入对应视频参数；
10. 第一次调用 `add_captions` 添加剧本标题，标题覆盖全片并放置在画面上方；
11. 第二次调用 `add_captions` 添加生平经历字幕，每条字幕只覆盖对应视频的时间范围，并放置在画面下方；
12. 标题文字使用剧本生成的标题，字幕文字使用剧本生成的对应生平经历，不得自行改写历史内容；
13. 完成视频、转场、标题和字幕后，调用 `save_draft` 保存草稿；
14. 输出最终草稿链接，并保存视频顺序、时间线、转场、标题和字幕记录。

## 十、输出记录

至少保存以下内容：

- 最终 `draft_url`；
- 创建草稿结果；
- 添加视频结果；
- 添加标题结果；
- 添加字幕结果；
- 添加音频结果；
- 保存草稿结果；
- 每段视频的编号、URL、开始时间、结束时间、转场和字幕内容。

## 十一、错误处理

- 视频数量与字幕数量不一致：停止并报告；
- 视频或音频 URL 无法访问：记录具体素材并重试；
- 转场名称不支持：改用官方列表中的 `叠化`；
- 单个素材失败：只重试失败步骤，不重复创建已成功的内容；
- 任一步骤失败：保留已完成的接口响应和错误信息；
- 不得把接口失败报告为成功，也不得伪造草稿链接。

## 官方接口文档

- 总文档：<https://docs.jcaigc.cn/>
- add_videos：<https://docs.jcaigc.cn/docs/add_videos.zh.html>
- add_captions：<https://docs.jcaigc.cn/docs/add_captions.zh.html>
- add_audios：<https://docs.jcaigc.cn/docs/add_audios.zh.html>
- save_draft：<https://docs.jcaigc.cn/docs/save_draft.zh.html>
+

## 通用剪映小助手扩展接口

以下接口适用于人物传记、书籍分享、城市旅游以及其他视频项目。项目专属章节中的默认画布、字幕、音乐、转场和风格要求优先于本节通用默认值。

### 通用接口顺序

```text
检查素材
→ upload_file（本地文件没有公开URL时）
→ get_audio_duration（需要真实音频时长时）
→ create_draft
→ add_videos / add_images
→ add_captions
→ add_audios
→ add_filters / add_effects / add_masks（按项目需要）
→ add_keyframes（片尾淡黑、音量或其他关键帧效果）
→ save_draft
→ gen_video
→ gen_video_status
```

所有写入接口都必须沿用上一步返回的 `draft_url`。所有时间参数单位均为微秒。

### 文件上传：upload_file

本地素材没有公开URL时，先调用：

```text
POST /upload_file
```

上传成功后使用接口返回的公开素材URL，再传给 `add_videos`、`add_images` 或 `add_audios`。不得把本地路径直接当作远程URL提交。

### 音频时长查询：get_audio_duration

添加音频前，优先调用：

```text
POST /get_audio_duration
```

使用接口返回的真实音频时长填写 `audio_infos.duration`，不得凭经验猜测或硬编码。

### 添加图片：add_images

```text
POST /add_images
```

用于添加封面、片尾黑场图、背景图、装饰图或遮挡图。使用 `image_infos` JSON字符串，并明确 `image_url`、`start`、`end`。图片素材不得遮挡人物主体或字幕，除非项目明确要求。

### 滤镜：add_filters

```text
POST /add_filters
```

用于统一色调、复古、黑白或电影感滤镜。默认不额外改变原素材色调；只有项目要求或素材色彩明显不一致时才使用。滤镜名称必须来自官方支持列表。

### 特效：add_effects

```text
POST /add_effects
```

用于添加剪映支持的视频特效。历史纪录片、书籍分享和城市旅游项目默认不使用强烈、卡通、综艺或现代故障特效；使用前必须确认特效名称和作用时间有效。

### 遮罩：add_masks

```text
POST /add_masks
```

用于局部显隐、柔边遮挡、圆形或矩形遮罩等效果。遮罩需要明确目标片段ID、位置、宽高、羽化和旋转参数。遮罩不是普通转场的替代品。

### 遮罩关键帧：add_mask_keyframes

```text
POST /add_mask_keyframes
```

只有片段已经通过 `add_masks` 添加遮罩后，才能为遮罩添加位置、大小、羽化或旋转关键帧。不得对没有遮罩的片段直接调用。

### 普通关键帧：add_keyframes

```text
POST /add_keyframes
```

支持的常用属性：

```text
KFTypePositionX
KFTypePositionY
KFTypeScaleX
KFTypeScaleY
KFTypeRotation
KFTypeAlpha
KFTypeSaturation
KFTypeContrast
KFTypeBrightness
KFTypeVolume
```

#### 片尾淡黑默认规则

最后一段视频结束前约1秒开始添加透明度关键帧：

```json
[
  {
    "segment_id": "LAST_SEGMENT_ID",
    "property": "KFTypeAlpha",
    "offset": 片尾前1000000,
    "value": 1
  },
  {
    "segment_id": "LAST_SEGMENT_ID",
    "property": "KFTypeAlpha",
    "offset": 视频结束时间,
    "value": 0
  }
]
```

规则：

- 画面从正常不透明逐渐变为全透明；
- 底色为黑色时形成柔和淡黑；
- 不使用突然黑屏、闪黑或中间黑场；
- 添加关键帧后必须再次调用 `save_draft`；
- 必须保存关键帧接口响应和目标 `segment_id`；
- 如果需要额外保持纯黑时长，应添加片尾黑场图片或黑色视频素材，不得假设透明度关键帧会自动延长时间线。

#### 音频淡出

如果官方接口未提供可验证的音量关键帧或音频淡出参数，不得声称音乐已淡出。应使用官方明确支持的音量关键帧、音频淡出字段，或在剪映客户端中手动完成。视频画面淡黑和音乐淡出必须分别核验。

### 保存草稿：save_draft

所有视频、图片、字幕、音频、滤镜、特效、遮罩和关键帧处理完成且接口均成功后，再调用：

```text
POST /save_draft
```

任何一步失败都不得报告为草稿已完成。

### 最终导出：gen_video

```text
POST /gen_video
```

导出接口是异步接口。提交成功只代表任务进入队列，不代表视频已生成。按官方要求保存返回信息并继续查询。

### 导出状态：gen_video_status

```text
POST /gen_video_status
```

持续查询直到：

- `completed`：读取最终 `video_url`；
- `failed`：读取并记录 `error_message`；
- `pending` 或 `processing`：继续轮询；
- 不得在未完成时伪造最终视频链接。

### 通用结果记录

每个项目至少保存：

- `draft_url`；
- 创建草稿响应；
- 视频、图片、音频、字幕、滤镜、特效、遮罩和关键帧响应；
- 每个片段的编号、素材URL、开始时间、结束时间和时长；
- 转场名称及转场时长；
- 片尾淡黑开始时间、结束时间和关键帧参数；
- 音频起止时间、真实时长、音量和是否完成淡出；
- `gen_video` 响应；
- `gen_video_status` 最终状态；
- 最终导出视频URL；
- 失败步骤和重试记录。

### 官方接口索引

- 总文档：<https://docs.jcaigc.cn/>
- 调用指南：<https://docs.jcaigc.cn/guide/llm-guide.zh.html>
- 精简契约：<https://docs.jcaigc.cn/guide/llm-contract.zh.html>
- create_draft：<https://docs.jcaigc.cn/docs/create_draft.zh.html>
- upload_file：<https://docs.jcaigc.cn/docs/upload_file.zh.html>
- get_audio_duration：<https://docs.jcaigc.cn/docs/get_audio_duration.zh.html>
- add_videos：<https://docs.jcaigc.cn/docs/add_videos.zh.html>
- add_images：<https://docs.jcaigc.cn/docs/add_images.zh.html>
- add_audios：<https://docs.jcaigc.cn/docs/add_audios.zh.html>
- add_captions：<https://docs.jcaigc.cn/docs/add_captions.zh.html>
- add_filters：<https://docs.jcaigc.cn/docs/add_filters.zh.html>
- add_effects：<https://docs.jcaigc.cn/docs/add_effects.zh.html>
- add_masks：<https://docs.jcaigc.cn/docs/add_masks.zh.html>
- add_mask_keyframes：<https://docs.jcaigc.cn/docs/add_mask_keyframes.zh.html>
- add_keyframes：<https://docs.jcaigc.cn/docs/add_keyframes.zh.html>
- save_draft：<https://docs.jcaigc.cn/docs/save_draft.zh.html>
- gen_video：<https://docs.jcaigc.cn/docs/gen_video.zh.html>
- gen_video_status：<https://docs.jcaigc.cn/docs/gen_video_status.zh.html>
+

## 补充：常用查询、语音识别与辅助接口

本节为通用扩展能力。除特别注明外，均服务于当前项目对应的视频草稿。

### 一、草稿核验

#### get_draft

```text
GET /get_draft?draft_id=DRAFT_ID
```

用途：

- 检查草稿是否真实存在；
- 获取草稿内的视频、音频、图片和配置文件；
- 核验素材是否完整；
- 保存草稿后进行最终检查。

规则：

- `draft_id` 从 `draft_url` 中提取，不自行编造；
- 草稿接口成功不等于草稿内容完整；
- 必须在关键写入完成后调用并保存响应；
- 核验失败时不得报告草稿完成。

### 二、语音识别服务

语音识别不是剪映小助手接口，使用独立服务：

```text
https://autosubrt.jcaigc.cn/openapi/autosubrt/v1
```

不得与剪映小助手地址混用。

#### asr：语音转文案和逐字时间线

```text
POST /asr
```

返回带标点的完整文案、分句文本和逐字时间线。适合根据旁白音频自动生成字幕时间轴。

#### asr_text：语音转纯文本

```text
POST /asr/text
```

只返回文案，不返回时间线。适合只需要整理旁白文字的场景。

#### asr_srt：语音转SRT字幕

```text
POST /asr/srt
```

直接生成SRT字幕文件，适合快速生成外部字幕文件。

#### asr_text_align：文本与音频对齐

```text
POST /asr/text/align
```

把已经确定的旁白文案与音频逐句对齐。适合历史人物、书籍分享和城市旅游中已有固定文案的情况。

规则：

- 音频转字幕优先使用 `asr` 或 `asr_srt`；
- 已有正式文案但需要准确时间轴时使用 `asr_text_align`；
- 语音识别接口返回HTTP成功不等于业务成功，必须检查返回体中的 `code`；
- 失败时记录错误码和错误信息。

### 三、文字样式与动画查询

#### add_text_style

用于对字幕中的关键词设置颜色、字号和高亮样式。

适用关键词：

- 人物姓名；
- 书名和作者；
- 城市名和景点名；
- 历史事件；
- 重点旁白词语。

关键词必须使用官方支持的样式格式，不得凭空编造样式JSON。

#### get_text_animations

查询官方支持的文字入场、出场和循环动画名称。调用 `add_captions` 前，若项目需要文字动画，先查询名称。

#### get_text_effects

查询官方支持的文字花字和文字特效。默认项目不添加花字，除非用户明确要求。

### 四、画面、滤镜和贴纸查询

#### get_effects

查询官方支持的视频特效列表。使用 `add_effects` 前先查询准确的 `effect_title`，不得猜名称。

#### get_filters

查询官方支持的滤镜列表。使用 `add_filters` 前先查询准确的滤镜名称。

#### get_image_animations

查询官方支持的图片动画名称，适用于静态图片片头、片尾或图片素材动画。

#### search_sticker

按关键词搜索官方贴纸库，返回可用的 `sticker_id`。

#### add_sticker

将已确认的贴纸添加到草稿中，可设置：

- `sticker_id`；
- `start`；
- `end`；
- `scale`；
- `transform_x`；
- `transform_y`。

历史人物项目默认不使用贴纸；书籍分享和城市旅游只有在项目风格允许时才使用。

### 五、配置生成辅助接口

以下接口用于生成或校验传给主接口的JSON字符串，不直接替代主剪辑接口：

- `video_infos)：生成视频片段配置；
- `audio_infos)：生成音频轨道配置；
- `caption_infos)：生成字幕配置；
- `imgs_infos)：生成图片轨道配置；
- `keyframes_infos)：生成关键帧配置；
- `effect_infos)：生成特效配置；
- `filter_infos)：生成滤镜配置；
- `timelines)：生成连续时间线配置。

使用这些辅助接口时，仍必须检查最终生成的时间线、素材URL和参数，不能把辅助接口响应直接当作成片成功。

### 六、导出并发与素材URL

#### gen_video_active_count

查询当前正在导出的任务数量。批量导出前先检查并发状态，避免一次提交过多任务。

#### get_url

用于获取或处理素材URL。所有传入主剪辑接口的素材必须是接口可访问的URL。

### 七、三类项目的默认优先级

#### 人物传记

优先使用：

```text
get_draft
→ add_keyframes（片尾KFTypeAlpha淡黑）
→ get_effects / get_filters（必要时）
→ save_draft
→ gen_video
→ gen_video_status
```

默认不使用贴纸、花字、强烈动画和美颜。

#### 书籍分享

优先使用：

```text
asr / asr_srt / asr_text_align
→ add_text_style
→ get_text_animations / get_text_effects（必要时）
→ add_captions
→ add_keyframes（片尾淡黑）
→ save_draft
→ gen_video_status
```

中文字幕和英文字幕必须使用同一时间轴。

#### 城市旅游

优先使用：

```text
asr / asr_srt
→ add_text_style
→ get_filters / get_effects
→ add_images
→ add_keyframes
→ save_draft
→ gen_video_status
```

城市旅游项目默认不添加固定标题，除非项目提示词明确要求。

### 八、完整结果记录新增字段

除原有记录外，增加：

- `get_draft` 核验响应；
- ASR接口类型、音频URL、业务code和识别结果；
- 使用过的特效、滤镜、文字动画和贴纸ID；
- `segment_id` 与关键帧参数；
- `gen_video_active_count` 查询结果；
- `gen_video` 和 `gen_video_status` 响应；
- 最终导出URL；
- 每一步的失败原因和重试结果。

### 官方接口索引

- get_draft：<https://docs.jcaigc.cn/docs/get_draft.zh.html>
- asr：<https://docs.jcaigc.cn/docs/asr.zh.html>
- asr_text：<https://docs.jcaigc.cn/docs/asr_text.zh.html>
- asr_srt：<https://docs.jcaigc.cn/docs/asr_srt.zh.html>
- asr_text_align：<https://docs.jcaigc.cn/docs/asr_text_align.zh.html>
- add_text_style：<https://docs.jcaigc.cn/docs/add_text_style.zh.html>
- get_text_animations：<https://docs.jcaigc.cn/docs/get_text_animations.zh.html>
- get_text_effects：<https://docs.jcaigc.cn/docs/get_text_effects.zh.html>
- get_image_animations：<https://docs.jcaigc.cn/docs/get_image_animations.zh.html>
- get_effects：<https://docs.jcaigc.cn/docs/get_effects.zh.html>
- get_filters：<https://docs.jcaigc.cn/docs/get_filters.zh.html>
- search_sticker：<https://docs.jcaigc.cn/docs/search_sticker.zh.html>
- add_sticker：<https://docs.jcaigc.cn/docs/add_sticker.zh.html>
- video_infos：<https://docs.jcaigc.cn/docs/video_infos.zh.html>
- audio_infos：<https://docs.jcaigc.cn/docs/audio_infos.zh.html>
- caption_infos：<https://docs.jcaigc.cn/docs/caption_infos.zh.html>
- imgs_infos：<https://docs.jcaigc.cn/docs/imgs_infos.zh.html>
- keyframes_infos：<https://docs.jcaigc.cn/docs/keyframes_infos.zh.html>
- effect_infos：<https://docs.jcaigc.cn/docs/effect_infos.zh.html>
- filter_infos：<https://docs.jcaigc.cn/docs/filter_infos.zh.html>
- timelines：<https://docs.jcaigc.cn/docs/timelines.zh.html>
- gen_video_active_count：<https://docs.jcaigc.cn/docs/gen_video_active_count.zh.html>
- get_url：<https://docs.jcaigc.cn/docs/get_url.zh.html>

## 三类视频通用强制规则：片尾淡黑

人物传记、书籍分享、城市旅游三个项目均必须执行片尾淡黑，不得因项目类型不同而省略。

执行顺序：完成全部视频、转场、字幕、图片、音频、滤镜、特效和遮罩后，针对整条成片最后一个视频片段调用 `add_keyframes`，再调用 `save_draft`，最后导出并核验。

默认参数：

- 从整条视频结束前 `800000—1200000` 微秒开始淡出，默认 `1000000` 微秒；
- 最后一个片段的 `KFTypeAlpha` 从 `1` 平滑变化到 `0`；
- 淡出后保持纯黑 `300000—500000` 微秒；
- 纯黑保持需要通过片尾黑色视频或纯黑图片素材实现，不得假设透明度关键帧会自动延长时间线；
- 背景音乐和旁白在同一时间范围内同步淡出，优先使用官方支持的 `KFTypeVolume` 或音频淡出参数；
- 如果音频淡出接口不可用，必须明确记录“画面已淡黑、音频未自动淡出”，不得伪造完成；
- 淡黑只允许出现在整条视频最后，不得出现在中间转场；
- 不得使用突然黑屏、闪黑、闪白、黑场跳变或额外新画面。

关键帧示例：

```json
[
  {"segment_id":"LAST_SEGMENT_ID","property":"KFTypeAlpha","offset":片尾前1000000,"value":1},
  {"segment_id":"LAST_SEGMENT_ID","property":"KFTypeAlpha","offset":视频结束时间,"value":0}
]
```

核验要求：保存片尾淡黑开始时间、结束时间、纯黑保持时长、目标 `segment_id`、画面关键帧响应和音频淡出响应；未核验成功不得报告成片完成。

## 剪映草稿链接自动导入 PC 端流程

当 `save_draft` 成功并返回有效 `draft_url` 后，必须继续执行本地导入流程，不得只把链接返回给用户后停止。

### 自动执行顺序

```text
save_draft
→ 读取并验证 draft_url
→ 复制 draft_url 到系统剪贴板
→ 打开剪映小助手
→ 将 draft_url 粘贴到草稿创建或导入入口
→ 提交创建草稿
→ 等待小助手返回创建结果
→ 打开剪映 PC 端
→ 检查对应草稿是否出现在项目列表
→ 检查时间线、素材和字幕是否可见
→ 保存本地导入记录
```

### 执行规则

- `draft_url` 必须来自当前任务成功的 `save_draft` 响应，不得手动拼接或使用旧链接；
- 粘贴前必须清空剪映小助手输入框中的旧内容，防止导入错误草稿；
- 提交后必须等待小助手返回成功、任务完成或明确错误状态；
- 如果小助手未提供自动粘贴接口，可使用受控的本地桌面自动化完成复制、粘贴和点击；不得伪造导入成功；
- 打开剪映 PC 端后，按草稿名称、创建时间或返回的草稿标识核对目标项目；
- 必须检查草稿是否能打开、时间线是否完整、片尾淡黑是否存在、视频和字幕是否齐全；
- 导入失败时只重试导入步骤，不重复创建草稿；
- 剪映 PC 端未成功显示草稿时，必须记录失败原因和当前界面状态。

### 本地导入记录

至少保存：

- `draft_url`；
- 草稿名称或草稿标识；
- 复制时间；
- 小助手提交时间和返回结果；
- 剪映 PC 端打开时间；
- 项目是否出现在草稿列表；
- 时间线、视频、字幕、音频和片尾淡黑核验结果；
- 失败原因和重试次数。

### 能力边界

API 负责创建并返回剪映草稿链接；剪映小助手和本地桌面自动化负责复制、粘贴、创建、打开和核验。只有两个环节都成功，才能报告“草稿已自动导入剪映 PC 端”。
