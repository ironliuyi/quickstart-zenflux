# 小搭子（xiaodazi）服务实例部署文档

## 概述

小搭子（xiaodazi）是一款开源桌面 AI 智能体，基于 FastAPI + Vue 3 构建。它运行在您的本地机器上，支持多模型切换（Claude、GPT、通义千问、DeepSeek、Gemini、GLM 等），具备 150+ 即插即用技能，可操控桌面应用、管理文件、生成文档，并在会话之间记住您的偏好。

本文向您介绍如何通过计算巢快速部署小搭子服务实例，以及部署流程和使用说明。

## 计费说明

小搭子在计算巢上的费用主要涉及：

- 所选 vCPU 与内存规格
- 系统盘类型及容量
- 数据盘类型及容量
- 公网带宽

计费方式包括：

- 按量付费（小时）
- 包年包月

目前提供如下实例规格：

| 规格族 | vCPU 与内存 | 系统盘 | 数据盘 | 公网带宽 |
| --- | --- | --- | --- | --- |
| ecs.u1 系列 | 经济型，推荐 4vCPU 8GiB 及以上 | ESSD 云盘 200GiB | ESSD 云盘 200GiB | 固定带宽 5Mbps |
| ecs.e 系列 | 共享型，推荐 4vCPU 8GiB 及以上 | ESSD 云盘 200GiB | ESSD 云盘 200GiB | 固定带宽 5Mbps |

> **说明**：预估费用在创建实例时可实时看到。此外，小搭子运行时需要调用大模型 API（如通义千问、Claude、OpenAI 等），API 调用费用由各模型厂商收取，不包含在计算巢实例费用中。
>
> 如需更多规格或企业级支持服务，请联系我们：[liuyi@zenflux.cn](mailto:liuyi@zenflux.cn)。

## 部署架构

小搭子采用前后端分离的 Docker 容器化部署方案，通过 Docker Compose 编排两个服务：

```
                        ┌─────────────────────────────────────────┐
                        │           阿里云 ECS 实例                │
                        │                                         │
  用户浏览器 ──────────▶│  ┌─────────────────────────────────┐    │
        (HTTP:80)       │  │  Frontend 容器 (Nginx)           │    │
                        │  │  - 静态资源服务 (Vue 3 SPA)      │    │
                        │  │  - 反向代理 /api/* 请求           │    │
                        │  └──────────┬──────────────────────┘    │
                        │             │ proxy_pass :8000           │
                        │  ┌──────────▼──────────────────────┐    │
                        │  │  Backend 容器 (FastAPI)          │    │
                        │  │  - REST API + SSE 流式响应       │    │
                        │  │  - WebSocket 双向通信             │    │
                        │  │  - Agent 引擎 + 150+ 技能         │    │
                        │  └──────────┬──────────────────────┘    │
                        │             │                            │
                        │  ┌──────────▼──────────────────────┐    │
                        │  │  持久化存储 (Docker Volumes)      │    │
                        │  │  - zenflux-data: SQLite 数据库    │    │
                        │  │  - zenflux-workspace: 工作文件    │    │
                        │  └─────────────────────────────────┘    │
                        └─────────────────────────────────────────┘
```

- **Frontend 容器**：Nginx 提供 Vue 3 单页应用的静态资源服务，并将 `/api/*` 请求反向代理到后端
- **Backend 容器**：FastAPI 提供 RESTful API、SSE 流式推送和 WebSocket 通信，内含 Agent 执行引擎
- **持久化存储**：通过 Docker Volumes 持久化 SQLite 数据库、用户记忆、对话历史和文件上传

## RAM 账号所需权限

小搭子服务需要对 ECS、VPC 等资源进行访问和创建操作。若您使用 RAM 用户创建服务实例，需要在创建服务实例前，对使用的 RAM 用户账号添加相应资源的权限。添加 RAM 权限的详细操作，请参见 [为 RAM 用户授权](https://help.aliyun.com/document_detail/121945.html)。所需权限如下表所示：

| 权限策略名称 | 备注 |
| --- | --- |
| AliyunECSFullAccess | 管理云服务器服务（ECS）的权限 |
| AliyunVPCFullAccess | 管理专有网络（VPC）的权限 |

## 部署流程

### 部署步骤

1. 单击 [部署链接]()，进入服务实例部署界面。
2. 根据界面提示，选择地域、配置 VPC 网络、选择 ECS 实例规格。
3. 填写实例密码和大模型 API Key（至少填写通义千问 DASHSCOPE_API_KEY）。
4. 确认配置信息和费用，单击 **确认订单并创建**。
5. 等待服务实例创建完成，部署时间大约需要 3-5 分钟。

### 部署参数说明

您在创建服务实例的过程中，需要配置服务实例信息。下文介绍小搭子服务实例输入参数的详细信息。

| 参数组 | 参数项 | 示例 | 说明 |
| --- | --- | --- | --- |
| 基础配置 | 付费类型 | 按量付费 | 支持按量付费和包年包月 |
| 基础配置 | 地域 | 华东1（杭州） | 选中服务实例的地域，建议就近选中以获取更好的网络延时 |
| 基础配置 | 可用区 | 杭州可用区H | 选择 ECS 实例所在的可用区 |
| 网络配置 | VPC 选项 | 新建专有网络 | 可选择新建 VPC 或使用已有 VPC |
| 网络配置 | VPC CIDR | 192.168.0.0/16 | 新建 VPC 时的 IP 地址段 |
| 网络配置 | 交换机子网 | 192.168.1.0/24 | 新建交换机时的子网段 |
| ECS 配置 | 实例类型 | ecs.u1-c1m2.xlarge | 推荐 4vCPU 8GiB 及以上规格 |
| ECS 配置 | 实例密码 | ******** | 长度 8-30，需包含大写字母、小写字母和数字 |
| 大模型配置 | DASHSCOPE_API_KEY | sk-xxx | 通义千问 API Key（必填），从 [DashScope 控制台](https://dashscope.console.aliyun.com/) 获取 |

> **说明**：DASHSCOPE_API_KEY 为必填参数，这是小搭子默认使用的大模型服务。部署完成后，您可以在前端设置页面中配置更多大模型提供商（Claude、OpenAI、DeepSeek、Gemini 等）。

### 验证结果

1. **查看服务实例**：服务实例创建成功后，部署时间大约需要 3-5 分钟。部署完成后，在计算巢控制台页面上可以看到对应的服务实例状态为「已部署」。

2. **访问小搭子**：进入对应的服务实例详情后，获取实例的公网 IP 地址，在浏览器中访问：

    ```
    http://<ECS公网IP>
    ```

3. **健康检查**：可通过以下命令验证服务是否正常运行：

    ```bash
    curl http://<ECS公网IP>/health
    ```

    正常返回：

    ```json
    {"status": "ok", "version": "0.1.0"}
    ```

### 使用小搭子

1. **开始对话**：在浏览器中访问 `http://<ECS公网IP>`，即可打开小搭子的 Web 聊天界面，直接输入消息开始对话。

2. **配置更多大模型**：在前端界面的「设置」页面中，可以添加更多大模型提供商的 API Key：
    - **通义千问**（默认）：已在部署时配置
    - **Claude**：填入 Anthropic API Key
    - **OpenAI**：填入 OpenAI API Key
    - **DeepSeek**：填入 DeepSeek API Key
    - **Gemini**：填入 Google API Key（免费额度：1500 请求/天）
    - **智谱 GLM**：填入智谱 API Key

3. **探索技能**：小搭子内置 150+ 技能，涵盖文件管理、网页搜索、文档生成、数据分析等。直接用自然语言描述您的需求，智能体会自动选择合适的技能执行。

4. **API 文档**：访问 `http://<ECS公网IP>/docs` 可查看完整的后端 API 文档（Swagger UI）。

请访问小搭子项目主页了解更多使用方法：[GitHub 仓库](https://github.com/malue-ai/dazee-small)

## 问题排查

| 问题现象 | 可能原因 | 解决方案 |
| --- | --- | --- |
| 访问 `http://<IP>` 无响应 | 安全组未开放 80 端口 | 检查 ECS 安全组规则，确保 80 端口对 `0.0.0.0/0` 开放 |
| 页面空白或 502 错误 | 后端容器未就绪 | SSH 登录 ECS，运行 `docker compose ps` 检查容器状态；等待健康检查通过（约 60 秒） |
| 对话无响应或报错 | API Key 未配置或无效 | 检查 DASHSCOPE_API_KEY 是否正确；可 SSH 登录后运行 `docker compose logs backend` 查看日志 |
| 容器频繁重启 | 内存不足 | 建议升级到 4vCPU 8GiB 及以上规格 |
| 数据丢失 | Docker Volume 未正确挂载 | SSH 登录后运行 `docker volume ls` 确认 `zenflux-data` 和 `zenflux-workspace` 存在 |

如遇到其他问题，请通过以下方式获取帮助：

- 提交 Issue：[https://github.com/malue-ai/dazee-small/issues](https://github.com/malue-ai/dazee-small/issues)
- 联系技术支持：[liuyi@zenflux.cn](mailto:liuyi@zenflux.cn)

## 联系我们

欢迎访问小搭子项目主页了解更多信息：[https://github.com/malue-ai/dazee-small](https://github.com/malue-ai/dazee-small)

联系邮箱：[liuyi@zenflux.cn](mailto:liuyi@zenflux.cn)

开源地址：[https://github.com/malue-ai/dazee-small](https://github.com/malue-ai/dazee-small)
