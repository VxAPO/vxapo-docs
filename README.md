# VxAPO Documentation

VxAPO 的项目文档仓库，按主题维护 App / CLI / Driver 三层的引用规范、UI 设计规范
与 DSP 配置设计。文档以**源码实读**为基准，修订时与对应仓库代码对齐。

## 结构

- `app/` — App 引用规范 + UI 设计规范（主题令牌/窗口导航/视图卡片/曲线/弹窗/动效）
- `cli/` — CLI 引用规范（安装/卸载/配置/快照/验证）
- `driver/` — Driver 架构、模块规范、配置/DSP 设计、安装排障
- `overview/` — 项目概览（三层分离与系统组成）

每个主题目录下均包含 `zh/` 与 `en/` 版本。

## 分层对齐

App（决策与写配置）→ CLI（安装/卸载/注册表唯一入口）→ Driver（`audiodg` 内只读配置的
实时 DSP）。三份引用规范共用同一 config 契约（`[[effects]]` TOML 模型）：
参数键、数值边界（增益 `[-120,+48]`、31 段上限）、旧类型兼容映射
（maximizer/leveler → compressor 等）、热重载 spec 指纹语义完全一致。

## 设计参考与致谢

VxAPO 的安装模型、配置文件驱动 DSP 与热重载思路参考了
[Equalizer APO](https://sourceforge.net/projects/equalizerapo/)（© Jonas Thedering，
GPL-2.0）；本项目为独立实现，不包含其代码。

## 许可证

GPL-3.0-or-later

---

# VxAPO Documentation

This repository holds the VxAPO project documentation, organized by topic: reference
specifications for the App / CLI / Driver layers, the UI design specification, and DSP
configuration design. Documents are based on direct source reads and stay aligned with
the corresponding repositories.

## Structure

- `app/` — App reference + UI design (theme tokens / shell & navigation / views & cards /
  curve / dialogs / motion)
- `cli/` — CLI reference (install / uninstall / config / snapshot / verify)
- `driver/` — Driver architecture, module specification, configuration/DSP design,
  installation & troubleshooting
- `overview/` — Project overview (three-layer separation)

Each topic directory contains `zh/` and `en/` versions.

## Layer alignment

App (decisions & config writes) → CLI (the only entry point for install/uninstall and
registry writes) → Driver (read-only real-time DSP inside `audiodg`). The three
reference specifications share one config contract (the `[[effects]]` TOML model):
parameter keys, numerical bounds (gain `[-120,+48]`, 31-band limit), legacy type
mapping (`maximizer`/`leveler` → `compressor`, etc.), and hot-reload spec-fingerprint
semantics are identical across layers.

## Design references & acknowledgments

VxAPO's install model, config-file-driven DSP, and hot-reload approach are inspired by
[Equalizer APO](https://sourceforge.net/projects/equalizerapo/) (© Jonas Thedering,
GPL-2.0); this project is an independent implementation with no Equalizer APO code.

## License

GPL-3.0-or-later
