# MountainRoad

基于 Unreal Engine 5.8 的场景工程，包含 PCG、Houdini 数字资产（HDA）、材质与纹理资源。默认场景为 `Content/Map.umap`。

## 具体功能与资源入口

以下说明结合资源文件、PCG 资产中可读取的节点类型和引用整理。节点存在不代表所有分支都在当前场景中执行；参数数值、连线和最终效果仍以编辑器为准。

### 1. 山路程序化生成与总流程

核心入口为 `Content/PCG/PCG_MountRoad.uasset`，图中包含 Houdini PCG 节点，并引用道路表面、岩石、树木三个子图。还能识别到属性转换、属性排序、字符串处理、数据合并和静态网格生成节点，说明主图承担生成数据整理及子流程组织的职责。

Houdini 数字资产位于 `Content/HDAs/mount_road-1_0.uasset`。工程保存了多组 `main_geo` 和 `collision_geo` 输出，以及对应的 Bake 资源，可确认已有几何与碰撞几何的生成产物。

注意：主图中还可读到 `/Game/HDAs/mount_road` 引用，与当前磁盘上的 `mount_road-1_0` 名称不同。是否通过重定向或其他绑定解析，需要在编辑器中检查；不要仅凭文件名改动引用。

### 2. 基于样条的道路表面采样

入口：`Content/PCG/PCG_RoadSurface.uasset`。

图中包含获取样条（Get Spline Data）、从样条创建表面（Create Surface From Spline）、表面采样（Surface Sampler）、差集（Difference）和数据类型过滤节点。这些节点为道路相关区域提供表面和采样点数据，并支持区域差集处理。

修改时重点查看样条来源、表面创建方式、采样密度和差集输入。当前静态检查不能确定具体选中了哪些场景 Actor，也不能确认各样条必须采用的闭合方式。

### 3. 岩石程序化布置

入口：`Content/PCG/PCG_Rocks.uasset`。

已识别的处理能力包括：

- 获取地形数据并进行表面采样，建立候选点。
- 计算距离、执行密度过滤，并调用公共距离过滤子图。
- 使用属性噪声和点变换节点调整点数据。
- 使用 Self Pruning 节点进行点集自修剪，再通过 Static Mesh Spawner 生成网格实例。

调整岩石效果时，应检查生成器所用网格、采样设置、距离范围、点变换及修剪规则。节点的实际连接顺序与随机参数需要在图内确认。

### 4. 树木程序化布置

入口：`Content/PCG/PCG_Tree.uasset`。

图中同样包含地形读取、表面采样、距离计算、密度过滤、点变换、自修剪及静态网格生成节点，并引用 `PCG_DIstanceFilter`。这为按地形与距离条件筛选树木候选点、控制分布以及放置实例提供了基础。

树木与岩石拆分为独立子图，便于分别检查模型、采样密度、实例变换和分布范围。是否实现道路避让，以及避让对象与宽度，应检查 Distance 和 Difference 的实际输入。

### 5. 可复用距离过滤

入口：`Content/PCG/PCG_DIstanceFilter.uasset`，注意文件名中 `DIstance` 的大小写。

资产中可识别到 `MinDistance`、`MaxDistance`、`DistanceRatio` 等字段，以及用户参数读取、属性数学运算、属性过滤与属性噪声节点。树木与岩石图均引用该资产，因此它是两者共用的距离筛选逻辑入口。

若需要改变近处与远处的分布关系，优先检查此图及调用位置的参数覆盖。本文不提供未经确认的默认距离、单位或计算公式。

### 6. 道路与地形材质

| 资源 | 用途与检查入口 |
| --- | --- |
| `Materials/M_Road0`、`Materials/M_Road1` | 两份道路材质资源；具体差异需查看材质图 |
| `Materials/M_RoadEdge` | 道路边缘材质入口 |
| `Materials/M_RoadLine` | 道路线条材质入口 |
| `Materials/M_Lanscape`、根目录 `M_Lanscape` | 两份不同路径的地形材质资源，实际使用者需检查引用 |
| `Textures/Road/` | 两组沥青纹理及对应法线、粗糙度贴图 |
| `Textures/Edge/` | 路侧泥土纹理及对应法线、粗糙度贴图 |
| `Textures/road_white_line_2way_alpha` | 道路白线纹理资源 |
| `Textures/Sand/`、`Textures/Gravel/`、`Textures/Pebble/` | 沙地、碎石、卵石的基础颜色、法线和粗糙度贴图 |

工程还包含 `RVT_Road` 与 `RVT_LandscapeHeight` 两份运行时虚拟纹理相关资源。从命名看分别面向道路和地形高度；写入与采样关系、道路地形融合方式尚未确认，不能仅凭资源名称认定已经完成融合效果。

### 7. Houdini 输出与烘焙资源

- `Content/HoudiniEngine/Temp/` 保存多批 Houdini 生成资源。
- `Content/HoudiniEngine/Bake/` 保存烘焙后的几何、碰撞及部分材质资源。
- 重新生成 HDA 后，应核对道路外形、材质槽和碰撞结果，再决定是否更新烘焙产物。
- 不要把 `Content/HoudiniEngine/Temp` 当成普通工程缓存直接删除：场景或其他资产可能仍在引用其中的资源。

### 8. 房屋相关资源的当前状态

工程同时存在 `Content/PCG/PCG_House.uasset` 和 `Content/PCG_House.uasset`。其中前者可读取到对 `PCG_MountRoad` 的引用。目前证据不足以确认它实现了独立的房屋生成流程，需在编辑器检查资源类型和内容后再作功能承诺。

## 建议的编辑与验证流程

1. 打开 `Map`，确认地形、道路及已有生成物正常加载。
2. 在场景中找到实际使用的 PCG 组件，确认其 Graph 指向；从 `PCG_MountRoad` 查看主流程。
3. 检查 Houdini 节点绑定的 HDA 与输入，确认引用能够解析。
4. 修改道路输入后，先检查道路输出，再检查道路表面采样及树木、岩石子图。
5. 分别检查树木与岩石的网格选择、密度、距离条件及点变换，逐项调整并观察生成结果。
6. 检查道路材质、边缘和白线；如涉及 RVT，核对场景中的写入者与材质采样关系。
7. 需要固定生成结果时，检查或更新 Bake 资源；保存地图及相应的外部 Actor 数据。

建议验证道路连续性、道路与地形的高度关系、实例是否穿插路面、碰撞是否符合预期，以及远处实例的显示与性能。上述项目是验收建议，并非已经完成的测试结果。

## 当前功能边界

现有 C++ 主模块主要提供工程模块入口，程序化场景逻辑集中在资产中。本次检查未确认车辆驾驶、任务系统、交互界面、联网或运行时动态生成等玩法功能，也未执行编辑器运行与打包测试。不要将默认输入配置中的设备条目视为已经实现对应玩法或平台支持。

## 环境要求

- Unreal Engine **5.8**（以 `MountainRoad.uproject` 的引擎关联为准）。
- 编译 C++ 模块时，需要与该引擎版本匹配的 Visual Studio C++ 工具链及 Windows SDK。
- 当前 Windows 渲染配置使用 **DirectX 12 / Shader Model 6**，面向桌面高画质环境。
- 工程内附 Houdini Engine 插件，版本标识为 **3.0 - H22.0.368**；需要编辑或重新生成 HDA 时，请准备匹配的 Houdini 环境及适用许可。

## 打开项目

1. 安装并配置 Unreal Engine 5.8。
2. 打开 `MountainRoad.uproject`。如提示模块缺失或版本不匹配，先生成项目文件，再编译编辑器目标。
3. 等待首次资源处理和着色器编译完成。
4. 编辑器默认打开 `/Game/Map`；也可以在内容浏览器中手动打开 `Content/Map.umap`。

### 编译 C++ 模块

1. 右键 `MountainRoad.uproject`，生成 Visual Studio 项目文件。
2. 打开生成的 `MountainRoad.sln`。
3. 选择 `Development Editor` 和 `Win64`，构建 `MountainRoad` 项目（编辑器目标为 `MountainRoadEditor`）。
4. 构建完成后重新打开工程。

## 目录结构

```text
MountainRoad/
├── MountainRoad.uproject       # Unreal 工程入口
├── Config/                     # 引擎、输入及项目配置
├── Content/
│   ├── Map.umap                # 默认编辑器场景与游戏地图
│   ├── HDAs/                   # Houdini 数字资产
│   ├── HoudiniEngine/           # Houdini 相关资源
│   ├── PCG/                    # 程序化内容生成资源
│   ├── Materials/              # 材质资源
│   ├── Textures/               # 纹理资源
│   ├── __ExternalActors__/     # 地图外部 Actor 数据
│   └── __ExternalObjects__/    # 地图外部对象数据
├── Plugins/Runtime/HoudiniEngine/
│                               # 项目内 Houdini Engine 插件
└── Source/
    ├── MountainRoad/           # 主运行时 C++ 模块
    ├── MountainRoad.Target.cs
    └── MountainRoadEditor.Target.cs
```

## 插件与渲染配置

工程显式启用了 `ModelingToolsEditorMode`（仅编辑器）和 `PCGGeometryScriptInterop`。项目内的 Houdini Engine 插件配置为默认启用。

当前渲染配置包括 Lumen 动态全局光照与反射、虚拟阴影贴图、网格距离场、光线追踪及 Substrate；静态光照已禁用。相关设置位于 `Config/DefaultEngine.ini`，可通过编辑器的项目设置调整。

## 协作与资源维护

- 保留 `Content`、`Config`、`Source`、所需插件及 `.uproject` 文件。
- 移动或重命名 Unreal 资源时，优先在编辑器的内容浏览器中操作，避免破坏资源引用。
- 地图的外部 Actor 和对象数据需要随地图一起同步。
- `Intermediate`、`DerivedDataCache`、`Saved` 和 `.vs` 属于本机生成或缓存数据；`Saved` 也可能包含自动保存和备份，清理前应检查。
- 本文依据工程文件和配置整理；尚未验证编辑器启动、运行或打包结果。

