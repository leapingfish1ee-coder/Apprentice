# Apprentice Performance Baseline

基准日期：2026-09-28  
基准页面：已部署提交 `19e6d54c1ea421568526ff00ead39c380c96b7ef`。后续仅加入性能上报逻辑，不改变地图与雾效渲染，因此本结果可作为当前视觉实现的基线。

## Scenario

- Chromium headless。
- 每个场景连续运行约 8 秒。
- 角色在纵向安全道路上持续移动。
- 同时运行：玩家格间移动、镜头跟随、LOS 视野计算、220ms fog transition、pixel/glitch fog 常态呼吸。
- 外部木纹资源在基准中阻止加载；鼠标指针并非 Canvas 主渲染成本，因此不影响地图性能结论。
- 统计窗口：最近 240 帧。
- slow frame：帧间隔 > 20ms。

## Results

| Viewport | Avg FPS | Current FPS | Avg frame | P95 frame | Avg Canvas draw | P95 Canvas draw | Slow frames |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 800×500 @ DPR 1 | 57.6 | 58.1 | 17.36ms | 16.8ms | 12.70ms | 15.4ms | 2.5% |
| 1440×900 @ DPR 1 | 45.7 | 47.4 | 21.87ms | 33.4ms | 13.95ms | 18.1ms | 27.9% |
| 1440×900 @ DPR 2 | 33.3 | 32.7 | 30.00ms | 33.4ms | 14.71ms | 17.8ms | 75.8% |

## Assessment

当前实现对小视口基本接近 60 FPS，但余量不大。800×500 下 Canvas draw 已消耗约 12.7ms，接近 16.67ms 的完整 60Hz 帧预算。

1440×900 @ DPR 1 已不能稳定维持 60 FPS，平均约 46 FPS；P95 frame 达 33.4ms，说明存在明显的 30 FPS 级长帧。

1440×900 @ DPR 2 是当前主要风险场景，平均约 33 FPS，超过 20ms 的帧约占 75.8%。高 DPI 使 Canvas backing store 达到 2880×1800，像素填充与光栅/合成成本明显增加。

Canvas API 调用测得的 draw time 只覆盖主线程发出绘制命令的时间，不完整包含后续光栅化与合成；因此大视口下 frame time 的额外增长说明瓶颈不只是 JavaScript，还包含 Canvas 像素填充与渲染管线成本。

## Current hotspots

1. Fog 每帧在可见区域重复进行 8×8 子格采样与大量 `fillRect`。
2. 每个 fog 子格常态计算随机相位与 `Math.sin` 呼吸。
3. 地图静态地形、对象和 fog 当前全部在同一个 Canvas 每帧重绘。
4. DPR 最多允许到 2，高分屏会显著扩大 backing store。
5. Fog 的双线性 `sampleFog` 在大量子格上逐帧执行。

## Optimization direction

优先级最高的是 fog，而不是降低角色移动动效。

- 将静态地图缓存到 OffscreenCanvas / 后台 Canvas，只在镜头变化时做位图拷贝。
- 将 fog 做成低分辨率独立 mask，再以 nearest-neighbor / pixelated 方式放大合成。
- Fog 常态烟雾纹理可降到 20–30Hz 更新，玩家与镜头仍保持 requestAnimationFrame。
- 预计算 glitch block 的位置、周期和相位，避免每帧重复 hash。
- 将双线性 fog sampling 从每个绘制子块的即时计算，改为状态变化时更新一个低分辨率 fog buffer。
- 对高 DPI 可考虑动态 resolution scale 或将 Canvas DPR 上限从 2 调整为 1.5，前提是视觉检查确认细节损失可接受。

优化验收目标：1440×900 @ DPR 1 接近稳定 60 FPS，P95 frame ≤ 20ms；DPR 2 至少明显高于当前约 33 FPS，并降低 slow-frame 比例。
