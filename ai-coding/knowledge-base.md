# MDP 规则仓库结构设计—工程级知识库的落地思考

## 前言

在 AI Coding 已经成为工程常态的今天，一个悖论正在浮出水面：AI 生成代码的速度越来越快，但生成代码的"质量一致性"却没有相应提升。根本原因不在于模型能力，而在于 AI 缺少足够的项目级约束——它不知道"这个团队认为什么是好代码"。

MDP 规则仓库的出现，正是为了解决这个问题。它不是一份技术文档，而是一套写给 AI 代理读的工程规范体系。本文将对这套结构进行深度分析，探讨它背后的设计哲学、落地路径以及与 wiki-architect-deep 知识库体系的协同关系。

---

## 第一章：对 MDP 规则仓库结构的深度分析

### 1.1 结构还原分析

在正式分析之前，先将用户提出的仓库结构完整呈现：

```Plain Text
知识库结构 (MDP 规则仓库)
├── AGENTS.md                          # AI 代理认知框架加载器（自动生成）
└── .mdp/                              # 规则仓库主目录
    ├── context/                       # 业务上下文文档目录
    │   └── (自定义业务知识文档)
    ├── rules/                         # 编码规范文档目录
    │   ├── company/                   # 公司级规则
    │   │   └── java/                  # Java 编码规范（11 个文件）
    │   ├── mdp/                       # MDP 框架规范
    │   └── template/                  # 文档模板
    ├── team/                          # 团队级规则
    └── project/                       # 项目级规则
```

这个结构看起来是一个普通的文档目录，但仔细分析会发现三个关键设计决策。

**第一个决策：AGENTS.md 放在根目录**

AGENTS.md 不是放在`.mdp/`里，而是放在仓库根目录。这个位置选择是有意为之的——根目录是 Claude Code、Cursor 等 AI 工具默认加载 context 的位置。把 AGENTS.md 放在这里，意味着每次 AI 代理启动会话时，这个文件会被自动注入到系统提示中。它是 AI 代理的"认知框架加载器"，而不是给人类阅读的索引文档。

**第二个决策：context/ 与 rules/ 的分离**

`.mdp/context/`存放业务上下文知识（这是什么业务、为什么这么设计、核心流程是什么），`.mdp/rules/`存放编码规范（代码应该怎么写、禁止什么、必须什么）。这两类知识性质完全不同：context 是"描述性知识"（descriptive），rules 是"规约性知识"（normative）。把它们分开，让 AI 代理能够精确控制加载范围——在需要理解业务背景时加载 context，在需要生成规范代码时加载 rules。

**第三个决策：三级层次结构**

`company → team → project`是经典的企业软件知识治理模式，与 ESLint 的`extends`机制同构。公司级规则是所有团队必须遵守的基线，团队级规则是团队的特定规范（可以在公司级基础上扩展或覆盖），项目级规则是项目特有的约束（优先级最高）。这种层次结构天然支持"知识复用 + 局部定制"，适合大规模组织的知识治理。

### 1.2 设计思维解读：规则优先于知识

这个结构体现了一个核心设计理念：**知识库的核心不是代码描述，而是约束和规范**。

传统的技术文档体系（README、Wiki）是以"描述"为中心的——告诉人类代码是什么、如何工作。而 MDP 规则仓库是以"规约"为中心的——告诉 AI 代理代码应该怎么写。

这个转变背后有一个深刻的工程洞察：人类开发者可以通过代码审查来纠正 AI 生成的错误规范代码，但 AI 代理本身无法自我纠正，除非它从一开始就被注入了正确的规范约束。换言之，规则仓库是 AI 代理的"内在标准"，而不是外部检查器。

### 1.3 MDP 规则仓库与 wiki-architect-deep 的本质差异

两者常被混淆，但它们解决的是完全不同的问题：

| 维度 | wiki-architect-deep 输出 | MDP 规则仓库 |
|---|---|---|
| 知识来源 | 从代码仓库自动提取 | 人工编写规范（或规范人工审核） |
| 知识类型 | 描述性（What & How it is） | 规约性（Should & Must） |
| 核心问题 | 代码现在是怎么写的？ | 代码应该怎么写？ |
| 更新触发 | 代码变更后重新执行 Skill | 规范变更时人工更新 |
| 主要消费者 | AI 代理理解代码上下文 | AI 代理生成符合规范的代码 |
| 文件位置 | `./knowledge-docs/` | `.mdp/` |
| 核心索引文件 | `knowledge.yml` | `AGENTS.md` |
| 内容时效性 | 与代码同步（自动） | 依赖人工维护（存在滞后风险） |
| 适合加载时机 | 理解现有代码时 | 生成新代码时 |

**核心洞见**：规则仓库回答"代码应该怎么写"，知识库回答"代码现在是怎么写的"。这两个问题在人类开发者的脑子里是混在一起的，但在 AI 代理的上下文加载中，必须被分开处理——否则 AI 会把"现在的写法"当成"应该的写法"，永久固化代码债务。

---

## 第二章：这种结构的设计优势

### 2.1 优势一：关注点分离清晰

`context/`与`rules/`的分离不只是目录组织上的整洁，它在语义层面实现了"业务语义"与"编码规范"的解耦。

在没有这种分离的情况下（例如把业务描述和规范混在一个文档里），会出现以下问题：
- AI 代理在加载"如何调用支付接口"的业务知识时，被迫同时加载大量 Java 编码规范，浪费上下文
- 业务知识更新（新增一个支付渠道）和规范更新（修改日志格式要求）被耦合在同一个文件里，变更历史混乱
- 无法针对不同场景按需加载：有时只需要业务上下文，有时只需要编码规范

通过`context/`和`rules/`的分离，AGENTS.md 可以精确控制加载策略：在代码生成场景加载全量`rules/`，在业务分析场景加载全量`context/`，在混合场景按模块精确加载。

### 2.2 优势二：层次继承与覆盖

`company → team → project`的三级层次是这个结构最有工程价值的设计。参考 ESLint 的 extends 机制，这种层次结构支持：
- **公司级规则**：全公司必须遵守的技术规范（Java 命名规范、SQL 注入防护、日志格式等），由架构委员会或平台团队维护
- **团队级规则**：团队特定的规范（例如某团队用 Prometheus 监控而非 Falcon，或者某团队有特定的模块拆分规范），继承公司级规则并在其基础上扩展
- **项目级规则**：项目特有的约束（例如特定的配置中心路径、特定的数据库分片规则），优先级最高

这种层次结构解决了大规模组织中"规范复用 vs 局部定制"的经典矛盾：不需要每个项目都从头写规范，但也不会因为强制统一而丧失灵活性。

### 2.3 优势三：面向 AI 代理而设计

AGENTS.md 是这个结构的灵魂。它不是传统意义上的 README 或技术文档——它的设计目标是"在一次上下文加载中，让 AI 代理建立对项目的正确认知框架"。

这意味着 AGENTS.md 的写作原则与传统文档完全不同：
- **不需要背景介绍**：AI 不需要知道公司历史，它需要知道"遇到这类情况，该怎么处理"
- **规则优先于描述**：不是"我们用 Spring Boot"，而是"必须用 Spring Boot 3.x，禁止降级到 2.x"
- **精确的文件引用**：不是"参考 Java 规范"，而是"`.mdp/rules/company/java/exception-handling.md`"
- **明确的优先级标注**：区分"必读"（每次代码生成都要参考）和"按需参考"（特定场景才加载）

### 2.4 优势四：版本控制友好

整个`.mdp/`目录放在 Git 仓库中，带来了三个工程价值：

**变更历史可追溯**：每次规范更新都有 commit，可以精确查看"这条规则是什么时候加的""为什么加""谁加的"。

**PR Review 保证质量**：规范的新增和修改需要经过代码审查，防止个人偏好被写进团队规范。这是把规范治理纳入工程流程的关键。

**分支隔离实验**：可以在 feature branch 上试验新规范，在主分支稳定后合并，而不会影响 AI 代理在主分支上的行为。

### 2.4.1 规则层次继承关系（Mermaid）

```Mermaid
flowchart TD
    A["company/ 公司级规则<br/>（架构委员会维护）"] --> B["team/ 团队级规则<br/>（团队负责人维护）"]
    B --> C["project/ 项目级规则<br/>（项目 TL 维护）"]
    C --> D["AGENTS.md<br/>（自动生成索引）"]
    D --> E["AI 代理<br/>（Claude Code / Cursor）"]
    E --> F["代码生成<br/>（符合规范）"]
    
    A -.->|"继承"| B
    B -.->|"继承 + 可覆盖"| C
    C -.->|"最高优先级"| D
    
    G["context/ 业务上下文"] --> D
    H["wiki-architect-deep 产物"] -.->|"转化"| G
    
    style A fill:#e8f4f8,stroke:#2196F3
    style B fill:#e8f5e9,stroke:#4CAF50
    style C fill:#fff3e0,stroke:#FF9800
    style D fill:#fce4ec,stroke:#E91E63
    style E fill:#f3e5f5,stroke:#9C27B0
    style F fill:#e0f2f1,stroke:#009688
```

---

## 第三章：如何用 wiki-architect-deep 生成这种结构的内容

### 3.1 直接对应关系

wiki-architect-deep 是一个 15 Phase 的文档架构师 Skill，能从代码仓库中自动提取知识，生成`./knowledge-docs/`下的系列文档。这些文档与 MDP 规则仓库中的目标位置有明确的对应关系：

| wiki-architect-deep 产物 | MDP 规则仓库目标位置 | 转化说明 |
|---|---|---|
| `deep-dive/architecture/decisions.md` | `.mdp/context/architecture-decisions.md` | 直接复制，人工补充 "therefore we do X" 的行动指导 |
| `deep-dive/architecture/constraints.md` | `.mdp/rules/project/dev-constraints.md` | 将描述性约束转化为规则格式（"禁止 X""必须 Y"） |
| `deep-dive/modules/<module>/overview.md` | `.mdp/context/<module>.md` | 直接作为业务上下文，按模块组织 |
| `knowledge.yml` | 用于自动生成`AGENTS.md` | 提取技术栈、项目快照，脚本化生成索引 |
| `learning-path/` | `.mdp/context/onboarding-path.md` | 新人上手路径作为业务上下文 |
| `user-stories/<module>/<feature>.md` | `.mdp/context/user-stories/<module>.md` | 核心业务流程作为上下文参考 |
| `deep-dive/architecture/project-health.md` | `.mdp/context/tech-debt.md` | 技术债务现状作为上下文（不是规范） |

### 3.2 转化的核心原则

这里有一个经常被忽视的关键点：**wiki-architect-deep 的产物是"描述性知识"，不是"规约性知识"**。

"描述性知识"告诉我们现状："项目中使用了 Spring Boot 2.7，事务管理采用`@Transactional`注解"。

"规约性知识"告诉我们应该："必须使用 Spring Boot 3.x，禁止在 Service 层以外的地方使用`@Transactional`"。

`constraints.md`是 wiki-architect-deep 产物中最接近规约性的文档，可以相对直接地转化为`.mdp/rules/project/`。但即便是 constraints.md，也需要人工审核，明确区分"这是现有代码的客观约束"和"这是团队认可的规范约束"。

`architecture/decisions.md`需要更多人工加工——它记录了架构决策的历史和原因，但需要人工补充"因此，AI 在生成代码时应该……"的行动指导，才能成为真正对 AI 有用的规约性知识。

### 3.3 操作流程

```Mermaid
flowchart TD
    A["执行 wiki-architect-deep Skill"] --> B["产出 knowledge-docs/ 目录"]
    B --> C["人工审核产物"]
    
    C --> D{"知识类型判断"}
    D -->|"描述性知识<br/>（现状、背景、流程）"| E[".mdp/context/ 目录"]
    D -->|"约束性知识<br/>（限制、禁止、要求）"| F[".mdp/rules/project/ 目录"]
    D -->|"混合型<br/>（既有描述又有约束）"| G["人工拆分后分别写入"]
    
    E --> H["更新 AGENTS.md 的 context 索引"]
    F --> I["更新 AGENTS.md 的 rules 索引"]
    G --> H
    G --> I
    
    H --> J["验证 AGENTS.md 准确性"]
    I --> J
    J --> K["提交 PR，由团队 Review"]
    
    style A fill:#e3f2fd,stroke:#1976D2
    style B fill:#e8f5e9,stroke:#388E3C
    style D fill:#fff8e1,stroke:#F57C00
    style E fill:#e8f5e9,stroke:#388E3C
    style F fill:#fce4ec,stroke:#C62828
    style G fill:#f3e5f5,stroke:#7B1FA2
    style J fill:#e0f2f1,stroke:#00796B
    style K fill:#e8eaf6,stroke:#3949AB
```

### 3.4 注意事项与常见陷阱

**陷阱一：把"描述"直接当"规范"**

wiki-architect-deep 可能产出类似"项目中使用了 Guava Cache 做本地缓存"的描述。如果直接放入`rules/`，AI 代理会把"现在用了 Guava Cache"理解为"应该用 Guava Cache"，锁死了技术选型的灵活性。正确做法是把这类内容放入`context/`，而把"使用 Guava Cache 时，必须设置最大容量 upper bound"这类规约性内容放入`rules/`。

**陷阱二：对所有产物一视同仁**

`learning-path/`是给人类新人看的上手路径，`user-stories/`是业务流程描述，`project-health.md`是技术债务现状。这三类文档放入`context/`是合适的，但不应该放入`rules/`——它们描述现状，不是规范要求。

**陷阱三：忽视 knowledge.yml 的自动化潜力**

knowledge.yml 包含了项目的结构化元数据（技术栈、模块列表、关键依赖等），这是自动生成 AGENTS.md 的最佳数据源。很多团队手工维护 AGENTS.md，导致它很快过时。正确做法是把 knowledge.yml 作为数据源，设计脚本自动生成 AGENTS.md 的"项目快照"部分。

---

## 第四章：如何与 AGENTS.md 联动（自动生成索引）

### 4.1 AGENTS.md 的定位重新理解

在深入讨论自动生成之前，需要先明确 AGENTS.md 的本质定位。

AGENTS.md 不是"文档索引"，不是"README"，也不是"项目介绍"。它是**AI 代理的系统提示前置注入文件**——是在每一次 AI 会话开始时，被自动注入到 AI 上下文中的"认知框架"。

这个定位决定了 AGENTS.md 的写作原则：
- 优化目标是**AI 的理解效率**，而不是人类的阅读体验
- 每一行都应该对 AI 的代码生成行为产生可衡量的影响
- 精确的文件路径引用比模糊的描述更有价值
- "必须"和"禁止"比"建议"更有效

### 4.2 AGENTS.md 的最优结构

```Markdown
# AI Agent 知识索引

## 项目快照（自动从 knowledge.yml 生成）
- **项目名**：{project_name}
- **技术栈**：Java 17 + Spring Boot 3.2 + MyBatis + MySQL 8.0
- **MDP 版本**：5.x.x
- **模块结构**：{module_list}

## 编码规范（必读 - 每次代码生成前加载）

### 公司级 Java 规范（.mdp/rules/company/java/）
- 代码风格与命名：`.mdp/rules/company/java/code-style.md`
- 异常处理规范：`.mdp/rules/company/java/exception-handling.md`
- 日志记录规范：`.mdp/rules/company/java/logging.md`
- OOP 最佳实践：`.mdp/rules/company/java/oop-principles.md`
- 集合使用规范：`.mdp/rules/company/java/collections.md`
- 并发编程规范：`.mdp/rules/company/java/concurrency.md`
- 常量定义规范：`.mdp/rules/company/java/constants.md`
- 注释规范：`.mdp/rules/company/java/comments.md`
- 控制流规范：`.mdp/rules/company/java/control-flow.md`
- SQL 基础规范：`.mdp/rules/company/java/sql-basics.md`
- SQL 注入防护：`.mdp/rules/company/java/sql-injection.md`

### MDP 框架规范（.mdp/rules/mdp/）
- 框架识别规范：`.mdp/rules/mdp/architecture-identification.md`
- 组件使用规范：`.mdp/rules/mdp/component-usage.md`

### 团队规范（.mdp/team/）
- {team_rules_index}

### 项目级规范（.mdp/project/ - 优先级最高）
- 开发约束：`.mdp/rules/project/dev-constraints.md`
- {project_specific_rules}

## 业务上下文（按需参考 - 特定场景加载）
- 架构决策记录：`.mdp/context/architecture-decisions.md`
- 模块索引：`.mdp/context/modules.md`
- 核心业务流程：`.mdp/context/key-flows.md`
- 技术债务现状：`.mdp/context/tech-debt.md`
- 新人上手路径：`.mdp/context/onboarding-path.md`

## 开发约束（强制 - 违反需充分理由）
- 禁止手动修改的文件：{auto_generated_files}
- 事务规则：所有数据库操作必须在 Service 层统一管理事务
- 配置中心规则：所有配置必须通过 Lion 配置中心管理，禁止硬编码
- 数据库访问：必须通过 Zebra 数据源，禁止直接配置 JDBC
```

### 4.3 自动生成方案设计

AGENTS.md 应该是**自动生成的**，而不是手工维护的。手工维护的 AGENTS.md 在规则更新后很容易因为遗忘而不同步，导致 AI 代理使用过时的索引。

自动生成的核心流程是：
1. **读取 knowledge.yml**：提取项目名、技术栈、版本、模块列表等结构化信息，填充"项目快照"部分
2. **扫描 .mdp/rules/ 目录树**：自动发现所有规范文件，生成分类的"编码规范"索引
3. **扫描 .mdp/context/ 目录树**：自动发现所有上下文文件，生成"业务上下文"索引
4. **读取约束配置**：从`.mdp/project/dev-constraints.md`或专用配置文件提取强制约束
5. **输出 AGENTS.md**：按上述结构生成完整文件

这个流程可以实现为：
- 一个`generate-agents-md`Shell 脚本，通过`find`和`yq`工具生成
- 一个 Claude Code Skill（`/generate-agents-md`），在每次规则更新后手动触发
- 一个 Git Hook（pre-commit），在每次提交`.mdp/`变更时自动重新生成

### 4.4 关键设计原则：必读 vs 按需参考

AGENTS.md 中最重要的设计决策是区分"必读"和"按需参考"：

**必读（每次代码生成都要加载）**：
- 公司级 Java 规范（所有 Java 代码必须遵守）
- MDP 框架规范（所有组件使用必须合规）
- 强制开发约束（配置中心、事务规则等）

**按需参考（特定场景才加载）**：
- 业务上下文文档（只在处理特定业务逻辑时需要）
- 架构决策记录（只在做架构级修改时需要）
- 技术债务现状（只在重构时需要）

这种区分的意义在于：AI 代理每次代码生成时，如果把全量知识库都加载进上下文，会造成严重的 token 浪费，甚至因为上下文过长而降低核心规范的"注意权重"。把知识按优先级分类，让 AI 代理可以精确控制加载策略，是提升代码生成准确性的关键工程手段。

---

## 第五章：全栈/前端/后端工程知识库的差异化设计

### 5.1 不同工程类型的知识库差异

MDP 规则仓库的三级结构（company → team → project）在不同工程类型下，填充的内容有显著差异：

| 维度 | 全栈工程 | 前端工程 | 后端工程 |
|---|---|---|---|
| `context/`核心内容 | 前后端联动流程 + DB 数据模型 + API 契约 | 组件库使用规范 + 状态管理流程 + 页面路由结构 | API 设计规范 + DB 事务规则 + 服务依赖图 |
| `rules/`核心内容 | 全栈约束（接口版本控制、类型共享）+ 前后端接口契约 | CSS 规范（BEM/Tailwind）+ 组件结构规范 + 状态管理规范 | Java/Go 编码规范 + SQL 规范 + 日志规范 |
| 用户故事侧重 | 端到端链路（从用户操作到数据库变更） | 组件交互 + 状态流（用户行为如何触发状态变更） | 系统流程 + 数据操作（API 调用链 + 数据库事务） |
| `AGENTS.md`重点 | API 映射表（前端调哪个接口）+ 类型定义路径 | 组件目录索引（特定场景用哪个组件） | 模块接口索引（服务间调用关系） |
| 最常见的规范冲突 | 前端组件命名 vs 后端实体命名的一致性 | 设计稿规范 vs 代码实现的映射关系 | ORM 规范 vs SQL 性能优化的权衡 |
| 知识更新频率 | 高（前后端协作，变更频繁） | 中（组件库版本升级时批量更新） | 低（核心规范相对稳定） |

### 5.2 分层模型的优先级规则

三层结构中存在一个容易被忽视的设计问题：当三层规则存在冲突时，应该怎么处理？

规则是：**项目级规则 > 团队级规则 > 公司级规则**。

但这里有一个重要的工程约束：**覆盖需要有理由记录**。当项目级规则覆盖了公司级规则时，必须在文档头部注明：

```Markdown
---
overrides: .mdp/rules/company/java/exception-handling.md
override-reason: 本项目为实时计算系统，部分异常需要静默处理以保证吞吐量，详见架构决策记录 ADR-007
---
```

这个设计防止了两个问题：
- 防止团队在不知情的情况下违反公司规范（所有覆盖都是显式的、有记录的）
- 防止技术债务通过规则覆盖悄悄积累（每个覆盖都有 rationale，可以在架构评审中回顾）

### 5.3 三层结构与 AI 消费路径

```Mermaid
flowchart LR
    subgraph 规则层次
        A["company/<br/>公司级规则<br/>（最低优先级）"]
        B["team/<br/>团队级规则<br/>（中等优先级）"]
        C["project/<br/>项目级规则<br/>（最高优先级）"]
    end
    
    subgraph 上下文层次
        D["context/<br/>业务上下文"]
        E["wiki-architect-deep<br/>产物（自动）"]
    end
    
    subgraph AI 代理
        F["AGENTS.md<br/>统一入口"]
        G["Claude Code<br/>代码生成"]
    end
    
    A -->|"继承"| B
    B -->|"继承 + 覆盖"| C
    C -->|"合并后"| F
    D --> F
    E -.->|"转化"| D
    F --> G
    
    G --> H["符合规范的代码"]
    
    style A fill:#bbdefb,stroke:#1565C0
    style B fill:#c8e6c9,stroke:#2E7D32
    style C fill:#ffe0b2,stroke:#E65100
    style F fill:#f8bbd0,stroke:#880E4F
    style G fill:#e1bee7,stroke:#4A148C
    style H fill:#b2dfdb,stroke:#004D40
```

### 5.4 前端工程的特殊考量

前端工程有一个后端不存在的特殊问题：**设计稿到代码的映射规范**。

在 MDP 规则仓库的前端版本中，`rules/company/frontend/`应该包含：
- `component-naming.md`：组件命名与 Figma 图层命名的对应规则
- `design-token.md`：设计 token 与 CSS 变量的映射规范
- `responsive-breakpoints.md`：响应式断点的标准值（禁止自定义魔法数字）

`context/`应该包含：
- `component-catalog.md`：组件库目录（这个场景用哪个组件）
- `design-system-tokens.md`：设计系统的 token 列表（AI 生成 CSS 时参考）

这类前端特有的知识，在通用的公司级规则中无法覆盖，必须在`team/`或`project/`层次定制。

---

## 第六章：知识库与 AI Coding 工作流的集成

### 6.1 集成点一：Claude Code / Cursor 的上下文注入

最直接的集成方式是配置 AI 工具自动加载 AGENTS.md：

**Claude Code 配置**（`.claude/settings.json`）：

```JSON
{
  "contextFiles": [
    "AGENTS.md",
    ".mdp/rules/company/java/exception-handling.md",
    ".mdp/rules/company/java/logging.md"
  ]
}
```

**Cursor 配置**（`.cursor/rules/`）：
将 AGENTS.md 软链接到`.cursor/rules/agents.mdc`，Cursor 会自动在每个会话中注入。

这种配置让 AI 在每个会话开始时自动加载规范，无需用户手动提示"遵循我们的编码规范"。规范注入变成了透明的后台行为，而不是用户需要记住的操作步骤。

### 6.2 集成点二：Skill 触发器与知识库联动

在 AI Coding 工作流中，wiki-architect-deep 可以作为知识库更新的触发器：
- 当用户说"帮我实现 XX 功能"时，作为前置检查，验证`.mdp/context/`中是否有相关模块的上下文文档
- 如果上下文文档不存在或超过 30 天未更新，自动触发 wiki-architect-deep 对相关模块进行扫描
- 扫描完成后，将产出的描述性知识存入`context/`，然后继续执行代码生成任务

这种联动确保了"知识库驱动的代码生成"——AI 生成代码之前，先确认知识库是最新的。

### 6.3 集成点三：PR 代码审查中的规范检查

将`.mdp/rules/`中的规范配置到 AI 代码审查中：

```Markdown
## AI 代码审查配置（.mdp/review-config.md）

### 必须检查的规范
- .mdp/rules/company/java/exception-handling.md（所有 try-catch 块）
- .mdp/rules/company/java/sql-injection.md（所有数据库操作）
- .mdp/rules/company/java/logging.md（所有 Logger 使用）

### 发现违规时
- CRITICAL：SQL 注入风险，阻止合并
- HIGH：未处理的异常，要求修改
- MEDIUM：日志格式不规范，建议修改
```

这样，每次 PR 提交时，AI 代码审查器会自动加载对应的规范文件，针对变更代码检查是否违反了团队约定的规范。

### 6.4 集成点四：MCP 知识查询（未来方向）

当知识库规模足够大时，把全量`.mdp/context/`都注入 AI 上下文会造成严重的 token 浪费。更好的方案是将知识库暴露为 MCP（Model Context Protocol）资源服务：
- `knowledge.yml`作为元数据索引，AI 代理可以查询"项目中有哪些模块"
- `.mdp/context/<module>.md`作为可按需查询的知识资源，AI 代理在处理特定模块时精确加载
- `.mdp/rules/`的规范文件作为工具调用的参数，AI 代理在生成代码时按类型查询

这种方案将知识库从"静态注入的上下文"升级为"动态查询的知识服务"，是知识库与 AI Coding 工具集成的未来方向。

### 6.5 完整的 AI Coding 工作流集成图

```Mermaid
flowchart TD
    A["开发者输入<br/>（'帮我实现 XX 功能'）"] --> B["AI 代理启动"]
    B --> C["自动加载 AGENTS.md<br/>（系统提示注入）"]
    C --> D{"知识库是否最新？"}
    
    D -->|"上下文文档 > 30 天"| E["触发 wiki-architect-deep<br/>更新 context/"]
    D -->|"知识库是最新的"| F["加载相关规范文件"]
    E --> F
    
    F --> G["AI 代码生成<br/>（遵循规范约束）"]
    G --> H["开发者 Review"]
    H --> I["提交 PR"]
    I --> J["AI 代码审查<br/>（加载 .mdp/rules/ 检查规范）"]
    J --> K{"发现规范违规？"}
    
    K -->|"CRITICAL"| L["阻止合并<br/>（必须修改）"]
    K -->|"MEDIUM/LOW"| M["添加审查意见<br/>（建议修改）"]
    K -->|"无违规"| N["通过审查<br/>合并代码"]
    
    L --> G
    M --> H
    
    style A fill:#e3f2fd,stroke:#1976D2
    style C fill:#fce4ec,stroke:#C62828
    style E fill:#e8f5e9,stroke:#2E7D32
    style G fill:#f3e5f5,stroke:#7B1FA2
    style J fill:#fff3e0,stroke:#E65100
    style N fill:#e0f2f1,stroke:#00796B
```

---

## 第七章：优化建议与深度思考

### 7.1 现有结构的三个优化建议

**优化建议一：增加 context/knowledge-index.md**

当前结构中，`context/`存放人工编写或转化的业务上下文，但 wiki-architect-deep 产出的完整知识库（`knowledge-docs/`）是独立存在的。建议在`context/`中增加一个`knowledge-index.md`，作为从 MDP 规则仓库到 wiki-architect-deep 知识库的桥接文件：

```Markdown
# 知识库索引

## 自动生成知识库位置
- 完整知识库：./knowledge-docs/
- 核心索引：./knowledge-docs/knowledge.yml
- 最近生成时间：2026-04-10

## 已转化到本仓库的知识
- 架构决策：.mdp/context/architecture-decisions.md（来源：knowledge-docs/deep-dive/architecture/decisions.md）
- 模块概