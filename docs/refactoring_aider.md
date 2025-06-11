

### **1\. 总体架构**

划分为两个主要部分：

**A. IDEA 插件 (客户端)**

* **角色**：一个功能丰富的、**有状态的**客户端。它负责所有与用户本地环境直接交互的任务。  
* **技术栈**：Kotlin (用于开发 Android Studio 插件)。  
* **核心职责**:  
  1. 提供用户界面。  
  2. 与本地文件系统交互。  
  3. 执行本地代码分析（生成RepoMap）。  
  4. 管理会话状态（聊天历史、已添加文件等）。  
  5. 作为 API 客户端，与云端服务通信。

**B. 云端 LLM 服务 (服务端)**

* **角色**：一个**无状态的**、专注的后端服务。它唯一的职责就是处理与大型语言模型（LLM）相关的所有逻辑。  
* **核心职责**:  
  1. 接收来自 Android Studio 插件的请求。   
  2. 根据请求内容，格式化 Prompt (继承 aider/coders/\*\_prompts.py 的逻辑)。  
  3. 调用 LLM API (litellm 的逻辑)。  
  4. 将 LLM 的原始响应（流式或非流式）返回给插件。

### ---

**2\. Android Studio 插件 (客户端) 功能与实现**

这是重构中工作量最大、最复杂的部分。

**a. 本地环境交互 (Local Environment Interaction)**

* **文件系统**：利用 IntelliJ Platform 的虚拟文件系统（Virtual File System, VFS）API 来读取、写入和监听文件变化。这比直接用 java.io.File 更高效、更安全，并且能与 IDE 的其他功能（如撤销、本地历史）更好地集成。

**b. Repo Map 功能实现**

* **tree-sitter 集成**：使用bonede/tree-sitter-ng来解析代码。  
* **图分析**: 使用JGraphT来重新实现 PageRank 算法和符号关系图的构建。

**c. 状态管理 (State Management)**

* 插件将是**有状态**的。它需要保存当前会话的所有信息。  
* **聊天上下文**: 包括聊天历史记录 (done\_messages, cur\_messages from base\_coder.py)。  
* **文件上下文**: 用户通过 UI 添加到会话中的文件列表 (abs\_fnames)。  
* 这些状态可以利用 IntelliJ 的 Project 级别的 [Service](https://plugins.jetbrains.com/docs/intellij/plugin-services.html) 来存储，这样可以为每个项目（Project Window）维护一个独立的会话。 

### ---

**3\. 云端服务 (服务端) 职责与实现建议**

这部分相对简单，核心是**无状态化**。

**a. 无状态 API 设计 (Stateless API Design)**

* “无状态”意味着云端服务**不保存任何两次请求之间的会话信息**。每一次从 IDE 插件发起的 API 请求都必须包含**完整、独立**的所有上下文。  
* API 端点可以很简单，例如 POST /api/v1/chat/completions。  
* 请求的 Body (JSON) 必须包含：  
  JSON  
  {  
    "model": "gpt-4o", // 用户选择的模型  
    "messages": \[ // 完整的消息历史，由插件端组装  
      {"role": "system", "content": "..."}, // 经过插件格式化的系统提示  
      {"role": "user", "content": "..."}, // Repo Map, 文件内容等  
      {"role": "assistant", "content": "..."},  
      {"role": "user", "content": "..."} // 当前用户的请求  
    \],  
    "stream": true, // 是否流式返回  
    // 其他模型参数,如 temperature  
  }

* 这意味着插件端需要负责实现 aider/coders/base\_coder.py 中的 format\_chat\_chunks 或类似逻辑，将系统提示、Repo Map、文件内容、聊天历史等组装成一个完整的 messages 列表。

**b. LLM 交互逻辑的迁移**

* 将 aider/llm.py, aider/models.py 的核心逻辑迁移到云端。  
* 云端服务使用 litellm 来实际调用各种 LLM API。  
* 所有 aider/coders/\*\_prompts.py 中的 Prompt 模板和逻辑也应在云端，由云端服务在调用 LLM 前最后一刻进行格式化。

### ---

**4\. 需要注意的问题**

1. **状态同步与一致性**  
   * 用户可能在 IDE 的编辑器中直接修改了文件，而不是通过插件。插件必须能感知到这些变化。  
2. **Coder 逻辑的拆分**  
   * 当前 Aider 的 base_coder类是一个巨大的、有状态的类，混合了多种职责（状态管理、Prompt 格式化、响应解析、文件应用）。
3. **Git/Shell/Linter 等耗时操作**

### ---

**5\. base_coder.py拆分**
base_coder 中的功能/方法
| 功能/方法 | 最终归属 | 理由 |
|----------|----------|------|
| 状态管理 (abs_fnames, cur_messages 等) |  插件 | 会话状态必须与用户的本地项目绑定，是有状态的。 |
| 上下文组装 (get_files_content, get_repo_map) |  插件 | 需要访问本地文件系统和执行本地代码分析。 |
| Prompt 模板 (gpt_prompts 及其内容) | 云端服务 | 便于集中管理和快速迭代 Prompt，无需更新插件。 |
| 注入系统提示 (部分 format_chat_chunks) | 云端服务 | 云端负责最后的 Prompt 组装，确保 LLM 接收到正确的指令。 |
| 发送请求与重试 (send_message, send) | 云端服务 | 封装了 LLM API 调用、网络重试逻辑。 |
| 解析响应 (get_edits) | IDE 插件 | 为了实现流式 UI 体验，并使云端服务与具体编辑格式解耦。 |
| 应用编辑 (apply_updates, apply_edits) |  插件 | 直接操作本地文件，必须在客户端执行。 |
| Git/Shell/Linter 操作 (auto_commit, run_shell_commands) |  插件 | 与本地环境和工具链交互，必须在客户端执行。 |

