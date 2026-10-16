# auth-service

统一认证与多端会话管理服务。基于 **Java 17 + Spring Boot 3 + Spring Security 6 + MySQL 8 + Redis 7**，实现**多点登录**：同一账号可在多台设备同时在线，各端会话相互独立、可单独下线。

> **当前状态：设计阶段**
> 设计计划书已完成并通过结构校验，应用代码尚未开发。下一里程碑为工程脚手架与核心链路实现。

---

## 文档

| 文档 | 说明 |
| --- | --- |
| [设计计划书（Markdown）](docs/登录功能-多点登录-设计计划书.md) | 架构、数据模型、接口、时序、安全、测试、Git 流程与排期 |
| [设计计划书（Word）](docs/登录功能-多点登录-设计计划书.docx) | 同一内容的排版版本，用于评审与归档 |
| [ADR 目录](docs/adr/) | 架构决策记录 |

重新生成 Word 版：

```bash
python tools/build_docx.py
```

---

## 核心设计要点

| 主题 | 方案 |
| --- | --- |
| 登录语义 | **多点登录**：一次登录 = 一台设备 = 一条 `t_user_session` 记录，新登录不影响既有会话 |
| 令牌体系 | 短期 JWT（Access Token，30 分钟）+ 服务端存储的不透明 Refresh Token（滑动 30 天） |
| 主动撤销 | 撤销会话后写 Redis 黑名单 `auth:revoked:sid:{sid}`，TTL = Access Token 剩余寿命 |
| 刷新安全 | Refresh Token 每次刷新轮换，库中只存 SHA-256 哈希；旧令牌被重放即撤销整个会话并告警 |
| 密码存储 | Argon2id（`password_algo` 字段支持渐进式升级） |
| 多端管理 | 在线设备列表、按设备下线、全端下线、设备数上限与淘汰策略 |

完整论证见[设计计划书](docs/登录功能-多点登录-设计计划书.md)第 2.3 节「关键设计决策」。

---

## 计划中的工程结构

```
auth-service/
├── docs/                     # 设计文档与 ADR
├── tools/                    # 文档生成等辅助脚本
├── deploy/                   # docker-compose / Dockerfile / k8s 模板
├── src/main/java/com/example/auth/
│   ├── api/                  # Controller、DTO、异常处理
│   ├── application/          # 用例编排与事务边界
│   ├── domain/               # 实体、值对象、领域服务（不依赖框架）
│   ├── infrastructure/       # JPA 仓储、Redis、邮件短信、审计
│   ├── security/             # SecurityConfig、JwtAuthFilter
│   └── config/
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/         # Flyway 迁移脚本
└── src/test/java/            # unit / integration / api
```

分层依赖规则：`api → application → domain ← infrastructure`，由 ArchUnit 在 CI 中断言。

---

## 本地开发（代码就绪后生效）

```bash
# 1. 准备环境变量
cp .env.example .env        # 填入本地 MySQL / Redis 连接信息

# 2. 启动依赖组件
docker compose -f deploy/docker-compose.yml up -d

# 3. 启动服务
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# 4. 接口文档
open http://localhost:8080/swagger-ui/index.html
```

---

## 贡献约定

- 分支：`feature/*`、`fix/*`、`chore/*`，生命周期不超过 3 天
- 提交信息：遵循 [Conventional Commits](https://www.conventionalcommits.org/)，例如 `feat(session): 支持按设备单独撤销会话`
- 合并：必须通过 PR，CI 全绿，至少 1 名评审人
- **密钥纪律**：任何真实密钥、口令、令牌一律不入库；开发用 `.env`（已忽略），生产用 K8s Secret / 云 KMS

完整流程见[设计计划书](docs/登录功能-多点登录-设计计划书.md)第 12 章「代码库与 Git 工作流」。

---

## 安全

本服务处理用户凭证，安全问题请勿通过公开 Issue 反馈。发现漏洞请联系维护者私下沟通。
