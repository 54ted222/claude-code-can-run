# 01 - 專案總覽

## 這是什麼？

這是 Anthropic 官方 **Claude Code CLI** 工具的反編譯（decompiled）版本。Claude Code 是一個終端機 AI 助手，讓開發者可以在命令列中與 Claude 互動，執行程式碼編輯、檔案操作、Git 操作等任務。

## 技術棧

| 項目 | 技術 |
|------|------|
| **Runtime** | Bun（非 Node.js） |
| **語言** | TypeScript + TSX |
| **UI 框架** | React + Ink（終端 UI） |
| **CLI 框架** | Commander.js |
| **API SDK** | @anthropic-ai/sdk |
| **模組系統** | ESM（`"type": "module"`） |
| **構建** | Bun 單檔打包 |
| **Monorepo** | Bun workspaces（`packages/`） |

## 目錄結構總覽

```
claude-code-can-run/
├── src/                    # 主要原始碼
│   ├── entrypoints/        # 程式進入點
│   ├── main.tsx            # CLI 定義（Commander.js）
│   ├── query.ts            # 核心 API 查詢函數
│   ├── QueryEngine.ts      # 查詢引擎（對話管理）
│   ├── context.ts          # 系統提示詞建構
│   ├── tools.ts            # 工具註冊表
│   ├── Tool.ts             # 工具介面定義
│   ├── tools/              # 所有工具實作（50+）
│   ├── screens/            # 畫面（REPL、Doctor）
│   ├── components/         # React/Ink UI 元件（100+）
│   ├── ink/                # 自訂 Ink 框架（forked）
│   ├── ink.ts              # Ink 渲染包裝器
│   ├── state/              # 狀態管理（Zustand-style）
│   ├── bootstrap/          # 啟動時全域狀態
│   ├── services/           # 服務層（API、MCP、分析）
│   ├── hooks/              # React hooks
│   ├── utils/              # 工具函數（300+）
│   ├── types/              # TypeScript 型別定義
│   ├── commands/           # CLI 子命令
│   ├── skills/             # 技能系統
│   └── ...                 # 其他模組
├── packages/               # Monorepo 內部套件
│   ├── @ant/               # Computer Use 存根
│   ├── color-diff-napi/    # 顏色差異（完整實作）
│   └── *-napi/             # 原生模組存根
├── package.json
├── tsconfig.json
├── bunfig.toml
└── CLAUDE.md               # AI 開發指引
```

## 架構圖

```mermaid
graph TB
    subgraph Entry["進入點"]
        CLI["cli.tsx<br/>Polyfills + 啟動"]
        MAIN["main.tsx<br/>Commander.js CLI 定義"]
        INIT["init.ts<br/>一次性初始化"]
    end

    subgraph Core["核心引擎"]
        QE["QueryEngine.ts<br/>對話管理器"]
        Q["query.ts<br/>API 查詢函數"]
        CTX["context.ts<br/>系統提示詞建構"]
    end

    subgraph UI["使用者介面 (React/Ink)"]
        REPL["REPL.tsx<br/>互動式 REPL 畫面"]
        COMP["components/<br/>100+ UI 元件"]
        INK["ink/<br/>自訂 Ink 框架"]
    end

    subgraph Tools["工具系統"]
        TR["tools.ts<br/>工具註冊表"]
        TI["Tool.ts<br/>工具介面"]
        TD["tools/*<br/>50+ 工具實作"]
    end

    subgraph Services["服務層"]
        API["services/api/<br/>Claude API 客戶端"]
        MCP["services/mcp/<br/>MCP 協議"]
        PROV["utils/model/providers.ts<br/>多供應商支援"]
    end

    subgraph State["狀態管理"]
        AS["state/AppState.tsx<br/>應用狀態"]
        ST["state/store.ts<br/>Zustand Store"]
        BS["bootstrap/state.ts<br/>全域單例"]
    end

    CLI --> MAIN --> INIT
    MAIN --> REPL
    REPL --> QE --> Q --> API
    Q --> CTX
    Q --> TR --> TD
    API --> PROV
    REPL --> COMP --> INK
    REPL --> AS --> ST
    QE --> BS
```

## 啟動流程

```mermaid
sequenceDiagram
    participant User as 使用者
    participant CLI as cli.tsx
    participant Main as main.tsx
    participant Init as init.ts
    participant REPL as REPL.tsx
    participant QE as QueryEngine
    participant API as Claude API

    User->>CLI: bun run cli.tsx
    CLI->>CLI: 注入 polyfills<br/>(feature(), MACRO)
    CLI->>Main: import main
    Main->>Main: Commander.js 解析參數
    Main->>Init: 初始化服務
    Init->>Init: 設定 telemetry、config
    
    alt 互動模式
        Main->>REPL: 啟動 REPL 畫面
        REPL->>User: 顯示提示符
        User->>REPL: 輸入提示詞
        REPL->>QE: 送出查詢
    else Pipe 模式 (-p)
        Main->>QE: 直接送出 stdin
    end
    
    QE->>API: 串流 API 請求
    API-->>QE: 串流回應 + 工具呼叫
    QE->>QE: 執行工具、管理對話
    QE-->>REPL: 顯示結果
```

## 重要注意事項

1. **~1341 個 tsc 錯誤** — 來自反編譯，不影響 Bun 運行時執行
2. **`feature()` 永遠回傳 `false`** — 所有功能旗標後的程式碼都是死碼
3. **React Compiler 輸出** — 元件中有 `_c()` 記憶化樣板，這是正常的
4. **多個模組已被刪除/存根化** — Voice、MagicDocs、Plugins 等功能已移除
