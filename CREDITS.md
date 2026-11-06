# 来源与致谢

本仓库中 `ros2_arm_ws/` 与 `el05_serial_ws/` 两个工作区的原始实现由 **Raki（秦均灿）** 编写。

| 项 | 内容 |
|---|---|
| 原作者 | Raki（亲均灿） |
| 联系 | 3433253304@qq.com |
| 原始仓库 | https://github.com/Rakiv-dsh/arm-urdf |
| 并入版本 | 提交 `2026-11-02 21:49` |
| 并入日期 | 2026-11-06 |
| 整理 | SZTU |

## 并入方式

| 原路径 | 现路径 | 说明 |
|---|---|---|
| `arm_ws/` | `ros2_arm_ws/` | 目录改名，内容为本仓库当前实现 |
| `el05_ws/` | `el05_serial_ws/` | 目录改名，内容为本仓库当前实现 |
| `integrations/` | 未覆盖 | 两仓库此目录内容原本一致，保留本仓库版本 |

原仓库的 `README.md` 未并入，其内容已折算进本仓库根目录的 `README.md`。

## 回退

合并前的状态保存在标签 `pre-rakiv-merge`：

```bash
git checkout pre-rakiv-merge   # 查看合并前内容
git diff pre-rakiv-merge       # 查看全部差异
```

## 许可说明

原仓库未附 `LICENSE` 文件，因此这里的代码按默认版权（全部权利保留）处理，仅在本组内部使用与修改。**若要公开发布或对外分发，需先取得原作者同意。**

## 改动记录

本仓库对并入内容做过以下改动（如后续有更多改动，逐条追加）：

- 目录改名：`arm_ws` → `ros2_arm_ws`，`el05_ws` → `el05_serial_ws`
- 未改动任何源码、URDF、mesh 与配置文件的内容
