# Fork & 安全 Agent 工作流（中文说明）

本仓库是所有 genCodeDesc 分支的**基础模板**。为确保所有代码代理（Copilot、LLM 等）开发都在安全、可复现、隔离的环境中进行，请遵循以下流程：

## 1. Fork 本仓库

- 创建你自己的分支（如 `MyGenCodeDesc_Copilot_GPT-5.4-Xhigh_Python`）。

## 2. 用 VS Code 打开

- 将你的分支克隆到本地。
- 用 VS Code 打开该文件夹。

## 3. 在 Dev Container 中重启

- 如果 VS Code 提示，点击“Reopen in Container”（需先启动 Docker Desktop）。
- 或者：按下 Command Palette（⇧⌘P）→ 输入 “Dev Containers: Reopen in Container”。

## 4. 安全开发

- 所有代码代理的工作、构建、测试都**在容器内**完成。
- 容器默认使用非 root 用户，且不会默认挂载主机密钥或 Docker socket。
- 你可以在容器内安装 Python、C++、Rust 等开发工具。

## 5. 提交与推送

- 所有更改都在容器内提交。
- 推送到你的分支并正常发起 PR。

## 环境要求

- 主机需安装并运行 **Docker Desktop**（支持 macOS、Windows、Linux）。
- VS Code 需安装 "Dev Containers" 扩展。

## 为什么这样做？

- 保持主机系统干净、安全。
- 确保所有分支有一致、可复现的开发环境。
- 让代码代理可以安全地实验，无风险污染主机。

---

更多信息请参见 [README.md](README.md)。
