# ai/ 文档索引

本目录存放对 DjangoBlog 的静态代码分析产物，可用于内部存档、评审与后续入库。

- 分析基线：当前工作区 `master` 分支代码（Django 5.2.18 / Python 3.10+）
- 分析方式：路由、视图、模型、模板、配置、部署脚本交叉核对
- 局限：未做运行期压测与全量功能回归；示例中的域名、账号、密钥均为占位符

## 文件说明

| 文件 | 内容 | 建议入库位置 |
| --- | --- | --- |
| `analysis.md` | 完整分析（15 章，含接口全量清单、页面说明、29 条技术债） | 内部存档，不入库 |
| `technical.md` | 可直接入库的主技术文档（架构、模型、接口、插件、部署） | `docs/technical.md` |
| `api-examples.md` | 全部接口的 curl 示例与请求/响应样例 | `docs/api-examples.md` |
| `diagrams.md` | 部署图、时序图、ER 图、状态图（Mermaid） | `docs/diagrams.md` |

## 建议阅读顺序

1. `technical.md` 第 1～3 章：先建立整体架构认知
2. `diagrams.md` 图 1～3：看清请求链路与部署形态
3. `technical.md` 第 4～6 章：数据模型、接口与页面
4. `api-examples.md`：联调时按接口查示例
5. `analysis.md` 第 14 章：排查与重构前先读风险清单

## 迁移到 docs/

```bash
mkdir -p docs
cp ai/technical.md docs/technical.md
cp ai/api-examples.md docs/api-examples.md
cp ai/diagrams.md docs/diagrams.md
```

迁移后请把 `technical.md` 中的相对链接改为 `./api-examples.md`、`./diagrams.md`（当前已是相对路径，可直接使用）。

## 维护约定

- 修改路由、模型字段、环境变量后，同步更新 `technical.md` 第 4、5、10 章
- 新增接口时，在 `api-examples.md` 追加一条示例，并在 `analysis.md` 第 5 章登记
- 文档中不写入真实 `SECRET_KEY`、数据库口令、OAuth AppSecret、OpenAI Key