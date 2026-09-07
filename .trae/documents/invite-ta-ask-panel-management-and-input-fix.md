# 【邀请ta】输入栏文字溢出修复 + 邀请分组批量管理

## Context（背景）

用户反馈聊天页底部的【邀请TA】半框（`#chat-ask-panel`）存在两个问题：

1. **底部输入栏占位文字「想邀请TA做什么」飞出输入栏**：`#chat-ask-input` 在安卓 Chrome 上会被 `mobile-adapt.js` 转成 contenteditable `.ce-box` div。键盘弹起、半框被 `syncAndroidKb` 平移时，`ce-box` 文字停在旧合成层位置，表现为「文字与输入框分离 / 文字飞出输入栏」。这与 v3.26.x #118 已修复的 `.tc-input.ce-box` 是同一类问题——本输入栏缺 `will-change:transform` 的常驻合成层保护。
2. **顶部分组栏右侧缺少管理功能**：`renderInviteBank()` 的顶部分组栏只有分组 chips + 「＋ 分组」，没有**批量管理**（全选/删除/移动）、**分组的重命名/删除**。字卡本身的单条编辑/删除（`poke-card-ops`）已有，但缺少批量与分组级管理。

用户确认需求（已澄清）：
- 批量管理动作 = **全选 + 删除 + 移动**（对齐字卡库「我的添加」管理页）。
- 自建分组支持**重命名 + 删除**；「预设」为系统内置分组，只可加字卡、不可改名/删除。

目标：修复输入栏文字溢出，并给【邀请TA】分组栏补上批量管理 + 分组管理，交互与 `ta-invite.js`「我的添加」一致。

## 涉及文件（均在 AI-A 域内，不越界）

- `src/js/chat.js` — 邀请 bank 渲染 + 新增批量/分组管理逻辑
- `src/css/chat-main.css` — 输入栏 ce-box 修复 + 分组管理小样式
- `src/css/dark.css` —（可选）若批量条按钮暗色下突兀，补一行覆盖；默认复用现有 `.ti-batch-*` 即可，与「我的添加」行为保持一致

## 改动一：输入栏文字飞出修复

在 `src/css/chat-main.css` 的 `.chat-ask-input:focus`（约 L343-349，已有 `transform:translateZ(0)`）旁，新增一条与 #118 `.tc-input.ce-box` 完全同款的基础态保护：

```css
/* v3.x：邀请TA/问问TA 输入栏转 .ce-box 后常驻合成层——防 syncAndroidKb 平移残留文字旧位（同 #118 tc-input.ce-box） */
.chat-ask-input.ce-box { will-change: transform; }
```

仅影响安卓 ce-box 态，iOS / 原生输入不受影响。

## 改动二：顶部分组栏批量管理 + 分组重命名/删除

在 `src/js/chat.js` `renderInviteBank()`（L5177-5233）上扩展。复用现有存储函数 `myInviteG()` / `myInviteGroupsSave()` / `myInviteView()` / `myInviteCurGroupKey()`，复用 `window.openModal`（`personalize.js`，支持 `pills` 分组选择）。

### 1) 新增状态（紧跟现有 `myInviteCurGroup` 相关变量附近定义）

```js
let tiInviteBatch = false;      // 批量管理模式态
let tiInviteSel = new Set();    // 当前 user 分组内选中字卡的下标集合
```

- 切换分组 chip、退出批量、`contact-switched` 时清空 `tiInviteSel`、`tiInviteBatch=false`。
- 预设分组（`g.key==='__preset'`）**不进**批量模式：点「批量管理」时若当前分组是预设 → `toast('预设为系统内置分组，请切换到自建分组后批量管理')` 并不进入。

### 2) 顶部分组栏渲染（`renderInviteBank` 内）

- 保留现有分组 chips + 「＋ 分组」。
- 在 chips 末尾追加「批量管理」chip（`.emoji-g-chip inv-g-batch`），进入后变选中/「完成」态以表示可退出。
- 批量模式下，每个**自建分组** chip 追加迷你操作符：
  ```js
  chip.innerHTML = esc(label)+count
    + '<span class="inv-g-op" data-op="rn" data-gk="…">✎</span>'
    + '<span class="inv-g-op" data-op="rm" data-gk="…">✕</span>';
  ```
  ✎ 重命名、✕ 删除分组；`stopPropagation` 不触发分组切换，且只对 `user` 分组显示（预设 chip 保持纯文本）。
  - 重命名：`openModal('重命名分组', curName, cb)`；查重（含其他分组同名）；改 `g[0]` 后保存渲染。
  - 删除分组：`openModal` 确认；从 `myInviteG()` 移除该组及其字卡；`myInviteCurGroup` 交由 `myInviteCurGroupKey()` 兜底回退；保存渲染。

### 3) 字卡渲染（批量态）

- 正常态：保持现状（预设 = 点击发送；自建组 = `poke-card-ops` 单条 ✎/✕）。
- 批量态（仅自建分组）：每行切为批量 checkbox 视图（对齐 ta-invite `.ti-batch-row`）：
  ```js
  item.classList.add('invite-batch-item');
  item.innerHTML = '<label class="inv-batch-cb"><input type="checkbox" class="inv-batch-cb-in" data-bidx="'+i+'"'+(SEL_HAS?' checked':'')+'></label><div class="cc-txt"><div class="t">'+escTxt(c)+'</div></div>';
  ```
  勾选切换 `tiInviteSel`，同步底条「已选 N」与删除/移动按钮可用态。批量态不绑定「点击发送」。

### 4) 批量底条（`#invite-list` 之后，sticky 贴底，复用 `.ti-batch-bar` 系列类）

```html
已选 N ｜ [全选] [移动] [删除] [取消]
```
- 全选：当前自建分组全部字卡 全选/取消全选。
- 移动：`openModal('移动到分组','',cb,{ pills:[其它分组名], pill:第1个, noInput:true })`；逐条从当前组 `splice` → push 到目标组（目标不存在则 `push([名,[…]])`）；保存渲染，toast 已移动 N 条。
- 删除：确认弹窗；按下标降序移除；保存渲染，toast 已删除 N 条。
- 取消：退出批量、清选择、重渲染。
- 删除/移动后若当前组变空，`myInviteCurGroupKey()` 兜底回退分组。

### 5) 复用说明

- 批量条直接用 ta-invite 已在 `chat-pages.css` 定义好的 `.ti-batch-bar / .ti-batch-btn / .ti-batch-del-btn / .ti-batch-cnt`（全局类名，已含主题变量），无需重复定义样式，暗色行为与「我的添加」一致。
- 仅新增少量局部样式到 `chat-main.css`：`.inv-g-batch`（批量管理 chip 选中态）、`.inv-g-op`（分组 chip 上的 ✎/✕ 迷你按钮）、`.invite-batch-item`（对齐 top）、`.inv-batch-cb`（勾选框，仿 `.ti-batch-cb`）。

## 验证

1. `node --check src/js/chat.js`
2. `node build.mjs --check-sentinels`（不构建，防覆盖他人在建修复）——本改动需在 `build.mjs` `FIX_SENTINELS` 登记，逻辑锚点取：
   - `.chat-ask-input.ce-box { will-change: transform; }`（输入栏修复）
   - 批量态渲染特征串（如 `inv-batch-cb-in` 或 `tiInviteBatch` 相关、`inv-g-op`）
3. `node build.mjs` 后 `npm run verify`（无头 Chrome 390×844）：打开【邀请TA】面板
   - 输入栏占位文字位于框内、不溢出；
   - 顶部分组栏右侧出现「批量管理」，自建分组批量下出现 ✎✕，预设分组无操作符；
   - 批量：勾选多张字卡 → 全选/删除/移动组切/取消均生效；删除后分组数正确。
4. 真机（OPPO Find X9 Chrome）：安卓 ce-box 态键盘弹起输入栏文字不残留框外；批量操作流畅。
5. 改完在 WORKLOG 追加记录（当前构建者确认后收口）。

## 备注（构建纪律）

- 仅改 `src/`；构建与提交由当前构建者收口，本次改动不自行 build。
- 若预实现时 `dark.css` 的 `.ti-batch-btn` 白色按钮在暗色下明显突兀，可加一行暗色覆盖（属跨域改动，需在 WORKLOG 留记录）。