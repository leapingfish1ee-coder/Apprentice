# Apprentice Design Tokens

本文件是页面设计与动效 Token 的唯一记录源。实现调整时应同步更新此文档。

## Source guidance

交互动效遵循 Emil Kowalski 官方 `emil-design-eng` skill：

- https://github.com/emilkowalski/skills/blob/main/skills/emil-design-eng/SKILL.md
- 进入/退出：强 ease-out。
- 屏幕内状态变化：强 ease-in-out。
- UI 动效通常控制在 300ms 内。
- 动态状态应可中断并从当前位置重新定向。
- 优先 transform / opacity，避免通过布局属性制造动效。
- `prefers-reduced-motion` 保留信息，但移除非必要运动。

## Core color

| Token | Value | Purpose |
| --- | --- | --- |
| canvas | `#f2efe6` | 页面与地图外画布 |
| ink | `#3d372f` | 主要文字 |
| muted | `#756d61` | 次级状态 |
| faint | `#a69c8e` | 提示与弱标签 |
| rule | `rgba(61,55,47,.18)` | HUD 细分隔 |

## Easing

| Token | Value | Purpose |
| --- | --- | --- |
| ease-out | `cubic-bezier(.23,1,.32,1)` | 进入、退出、即时反馈 |
| ease-in-out | `cubic-bezier(.77,0,.175,1)` | 屏幕内连续状态变化 |

## Pointer motion

| Token | Value |
| --- | --- |
| pointer-enter | `70ms` |
| pointer-exit | `50ms` |
| pointer-wobble | `150ms` |
| wobble angles | `0 → -4.5 → 3.2 → -1.8 → .8 → 0deg` |

指针移入使用 ease-out，移出采用更快退出；点击颤动围绕真实热点，不使用缩放。

## Map motion

| Token | Value | Purpose |
| --- | --- | --- |
| tile move | `115ms` | 玩家单格移动 |
| camera follow factor | `.18/frame` | 镜头追随 |
| zoom follow factor | `.18/frame` | 滚轮缩放收敛 |
| zoom range | `.72–1.65` | 地图镜头 |

## Vision

| Token | Value | Purpose |
| --- | --- | --- |
| clear radius | `4.25 tiles` | 完整显示区域 |
| max radius | `6.25 tiles` | glitch 过渡终点 |
| unseen alpha | `1.0` | 未发现区域纯黑 |
| seen-outside alpha | `.64` | 已发现但视野外 |
| vision transition | `220ms` | 视野状态变化 |
| vision easing | strong ease-in-out | 屏幕内雾层 morph |
| opaque tiles | tree / stone / house / shrine | LOS 阻挡 |

视野变化使用可重定向 alpha 过渡。新输入发生时，以当前 fog alpha 为新的起点，不重新从旧 keyframe 起点播放。

## Glitch fog

| Token | Value | Purpose |
| --- | --- | --- |
| cells per tile | `8 × 8` | 像素化粒度 |
| alpha variance | `.11` | 固定块间差异 |
| pulse amplitude | `.045` | 常态明灭幅度 |
| pulse period | `3.8–6.8s` | 每个像素块独立随机周期 |
| geometry | fill only | 禁止描边 |
| cell overlap | `.35px each side` | 消除网格缝/描边感 |

Glitch 只通过相邻像素块透明度差和少量横向块合并形成，不使用 stroke、outline 或格线。常态动画只改变 opacity，不移动块的位置，形成低频烟雾式呼吸，避免躁动。

## Reduced motion

启用 `prefers-reduced-motion: reduce` 时：

- 视野 alpha 立即收敛到目标。
- glitch 常态 pulse 停止。
- 指针点击颤动停止。
- 保留黑雾层级、已探索/未探索信息和颜色/透明度差异。
