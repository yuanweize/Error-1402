# Error 1402 – 伪装错误页（静态单页）

语言: 简体中文 | [English](./README.en.md)

## 截图

English

![Error 1402 - English](./assets/screenshot-en.png)

中文

![错误 1402 - 中文](./assets/screenshot-zh.png)

一个用于“伪装/掩护”真实服务的静态错误页模板。页面显示为类似 CDN 的错误页面（示例文案为 EURUN CDN 的风格），包含 Ray ID、UTC 时间戳与多源 IP 检测，用于在对外暴露时降低真实站点特征或在访问受限时提供统一的错误外观。

> 说明：本页展示的“Error 1402”并非标准 HTTP 状态码，属于“故意模糊化”的展示内容。实际 HTTP 状态码由你的服务器/反向代理配置决定（通常建议返回 403 或 404）。


## 这是什么

- 目的：提供一个“看起来像是 CDN/安全设备返回的错误页”的前端静态页面，用于掩护后端真实应用或作为访问受限时的统一落地页。
- 形态：纯静态、单文件 `index.html`，内联 CSS/JS，无外部依赖，开箱即用。
- 特点：
	- 生成伪造的 Ray ID（可点击复制）和 UTC 时间戳；
	- 并发从 3 个公共服务获取访问者 IP（IPIP、IPInfo、IPify），成功/失败分别标记；
	- 自适应（桌面与移动端）排版；
	- 品牌文案、公司信息、联系邮箱等易于替换；
	- 代码简洁，易于二次定制。


## 快速开始（部署）

你可以将 `index.html` 直接放到任意静态托管或 Web 服务器上：

- 静态托管：GitHub Pages / Cloudflare Pages / Vercel / Netlify 等；
- 传统服务器：Nginx / Apache / Caddy 直接作为站点根或特定路径的默认页；
- 反向代理前置：作为默认后备（fallback）页或错误页（如 403/404）的返回内容。

推荐做法：
- 作为“未知域名/未配置主机”的默认站点返回；
- 作为“拒绝访问（403）/未找到（404）”的错误页；
- 在维护窗口或灰度/切换期间用作临时占位页。

> 小贴士：出于迷惑性考虑，页面标题/文案展示为“Error 1402”，但你仍可让服务器返回 403/404/503 等常见状态码，不影响展示效果。


## 在反向代理/Nginx 中的思路示例（伪代码）

- 将 `index.html` 放到一个可读目录；
- 将 403/404 的错误处理指向该文件；
- 或者为默认服务/兜底 server 块直接返回该文件内容。

以上为思路示例，请依据你的生产环境做安全与合规配置（TLS、主机名匹配、缓存策略等）。


## 常见服务器配置示例（可直接拷贝）

以下示例均以 Linux 路径演示，请将 `/var/www/error1402` 替换为你存放 `index.html` 的实际目录。

### Nginx

1) 映射为错误页（保留真实状态码，推荐）

```nginx
server {
	listen 80;
	server_name example.com;

	# 你的正常站点配置...

	# 将 403/404/503 错误的响应体替换为伪装页，但保留原始状态码
	error_page 403 /error-1402.html;
	error_page 404 /error-1402.html;
	error_page 503 /error-1402.html;

	# 方式 A：单独复制一个文件名，避免与业务 index 冲突
	location = /error-1402.html {
		root /var/www/error1402;   # 目录中包含 index.html（可另存为 error-1402.html）
		internal;                  # 仅供 error_page 内部跳转使用
		default_type text/html;
		add_header Cache-Control "no-store";
	}
	# 方式 B：直接复用 index.html（任选其一）
	# location = /error-1402.html {
	#     alias /var/www/error1402/index.html;
	#     internal;
	#     default_type text/html;
	#     add_header Cache-Control "no-store";
	# }
}
```

2) 作为默认兜底站点（用于未匹配的主机名/域名）

保留错误状态码示例（以 404 为例）：

```nginx
server {
	listen 80 default_server;
	server_name _;

	# 将任何请求都返回 404，但用伪装页作为响应体
	error_page 404 /index.html;
	location = /index.html {
		root /var/www/error1402;
		internal;
		default_type text/html;
		add_header Cache-Control "no-store";
	}
	location / {
		return 404;
	}
}
```

直接 200 展示示例（不保留错误码）：

```nginx
server {
	listen 80 default_server;
	server_name _;
	root /var/www/error1402;

	location / {
		try_files /index.html =404;  # 总是返回 index.html，状态码为 200
		add_header Cache-Control "no-store";
	}
}
```


### Apache (httpd)

1) 映射为错误页（保留真实状态码）

将伪装页文件置于站点 DocumentRoot 内（可命名为 `error-1402.html` 或直接使用 `index.html`）。

```apache
# 虚拟主机或 .htaccess 中
ErrorDocument 403 /error-1402.html
ErrorDocument 404 /error-1402.html
ErrorDocument 503 /error-1402.html

# 可选：为存放目录设置响应头
<Directory "/var/www/error1402">
	Options -Indexes
	AllowOverride None
	Require all granted
	<IfModule mod_headers.c>
		Header set Cache-Control "no-store"
	</IfModule>
	AddType text/html .html
	DefaultType text/html
	CharsetSourceEnc UTF-8
	AddDefaultCharset UTF-8
</Directory>
```

2) 作为默认兜底站点（保留 403 或 404）

示例：总是返回 403，并用伪装页作为响应体。

```apache
<VirtualHost *:80>
	ServerName _default_
	DocumentRoot "/var/www/error1402"

	# 始终返回 403（Forbidden）
	<IfModule mod_rewrite.c>
		RewriteEngine On
		RewriteRule ^ - [F]
	</IfModule>

	# 用伪装页作为 403 的响应体
	ErrorDocument 403 /index.html

	<Directory "/var/www/error1402">
		Options -Indexes
		AllowOverride None
		Require all granted
		<IfModule mod_headers.c>
			Header set Cache-Control "no-store"
		</IfModule>
	</Directory>
</VirtualHost>
```

若希望返回 404，同理将 `[F]` 改为：

```apache
RewriteRule ^ - [R=404]
ErrorDocument 404 /index.html
```


### Caddy v2

1) 映射为错误页（保留真实状态码）

```caddyfile
example.com {
	# 你的正常站点配置...

	handle_errors {
		@4xx expression {http.error.status_code >= 400 && http.error.status_code < 500}
		rewrite @4xx /index.html
		file_server
	}
}
```

2) 作为默认兜底站点

保留错误状态码示例（以 403 为例）：

```caddyfile
:80 {
	root * /var/www/error1402
	# 主动返回 403，交给 handle_errors 渲染伪装页
	respond / 403

	handle_errors {
		rewrite * /index.html
		file_server
	}
}
```

直接 200 展示示例：

```caddyfile
:80 {
	root * /var/www/error1402
	try_files /index.html
	file_server
}
```

> 提示：上面示例均未包含 TLS。若用于生产环境，请配合 HTTPS、HSTS、缓存策略与访问控制等进行完善配置。


## 自定义与二次开发

打开并编辑 `index.html`：

- 品牌与文案：
	- `<title>`、页面主标题、副标题（“What happened? / What can I do?” 等），底部公司信息与邮箱均可直接替换。
- 错误码与提示：
	- 将 “Error 1402” 改为你需要的文案；也可以改为“403 Forbidden”等更常见的呈现。
- Ray ID 与时间戳：
	- 由前端 JS 伪随机生成，仅用于“看起来像真的”；点击 Ray ID 支持复制。
- IP 检测：
	- 位于脚本中 `ipServices` 数组，可增删第三方查询源；
	- 默认 3 秒超时并发请求；
	- 若不希望产生外部网络请求，将 `ipServices` 置空，或注释 `fetchUserIp()` 调用，并移除相应 UI。
- 样式：
	- 所有样式为内联 CSS，可直接修改色彩、布局与排版；
	- 移动端响应式在 `@media (max-width:768px)` 中调整。


### 多语言与懒加载（i18n）

- 页面右上角提供语言选择器，现已内置 30+ 常用语言，语言包采用“按需加载”的外部 JSON 文件：`assets/i18n/<code>.json`。
- 默认语言自动检测优先级：
	1. 用户明确选择的语言（`localStorage.lang`，仅当 `localStorage.langPinned=1` 时才生效，避免误固定）；
	2. 逐项解析浏览器语言列表：对每一项先尝试“精确匹配”，若无则尝试其“基础前缀匹配”（例如 `zh-CN` → `zh`），再继续下一项；
	3. 仍未命中时兜底为 `en`。
- 语言加载与回退链：会按“精确语言 → 基础前缀 → 英文(页面内置基线)”三层合并加载，缺失键会自动用上层回退补齐。
- 英文(en)不再从 `assets/i18n/en.json` 拉取，而是以内置 DOM 文案作为“英文基线”。只有非英文语言才会按需拉取对应 JSON。
- 已内置语言（代码 → 母语名，节选）：
	- `en` English，`zh` 中文，`es` Español，`fr` Français，`de` Deutsch，`ja` 日本語，`ko` 한국어，`ru` Русский，`ar` العربية，`pt` Português，`it` Italiano，`nl` Nederlands，`sv` Svenska，`no` Norsk，`pl` Polski，`tr` Türkçe，`hi` हिन्दी，`th` ไทย，`vi` Tiếng Việt，`id` Bahasa Indonesia，`he` עברית，`uk` Українська，`cs` Čeština，`ro` Română，`el` Ελληνικά，`hu` Magyar，`da` Dansk，`fi` Suomi，`bg` Български，`sk` Slovenčina，`ca` Català，`ms` Bahasa Melayu，`fil` Filipino。
- RTL（从右到左）语言支持：当选择 `ar` 或 `he` 时，会自动设置 `dir="rtl"`，其它语言为 `ltr`。
- 可翻译键位包含：
	- `title`, `error_code`, `ray_prefix`, `error_description`
	- `ip_title`, `detecting`, `ip_source_failed`, `copied`
	- `what_happened`, `what_happened_p1`, `what_happened_p2`
	- `what_can_i_do`, `what_can_i_do_p1`, `what_can_i_do_p2`
	- `footer_line1`, `footer_line2`, `sep`

#### 什么是“懒加载”语言包？

- 为减少首屏 JS 体积，页面仅在需要时请求相应的 `assets/i18n/<code>.json` 语言文件，并在内存中缓存；
- 若语言文件缺失或网络失败，将自动按回退链合并英文“内置基线”文案，确保页面可用；
- 部署注意：
	- 浏览器出于安全限制，`file://` 直接双击打开可能会拦截 `fetch()` 读取 JSON。请在本地启动一个简易的 HTTP 服务再访问（例如使用任意静态服务器）；
	- 建议为 `assets/i18n/*.json` 设置缓存头：`Cache-Control: public, max-age=86400, immutable`，以减少重复请求；
	- 如需新增 RTL 语言，请将代码加入脚本中的 `rtlLangs` 集合。

#### 新增语言步骤

1. 复制任意现有语言文件（建议 `zh.json` 或其它）为 `assets/i18n/<code>.json`，逐项翻译所有键；
2. 在 `index.html` 的 `languageMeta` 中加入 `<code>: '母语名'`，即可在选择器中显示；
3. 若该语言需要特殊分隔符（如中文“：”），在 JSON 中设置 `"sep"`；
4. 如需新增页面段落，对元素加 `data-i18n="key"` 并在各语言 JSON 中补充对应键。

#### 调试与故障排查

- 在地址栏后缀添加 `?debug=i18n` 可显示一个小型调试面板，包含：`navigator.language`、`navigator.languages`、算法解析出的 `resolved`、最终 `currentLang`、以及语言包是否加载成功（pack loaded）。
- 如果你发现清空 `localStorage` 后第一次访问仍然不是期望语言：
	- 确认不是用 `file://` 打开（会阻止 JSON 拉取，导致回退到英文基线）；
	- 打开 `?debug=i18n` 检查浏览器语言顺序；
	- 注意我们已修复解析顺序：现在会对浏览器语言列表逐项处理“精确→前缀”，例如 `zh-CN` 将先尝试 `zh-CN`，再尝试 `zh`，然后才会看后续的 `en`；
	- 若你手动选择了语言，页面会设置 `langPinned=1` 并优先使用本地保存的语言（方便固定展示）。


## 隐私与合规说明

- 默认会向以下公共服务发起请求以检测访问者 IP：
	- `https://myip.ipip.net/json`
	- `https://ipinfo.io/json`
	- `https://api.ipify.org?format=json`
- 这些请求会将访问者 IP 暴露给对应第三方服务（它们本就会看到你的出口 IP）。若你对隐私/合规有严格要求，请禁用该功能或改为调用你自建的 IP 服务。
- 请确保对该页面的使用符合当地法律法规以及你的组织安全策略；避免用于误导性或违法用途。


## 常见问题（FAQ）

1) Error 1402 是真实的 HTTP 状态码吗？

- 不是。这里故意使用“非标准码”增强迷惑性。实际状态码由服务器决定；推荐返回常见错误码（如 403/404），前端仍会展示“Error 1402”的文案。

2) Ray ID 是真实可追踪的吗？

- 否。Ray ID 为前端随机生成，仅用于外观伪装，可点击复制提升“拟真度”。

3) 为什么会有外部网络请求？

- 仅用于展示“Your IP Address”模块的多源检测。可按需关闭或替换为自建服务。

4) 是否支持移动端？

- 支持。样式包含简单的响应式规则，可按需调整断点与字号。


## 目录结构

```
.
├─ index.html           # 主页面（内联 CSS/JS，按需拉取语言 JSON）
├─ assets/
│  └─ i18n/             # 按语言代码命名的 JSON 语言包
│     ├─ en.json
│     ├─ zh.json
│     ├─ es.json  …
│     └─ (更多语言)
├─ README.md            # 项目说明（中文）
└─ README.en.md         # 项目说明（英文）
```


## 体积与压缩建议

- 本项目为单文件静态页，默认已较为精简；
- 若追求带宽/时延进一步优化，可：
	- 使用 HTML/CSS/JS 压缩（Minify）；
	- 在服务器启用 Gzip 或 Brotli 压缩；
	- 将第三方 IP 检测移除以减少首屏外部请求。


## 许可证（License）

仓库当前未包含许可证文件。若需开源分发，建议补充一个明确的开源许可证（如 MIT/Apache-2.0）。在未声明许可前，请谨慎进行再分发与商用。


## 致谢 / Credits

- 页面文案与布局灵感来源于常见 CDN 错误页风格；
- IP 查询接口来自公开服务：IPIP、IPInfo、IPify（仅用于演示）。


---
如需进一步定制（多语言、深度拟真、接入你现有监控/日志体系），欢迎根据 `index.html` 自行扩展。
