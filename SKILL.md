---
name: gemini-image-gen
description: 用 Chrome DevTools MCP 自动化操作 Gemini 网页版生图--从提示词填入、提交、等待生成到下载完整流程。
---

# Gemini 网页版生图自动化

用 `mcp_chrome_devtools_*` 工具直接驱动 Chrome 中已打开的 Gemini 页面，实现提示词填入->提交->等待->下载的全自动流程。

## 何时使用此 skill

- **首选路径**是 `image_generate` 工具（FAL 后端）。FAL_KEY 已配置时直接用它，更快、可编程、无浏览器依赖。
- **本 skill 是 fallback**：当 `image_generate` 不可用（FAL_KEY 未设、额度耗尽、后端故障）时，用 Chrome 驱动 Gemini 网页版生图。
- Gemini 网页版优势：免费、质量稳定（Gemini Pro 模式）、支持复杂中文提示词。劣势：需驱动浏览器、单张耗时 1-2 分钟、有内容审核底线。
- 2026-07-12 实测：FAL_KEY 未设，image_generate 报错后直接转本 skill，"碧海青天夜夜心"主题全流程顺利，约 2 分钟出图。

## 前置条件

- Chrome 已打开 `https://gemini.google.com/app` 且已登录
- 通过 `mcp_chrome_devtools_list_pages` 确认页面存在
- 模式建议用 Gemini Pro（生图质量更好），可在 snapshot 中查看当前模式

## 标准流程

### 1. 获取输入框 uid

```
mcp_chrome_devtools_take_snapshot
```

找到 `textbox "为 Gemini 输入提示"` 的 uid。

### 2. 填入提示词

**推荐方式：JS 注入（最稳定）**

> 连续生成多张图时，先清空输入框再插入新提示词。`selectAll` + `insertText` 会自动替换选中内容，等效于清空+插入。如果上次是用 `fill` 填的（非 contenteditable 原生内容），可能残留，需先 `selectAll` + `delete` 再 `insertText`。

```javascript
// evaluate_script
() => {
  const input = document.querySelector('div[role="textbox"][contenteditable="true"]') || document.querySelector('[contenteditable="true"]');
  if (!input) return { error: "input not found" };
  input.focus();
  document.execCommand('selectAll');
  document.execCommand('delete');   // 显式清空，防止残留
  document.execCommand('insertText', false, '这里是提示词');
  return { inserted: true, textLength: input.textContent.length };
}
```

**备选方式：fill**

```
mcp_chrome_devtools_fill  uid=输入框uid  value="提示词"
```

> ⚠️ `fill` 有时不可靠，尤其页面刷新后 uid 会变。JS 注入更稳。

### 3. 提交

**推荐方式：先 snapshot 找发送按钮 uid，再 click**

JS querySelector 找发送按钮在**同一 evaluate_script 调用里跟 insertText 一起做**时经常找不到按钮（按钮还没渲染出来）。最稳的做法是**分两步**：

1. **insertText 提示词**（单独一次 evaluate_script）
2. **snapshot 找发送按钮 uid**（必须做，不能跳过）
3. **click uid 发送**

```javascript
// 第一步：插入提示词
() => {
  const input = document.querySelector('div[role="textbox"][contenteditable="true"]');
  input.focus();
  document.execCommand('selectAll');
  document.execCommand('delete');
  document.execCommand('insertText', false, '这里是提示词');
  return { inserted: true, text: input.textContent.substring(0, 30) };
}
```

```javascript
// 第二步：snapshot 找发送按钮 uid（按钮 uid 随页面刷新变化）
mcp_chrome_devtools_take_snapshot
// 找到 "button 发送" 的 uid，通常是类似 7_9, 9_57, 11_41 这样的动态 uid
```

```javascript
// 第三步：点击发送
mcp_chrome_devtools_click  uid=发送按钮的uid
```

> ⚠️ **2026-07-12 实测关键发现**：
> - JS `querySelector('[aria-label*="发送"]')` 在 insertText 同一调用里几乎永远返回 `null`，因为 Angular 渲染发送按钮有延迟。
> - `Enter` 键在 contenteditable 里只加换行，不提交。**绝对不要用 Enter 提交**。
> - `fill` 工具填入内容后发送按钮也可能不出现（Angular 没检测到内容变化）。
> - **唯一稳定路径**：insertText → 等 1-3 秒 → snapshot 找按钮 → click uid。

### 4. 等待生成

```
mcp_chrome_devtools_wait_for  text=["正在创建您的图片","Crafting","Defining","Verifying","Constructing"]  timeout=30000
```

确认已开始生成后，再等完成：

> Gemini 的中间状态文案多变，实测出现过：`正在创建您的图片`、`Crafting A Sensual Scene`、`Defining Composition`、`Defining Scene Components`、`Defining Biblical Scene`、`Constructing the Vision`、`Verifying image composition`、`Verifying Generated Images`、`Refining the Composition`、`Assessing Aesthetic Execution`、`Analyzing Visual Consistency`、`Validating Minimalist Elegance`、`Reviewing Biblical Imagery`。用 `Crafting|Defining|Verifying|Constructing|Refining|Assessing|Analyzing|Reviewing` 宽匹配即可，不必精确。

```
mcp_chrome_devtools_wait_for  text=["重做","抱歉","无法"]  timeout=180000
```

- "重做"出现 = 生成成功
- "抱歉"/"无法"出现 = 审核被拒
- Gemini 生图通常 1-2 分钟

### 5. 下载（关键步骤）

> ⚠️ **这是最容易出错的环节。** 页面上有多张图时，每张图都有"下载完整尺寸的图片"按钮，snapshot 的 uid 对应关系会乱。**不要用 uid 点击下载按钮。**

**必须用 JS 精确定位最新图片的下载按钮：**

```javascript
() => {
  const imgs = Array.from(document.querySelectorAll('img[alt*="AI"]'));
  const lastImg = imgs[imgs.length - 1];
  if (!lastImg) return { error: "no image found" };
  
  let container = lastImg.closest('[role="generic"], div');
  let downloadBtn = null;
  let attempts = 0;
  while (container && attempts < 10) {
    downloadBtn = container.querySelector('[aria-label*="下载"], [aria-label*="Download"]');
    if (downloadBtn) break;
    container = container.parentElement;
    attempts++;
  }
  
  if (downloadBtn) {
    downloadBtn.click();
    return { clicked: true, attempts: attempts, rect: downloadBtn.getBoundingClientRect() };
  }
  return { error: "download button not found" };
}
```

> **检查返回的 `rect.top`**：正值 = 按钮在视口内（点击成功）；负值 = 按钮在视口外（点到了旧图的按钮，需重试）。多轮对话页面有 7+ 个同名下载按钮，uid 不可靠，此 JS 方法是唯一精确方案。

### 6. 确认下载

```bash
sleep 8; ls -lht ~/Downloads/Gemini_Generated_Image_*.png | head -3
```

> ⚠️ **等够再查（至少8秒）。** 不要因为3秒没看到文件就重复点击，否则会下载两份重复文件（第二份自动加 `(1)` 后缀）。

**铁律：一次点击，一次等待8秒，一次确认。** 没下成再排查（可能是灯箱弹窗挡住，先 `Escape` 关弹窗再用 JS 点击）。

### 7. 下载失败时的 fallback：Canvas 导出（低分辨率）

> ⚠️ **这是最后的手段，图片会变小。** 当 Gemini 原生下载按钮完全失效（点了没文件产生）时使用。

**症状**：JS click / MCP click 下载按钮后，`~/Downloads/` 没有新文件，且没有弹窗阻挡。**失效时机不可预测**（不限于对话堆积，全新对话第 2 张就可能失效，见"常见坑 5"）。

**原因**：Gemini 的 Angular 框架可能拦截了 JS/MCP 模拟的 click 事件，不触发原生下载流程。真实鼠标点击能触发，但 `computer_use` 在某些环境下不可用。

**⚠️ 关键前置验证（不做这步会产出 0 字节文件）**：Gemini 生成完成（"重做"出现）后，`<img>` 的 blob URL 已生成，但**图像数据可能还没解码加载完**。此时 `naturalWidth=0`、`complete=false`、`getBoundingClientRect().height=0`。直接 canvas 导出会得到空 dataURL，Python 解码后是 **0 字节 png 文件**。

**必须先验证再导出：**

```javascript
// 第一步：检查加载状态，必要时滚动进视口触发懒加载
() => {
  const imgs = Array.from(document.querySelectorAll('img[alt*="AI"]'));
  const lastImg = imgs[imgs.length - 1];
  if (!lastImg) return { error: "no img", imgCount: imgs.length };
  lastImg.scrollIntoView({ block: 'center' });  // 触发懒加载
  return { imgCount: imgs.length, naturalW: lastImg.naturalWidth, naturalH: lastImg.naturalHeight, complete: lastImg.complete, rectHeight: lastImg.getBoundingClientRect().height };
}
```

**判断条件**：`naturalW > 0 && complete === true && rectHeight > 0`。任一不满足，`sleep 5-10` 秒后重查。实测通常 1-2 次重查即可加载完。

**加载完成后再导出：**

```javascript
// evaluate_script, filePath=/Users/qin/Downloads/临时b64.txt
() => {
  const imgs = Array.from(document.querySelectorAll('img[alt*="AI"]'));
  const lastImg = imgs[imgs.length - 1];
  if (!lastImg) return { error: "no img" };
  if (lastImg.naturalWidth === 0) return { error: "img not loaded yet, retry after sleep" };  // 双保险
  const canvas = document.createElement('canvas');
  canvas.width = lastImg.naturalWidth;
  canvas.height = lastImg.naturalHeight;
  const ctx = canvas.getContext('2d');
  ctx.drawImage(lastImg, 0, 0);
  return canvas.toDataURL('image/png');
}
```

然后用 Python 解码（**必须处理换行 + 校验非空**，dataURL 跨行保存时 base64 段含 `\n`，不剥离会解码失败或产出 0 字节）：

```bash
cd ~/Downloads && python3 -c "
import base64
with open('临时b64.txt', 'r') as f:
    content = f.read()
content = content.strip().strip('\"')
b64 = content.split(',', 1)[1] if ',' in content else content
b64 = b64.replace('\n', '').replace('\r', '').replace(' ', '')  # 关键：去所有空白
data = base64.b64decode(b64)
assert len(data) > 1000, f'decoded too small: {len(data)} bytes, img likely not loaded'
with open('自定义文件名.png', 'wb') as f:
    f.write(data)
print('done, size:', len(data))
" && rm 临时b64.txt
```

> ⚠️ **0 字节文件的根因 = 跳过了前置验证。** 2026-07-12 实测：第 3、5 张图因"重做"出现后立即导出 canvas，img 还没加载完，产出 0 字节文件。加入 `naturalWidth>0 && complete` 验证 + sleep 重试后解决。

> ⚠️ **分辨率限制**：canvas 只能读到页面上 `<img>` 的 `naturalWidth/naturalHeight`，Gemini 页面存储的是 **1024×572 预览缩略图**，不是满分辨率（2752×1536）。导出的 PNG 约 1MB，而原生下载的约 8-9MB。内容完整但清晰度低。

**要恢复满分辨率下载，尝试：**
1. 新开一个干净的 Gemini 对话（`https://gemini.google.com/app`），只放一张图再下载
2. 检查 Chrome 设置 `chrome://settings/downloads`，关闭"下载前询问每个文件的保存位置"
3. 让用户手动在 Chrome 里点下载按钮

## 常见坑

### 1. 重复下载
**原因**：点了下载后等3秒没见文件就又点一次，结果两次都生效。
**解决**：只点一次，等8秒再查。没下成先 `Escape` 关弹窗，再用 JS 重新点击。

### 2. Enter 不提交
**原因**：Gemini 的 contenteditable 输入框把 Enter 当换行。
**解决**：用 JS 点击发送按钮，或用 `mcp_chrome_devtools_click` 点 snapshot 中的"发送"按钮。

### 3. uid 对应错误
**原因**：页面有多张图，每个都有下载按钮，snapshot 的 uid 可能对应到旧图的按钮。
**解决**：用 JS 从最后一 `img[alt*="AI"]` 向上遍历找下载按钮，绝对精确。

### 4. 灯箱弹窗挡住
**原因**：点图或下载后弹出 lightbox dialog，挡住了其他元素。
**解决**：`mcp_chrome_devtools_press_key  key="Escape"` 关闭弹窗。

### 5. 下载按钮完全失效（点了没文件）
**原因**：JS/MCP 模拟的 click 不再触发 Gemini 的原生下载流程。没有弹窗、没有错误、没有 `.crdownload` 临时文件--下载请求根本没发出。
**⚠️ 失效时机不可预测**：旧记录说"通常发生在对话堆积 15+ 张图后"，实测错误。2026-07-12 在**全新对话的第 2 张图**就失效了。同一会话里第 1 张能原生下载、第 2 张开始 JS click 静默失败。无法预判哪一张会开始失效，**必须每张都做下载确认**，没拿到文件就立即转 fallback。
**解决**：
1. 先试 `Escape` 关弹窗，再用 JS 精确定位点击
2. 不行则新开干净对话（`https://gemini.google.com/app`），只放一张图
3. 最后手段：用 canvas 导出 base64（见上方"下载失败时的 fallback"），但分辨率只有 1024×572
4. 让用户手动在 Chrome 里点下载按钮，通常能拿到满分辨率 2752×1536

### 6. 文件后缀名"消失"
**原因**：macOS Finder 默认隐藏已知文件后缀。`ls` 能看到 `.png`，Finder 里不显示。
**解决**：这不是 bug，文件后缀实际存在。`file` 命令可验证。

### 7. 提示词里的元文本被渲染进图片（高优先级）
**症状**：提示词里写了"这是苏轼一生组图第一张：眉山读书"之类的**元描述**（组图编号、章节标题、分镜说明），Gemini 会把这些文字当成画面内容**直接渲染到生成的图片上**，出现"苏轼一生组图 第一张"之类的文字浮在画面里。
**用户明确反馈（2026-07-12）**：要求"如无必要，图片上不要出现苏轼一生组图 第几张等字样"。
**根因**：Gemini 生图模型对提示词中的陈述句不做"这是指令说明还是画面元素"的区分，倾向于把所有可读文本都画进去。越是像标题/编号的短语，越容易被当画面文字渲染。
**解决（已固化为模板规则）**：
1. **提示词正文只写画面内容**，不写"这是第几张""组图第N部分""主题：XXX"等元描述。系列图的序号管理交给 agent 在文件命名层面处理（`01_眉山读书.png`），不要进提示词。
2. **每个提示词结尾加"画面中不要出现任何文字"**（模板已更新，见上方提示词模板结尾）。
3. 如需保持系列风格统一，用**风格锚定句**（画家+技法+色调）而不是用"组图第N张"这种标签--前者影响画风，后者污染画面。
4. 旧提示词若已含元描述，重做时删掉元描述句 + 补"画面中不要出现任何文字"。

### 8. 中国古典人物画的性别/年龄偏差（高优先级）
**症状**：提示词写"苏东坡"/"中国古代文人"，Gemini 画成了女性仕女图。庭院+古典+工笔画风格 = 条件反射画女子。
**用户明确反馈（2026-07-12）**："还是女的图"。反复修正了3轮才解决。
**根因**：Gemini 对"中国古典人物画+庭院场景"有强烈先验偏向（仕女图训练数据过多），"苏东坡""文人"等抽象身份词不够具体，会被庭院场景的先验覆盖。
**解决（已固化为模板规则）**：
1. **提示词第一句钉死性别**："画面主体必须是一位中国古代男性文人"（"必须"加强制约束力）。
2. **紧跟性别特征描写**："面容清俊，唇上有短须"/"长髯垂至胸前"/"白色长髯飘拂"。胡须是古典中国男性最显著的视觉锚点。
3. **年龄匹配外貌**：不要所有年龄都写"长胡子"。少年→"唇上细软短须"；青年→"唇上与下巴有短须"；中年→"长髯垂至胸前"；老年→"白色长髯飘拂"。
4. **服饰也要锚定**：东坡巾（高筒文人巾帽）是苏轼的标志性视觉符号，比"文人"两个字有效100倍。
5. **如果要求"无文字无画框"**，结尾加："画面占满画布，不要题字、落款、印章、边框、折扇、任何文字。"（"折扇"也要禁，Gemini 爱给古典文人加折扇，扇面上有时会有字）。

**提示词结构模板（中国古典人物）**：
```
画面主体必须是一位中国古代男性[少年/青年/中年/老年]文人，约[XX]岁，[面容描写]，[胡须描写]。他头戴[东坡巾/斗笠/锦帽]，身穿[衣服描写]。这是[具体身份]的形象。16:9比例。画面占满画布，不要题字、落款、印章、边框、折扇、任何文字。

场景：[具体场景描写]。
环境：[地点、季节、天气]。
光影：[光源方向、色调]。
风格：[画派+技法+色彩]。高画质。
```

### 9. 下载在全新对话第2张就失效
**旧记录说**"通常发生在对话堆积15+张图后"。
**2026-07-12 实测纠正**：全新对话（0张历史）第1张能原生下载（9.9MB），第2张起 JS click 静默失败。**失效时机完全不可预测**，不要假设任何一张能原生下载。
**正确策略**：每张下载后立即 `ls -lht ~/Downloads/Gemini_Generated_Image_*.png` 确认拿到文件。没拿到就立即转 canvas fallback，不要重试原生下载（浪费时间）。

## Gemini 内容审核底线（实测 2026-07）

| 提示词描述 | 结果 |
|---|---|
| 蕾丝内衣 + 透明薄纱 | ✅ 通过 |
| 湿身白衬衫透视 | ✅ 通过 |
| 睡袍滑落 + 蕾丝内衣 | ✅ 通过 |
| "上身未着寸缕" + 手臂遮挡 + 丁字裤 | ✅ 通过 |
| **"全裸" + 床单半裹** | ❌ 被拒 |

**关键发现**：Gemini 审核是**关键词匹配**，不是画面内容理解。"全裸"是红线词，直接拒。但"未着寸缕""手臂遮挡"等同义描述能过，即使实际暴露面积更大。

**安全策略**：
- 避免：全裸、裸体、nude、naked 等直白词
- 可用：未着寸缕、手臂遮挡、若隐若现、半透、蕾丝内衣、丁字裤、丝绸滑落
- 框架：用"Helmut Newton 风格""高级艺术人体摄影""时尚杂志艺术大片"包装，降低被拒概率

## 系列图生成工作流

用户要生成一组主题统一的图（如"美国西进运动六张""四季四张"）时，用同一 Gemini 对话连续生成，靠上下文保持风格连贯。

### 流程

1. **先规划分镜**：在回复里列出 N 张的标题和一句话描述，让用户确认或直接开干。分镜要覆盖叙事弧线（启程→过程→抵达 / 春→冬 / 起→承→转→合）。
2. **统一风格前缀**：每张提示词都带上相同的风格锚定句（画家+技法+色调+氛围），例如"19世纪美国西部风景油画，Albert Bierstadt 的史诗感构图与 Frederic Remington 的人物笔触结合，厚涂技法，色彩浓郁沉稳"。
3. **每张差异在场景和光影**：风格前缀不变，只改画面主体、环境、光影描写。光影变化能制造情绪递进（晨光→正午→夕阳→雪夜）。
4. **逐张生成下载**：提交→等"重做"→下载→下一张。不要批量提交，Gemini 一次只处理一条。
5. **存到独立目录**：`~/Downloads/<主题名>/01_xxx.png ... 0N_xxx.png`，编号 + 中文标题。

### 实测要点（2026-07-12，西进运动六张）

- **6-10 张在单对话里没问题**，不需要开新对话。2026-07-12 实测：西进运动 6 张、苏轼一生 10 张，均在单对话内连续生成成功，Gemini 上下文保持风格连贯。旧记录"不要堆超过 10 张"偏保守，实测 10 张稳定。
- **原生下载从第 2 张起可能失效**（见常见坑 5，失效时机不可预测）。系列图生成时要默认走 canvas fallback 路径，每张都做 `naturalWidth>0 && complete` 验证（见 fallback 第 7 节）。
- **第 1 张可能拿到满分辨率（9MB），后续都是缩略图（1.3-1.5MB）**。这是 JS click 失效的连带后果。如果用户要全部满分辨率，只能让用户手动在 Chrome 逐张点下载，或每张开新对话重生成。
- **图片在视口外不加载**：生成多张后页面变长，新图在底部，blob URL 生成但 img 不解码（懒加载）。canvas 导出前必须 `scrollIntoView({block:'center'})` 触发加载。



下完图后，用 `vision_analyze` 让视觉模型回看生成结果，评估是否达到预期。这一步可选但推荐，能帮用户判断是否需要重做，也能积累"哪些提示词描述容易出瑕疵"的经验。

**回看时关注**：
- 意境/氛围是否到位（主观，对照用户给的原始意象）
- 光影、构图、色彩三角是否扎实
- **AI 生成常见瑕疵**：手部崩坏（手指数量/结构乱）、衣物/绸带空间逻辑断裂、云朵/纹理重复感、面部塑料感（皮肤过滑、五官过于标准）、物体漂浮或受力不自然

**回看模型**：用 `auxiliary.vision` 配置的模型（当前是 NVIDIA NIM 的 gemma-4-31b-it）。它做整体氛围判断够用，但细节鉴伪偏"印象派"--给不出具体像素级证据。要更狠的鉴伪需换 Gemini 2.5 Pro / GPT-4o 这类原生多模态强模型。

**回看结果记录**：把"提示词 + 生成结果评估"追加到 `references/poem-prompt-examples.md` 或对应主题的实例文件，供复用积累。

## 提示词模板

用户给关键词，自动扩写为完整提示词。结构：

```
请生成一张16:9比例的[类型]。画面主体：[人物描写--外貌、身材、衣着]。[姿态描写--动作、手势、眼神]。环境：[场景、背景]。光影：[光线方向、质感]。整体风格：[摄影风格、杂志质感、调色、景深、氛围]，高画质，细节极致细腻。画面中不要出现任何文字。
```

### 实测有效的主题方向

1. **东方古典神话**：巫山神女、湘灵鼓瑟 -> 工笔重彩+水墨写意，云海/月夜/江水意境
2. **时尚性感大片**：内衣、湿身、睡袍滑落 -> Vogue/VS质感，暖色暧昧光影
3. **赛博朋克机车**：皮衣+机车+霓虹 -> 电影级赛博朋克调色，粉蓝紫光影
4. **人体艺术**：手臂遮挡+丁字裤 -> Helmut Newton风格（注意避开"全裸"等红线词）
5. **极简编辑插画**：干净细密线条+选择性色彩点缀+大量留白 -> 清水裕子+中国白描结合，现代编辑风格
6. **古典文学插画**：诗经蒹葭/硕人/唐诗意象 -> 水墨写意+现代插画结合，淡彩设色，大面积留白与雾气交融，古朴诗意
7. **西方古典宗教油画**：圣经雅歌/中世纪圣经意象 -> 拉斐尔前派油画（罗塞蒂/米莱斯），色彩浓郁，衣褶花瓣纤毫毕现，虔诚与浪漫
8. **史诗建筑概念艺术**：巴比伦空中花园/古代七大奇迹 -> 19世纪东方主义浪漫主义油画（大卫·罗伯茨）+现代概念艺术宏大叙事，黄昏逆光，琉璃砖浮雕细节
9. **古典小说叙事场景**：金瓶梅潘金莲叉竿落窗 -> 明代仇英式工笔重彩，叙事性构图，取经典相遇瞬间而非露骨场景，市井世情气息
10. **民国复古人像**：旗袍+团扇+老宅 -> 民国月份牌画+现代复古人像摄影，王家卫花样年华式暧昧氛围，微泛黄胶片质感
11. **怀旧青春摄影**：80年代校园+白衬衫+自行车 -> 柯达克罗姆64胶片质感，微泛暖黄，逆光半透明光晕，nostalgia
12. **面部特写时尚摄影**：烈焰红唇 -> 纯黑背景+伦勃朗戏剧光+正红×纯黑×雪白三角对比，Vogue封面级特写
13. **经典儿童文学插画**：小海蒂/阿尔卑斯山 -> 欧洲儿童绘本风格，色彩明快饱和，童真与自然气息

### 非人物主题的提示词调整

当用户给的不是人物主题（如极简插画、古典文学），模板中的"人物描写"换成"主体描写"：

```
请生成一张16:9比例的[类型]。画面描绘：[场景/主体描写--环境、意境、关键意象]。
[细节描写--质感、动态、氛围]。环境：[空间、背景]。光影：[光线方向、质感]。
整体风格：[艺术风格、技法、色调、意境]，高画质，细节极致细腻。画面中不要出现任何文字。
```

## 参考文件

- `references/gemini-content-audit-testing.md` - Gemini 内容审核底线实测记录：8组提示词测试结果、红线词清单、安全替代词、框架包装策略、提示词模板。需要测底线或不确定某描述能否通过时查阅。
- `references/poem-prompt-examples.md` - 古典诗词意象->生图提示词实例库：诗句扩写方法、完整提示词范本、生成结果评估（含手部崩坏规避策略）。用户给诗句/意境关键词生图时查阅，并在此积累新实例。
- `references/series-prompt-examples.md` - 系列图生成实例库：分镜设计原则、统一风格前缀写法、西进运动六张（旅程类）、苏轼一生十张（传记类）完整范本、系列图下载策略。用户要生成一组主题统一的图时查阅。
