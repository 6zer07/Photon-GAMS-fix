<br>

<h1 align = "center">Photon GAMS — fixed (t12) · 中文说明</h1>

<p align = "center">基于 <a href="https://github.com/OUdefie17/Photon-GAMS">OUdefie17/Photon-GAMS</a> 的社区修复分支</p>

English: [README.md](README.md)

## 这个仓库是什么

这是 **社区修复分支（fork）**。上游
[OUdefie17/Photon-GAMS](https://github.com/OUdefie17/Photon-GAMS) 最后一次推送停留在
**2025-03-30**，本分支在其基础上继续维护，`main` 分支携带 **t12** 修复集：共 **20 个着色器文件**
被修改，逐条原因见 [CHANGELOG.md](CHANGELOG.md)。

除 `README.md`、`README.zh_CN.md`、`CHANGELOG.md`、`docs/t12-code-changes.patch` 四个新增/文档文件外，
仓库内容与实测通过的光影包 **t12 逐字节一致**。

* 逐文件改动记录：[CHANGELOG.md](CHANGELOG.md)
* 相对上游 `main` 的原始补丁：[docs/t12-code-changes.patch](docs/t12-code-changes.patch)

## t12 修了什么

### 1. 1.21 版透明贴图与粒子不显示（预乘 alpha）

半透明程序以前输出的是**非预乘**颜色，而管线用的是 `ONE / ONE_MINUS_SRC_ALPHA` 混合，于是使用预乘
透明度的版本里这些图层会偏暗、偏亮甚至看不见。现在 `gbuffers_textured`、`gbuffers_particles` 以及
粒子/实体/方块的半透明程序统一输出**预乘 alpha**（`PREMULTIPLIED_ALPHA`），并删掉了与新混合模式
互相打架的旧代码（`a = sqrt(a)` 与 `rgb / max(a, eps)`），混合函数一并改成
`ONE / ONE_MINUS_SRC_ALPHA`，行为与 Photon 1.3 对齐。`gbuffers_weather` 的雨雪同样预乘。

### 2. 粒子深度与粒子发光

* 粒子改由**写入半透明层（`colortex13`）**的程序绘制（`world*/gbuffers_particles.{vsh,fsh}`），不再
  作为不透明延迟几何体——后者会丢掉混合，这正是 Ars Nouveau 等模组的仪式粒子表现异常的原因。
* 该半透明层在合成时**不做深度测试**，因此被实体/地形挡住的粒子必须在着色器里剔除：新增开关
  **`PARTICLE_OCCLUSION`**（默认开），用 `if (depth1 < gl_FragCoord.z - 1e-5) discard;` 处理。
* 粒子不再穿透手持物品：`gbuffers_hand` 在自己通过深度测试的像素上把 `colortex13` 清 0，抹掉手后面
  的半透明层；手持物品背后的水也一并修好。
* 粒子在半透明顶点阶段获得了正确的材质编号——`27`（普通粒子）、`47`（发光粒子）——**发光粒子终于会真正
  发光**，而不是被当成未分类材质着色。
* `get_directional_lightmaps()` 改为显式接收场景坐标参数，不再依赖全局 `scene_pos`：在半透明/粒子程序
  里那个全局量是错的，会算出错误的屏幕空间导数，进而导致方向光贴图（发光/明暗）出错。

### 3. 模组兼容：不再一刀切删掉所有蓝色粒子

上游硬编码了这段：

```glsl
// Kill the little rain splash particles
if (base_color.r < 0.29 && base_color.g < 0.45 && base_color.b > 0.75) discard;
```

它会删掉游戏里**所有偏蓝的粒子**，包括森罗物语酒馆（Kaleidoscope Tavern）水龙头/餐具的粒子以及各类
方块动画模型，导致这些动画缺粒子。现在这条规则只在**确实在下雨且露天**时才生效：

```glsl
if (rainStrength > 0.05 && light_levels.y > 0.1
    && base_color.r < 0.29 && base_color.g < 0.45 && base_color.b > 0.75) discard;
```

地面雨花依旧被去掉，室内和模组的蓝色粒子保留。

### 4. 新增开关

| 开关 | 默认 | 作用 |
| --- | --- | --- |
| `USE_SEPARATE_ENTITY_DRAWS` | 关 | 暴露 Iris 的 `separateEntityDraws`。需要 **Iris 26.1+**，在旧版本上启用会使半透明实体与方块渲染错误。 |
| `DITHERED_TRANSLUCENCY_FALLBACK` | 开 | 在无法使用/未启用分离绘制实体时，用抖动透明度改善本应半透明物体的外观。 |
| `PARTICLE_OCCLUSION` | 开 | 剔除被实体/地形挡住的粒子；**关**掉才能保留模组刻意关闭深度测试的穿透效果。 |
| `PUDDLE_MODE` | 全覆盖 | `斑块水坑` 为原来的随机斑块，`全覆盖水坑` 在湿润时整片地面湿透。 |
| `TRANSLUCENT_ALPHA` | 0.75 | 半透明表面（模组翅膀、类玻璃面片等）的不透明度倍率；对闪电（材质 102）和地狱门（62）不生效。 |

### 5. 其它改动

* 水坑：`f0` 由 0.02 提到 0.2，粗糙度重做，**排除树叶**（材质 5），法线平坦判定更严，室内判定由
  `pow5(skylight)` 改成 14/15 处的线性过渡，屋檐下水坑不再出现。
* alpha 测试改用 uniform `alphaTestRef`，不再硬编码 `0.1`。
* 伤害叠加层 / 附魔光泽 / 延迟清屏的 `IS_IRIS` 分支改为 `USE_SEPARATE_ENTITY_DRAWS` 分支，表达“实际
  启用的功能”而不是“运行在哪个加载器上”。
* `shaders.properties` 里固定 `particles.ordering = mixed`，并把整段 `alphaTest.*` 注释掉（用默认值），
  不再给所有程序强制 `off`。

## 已知限制

* 雨花过滤只能靠颜色与环境判断（着色器看不到粒子类型）：**装在室外且正在下雨**时的水龙头水滴仍会被去掉。
* `PARTICLE_OCCLUSION = ON` 会隐藏模组刻意做的“穿透一切”效果（如 Ars Nouveau 仪式光带），需要时设为 `OFF`。
* 关闭分离绘制实体可能让半透明物体变得不透明（旗帜变白、篝火上的肉不渲染），这是该开关自身的取舍。

## 安装

1. 下载本仓库（Code → Download ZIP）或直接 clone。
2. 把文件夹（或它的压缩包）放进 `.minecraft/shaderpacks/`。
3. 在**视频设置 → 光影**中选择它（需要 Iris）。改完选项按 `R` 重载。

本分支的预乘 alpha 路径正是让 1.21+ 上透明贴图与粒子正常工作的关键；在 1.20.1 上观感与上游一致。

## 致谢与许可

上游作者与贡献者：OUdefie17、Arona74、-Daytendo64-、sw-52，以及原始 Photon 作者 Sixthsurge。

[LICENSE](LICENSE) 为 Photon 原作者 SixthSurge 的自定义许可，本仓库**原样保留**：允许修改与再分发，
但**不得**在会给创作者带来经济收益的分享平台（Modrinth、CurseForge 等）发布本体或衍生作品，也不得售卖。
