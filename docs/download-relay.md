# 下载 Apple 私有 blob 的通用方法：GitHub CI 转存 + 加速器

适用于：**源站慢、会卡死、或完全连不上**（本机 2026-10-29 实测 Apple
`download.info.apple.com` 单连接 85 KB/s → 8 MB 测试 40 s 无响应 → 连不上）。

核心思路（来自 skill `github-relay-mirror`）：

```
慢源站 ──(CI，数据中心，MB/s 级)──> GitHub Release ──(加速器 800 KB/s)──> 本机
```

两端都不走住宅链路直连源站，所以两端都快。**转存的价值是换出口，不是版本管理。**

## 关键结论（本机实测 2026-10-29）

| 环节 | 结果 |
|---|---|
| 源站（Apple，住宅链路） | 85 KB/s → 连不上 |
| GitHub Actions runner 拉 Apple 542 MB | 约 45 s 内完成（数据中心路由） |
| 加速器实测（串行，38 候选） | `ghproxy.felicity.land` 792 KB/s、`ghfast.top` 747、`gh.noki.icu` 736 |
| GitHub 官方 `digest` 与 CI 自算一致 | ✅ 第三份独立校验 |

## 方法 A：只发布关键小文件（推荐）

不用把 542 MB 拉回本机。让 CI 解包后把**小文件单独发布**：

1. 往一个 **public** 仓库（如 `khagav/crdroid-nabu-build`）加一个一次性工作流，
   推到分支触发：
   ```yaml
   name: relay-facetimehd-data
   on: { push: { branches: [ relay-facetimehd ] } }
   permissions: { contents: write }
   jobs:
     extract:
       runs-on: ubuntu-latest
       steps:
         - run: sudo apt-get update -qq && sudo apt-get install -y -qq p7zip-full
         - run: |
             curl -fL --retry 5 --retry-all-errors --speed-time 60 --speed-limit 2048 \
               -C - -o bc.zip "<APPLE_URL>"
             echo "<ZIP_SHA256>  bc.zip" | sha256sum -c -
             git clone ...
             # 或直接 7z 解 + dd 抠出小文件
         - uses: softprops/action-gh-release@v2
           with:
             tag_name: relay-facetimehd-data
             files: "9112_01XX.dat 1771_01XX.dat 1871_01XX.dat 1874_01XX.dat"
   ```
2. 本机只拉小文件（93 KB）：`https://github.com/khagav/crdroid-nabu-build/releases/download/relay-facetimehd-data/1871_01XX.dat`
3. 校验：用 GitHub API 读每个 asset 的 `size` + `digest`（第三方哈希），
   再本机 `sha256sum` 复核。

> **复用**：这个 Release 已存在（`relay-facetimehd-data`），下次直接从
> Release 拉 `.dat`，不必重跑 CI。

## 方法 B：加速器现测再用（从 github.akams.cn 拿候选）

`github.akams.cn` 本身**不是**加速器（前缀式 404，`?url=` 只返回 HTML 外壳），
但它的前端 chunk 里有 55+ 个加速器域名清单，用它当**候选来源**：

```bash
curl -s https://github.akams.cn/ > ak.html
grep -oE '/_next/static/chunks/[a-zA-Z0-9._-]+\.js' ak.html | sort -u \
  | while read -r p; do curl -s "https://github.akams.cn$p"; done \
  | grep -ohE '[a-zA-Z0-9._-]+\.(cn|xyz|top|land|site|icu|net|com|org|io)' | sort -u
```

**必须串行测**（并行会共享上行带宽，排名全错）；`curl --retry` 救不了卡死，
要用 `--speed-time/--speed-limit`；用前先验加速器返回 `206` + `content-range`。

```bash
# 串行排名（-t 每 host 秒数；用真实文件 -u，小文件排名不可信）
scripts/bench-proxies.sh -t 12 /tmp/hostfile.txt
```

## 校验（转存链路每跳都可能出错）

| 层 | 手段 |
|---|---|
| 字节数 | 与源站 Content-Length 比（CI + 本机各一次） |
| 哈希 | sha256：CI 算一次，本机再算一次，比对 |
| 第三方 | GitHub Release asset 的 `digest` 交叉验证（免费，别跳过） |
| 业务语义 | 文件内部声明（如 .dat 大小/用途） |

## 常见失败

| 症状 | 原因 |
|---|---|
| 下载停在某字节不动 | 服务器接受连接后停发数据 → 必须 `--speed-time/--speed-limit` |
| `-C -` 后文件变大/损坏 | 服务器不支持 Range 返回 200 → 先验 `206` |
| 加速器 404 | 前缀式要带完整 `https://github.com/...` |
| 加速器 429 | 限流，换 host |
| 私有仓库资源 404 | 加速器匿名转发取不到私有资源 → 仓库要 public |
| 单文件 > 2 GB | Release 装不下，需切分 |
