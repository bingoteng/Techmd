# Techmd

Markdown 知识笔记仓库（Typora / MarkText）。

## 目录结构

```
Techmd
├── marktext/          # MarkText 笔记（含本地 assets 编辑缓存）
├── typora/            # Typora 笔记（含本地 assets 编辑缓存）
├── README.md
└── .gitignore
```

## 图片管理规范

### 核心模型

- **Techmd（Git）**：只管理 Markdown 文档，图片引用一律使用图床 CDN URL。
- **本地 `assets/`**：Typora/MarkText 编辑时的临时图片缓存，**不纳入 Git**（已被 `.gitignore` 的 `**/assets/` 忽略）。
- **`bingoteng/Photos`**：独立图床仓库，最终图片存放地，通过 jsDelivr CDN 访问：
  `https://cdn.jsdelivr.net/gh/bingoteng/Photos/Linux/<文件名>`

### 一次笔记编辑流程

#### 1. 编辑阶段（Typora / MarkText）

- 正常写笔记、插入图片，图片自动拷贝到本地 `assets/`。
- 此阶段图片从本地加载，可离线编辑，不用担心误提交。

#### 2. 整理发布阶段（笔记完成后）

1. 扫描该笔记的图片引用，注意同时检查两种形式：
   - Markdown：`![alt](assets/xxx.png)`
   - HTML：`<img src="assets/xxx.png">`（最容易遗漏）
2. **只上传“引用到的图片”**到图床（当前 PicGo 已配置为 `bingoteng/Photos/Linux/`），不要把整个 assets 文件夹上传。
3. 将本地引用替换为 CDN URL，保持内容结构不变：
   - `![alt](assets/xxx.png)` → `![alt](https://cdn.jsdelivr.net/gh/bingoteng/Photos/Linux/xxx.png)`
4. 上传后抽查 1–2 个 URL 确认返回 200。
5. 本地 `assets/` 缓存可保留（方便以后改图），确认无其他笔记引用后可手动清理。

#### 3. 提交同步阶段（双端共享仓库）

- `git add` 只加改动的 `.md` 文件（`assets` 自动被忽略）。
- `git commit` + `git push`。
- 另一端 `git pull` 无冲突、不会删除任何本地图片（图片在 CDN，两端各自保留本地缓存）。

#### 4. 定期维护（可选）

- 用 dry-run 扫描所有 Markdown 的引用集合，对比图床仓库文件，列出“未被引用”的远程图片。
- 人工确认后再删除（jsDelivr 有缓存，删除后不会立即 404；git 历史仍可找回）。

### 红线规则

- 本地 `assets/` 永远不入库（不要 `git add -f`）。
- 已提交笔记中不要出现本地相对路径或 `D:/...` 绝对路径（跨设备必断）。
- 上传遵循“以 Markdown 引用为准”，避免污染图床仓库。
- 不要在图床副本 + Markdown 引用更新完成前删除本地图片。

## 遗留待办（一次性）

- `bingoteng/image` 历史失效引用 11 处 + 本地路径引用 27 处：需从 Windows 端 `D:/Typroa_notes/99-Store_photo_addr/` 取回源文件后一次性迁移。
- `typora/Y_辅助文档/` 的 PDF 资料（约 53 MB）后续考虑迁移出仓库（独立资料库 / Git LFS）。
