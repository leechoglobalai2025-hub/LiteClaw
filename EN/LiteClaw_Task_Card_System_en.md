

> **Usage principle: Each card = one independent Claude Code session. Verify completion before starting the next.** **Never feed multiple cards at once. Agent single-shot is optimal — continuous accumulation guarantees entropy.**

---

## Usage

```bash
# Feed Claude Code one task card at a time
claude -p "$(cat tasks/TASK-01.md)"

# After completion, manual verification
# Verification passed → give next card
# Verification failed → new session, redo this card (do not patch in same session)
```

---

## Phase 0: Project Initialization

### TASK-00: Fork + Project Skeleton

```markdown
# TASK-00: 初始化 LiteClaw 项目

## Background
LiteClaw 是重构为架构正确的个人 AI Agent 网关。

## Task
1. 克隆 nanobot 仓库到 ~/projects/liteclaw
2. 在项目根目录创建以下新目录（空目录放 .gitkeep）：
   - liteclaw/security/
   - liteclaw/router/
   - liteclaw/context/
   - liteclaw/storage/
   - liteclaw/browser/
   - liteclaw/audit/
   - liteclaw/ui/
3. 创建 ARCHITECTURE.md，内容是以下分层说明：
   L0 Security → L1 Channel → L2 Router → L3 Context → 
   L4 LLM → L5 Execution → L6 Storage → L7 Audit
4. 更新 README.md，项目名改为 LiteClaw，说明这是 nanobot 的架构重构版
5. git commit -m "init: fork nanobot as LiteClaw, add new layer directories"

## Acceptance Criteria
- [ ] 项目可以正常 `pip install -e .` 安装
- [ ] 原 nanobot 功能可用：`liteclaw agent -m "hello"` 能得到回复
- [ ] 新目录存在
- [ ] ARCHITECTURE.md 存在且内容正确

## Constraints
- 不要修改 nanobot 现有代码的功能
- 只做结构性改动
```

---

## Phase 1: Security Layer

### TASK-01: Credential Store

```markdown
# TASK-01: 实现 Credential Store

## Background
LiteClaw 的安全核心原则：Agent 永远看不到任何 API Key。
密钥通过环境变量或 OS keychain 存储，在 HTTP 请求层注入。

## Task
在 liteclaw/security/credential_store.py 中实现：

class CredentialStore:
    """
    密钥存储。支持两种后端：
    1. 环境变量（默认）：读取 LITECLAW_GEMINI_KEY 等
    2. keyring 库（可选）：读取 OS Keychain
    
    关键约束：
    - get_key() 只在 LLM 调用层使用，永不进入 Agent 上下文
    - __repr__ 和 __str__ 永远返回 "***REDACTED***"
    - 序列化（json, pickle）时永远返回空值
    """
    
    def __init__(self, backend: str = "env"):
        ...
    
    def get_key(self, service: str) -> str:
        """获取密钥。service 例如 'gemini', 'anthropic', 'telegram'"""
        ...
    
    def set_key(self, service: str, key: str):
        """存储密钥到后端"""
        ...
    
    def list_services(self) -> list[str]:
        """列出已配置的服务名称（不返回 key 值）"""
        ...

## Test Requirements
创建 tests/test_credential_store.py：
- test_get_key_from_env: 设置环境变量，验证能读取
- test_repr_redacted: 验证 str(store) 不泄露任何 key
- test_serialization_safe: 验证 json.dumps 不泄露 key

## Acceptance Criteria
- [ ] 所有 3 个测试通过
- [ ] `str(store)` 和 `repr(store)` 不含任何密钥字符
- [ ] 代码不超过 80 行

## Constraints
- 不要修改其他文件
- 只依赖标准库 + keyring（可选依赖）
```

### TASK-02: Sanitizer (Log Redactor)

```markdown
# TASK-02: 实现日志脱敏器

## Background
所有日志、审计记录、对话存档都必须脱敏，确保 API Key 不泄露。

## Task
在 liteclaw/security/sanitizer.py 中实现：

class Sanitizer:
    """
    自动检测并脱敏敏感信息。
    
    需要脱敏的模式：
    - API Key 格式：sk-xxx, AIza-xxx, ghp_xxx, BSA-xxx 等
    - Bearer Token：Bearer eyJxxx
    - 环境变量赋值：export XXX_KEY=xxx
    - config 中的 apiKey 字段值
    - 任何连续 20+ 字符的 base64-like 字符串
    """
    
    def sanitize(self, text: str) -> str:
        """脱敏文本，返回脱敏后的版本"""
        ...
    
    def sanitize_dict(self, data: dict) -> dict:
        """深度脱敏字典中的所有值"""
        ...

## Test Requirements
创建 tests/test_sanitizer.py，至少覆盖：
- test_api_key_patterns: "sk-or-v1-abc123..." → "sk-or-v1-***REDACTED***"
- test_bearer_token: "Bearer eyJhb..." → "Bearer ***REDACTED***"
- test_nested_dict: 深层嵌套 dict 中的 key 被脱敏
- test_normal_text_unchanged: 普通文本不被修改
- test_config_apikey: {"apiKey": "real-key"} → {"apiKey": "***REDACTED***"}

## Acceptance Criteria
- [ ] 所有 5+ 测试通过
- [ ] 正则不会误杀普通文本
- [ ] 代码不超过 100 行
```

### TASK-03: Agent Firewall

```markdown
# TASK-03: 实现 Agent 权限防火墙

## Background
Agent 生成的任何工具调用，如果尝试读取密钥、密码、token，必须被拦截。

## Task
在 liteclaw/security/agent_firewall.py 中实现：

class AgentFirewall:
    """
    拦截 Agent 的危险行为。
    
    拦截规则：
    1. 工具参数中包含 key/secret/token/password 的读取请求
    2. Shell 命令中包含 cat/echo/print 密钥文件的操作
    3. 文件读取请求指向 .env, config.json 中的敏感字段
    """
    
    def check_tool_call(self, tool_name: str, params: dict) -> FirewallDecision:
        """
        检查工具调用是否安全。
        返回 FirewallDecision(allowed=bool, reason=str)
        """
        ...
    
    def check_shell_command(self, command: str) -> FirewallDecision:
        """检查 shell 命令是否安全"""
        ...

## Test Requirements
- test_block_cat_env: `cat .env` → blocked
- test_block_echo_key: `echo $API_KEY` → blocked
- test_allow_normal_shell: `ls -la` → allowed
- test_block_config_read: read_file(".nanobot/config.json") 中 apiKey 字段 → blocked
- test_allow_normal_read: read_file("main.py") → allowed

## Acceptance Criteria
- [ ] 所有测试通过
- [ ] 误拦率为零（正常操作不被拦截）
- [ ] 代码不超过 120 行
```

### TASK-04: Integrate Security Layer into Agent Loop

```markdown
# TASK-04: 将安全层集成到现有 Agent Loop

## Background
前三个任务创建了安全组件，现在需要把它们接入 nanobot 的 agent loop。

## Task
1. 修改 liteclaw/providers/ 下的 LLM provider：
   - 从 CredentialStore 读取 API Key（而非 config.json 明文）
   - 确保 Key 只在 HTTP header 注入，不进入消息列表

2. 修改 liteclaw/agent/loop.py：
   - 每个工具调用前，通过 AgentFirewall.check_tool_call() 检查
   - 被拦截的调用，返回 "Operation blocked by security policy" 给 LLM

3. 修改日志输出：
   - 所有 logger 输出通过 Sanitizer.sanitize() 过滤

4. 修改 config/schema.py：
   - 移除 providers.*.apiKey 字段的默认存储
   - 改为 providers.*.keyRef 引用（如 "$LITECLAW_GEMINI_KEY"）

## Acceptance Criteria
- [ ] `liteclaw agent -m "hello"` 仍然可用（从环境变量读 Key）
- [ ] config.json 中不再需要明文 API Key
- [ ] 日志中不出现任何密钥
- [ ] Agent 请求 "读取 config.json 中的 apiKey" 会被拦截

## Constraints
- 最小改动原则：只改必须改的文件
- 保持 nanobot 原有功能不受影响
```

---

## Phase 2: Router Layer

### TASK-05: Task Classifier

```markdown
# TASK-05: 实现本地任务分级器

## Task
在 liteclaw/router/classifier.py 中实现：

class TaskClassifier:
    """
    本地任务分级器。零 API 成本，延迟 < 50ms。
    
    三个级别：
    - SIMPLE: 短句、问答、状态查询（< 120 字符，无代码/多步/分析关键词）
    - MODERATE: 中等任务、文档分析、简单代码
    - COMPLEX: 复杂推理、架构设计、多步骤任务
    
    分级基于：消息长度、关键词匹配、历史上下文（前一条消息）
    """

## Test Cases (all must be covered)
- "你好" → SIMPLE
- "现在几点了" → SIMPLE
- "帮我分析这段代码的架构问题并提出改进方案" → COMPLEX
- "翻译这段话" → MODERATE
- "设计一个分布式系统，需要考虑..." → COMPLEX
- "git status" → SIMPLE
- "帮我写一个 Python 爬虫" → MODERATE

## Acceptance Criteria
- [ ] 7 个测试用例全部通过
- [ ] 分级延迟 < 10ms（纯本地计算）
- [ ] 代码不超过 80 行
```

### TASK-06: Model Selector + Failover

```markdown
# TASK-06: 实现模型选择器和确定性 Failover

## Task
在 liteclaw/router/model_selector.py 中实现：

class ModelSelector:
    """
    根据任务级别选择模型，并提供确定性 failover。
    
    路由表（可通过 config 自定义）：
    SIMPLE   → gemini-flash-lite → (无降级)
    MODERATE → gemini-3-flash   → gemini-flash-lite
    COMPLEX  → gemini-3-pro     → gemini-3-flash (thinking)
    
    Failover 规则：
    - 只在明确失败时触发（429/500/502/503/timeout）
    - 404 = 配置错误，报警不 failover
    - 成功响应后绝不 failover
    - Per-model 冷却，不是 per-provider
    """

## Dependencies
- liteclaw/router/classifier.py (TASK-05)
- liteclaw/router/rate_limiter.py (TASK-07，可先 mock)

## Acceptance Criteria
- [ ] SIMPLE 消息走最便宜模型
- [ ] 429 触发降级到下一级
- [ ] 404 不触发降级，而是报错
- [ ] 成功响应不触发降级
```

### TASK-07: Rate Limiter (Local)

```markdown
# TASK-07: 实现本地 Rate Limit 防护

## Task
在 liteclaw/router/rate_limiter.py 中实现：

class RateLimiter:
    """
    本地限流计数器。在 API 调用前拦截，不等 429。
    
    关键特性：
    - Per-model 独立计数（不是 per-provider）
    - 安全线 = 官方限制的 80%
    - 各模型的重置时间不同：
      * Gemini: RPD 在太平洋时间午夜重置 = 首尔时间 16:00/17:00
      * Anthropic: 滚动窗口 
      * OpenAI: 滚动窗口
    - 接近安全线时返回 THROTTLE（建议降级）
    - 超过限制时返回 BLOCKED（必须等待）
    """

    def check(self, model_id: str, estimated_tokens: int) -> RateLimitStatus:
        """
        返回: CLEAR / THROTTLE / BLOCKED
        THROTTLE = 建议降级到更便宜模型
        BLOCKED = 必须等待，返回预计恢复时间
        """
        ...
    
    def record_usage(self, model_id: str, input_tokens: int, output_tokens: int):
        """记录一次实际使用"""
        ...
    
    def get_status(self, model_id: str) -> ModelQuotaStatus:
        """获取模型当前配额状态（用于 UI 显示）"""
        ...

## Test Requirements
- test_clear_when_low_usage: 使用量低时返回 CLEAR
- test_throttle_near_limit: 接近 80% 时返回 THROTTLE
- test_blocked_at_limit: 达到限制时返回 BLOCKED
- test_reset_time_gemini: Gemini RPD 在正确时间重置
- test_per_model_isolation: 模型 A 超限不影响模型 B

## Acceptance Criteria
- [ ] 5 个测试全部通过
- [ ] Gemini RPD 重置时间正确（太平洋午夜）
- [ ] 代码不超过 150 行
```

### TASK-08: Token Budget Calculator

```markdown
# TASK-08: 实现 Token 预算计算器

## Task
在 liteclaw/router/budget.py 中实现：

class TokenBudget:
    """
    每次 LLM 调用前计算 Token 预算。
    
    公式：
    available = contextWindow - reserveOutput - systemPrompt - currentMessage - toolSchemas
    historyBudget = available * 0.5  (给对话历史)
    ragBudget = available * 0.3      (给 RAG 检索结果)
    toolResultBudget = available * 0.2 (给工具结果)
    """
    
    def calculate(self, model_id: str, system_prompt: str, 
                  current_message: str, tool_schemas: list) -> Budget:
        """返回 Budget 对象，含各部分的 token 上限"""
        ...

## Dependencies
- tiktoken 或 gpt-tokenizer（本地 token 计数）

## Acceptance Criteria
- [ ] 预算总和不超过模型上下文窗口
- [ ] 每个部分的预算 > 0
- [ ] 计算耗时 < 50ms
```

### TASK-09: Integrate Router Layer into Agent Loop

```markdown
# TASK-09: 将路由层集成到 Agent Loop

## Task
修改 liteclaw/agent/loop.py：

1. 每条消息进入时，先通过 TaskClassifier 分级
2. 通过 ModelSelector 选择目标模型
3. 通过 RateLimiter 检查配额（THROTTLE → 降级, BLOCKED → 等待）
4. 通过 TokenBudget 计算预算（传给 Context Layer 使用）
5. 用选定的模型调用 LLM（而非固定模型）
6. 调用后通过 RateLimiter.record_usage() 记录使用量

## Acceptance Criteria
- [ ] 简单消息走便宜模型
- [ ] 复杂消息走强模型
- [ ] 接近限制时自动降级
- [ ] 日志清晰显示路由决策
```

---

## Phase 3: Context Layer (Core Refactoring)

### TASK-10: Dynamic Window Manager

```markdown
# TASK-10: 实现动态窗口管理器

## Background
OpenClaw 的核心病根是全量推送上下文。LiteClaw 用 3-15 轮动态窗口替代。
窗口大小根据任务复杂度自动调整，用户也可手动选择发送哪些轮次。

## Task
在 liteclaw/context/window.py 中实现：

class DynamicWindow:
    """
    动态对话窗口管理器。
    
    窗口规则：
    - SIMPLE 任务: 最近 3 轮
    - MODERATE 任务: 最近 5 轮
    - COMPLEX 任务: 最近 7-10 轮
    - 用户说"继续/接着": 最近 5 轮
    - 用户引用历史（"上次我们讨论的..."）: 3 轮 + 触发 RAG
    - 新话题/新任务: 最近 3 轮（干净启动，避免熵增）
    - 用户手动选择模式: 使用用户指定的轮次（最大 15 轮）
    
    环境元数据常驻区（永不裁剪）：
    - 操作系统信息
    - 当前工作目录
    - 项目依赖版本
    - 用户偏好设置
    这些信息 token 成本很低（~200-500 token）但缺失会导致回答完全错误。
    """
    
    def __init__(self, config: WindowConfig):
        ...
    
    def select_window(self, task_level: str, messages: list[Message], 
                      current_message: str) -> list[Message]:
        """
        根据任务级别和当前消息，选择要发送的历史轮次。
        返回筛选后的消息列表。
        """
        ...
    
    def detect_topic_change(self, current: str, recent: list[Message]) -> bool:
        """检测是否切换了新话题（如果是，缩小窗口避免熵增）"""
        ...
    
    def detect_history_reference(self, message: str) -> bool:
        """检测用户是否在引用更早的历史（如果是，触发 RAG 补充）"""
        ...
    
    def get_env_metadata(self) -> str:
        """
        获取环境元数据（常驻区，永不裁剪）。
        包含：OS、Python版本、当前目录、项目依赖等。
        """
        ...

## Test Requirements
创建 tests/test_window.py：
- test_simple_3_turns: SIMPLE 任务只返回最近 3 轮
- test_complex_10_turns: COMPLEX 任务返回最近 7-10 轮
- test_topic_change_reset: 检测到新话题时窗口缩小到 3 轮
- test_history_ref_triggers_rag: "上次我们讨论的" 触发 RAG 标记
- test_env_metadata_always_present: 环境元数据始终包含在返回中
- test_max_15_turns: 手动模式不超过 15 轮

## Acceptance Criteria
- [ ] 6 个测试全部通过
- [ ] 环境元数据不超过 500 token
- [ ] 话题变更检测准确率 > 80%
- [ ] 代码不超过 150 行

## Constraints
- 话题检测用关键词重叠率，不调 API
- 环境元数据通过 platform、sys、subprocess 获取
```

### TASK-11: RAG History Retrieval

````markdown
# TASK-11: 实现 RAG 历史对话检索系统

## Background
超出动态窗口的历史对话，通过 RAG 按需召回。
双通道检索：FTS5 全文搜索（精确匹配）+ 向量语义搜索（模糊匹配）。

## Task
在 liteclaw/context/rag.py 中实现：

class HistoryRAG:
    """
    历史对话 RAG 检索系统。
    
    存储：SQLite + FTS5 全文索引 + sqlite-vec 向量索引
    
    检索流程：
    1. 从当前消息提取关键词
    2. FTS5 全文检索（精确匹配）→ 候选集 A
    3. 向量相似度检索（语义匹配）→ 候选集 B
    4. Reciprocal Rank Fusion 合并排序
    5. 按 Token 预算裁剪 top-N 结果
    6. 格式化为上下文片段返回
    """
    
    def __init__(self, db_path: str):
        """初始化数据库连接，创建 FTS5 和向量表（如不存在）"""
        ...
    
    def index_message(self, message: Message):
        """将新消息索引到 FTS5 和向量库"""
        ...
    
    def search(self, query: str, budget_tokens: int, 
               exclude_session: str = None) -> list[RetrievedContext]:
        """
        检索相关历史。
        query: 当前用户消息
        budget_tokens: 可用的 token 预算
        exclude_session: 排除当前会话（避免重复）
        返回: 按相关度排序的历史片段列表
        """
        ...
    
    def _fts_search(self, keywords: list[str], limit: int) -> list[SearchResult]:
        """FTS5 全文检索"""
        ...
    
    def _vector_search(self, query_embedding: list[float], limit: int) -> list[SearchResult]:
        """向量相似度检索"""
        ...
    
    def _rank_fusion(self, fts_results: list, vec_results: list) -> list[SearchResult]:
        """Reciprocal Rank Fusion 合并两路结果"""
        ...
    
    def _extract_keywords(self, text: str) -> list[str]:
        """从文本中提取搜索关键词（本地，不调 API）"""
        ...

## Database Schema
```sql
CREATE TABLE messages (
    id INTEGER PRIMARY KEY,
    session_id TEXT NOT NULL,
    role TEXT NOT NULL,
    content TEXT NOT NULL,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    model_used TEXT,
    token_count INTEGER
);

CREATE VIRTUAL TABLE messages_fts USING fts5(
    content, 
    content=messages, 
    content_rowid=id,
    tokenize='unicode61'
);

-- 向量索引（使用 sqlite-vec 扩展）
-- embedding 维度取决于选用的 embedding 模型
-- 推荐: all-MiniLM-L6-v2 (384维) 本地运行
CREATE VIRTUAL TABLE messages_vec USING vec0(
    embedding float[384]
);
````

## Test Requirements

创建 tests/test_rag.py：

- test_index_and_fts_search: 索引消息后 FTS 能搜到
- test_vector_search: 语义相近的消息能被检索到
- test_rank_fusion: 两路结果正确合并
- test_budget_limit: 返回结果不超过 token 预算
- test_exclude_session: 当前会话的消息被排除
- test_keyword_extraction: 关键词提取准确

## Acceptance Criteria

- [ ] 6 个测试全部通过
- [ ] FTS 检索延迟 < 50ms
- [ ] 向量检索延迟 < 200ms（本地 embedding）
- [ ] 代码不超过 250 行

## Dependencies

- better-sqlite3 或 sqlite3（标准库）
- sqlite-vec（向量扩展，pip install sqlite-vec）
- sentence-transformers（本地 embedding，可选用 all-MiniLM-L6-v2）

## Constraints

- Embedding 在本地生成，不调外部 API
- 如果 sqlite-vec 安装失败，降级为纯 FTS5 模式

````

### TASK-12: Prompt Assembler

```markdown
# TASK-12: 实现基于预算的 Prompt 组装器

## Background
Context Layer 的最终输出：将所有信息在 Token 预算内组装成完整 Prompt。
关键创新：不是"有什么塞什么"，而是"预算多少放什么"。

## Task
在 liteclaw/context/assembler.py 中实现：

class PromptAssembler:
    """
    基于 Token 预算的 Prompt 组装器。
    
    组装顺序（优先级从高到低）：
    1. System Prompt（固定，~1.5K token）
    2. 环境元数据（常驻，~200-500 token）——永不裁剪
    3. 当前用户消息（固定）
    4. 工具 Schema（如有，~1K token）
    5. 最近对话窗口（3-15 轮，由 DynamicWindow 选定）
    6. RAG 检索结果（用剩余预算填充）
    7. 活跃 Skill 元数据（如有，~200 token）
    
    每一步都计算已用 token，超出预算则停止填充。
    """
    
    def __init__(self, token_counter):
        """token_counter: 本地 token 计数器（tiktoken）"""
        ...
    
    def assemble(self, params: AssembleParams) -> AssembledPrompt:
        """
        组装最终 Prompt。
        
        params 包含：
        - budget: TokenBudget（来自 TASK-08）
        - system_prompt: str
        - env_metadata: str（来自 DynamicWindow.get_env_metadata）
        - current_message: str
        - tool_schemas: list[dict]（可选）
        - window_messages: list[Message]（来自 DynamicWindow）
        - rag_results: list[RetrievedContext]（来自 HistoryRAG）
        - active_skill: SkillMetadata（可选）
        
        返回：
        - AssembledPrompt: 包含 system, messages, tools, estimated_tokens
        """
        ...
    
    def _fit_within_budget(self, items: list, budget: int) -> list:
        """在 token 预算内尽量多放内容，超出则截断"""
        ...
    
    def _truncate_tool_result(self, result: str, max_chars: int = 4000) -> str:
        """
        截断工具结果。超过 max_chars 时保留 head 1500 + tail 1500。
        工具结果不进入持久化历史，只保留单行摘要。
        """
        ...

## Test Requirements
创建 tests/test_assembler.py：
- test_budget_not_exceeded: 组装结果不超过总预算
- test_system_prompt_always_present: 系统提示词始终在
- test_env_metadata_always_present: 环境元数据始终在
- test_current_message_always_present: 当前消息始终在
- test_rag_fills_remaining: RAG 结果用剩余预算填充
- test_tool_result_truncation: 超长工具结果被正确截断（head+tail）
- test_priority_order: 预算不足时，低优先级内容被丢弃而非高优先级

## Acceptance Criteria
- [ ] 7 个测试全部通过
- [ ] 环境元数据和系统提示词在任何预算条件下都不被裁剪
- [ ] 工具结果截断保留 head 1500 + tail 1500 字符
- [ ] 代码不超过 180 行

## Dependencies
- tiktoken（本地 token 计数）
````

### TASK-13: Integrate Context Layer, Replace Original context.py

```markdown
# TASK-13: 集成上下文层到 Agent Loop

## Background
将 TASK-10/11/12 的组件接入 Agent Loop，替换 nanobot 原有的
全量上下文推送逻辑（agent/context.py）。

## Task
1. 创建 liteclaw/context/__init__.py 作为统一入口：

class ContextManager:
    """
    上下文管理器。统一调用 DynamicWindow + HistoryRAG + PromptAssembler。
    
    流程：
    1. 接收当前消息和任务级别
    2. DynamicWindow 选择窗口轮次
    3. 如果检测到历史引用 → HistoryRAG 检索补充
    4. PromptAssembler 在预算内组装最终 Prompt
    5. 返回组装好的 Prompt 给 LLM Layer
    """
    
    def build_context(self, current_message: str, task_level: str,
                      budget: TokenBudget, session_id: str) -> AssembledPrompt:
        ...

2. 修改 liteclaw/agent/loop.py：
   - 删除对原 context.py 的 ContextBuilder 调用
   - 替换为 ContextManager.build_context()
   - 每轮对话结束后，调用 HistoryRAG.index_message() 索引新消息

3. 保留原 context.py 的 bootstrap 文件加载逻辑（AGENTS.md, SOUL.md），
   但将其整合到 ContextManager 的 system_prompt 构建中。

4. 对话结束后，工具结果只存摘要到历史，不存原始输出。

## Acceptance Criteria
- [ ] `liteclaw agent -m "hello"` 正常工作
- [ ] 对话第 5 轮时，input token 数量不会比第 1 轮多出 10 倍（证明没有全量推送）
- [ ] 引用历史内容时能通过 RAG 召回
- [ ] 环境元数据在每轮都存在
- [ ] 原 nanobot 的 AGENTS.md/SOUL.md 仍然生效

## Constraints
- 不要删除原 context.py，重命名为 context_legacy.py 备份
- 最小改动 agent/loop.py
```

---

## Phase 4: Audit Layer

### TASK-14: Operation Risk Classifier

```markdown
# TASK-14: 实现操作风险分类器

## Background
LiteClaw 的所有操作行为都需要先评估风险等级，决定是否需要用户审批。
这是"防止 Agent 犯错"的第一道防线。

## Task
在 liteclaw/audit/risk_classifier.py 中实现：

class RiskClassifier:
    """
    操作风险分类器。本地规则引擎，零 API 成本。
    
    风险等级：
    - READ_ONLY: 只读操作 → 自动放行
      搜索、浏览网页、读取文件、查看代码、git status/log/diff
    
    - WRITE_LOCAL: 本地写入 → 自动放行，但详细记录
      编辑本地文件、git commit、创建文件、安装本地包
    
    - WRITE_REMOTE: 远程写入 → 需要用户通过 Telegram 确认
      git push、发送消息、提交表单、发布内容、发送邮件
    
    - DESTRUCTIVE: 破坏性操作 → 永久拦截，不可覆盖
      rm -rf、drop database、force push main、删除仓库
    
    - CREDENTIAL: 涉及凭证 → 永久拦截，不可覆盖
      读取/打印/发送任何 API Key、密码、Token
    """
    
    def classify(self, operation_type: str, target: str, 
                 params: dict = None) -> RiskLevel:
        """
        分类操作风险。
        operation_type: 'shell' | 'browser' | 'file_write' | 'file_read' | 'network'
        target: 操作目标（命令、URL、文件路径等）
        params: 额外参数
        """
        ...
    
    def classify_shell(self, command: str) -> RiskLevel:
        """专门分类 shell 命令的风险"""
        ...
    
    def classify_browser(self, action: str, url: str) -> RiskLevel:
        """专门分类浏览器操作的风险"""
        ...

## Risk Pattern Matching Rules

DESTRUCTIVE_PATTERNS = [
    r'rm\s+(-rf|-r\s+-f|-fr)', r'rmdir\s+/', r'format\s+',
    r'drop\s+(database|table)', r'truncate\s+table',
    r'git\s+push.*--force.*main', r'git\s+push.*-f.*main',
    r'DELETE\s+FROM\s+\w+\s*(;|$)',  # 无 WHERE 的 DELETE
]

CREDENTIAL_PATTERNS = [
    r'(cat|less|more|head|tail|echo|print).*\.(env|pem|key)',
    r'(cat|less|more).*config\.json',
    r'echo\s+\$\w*(KEY|SECRET|TOKEN|PASSWORD|PASS)',
    r'curl.*(-H|--header).*[Aa]uthorization',
    r'export\s+\w*(KEY|SECRET|TOKEN)',
]

WRITE_REMOTE_PATTERNS = [
    r'git\s+push', r'git\s+remote\s+',
    r'curl\s+.*(-X\s+POST|-X\s+PUT|-d\s+)',
    r'ssh\s+', r'scp\s+', r'rsync\s+.*:',
    r'npm\s+publish', r'pip\s+upload',
    r'docker\s+push',
]

## Test Requirements
创建 tests/test_risk_classifier.py：
- test_read_only: `ls -la`, `cat main.py`, `git log` → READ_ONLY
- test_write_local: `echo "x" > file.txt`, `git commit` → WRITE_LOCAL
- test_write_remote: `git push`, `curl -X POST` → WRITE_REMOTE
- test_destructive: `rm -rf /`, `drop database` → DESTRUCTIVE
- test_credential: `cat .env`, `echo $API_KEY` → CREDENTIAL
- test_browser_read: navigate + extract → READ_ONLY
- test_browser_submit: click submit button → WRITE_REMOTE

## Acceptance Criteria
- [ ] 7 个测试全部通过
- [ ] DESTRUCTIVE 和 CREDENTIAL 零漏判（最重要！）
- [ ] 正常操作零误判
- [ ] 代码不超过 150 行
```

### TASK-15: Three-Phase Audit Engine

```markdown
# TASK-15: 实现三阶段审计引擎（Pre / Monitor / Post）

## Background
所有 Agent 操作都经过三阶段审计：执行前检查 → 执行中监控 → 执行后分析。
这是"防止犯错 + 后台分析"的核心实现。

## Task
在 liteclaw/audit/engine.py 中实现：

class AuditEngine:
    """
    三阶段审计引擎。
    """
    
    # ──── 阶段一：操作前 ────
    async def pre_check(self, operation: Operation) -> AuditDecision:
        """
        操作前风险评估和审批。
        
        流程：
        1. 调用 RiskClassifier 分类风险
        2. 记录操作意图到审计日志
        3. 根据风险等级决策：
           - READ_ONLY → ALLOW
           - WRITE_LOCAL → ALLOW（详细记录）
           - WRITE_REMOTE → 发送确认到 Telegram，等待用户回复
           - DESTRUCTIVE → BLOCK
           - CREDENTIAL → BLOCK
        4. 返回 AuditDecision(allowed, reason, risk_level)
        """
        ...
    
    async def request_approval(self, operation: Operation) -> bool:
        """
        通过 Telegram 向用户请求审批。
        发送操作描述，等待用户回复 ✅ 或 ❌。
        超时 5 分钟自动拒绝。
        """
        ...
    
    # ──── 阶段二：操作中 ────
    async def monitor_step(self, operation_id: str, step: ExecutionStep):
        """
        实时监控执行步骤。
        
        功能：
        5. 记录每一步到审计日志
        6. 浏览器操作在关键节点保存截图
        7. Shell 操作记录命令和输出
        8. 实时异常检测（操作偏离预期时紧急停止）
        """
        ...
    
    def detect_anomaly(self, step: ExecutionStep, 
                       expected_intent: str) -> bool:
        """
        异常检测：操作是否偏离了原始意图。
        例如：意图是"搜索信息"，但实际在提交表单 → 异常
        """
        ...
    
    async def emergency_stop(self, operation_id: str):
        """紧急停止操作，通知用户"""
        ...
    
    # ──── 阶段三：操作后 ────
    async def post_audit(self, operation_id: str, result: OperationResult):
        """
        操作后分析。
        
        功能：
        9. 对比操作意图 vs 实际执行
        10. 检测是否有偏差
        11. 生成审计摘要
        12. 记录 token/成本消耗
        13. 返回审计报告（交给 TASK-16 生成 Markdown）
        """
        ...

## Data Models

@dataclass
class Operation:
    id: str                    # 唯一操作 ID
    intent: str                # 用户原始意图
    engine: str                # 'shell' | 'browser' | 'vscode'
    target: str                # 操作目标
    params: dict               # 操作参数
    timestamp: datetime

@dataclass
class ExecutionStep:
    step_number: int
    action: str                # 'navigate' | 'click' | 'type' | 'shell_exec' | ...
    target: str
    result: str
    duration_ms: int
    screenshot_path: str = None  # 浏览器操作的截图路径

@dataclass
class AuditDecision:
    allowed: bool
    reason: str
    risk_level: RiskLevel
    requires_approval: bool = False

## Test Requirements
创建 tests/test_audit_engine.py：
- test_pre_check_read_only: READ_ONLY 操作自动通过
- test_pre_check_destructive: DESTRUCTIVE 操作被拦截
- test_pre_check_credential: CREDENTIAL 操作被拦截
- test_monitor_records_steps: 操作步骤被正确记录
- test_anomaly_detection: 偏离意图时检测到异常
- test_post_audit_generates_report: 操作后生成审计报告

## Acceptance Criteria
- [ ] 6 个测试全部通过
- [ ] DESTRUCTIVE 和 CREDENTIAL 100% 拦截
- [ ] 审批超时 5 分钟自动拒绝
- [ ] 代码不超过 300 行

## Constraints
- Telegram 审批部分可先用 mock（实际通道在 TASK-20 集成）
- 异常检测用规则匹配，不调 API
```

### TASK-16: Audit Log Generator (Obsidian Markdown)

```markdown
# TASK-16: 实现审计日志生成器

## Background
所有审计记录生成为 Obsidian 友好的 Markdown 文件，便于人工回溯查阅。

## Task
在 liteclaw/audit/logger.py 中实现：

class AuditLogger:
    """
    审计日志生成器。输出 Obsidian Markdown 格式。
    
    目录结构：
    {vault}/LiteClaw/audit/
    ├── 2026-02-12/
    │   ├── 001-twitter-search.md
    │   ├── 002-code-edit.md
    │   ├── 003-git-push-BLOCKED.md
    │   └── daily-summary.md
    └── weekly-report.md
    """
    
    def __init__(self, vault_path: str):
        """vault_path: Obsidian vault 根目录"""
        ...
    
    def log_operation(self, operation: Operation, decision: AuditDecision,
                      steps: list[ExecutionStep], result: OperationResult) -> str:
        """
        生成单个操作的审计日志文件。
        返回文件路径。
        
        文件格式：
        ---
        date: 2026-02-12T14:30:00+09:00
        operation_id: op_001
        risk_level: READ_ONLY
        status: completed | blocked | error
        engine: browser | shell | vscode
        token_cost: 1234
        tags: [browser, search, twitter]
        ---
        
        # Operation: Twitter Search
        
        ## Intent
        用户请求搜索 Twitter 上关于 AI Agent 的讨论
        
        ## Risk Assessment
        - 风险等级: READ_ONLY
        - 需要审批: 否
        - 决策: 自动通过
        
        ## Execution Steps
        | # | 动作 | 目标 | 耗时 | 状态 |
        |---|------|------|------|------|
        | 1 | navigate | twitter.com | 1.2s | ✅ |
        | 2 | type | "AI Agent" | 0.8s | ✅ |
        
        ## Result Summary
        提取了 10 条讨论...
        
        ## Anomaly Check
        无异常。
        """
        ...
    
    def generate_daily_summary(self, date: str) -> str:
        """
        生成每日审计汇总。
        统计：操作总数、各风险等级数量、拦截次数、总成本。
        """
        ...
    
    def generate_weekly_report(self, week_start: str) -> str:
        """生成周报"""
        ...

## Test Requirements
创建 tests/test_audit_logger.py：
- test_operation_log_format: 生成的 Markdown 格式正确（含 frontmatter）
- test_blocked_operation_log: 被拦截的操作正确标记
- test_daily_summary: 每日汇总统计准确
- test_file_naming: 文件名格式正确（序号-描述.md）
- test_obsidian_compatible: 生成的文件可被 Obsidian 正确解析

## Acceptance Criteria
- [ ] 5 个测试全部通过
- [ ] Markdown 文件含 YAML frontmatter（Obsidian 元数据）
- [ ] 被拦截的操作文件名含 -BLOCKED 后缀
- [ ] 代码不超过 200 行

## Constraints
- 日志写入用追加模式，不覆盖已有文件
- 文件名用序号递增（001, 002, ...）
- 截图文件引用用相对路径（Obsidian 兼容）
```

---

## Phase 5: Execution Layer

### TASK-17: Shell/Terminal Engine + Claude Code Controller

```markdown
# TASK-17: 实现 Shell 终端引擎和 Claude Code 控制器

## Background
LiteClaw 需要通过终端执行命令、操控 Claude Code 等 CLI 工具。
所有操作必须经过审计层，危险命令自动拦截。

## Task
在 liteclaw/execution/shell_engine.py 中实现：

class ShellEngine:
    """
    安全的 Shell 执行引擎。
    所有命令先过审计层，再执行，执行后记录。
    """
    
    def __init__(self, audit: AuditEngine, work_dir: str = None):
        ...
    
    async def execute(self, command: str, intent: str = "") -> ShellResult:
        """
        执行 Shell 命令。
        
        流程：
        1. AuditEngine.pre_check() 评估风险
        2. 如果 BLOCKED → 返回拒绝，不执行
        3. 如果 WRITE_REMOTE → 等待用户审批
        4. 执行命令（subprocess，设置超时）
        5. AuditEngine.monitor_step() 记录
        6. AuditEngine.post_audit() 分析
        7. 返回结果
        """
        ...
    
    async def execute_streaming(self, command: str, 
                                 on_output: Callable) -> ShellResult:
        """
        流式执行命令（用于长时间运行的任务）。
        实时回调输出，并实时监控异常。
        """
        ...

class ClaudeCodeController:
    """
    Claude Code CLI 控制器。
    通过子进程调用 claude 命令，实时监控输出。
    """
    
    def __init__(self, shell: ShellEngine, audit: AuditEngine):
        ...
    
    async def run(self, prompt: str, project_path: str,
                  allowed_tools: list[str] = None) -> ClaudeCodeResult:
        """
        运行 Claude Code 任务。
        
        allowed_tools: 限制可用工具，如 ['Edit', 'Read', 'Bash(git diff)']
        默认禁止: Write（新建文件需确认）、Bash（危险命令由审计层拦截）
        
        实时监控输出，检测到高风险操作时自动终止。
        """
        ...
    
    async def _monitor_output(self, process, operation_id: str):
        """实时读取 Claude Code 输出，检测危险操作"""
        ...

## Test Requirements
创建 tests/test_shell_engine.py：
- test_safe_command: `ls -la` 执行成功
- test_blocked_destructive: `rm -rf /` 被拦截不执行
- test_blocked_credential: `cat .env` 被拦截
- test_timeout: 超时命令被终止（设置 30 秒超时）
- test_output_recorded: 命令输出被记录到审计日志
- test_claude_code_launch: Claude Code 可以启动（mock subprocess）

## Acceptance Criteria
- [ ] 6 个测试全部通过
- [ ] 危险命令 100% 拦截，不执行
- [ ] 命令超时 30 秒自动终止
- [ ] 所有执行记录完整
- [ ] 代码不超过 200 行

## Constraints
- 命令执行用 asyncio.create_subprocess_exec，不用 os.system
- 设置 cwd 为项目工作目录
- stdout 和 stderr 都要捕获
```

### TASK-18: Playwright Browser Engine

```markdown
# TASK-18: 实现 Playwright 浏览器自动化引擎

## Background
LiteClaw 通过 Playwright 执行网页操作：社交媒体信息收集、
Web 应用操作等。使用 DOM 操作而非截图，精确且省 token。

## Task
在 liteclaw/execution/browser_engine.py 中实现：

class BrowserEngine:
    """
    基于 Playwright 的浏览器自动化引擎。
    
    特性：
    - DOM 操作（不用截图分析，省 token）
    - 人类行为模拟（随机延迟、模拟打字速度）
    - 关键节点自动截图（供审计用）
    - 所有操作经过审计层
    """
    
    def __init__(self, audit: AuditEngine, headless: bool = True):
        ...
    
    async def start(self):
        """启动浏览器实例"""
        ...
    
    async def stop(self):
        """关闭浏览器"""
        ...
    
    async def navigate(self, url: str, intent: str = "") -> BrowserResult:
        """导航到 URL"""
        ...
    
    async def click(self, selector: str, intent: str = "") -> BrowserResult:
        """点击元素"""
        ...
    
    async def type_text(self, selector: str, text: str, 
                        intent: str = "") -> BrowserResult:
        """在输入框输入文字（模拟人类打字速度）"""
        ...
    
    async def extract_text(self, selector: str = "body") -> str:
        """提取页面文本内容"""
        ...
    
    async def screenshot(self, name: str = None) -> str:
        """截图保存，返回文件路径"""
        ...
    
    async def execute_workflow(self, steps: list[BrowserStep], 
                               intent: str) -> WorkflowResult:
        """
        执行多步浏览器工作流。
        每步都经过审计，关键节点自动截图。
        
        steps 示例：
        [
            BrowserStep(action='navigate', target='https://twitter.com/search'),
            BrowserStep(action='type', target='input[name=q]', value='AI Agent'),
            BrowserStep(action='click', target='button[type=submit]'),
            BrowserStep(action='extract', target='.tweet-text'),
        ]
        """
        ...
    
    async def _human_delay(self, min_ms: int = 200, max_ms: int = 800):
        """模拟人类操作延迟"""
        ...

@dataclass
class BrowserStep:
    action: str        # 'navigate' | 'click' | 'type' | 'extract' | 'screenshot'
    target: str        # URL 或 CSS selector
    value: str = None  # type 操作的输入值
    wait_for: str = None  # 等待条件

## Test Requirements
创建 tests/test_browser_engine.py：
- test_navigate: 导航到页面成功
- test_extract_text: 提取页面文本
- test_screenshot_saved: 截图文件保存到正确位置
- test_human_delay: 延迟在指定范围内
- test_workflow_execution: 多步工作流按顺序执行
- test_audit_integration: 每步操作都被审计记录

## Acceptance Criteria
- [ ] 6 个测试全部通过
- [ ] 浏览器默认 headless 模式
- [ ] 人类延迟在 200-800ms 范围
- [ ] 截图保存到 {vault}/LiteClaw/screenshots/ 目录
- [ ] 代码不超过 250 行

## Dependencies
- playwright（pip install playwright && playwright install chromium）

## Constraints
- 只安装 chromium，不装 firefox/webkit（省空间）
- 提交表单类操作（WRITE_REMOTE）必须经审计层审批
- 截图文件名含操作 ID 和步骤号
```

### TASK-19: VSCode CLI Controller

```markdown
# TASK-19: 实现 VSCode CLI 控制器

## Background
通过 VSCode CLI 和文件直接操作来控制 IDE，不需要 GUI 截图。
确定性高、完全可审计。

## Task
在 liteclaw/execution/vscode_engine.py 中实现：

class VSCodeEngine:
    """
    VSCode 控制器。通过 CLI 命令和文件系统操作控制 IDE。
    不使用 GUI 操作（不用 PyAutoGUI/截图）。
    """
    
    def __init__(self, shell: ShellEngine, audit: AuditEngine):
        ...
    
    async def open_file(self, path: str) -> VSCodeResult:
        """用 VSCode 打开文件: code {path}"""
        ...
    
    async def open_project(self, path: str) -> VSCodeResult:
        """打开项目文件夹: code {path}"""
        ...
    
    async def edit_file(self, path: str, 
                        changes: list[FileChange]) -> VSCodeResult:
        """
        编辑文件。不通过 GUI，直接修改文件内容。
        VSCode 会自动检测到文件变更并更新显示。
        
        每个 change 是精确的文本替换：
        FileChange(old="原文本", new="新文本")
        
        所有改动记录到审计日志，并生成 diff。
        """
        ...
    
    async def diff_check(self, path: str) -> str:
        """通过 git diff 检查文件改动"""
        ...
    
    async def run_task(self, task_label: str) -> VSCodeResult:
        """运行 VSCode Task（定义在 .vscode/tasks.json 中）"""
        ...
    
    async def install_extension(self, ext_id: str) -> VSCodeResult:
        """安装 VSCode 扩展"""
        ...
    
    async def get_open_files(self) -> list[str]:
        """获取当前打开的文件列表（通过 workspace 状态）"""
        ...

@dataclass
class FileChange:
    old: str    # 要替换的原文本（必须精确匹配）
    new: str    # 替换后的新文本

## Test Requirements
创建 tests/test_vscode_engine.py：
- test_open_file: code 命令正确执行（mock subprocess）
- test_edit_file_precise: 精确文本替换生效
- test_edit_file_diff: 改动生成正确的 diff
- test_edit_not_found: 原文本不存在时报错而非静默失败
- test_audit_recorded: 所有操作记录到审计日志

## Acceptance Criteria
- [ ] 5 个测试全部通过
- [ ] 文件编辑使用精确文本替换（不是行号替换）
- [ ] 替换失败时抛出明确错误
- [ ] 每次编辑自动生成 git diff
- [ ] 代码不超过 150 行

## Constraints
- 确认 `code` 命令可用（不可用时给出友好提示）
- 文件编辑前自动备份（.bak 文件）
- 所有文件操作限制在工作目录内（防止路径穿越）
```

### TASK-20: Integrate Execution + Audit Layer Linkage

```markdown
# TASK-20: 集成执行层和审计层到 Agent Loop

## Background
将三个执行引擎（Shell/Browser/VSCode）和审计层统一接入 Agent Loop。
Agent 的工具调用自动路由到对应引擎，全程审计。

## Task
1. 创建 liteclaw/execution/__init__.py 作为统一入口：

class ExecutionManager:
    """
    统一执行管理器。根据工具类型路由到对应引擎。
    """
    
    def __init__(self, shell: ShellEngine, browser: BrowserEngine,
                 vscode: VSCodeEngine, audit: AuditEngine):
        ...
    
    async def execute_tool(self, tool_name: str, params: dict,
                           intent: str) -> ToolResult:
        """
        统一执行入口。
        
        路由规则：
        - exec/shell/bash → ShellEngine
        - web_search/web_fetch/browse → BrowserEngine
        - code_open/code_edit → VSCodeEngine
        - claude_code → ClaudeCodeController
        - read_file/write_file → ShellEngine（经审计）
        
        全部流程：
        1. 审计 pre_check
        2. 路由到对应引擎
        3. 审计 monitor（实时）
        4. 审计 post_audit
        5. 返回结果
        """
        ...

2. 修改 liteclaw/agent/loop.py：
   - LLM 返回 tool_use 时，通过 ExecutionManager.execute_tool() 执行
   - 被拦截的操作，向 LLM 返回 "Operation blocked: {reason}"
   - LLM 可以根据拦截原因调整策略

3. 实现 Telegram 审批通道：
   - WRITE_REMOTE 操作发送确认请求到 Telegram
   - 用户回复 ✅ → 继续执行
   - 用户回复 ❌ → 取消操作
   - 超时 5 分钟 → 自动取消

## Acceptance Criteria
- [ ] Shell 命令通过 Agent 工具调用可执行
- [ ] 浏览器操作通过 Agent 工具调用可执行
- [ ] 危险操作被拦截，LLM 收到拦截消息
- [ ] WRITE_REMOTE 操作弹出 Telegram 确认
- [ ] 审计日志记录完整
- [ ] Agent loop 中工具调用最大 20 次迭代（防止无限循环）

## Constraints
- 浏览器操作默认 headless
- 所有执行超时 60 秒
- 工具结果超过 4000 字符自动截断
```

---

## Phase 6: Storage Layer

### TASK-21: SQLite + FTS5 Database

```markdown
# TASK-21: 实现核心数据库层

## Background
LiteClaw 的所有持久化数据存储在 SQLite 中：
对话历史（含 FTS5 索引）、成本记录、Rate Limit 状态。

## Task
在 liteclaw/storage/database.py 中实现：

class Database:
    """
    LiteClaw 核心数据库。SQLite + FTS5。
    
    三个数据库文件：
    - messages.db: 对话历史 + FTS5 全文索引 + 向量索引
    - cost_log.db: API 调用成本记录
    - state.db: Rate Limit 计数器、系统状态
    """
    
    def __init__(self, data_dir: str):
        """data_dir: 数据目录路径，如 ~/.liteclaw/data/"""
        ...
    
    def init_tables(self):
        """创建所有表和索引（如不存在）"""
        ...
    
    # ── 对话历史 ──
    def save_message(self, session_id: str, role: str, content: str,
                     model: str = None, tokens: int = None):
        """保存消息并自动更新 FTS 索引"""
        ...
    
    def get_session_messages(self, session_id: str, 
                             limit: int = None) -> list[Message]:
        """获取会话消息"""
        ...
    
    def search_messages(self, query: str, limit: int = 10) -> list[Message]:
        """FTS5 全文搜索"""
        ...
    
    # ── 成本记录 ──
    def log_api_call(self, session_id: str, model: str, 
                     input_tokens: int, output_tokens: int,
                     cost_usd: float, task_level: str):
        """记录一次 API 调用的成本"""
        ...
    
    def get_cost_summary(self, period: str = 'today') -> CostSummary:
        """
        获取成本汇总。
        period: 'today' | 'week' | 'month' | '2026-02-12'
        """
        ...
    
    # ── Rate Limit 状态 ──
    def save_rate_state(self, model_id: str, state: dict):
        """保存 Rate Limit 计数器状态"""
        ...
    
    def load_rate_state(self, model_id: str) -> dict:
        """加载 Rate Limit 计数器状态"""
        ...

## Database Schema

-- messages.db
CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL,
    role TEXT NOT NULL CHECK(role IN ('user','assistant','tool','system')),
    content TEXT NOT NULL,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    model_used TEXT,
    token_count INTEGER,
    cost_usd REAL
);

CREATE INDEX idx_messages_session ON messages(session_id);
CREATE INDEX idx_messages_timestamp ON messages(timestamp);

CREATE VIRTUAL TABLE messages_fts USING fts5(
    content,
    content=messages,
    content_rowid=id,
    tokenize='unicode61'
);

-- cost_log.db
CREATE TABLE api_calls (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    session_id TEXT,
    model TEXT NOT NULL,
    task_level TEXT,
    input_tokens INTEGER,
    output_tokens INTEGER,
    thinking_tokens INTEGER DEFAULT 0,
    cost_usd REAL,
    was_fallback BOOLEAN DEFAULT FALSE
);

-- state.db
CREATE TABLE rate_limits (
    model_id TEXT PRIMARY KEY,
    state_json TEXT,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

## Test Requirements
创建 tests/test_database.py：
- test_save_and_retrieve: 保存消息后能取回
- test_fts_search: FTS5 搜索正确返回
- test_cost_summary_today: 今日成本汇总准确
- test_cost_summary_month: 月度成本汇总准确
- test_rate_state_persistence: Rate Limit 状态保存/加载正确
- test_concurrent_writes: 并发写入不报错（SQLite WAL 模式）

## Acceptance Criteria
- [ ] 6 个测试全部通过
- [ ] 数据库使用 WAL 模式（并发友好）
- [ ] FTS5 搜索支持中英文
- [ ] 代码不超过 250 行

## Constraints
- 使用 Python 标准库 sqlite3
- 数据库文件自动创建在 data_dir 下
- 所有写操作使用事务
```

### TASK-22: Obsidian Writer

```markdown
# TASK-22: 实现 Obsidian 对话和审计写入器

## Background
所有对话和审计记录以 Markdown 格式写入 Obsidian vault，
方便人工查阅和搜索。

## Task
在 liteclaw/storage/obsidian.py 中实现：

class ObsidianWriter:
    """
    Obsidian vault 写入器。
    
    目录结构：
    {vault}/LiteClaw/
    ├── conversations/
    │   └── 2026-02-12/
    │       ├── 14-30-web-research.md
    │       └── 15-45-code-review.md
    ├── audit/           ← TASK-16 的 AuditLogger 写入
    ├── memory/
    │   ├── MEMORY.md
    │   └── preferences.md
    ├── screenshots/
    └── logs/
        └── cost-report-2026-02.md
    """
    
    def __init__(self, vault_path: str):
        """vault_path: Obsidian vault 根目录"""
        ...
    
    def save_conversation(self, session: Session) -> str:
        """
        保存完整对话为 Markdown。
        
        文件格式：
        ---
        date: 2026-02-12T14:30:00+09:00
        session_id: sess_abc123
        models_used: [gemini-3-flash, gemini-flash-lite]
        total_tokens: 12450
        estimated_cost: $0.008
        task_levels: [MODERATE, SIMPLE]
        tags: [web-research, AI]
        ---
        
        # Web Research: AI Agent Architecture
        
        ## Turn 1
        **User** (14:30:15):
        帮我搜索主流 AI Agent 架构
        
        **Assistant** (14:30:22) `[gemini-3-flash | 2340 tokens]`:
        ...
        """
        ...
    
    def save_conversation_turn(self, session_id: str, role: str,
                                content: str, metadata: dict):
        """实时追加单轮对话到当前会话文件（增量写入）"""
        ...
    
    def save_memory(self, key: str, content: str):
        """保存/更新记忆文件"""
        ...
    
    def generate_cost_report(self, period: str, 
                              cost_data: CostSummary) -> str:
        """生成成本报告 Markdown"""
        ...

## Test Requirements
创建 tests/test_obsidian.py：
- test_conversation_format: 生成的 Markdown 含 YAML frontmatter
- test_incremental_write: 增量追加不覆盖已有内容
- test_directory_auto_create: 目录自动创建
- test_filename_format: 文件名格式正确（HH-MM-描述.md）
- test_cost_report: 成本报告生成正确

## Acceptance Criteria
- [ ] 5 个测试全部通过
- [ ] Markdown 可被 Obsidian 正确渲染
- [ ] YAML frontmatter 格式正确
- [ ] 文件名不含特殊字符
- [ ] 代码不超过 200 行

## Constraints
- 文件编码 UTF-8
- 时间戳用用户本地时区（config 中配置，默认 KST）
- 文件名中的描述从对话第一条消息自动提取（前 20 字）
```

### TASK-23: Cost Tracker

```markdown
# TASK-23: 实现成本追踪器

## Background
实时追踪每次 API 调用的 token 消耗和费用。
提供日/周/月统计，供 Dashboard 显示。

## Task
在 liteclaw/storage/cost_tracker.py 中实现：

class CostTracker:
    """
    成本追踪器。记录每次 API 调用，提供统计查询。
    """
    
    # 各模型定价表（可通过 config 更新）
    PRICING = {
        'gemini-flash-lite': {'input': 0.0, 'output': 0.0},  # 免费
        'gemini-3-flash': {'input': 0.15, 'output': 0.60},   # $/M tokens
        'gemini-3-pro': {'input': 1.25, 'output': 5.00},
        'claude-sonnet': {'input': 3.00, 'output': 15.00},
        'claude-opus': {'input': 15.00, 'output': 75.00},
    }
    
    def __init__(self, db: Database):
        ...
    
    def record(self, session_id: str, model: str, 
               input_tokens: int, output_tokens: int,
               task_level: str, was_fallback: bool = False):
        """记录一次 API 调用，自动计算费用"""
        ...
    
    def estimate_cost(self, model: str, input_tokens: int, 
                      output_tokens: int) -> float:
        """预估费用（调用前用）"""
        ...
    
    def get_today(self) -> CostSummary:
        """今日统计"""
        ...
    
    def get_week(self) -> CostSummary:
        """本周统计"""
        ...
    
    def get_month(self) -> CostSummary:
        """本月统计"""
        ...
    
    def get_by_model(self, period: str = 'today') -> dict[str, ModelCost]:
        """按模型分组统计"""
        ...
    
    def get_by_task_level(self, period: str = 'today') -> dict[str, float]:
        """按任务级别分组统计"""
        ...

@dataclass
class CostSummary:
    total_calls: int
    total_input_tokens: int
    total_output_tokens: int
    total_cost_usd: float
    by_model: dict[str, ModelCost]
    period: str

## Test Requirements
创建 tests/test_cost_tracker.py：
- test_record_and_query: 记录后查询正确
- test_cost_calculation: 费用计算准确（手算验证）
- test_today_summary: 今日汇总正确
- test_by_model: 按模型分组正确
- test_free_model: 免费模型费用为 0

## Acceptance Criteria
- [ ] 5 个测试全部通过
- [ ] 费用计算精确到小数点后 6 位
- [ ] 代码不超过 120 行
```

---

## Phase 7: UI Layer

### TASK-24: Gradio Base Framework + Chat Interface

```markdown
# TASK-24: 实现 Gradio Web UI 基础框架和 Chat 界面

## Background
LiteClaw 需要一个 Web 管理界面。使用 Gradio 快速搭建 MVP。
Chat 界面是最核心的功能。

## Task
在 liteclaw/ui/app.py 中实现：

def create_app(config, agent_loop, cost_tracker, rate_limiter) -> gr.Blocks:
    """
    创建 Gradio Web 应用。
    包含 5 个 Tab：Chat | Models | Dashboard | Skills | Settings
    """

Chat Tab 功能：
1. 对话输入框 + 发送按钮
2. 对话历史显示（Markdown 渲染）
3. 每条回复显示元信息：使用的模型、token 消耗、成本、任务级别
4. 对话历史轮次勾选框（手动选择模式）
   - 每轮对话旁边有 checkbox
   - 勾选 = 发送给模型，取消 = 不发送
   - 默认按动态窗口规则自动勾选
5. 会话管理：新建会话、切换会话、删除会话
6. 当前模型和 Rate Limit 状态显示

## Startup Method
在 liteclaw/cli/commands.py 中添加命令：
    liteclaw ui --port 7860

## Test Requirements
- test_app_launches: Gradio 应用能启动不报错
- test_chat_response: 发送消息能收到回复

## Acceptance Criteria
- [ ] `liteclaw ui` 能启动 Web 界面
- [ ] Chat 界面可以发送消息和接收回复
- [ ] 每条回复显示模型名和 token 消耗
- [ ] 对话历史有轮次勾选框
- [ ] 代码不超过 200 行

## Dependencies
- gradio>=4.0
```

### TASK-25: Models Management Interface

```markdown
# TASK-25: 实现 Models 管理界面

## Background
用户需要在 UI 中管理模型：增删改、设置限额、查看状态。

## Task
在 Chat Tab 的基础上，实现 Models Tab：

Models Tab 功能：
1. 已配置模型列表
   - 每个模型显示：名称、Provider、分配级别、启用/禁用状态
   - RPM/TPM/RPD 进度条（实时显示当前使用量/限额）
   - 重置倒计时（如 "Reset: 4:00 PM KST (2h 15m)"）

2. 添加模型表单
   - Provider 下拉选择（Google/Anthropic/OpenAI/Custom）
   - Model ID 输入
   - 分配级别选择（SIMPLE/MODERATE/COMPLEX）
   - 限额设置（RPM、TPM、RPD、每日预算）
   - 重置时间配置

3. 编辑/删除模型
   - 点击模型卡片进入编辑模式
   - 可禁用（不删除配置）
   - 可删除

4. 模型状态刷新按钮

## Acceptance Criteria
- [ ] 模型列表正确显示
- [ ] 可以添加新模型
- [ ] 可以编辑和删除模型
- [ ] Rate Limit 进度条实时更新
- [ ] 重置倒计时显示正确
- [ ] 代码不超过 250 行
```

### TASK-26: Dashboard Monitoring Interface

```markdown
# TASK-26: 实现 Dashboard 监控界面

## Background
成本监控和系统状态总览。

## Task
在 Models Tab 的基础上，实现 Dashboard Tab：

Dashboard Tab 功能：
1. 今日成本卡片
   - 总花费（$X.XX）
   - 月累计花费
   - 月预算进度条

2. 按模型统计表
   - 模型名 | 调用次数 | Token 消耗 | 费用
   - 可切换时间范围：今日/本周/本月

3. Rate Limit 状态面板
   - 每个模型的 RPM/TPM/RPD 进度条
   - 颜色编码：绿（< 60%）→ 黄（60-80%）→ 红（> 80%）

4. 最近会话列表
   - 时间 | 描述 | 轮次 | 花费
   - 点击可跳转到对话

5. 成本趋势图（可选）
   - 最近 7 天每日花费折线图

## Acceptance Criteria
- [ ] 今日成本显示正确
- [ ] 按模型统计表正确
- [ ] Rate Limit 进度条实时更新
- [ ] 最近会话列表可点击
- [ ] 代码不超过 200 行
```

### TASK-27: Skills Management Interface

````markdown
# TASK-27: 实现 Skills 管理界面

## Background
管理人类 SOP 设计的 Skill 文件。

## Task
实现 Skills Tab：

Skills Tab 功能：
1. Skill 列表
   - 名称 | 描述 | 状态（always-loaded / available / disabled）
   - 来源目录路径

2. Skill 内容查看器
   - 选择一个 Skill，显示其 SKILL.md 内容
   - Markdown 渲染

3. Skill 编辑器
   - 可以在线编辑 Skill 的 Markdown 内容
   - 保存按钮（写回文件）

4. Skill 状态切换
   - always-loaded：每次对话都加载
   - available：Agent 按需加载
   - disabled：禁用

5. 新建 Skill
   - 输入名称，自动创建目录和 SKILL.md 模板

## Skill Template
```markdown
---
name: new-skill
description: 描述这个 Skill 的用途
status: available
---

# Skill: New Skill

## 目的
...

## 前置条件
...

## 执行步骤（SOP）
1. ...
2. ...

## 质量标准
...

## 失败处理
...
````

## Acceptance Criteria

- [ ] Skill 列表正确显示
- [ ] 可以查看 Skill 内容
- [ ] 可以编辑并保存
- [ ] 可以新建 Skill（自动创建模板）
- [ ] 状态切换生效
- [ ] 代码不超过 180 行

````

### TASK-28: Settings + Audit Viewer Interface

```markdown
# TASK-28: 实现 Settings 和审计查看界面

## Background
系统配置管理 + 审计日志查看。

## Task
实现 Settings Tab（含审计子面板）：

Settings 功能：
1. 通道配置
   - Telegram: Bot Token（masked显示）、允许的用户 ID
   - Discord: Bot Token、允许的服务器/频道
   - 启用/禁用开关

2. 存储配置
   - Obsidian vault 路径
   - 数据目录路径
   - 时区设置（默认 Asia/Seoul）

3. 安全设置
   - API Key 管理（只显示服务名和状态，不显示 Key 值）
   - 添加/更新 Key（输入后立即存入 CredentialStore，界面不保留）
   - Agent 防火墙开关

4. 模型默认配置
   - 默认温度
   - 最大工具迭代次数
   - 输出 max_tokens 限制（你发现的"压缩输出提升质量"策略）

审计查看功能：
5. 审计日志浏览器
   - 日期选择器
   - 当日操作列表：时间 | 操作 | 风险等级 | 状态
   - 点击查看详情（显示完整 Markdown 审计日志）
   - 被拦截的操作用红色标记

6. 审计统计
   - 本周操作总数
   - 各风险等级分布
   - 拦截次数

## Acceptance Criteria
- [ ] 通道配置可编辑并保存
- [ ] API Key 输入后不在界面显示明文
- [ ] Obsidian 路径可配置
- [ ] 审计日志可按日期浏览
- [ ] 被拦截操作红色标记
- [ ] 代码不超过 250 行

## Constraints
- API Key 输入框用 password 类型
- 保存配置时不重启服务（热更新）
````

---

## Total: 29 Task Cards (TASK-00 ~ TASK-28)

Each card = one Claude Code session · Each card = 30-90 min · Each card = clear acceptance criteria

**Estimated total: 29 × 1hr ≈ 29hrs ≈ 5-6 working days (focused dev)**

---

## Claude Code Usage SOP

```
1. 设置环境变量
   export ANTHROPIC_API_KEY=xxx
   
2. 进入项目目录
   cd ~/projects/liteclaw
   
3. 开新会话，给一张任务卡
   claude -p "$(cat tasks/TASK-01.md)"
   
4. Claude Code 完成后，人工验收：
   - 跑测试：pytest tests/test_xxx.py -v
   - 检查代码行数
   - 检查是否修改了不该改的文件
   
5. 验收通过 → git commit → 下一张卡
   验收失败 → 开新会话重做（不要在同一会话修补！）
   
6. 重复直到所有卡完成
```