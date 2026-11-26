# DjangoBlog 接口示例

> 默认地址：`http://127.0.0.1:8000`
> 默认语言为简体中文，无 URL 前缀；英文为 `/en/...`，繁体为 `/zh-hant/...`
> 所有示例中的账号、文章 ID、签名均为占位符，请按实际数据替换

## 0. 通用准备

```bash
export BASE=http://127.0.0.1:8000
export JAR=/tmp/djangoblog.cookies
```

获取 CSRF Token（表单页均含 `csrfmiddlewaretoken`）：

```bash
csrf() {
  curl -s -c "$JAR" "$1" \
    | grep -o 'name="csrfmiddlewaretoken" value="[^"]*"' \
    | head -1 | sed 's/.*value="//;s/"$//'
}
```

JSON 接口使用请求头：

```bash
-H "X-CSRFToken: $TOKEN" -H "Referer: $BASE/"
```

## 1. 健康检查

```bash
curl -s $BASE/health/
```

```json
{"status": "healthy", "timestamp": 1795680000.123}
```

## 2. 内容浏览

### 2.1 首页与分页

```bash
curl -s -o /dev/null -w "%{http_code}\n" $BASE/
curl -s -o /dev/null -w "%{http_code}\n" $BASE/page/2/
```

响应为 HTML（200），关键内容：文章卡片、分页、侧边栏。

### 2.2 文章详情

```bash
curl -s "$BASE/article/2026/11/27/1.html" | head -40
```

- 200：返回详情页
- 404：文章不存在（`get_object_or_404`）

### 2.3 分类 / 作者 / 标签

```bash
curl -s "$BASE/category/django.html"
curl -s "$BASE/category/django/2.html"
curl -s "$BASE/author/admin.html"
curl -s "$BASE/tag/python.html"
curl -s "$BASE/tag/python/2.html"
```

### 2.4 归档与友链

```bash
curl -s "$BASE/archives.html"
curl -s "$BASE/links.html"
```

`/archives.html` 由 `cache_page(3600)` 包裹，响应头含 `Cache-Control: max-age=3600`。

## 3. 搜索

```bash
curl -s "$BASE/search?q=Django&page=1" | head -60
```

- 参数：`q` 关键词、`page` 页码
- 响应：HTML 结果页，命中词带高亮
- 未建索引时可能返回空结果，先执行 `python manage.py rebuild_index`（Whoosh）

## 4. RSS 与 Sitemap

```bash
curl -s $BASE/feed/ | head -20
curl -s $BASE/rss/  | head -20
curl -s $BASE/sitemap.xml | head -40
```

RSS 返回 `application/rss+xml`，Sitemap 返回 `application/xml`。

## 5. 语言切换

```bash
TOKEN=$(csrf "$BASE/")
curl -s -i -b "$JAR" -c "$JAR" -e "$BASE/" \
  -d "csrfmiddlewaretoken=$TOKEN" \
  -d "language=zh-hans" \
  -d "next=/" \
  "$BASE/i18n/setlang/"
```

```
HTTP/1.1 302 Found
Location: /
Set-Cookie: django_language=zh-hans; ...
```

语言可选值：`en`、`zh-hans`、`zh-hant`。

## 6. 认证

### 6.1 注册

```bash
TOKEN=$(csrf "$BASE/register/")
curl -s -i -b "$JAR" -c "$JAR" -e "$BASE/register/" \
  -d "csrfmiddlewaretoken=$TOKEN" \
  -d "username=demo" \
  -d "email=demo@example.com" \
  -d "password1=DemoPass123!" \
  -d "password2=DemoPass123!" \
  "$BASE/register/"
```

```
HTTP/1.1 302 Found
Location: /account/result.html?type=register&id=12
```

此时用户 `is_active=False`，系统发送激活邮件。激活链接形如：

```
http://<site>/account/result.html?type=validation&id=12&sign=<hmac>
```

```bash
curl -s "$BASE/account/result.html?type=validation&id=12&sign=<hmac>"
```

- 200：HTML 提示“验证成功”
- 403：签名不匹配

### 6.2 登录

```bash
TOKEN=$(csrf "$BASE/login/")
curl -s -i -b "$JAR" -c "$JAR" -e "$BASE/login/" \
  -d "csrfmiddlewaretoken=$TOKEN" \
  -d "username=demo" \
  -d "password=DemoPass123!" \
  -d "remember=on" \
  -d "next=/" \
  "$BASE/login/"
```

```
HTTP/1.1 302 Found
Location: /
Set-Cookie: logged_user=true; Max-Age=2626560; Path=/; SameSite=Lax
Set-Cookie: sessionid=...; ...
```

- `remember=on`：30 天（`REMEMBER_ME_LOGIN_TTL`）
- 不传 `remember`：2 周（`SESSION_COOKIE_AGE`）
- 支持用户名或邮箱登录（`EmailOrUsernameModelBackend`）

### 6.3 登出

```bash
curl -s -i -b "$JAR" -c "$JAR" "$BASE/logout/"
```

```
HTTP/1.1 302 Found
Location: /login/
Set-Cookie: logged_user=""; expires=...
```

### 6.4 找回密码

发送验证码：

```bash
curl -s -X POST -b "$JAR" -c "$JAR" -e "$BASE/forget_password/" \
  -d "email=demo@example.com" \
  "$BASE/forget_password_code/"
```

```
ok
```

提交新密码：

```bash
TOKEN=$(csrf "$BASE/forget_password/")
curl -s -i -b "$JAR" -c "$JAR" -e "$BASE/forget_password/" \
  -d "csrfmiddlewaretoken=$TOKEN" \
  -d "email=demo@example.com" \
  -d "code=123456" \
  -d "new_password1=NewPass123!" \
  -d "new_password2=NewPass123!" \
  "$BASE/forget_password/"
```

```
HTTP/1.1 302 Found
Location: /login/
```

验证码缓存 5 分钟；邮箱不存在时表单报错。

## 7. 评论与反应

### 7.1 提交评论（需登录）

```bash
TOKEN=$(csrf "$BASE/login/")
curl -s -i -b "$JAR" -c "$JAR" -e "$BASE/login/" \
  -d "csrfmiddlewaretoken=$TOKEN" -d "username=demo" -d "password=DemoPass123!" \
  "$BASE/login/"

TOKEN=$(csrf "$BASE/article/2026/11/27/1.html")
curl -s -i -b "$JAR" -c "$JAR" -e "$BASE/article/2026/11/27/1.html" \
  -d "csrfmiddlewaretoken=$TOKEN" \
  -d "body=这是一条测试评论" \
  -d "parent_comment_id=" \
  "$BASE/article/1/postcomment"
```

```
HTTP/1.1 302 Found
Location: /article/2026/11/27/1.html#div-comment-8
```

回复评论时传 `parent_comment_id=7`。若站点开启审核，评论 `is_enable=False`，前台不展示。

### 7.2 获取反应统计（公开）

```bash
curl -s "$BASE/comment/8/react"
```

```json
{
  "success": true,
  "reactions": {
    "👍": {"count": 2, "has_reacted": false, "users": ["alice", "bob"]},
    "❤️": {"count": 1, "has_reacted": false, "users": ["carol"]}
  }
}
```

只统计 `is_enable=True` 的评论；未登录时 `has_reacted` 恒为 false。

### 7.3 切换反应（需登录）

```bash
TOKEN=$(csrf "$BASE/")
curl -s -b "$JAR" -c "$JAR" -e "$BASE/" \
  -H "X-CSRFToken: $TOKEN" \
  --data-urlencode "reaction_type=👍" \
  "$BASE/comment/8/react"
```

```json
{
  "success": true,
  "action": "added",
  "reactions": {
    "👍": {"count": 3, "has_reacted": true, "users": ["alice", "bob", "demo"]}
  }
}
```

再次提交相同表情：

```json
{"success": true, "action": "removed", "reactions": {"👍": {"count": 2, "has_reacted": false, "users": ["alice", "bob"]}}}
```

错误响应：

| 场景 | 状态码 | 响应 |
| --- | --- | --- |
| 未登录 | 401 | `{"success": false, "error": "Authentication required"}` |
| 表情非法 | 400 | `{"error": "Invalid reaction type"}` |
| 评论不存在或未审核 | 404 | HTML 404 |

合法表情：👍 👎 ❤️ 😄 🎉 😕 🚀 👀

## 8. OAuth 登录

### 8.1 发起授权

```bash
curl -s -i "$BASE/oauth/oauthlogin?type=github&next_url=/"
```

```
HTTP/1.1 302 Found
Location: https://github.com/login/oauth/authorize?client_id=...&redirect_uri=...&state=...
```

`type` 可选：`weibo`、`google`、`github`、`facebook`、`qq`。`next_url` 会做同源校验。

### 8.2 授权回调

在平台后台把回调地址配置为：

```
https://<your-domain>/oauth/authorize?type=github
```

平台重定向后：

```bash
curl -s -i "$BASE/oauth/authorize?type=github&code=<code>"
```

分支结果：

| 条件 | 结果 |
| --- | --- |
| 已绑定邮箱 | 302 到 `next_url`，同时登录并设置 `logged_user` |
| 平台未返回邮箱 | 302 到 `/oauth/requireemail/<id>.html` |
| code 无效 | 302 回首页或重新授权 |

### 8.3 补录邮箱

```bash
TOKEN=$(csrf "$BASE/oauth/requireemail/3.html")
curl -s -i -b "$JAR" -c "$JAR" -e "$BASE/oauth/requireemail/3.html" \
  -d "csrfmiddlewaretoken=$TOKEN" \
  -d "email=demo@example.com" \
  -d "oauthid=3" \
  "$BASE/oauth/requireemail/3.html"
```

```
HTTP/1.1 302 Found
Location: /oauth/bindsuccess/3.html?type=email
```

邮件内的确认链接：

```
https://<your-domain>/oauth/emailconfirm/3/<sign>.html
```

访问后校验签名、绑定用户并登录，跳转 `/oauth/bindsuccess/3.html?type=success`。

## 9. 图床上传

签名计算：

```bash
SIGN=$(python manage.py shell -c "
from djangoblog.utils import get_sha256
from django.conf import settings
print(get_sha256(get_sha256(settings.SECRET_KEY)))
")
```

上传：

```bash
curl -s -X POST "$BASE/upload?sign=$SIGN" \
  -F "file=@/tmp/demo.png"
```

响应为 Python 列表字符串（不是 JSON）：

```
['/Users/meiling/PycharmProjects/DjangoBlog/static/image/2026/11/27/6f1c....png']
```

注意：

- 图片按文件名是否含 `jpg/png/jpeg/bmp` 判断类型，非图片可通过 `files` 目录上传任意扩展名
- 图片会以 `quality=20` 重新压缩
- 该目录不在 `STATICFILES_DIRS`/collectstatic 范围，返回的字符串不能直接作为可访问 URL

错误：

| 场景 | 响应 |
| --- | --- |
| 缺少 `sign` 或错误 | 403 Forbidden |
| GET 请求 | 200，文本 `only for post` |

## 10. 缓存清理

```bash
curl -s "$BASE/clean"
```

```
ok
```

该接口无鉴权，会执行 `cache.clear()`，生产环境应限制访问。

## 11. 微信机器人

服务器地址配置为：

```
https://<your-domain>/robot
```

验证（GET）：

```bash
curl -s "$BASE/robot?signature=<sig>&timestamp=<ts>&nonce=<nonce>&echostr=hello"
```

签名正确时原样返回 `echostr`。

消息（POST，XML）：

```bash
curl -s -X POST "$BASE/robot" \
  -H "Content-Type: text/xml" \
  --data-binary @/tmp/wechat.xml
```

返回 XML 文本消息。Token 需与公众号后台一致。

## 12. 后台

```bash
curl -s -o /dev/null -w "%{http_code}\n" "$BASE/admin/"
```

未登录返回 302 到 `/admin/login/?next=/admin/`；登录后仅 superuser 可访问。

## 13. 错误响应汇总

| 状态码 | 触发方式 | 示例 |
| --- | --- | --- |
| 302 | 登录、评论、语言切换、OAuth | `Location: /` |
| 400 | `reaction_type=abc` | `{"error": "Invalid reaction type"}` |
| 401 | 未登录 POST 反应 | `{"success": false, "error": "Authentication required"}` |
| 403 | CSRF 缺失、图床签名错误、激活签名错误 | `CSRF verification failed` |
| 404 | 文章/评论不存在 | 自定义 404 页 |
| 500 | 评论关闭、图床文件无法作为图片打开 | 自定义 500 页 |

## 14. 联调提示

- 所有 POST 表单都需要 CSRF；先 GET 页面拿 cookie 与 token
- `Referer` 在 HTTPS 下会被用于 Origin 校验，建议始终带上
- 分页参数统一为 `page`；评论区使用 `comment_page`
- `search` 路由是前缀匹配（`^search`），`/searchfoo` 也会命中
- 评论关闭时会抛异常返回 500，联调时应先确认 `comment_status='o'`
