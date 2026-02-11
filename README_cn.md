# vim-floaterm-enhance

[English](README.md)

基于 [vim-floaterm](https://github.com/voldikss/vim-floaterm) 的增强插件。在 Vim 里直接跟 AI CLI 工具（claude、opencode 等）交互，把代码丢给 REPL 执行，或者用 AsyncRun 在浮动终端里跑命令。

---

# 安装

**前提**：需要 Vim 8+（要有 `:terminal`）或 Neovim 0.8+，并先装好 [vim-floaterm](https://github.com/voldikss/vim-floaterm) 和 [fzf.vim](https://github.com/junegunn/fzf.vim)。

**vim-plug**

```vim
Plug 'voldikss/vim-floaterm'
Plug 'leoatchina/vim-floaterm-enhance'
```

**lazy.nvim**

```lua
{
  'leoatchina/vim-floaterm-enhance',
  dependencies = { 'voldikss/vim-floaterm' },
}
```


---

# 配置

## AI 配置

在 vimrc 里设置 `g:floaterm_ai_programs`：

```vim
let g:floaterm_ai_programs = [
    \ ["claude", "--wintype=vsplit --position=left --width=0.3"],
    \ ["opencode", "--wintype=float --position=topright --width=0.45 --height=0.8", "AI"],
  \ ]
" 格式: [命令, floaterm 窗口参数, 标识(可选)]
" 窗口参数就是 floaterm 支持的那些: --wintype, --position, --width, --height 等
" 第三个参数可以不写，默认是 "AI"
```

## REPL 配置

REPL 的配置跟 AI 不一样。插件启动时已经内置了常见语言的 REPL 程序（Python 用 ipython，R 用 radian 等等），所以大部分情况下你不需要额外配置。

如果你想加自己的 REPL 程序，推荐用 `floaterm#repl#update_program()`，**不要直接改 `g:floaterm_repl_programs`**。这个函数会自动检查程序存不存在、去重、按优先级排序：

```vim
" 在 vimrc 里加
call floaterm#repl#update_program('python', ['ipython --no-autoindent', 'python3'])
call floaterm#repl#update_program('r', ['radian', 'R'])
call floaterm#repl#update_program('javascript', ['node'])

" 也可以带 floaterm 窗口参数
call floaterm#repl#update_program('julia', ['julia'], '--wintype=vsplit')
```

插件内置的默认 REPL 列表见 [plugin/floaterm-repl.vim](plugin/floaterm-repl.vim)。

---

# AI 集成

在 Vim 里写代码的时候，可以直接把文件、代码片段、目录路径丢给 claude、opencode 这些 AI CLI 工具，不用切窗口。

## 启动流程

```mermaid
graph TB
    Start[FloatermAiStart] --> Check{已配置?}
    Check -->|否| Error[显示错误]
    Check -->|是| Select{模式}
    Select -->|自动| RunFirst[运行首个程序]
    Select -->|交互| FZF[显示 FZF 菜单]
    FZF --> Run[运行选中程序]
    Run --> SetBuf[设置 g:floaterm_ai_lst]
```

## 发送上下文流程

```mermaid
graph TB
    Send[FloatermAiSend] --> GetBuf{获取 AI 缓冲区}
    GetBuf -->|无| Err2[显示错误]
    GetBuf -->|找到| Format[格式化路径]
    Format --> Type{类型}
    Type -->|File| SendFile[发送文件路径]
    Type -->|Line| SendRange[发送行范围]
    Type -->|Dir| SendDir[发送目录路径]
    Type -->|Selection| SendSel[发送选区]
    SendFile --> Term[发送到终端]
    SendRange --> Term
    SendDir --> Term
    SendSel --> Term
    Term --> Stay{Bang?}
    Stay -->|是| StayEd[留在编辑器]
    Stay -->|否| JumpTerm[跳转到终端]
```

## AI 命令

| 模式 | 命令 | 说明 |
| :--- | :--- | :--- |
| **启动** |
| n | `:FloatermAiStart[!]` | 启动 AI 程序。不带 `!` 弹出选择菜单，带 `!` 直接启动第一个 |
| n | `:FloatermAiSendCr` | 给 AI 终端发个回车 |
| **发送上下文** |
| n/v | `:FloatermAiSendLine[!]` | 发送当前行或选区。带 `!` 留在编辑器 |
| n | `:FloatermAiSendFile[!]` | 发送当前文件路径。带 `!` 留在编辑器 |
| n | `:FloatermAiSendDir[!]` | 发送当前目录路径。带 `!` 留在编辑器 |
| n | `:FloatermAiFzfFiles[!]` | 用 FZF 选文件发送。带 `!` 留在编辑器 |

> 所有 Send 命令：不带 `!` 会跳转到 AI 终端，带 `!` 留在当前编辑器。`n` = normal 模式，`v` = visual 模式。

---

# REPL 集成

把编辑器里的代码直接发到 ipython、R、node 这些 REPL 里执行，支持逐行、代码块、整个文件等多种方式。

## 启动流程

```mermaid
graph TB
    Start[FloatermReplStart] --> Check{配置了程序?}
    Check -->|否| Error[显示错误]
    Check -->|是| SelectMode{自动还是手选?}
    SelectMode -->|自动| GetFirst[取第一个程序]
    SelectMode -->|交互| ShowMenu[显示 FZF 菜单]
    GetFirst --> CheckRunning{已经跑着了?}
    ShowMenu --> UserSelect[用户选择]
    UserSelect --> CheckRunning
    CheckRunning -->|是| OpenExist[打开已有终端]
    CheckRunning -->|否| CreateNew[创建新终端]
    CreateNew --> SetName[设置终端名称]
    SetName --> StoreMap[存到 g:floaterm_repl_dict]
    OpenExist --> Return[回到编辑器]
    StoreMap --> Return
```

## 代码发送流程

```mermaid
graph TB
    Send[FloatermReplSend] --> GetRange[获取代码范围]
    GetRange --> GetFT[获取文件类型]
    GetFT --> FindTerm{找 REPL 终端}
    FindTerm -->|没找到| Prompt[提示启动 REPL]
    FindTerm -->|找到| GetLines[获取代码行]
    GetLines --> Filter[过滤注释和空行]
    Filter --> SendData[发送到终端]
    SendData --> KeepCheck{keep 参数}
    KeepCheck -->|keep=0| MoveNext[移到下一行]
    KeepCheck -->|keep=1| Stay[留在原位]
```

## REPL 命令

| 模式 | 命令 | 说明 |
| :--- | :--- | :--- |
| **启动** |
| n | `:FloatermReplStart[!]` | 启动 REPL。不带 `!` 弹出选择菜单，带 `!` 直接启动第一个 |
| n | `:FloatermReplSendCrOrStart[!]` | 发个回车；如果 REPL 没启动就先启动。带 `!` 留在编辑器 |
| n | `:FloatermReplSendExit` | 给 REPL 发退出命令 |
| n | `:FloatermReplSendClear` | 给 REPL 发清屏命令 |
| **发送代码** |
| n/v | `:FloatermReplSend[!]` | 发送当前行或选区。不带 `!` 光标移到下一行，带 `!` 留在原位 |
| n/v | `:FloatermReplSendBlock[!]` | 发送代码块（用 `%%` 标记分隔）。不带 `!` 移到下一行 |
| n | `:FloatermReplSendToEnd!` | 从当前行发到文件末尾 |
| n | `:FloatermReplSendFromBegin!` | 从文件开头发到当前行 |
| n | `:FloatermReplSendAll!` | 发送整个文件 |
| n/v | `:FloatermReplSendWord` | 发送光标下的单词或选区 |
| **标记** |
| n/v | `:FloatermReplMark` | 标记选区，留着后面发 |
| n | `:FloatermReplSendMark` | 发送之前标记的代码 |
| n | `:FloatermReplShowMark` | 看一下之前标记了什么 |

> Send 命令：不带 `!` 光标跳到下一行（方便连续执行），带 `!` 光标不动。`n` = normal 模式，`v` = visual 模式。

---

# AsyncRun 集成

跟 [asyncrun.vim](https://github.com/skywind3000/asyncrun.vim) 配合，在浮动终端里跑命令。插件自动注册了三个 runner：

- **`floaterm_right`** — 右侧垂直分割
- **`floaterm_float`** — 浮动窗口
- **`floaterm_bottom`** — 底部水平分割

```mermaid
graph TB
    AsyncRun[AsyncRun 命令] --> SelectType{选择位置}
    SelectType -->|floaterm_right| Right[垂直分割 + 右侧]
    SelectType -->|floaterm_float| Float[浮动窗口]
    SelectType -->|floaterm_bottom| Bottom[水平分割 + 底部]
    Right --> FindExist{已有终端?}
    Float --> FindExist
    Bottom --> FindExist
    FindExist -->|是| OpenExist[打开已有终端]
    FindExist -->|否| CreateNew[创建新终端]
    OpenExist --> SendCmd[发送 cd + 命令]
    CreateNew --> SendCmd
    SendCmd --> FocusCheck{focus 参数}
    FocusCheck -->|focus=0| BackEditor[回到编辑器]
    FocusCheck -->|focus=1| InTerm[留在终端]
```

用法示例：

```vim
:AsyncRun -mode=term -pos=floaterm_float echo "Hello, World!"
:AsyncRun -mode=term -pos=floaterm_right python %
:AsyncRun -mode=term -pos=floaterm_bottom node %
```

---

# 终端列表

`:FloatermFzfList` 命令通过 FZF 列出所有 floaterm 终端窗口，方便快速切换。每个条目会显示终端的程序类型、缓冲区号、标题、命令、窗口类型和位置等信息。

```vim
:FloatermFzfList
```

---

# 核心变量

| 变量 | 类型 | 说明 |
|------|------|------|
| `g:floaterm_ai_lst` | List | AI 终端的缓冲区号列表 |
| `g:floaterm_ai_programs` | List | AI 程序配置 |
| `g:floaterm_repl_dict` | Dict | `{filetype}-{bufnr}` → REPL 终端 bufnr 的映射 |
| `g:floaterm_repl_programs` | Dict | 文件类型 → REPL 命令列表 |
| `g:floaterm_prog_split_ratio` | Float | 分割窗口比例，默认 0.38 |
| `g:floaterm_prog_float_ratio` | Float | 浮动窗口比例，默认 0.45 |
| `g:floaterm_prog_col_row_ratio` | Float | 宽高比阈值，超过这个值自动用右侧分割而不是底部，默认 2.5 |

---

# 依赖

**必需**
- Vim 8+（要有 `:terminal`）或 Neovim
- [vim-floaterm](https://github.com/voldikss/vim-floaterm)
- [fzf.vim](https://github.com/junegunn/fzf.vim) — 交互式选择要用

**AI 功能**
- 装一个支持 CLI 交互的 AI 工具就行，比如 `claude`、`codex`、`opencode`

**REPL 功能**
- 对应语言的 REPL 程序：Python 的 `ipython`/`python`，R 的 `radian`/`R`，Node.js 的 `node` 等

**AsyncRun 功能**
- [asyncrun.vim](https://github.com/skywind3000/asyncrun.vim)

---

# 类似插件

如果你用 Neovim 想要更深度的 AI 集成，可以看看这些：

- [sidekick.nvim](https://github.com/folke/sidekick.nvim) — Copilot NES + AI CLI 终端，folke 出品
- [avante.nvim](https://github.com/yetone/avante.nvim) — 在 Neovim 里复刻 Cursor 的体验
- [opencode.nvim](https://github.com/nickjvandyke/opencode.nvim) — opencode 的 Neovim 深度集成
- [codecompanion.nvim](https://github.com/olimorris/codecompanion.nvim) — 支持多家 LLM 的 AI 编码助手

REPL 方面：

- [vim-repl](https://github.com/sillybun/vim-repl) — 纯 Vim 的 REPL，支持 ipython 调试
- [iron.nvim](https://github.com/Vigemus/iron.nvim) — Neovim Lua 生态的 REPL 方案
