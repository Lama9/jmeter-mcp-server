# JMeter MCP Server Windows 部署指南

本指南记录了在 Windows 环境下可靠运行 JMeter MCP Server 所需的关键配置步骤。

## 1. 前置条件

* **Python 3.10+**: 确保已安装 Python 并添加到系统 PATH。
* **Java JDK**: JMeter 需要 Java 环境。安装 JDK 8 或更高版本（推荐 JDK 17）。
* **Apache JMeter**: 解压二进制发行包到某个目录（例如 `C:\apache-jmeter-5.6.3`）。

## 2. 环境变量配置 (`.env`)

在服务器根目录下创建一个 `.env` 文件。关键点：在 Windows 上，**JMETER_BIN 必须指向必须要新建的 wrapper 脚本**，而不是目录或默认的 `.bat` 文件。

```ini
JMETER_HOME=C:/path/to/apache-jmeter-5.6.3
# 关键：使用我们自定义的 'no_pause' wrapper 脚本
JMETER_BIN=C:/path/to/apache-jmeter-5.6.3/bin/jmeter_no_pause.bat
JAVA_HOME=C:/Program Files/Java/jdk-17
JMETER_JAVA_OPTS="-Xms1g -Xmx2g"

# 可选：JTL 和报告文件的默认输出目录
# 支持绝对路径 (D:/Results) 或相对路径 (./results)
# 相对路径基于脚本目录解析，目录不存在时会自动创建
JMETER_OUTPUT_DIR=./JMeterResults
```

## 3. "卡死"问题与 Wrapper 脚本

默认的 `jmeter.bat` 在执行结束或出错时包含 `pause` 命令。当通过 Python `subprocess` 调用时，这会导致进程**无限期挂起**，因为它在等待一个永远无法传达的按键输入。

**解决方案：** 用 `jmeter_no_pause.bat` 替代 `jmeter.bat`。

复制一份 `jmeter.bat` 并重命名为 `C:/path/to/apache-jmeter-5.6.3/bin/jmeter_no_pause.bat`，然后修改：

1. 将所有 `goto pause` 替换为 `goto end`。
2. 注释掉 `:pause` 标签下的 `pause` 命令。

## 4. `jmeter_server.py` 代码补丁

如果你将此代码部署到新机器，请确保 `jmeter_server.py` 中的 `run_jmeter` 函数包含以下针对 Windows 的修复：

### A. 修复 `JAVA_HOME` 可见性

MCP Server 通常运行在受限环境中（特别是通过 IDE 或 Agent 启动时），这些环境**不会继承**系统用户的环境变量（如 `JAVA_HOME`）。

你需要显式地将 `JAVA_HOME` 注入到子进程的环境中：

```python
# 强制设置 JAVA_HOME 和 PATH，确保子进程能找到 Java
env = os.environ.copy()
# 从环境（.env）中读取 JAVA_HOME
java_home = os.getenv('JAVA_HOME')
if java_home:
    env['JAVA_HOME'] = java_home
    env['PATH'] = f"{java_home}\\bin;{env.get('PATH', '')}"
```

**注意**：你需要在 `.env` 文件中定义 `JAVA_HOME`。

### B. 修复 `JMETER_BIN` 路径冲突

`.env` 中的变量 `JMETER_BIN` 告诉 Python **运行哪个文件**，但如果这个变量被传递给 JMeter 的批处理脚本，会覆盖批处理文件内部的 `JMETER_BIN` 变量（批处理脚本期望它是一个**目录**）。这会导致 `Unable to access jarfile` 错误。

**你必须在调用 subprocess 之前从 env 中移除 `JMETER_BIN`：**

```python
if 'JMETER_BIN' in env:
    del env['JMETER_BIN'] # 防止与批处理文件内部变量冲突
```

### C. 使用 `shell=True` 和 `stdin=DEVNULL`

为了正确执行批处理文件并防止输入阻塞：

```python
subprocess.run(
    cmd, 
    capture_output=True, 
    text=True, 
    stdin=subprocess.DEVNULL,  # 防止在输入提示处卡死
    env=env,                   # 传递打过补丁的 env
    shell=True                 # 执行 .bat 文件所必需
)
```

## 5. 输出目录配置 (`JMETER_OUTPUT_DIR`)

v1.1.0 新增功能：可通过 `.env` 配置统一的输出目录。

**特性：**
* 支持**绝对路径**（如 `D:/JMeterResults`）和**相对路径**（如 `./results`）
* 相对路径基于**脚本目录**解析，而非当前工作目录 (CWD)
* 目录不存在时会**自动创建**

**使用说明：**

1. 在 `.env` 中添加：`JMETER_OUTPUT_DIR=./JMeterResults`
2. 重启 MCP Server
3. 调用时只需传文件名（如 `test.jtl`），系统会自动保存到配置的目录

**注意**：如果调用时传入的是**绝对路径**（如 `D:/test.jtl`），则会直接使用该路径，忽略 `JMETER_OUTPUT_DIR` 配置。

## 6. 故障排查清单

1. **\"Not able to find Java executable\" (找不到 Java)**:
    * 检查 `.env` 中是否设置了 `JAVA_HOME`。
    * 检查 Python 代码是否将 `java_home\\bin` 加入到了 `PATH`。

2. **Hang immediately / Timeout (卡死/超时)**:
    * 检查是否使用了 `default jmeter.bat`（它最后有 pause）。请使用 `jmeter_no_pause.bat`。
    * 确保 Python 代码里加了 `stdin=subprocess.DEVNULL`。

3. **\"Unable to access jarfile ...ApacheJMeter.jar\" (找不到 jar 包)**:
    * **根本原因**: `JMETER_BIN` 环境变量冲突。
    * **修复**: 确保在 Python 代码里执行了 `del env['JMETER_BIN']`。

4. **Silent Failure (静默失败：无输出，代码改了也没反应)**:
    * **重启 MCP SERVER**。Agent 进程会将 Python 代码缓存在内存中，修改文件后必须重启服务才能生效。

5. **JTL 文件未生成 (log_file 参数无效)**:
    * 确保使用的是修复后的版本。旧版本中 `log_file` 参数在 `generate_report=False` 时会被忽略。
    * 检查 `.env` 是否从脚本目录正确加载（`load_dotenv()` 需要显式指定路径）。
