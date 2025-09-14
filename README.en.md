# Error 1402 – Fake Error Page (Static Single Page)

Language: [简体中文](./README.md) | English

## Screenshots

English

![Error 1402 - English](./assets/screenshot-en.png)

Chinese

![Error 1402 - Chinese](./assets/screenshot-zh.png)

A static error page template intended to mask or cover real services. It mimics a CDN-style error page (sample copy styled as "EURUN CDN"), featuring a Ray ID, UTC timestamp, and multi-source IP detection. It helps reduce real service exposure or provide a uniform error appearance when access is restricted.

> Note: "Error 1402" is not a standard HTTP status code. It's intentionally ambiguous. Your actual HTTP status code should be set by your server/reverse proxy (commonly 403 or 404).


## What is this

- Purpose: Provide a frontend-only static page that looks like a CDN/security device error page, to obscure backend apps or serve as a unified landing page for restricted access.
- Form: Single-file `index.html` with inline CSS/JS, no external dependencies.
- Highlights:
  - Pseudo Ray ID (click to copy) and UTC timestamp;
  - Concurrent IP detection from three public sources (IPIP, IPInfo, IPify) with success/failure indication;
  - Responsive layout for desktop and mobile;
  - Easily replaceable branding copy, company info, contact email;
  - Clean code and easy customization.


## Quick start (Deploy)

You can host `index.html` on any static hosting or web server:

- Static hosting: GitHub Pages / Cloudflare Pages / Vercel / Netlify, etc.
- Traditional servers: Nginx / Apache / Caddy — use it as site root or default page for a specific path.
- Reverse proxy fronting: as a fallback/default page or error page (e.g., 403/404) response.

Recommended scenarios:
- As the default site for unknown/unconfigured hostnames.
- As the error page for 403/404.
- As a temporary placeholder during maintenance or cutover.

> Tip: For obfuscation, the page title/copy shows "Error 1402", while your server can still return common status codes (e.g., 403/404/503). The look and feel is preserved.


## Reverse proxy/Nginx idea (pseudo)

- Put `index.html` in a readable directory;
- Map 403/404 errors to this file;
- Or return this file as the default for a fallback server block.

Adjust for production needs (TLS, hostname matching, caching policy, etc.).


## Ready-to-copy server configs

All examples use Linux-style paths. Replace `/var/www/error1402` with your actual directory.

### Nginx

1) Map to error page (preserve real status, recommended)

```nginx
server {
    listen 80;
    server_name example.com;

    # your normal site config ...

    # replace body for 403/404/503 with fake page, keep status code
    error_page 403 /error-1402.html;
    error_page 404 /error-1402.html;
    error_page 503 /error-1402.html;

    # Option A: a separate file name to avoid index conflicts
    location = /error-1402.html {
        root /var/www/error1402;   # directory contains index.html (you can copy/rename)
        internal;                  # only for internal redirects via error_page
        default_type text/html;
        add_header Cache-Control "no-store";
    }
    # Option B: reuse index.html directly (choose one)
    # location = /error-1402.html {
    #     alias /var/www/error1402/index.html;
    #     internal;
    #     default_type text/html;
    #     add_header Cache-Control "no-store";
    # }
}
```

2) As default/fallback site (for unmatched hostnames)

Return 404 but render fake page body:

```nginx
server {
    listen 80 default_server;
    server_name _;

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

Serve with 200 directly (no error code):

```nginx
server {
    listen 80 default_server;
    server_name _;
    root /var/www/error1402;

    location / {
        try_files /index.html =404;  # always serves index.html with 200
        add_header Cache-Control "no-store";
    }
}
```

### Apache (httpd)

1) Map to error page (preserve real status)

Place the fake page file under your site DocumentRoot (name it `error-1402.html` or reuse `index.html`).

```apache
# inside vhost or .htaccess
ErrorDocument 403 /error-1402.html
ErrorDocument 404 /error-1402.html
ErrorDocument 503 /error-1402.html

# optional: headers for the directory
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

2) Default/fallback vhost (keep 403 or 404)

Example: always return 403 with the fake page body.

```apache
<VirtualHost *:80>
    ServerName _default_
    DocumentRoot "/var/www/error1402"

    # always return 403 (Forbidden)
    <IfModule mod_rewrite.c>
        RewriteEngine On
        RewriteRule ^ - [F]
    </IfModule>

    # use fake page as 403 body
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

To return 404 instead:

```apache
RewriteRule ^ - [R=404]
ErrorDocument 404 /index.html
```

### Caddy v2

1) Map to error page (preserve real status)

```caddyfile
example.com {
    # your normal site ...

    handle_errors {
        @4xx expression {http.error.status_code >= 400 && http.error.status_code < 500}
        rewrite @4xx /index.html
        file_server
    }
}
```

2) Default/fallback site

Keep error code example (403):

```caddyfile
:80 {
    root * /var/www/error1402
    respond / 403

    handle_errors {
        rewrite * /index.html
        file_server
    }
}
```

Serve 200 directly:

```caddyfile
:80 {
    root * /var/www/error1402
    try_files /index.html
    file_server
}
```

> Heads-up: TLS is omitted in samples. For production, add HTTPS, HSTS, caching policy, and proper access control.


## Customization & development

Edit `index.html` directly:

- Branding & copy:
  - Replace `<title>`, headings, subheadings ("What happened? / What can I do?"), footer company info, and email.
- Error code & messages:
  - Change "Error 1402" to your own label, or use common ones like "403 Forbidden".
- Ray ID & timestamp:
  - Generated on the client only (for realism). Ray ID is clickable to copy.
- IP detection:
  - See `ipServices` array; add/remove sources as needed.
  - 3 seconds timeout per source, requests run concurrently.
  - If you don't want outbound requests, empty `ipServices` or comment out `fetchUserIp()` and remove the UI.
- Styles:
  - Inline CSS; adjust colors/layout/typography as needed.
  - Responsive tweaks in `@media (max-width:768px)`.


### Language switch (EN/ZH)

- Language selector on the top-right supports English and Chinese.
- Default language selection:
  - Use `localStorage.lang` if set;
  - Otherwise, detect from `navigator.language` (Chinese if starts with `zh`, else English).
- Translatable keys are defined in the inlined `i18n` object:
  - `title`, `error_code`, `ray_prefix`, `error_description`
  - `ip_title`, `detecting`, `ip_source_failed`, `copied`
  - `what_happened`, `what_happened_p1`, `what_happened_p2`
  - `what_can_i_do`, `what_can_i_do_p1`, `what_can_i_do_p2`
  - `footer_line1`, `footer_line2`, `lang_en`, `lang_zh`, `sep`
- To add a new language:
  1. Add a language object (e.g., `jp`) in `i18n` and fill the keys above.
  2. Add an `<option>` to the selector for the new language.
  3. Provide a proper `sep` (e.g., `：` for CJK) if needed.
  4. For new paragraphs/modules, add `data-i18n="key"` to elements and define the key in the dictionary.


## Privacy & compliance

- The page queries the following public services to detect the visitor's IP:
  - `https://myip.ipip.net/json`
  - `https://ipinfo.io/json`
  - `https://api.ipify.org?format=json`
- These calls share the visitor's IP with those providers (which they would see anyway as your egress IP). If privacy/compliance is critical, disable this feature or use your own IP service.
- Ensure your usage complies with local laws and your org’s security policies; avoid deceptive or illegal use.


## FAQ

1) Is "Error 1402" a real HTTP status code?

- No. It's intentionally non-standard for obfuscation. Your server sets the actual HTTP status (recommend 403/404). The page still shows the "Error 1402" label.

2) Is the Ray ID actually traceable?

- No. It's randomly generated on the client for realism only, but is copyable.

3) Why are there external requests?

- Only for the "Your IP Address" module. Disable or replace with your own service if needed.

4) Is mobile supported?

- Yes. There's a simple responsive layout that you can tweak as needed.


## Structure

```
.
├─ index.html   # main page (inline CSS/JS)
└─ README.md    # project intro (Chinese) — see also README.en.md
```


## Size & compression tips

- Single-file static page, already quite compact.
- Further optimize by:
  - HTML/CSS/JS minification;
  - Enabling Gzip or Brotli on server;
  - Removing external IP checks to reduce first paint requests.


## License

No license file is included yet. If you plan to open source it, add an explicit license (e.g., MIT/Apache-2.0). Until then, be cautious with redistribution and commercial use.


## Credits

- Layout and copy inspired by common CDN error pages;
- IP services: IPIP, IPInfo, IPify (demo only).

---
For deeper customization (multi-language extension, stronger mimicking, or integration with your monitoring/logging), feel free to extend `index.html`.
