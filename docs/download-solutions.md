# 网盘下载方案汇总

> 本文件记录所有已实现和可行的网盘下载方案，供后续扩展新 workflow 时参考。

---

## 已实现的 Workflow

### 1. cloud.mail.ru

| 项目 | 内容 |
|------|------|
| Workflow 文件 | `download.yml` |
| Workflow 名称 | `Download cloud.mail.ru` |
| 状态 | ✅ 已验证可用 |
| 用户输入 | `share_url`：cloud.mail.ru 公开分享链接 |

**原理：**
1. `curl` 打开分享页面 HTML
2. Python 正则提取 `"weblink_get"` 中的 CDN base URL
3. 拼接 `{base_url}/{share_path}`
4. 跟随 301 重定向到真实 CDN 下载地址
5. `curl -L` 下载文件
6. 分片 450MB → upload-artifact

**关键代码模式：**
```python
# 解析 weblink_get
m = re.search(r'"weblink_get"\s*:\s*\{[^}]*"url"\s*:\s*"([^"]+)"', html)
base_url = m.group(1)
download_url = f"{base_url}/{share_path}"
# curl -L 跟随 301 下载
```

**注意事项：**
- 无需登录，公开分享链接直接可用
- CDN 节点不固定（cloclo57/58/60/61 等），每次可能不同
- 301 重定向后的 URL 才是真实下载地址
- `grep -oP` 在解析 HTML 时不可靠（会吃多余字符），必须用 Python regex

---

### 2. Rapidgator

| 项目 | 内容 |
|------|------|
| Workflow 文件 | `download-rapidgator.yml` |
| Workflow 名称 | `Download Rapidgator` |
| 状态 | ⚠️ 需要账号 |
| 用户输入 | `file_url`：Rapidgator 文件链接 |
| 所需 Secrets | `RG_EMAIL`、`RG_PASSWORD` |

**原理（官方 API）：**
1. 登录：`GET /api/v2/user/login?login=EMAIL&password=PASSWORD` → 获取 token
2. 查文件：`GET /api/v2/file/check_link?url=URL&token=TOKEN` → 获取文件名/大小
3. 获取直链：`GET /api/v2/file/download?url=URL&token=TOKEN` → 获取 download_url
4. `curl -L` 下载直链

**限制：**
- 免费用户：每 2 小时只能下载 1 个文件，有限速，无直链（部分）
- 需要注册 Rapidgator 账号
- API 可能限制 GitHub Actions 出口 IP

**备选方案（无需账号）：**
- 用 Playwright 模拟浏览器，等待倒计时后点击下载
- 但免费用户有 60-120 秒等待 + CAPTCHA 风险

---

## 可行但未实现的方案

### 3. Google Drive

| 项目 | 内容 |
|------|------|
| 状态 | ✅ 技术可行 |
| 是否需要登录 | 公开分享不需要 |

**原理：**
1. 从分享链接提取 file ID
2. 调用 `https://drive.google.com/uc?export=download&id=FILE_ID`
3. 跟随重定向，处理大文件的病毒扫描确认页面
4. 用 `confirm` token 下载

**关键模式：**
```python
# 提取 file ID
file_id = url.split("/d/")[1].split("/")[0]
# 或从 URL 参数提取: ?id=FILE_ID
# 大文件需要两次请求: 第一次拿 confirm token, 第二次带 token 下载
```

**注意事项：**
- 大文件（>100MB）会触发"病毒扫描警告"页面，需要解析 confirm token
- Google 可能封 GitHub Actions 的 IP
- 有 Google API key 可以更稳定

---

### 4. Mega

| 项目 | 内容 |
|------|------|
| 状态 | ⚠️ 需要特殊处理 |
| 是否需要登录 | 公开分享不需要 |

**原理：**
- Mega 使用客户端加密，不能简单 curl 下载
- 需要用 `megatools`（命令行工具）或 Python `mega.py` 库
- 工具需要解密文件，内存占用大

**实现方式：**
```bash
# megatools
pip install megatools
megadl 'https://mega.nz/file/XXX#KEY'
```

**注意事项：**
- GitHub Actions runner 内存 7GB，大文件可能 OOM
- megatools 需要 `apt install megatools` 或 pip 安装
- 需要处理公私钥加密

---

### 5. 百度网盘

| 项目 | 内容 |
|------|------|
| 状态 | ❌ 高度受限 |
| 是否需要登录 | 必须登录 |

**现实情况：**
- 必须登录百度账号（cookie / token）
- 下载限速严重（免费用户 100KB/s 级别）
- 有各种安全验证（滑块、短信）
- GitHub Actions IP 可能被风控

**可能的方案：**
- 用 BaiduPCS-Go 或类似工具 + 用户 cookie
- 但 cookie 有效期短，不适合 CI/CD
- **结论：不建议实现，体验太差**

---

### 6. 阿里云盘 / 夸克网盘

| 项目 | 内容 |
|------|------|
| 状态 | ⚠️ 技术可行 |
| 是否需要登录 | API 需要 token |

**阿里云盘：**
- 有公开 API，分享链接可以获取文件信息
- 下载需要 refresh_token
- GitHub 上有开源客户端（alist、aliyundrive-open-index）

**夸克网盘：**
- 类似百度网盘，需要 cookie
- API 不公开，逆向工程不稳定

---

### 7. 1fichier / NitroFlare / Rapidgator 等传统网盘

| 项目 | 内容 |
|------|------|
| 状态 | ✅ 大多有 API |

**通用模式：**
1. 登录拿 token
2. 查文件信息
3. 获取直链（通常有时效性）
4. curl 下载

**常见网盘 API：**
- 1fichier：`https://1fichier.com/api/download.cgi?token=TOKEN&id=FILE_ID`
- NitroFlare：`https://nitroflare.com/api/...`
- Katfile：有 API 文档

---

## 新增 Workflow 的标准模板

当需要为新网盘创建 workflow 时，按以下步骤：

### Step 1: 调研

1. 搜索 `[网盘名] API download curl python`
2. 确认是否需要登录
3. 找到获取直链的 API 端点
4. 测试 API 是否能从 GitHub Actions runner 访问

### Step 2: 判断方案类型

| 条件 | 方案 |
|------|------|
| 有公开 API，不需要登录 | curl + Python 解析（最简单） |
| 有 API，需要登录 | curl + API 认证 + Secrets |
| 无 API，只有网页 | Playwright 模拟浏览器 |
| 文件加密（如 Mega） | 专用工具解密 |

### Step 3: Workflow 结构

```yaml
name: Download [网盘名]

on:
  workflow_dispatch:
    inputs:
      file_url:
        description: '[网盘名] 文件链接'
        required: true

jobs:
  download:
    runs-on: ubuntu-latest
    steps:
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Resolve and download
        run: |
          python3 << 'PYEOF'
          # 1. 解析文件 URL
          # 2. 调用 API 获取直链（或模拟浏览器）
          # 3. curl -L 下载到 /tmp/downloaded_file
          PYEOF
        env:
          FILE_URL: ${{ github.event.inputs.file_url }}
          # 如果需要登录：
          # RG_EMAIL: ${{ secrets.RG_EMAIL }}

      - name: Split into chunks
        run: |
          mkdir -p /tmp/parts
          split -b 450M -d -a 3 /tmp/downloaded_file /tmp/parts/file_part_

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: [网盘名]-download
          path: /tmp/parts/file_part_*
          retention-days: 7
```

### Step 4: 注意事项

- **文件名**：不要用原始文件名做变量（空格、特殊字符），统一用 `downloaded_file`
- **分片**：450MB 是安全值（artifact 单文件上限 500MB）
- **验证**：下载后检查文件大小，太小说明失败
- **重试**：curl 加 `--retry 5 --retry-delay 30`
- **清理**：本地 clone 后必须 `rm -rf` 清理
- **Secrets**：需要登录的网盘，在仓库 Settings → Secrets 配置

---

## 环境信息

| 项目 | 内容 |
|------|------|
| 仓库 | `https://github.com/Jammmmmaa/linshi` |
| Runner | `ubuntu-latest`（GitHub Actions） |
| Python | 3.11（通过 actions/setup-python） |
| 系统工具 | curl、split、jq |
| Artifact 限制 | 单文件 500MB，保留 7 天 |

---

## 问题排查

| 问题 | 原因 | 解决 |
|------|------|------|
| `grep -oP` 结果异常 | macOS grep 不支持 -P，或匹配到多余字符 | 用 Python regex |
| 403 / 被拒绝 | IP 被封、需要登录、cookie 过期 | 检查是否需要认证 |
| 下载文件太小 | 重定向没跟、拿到的是错误页面 | curl 加 `-L`，检查大小 |
| artifact 上传失败 | 文件超过 500MB | 减小 split 的 chunk size |
| `curl: (3) URL rejected` | URL 有换行或特殊字符 | 清理变量中的空白字符 |
