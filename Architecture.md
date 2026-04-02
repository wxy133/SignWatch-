# SignWatch 架构说明

## 1. 工程分层

- `SignWatchShared`
  - 纯业务与数据层
  - 手势词汇、传感器帧、识别结果、识别记录、校准配置
  - 滑动窗口缓存、特征提取、启发式分类器、历史存储
- `SignWatchWatchApp`
  - watchOS 采集与展示层
  - CoreMotion 采样、识别状态机、识别主界面、历史页、校准页
- `SignWatchiOS`
  - iPhone companion
  - Watch Connectivity 消息接收、语音播报、设置页

## 2. 识别流程

1. watchOS 以 50Hz 采集 `userAcceleration`、`rotationRate` 与姿态角。
2. `MotionSampleBuffer` 按时间窗口保留近 4 秒数据。
3. `MotionRecognitionPipeline` 每隔 0.35 秒抽取 1.5 秒窗口。
4. `GestureFeatureExtractor` 生成均值、峰值、角度变化、漂移与换向特征。
5. `HeuristicGestureClassifier` 用默认模板和本地校准模板进行相似度匹配。
6. 置信度高于 0.60 时生成识别结果，写入历史并尝试向 iPhone 发送播报请求。
7. 连续 3 次低置信度触发校准提示。

## 3. 校准逻辑

- 首次进入时可手动开始校准。
- 校准页按 10 个手语词逐项采集当前最近一段有效动作窗口。
- 每个词的校准样本会保存在本地 JSON 文件中，不上传到外部服务。
- 分类阶段会将默认模板与校准平均模板进行融合。

## 4. iPhone 联动

- watchOS 通过 `WCSession.sendMessageData` 发送 `SpeechRequest`。
- iOS companion 收到消息后调用 `AVSpeechSynthesizer` 使用中文语音播报。
- 当 iPhone 不可达时，watch 端仅显示文字并提示等待手机可达，不报错。

## 5. 模型接入点

- 当前项目默认使用启发式基线分类器，保证离线、本地、零服务器。
- `Models/SignWatchGestureClassifier.mlpackage` 预留为后续真实训练模型入口。
- 如果替换为正式 Core ML 模型，可在 `MotionRecognitionPipeline` 中接入 `MLModel` 推理结果并复用现有 UI、历史与联动逻辑。
