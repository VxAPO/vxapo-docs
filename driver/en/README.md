# VxAPO Driver Documentation

This directory covers the architecture, module specification, configuration/DSP design, and installation/troubleshooting for `vxapo-driver` (Windows APO DLL).

## Document map

| File | Content |
|------|---------|
| `Architecture and Module Specification.md` | module tree, dependency rules, data flow, thread safety, key design decisions |
| `Configuration and DSP Design.md` | config.toml model, effect types, hybrid PEQ, DSP numerical safety |
| `Installation and Troubleshooting.md` | install/uninstall flow, slots and child APO, snapshot, audiodg root cause |
| `Code Review Summary.md` | consolidated historical code-review findings |
| `模块引用规范/` | detailed module reference specifications (developer guide, per module) |

> The authoritative source is `VxAPO/vxapo-driver/src`; this documentation is a structured description of the current implementation.
