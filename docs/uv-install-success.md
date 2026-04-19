# OpenHarness 环境下安装 uv 成功流程

更新时间：2026-04-19

## 1. 现象与原因
在 `OpenHarness` 目录执行安装时，曾出现以下网络错误：
- `snap install astral-uv --classic` 报 `EOF`
- `curl -LsSf https://astral.sh/uv/install.sh | sh` 报 `SSL routines::unexpected eof while reading`

这类错误通常是临时网络抖动或 TLS 连接被中断导致，不是 `uv` 包本身问题。

## 2. 先做连通性检查
```bash
curl -I --max-time 12 https://astral.sh
curl -I --max-time 12 https://github.com
curl -I --max-time 12 https://pypi.org
```
预期：返回 `HTTP/2 200`（或其他 2xx/3xx 响应）。

## 3. 使用官方脚本安装（本次成功路径）
在 `OpenHarness` 目录执行：

```bash
cd /home/chen-hao/repositories/laborhelper/OpenHarness
curl -LsSf https://astral.sh/uv/install.sh | sh
```

安装器会把二进制放到：
- `~/.local/bin/uv`
- `~/.local/bin/uvx`

## 4. 验证安装
```bash
export PATH="$HOME/.local/bin:$PATH"
uv --version
command -v uv
```

本次实际验证结果：
- `uv 0.11.7 (x86_64-unknown-linux-gnu)`
- `/home/chen-hao/.local/bin/uv`

## 5. 若新终端找不到 uv，补充 PATH
把以下内容加到 `~/.bashrc`：

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## 6. 可选：快速自检
```bash
uv --version
uvx --version
```
