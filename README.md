# 实践课1 Pi Agent 安装与使用

## 1 安装和启动（S1）

在 Windows PowerShell 中使用指导书里的官方安装命令安装 Pi。安装器按提示配置了 Node.js、便携版 Git Bash 和用户 PATH。重新打开终端，在练习目录中运行 `pi`，界面显示版本 **v1.5.0** 和工作目录 `~\pi-test`。后续练习移到了 `D:\开源软件构建\pi-test`。

![S1 Pi 启动界面](截图/S1-安装启动.png)

## 2 配置模型并完成首次会话（S2）

在 Pi 中通过 `/login` 选择 API Key 登录方式，配置 DeepSeek。第一次发送「你好」时收到 `402 Insufficient Balance`。随后再次发送「你好」，`deepseek-v4-pro` 回复「你好！有什么可以帮你的吗？」。

![S2 提问和模型回复](截图/S2-首次会话.png)

## 3 切换模型和思考级别（S3）

| 项目 | 调整前 | 调整后 |
| --- | --- | --- |
| 模型 | `deepseek-v4-pro` | `deepseek-flash` |
| 思考级别（在 Flash 上调整） | `off` | `low` |

使用 `/model` 切换模型后，发送「请用一句话解释持续集成。」，Flash 正常给出解释。再使用 `/thinking`，将 Flash 的思考级别从 `off` 调到 `low`。这次的一句话解释在 `off` 下已经足够。遇到需要多步分析的任务，例如排查多个原因时，可以从 `low` 开始，根据结果再调整。

切换前的模型名称见 S2。

![S3 切换到 Flash 后的会话与 off 状态](截图/S3-切换模型.png)

![S3 调整到 low 后的状态](截图/S3-调整思考级别.png)

## 4 引用代码与粘贴图片（S4–S5）

### 引用代码（S4）

在练习目录保存 `hello.py`。Pi 启动时曾提示 `fd` 下载超时，补齐 `fd` 并重启后，在输入框键入 `@hello`，按 `Tab` 补全为 `@hello.py`，再请 Pi 解释文件并说明运行结果。Pi 读取了文件，解释了 `add(a, b)` 返回两个参数的和，并正确给出 `print(add(2, 3))` 的输出 `5`。

![S4 引用 hello.py 及模型回复](截图/S4-引用代码.png)

### 粘贴图片（S5）

切换到支持图片输入的 `deepseek-flash`。使用 Windows 截图工具将小图截入剪贴板，在 Pi 中按 `Alt+V`，输入框出现 Pi 保存的临时 PNG 路径。发送「请描述图片中的颜色、形状和数字。」后，Pi 读取图片并识别出白色背景、蓝色正方形和黑色数字 **7**。

![S5 测试原图](截图/S5-测试原图.png)

![S5 图片读取记录与识别结果](截图/S5-图片识别.png)

## 5 安装和试用插件（S6）

从 Pi Packages 选择 `pi-web-access`，它提供网页获取等功能。使用 `pi install npm:pi-web-access -l` 安装到当前项目，退出后单独运行 `pi`。信任练习目录后，启动界面的 `[Extensions]` 区域显示 `pi-web-access:dist`。本次测试的网页获取功能没有额外配置插件密钥。 [插件来源](https://pi.dev/packages/pi-web-access)

让 Pi 用 `fetch_content` 获取 `https://example.com` 的网页标题。第一次提取提示内容可能不完整，随后用 `raw` 模式成功获取 HTML，并从中读出标题 **Example Domain**。

![S6 插件安装成功](截图/S6-安装.png)

![S6 重启后加载插件](截图/S6-加载.png)

![S6 调用插件并取得网页标题](截图/S6-调用.png)

PPT Skill 为选做，本次未做。

[课程作业要求](https://github.com/hunnuOpenSourceSoftware2026/assignments/blob/main/practice/01-Pi-Agent-start.md)
\n