# DjangoBlog 架构与时序图

> 所有图使用 Mermaid，可在 GitHub、GitLab、Typora、VS Code 中直接渲染。

## 1. 部署拓扑

```mermaid
flowchart LR
    U["User Browser"] --> N["Nginx :80/443"]
    N -->|"proxy_pass"| G["Gunicorn gevent :8000"]
    N -->|"/static /media"| FS["Static Files"]
    G --> DJ["Django App"]
    DJ --> DB[("MySQL")]
    DJ --> RC[("Redis / LocMem")]
    DJ --> WH["Whoosh Index"]
    DJ --> ES["Elasticsearch 8"]
    DJ --> SMTP["SMTP Server"]
    DJ --> OA["OAuth Providers"]
    DJ --> CF["Cloudflare API"]
    DJ --> BD["Baidu Notify"]
    DJ --> WX["WeChat Server"]
```

## 2. 请求生命周期

```mermaid
sequenceDiagram
    participant B as Browser
    participant N as Nginx
    participant G as Gunicorn
    participant MW as Middleware
    participant V as View
    participant C as Cache
    participant DB as MySQL
    participant T as Template

    B->>N: HTTP Request
    N->>G: proxy_pass
    G->>MW: WSGI Request
    MW->>MW: Security / Session / Locale / GZip / CSRF / Auth
    MW->>V: dispatch
    V->>C: get cache key
    alt cache hit
        C-->>V: QuerySet
    else cache miss
        V->>DB: ORM query
        DB-->>V: rows
        V->>C: set cache
    end
    V->>T: render context
    T->>T: load_sidebar / vite_tags / plugin hooks
    T-->>V: HTML
    V-->>MW: Response
    MW->>MW: OnlineMiddleware 替换 LOAD_TIMES
    MW-->>B: HTTP Response
```

## 3. 文章详情页渲染

```mermaid
sequenceDiagram
    participant B as Browser
    participant V as ArticleDetailView
    participant DB as MySQL
    participant H as Hooks
    participant P as Plugins
    participant T as article_detail.html

    B->>V: GET /article/2026/11/27/1.html
    V->>DB: 查询 Article
    V->>DB: 父评论分页 + 子评论预加载
    V->>DB: comment_list() 缓存或查询
    V->>V: 计算 SEO title/description/keywords
    V->>H: run_action(article_detail_load)
    H->>P: article_recommendation
    V->>H: run_action(after_article_body_get)
    H->>P: view_count.viewed()
    P->>DB: UPDATE views = views + 1
    V->>T: render(article, p_comments, next/prev)
    T->>H: 位置钩子 article_top / article_bottom
    T-->>B: HTML
```

## 4. 评论提交与通知

```mermaid
sequenceDiagram
    participant B as Browser
    participant V as CommentPostView
    participant DB as MySQL
    participant S as post_save Signal
    participant C as Cache
    participant M as SMTP
    participant BD as Baidu

    B->>V: POST /article/1/postcomment (body, parent_comment_id)
    V->>V: 登录校验 + CSRF
    V->>DB: 校验文章与评论开关
    V->>DB: INSERT Comment
    V-->>B: 302 /article/...#div-comment-N
    DB-->>S: post_save(Comment)
    S->>C: 删除 article_comments_N / sidebar / seo_processor
    S->>BD: 推送文章 URL
    S->>M: 线程内发送评论通知邮件
    M-->>DB: 写 EmailSendLog
```

## 5. 全文检索

```mermaid
sequenceDiagram
    participant B as Browser
    participant V as EsSearchView
    participant H as Haystack
    participant W as Whoosh
    participant E as Elasticsearch
    participant T as search.html

    B->>V: GET /search?q=Django&page=1
    V->>H: build_form() + highlight()
    alt 默认模式
        H->>W: 查询 whoosh_index
        W-->>H: results
    else 配置 ES
        H->>E: 查询 blog 索引
        E-->>H: results
    end
    H-->>V: page object
    V->>T: render(query, page, paginator)
    T-->>B: HTML（命中词高亮）
```

## 6. OAuth 登录

```mermaid
sequenceDiagram
    participant U as User
    participant A as Django
    participant P as OAuth Provider
    participant DB as MySQL
    participant M as SMTP

    U->>A: GET /oauth/oauthlogin?type=github
    A-->>U: 302 Provider Authorize URL
    U->>P: 授权
    P-->>A: GET /oauth/authorize?type=github&code=...
    A->>P: 用 code 换 access_token
    P-->>A: token + userinfo
    alt 返回邮箱
        A->>DB: 按 email 获取或创建 BlogUser
        A->>DB: 保存/更新 OAuthUser
        A->>A: login() + logged_user cookie
        A-->>U: 302 next_url
    else 无邮箱
        A->>DB: 保存 OAuthUser
        A-->>U: 302 /oauth/requireemail/<id>.html
        U->>A: POST email
        A->>M: 发送确认邮件
        A-->>U: 302 bindsuccess?type=email
        U->>A: GET emailconfirm/<id>/<sign>.html
        A->>DB: 绑定并激活
        A-->>U: 302 bindsuccess?type=success
    end
```

## 7. 图床上传

```mermaid
sequenceDiagram
    participant C as Client
    participant V as fileupload
    participant FS as static/image
    participant PIL as Pillow

    C->>V: POST /upload?sign=<HMAC>
    V->>V: 校验 sign == HMAC(SECRET_KEY, HMAC(SECRET_KEY, SECRET_KEY))
    alt 签名错误
        V-->>C: 403
    else 签名正确
        V->>FS: 按 Y/m/d 建目录并写文件
        V->>PIL: 图片则 quality=20 重压缩
        V-->>C: ['<绝对路径>']
    end
```

## 8. 缓存失效

```mermaid
flowchart TD
    A["Article / Comment / Category / Tag 保存"] --> B{"实例类型"}
    B -->|Article| C["删除 index_1 / category_list_x_1 / tag_x_1 / author_x_1"]
    B -->|Comment| D["删除 article_comments_id"]
    B -->|Category| E["删除 category_list_name_1"]
    B -->|Tag| F["删除 tag_name_1"]
    C --> G["删除 sidebar{i,l,p,a,s}"]
    D --> G
    E --> G
    F --> G
    G --> H["删除 seo_processor"]
    H --> I["百度推送"]
    B --> J["仅 views 更新"] --> K["直接返回，不清理"]
```

## 9. ER 图

```mermaid
erDiagram
    BLOGUSER ||--o{ ARTICLE : writes
    BLOGUSER ||--o{ COMMENT : writes
    BLOGUSER ||--o{ COMMENTREACTION : reacts
    BLOGUSER ||--o{ OAUTHUSER : binds
    CATEGORY ||--o{ ARTICLE : contains
    CATEGORY ||--o{ CATEGORY : parent
    ARTICLE ||--o{ COMMENT : has
    ARTICLE }o--o{ TAG : tagged
    COMMENT ||--o{ COMMENT : replies
    COMMENT ||--o{ COMMENTREACTION : receives

    BLOGUSER {
        int id PK
        string username
        string email
        string nickname
        string source
        datetime creation_time
    }
    ARTICLE {
        int id PK
        string title
        text body
        datetime pub_time
        char status
        char comment_status
        char type
        int views
        int article_order
        bool show_toc
        int author_id FK
        int category_id FK
    }
    CATEGORY {
        int id PK
        string name
        string slug
        int parent_category_id FK
        int index
    }
    TAG {
        int id PK
        string name
        string slug
    }
    COMMENT {
        int id PK
        text body
        int author_id FK
        int article_id FK
        int parent_comment_id FK
        bool is_enable
    }
    COMMENTREACTION {
        int id PK
        int comment_id FK
        int user_id FK
        string reaction_type
    }
    OAUTHUSER {
        int id PK
        int author_id FK
        string openid
        string nickname
        string type
        string email
        string token
        string picture
    }
```

独立配置类实体：

```mermaid
erDiagram
    OAUTHCONFIG {
        int id PK
        string type
        string appkey
        string appsecret
        string callback_url
        bool is_enable
    }
    LINKS {
        int id PK
        string name
        string link
        int sequence
        bool is_enable
        char show_type
    }
    SIDEBAR {
        int id PK
        string name
        text content
        int sequence
        bool is_enable
    }
    BLOGSETTINGS {
        int id PK
        string site_name
        string site_description
        string site_keywords
        int article_sub_length
        int article_comment_count
        bool comment_need_review
        string color_scheme
    }
    EMAILSENDLOG {
        int id PK
        string emailto
        string title
        text content
        bool send_result
    }
```

## 10. 文章状态

```mermaid
stateDiagram-v2
    [*] --> Draft: 新建 status=d
    Draft --> Published: 发布 status=p
    Published --> Draft: 下架
    Published --> [*]: 删除
    Draft --> [*]: 删除
```

评论开关独立：

```mermaid
stateDiagram-v2
    [*] --> Open: comment_status=o
    Open --> Closed: comment_status=c
    Closed --> Open: 重新开启
```

## 11. 插件调用

```mermaid
flowchart LR
    R["DjangoblogAppConfig.ready()"] --> L["loader 读取 ACTIVE_PLUGINS"]
    L --> I["import plugins.x.plugin"]
    I --> P["BasePlugin.__init__"]
    P --> H["register_hooks()"]
    V["View"] -->|"run_action"| H
    T["Template"] -->|"位置钩子"| H
    H --> O["插件输出 HTML / 副作用"]
    O --> T
```

## 12. 容器启动

```mermaid
sequenceDiagram
    participant D as Docker
    participant E as entrypoint.sh
    participant M as manage.py
    participant G as Gunicorn
    participant DB as MySQL

    D->>E: ENTRYPOINT
    E->>M: makemigrations
    E->>M: migrate
    M->>DB: DDL
    E->>M: collectstatic
    E->>E: 校验 dist 与 manifest
    E->>M: compress --force
    E->>M: build_index
    E->>M: compilemessages
    E->>G: exec gunicorn gevent
    G-->>D: 监听 0.0.0.0:8000
```
