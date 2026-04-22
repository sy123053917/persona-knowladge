---
title: Harness Engineering 实战：从零构建 AI Agent 的「缰绳与马鞍」
date: 2026-04-22 11:30:00
categories:
  - AI
  - 技术
tags:
  - Harness Engineering
  - AI Agent
  - 工程实践
  - 系统架构
---

> 提示词已经不够用了！深度拆解 AI 下半场密码：Harness Engineering —— 给烈马配上缰绳、马鞍、跑道护栏和反馈镜子

---

## 一、 Harness Engineering 是什么？

### 1.1 一个形象的比喻

想象 AI 大模型是一匹**烈马**：

- 它力大无穷，能日行千里
- 但它野性难驯，随时可能脱缰
- 它不认识路，可能一头撞向悬崖
- 它跑累了就停下，不管任务完成没有

**Harness Engineering** 就是给这匹烈马配上：

| 组件 | 作用 | 工程对应 |
|------|------|----------|
| 🪢 **缰绳** | 控制方向，随时能拽回来 | 工具调用控制、审批机制 |
| 🐴 **马鞍** | 让骑手坐得稳，持久骑行 | 上下文管理、记忆系统 |
| 🚧 **跑道护栏** | 划定边界，防止跑出赛道 | 安全沙箱、权限控制 |
| 🪞 **反馈镜子** | 让骑手看到状态，及时调整 | 监控告警、日志追踪 |

**核心结论**：Harness 不是用来优化模型本身，让它变得更「聪明」；而是构建一整套**运行控制系统**，让它在复杂的真实业务中：**跑得稳、跑得久、绝对不跑偏**。

### 1.2 为什么需要 Harness？

在早期的 AI 开发中，大家喜欢把要求一股脑全写进 System Prompt：

```
"你必须输出合法 JSON"
"不要随意删除文件"
"遇到错误要重试"
"不要陷入无限循环"
```

但大语言模型的本质特征是**非确定性**——同样的输入可能产生不同的输出。Prompt 就像对烈马喊话：「往左！往右！」但它听不听、听多少，完全不可控。

**Harness Engineering 的核心哲学**：[用确定性的代码] 包裹 [非确定性的 AI]

```
┌─────────────────────────────────────────┐
│           确定性代码（Harness）          │
│  ┌─────────────────────────────────┐   │
│  │      非确定性 AI（大模型）        │   │
│  │                                 │   │
│  │    输入 → 推理 → 输出           │   │
│  │      （不可控的黑盒）            │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ✓ 工具调用控制  ✓ 上下文管理           │
│  ✓ 安全审批      ✓ 资源限制             │
│  ✓ 错误恢复      ✓ 监控告警             │
└─────────────────────────────────────────┘
```

---

## 二、 Harness 的五大核心组件

### 2.1 缰绳：工具调用控制系统（Tool Calling Harness）

**缰绳的作用**：控制 Agent 能做什么、不能做什么，随时能拽回来。

#### 问题场景

Agent 拿到工具后可能：
- 调用错误的工具（让计算器写文件）
- 参数格式错误（JSON 解析失败）
- 陷入无限循环（反复调用同一个工具）
- 执行危险操作（`rm -rf /`）

#### Harness 解决方案

```python
from dataclasses import dataclass
from enum import Enum
from typing import Any, Optional, Callable
import json

class ToolRiskLevel(Enum):
    SAFE = "safe"           # 只读操作
    NORMAL = "normal"       # 常规写操作
    DANGEROUS = "dangerous" # 可能影响系统
    CRITICAL = "critical"   # 高风险操作

@dataclass
class ToolCall:
    """标准化的工具调用"""
    tool_name: str
    parameters: dict
    call_id: str
    risk_level: ToolRiskLevel

@dataclass
class ToolResult:
    """标准化的工具执行结果"""
    success: bool
    data: Any
    error: Optional[str] = None
    execution_time_ms: int = 0


class ToolCallingHarness:
    """工具调用控制系统 - 缰绳"""
    
    def __init__(self):
        self.tools: dict[str, Callable] = {}
        self.risk_rules: dict[str, ToolRiskLevel] = {}
        self.call_history: list[ToolCall] = []
        self.max_calls_per_minute = 30
        self.max_recursion_depth = 3
        
    def register_tool(
        self,
        name: str,
        handler: Callable,
        risk_level: ToolRiskLevel = ToolRiskLevel.NORMAL
    ):
        """注册工具"""
        self.tools[name] = handler
        self.risk_rules[name] = risk_level
    
    async def execute_tool_call(
        self,
        call: ToolCall,
        approval_callback: Optional[Callable] = None
    ) -> ToolResult:
        """
        执行工具调用（带完整控制）
        """
        # 1. 检查工具是否存在
        if call.tool_name not in self.tools:
            return ToolResult(
                success=False,
                data=None,
                error=f"Unknown tool: {call.tool_name}"
            )
        
        # 2. 频率限制检查
        if not self._check_rate_limit():
            return ToolResult(
                success=False,
                data=None,
                error="Rate limit exceeded: too many tool calls"
            )
        
        # 3. 递归深度检查
        current_depth = self._get_recursion_depth(call.tool_name)
        if current_depth >= self.max_recursion_depth:
            return ToolResult(
                success=False,
                data=None,
                error=f"Max recursion depth ({self.max_recursion_depth}) reached"
            )
        
        # 4. 风险评估与审批
        risk = self.risk_rules.get(call.tool_name, ToolRiskLevel.NORMAL)
        
        if risk in [ToolRiskLevel.DANGEROUS, ToolRiskLevel.CRITICAL]:
            if approval_callback:
                approved = await approval_callback(call)
                if not approved:
                    return ToolResult(
                        success=False,
                        data=None,
                        error=f"Tool call '{call.tool_name}' not approved by user"
                    )
        
        # 5. 参数校验
        try:
            self._validate_parameters(call)
        except ValueError as e:
            return ToolResult(
                success=False,
                data=None,
                error=f"Parameter validation failed: {e}"
            )
        
        # 6. 执行工具
        self.call_history.append(call)
        
        try:
            import time
            start = time.time()
            
            handler = self.tools[call.tool_name]
            result_data = await handler(**call.parameters)
            
            execution_time = int((time.time() - start) * 1000)
            
            return ToolResult(
                success=True,
                data=result_data,
                execution_time_ms=execution_time
            )
            
        except Exception as e:
            return ToolResult(
                success=False,
                data=None,
                error=f"Execution error: {str(e)}"
            )
    
    def _check_rate_limit(self) -> bool:
        """检查调用频率"""
        from datetime import datetime, timedelta
        
        now = datetime.now()
        one_minute_ago = now - timedelta(minutes=1)
        
        recent_calls = [
            call for call in self.call_history
            if hasattr(call, 'timestamp') and call.timestamp > one_minute_ago
        ]
        
        return len(recent_calls) < self.max_calls_per_minute
    
    def _get_recursion_depth(self, tool_name: str) -> int:
        """检查递归深度"""
        depth = 0
        for call in reversed(self.call_history):
            if call.tool_name == tool_name:
                depth += 1
            else:
                break
        return depth
    
    def _validate_parameters(self, call: ToolCall):
        """参数校验"""
        # 检查必填参数
        # 检查参数类型
        # 检查参数范围
        pass


# 使用示例
async def main():
    harness = ToolCallingHarness()
    
    # 注册工具
    harness.register_tool(
        "file_read",
        lambda path: open(path).read(),
        ToolRiskLevel.SAFE
    )
    
    harness.register_tool(
        "file_delete",
        lambda path: __import__('os').remove(path),
        ToolRiskLevel.DANGEROUS
    )
    
    # 模拟调用
    call = ToolCall(
        tool_name="file_delete",
        parameters={"path": "/tmp/test.txt"},
        call_id="call_001",
        risk_level=ToolRiskLevel.DANGEROUS
    )
    
    # 需要审批的调用
    async def approval_callback(call: ToolCall) -> bool:
        print(f"⚠️ 危险操作请求: {call.tool_name}")
        response = input("允许执行吗? (yes/no): ")
        return response.lower() == "yes"
    
    result = await harness.execute_tool_call(call, approval_callback)
    print(f"结果: {result}")
```

**缰绳的关键控制点**：
1. **白名单机制**：只有注册的工具才能被调用
2. **频率限制**：防止工具调用滥用
3. **递归深度**：防止无限循环
4. **风险分级**：危险操作需要审批
5. **参数校验**：确保输入合法性

---

### 2.2 马鞍：上下文管理系统（Context Management Harness）

**马鞍的作用**：让 AI 坐得稳，能够长时间、长距离地持续工作。

#### 问题场景

Agent 工作一段时间后：
- Token 超限，需要截断对话历史
- 遗忘早期的关键信息
- 搞不清楚当前的工作目录和文件状态
- 不知道用户的偏好和项目规范

#### Harness 解决方案

```python
from typing import List, Dict, Optional
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class ContextWindow:
    """上下文窗口 - 马鞍的座位"""
    max_tokens: int = 128000
    reserved_tokens: int = 4000  # 为输出预留
    messages: List[Dict] = field(default_factory=list)
    
    def add_message(self, role: str, content: str, metadata: Dict = None):
        """添加消息"""
        self.messages.append({
            "role": role,
            "content": content,
            "timestamp": datetime.now().isoformat(),
            "metadata": metadata or {}
        })
        
        # 检查是否需要压缩
        if self.estimate_tokens() > self.max_tokens - self.reserved_tokens:
            self.compress()
    
    def estimate_tokens(self) -> int:
        """估算当前 token 数（简化版）"""
        total = 0
        for msg in self.messages:
            # 简单估算：1个中文字符≈1个token，英文单词≈1.3个token
            content = msg.get("content", "")
            total += len(content)
        return total
    
    def compress(self):
        """压缩上下文 - 马鞍的减震系统"""
        if len(self.messages) <= 20:
            return
        
        # 策略：保留开头（系统提示+初始几轮）和结尾（最近几轮）
        head_count = 10
        tail_count = 10
        
        head = self.messages[:head_count]
        tail = self.messages[-tail_count:]
        
        # 中间部分用摘要替代
        middle = self.messages[head_count:-tail_count]
        summary = self._summarize_messages(middle)
        
        self.messages = head + [{
            "role": "system",
            "content": f"[历史对话摘要] {summary}",
            "timestamp": datetime.now().isoformat()
        }] + tail
    
    def _summarize_messages(self, messages: List[Dict]) -> str:
        """摘要中间消息（实际应调用 LLM）"""
        return f"省略 {len(messages)} 条消息，涉及 {self._extract_topics(messages)}"
    
    def _extract_topics(self, messages: List[Dict]) -> str:
        """提取主题关键词"""
        # 简化实现
        return "文件操作、代码修改、测试运行"


class ContextManagementHarness:
    """上下文管理系统 - 马鞍"""
    
    def __init__(self):
        self.system_prompt_template = """You are an AI assistant working on a software project.

## Project Information
{project_info}

## User Preferences
{user_preferences}

## Current Context
{current_context}

## Guidelines
1. Always check file content before making changes
2. Follow the project's coding style
3. Run tests after modifications
4. Ask for confirmation before destructive operations
"""
        self.context_window = ContextWindow()
        self.project_info = {}
        self.user_preferences = {}
    
    def build_system_prompt(
        self,
        project_path: str,
        user_prefs: Dict
    ) -> str:
        """构建系统提示"""
        # 动态加载项目信息
        self.project_info = self._load_project_info(project_path)
        self.user_preferences = user_prefs
        
        # 获取当前工作状态
        current_context = self._get_current_context()
        
        return self.system_prompt_template.format(
            project_info=self._format_project_info(),
            user_preferences=self._format_user_prefs(),
            current_context=current_context
        )
    
    def _load_project_info(self, project_path: str) -> Dict:
        """加载项目信息"""
        import os
        
        info = {
            "path": project_path,
            "files": [],
            "structure": {}
        }
        
        # 读取项目结构
        if os.path.exists(project_path):
            for root, dirs, files in os.walk(project_path):
                # 跳过隐藏目录和常见忽略目录
                dirs[:] = [d for d in dirs if not d.startswith('.') and d not in ['node_modules', '__pycache__']]
                
                rel_path = os.path.relpath(root, project_path)
                if rel_path == '.':
                    rel_path = ''
                
                info["files"].extend([
                    os.path.join(rel_path, f) for f in files
                    if not f.startswith('.')
                ][:100])  # 限制文件数量
        
        return info
    
    def _format_project_info(self) -> str:
        """格式化项目信息"""
        lines = [
            f"- Project Path: {self.project_info.get('path', 'Unknown')}",
            f"- File Count: {len(self.project_info.get('files', []))}",
            "- Key Files: " + ", ".join(self.project_info.get('files', [])[:5])
        ]
        return "\n".join(lines)
    
    def _format_user_prefs(self) -> str:
        """格式化用户偏好"""
        if not self.user_preferences:
            return "- No specific preferences set"
        
        return "\n".join([
            f"- {k}: {v}" for k, v in self.user_preferences.items()
        ])
    
    def _get_current_context(self) -> str:
        """获取当前工作上下文"""
        # 例如：当前打开的文件、最近的修改等
        return "- Working on: main.py\n- Last action: Fixed bug in line 42"
    
    def inject_file_content(self, file_path: str, max_lines: int = 50) -> str:
        """注入文件内容到上下文"""
        try:
            with open(file_path, 'r', encoding='utf-8') as f:
                content = f.read()
                lines = content.split('\n')
                
                if len(lines) > max_lines:
                    # 显示开头和结尾，中间省略
                    head = '\n'.join(lines[:max_lines//2])
                    tail = '\n'.join(lines[-max_lines//2:])
                    content = f"{head}\n... ({len(lines) - max_lines} lines omitted) ...\n{tail}"
                
                return f"### File: {file_path}\n```\n{content}\n```"
        except Exception as e:
            return f"### File: {file_path}\nError reading file: {e}"
    
    def prepare_messages(
        self,
        user_input: str,
        system_prompt: str
    ) -> List[Dict]:
        """准备完整的消息列表"""
        # 添加系统提示
        self.context_window.add_message("system", system_prompt)
        
        # 添加用户输入
        self.context_window.add_message("user", user_input)
        
        return self.context_window.messages


# 使用示例
def main():
    harness = ContextManagementHarness()
    
    # 构建系统提示
    system_prompt = harness.build_system_prompt(
        project_path="/path/to/project",
        user_prefs={
            "code_style": "pep8",
            "test_framework": "pytest",
            "language": "Python"
        }
    )
    
    # 准备消息
    messages = harness.prepare_messages(
        user_input="帮我修复 main.py 中的 bug",
        system_prompt=system_prompt
    )
    
    # 添加相关文件内容
    file_content = harness.inject_file_content("main.py")
    messages.append({"role": "user", "content": f"Here's the file content:\n{file_content}"})
    
    print(f"Total messages: {len(messages)}")
    print(f"Estimated tokens: {harness.context_window.estimate_tokens()}")
```

**马鞍的关键设计**：
1. **Token 预算管理**：预留空间，防止超限
2. **智能压缩**：保留关键信息，丢弃冗余内容
3. **项目信息注入**：让 AI 了解工作目录和文件结构
4. **用户偏好记忆**：自动加载用户的编码风格偏好
5. **文件内容注入**：需要时自动读取文件内容

---

### 2.3 跑道护栏：安全沙箱系统（Security Sandbox Harness）

**护栏的作用**：划定边界，防止 Agent 跑出赛道，造成伤害。

#### 问题场景

Agent 可能：
- 删除重要系统文件
- 访问敏感数据（密码、密钥）
- 执行网络请求到恶意地址
- 消耗过多资源（CPU、内存、磁盘）

#### Harness 解决方案

```python
import os
import re
from enum import Enum
from typing import List, Set, Optional
from dataclasses import dataclass

class SandboxViolation(Enum):
    FILE_ACCESS_DENIED = "file_access_denied"
    NETWORK_BLOCKED = "network_blocked"
    RESOURCE_EXCEEDED = "resource_exceeded"
    COMMAND_NOT_ALLOWED = "command_not_allowed"

@dataclass
class SandboxConfig:
    """沙箱配置"""
    # 文件系统限制
    allowed_paths: List[str] = None
    blocked_paths: List[str] = None
    read_only: bool = False
    
    # 网络限制
    allow_network: bool = False
    allowed_hosts: List[str] = None
    blocked_hosts: List[str] = None
    
    # 资源限制
    max_cpu_percent: float = 50.0
    max_memory_mb: int = 512
    max_disk_mb: int = 1024
    max_execution_time_sec: int = 300
    
    # 命令限制
    allowed_commands: List[str] = None
    blocked_commands: List[str] = None


class SecuritySandboxHarness:
    """安全沙箱系统 - 跑道护栏"""
    
    # 危险命令列表
    DANGEROUS_COMMANDS = [
        r"rm\s+-rf\s+/",
        r"dd\s+if=.*of=/dev/",
        r">\s*/dev/",
        r":\(\)\{\s*:\|:\&\s*\};:",  # Fork bomb
    ]
    
    # 敏感文件路径
    SENSITIVE_PATHS = [
        "/etc/passwd",
        "/etc/shadow",
        "~/.ssh/",
        "~/.aws/",
        "~/.config/gh/",
    ]
    
    def __init__(self, config: SandboxConfig = None):
        self.config = config or SandboxConfig()
        self.violations: List[SandboxViolation] = []
    
    def check_file_access(self, path: str, operation: str = "read") -> bool:
        """
        检查文件访问权限
        """
        # 标准化路径
        real_path = os.path.realpath(os.path.expanduser(path))
        
        # 检查是否在敏感路径
        for sensitive in self.SENSITIVE_PATHS:
            sensitive_real = os.path.realpath(os.path.expanduser(sensitive))
            if real_path.startswith(sensitive_real):
                self.violations.append(SandboxViolation.FILE_ACCESS_DENIED)
                return False
        
        # 检查是否在阻止列表
        if self.config.blocked_paths:
            for blocked in self.config.blocked_paths:
                blocked_real = os.path.realpath(os.path.expanduser(blocked))
                if real_path.startswith(blocked_real):
                    self.violations.append(SandboxViolation.FILE_ACCESS_DENIED)
                    return False
        
        # 检查是否在允许列表（如果配置了白名单）
        if self.config.allowed_paths:
            allowed = False
            for allowed_path in self.config.allowed_paths:
                allowed_real = os.path.realpath(os.path.expanduser(allowed_path))
                if real_path.startswith(allowed_real):
                    allowed = True
                    break
            
            if not allowed:
                self.violations.append(SandboxViolation.FILE_ACCESS_DENIED)
                return False
        
        # 检查写权限
        if operation == "write" and self.config.read_only:
            self.violations.append(SandboxViolation.FILE_ACCESS_DENIED)
            return False
        
        return True
    
    def check_command(self, command: str) -> bool:
        """
        检查命令是否允许执行
        """
        command_lower = command.lower()
        
        # 检查危险命令模式
        for pattern in self.DANGEROUS_COMMANDS:
            if re.search(pattern, command_lower):
                self.violations.append(SandboxViolation.COMMAND_NOT_ALLOWED)
                return False
        
        # 检查白名单
        if self.config.allowed_commands:
            cmd = command.split()[0] if command else ""
            if cmd not in self.config.allowed_commands:
                self.violations.append(SandboxViolation.COMMAND_NOT_ALLOWED)
                return False
        
        # 检查黑名单
        if self.config.blocked_commands:
            cmd = command.split()[0] if command else ""
            if cmd in self.config.blocked_commands:
                self.violations.append(SandboxViolation.COMMAND_NOT_ALLOWED)
                return False
        
        return True
    
    def check_network_access(self, url: str) -> bool:
        """
        检查网络访问权限
        """
        if not self.config.allow_network:
            self.violations.append(SandboxViolation.NETWORK_BLOCKED)
            return False
        
        # 解析主机名
        from urllib.parse import urlparse
        parsed = urlparse(url)
        host = parsed.netloc
        
        # 检查黑名单
        if self.config.blocked_hosts:
            for blocked in self.config.blocked_hosts:
                if host == blocked or host.endswith(f".{blocked}"):
                    self.violations.append(SandboxViolation.NETWORK_BLOCKED)
                    return False
        
        # 检查白名单
        if self.config.allowed_hosts:
            allowed = False
            for allowed_host in self.config.allowed_hosts:
                if host == allowed_host or host.endswith(f".{allowed_host}"):
                    allowed = True
                    break
            
            if not allowed:
                self.violations.append(SandboxViolation.NETWORK_BLOCKED)
                return False
        
        return True
    
    def create_resource_monitor(self):
        """
        创建资源监控器（上下文管理器）
        """
        return ResourceMonitor(
            max_cpu=self.config.max_cpu_percent,
            max_memory=self.config.max_memory_mb,
            max_time=self.config.max_execution_time_sec
        )
    
    def get_violations(self) -> List[SandboxViolation]:
        """获取所有违规记录"""
        return self.violations.copy()
    
    def clear_violations(self):
        """清空违规记录"""
        self.violations.clear()


class ResourceMonitor:
    """资源监控器"""
    
    def __init__(self, max_cpu: float, max_memory: int, max_time: int):
        self.max_cpu = max_cpu
        self.max_memory = max_memory
        self.max_time = max_time
        self.start_time = None
        self.process = None
    
    def __enter__(self):
        import time
        self.start_time = time.time()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        pass
    
    def check_resources(self) -> bool:
        """检查资源使用是否超限"""
        import time
        import psutil
        
        # 检查执行时间
        if self.start_time:
            elapsed = time.time() - self.start_time
            if elapsed > self.max_time:
                return False
        
        # 检查内存使用
        memory = psutil.virtual_memory()
        used_mb = memory.used / (1024 * 1024)
        if used_mb > self.max_memory:
            return False
        
        # 检查 CPU 使用
        cpu_percent = psutil.cpu_percent(interval=1)
        if cpu_percent > self.max_cpu:
            return False
        
        return True


# 使用示例
def main():
    # 配置沙箱
    config = SandboxConfig(
        allowed_paths=["/home/user/project", "/tmp"],
        blocked_paths=["/etc", "/usr/bin"],
        read_only=False,
        allow_network=True,
        allowed_hosts=["api.github.com", "pypi.org"],
        max_execution_time_sec=60
    )
    
    sandbox = SecuritySandboxHarness(config)
    
    # 测试文件访问
    print("File access test:")
    print(f"  /home/user/project/main.py: {sandbox.check_file_access('/home/user/project/main.py')}")
    print(f"  /etc/passwd: {sandbox.check_file_access('/etc/passwd')}")
    
    # 测试命令
    print("\nCommand test:")
    print(f"  'ls -la': {sandbox.check_command('ls -la')}")
    print(f"  'rm -rf /': {sandbox.check_command('rm -rf /')}")
    
    # 测试网络
    print("\nNetwork test:")
    print(f"  https://api.github.com: {sandbox.check_network_access('https://api.github.com')}")
    print(f"  https://evil.com: {sandbox.check_network_access('https://evil.com')}")
    
    print(f"\nViolations: {sandbox.get_violations()}")

if __name__ == "__main__":
    main()
```

**护栏的关键防护**：
1. **文件系统沙箱**：白名单/黑名单路径控制
2. **命令过滤**：危险命令模式拦截
3. **网络隔离**：主机级访问控制
4. **资源限制**：CPU、内存、时间上限
5. **敏感路径保护**：禁止访问密码、密钥等

---

### 2.4 反馈镜子：监控与可观测性系统（Observability Harness）

**镜子的作用**：让开发者实时看到 Agent 的状态，及时发现和解决问题。

#### 问题场景

Agent 运行时：
- 不知道它在做什么、做到哪一步
- 出错了不知道错在哪里
- Token 用完了才发现
- 性能问题难以定位

#### Harness 解决方案

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import List, Dict, Optional, Callable
from enum import Enum
import json

class LogLevel(Enum):
    DEBUG = "debug"
    INFO = "info"
    WARNING = "warning"
    ERROR = "error"

class MetricType(Enum):
    TOKEN_USAGE = "token_usage"
    TOOL_CALLS = "tool_calls"
    LATENCY = "latency"
    ERROR_RATE = "error_rate"
    COST = "cost"

@dataclass
class LogEntry:
    """日志条目"""
    timestamp: datetime
    level: LogLevel
    component: str
    message: str
    context: Dict = field(default_factory=dict)

@dataclass
class MetricPoint:
    """指标数据点"""
    timestamp: datetime
    metric_type: MetricType
    value: float
    labels: Dict = field(default_factory=dict)


class ObservabilityHarness:
    """可观测性系统 - 反馈镜子"""
    
    def __init__(self, session_id: str):
        self.session_id = session_id
        self.logs: List[LogEntry] = []
        self.metrics: List[MetricPoint] = []
        self.callbacks: Dict[str, List[Callable]] = {
            "on_log": [],
            "on_metric": [],
            "on_error": []
        }
        
        # 统计信息
        self.stats = {
            "total_tokens": 0,
            "total_tool_calls": 0,
            "total_cost": 0.0,
            "errors": 0,
            "start_time": datetime.now()
        }
    
    def add_callback(self, event: str, callback: Callable):
        """添加事件回调"""
        if event in self.callbacks:
            self.callbacks[event].append(callback)
    
    def log(
        self,
        level: LogLevel,
        component: str,
        message: str,
        context: Dict = None
    ):
        """记录日志"""
        entry = LogEntry(
            timestamp=datetime.now(),
            level=level,
            component=component,
            message=message,
            context=context or {}
        )
        
        self.logs.append(entry)
        
        # 触发回调
        for callback in self.callbacks["on_log"]:
            try:
                callback(entry)
            except Exception:
                pass
        
        # 实时打印（开发时）
        self._print_log(entry)
    
    def _print_log(self, entry: LogEntry):
        """打印日志到控制台"""
        emoji = {
            LogLevel.DEBUG: "🔍",
            LogLevel.INFO: "ℹ️",
            LogLevel.WARNING: "⚠️",
            LogLevel.ERROR: "❌"
        }
        
        time_str = entry.timestamp.strftime("%H:%M:%S")
        print(f"{emoji.get(entry.level, '•')} [{time_str}] [{entry.component}] {entry.message}")
    
    def record_metric(
        self,
        metric_type: MetricType,
        value: float,
        labels: Dict = None
    ):
        """记录指标"""
        point = MetricPoint(
            timestamp=datetime.now(),
            metric_type=metric_type,
            value=value,
            labels=labels or {}
        )
        
        self.metrics.append(point)
        
        # 更新统计
        if metric_type == MetricType.TOKEN_USAGE:
            self.stats["total_tokens"] += int(value)
        elif metric_type == MetricType.TOOL_CALLS:
            self.stats["total_tool_calls"] += int(value)
        elif metric_type == MetricType.COST:
            self.stats["total_cost"] += value
        
        # 触发回调
        for callback in self.callbacks["on_metric"]:
            try:
                callback(point)
            except Exception:
                passn        
        # 检查告警
        self._check_alerts(point)
    
    def _check_alerts(self, point: MetricPoint):
        """检查是否需要告警"""
        if point.metric_type == MetricType.TOKEN_USAGE:
            if point.value > 10000:
                self.log(
                    LogLevel.WARNING,
                    "Observability",
                    f"High token usage: {point.value}",
                    {"threshold": 10000}
                )
        
        elif point.metric_type == MetricType.COST:
            if self.stats["total_cost"] > 5.0:  # $5
                self.log(
                    LogLevel.WARNING,
                    "Observability",
                    f"Cost limit approaching: ${self.stats['total_cost']:.2f}",
                    {"limit": 5.0}
                )
    
    def record_error(self, error: Exception, context: Dict = None):
        """记录错误"""
        self.stats["errors"] += 1
        
        self.log(
            LogLevel.ERROR,
            "ErrorHandler",
            str(error),
            {
                "error_type": type(error).__name__,
                "context": context or {}
            }
        )
        
        # 触发错误回调
        for callback in self.callbacks["on_error"]:
            try:
                callback(error, context)
            except Exception:
                pass
    
    def get_summary(self) -> Dict:
        """获取会话摘要"""
        duration = (datetime.now() - self.stats["start_time"]).total_seconds()
        
        return {
            "session_id": self.session_id,
            "duration_seconds": duration,
            "total_tokens": self.stats["total_tokens"],
            "total_tool_calls": self.stats["total_tool_calls"],
            "total_cost_usd": round(self.stats["total_cost"], 4),
            "errors": self.stats["errors"],
            "logs_count": len(self.logs),
            "metrics_count": len(self.metrics)
        }
    
    def export_logs(self, file_path: str):
        """导出日志到文件"""
        with open(file_path, 'w', encoding='utf-8') as f:
            for entry in self.logs:
                f.write(json.dumps({
                    "timestamp": entry.timestamp.isoformat(),
                    "level": entry.level.value,
                    "component": entry.component,
                    "message": entry.message,
                    "context": entry.context
                }, ensure_ascii=False) + "\n")
    
    def export_metrics(self, file_path: str):
        """导出指标到文件"""
        with open(file_path, 'w', encoding='utf-8') as f:
            for point in self.metrics:
                f.write(json.dumps({
                    "timestamp": point.timestamp.isoformat(),
                    "type": point.metric_type.value,
                    "value": point.value,
                    "labels": point.labels
                }, ensure_ascii=False) + "\n")


# 使用示例
async def main():
    obs = ObservabilityHarness(session_id="session_001")
    
    # 添加实时回调示例
    def on_error_callback(error, context):
        print(f"🚨 ALERT: Error occurred - {error}")
    
    obs.add_callback("on_error", on_error_callback)
    
    # 模拟工作流
    obs.log(LogLevel.INFO, "Workflow", "Starting task processing")
    
    # 记录 Token 使用
    obs.record_metric(MetricType.TOKEN_USAGE, 1500, {"model": "gpt-4"})
    
    # 记录工具调用
    obs.record_metric(MetricType.TOOL_CALLS, 1, {"tool": "file_read"})
    obs.log(LogLevel.INFO, "Tool", "Reading file: main.py")
    
    # 模拟错误
    try:
        raise ValueError("File not found")
    except Exception as e:
        obs.record_error(e, {"file": "main.py"})
    
    # 记录成本
    obs.record_metric(MetricType.COST, 0.045, {"model": "gpt-4"})
    
    # 打印摘要
    print("\n" + "="*50)
    print("Session Summary:")
    print(json.dumps(obs.get_summary(), indent=2))

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

**镜子的关键功能**：
1. **结构化日志**：组件化、带上下文的日志记录
2. **指标采集**：Token、成本、延迟、错误率
3. **实时告警**：超限自动提醒
4. **回调机制**：支持外部系统集成
5. **会话摘要**：一键导出完整报告

---

## 三、整合：构建完整的 Harness 系统

### 3.1 统一入口

```python
class AgentHarness:
    """
    AI Agent 的完整 Harness 系统
    整合：缰绳（工具控制）+ 马鞍（上下文）+ 护栏（安全）+ 镜子（监控）
    """
    
    def __init__(self, config: dict = None):
        self.config = config or {}
        
        # 初始化四大组件
        self.tool_harness = ToolCallingHarness()
        self.context_harness = ContextManagementHarness()
        self.security_harness = SecuritySandboxHarness()
        self.observability_harness = ObservabilityHarness(
            session_id=self._generate_session_id()
        )
        
        # 注册审批回调
        self.tool_harness.approval_callback = self._request_approval
        
        self.observability_harness.log(
            LogLevel.INFO,
            "AgentHarness",
            "Harness system initialized"
        )
    
    async def execute(self, user_input: str, project_path: str = ".") -> str:
        """
        执行用户请求（完整流程）
        """
        obs = self.observability_harness
        
        try:
            # 1. 构建上下文
            obs.log(LogLevel.INFO, "Context", "Building system prompt")
            system_prompt = self.context_harness.build_system_prompt(
                project_path=project_path,
                user_prefs=self.config.get("user_prefs", {})
            )
            
            # 2. 准备消息
            messages = self.context_harness.prepare_messages(
                user_input=user_input,
                system_prompt=system_prompt
            )
            
            # 3. 调用 LLM（模拟）
            obs.log(LogLevel.INFO, "LLM", "Sending request to model")
            llm_response = await self._call_llm(messages)
            
            obs.record_metric(
                MetricType.TOKEN_USAGE,
                llm_response.get("tokens_used", 0)
            )
            
            # 4. 处理工具调用
            if llm_response.get("tool_calls"):
                obs.log(
                    LogLevel.INFO,
                    "Tool",
                    f"Processing {len(llm_response['tool_calls'])} tool calls"
                )
                
                for tool_call in llm_response["tool_calls"]:
                    # 安全检查
                    if not self._security_check(tool_call):
                        continue
                    
                    # 执行工具
                    result = await self.tool_harness.execute_tool_call(tool_call)
                    
                    # 记录结果
                    if result.success:
                        obs.log(LogLevel.INFO, "Tool", f"Success: {tool_call.tool_name}")
                    else:
                        obs.record_error(
                            Exception(result.error),
                            {"tool": tool_call.tool_name}
                        )
            
            # 5. 生成最终响应
            final_response = llm_response.get("content", "Task completed")
            
            obs.log(LogLevel.INFO, "AgentHarness", "Task completed successfully")
            
            return final_response
            
        except Exception as e:
            obs.record_error(e)
            raise
    
    def _security_check(self, tool_call) -> bool:
        """安全检查"""
        # 文件操作检查
        if "path" in tool_call.parameters:
            if not self.security_harness.check_file_access(
                tool_call.parameters["path"]
            ):
                return False
        
        # 命令检查
        if "command" in tool_call.parameters:
            if not self.security_harness.check_command(
                tool_call.parameters["command"]
            ):
                return False
        
        return True
    
    async def _request_approval(self, call) -> bool:
        """请求用户审批"""
        self.observability_harness.log(
            LogLevel.WARNING,
            "Approval",
            f"Approval needed for: {call.tool_name}"
        )
        
        # 实际实现中弹出 UI 或发送消息
        # 这里简化处理
        return True
    
    def get_report(self) -> dict:
        """获取完整报告"""
        return {
            "observability": self.observability_harness.get_summary(),
            "security_violations": self.security_harness.get_violations(),
            "tool_stats": {
                "total_calls": len(self.tool_harness.call_history)
            }
        }
    
    def _generate_session_id(self) -> str:
        import uuid
        return str(uuid.uuid4())[:8]
    
    async def _call_llm(self, messages: list) -> dict:
        """调用 LLM（模拟）"""
        # 实际实现中调用 OpenAI/Anthropic API
        return {
            "content": "I've analyzed your request.",
            "tool_calls": [],
            "tokens_used": 1500
        }
```

---

## 四、从 0 到 1：实战 Checklist

### Week 1: 基础 Harness
- [ ] 搭建项目结构
- [ ] 实现 ToolCallingHarness（3个基础工具）
- [ ] 添加简单的审批机制
- [ ] 写单元测试

### Week 2: 上下文管理
- [ ] 实现 ContextManagementHarness
- [ ] 添加对话历史压缩
- [ ] 集成项目信息注入
- [ ] 支持文件内容注入

### Week 3: 安全防护
- [ ] 实现 SecuritySandboxHarness
- [ ] 配置危险命令黑名单
- [ ] 添加文件路径白名单
- [ ] 集成资源监控

### Week 4: 可观测性
- [ ] 实现 ObservabilityHarness
- [ ] 添加结构化日志
- [ ] 集成指标采集
- [ ] 设置告警阈值

### Week 5: 生产化
- [ ] 整合所有组件
- [ ] 完善错误处理
- [ ] 性能优化
- [ ] 编写文档

---

## 五、关键设计原则

### 5.1 渐进式启用

不要一次性启用所有安全限制，而是根据场景逐步加强：

```python
# 开发环境
config = {
    "approval_mode": "off",  # 关闭审批，快速迭代
    "sandbox_level": "low"   # 宽松沙箱
}

# 测试环境
config = {
    "approval_mode": "smart",  # 智能审批
    "sandbox_level": "medium"
}

# 生产环境
config = {
    "approval_mode": "manual",  # 全部审批
    "sandbox_level": "high"     # 严格沙箱
}
```

### 5.2 防御性编程

宁可误报，不可漏报：

```python
# 好的做法：保守的审批
if risk_level in [MEDIUM, HIGH, CRITICAL]:
    require_approval()

# 不好的做法：过于宽松
if risk_level == CRITICAL:
    require_approval()
```

### 5.3 可观测性优先

所有关键操作都要留下痕迹：

```python
# 每个工具调用都要记录
observability.log(INFO, "Tool", f"Executing {tool_name}")
observability.record_metric(TOOL_CALLS, 1)

# 每个安全决策都要记录
observability.log(INFO, "Security", f"Access {path}: {allowed}")
```

---

## 结语

Harness Engineering 的核心不是让 AI 更聪明，而是让 AI **更可靠、更可控、更可预测**。

就像驯马师不会试图改变马的本性，而是通过缰绳、马鞍、护栏和镜子，让烈马能够安全、高效地完成任务。

**人类掌舵，AI 执行。** 这就是 Harness Engineering 的终极哲学。

---

## 参考资源

- [Anthropic Claude Code 官方文档](https://docs.anthropic.com/claude-code)
- [OpenClaw ACP 协议](https://docs.openclaw.ai/acp)
- [Nous Research Hermes Agent](https://hermes-agent.nousresearch.com)

*本文代码基于 MIT 协议，可自由使用*
