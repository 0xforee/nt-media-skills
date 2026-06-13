---
name: "nt-media-search"
description: "通过 curl 命令调用 API，实现 NasTools(NT) 媒体资源的自动化搜索、选种和下载。"
---

# NT 媒体自动下载助手 (curl 版)

本 skill 指导您如何使用 `curl` 命令与 NT 媒体服务的 API 进行交互，完成从“想看某部片”到“在下载器中创建任务”的完整流程。

**先决条件**:
-   你需要 NT 媒体服务地址（IP 或者域名）。
-   你需要一个 NT 媒体服务的用户名和密码。
-   你需要安装 `curl` 和 `jq`，一个强大的命令行获取 HTTP 请求工具和 一个 JSON 处理工具，用于解析 API 返回的数据。如果无法安装，请根据用户环境查找选择替代工具。

---

## 核心流程
### 1. 初始化配置
1.  **配置**: 完成用户的服务器地址配置，以及用户名和密码配置（用于获取后续所有请求需要的认证 `token`）。
### 2. 搜片并完成下载
1.  **搜片**: 根据关键字或标签查找媒体信息。
2.  **搜种**: 触发种子搜索，并获取结果。
3.  **下载**: 根据搜种结果创建下载任务。
4.  **管理**: 查看和管理您的下载任务。
### 3. 订阅并管理订阅
1.  **订阅**: 订阅您感兴趣的媒体资源。
2.  **管理**: 查看和管理您的订阅资源。

---

## 一、配置
> **重要**: 在执行任何其他命令之前，必须先成功完成以下配置。
如果未成功获取到以下配置，请引导用户提供

### 1. 设置服务器地址

提供 NT 服务器地址。

# 将 <your_nt_server> 替换为你的 NT 服务器地址 (例如 http://192.168.0.130:3000)
NT_SERVER="<your_nt_server>"


### 2. 登录并获取 Token

使用用户名和密码登录，获取并设置 `NT_TOKEN` 环境变量。

```bash
export NT_TOKEN=$(curl -s -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "accept: application/json" \
  --data-raw "username=your_username&password=your_password" \
  $NT_SERVER/api/v1/user/login | jq -r '.data.token')

# 验证 token 是否成功获取
echo $NT_TOKEN
```

---

## 二、搜片（在 tmdb 上搜索是否有合适的片子，返回片子的详情，此步骤用于确定你想下载的片是哪个，以及片子是否存在），一般情况下可以直接使用下边的搜种子

使用关键字进行搜索。

```bash
curl -s -X POST \
  -H "Authorization: Bearer $NT_TOKEN" \
  -H "accept: application/json" \
  --data-urlencode "type=SEARCH" \
  --data-urlencode "subtype=" \
  --data-urlencode "page=1" \
  --data-urlencode "keyword=你想搜索的片名" \
  $NT_SERVER/api/v1/recommend/list | jq .
```

---

## 三、搜种 (两步)

### 1. 步骤一：触发种子搜索

```bash
curl -s -X POST \
  -H "accept: application/json" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: $NT_TOKEN" \
  --data-urlencode "search_word=你想搜索的种子关键字" \
  --data-urlencode "unident=1" \
  $NT_SERVER/api/v1/search/keyword | jq .
```

### 2. 步骤二：获取搜索结果

搜索结果要等一下以下 api 返回结果才能成功获取到

```bash
# 稍等片刻后执行
curl -s -X POST \
  -H "accept: application/json" \
  -H "Authorization: $NT_TOKEN" \
  $NT_SERVER/api/v1/search/result | jq .
```

---

## 四、创建下载任务

从上一步的搜种结果中，找到想下载的种子的 `id`。

```bash
# 将 1 替换为实际的资源 id
curl -s -X POST \
  -H "accept: application/json" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: $NT_TOKEN" \
  --data-raw "id=1" \
  $NT_SERVER/api/v1/download/search | jq .
```

---

## 五、任务管理

### 1. 查询正在下载的任务
```bash
curl -s -X POST \
  -H "accept: application/json" \
  -H "Authorization: $NT_TOKEN" \
  -d '' \
  $NT_SERVER/api/v1/download/now | jq .
```

### 2. 查询指定种子 id 的下载进度（支持多个 id）, 先调用 now 获取
```bash
curl -s -X POST \
  -H "accept: application/json" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: $NT_TOKEN" \
  --data-raw "ids=1,2" \
  $NT_SERVER/api/v1/download/info | jq .
```

### 3. 删除下载任务
```bash
curl -s -X POST \
  -H "accept: application/json" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: $NT_TOKEN" \
  --data-raw "id=1" \
  $NT_SERVER/api/v1/download/remove | jq .
```

### 4. 启动下载任务
```bash
curl -s -X POST \
  -H "accept: application/json" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: $NT_TOKEN" \
  --data-raw "id=1" \
  $NT_SERVER/api/v1/download/start | jq .
```

### 5. 停止下载任务
```bash
curl -s -X POST \
  -H "accept: application/json" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: $NT_TOKEN" \
  --data-raw "id=1" \
  $NT_SERVER/api/v1/download/stop | jq .
```

---

## 六、订阅管理

### 1. 搜索媒体信息
通过关键字搜索 TMDB 上的媒体信息，获取 tmdb_id。

```bash
curl -s -X POST \
  -H "accept: application/json" \
  -H "Authorization: $NT_TOKEN" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "keyword=雨霖铃" \
  $NT_SERVER/api/v1/media/search | jq '.data.result[] | {title, year, tmdb_id, type}'
```

### 2. 添加订阅
支持电视剧和电影订阅，可设置搜索词、分辨率、画质等筛选条件。
使用前请参考参数说明。

```bash
curl -s -X POST \
  -H "accept: application/json" \
  -H "Authorization: $NT_TOKEN" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "name=雨霖铃" \
  --data-urlencode "type=TV" \
  --data-urlencode "year=2023" \
  $NT_SERVER/api/v1/subscribe/add | jq .
```

**参数说明：**
| 参数 | 必填 | 说明 |
|------|------|------|
| name | 是 | 订阅名称 |
| type | 是 | 类型：TV / MOV |
| year | 是 | 年份（如果不确定，可以从搜片结果中获取） |
| keyword | 否 | 自定义搜索词，不填则用 name |
| season | 否 | 季号，电视剧必填 |
| mediaid | 否 | TMDBID，可精确匹配媒体 |
| fuzzy_match | 否 | 模糊匹配：0-否 / 1-是（有 mediaid 时不使用此参数） |
| filter_pix | 否 | 分辨率筛选，如 4K、1080p |
| filter_restype | 否 | 资源类型，如 WEB-DL、BluRay |
| filter_team | 否 | 字幕组/发布组筛选 |
| over_edition | 否 | 洗版：0-否 / 1-是 |
| download_setting | 否 | 下载设置，0=默认 |
| save_path | 否 | 自定义保存路径 |
| rss_sites | 否 | RSS 站点（逗号分隔） |
| search_sites | 否 | 搜索站点（逗号分隔） |

### 3. 查看订阅列表
```bash
# 电视剧订阅
curl -s -X POST \
  -H "accept: application/json" \
  -H "Authorization: $NT_TOKEN" \
  -d '' \
  $NT_SERVER/api/v1/subscribe/tv/list | jq .

# 电影订阅
curl -s -X POST \
  -H "accept: application/json" \
  -H "Authorization: $NT_TOKEN" \
  -d '' \
  $NT_SERVER/api/v1/subscribe/movie/list | jq .
```

### 4. 查看订阅详情
```bash
curl -s -X POST \
  -H "accept: application/json" \
  -H "Authorization: $NT_TOKEN" \
  --data-raw "rssid=14" \
  $NT_SERVER/api/v1/subscribe/info | jq .
```

### 5. 删除订阅
```bash
curl -s -X POST \
  -H "accept: application/json" \
  -H "Authorization: $NT_TOKEN" \
  --data-raw "rssid=14" \
  $NT_SERVER/api/v1/subscribe/delete | jq .
```
| 参数 | 必填 | 说明 |
|------|------|------|
| rssid | 是 | 订阅 ID |
| name | 是 | 订阅名称 |
| type | 是 | 类型: TV/MOV |

### 6. 订阅历史
```bash
curl -s -X POST \
  -H "accept: application/json" \
  -H "Authorization: $NT_TOKEN" \
  -d '' \
  $NT_SERVER/api/v1/subscribe/history | jq .
```

---

## 附录：其他常用 API

### 查看媒体详情
```bash
curl -s -X POST \
  -H "accept: application/json" \
  -H "Authorization: $NT_TOKEN" \
  --data-urlencode "tmdbid=254486" \
  $NT_SERVER/api/v1/media/detail | jq .
```

### RSS 站点管理
```bash
# 列出 RSS 解析器
curl -s -X POST \
  -H "accept: application/json" \
  -H "Authorization: $NT_TOKEN" \
  -d '' \
  $NT_SERVER/api/v1/rss/parser/list | jq .

# 预览 RSS
curl -s -X POST \
  -H "accept: application/json" \
  -H "Authorization: $NT_TOKEN" \
  --data-urlencode "url=https://example.com/rss.xml" \
  $NT_SERVER/api/v1/rss/preview | jq .
```

### 系统信息
```bash
# 查询系统最新版本信息
curl -s -X POST \
  -H "accept: application/json" \
  -H "Authorization: $NT_TOKEN" \
  $NT_SERVER/api/v1/system/version | jq .

