# Windows Python — Pip 安装（从 WSL 侧操作）

## 场景
需要给 Windows 侧 Python 安装包，但 execute_code 的 subprocess 方式超时（大包如 akshare/pandas）。

## 原理
Windows Hermes Agent 的 terminal 工具实际走的是 WSL bash（git-bash/MSYS）。WSL 可以通过 `/mnt/c/` 访问 Windows 文件系统，包括 Windows Python 可执行文件。

## 命令模式

```bash
"/mnt/c/Program Files/Python312/python.exe" -m pip install <package> --default-timeout=120
```

## 实用技巧

### 1. 后台安装 + 通知（推荐大包用）
```bash
# terminal(background=True, notify_on_complete=True)
"/mnt/c/Program Files/Python312/python.exe" -m pip install pandas --default-timeout=120 2>&1 | tail -5
```

### 2. 先装主包（--no-deps），再分批装依赖
```python
# execute_code 中 subprocess 方式（适用于小包）
subprocess.run([sys.executable, "-m", "pip", "install", "akshare", "--no-deps", "--default-timeout=120"])
# 依赖分批
subprocess.run([sys.executable, "-m", "pip", "install", "decorator", "html5lib", "jsonpath", "--default-timeout=120"])
```

### 3. C扩展包优先用wheel（--only-binary）
```bash
"/mnt/c/Program Files/Python312/python.exe" -m pip install lxml --only-binary=lxml --default-timeout=120
```

## 已知踩坑

| 问题 | 表现 | 解决 |
|------|------|------|
| execute_code 超时 | TimeoutExpired after 300s | 改用 terminal background |
| terminal 直接写 Windows 路径 | command not found | 用 `/mnt/c/...` WSL格式 |
| 大包下载慢 | 长时间无输出 | 加 `--default-timeout=120` 延长超时 |
| 二进制校验失败 | Hash mismatch | 跳过（如 mini-racer），不影响核心功能 |

## 验证安装

```python
# execute_code 中执行
r = subprocess.run([sys.executable, "-c", "import akshare; print('ok')"], capture_output=True, text=True)
print(f"ok={r.returncode==0}")
```
