# 07 - 狀態管理

## 架構概覽

Claude Code 有兩種狀態管理機制：**React 狀態**（Zustand-style store）和**模組層級單例**。

```mermaid
graph TB
    subgraph ReactState["React 狀態 (src/state/)"]
        AS["AppState.tsx<br/>狀態型別 + Context"]
        STORE["store.ts<br/>Zustand Store"]
        SEL["selectors.ts<br/>狀態選擇器"]
        CHANGE["onChangeAppState.ts<br/>狀態變更監聽"]
    end

    subgraph GlobalState["全域單例 (src/bootstrap/)"]
        BS["state.ts<br/>Session 全域狀態"]
    end

    subgraph Consumers["消費者"]
        REPL["REPL.tsx"]
        COMP["components/*"]
        QE["QueryEngine.ts"]
    end

    AS --> STORE
    STORE --> SEL
    STORE --> CHANGE
    REPL --> AS
    COMP --> SEL
    QE --> BS
```

## `src/state/AppState.tsx` — 應用狀態

定義中央應用狀態的型別和 React Context Provider：

```mermaid
classDiagram
    class AppState {
        messages: Message[]
        tools: Tool[]
        permissions: PermissionState
        mcpConnections: MCPConnection[]
        model: string
        isLoading: boolean
        sessionId: string
        tokenCount: TokenCount
        costTracker: CostTracker
        ...
    }
```

### 主要狀態欄位

| 欄位 | 說明 |
|------|------|
| `messages` | 完整對話訊息歷史 |
| `tools` | 當前可用工具列表 |
| `permissions` | 權限狀態（已批准的工具等） |
| `mcpConnections` | MCP 伺服器連線 |
| `model` | 當前使用的模型 |
| `isLoading` | 是否正在等待 API 回應 |
| `sessionId` | 當前 session ID |
| `tokenCount` | Token 使用統計 |

## `src/state/store.ts` — Zustand-style Store

使用類似 Zustand 的 store 模式管理狀態：

```mermaid
graph LR
    ACTION["狀態更新動作"] --> STORE["store.ts"]
    STORE --> NOTIFY["通知訂閱者"]
    NOTIFY --> UI["UI 重新渲染"]
    NOTIFY --> SIDE["副作用處理"]
```

## `src/state/selectors.ts` — 狀態選擇器

提供從狀態中衍生資料的選擇器函數，避免不必要的重新渲染。

## `src/state/onChangeAppState.ts` — 狀態變更監聽

監聽特定狀態變更並觸發副作用。

## `src/bootstrap/state.ts` — 模組層級單例

用於不需要 React 響應性的全域狀態：

```typescript
// 這些是簡單的模組層級變數
export let sessionId: string;       // 當前 session UUID
export let cwd: string;             // 當前工作目錄
export let projectRoot: string;     // 專案根目錄
export let tokenCounts: {...};      // Token 使用統計
```

### 為什麼有兩種狀態？

| | React Store | 模組單例 |
|--|-------------|----------|
| **用途** | UI 渲染相關 | 非 UI 全域資料 |
| **響應性** | 自動觸發重新渲染 | 無響應性 |
| **存取方式** | React Context / hooks | 直接 import |
| **使用場景** | 元件、畫面 | QueryEngine、API 層 |

## 訊息型別系統（`src/types/message.ts`）

```mermaid
classDiagram
    class Message {
        <<abstract>>
        type: string
        timestamp: number
    }
    
    class UserMessage {
        type: "user"
        content: string | ContentBlock[]
    }
    
    class AssistantMessage {
        type: "assistant"
        content: ContentBlock[]
        model: string
        stopReason: string
    }
    
    class SystemMessage {
        type: "system"
        content: string
    }
    
    class ToolUseBlock {
        type: "tool_use"
        name: string
        input: object
    }
    
    class ToolResultBlock {
        type: "tool_result"
        tool_use_id: string
        content: string
    }
    
    Message <|-- UserMessage
    Message <|-- AssistantMessage
    Message <|-- SystemMessage
    AssistantMessage --> ToolUseBlock
    UserMessage --> ToolResultBlock
```

## 狀態資料流

```mermaid
sequenceDiagram
    participant User as 使用者
    participant UI as React UI
    participant Store as AppState Store
    participant QE as QueryEngine
    participant BS as bootstrap/state
    participant API as Claude API

    User->>UI: 輸入訊息
    UI->>Store: 新增 UserMessage
    Store->>UI: 觸發重新渲染
    UI->>QE: 送出查詢
    QE->>BS: 更新 token 計數
    QE->>API: 串流請求
    
    loop 串流回應
        API-->>QE: 回應片段
        QE->>Store: 更新 AssistantMessage
        Store->>UI: 即時渲染
    end
    
    QE->>Store: 最終狀態更新
    Store->>UI: 完成渲染
```

## 權限型別（`src/types/permissions.ts`）

```mermaid
graph TD
    PM["PermissionMode"]
    PM --> DEFAULT["default<br/>每次詢問"]
    PM --> EDIT["acceptEdits<br/>自動接受編輯"]
    PM --> DONT["dontAsk<br/>不詢問"]
    PM --> BYPASS["bypassPermissions<br/>跳過全部"]
    PM --> PLAN["plan<br/>只讀計畫"]
    
    PR["PermissionResult"]
    PR --> ALLOW["allow<br/>允許"]
    PR --> DENY["deny<br/>拒絕"]
    PR --> ALWAYS["always<br/>永久允許"]
```
