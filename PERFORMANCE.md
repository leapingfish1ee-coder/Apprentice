# Apprentice Performance Baseline

基准日期：2026-09-28

## Scenario

- Chromium headless。
- 每个场景连续运行约 8 秒。
- 角色在纵向安全道路上持续移动。
- 同时运行：玩家格间移动、镜头跟随、LOS 视野计算、220ms fog transition、pixel/glitch fog 常态呼吸。
- 统计窗口：最近 240 帧。
- slow frame：帧间隔 > 20ms。
- 精确视口通过 Chrome DevTools Protocol `Emulation.setDeviceMetricsOverride` 设置。
- 性能 job 与 Pages deploy 独立运行，因此部署服务异常不会污染性能测量。

## Before optimization

基线提交：`19e6d54c1ea421568526ff00ead39c380c96b7ef`

| Viewport | Avg FPS | Avg frame | P95 frame | Avg Canvas draw | P95 Canvas draw | Slow frames |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| 800×500 @ DPR 1 | 57.6 | 17.36ms | 16.8ms | 12.70ms | 15.4ms | 2.5% |
| 1440×900 @ DPR 1 | 45.7 | 21.87ms | 33.4ms | 13.95ms | 18.1ms | 27.9% |
| 1440×900 @ DPR 2 | 33.3 | 30.00ms | 33.4ms | 14.71ms | 17.8ms | 75.8% |

旧实现的主要成本来自：

1. Fog 每帧对可见地块进行 8×8 子格双线性采样，并发出大量 `fillRect`。
2. 每个 fog 子格逐帧重新计算 pulse phase / period / `Math.sin`。
3. 静态地形、树木、建筑与 fog 共用一个 Canvas，每帧全部重新绘制。
4. Canvas DPR 上限为 2，高分屏 backing store 像素成本显著放大。
5. `updateFogTransition()` 在没有 transition 时仍每帧遍历并复制完整 fog grid。

## Optimization implemented

优化主体提交：`0acbb116794f6c0ceb42977caea0544a90d8c91e`

### Static map cache

地形、道路、水面、树木、建筑、石块、花与祠堂启动时预渲染到 `2×` 后台 Canvas。主循环只根据镜头范围裁剪并缩放这一张位图，不再逐格重建路径与对象。

### Low-resolution fog texture

Fog 状态仍由原有 LOS 与 220ms transition 驱动，但视觉层改为 `288×224` alpha texture，即每个地图格保持 8×8 logical fog cells。

- 双线性采样坐标启动时预计算。
- pulse phase、period 与 glitch block 几何启动时预计算。
- fog texture 以 30Hz 更新。
- 主循环每帧只合成一张 nearest-neighbor fog texture。
- 玩家、镜头、缩放与输入继续使用 requestAnimationFrame，不降低交互帧率。

### Transition work elimination

没有 fog transition 时，`updateFogTransition()` 直接返回，不再每帧复制 36×28 fog state。

### High-DPI control

主 Canvas 的 DPR 上限从 2 调整为 1.5。DOM HUD 仍按浏览器原生分辨率渲染，因此文字清晰度不受该上限影响。

## After optimization

精确复测提交：`0ff5f62a99d4099abd42a451fb6a195b0260e07a`

| Viewport | Avg FPS | Avg frame | P95 frame | Avg Canvas draw | P95 Canvas draw | Slow frames |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| 800×500 @ DPR 1 | 60.0 | 16.67ms | 16.8ms | 1.63ms | 4.6ms | 0% |
| 1440×900 @ DPR 1 | 60.0 | 16.67ms | 16.8ms | 1.48ms | 3.9ms | 0% |
| 1440×900 @ effective DPR 1.5 | 60.0 | 16.67ms | 16.7ms | 1.72ms | 4.9ms | 0% |

## Comparison

800×500 @ DPR 1 的平均 Canvas draw 从 12.70ms 降到 1.63ms，约减少 87%。

1440×900 @ DPR 1 的平均 Canvas draw 从 13.95ms 降到 1.48ms，约减少 89%；平均帧率从约 45.7 FPS 提升到稳定 60 FPS，slow-frame 比例从 27.9% 降到 0%。

高 DPI 采用新的 DPR 1.5 上限，因此不能把优化后数据与旧 DPR 2 视为严格同条件 A/B；从产品运行条件看，原来约 33 FPS / 75.8% slow frames 的风险场景现在在 1440×900 高 DPI 设备上以 effective DPR 1.5 稳定 60 FPS / 0% slow frames。

## Visual validation

GitHub Actions 的固定 800×500 visual check 已通过。静态缓存后地图结构、道路、角色、查看标记与黑雾范围保持正常；fog 仍保留 pixel/glitch 边缘与低频呼吸，但不再逐帧重建大量矩形。

## Current assessment

当前测试场景已经达到既定目标：

- 1440×900 @ DPR 1：稳定 60 FPS。
- P95 frame：约 16.8ms，低于原目标 20ms。
- slow frames：0%。
- 高 DPI effective DPR 1.5：稳定 60 FPS。
- Canvas 主线程绘制成本约 1.5–1.7ms，已从主要瓶颈降为较低成本。

后续若地图尺寸、动态实体数量或粒子系统显著扩张，应继续使用同一 `performance-check` 场景做回归；当前不需要通过降低玩家移动动画或进一步牺牲视野表现来换取性能。
