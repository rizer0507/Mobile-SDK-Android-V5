# MSDK 中接入 ONNX / PyTorch 模型并联动镜头的实现建议

本文结合本仓库当前结构，回答两个高频问题：

1. 如何让 ONNX / `.pt` 模型“上机”并使用无人机机载算力。
2. 识别到目标后，反控镜头（对焦、变焦）在哪一层实现。

## 先说结论

- 这个仓库是 **DJI Mobile SDK Android V5**，定位是“移动端无人机解决方案”，即 App 侧开发，不是机载算法部署工程。可从仓库 README 对 MSDK 的定义看出它强调 mobile device 侧能力。  
- 因此：若你要严格使用“机载算力”运行 ONNX/PT，通常不是直接在这个仓库里把模型文件丢进去就完成，而是采用机载平台（如负载/机载计算平台）承载算法，再与 MSDK App 通信。  
- 若你接受“移动端算力”（手机/遥控器 Android）运行模型，则可以在本仓库 sample 的视频流回调处做推理，然后通过 CameraKey/KeyManager 反控镜头。

## 问题 1：ONNX / PT 如何搭载并利用算力

### A. 你要的是“机载算力”（Aircraft/Payload onboard）

建议采用“算法进程（机载） + MSDK App（控制/可视化）”分层：

1. 算法侧：将 `.pt` 导出为 ONNX（推荐），机载端使用 ONNX Runtime / TensorRT / 其他板卡推理引擎运行。  
2. 通信侧：算法结果（bbox、类别、置信度、目标中心点）回传给 MSDK App。  
3. 控制侧：MSDK App 根据检测结果调用相机键值接口执行对焦、点按变焦、云台控制。

> 本仓库本身并未提供“机载模型部署 SDK 示例”；它提供的是移动端 MSDK 控制框架与样例。

### B. 如果先跑通（可先用“移动端算力”）

可在 Android sample 中走以下最短路径：

1. 从相机流拿帧（YUV/RGB）。  
2. 帧预处理（resize/normalize）。  
3. ONNX Runtime Mobile 或 PyTorch Mobile 推理。  
4. NMS 后得到目标中心点。  
5. 通过 CameraKey 执行 TapZoom / Focus / Zoom。  

这样你能先验证：目标检测 → 镜头自动联动 的闭环，然后再把推理侧迁移到机载端。

## 问题 2：识别后反控镜头在哪个模块做

推荐放在 **`android-sdk-v5-sample` 的 ViewModel 层（models）**，由页面层触发、模型层执行键值调用：

- `LookAtFragment` 已经示例了触摸后调用 `lookAtViewModel.startTapZoomPoint(...)`。  
- `LookAtVM` 里用 `CameraKey.KeyTapZoomAtTarget` 做点按变焦 action。  
- 若是通用键值 set/get/action，可参考 `KeyBaseStructure` 中 `KeyManager.setValue` / `performAction` 的封装。  
- 对焦模式切换可参考 UXSDK 里的 `FocusModeWidgetModel`，其使用 `CameraKey.KeyCameraFocusMode`。  

也就是说，你可以新增一个例如 `TargetTrackingVM`：

- 输入：检测框中心点（归一化坐标 x,y）与目标尺度。  
- 输出：
  - 触发 `KeyTapZoomAtTarget`（把目标点送给镜头）；
  - 按目标尺寸动态调变焦倍率；
  - 必要时联动云台 yaw/pitch 做目标保持居中。

## 与当前仓库结构的对应关系

- MSDK 的定位（移动端方案）：见 `README.md` / `README_CN.md`。  
- Sample 组织：`android-sdk-v5-sample`（业务示例）+ `android-sdk-v5-uxsdk`（UI/组件）。  
- 镜头控制可落在 sample models（例如 `LookAtVM`），页面事件在 fragment（例如 `LookAtFragment`）。

## 实战建议（你这个“识别水草 → 自动对焦/放大”的场景）

1. 第一阶段：先在移动端完成闭环（识别 + TapZoom + FocusMode），确认控制逻辑正确。  
2. 第二阶段：把推理从移动端迁移到机载端，仅保留 MSDK 侧控制与可视化。  
3. 第三阶段：增加安全策略（置信度门限、目标持续帧数、变焦上限、飞行状态约束），避免误触发。

