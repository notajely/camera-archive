这份开发文档是为了将当前的 Hugo 静态博客架构重构为**结构化、数据库化**的个人相机档案系统。

---

# 📷 FilmCameraDB 重构与现代化开发文档 (v1.0)

## 1. 项目目标

将现有的“文章流”模式（基于 Shortcodes 手写参数）升级为“结构化数据”模式（基于 Front Matter 数据库）。
**核心收益：** 支持多维筛选（如按卡口、画幅查找）、支持站内即时搜索、分离数据与展示逻辑。

---

## 2. 核心架构变更 (Configuration)

### 目标文件：`hugo.toml`

需要在配置文件中注册新的分类法（Taxonomies）和开启搜索索引（JSON Output）。

**变更规范：**

```toml
# 1. 扩展分类法，建立数据库索引维度
[taxonomies]
  tag = "tags"
  brand = "brands"       # 现有
  mount = "mounts"       # 新增：卡口 (如 M39, Nikon F)
  film_format = "formats"# 新增：画幅 (如 135, 120)
  year = "years"         # 新增：年份 (可选)

# 2. 开启 JSON 输出，为站内搜索提供数据源
[outputs]
  home = ["HTML", "RSS", "JSON"]

# 3. 配置搜索权重 (Fuse.js)
[params.fuseOpts]
  isCaseSensitive = false
  shouldSort = true
  location = 0
  distance = 1000
  threshold = 0.4
  minMatchCharLength = 0
  # 搜索范围包括：标题、正文、卡口字段、品牌
  keys = ["title", "content", "params.specs.mount", "brands"]

```

---

## 3. 数据模型定义 (Data Schema)

### 目标文件：`content/cameras/**/*.md` (Front Matter)

废弃在正文中手写 `{{< spec >}}` 的方式，将所有参数移入 YAML 头部的 `specs` 和 `collection` 字段。

**新版 Front Matter 标准模板：**

```yaml
---
title: "Canon IV Sb（1952）"
date: 2026-02-17
draft: false

# --- 索引字段 (Taxonomies) ---
brands: ["canon"]
mounts: ["M39 (LTM)"]        # 对应 hugo.toml 中的定义
formats: ["135 (35mm)"]      # 对应 hugo.toml 中的定义
tags: ["rangefinder", "mechanical"]

# --- 公开参数 (Specs) ---
# 用于页面自动渲染参数表
specs:
  lens: "35mm f/3.5 Canon Serenar; 50mm f/1.9 Canon Serenar"
  focus_range: "1.07m to infinite"
  viewfinder: "Rangefinder, Rotating Variable Prism"
  shutter_type: "Cloth Focal Plane"
  shutter_speeds: "T, B, 1–1/1000 sec"
  flash_sync: "FP + X-sync"
  metering: "None"
  battery: "None"

# --- 个人收藏档案 (Collection - Private/Admin) ---
# 用于个人资产管理，页面可选择性展示或隐藏
collection:
  status: "Owned"            # Enum: Owned, Sold, Wishlist
  serial_body: "123456"      # 机身编号
  serial_lens: "789012"      # 镜头编号
  acquired_date: "2024-01-15"
  price: "1500 CNY"
  condition: "Ex+ (黄斑清晰)"
  maintenance_log:           # 简易维修记录
    - "2024-02: 调整黄斑垂直合焦"
---

```

---

## 4. 模板开发规范 (Template Implementation)

### 任务 A：创建参数表组件 (Partial)

新建文件 `layouts/partials/camera_specs.html`，用于自动化渲染 `specs` 数据。

**逻辑伪代码：**

```html
<div class="camera-specs-container">
  <h3>Technical Specifications</h3>
  <div class="specs-grid">
    {{ with .Params.specs.mount }}
      <div class="spec-item">
        <span class="label">Lens Mount</span>
        <span class="value"><a href="/mounts/{{ . | urlize }}">{{ . }}</a></span>
      </div>
    {{ end }}
    
    {{ with .Params.specs.shutter_speeds }}
      <div class="spec-item">
        <span class="label">Shutter</span>
        <span class="value">{{ . }}</span>
      </div>
    {{ end }}
    
    </div>
</div>

```

### 任务 B：创建收藏信息组件 (Partial)

新建文件 `layouts/partials/collection_info.html`。

**逻辑要求：**

* 展示收藏状态（Owned/Wanted）。
* 可以增加一个逻辑判断：如果 `hugo server` (本地开发模式) 或者是特定页面，才显示 `price` 和 `serial_body`，防止隐私泄露（或者直接简单渲染，视需求而定）。

### 任务 C：集成到页面布局

修改 `layouts/_default/single.html` (或 `layouts/cameras/single.html`)：

```html
{{ define "main" }}
<article class="post-single">
  <div class="post-content">
    {{ partial "camera_specs.html" . }}
    
    {{ .Content }}
    
    {{ partial "collection_info.html" . }}
  </div>
</article>
{{ end }}

```

---

## 5. 迁移与执行清单 (Action Checklist for Agent)

请 Agent 按以下顺序执行：

1. **配置更新**：修改 `hugo.toml`，添加 Taxonomies 和 Search Outputs。
2. **数据迁移**：
* 读取 `content/cameras/` 下的所有 `.md` 文件。
* 提取原 `{{< spec label="xxx" >}}content{{< /spec >}}` 中的内容。
* 重写 Front Matter，将提取的内容填入新的 `specs` YAML 结构中。
* 删除正文中的 Shortcodes 代码块。


3. **模板创建**：创建 `layouts/partials/camera_specs.html` 并实现样式。
4. **验证**：运行 `hugo server`，确保页面能正确显示参数，并且点击“M39”等卡口链接能跳转到分类聚合页。

---

创建gitignore 并注意必须要先进行plan再进行操作