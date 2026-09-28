# Map Design Direction

## Reference

本项目的地图重新设计参考《紫色晶石 / Stoneshard》的地图设计语言，而不是复制其具体地图、像素资产、地点或建筑。

公开资料中，《紫色晶石》的地图把地形、森林、山地、水体、道路和地点作为分层信息；道路可以是直路、弯路、T 字与岔路，森林、田野等生物群系构成旅行空间。其游戏截图还表现出明显的低饱和泥土、重叠树冠、车辙道路、破败建筑与密集环境杂物。

参考：
- https://www.stoneshardwiki.com/en/mechanics/terrain-and-roads/
- https://stoneshard.com/wiki/Biomes_%26_Weather
- https://store.steampowered.com/app/625960/Stoneshard/

## Project translation

### 1. Road-first composition

地图不再由规则十字道路切割，而由一条轻微弯折的东西主路组织视觉，再以北向村落支路和西向岔路形成选择。道路穿过河流时自动转换为旧木桥。

### 2. Forest wall

地图边缘使用高密度树木形成森林墙，内部树丛采用分块噪声形成自然团簇。树冠会越过逻辑格边界重叠，弱化“一个格子一棵圆树”的棋盘感。

### 3. Lived-in hamlet

村落不只是房屋图标。房屋周围先形成泥院，再布置木篱、破车、树桩和积水，让空间具备使用痕迹。

### 4. Gritty ground language

荒草、湿草、泥地、车辙路、积水与浅川使用相邻但可区分的低饱和土色 / 橄榄色。道路通过两道不完全平行的车辙和小型坑洼呈现，不依赖格线。

### 5. Environmental storytelling

倒木、残垣、荆丛、石块与路祠分布在道路和荒野之间，其中一部分参与碰撞或遮挡视野，让环境不仅是装饰。

### 6. Performance constraint

新增的地面纹理、树木、建筑和杂物全部预渲染进现有静态地图缓存。常态运行仍只做缓存位图、玩家、检查标记和低分辨率 fog texture 的合成，不恢复逐格每帧绘制。
