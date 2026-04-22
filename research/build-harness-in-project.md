---
title: 如何在项目中构建 Harness：AI Agent 工程化实战指南
date: 2026-04-22 10:40:00
categories:
  - AI
  - 技术
tags:
  - Harness Engineering
  - AI Agent
  - 工程实践
  - 架构设计
---

> 从「能用的 Agent」到「可靠的 Agent」—— 一份可落地的 Harness 构建指南

---

## 前言

在 AI Agent 开发中，我们经常遇到这样的困境：

- Agent 在本地测试时表现完美，上线后却频繁出错
- 模型调用成本不可控，一不小心就烧光预算
- 敏感操作没有审批，安全漏洞频出
- 对话上下文管理混乱，Agent 很快「失忆」

这些问题的根源不在于模型能力，而在于缺乏一个**可靠的 Harness 层**。

本文将带你从零开始，在项目中构建一个生产级的 Harness 系统。

---

## 一、Harness 的核心组件

一个完整的 Harness 系统包含五大支柱：

```
┌─────────────────────────────────────────────────────┐
│                    Harness System                    │
├─────────────┬─────────────┬─────────────────────────┤
│   工具层    │   上下文    │      安全审批层         │
│   (Tools)   │   管理层    │    (Approval Layer)     │
├─────────────┴─────────────┴─────────────────────────┤
│              会话管理 + 资源治理                       │
└─────────────────────────────────────────────────────┘
```

### 1.1 工具层（Tools）

工具是 Agent 与外部世界交互的接口。一个好的工具系统需要：

- **类型安全**：每个工具都有明确的输入/输出 schema
- **错误处理**：工具执行失败时提供清晰的错误信息
- **超时控制**：防止工具调用无限挂起

**代码示例**：

```python
from dataclasses import dataclass
from typing import Any, Optional
from enum import Enum

class ToolResultStatus(Enum):
    SUCCESS = "success"
    ERROR = "error"
    TIMEOUT = "timeout"

@dataclass
class ToolResult:
    status: ToolResultStatus
    data: Any
    error_message: Optional[str] = None
    execution_time_ms: int = 0

class Tool:
    """工具基类"""
    
    def __init__(self, name: str, description: str, timeout_ms: int = 30000):
        self.name = name
        self.description = description
        self.timeout_ms = timeout_ms
    
    async def execute(self, **params) -> ToolResult:
        """执行工具，子类必须实现"""
        raise NotImplementedError
    
    def get_schema(self) -> dict:
        """返回工具的 JSON Schema"""
        return {
            "name": self.name,
            "description": self.description,
            "parameters": self._get_parameters_schema()
        }
    
    def _get_parameters_schema(self) -> dict:
        """子类实现参数 schema"""
        return {}


class FileReadTool(Tool):
    """文件读取工具示例"""
    
    def __init__(self):
        super().__init__(
            name="file_read",
            description="读取文件内容",
            timeout_ms=5000
        )
    
    def _get_parameters_schema(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "文件路径"
                },
                "max_size": {
                    "type": "integer",
                    "description": "最大读取字节数",
                    "default": 100000
                }
            },
            "required": ["path"]
        }
    
    async def execute(self, path: str, max_size: int = 100000) -> ToolResult:
        import asyncio
        import time
        
        start_time = time.time()
        
        try:
            # 模拟异步文件读取
            content = await asyncio.wait_for(
                self._read_file(path, max_size),
                timeout=self.timeout_ms / 1000
            )
            
            execution_time = int((time.time() - start_time) * 1000)
            
            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                data={"content": content, "path": path},
                execution_time_ms=execution_time
            )
            
        except asyncio.TimeoutError:
            return ToolResult(
                status=ToolResultStatus.TIMEOUT,
                data=None,
                error_message=f"读取文件超时（>{self.timeout_ms}ms）",
                execution_time_ms=self.timeout_ms
            )
        except Exception as e:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                data=None,
                error_message=str(e),
                execution_time_ms=int((time.time() - start_time) * 1000)
            )
    
    async def _read_file(self, path: str, max_size: int) -> str:
        # 实际实现...
        await asyncio.sleep(0.1)  # 模拟IO
        return f"Content of {path}"
```

### 1.2 上下文管理层

上下文管理是 Harness 的核心。需要处理：

- **系统提示构建**：动态注入项目信息、用户偏好
- **对话历史压缩**：防止 token 超限
- **关键文件注入**：自动读取相关文件内容

```python
class ContextManager:
    """上下文管理器"""
    
    def __init__(self, max_tokens: int = 128000):
        self.max_tokens = max_tokens
        self.system_prompt_template = """
You are an AI assistant helping with software development.

Project Context:
{project_context}

User Preferences:
{user_preferences}

Available Tools:
{tools_description}

Guidelines:
1. Always check file content before making changes
2. Run tests after modifications
3. Ask for confirmation before destructive operations
"""
    
    def build_system_prompt(
        self,
        project_info: dict,
        user_prefs: dict,
        tools: list[Tool]
    ) -> str:
        """构建系统提示"""
        
        project_context = self._format_project_info(project_info)
        user_preferences = self._format_user_prefs(user_prefs)
        tools_description = self._format_tools(tools)
        
        return self.system_prompt_template.format(
            project_context=project_context,
            user_preferences=user_preferences,
            tools_description=tools_description
        )
    
    def compress_conversation(
        self,
        messages: list[dict],
        max_messages: int = 50
    ) -> list[dict]:
        """压缩对话历史"""
        
        if len(messages) <= max_messages:
            return messages
        
        # 保留开头（系统提示+初始几轮）和结尾（最近几轮）
        head_count = 10
        tail_count = max_messages - head_count
        
        head = messages[:head_count]
        tail = messages[-tail_count:]
        
        # 中间部分用摘要替代
        middle_summary = self._summarize_messages(
            messages[head_count:-tail_count]
        )
        
        return head + [{"role": "system", "content": middle_summary}] + tail
    
    def _format_project_info(self, info: dict) -> str:
        return f"""
- Project: {info.get('name', 'Unknown')}
- Language: {info.get('language', 'Unknown')}
- Structure: {info.get('structure', 'Unknown')}
"""
    
    def _format_user_prefs(self, prefs: dict) -> str:
        return "\n".join([
            f"- {k}: {v}" for k, v in prefs.items()
        ])
    
    def _format_tools(self, tools: list[Tool]) -> str:
        return "\n".join([
            f"- {tool.name}: {tool.description}"
            for tool in tools
        ])
    
    def _summarize_messages(self, messages: list[dict]) -> str:
        # 实际应用中调用 LLM 进行摘要
        return f"[Summary of {len(messages)} messages...]"
```

### 1.3 安全审批层

这是 Harness 的安全闸门。所有危险操作都需要用户确认。

```python
from enum import Enum
from typing import Callable, Optional
import asyncio

class ApprovalMode(Enum):
    """审批模式"""
    AUTO = "auto"           # 自动执行（仅非危险操作）
    SMART = "smart"         # 智能审批（基于风险评分）
    MANUAL = "manual"       # 全部手动审批
    OFF = "off"             # 关闭审批（仅开发环境）

class RiskLevel(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    CRITICAL = 4

class ApprovalManager:
    """审批管理器"""
    
    # 危险操作关键词
    DANGEROUS_PATTERNS = [
        ("rm -rf", RiskLevel.CRITICAL),
        ("DROP TABLE", RiskLevel.CRITICAL),
        ("DELETE FROM", RiskLevel.HIGH),
        ("curl.*|.*sh", RiskLevel.HIGH),
        ("eval(", RiskLevel.HIGH),
        ("exec(", RiskLevel.HIGH),
    ]
    
    def __init__(
        self,
        mode: ApprovalMode = ApprovalMode.SMART,
        timeout_seconds: int = 60
    ):
        self.mode = mode
        self.timeout_seconds = timeout_seconds
        self.approval_callback: Optional[Callable] = None
    
    def set_approval_callback(self, callback: Callable):
        """设置审批回调函数"""
        self.approval_callback = callback
    
    async def check_approval(
        self,
        tool_name: str,
        params: dict,
        tool_description: str = ""
    ) -> bool:
        """
        检查操作是否需要审批
        返回 True 表示可以继续执行
        """
        
        if self.mode == ApprovalMode.OFF:
            return True
        
        # 评估风险等级
        risk_level = self._assess_risk(tool_name, params, tool_description)
        
        if self.mode == ApprovalMode.AUTO:
            # 仅允许低风险操作
            return risk_level == RiskLevel.LOW
        
        if self.mode == ApprovalMode.SMART:
            # 中等风险以下自动通过
            if risk_level in [RiskLevel.LOW, RiskLevel.MEDIUM]:
                return True
        
        # 需要人工审批
        return await self._request_approval(
            tool_name, params, tool_description, risk_level
        )
    
    def _assess_risk(
        self,
        tool_name: str,
        params: dict,
        description: str
    ) -> RiskLevel:
        """评估操作风险等级"""
        
        # 将参数转为字符串用于模式匹配
        params_str = str(params).lower()
        full_text = f"{tool_name} {params_str} {description}".lower()
        
        max_risk = RiskLevel.LOW
        
        for pattern, risk in self.DANGEROUS_PATTERNS:
            import re
            if re.search(pattern, full_text):
                max_risk = max(max_risk, risk)
        
        # 文件操作风险评估
        if tool_name in ["file_write", "file_delete"]:
            path = params.get("path", "")
            if any(dangerous in path for dangerous in ["/etc", "/usr", "/bin"]):
                max_risk = max(max_risk, RiskLevel.HIGH)
        
        return max_risk
    
    async def _request_approval(
        self,
        tool_name: str,
        params: dict,
        description: str,
        risk_level: RiskLevel
    ) -> bool:
        """请求用户审批"""
        
        if not self.approval_callback:
            raise RuntimeError("Approval callback not set")
        
        # 构建审批提示
        prompt = f"""
⚠️ **需要您的确认**

操作: {tool_name}
风险等级: {risk_level.name}
描述: {description}
参数: {params}

是否允许执行？(yes/no/always)
"""
        
        try:
            result = await asyncio.wait_for(
                self.approval_callback(prompt),
                timeout=self.timeout_seconds
            )
            return result in ["yes", "always", "y"]
        except asyncio.TimeoutError:
            print("审批超时，操作已取消")
            return False
```

### 1.4 会话管理系统

支持任务中断、恢复和持久化。

```python
import json
import uuid
from datetime import datetime
from pathlib import Path

class Session:
    """会话对象"""
    
    def __init__(self, session_id: Optional[str] = None):
        self.session_id = session_id or str(uuid.uuid4())
        self.created_at = datetime.now()
        self.updated_at = datetime.now()
        self.messages: list[dict] = []
        self.state: dict = {}
        self.metadata: dict = {}
    
    def add_message(self, role: str, content: str, metadata: dict = None):
        """添加消息"""
        self.messages.append({
            "role": role,
            "content": content,
            "timestamp": datetime.now().isoformat(),
            "metadata": metadata or {}
        })
        self.updated_at = datetime.now()
    
    def to_dict(self) -> dict:
        return {
            "session_id": self.session_id,
            "created_at": self.created_at.isoformat(),
            "updated_at": self.updated_at.isoformat(),
            "messages": self.messages,
            "state": self.state,
            "metadata": self.metadata
        }
    
    @classmethod
    def from_dict(cls, data: dict) -> "Session":
        session = cls(data["session_id"])
        session.created_at = datetime.fromisoformat(data["created_at"])
        session.updated_at = datetime.fromisoformat(data["updated_at"])
        session.messages = data["messages"]
        session.state = data["state"]
        session.metadata = data["metadata"]
        return session


class SessionManager:
    """会话管理器"""
    
    def __init__(self, storage_path: str = "./sessions"):
        self.storage_path = Path(storage_path)
        self.storage_path.mkdir(exist_ok=True)
        self.active_sessions: dict[str, Session] = {}
    
    def create_session(self) -> Session:
        """创建新会话"""
        session = Session()
        self.active_sessions[session.session_id] = session
        return session
    
    def get_session(self, session_id: str) -> Optional[Session]:
        """获取会话"""
        # 先检查活跃会话
        if session_id in self.active_sessions:
            return self.active_sessions[session_id]
        
        # 尝试从磁盘加载
        return self._load_session(session_id)
    
    def save_session(self, session_id: str):
        """保存会话到磁盘"""
        if session_id not in self.active_sessions:
            return
        
        session = self.active_sessions[session_id]
        file_path = self.storage_path / f"{session_id}.json"
        
        with open(file_path, "w", encoding="utf-8") as f:
            json.dump(session.to_dict(), f, ensure_ascii=False, indent=2)
    
    def _load_session(self, session_id: str) -> Optional[Session]:
        """从磁盘加载会话"""
        file_path = self.storage_path / f"{session_id}.json"
        
        if not file_path.exists():
            return None
        
        with open(file_path, "r", encoding="utf-8") as f:
            data = json.load(f)
            session = Session.from_dict(data)
            self.active_sessions[session_id] = session
            return session
    
    def list_sessions(self) -> list[dict]:
        """列出所有会话"""
        sessions = []
        
        for file_path in self.storage_path.glob("*.json"):
            with open(file_path, "r", encoding="utf-8") as f:
                data = json.load(f)
                sessions.append({
                    "session_id": data["session_id"],
                    "created_at": data["created_at"],
                    "updated_at": data["updated_at"],
                    "message_count": len(data["messages"])
                })
        
        return sorted(sessions, key=lambda x: x["updated_at"], reverse=True)
```

### 1.5 资源治理层

控制成本和防止滥用。

```python
class ResourceGovernor:
    """资源治理器"""
    
    def __init__(
        self,
        max_tokens_per_session: int = 500000,
        max_tools_per_minute: int = 30,
        max_cost_per_session: float = 10.0  # USD
    ):
        self.max_tokens = max_tokens_per_session
        self.max_tools_per_minute = max_tools_per_minute
        self.max_cost = max_cost_per_session
        
        self.token_usage = 0
        self.tool_calls = []
        self.estimated_cost = 0.0
    
    def record_token_usage(self, tokens: int):
        """记录 token 使用"""
        self.token_usage += tokens
        
        if self.token_usage > self.max_tokens * 0.8:
            print(f"⚠️ Token 使用超过 80%: {self.token_usage}/{self.max_tokens}")
        
        if self.token_usage > self.max_tokens:
            raise ResourceExhaustedError(
                f"Token 使用超限: {self.token_usage}/{self.max_tokens}"
            )
    
    def record_tool_call(self, tool_name: str):
        """记录工具调用"""
        from datetime import datetime
        
        now = datetime.now()
        self.tool_calls.append({"tool": tool_name, "time": now})
        
        # 清理1分钟前的记录
        self.tool_calls = [
            call for call in self.tool_calls
            if (now - call["time"]).seconds < 60
        ]
        
        if len(self.tool_calls) > self.max_tools_per_minute:
            raise RateLimitError(
                f"工具调用频率超限: {len(self.tool_calls)}/{self.max_tools_per_minute} per minute"
            )
    
    def estimate_cost(self, model: str, tokens: int) -> float:
        """估算成本"""
        # 简化的成本估算
        rates = {
            "gpt-4": 0.03,           # per 1K tokens
            "gpt-3.5-turbo": 0.002,
            "claude-3-opus": 0.015,
        }
        
        rate = rates.get(model, 0.01)
        cost = (tokens / 1000) * rate
        
        self.estimated_cost += cost
        
        if self.estimated_cost > self.max_cost * 0.8:
            print(f"⚠️ 预估成本超过 80%: ${self.estimated_cost:.2f}/${self.max_cost:.2f}")
        
        return cost
    
    def get_usage_report(self) -> dict:
        """获取使用报告"""
        return {
            "token_usage": self.token_usage,
            "token_limit": self.max_tokens,
            "token_percentage": self.token_usage / self.max_tokens * 100,
            "tool_calls_last_minute": len(self.tool_calls),
            "estimated_cost": self.estimated_cost,
            "cost_limit": self.max_cost,
            "cost_percentage": self.estimated_cost / self.max_cost * 100
        }


class ResourceExhaustedError(Exception):
    pass

class RateLimitError(Exception):
    pass
```

---

## 二、整合：构建完整的 Harness

```python
class Harness:
    """Harness 主类"""
    
    def __init__(self, config: dict = None):
        self.config = config or {}
        
        # 初始化各组件
        self.tools: dict[str, Tool] = {}
        self.context_manager = ContextManager()
        self.approval_manager = ApprovalManager(
            mode=ApprovalMode.SMART
        )
        self.session_manager = SessionManager()
        self.resource_governor = ResourceGovernor()
        
        # 注册默认工具
        self._register_default_tools()
    
    def _register_default_tools(self):
        """注册默认工具"""
        self.register_tool(FileReadTool())
        # 注册更多工具...
    
    def register_tool(self, tool: Tool):
        """注册工具"""
        self.tools[tool.name] = tool
    
    async def execute(
        self,
        session_id: str,
        user_input: str,
        project_info: dict = None
    ) -> str:
        """
        执行用户请求
        
        完整流程：
        1. 获取或创建会话
        2. 构建上下文
        3. 调用 LLM
        4. 解析工具调用
        5. 安全审批
        6. 执行工具
        7. 返回结果
        """
        
        # 1. 获取会话
        session = self.session_manager.get_session(session_id)
        if not session:
            session = self.session_manager.create_session()
            session_id = session.session_id
        
        # 2. 记录用户输入
        session.add_message("user", user_input)
        
        # 3. 构建系统提示
        system_prompt = self.context_manager.build_system_prompt(
            project_info=project_info or {},
            user_prefs=self._load_user_prefs(),
            tools=list(self.tools.values())
        )
        
        # 4. 准备消息列表
        messages = [{"role": "system", "content": system_prompt}]
        messages.extend(session.messages)
        
        # 5. 压缩上下文
        messages = self.context_manager.compress_conversation(messages)
        
        # 6. 调用 LLM（简化示例）
        llm_response = await self._call_llm(messages)
        
        # 7. 处理工具调用
        if self._has_tool_calls(llm_response):
            tool_results = await self._execute_tool_calls(
                llm_response["tool_calls"],
                session
            )
            
            # 构建最终响应
            final_response = await self._generate_final_response(
                messages, tool_results
            )
        else:
            final_response = llm_response["content"]
        
        # 8. 记录助手响应
        session.add_message("assistant", final_response)
        
        # 9. 保存会话
        self.session_manager.save_session(session_id)
        
        return final_response
    
    async def _execute_tool_calls(
        self,
        tool_calls: list[dict],
        session: Session
    ) -> list[dict]:
        """执行工具调用"""
        results = []
        
        for call in tool_calls:
            tool_name = call["function"]["name"]
            params = json.loads(call["function"]["arguments"])
            
            if tool_name not in self.tools:
                results.append({
                    "tool": tool_name,
                    "error": f"Unknown tool: {tool_name}"
                })
                continue
            
            tool = self.tools[tool_name]
            
            # 安全审批
            approved = await self.approval_manager.check_approval(
                tool_name, params, tool.description
            )
            
            if not approved:
                results.append({
                    "tool": tool_name,
                    "error": "Operation not approved by user"
                })
                continue
            
            # 资源治理检查
            self.resource_governor.record_tool_call(tool_name)
            
            # 执行工具
            try:
                tool_result = await tool.execute(**params)
                results.append({
                    "tool": tool_name,
                    "result": tool_result
                })
            except Exception as e:
                results.append({
                    "tool": tool_name,
                    "error": str(e)
                })
        
        return results
    
    def get_usage_report(self) -> dict:
        """获取资源使用报告"""
        return self.resource_governor.get_usage_report()
    
    def _load_user_prefs(self) -> dict:
        """加载用户偏好"""
        # 从配置文件或数据库加载
        return {
            "code_style": "pep8",
            "test_framework": "pytest",
            "doc_format": "google"
        }
    
    async def _call_llm(self, messages: list[dict]) -> dict:
        """调用 LLM（简化示例）"""
        # 实际实现中调用 OpenAI/Anthropic API
        return {
            "content": "I'll help you with that task.",
            "tool_calls": []
        }
    
    def _has_tool_calls(self, response: dict) -> bool:
        """检查是否有工具调用"""
        return "tool_calls" in response and len(response["tool_calls"]) > 0
    
    async def _generate_final_response(
        self,
        messages: list[dict],
        tool_results: list[dict]
    ) -> str:
        """生成最终响应"""
        # 实际实现中再次调用 LLM
        return f"Task completed. Results: {tool_results}"
```

---

## 三、最佳实践

### 3.1 渐进式启用

不要在生产环境一次性启用所有功能：

```python
# 阶段1：基础工具 + 手动审批
harness = Harness(config={
    "approval_mode": ApprovalMode.MANUAL,
    "tools": ["file_read", "file_write"]  # 仅基础工具
})

# 阶段2：增加自动审批 + 更多工具
harness.approval_manager.mode = ApprovalMode.SMART
harness.register_tool(ShellExecuteTool())

# 阶段3：启用资源治理
harness.resource_governor.max_tokens_per_session = 1000000
```

### 3.2 工具设计原则

1. **单一职责**：每个工具只做一件事
2. **可测试**：工具逻辑独立于 LLM 调用
3. **幂等性**：相同输入产生相同输出（避免副作用）
4. **详细日志**：记录所有输入输出，便于调试

### 3.3 安全红线

```python
# 永远不要这样做
class DangerousTool(Tool):
    async def execute(self, command: str):
        # ❌ 直接执行任意命令
        import os
        return os.system(command)

# 正确做法
class SafeShellTool(Tool):
    ALLOWED_COMMANDS = ["ls", "cat", "grep", "find"]
    
    async def execute(self, command: str):
        # ✅ 白名单验证
        cmd = command.split()[0]
        if cmd not in self.ALLOWED_COMMANDS:
            raise ValueError(f"Command not allowed: {cmd}")
        
        # ✅ 使用安全沙箱执行
        return await self._execute_in_sandbox(command)
```

### 3.4 监控与告警

```python
class HarnessMonitor:
    """Harness 监控器"""
    
    def __init__(self, harness: Harness):
        self.harness = harness
    
    async def health_check(self) -> dict:
        """健康检查"""
        report = self.harness.get_usage_report()
        
        alerts = []
        
        if report["token_percentage"] > 90:
            alerts.append("Token usage critical")
        
        if report["cost_percentage"] > 90:
            alerts.append("Cost limit approaching")
        
        return {
            "healthy": len(alerts) == 0,
            "alerts": alerts,
            "usage": report
        }
```

---

## 四、从 0 到 1 的 checklist

在你的项目中构建 Harness，按以下顺序进行：

### Week 1: 基础框架
- [ ] 定义 Tool 基类和接口
- [ ] 实现 3-5 个核心工具（file_read, file_write, shell_execute）
- [ ] 添加基础错误处理和超时控制

### Week 2: 上下文管理
- [ ] 实现系统提示动态构建
- [ ] 添加对话历史压缩
- [ ] 集成项目信息注入

### Week 3: 安全层
- [ ] 实现审批管理器
- [ ] 定义危险操作清单
- [ ] 添加审批回调机制

### Week 4: 治理与优化
- [ ] 实现资源治理（token/cost 限制）
- [ ] 添加会话持久化
- [ ] 集成监控告警

### Week 5: 生产准备
- [ ] 完善测试覆盖
- [ ] 性能优化
- [ ] 文档编写

---

## 结语

构建一个可靠的 Harness 不是一蹴而就的，而是一个渐进式演进的过程。

从最简单的工具层开始，逐步添加上下文管理、安全审批、资源治理。每一次迭代都让你的 Agent 更可靠、更可控。

记住：**Harness 不是限制模型，而是为模型提供一个安全、可预测的工作环境。**

当你的 Harness 足够完善时，你会惊喜地发现：Agent 从「能用」变成了「可靠」，从「实验品」变成了「生产工具」。

---

## 参考资源

- [Claude Code 架构分析](https://github.com/anthropics/claude-code)
- [OpenClaw ACP 协议](https://docs.openclaw.ai/acp)
- [Hermes Agent 技能系统](https://hermes-agent.nousresearch.com/docs/skills)

*本文代码示例基于 MIT 协议，可自由使用*
