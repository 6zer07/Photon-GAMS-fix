# 版本历史索引

## 编号的两次重启

| 阶段 | 时间 | 编号 | 结局 |
| --- | --- | --- | --- |
| v 系列 | …–2026-08-20 | v1–v12（本地只剩 `Photon-GAMS-main-complementary-puddles-v12.zip` 这个名字） | 某 bug 修不好 → 放弃 |
| t 系列第一轮 | 2026-08-21 – 2026-09-12 | t1 – t20（t5 / t9 / t12 / t12c / t20 有包） | t12 → t20 围绕“模组覆盖层发黑”失败 → 放弃 |
| t 系列第二轮 | 2026-10-17 – | t13 / t14 / t16（t15 作废） | 当前主干 |

判定依据：`-t5.zip` 相对上游 `main` **只改 11 个文件、零新增零删除**，且不含 v12 的
`complementary_noise.png` 与 `c1/c15/material` 改动 → t 系列是回到干净 `main` 重新起线；
`t20` 相对 `t12` **新增 25 个 `main` 里不存在的程序文件**并附 `overlay_debug.glsl` → 第一轮的最后一次尝试。

## 逐版本

| 版本 | 日期 | 内容 | 日志 |
| --- | --- | --- | --- |
| t5 | 9/4 | 粒子接回光影粒子通道；修 Ars Nouveau 仪式粒子与半透明实体 | — |
| t9 | 9/5 | 雨花过滤改为“下雨 + 露天”，保住水龙头水滴 | — |
| t12 | 9/10 打包 / 10-02 提交 | 预乘 alpha、粒子走半透明层 + 着色器内遮挡、材质 27/47、蓝色粒子过滤、参数化 lightmaps、新选项 | [t12](logs/t12.md) |
| t12c / t20 | 9/12 | t12 的两个分支尝试（t20 未采用） | [t20](logs/t20.md) |
| t13 | 10/17 | 粒子遮挡判据、自发光不换色、火焰实体判据、透明度语义 | [t13](logs/t13.md) |
| t14 | 10/17 | 半透明表面恢复雾淡出 | [t14](logs/t14.md) |
| t15 | 10/19 | ❌ 作废（改回 colortex0 导致裂纹不显示） | [t15](logs/t15.md) |
| t16 | 10/19 | 破坏覆盖层不再印到手臂 | [t16](logs/t16.md) |

## 其它记录

* 回归自测清单：[logs/regression-checklist.md](logs/regression-checklist.md)
* t13 那次上传的说明：[logs/t13-upload-note.md](logs/t13-upload-note.md)
* 上游血缘：sixthsurge/photon → sw-52/gluon → OUdefie17/Photon-GAMS（`main` 停在 `5fa0eff`，2025-01-22）
