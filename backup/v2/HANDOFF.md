# 舞萌 Festival 转场复刻 · 交接文档

> 本文档供后续 agent 阅读，继续推进入场/出场动画的复刻工作。

## 1. 任务背景

- 目标：用 HTML / CSS / JS 复刻舞萌 Festival 转场动画（参考视频 `【Festival】舞萌转场复刻_16比9.mov`）。
- 素材：
  - 参考视频：1920×1080, 60fps, 150 帧（2.5s），位于工作区根目录。
  - 静态稳态 SVG：`舞萌转场复刻_16比9.svg`，viewBox `0 0 3840 2160` (16:9)，约 2.8MB，内嵌 ~250+ 个 base64 PNG（多数透明背景，画面白色区域是透明像素露出白底）。
- 视频时间线：
  - 帧 0~54：**入场动画**（约 0–900ms）
  - 帧 55~85：稳态（与 SVG 一致）
  - 帧 86~149：**出场动画**（尚未实现）

## 2. 已完成

- [x] 用 ffmpeg 把视频拆成 150 张单帧 PNG：`frames/f_001.png` … `frames/f_150.png`
- [x] 入场动画第一版实现：[index.html](index.html)
- [x] 启动了本地静态服务：`python -m http.server 8765`，访问 http://localhost:8765/index.html
- [x] 通过 playwright 截取多组关键时刻验证，并与参考帧 vstack 对比

## 3. SVG 图层结构（必读）

`舞萌转场复刻_16比9.svg` 中各 `<image>` 都在 `<defs>` 中（id `img1`…`img117`），`<use>` 组织如下层级（从底到顶）：

```
<svg viewBox="0 0 3840 2160">
  <g id="底板">                  // 背景大色块
    <g id="左上小方块">           // 装饰像素方块
    <use id="左上底板" href="#img1" .../>
    <g id="右下小方块">
    <use id="右下底板" href="#img33" .../>
  </g>
  <use id="滴拉熊" href="#img34" x="1420" y="580"/>   // img34 为 1000×1000，中心刚好 (1920,1080)
  <g id="HOLD">  ...  </g>      // 8 条 hold 注 (img35-41)
  <g id="Slide"> ...  </g>      // 6 条 slide (img42-47)
  <g id="两角淡紫">
    <g id="右下"> <use="右下粉大三角"/> <g id="右下粉色小方块">...</g> </g>
    <g id="左上"> <use="左上粉大三角"/> <g id="左上粉色小方块">...</g> </g>
  </g>
  <g id="两角白色">              // 与上一组同结构，三角更小，最上层
    <g id="右下"> ... </g>
    <g id="左上"> ... </g>
  </g>
</svg>
```

注意：`两角淡紫` 和 `两角白色` 内的 `id="左上"` / `id="右下"` 是重复 ID，CSS 用属性选择器区分：
`#两角淡紫 > [id="左上"]`、`#两角白色 > [id="右下"]` 等。

## 4. 入场动画规则（来自用户口述）

1. **滴拉熊（chip）**：
   - 始终居中，scale 从 3.2 → 0.86（缩过头，undershoot）→ 1.04（轻微回弹）→ 1
   - 顺时针旋转：380° → 0°
   - 透明度 0 → 1（前期保持半透明，作为大圆盘叠加效果）
   - 入场期间需置于绘制顺序最顶层（已在 JS 中 `svg.appendChild(chip)` 处理）
2. **左上 / 右下 子层**（底板小方块、底板、两角淡紫、两角白色 各 4 组）：
   - 从远处对角线方向飞入
   - 带 **overshoot**：越过最终位置一点（多靠拢）→ 再回弹至最终位置
3. **Slide / HOLD**：每条单独从 **左下 → 右上** 滑入；每条 duration / delay 略有差异，由 JS `jitterSlides()` 随机注入（0.40-0.70s）

## 5. 当前实现要点

文件：[index.html](index.html)

- 用 `fetch()` 加载 SVG，把整段 SVG 文本 `innerHTML` 注入 `.frame` 容器
- CSS 关键帧均使用 `linear` timing-function + 关键帧百分比布置缓动（避免 `cubic-bezier` 与关键帧 % 互相干扰导致"前期一闪而过"）
- chip 用 `transform-box: fill-box; transform-origin: 50% 50%;` 保证围绕自身中心旋转/缩放
- 角组用 SVG userspace 的 `translate(...)` 做对角线滑入
- `elevateChip()` 在加载后立刻把 chip 移到 SVG 末尾，使其在所有图层之上绘制；稳态时角组三角不覆盖中心区域，所以永久置顶视觉无副作用
- `jitterSlides()` 给每个 `#HOLD > use` 和 `#Slide > use` 注入随机 `animation-duration` / `animation-delay`

当前 duration 配置（CSS 变量）：

| 变量 | 值 | 含义 |
|------|----|------|
| `--dur-chip` | 0.95s | 滴拉熊整体 |
| `--dur-base` | 0.75s | 底板（含小方块） |
| `--dur-purple` | 0.78s | 两角淡紫，delay 0.05s |
| `--dur-white` | 0.78s | 两角白色，delay 0.10s |
| `--dur-bg` | 0.65s | slide/hold 默认 |

## 6. 验证结果

最新对比图：[cmp_final2.png](cmp_final2.png)（上行=参考视频帧 15/22/30/40/50/55，下行=我方对应时刻截图）

- ✓ 稳态匹配（参考 f55 vs `shot_steady.png`）
- ✓ 三轴叠加层级正确：底板 < 角组 < chip
- ✓ 各分层动画方向正确（chip 缩放旋转、角组对角 overshoot、slide 从 BL→TR）
- ✗ 整体节奏仍比参考视频晚约 100-200ms：
  - 我方 f15 (250ms) 才刚出现少量斜线，参考视频此时大圆盘 chip 已半透明铺满
  - 我方稳态到达时刻约 833ms，参考 ~700-800ms

## 7. 用户标注的已知问题（已澄清）

- ✅ "白色扫光"误判：参考视频里的"白色对角条"其实是 PNG 透明部分露出白底，**不是额外动画元素**。所有素材都在 SVG 里
- ✅ chip 入场是"大→小→略放大"的 overshoot，不是单调缩小
- ✅ 角组是"靠拢 + overshoot + 回弹"，不是单调滑入
- ✅ slide 方向是 BL→TR，不是 TR→BL
- ⚠️ v5_grid.png 中 1400ms 帧异常空白 → 已澄清：那次是用户手动点了"重播"按钮干扰了截图，**不是 bug**

## 8. 下一步建议

候选方向（任选其一/组合）：

1. **继续微调入场到帧级匹配**：
   - 把 chip "大圆盘可见阶段"再提前（目前 5%-32%，可改 0%-25% 加更长高 opacity 段）
   - 进一步缩短 `--dur-base` / `--dur-purple` 到 0.65s 左右
   - 用 playwright 截更密的帧（每 30ms 一次），叠加 ssim 量化对比
2. **实现出场动画（帧 85-149，约 1.08s）**：
   - 需要先用密集采样（与 `frames_grid/grid_in_dense.png` 同样方法）查看出场关键帧序列
   - 推测：可能是入场反向 + 不同节奏
3. **加播放控制**：进度条、暂停、单帧步进，便于调试

## 9. 工程便捷命令

```powershell
# 启动本地服务
cd "e:\NewFiles\py训练\舞萌opus复刻_svg"
python -m http.server 8765

# 拆视频帧（已完成）
ffmpeg -v error -i "【Festival】舞萌转场复刻_16比9.mov" -vsync 0 "frames/f_%03d.png"

# 生成入场密集网格（55 帧）
ffmpeg -v error -y -i "【Festival】舞萌转场复刻_16比9.mov" `
  -vf "select='between(n,1,55)',scale=320:180,tile=11x5" -frames:v 1 frames_grid/grid_in_dense.png

# 生成出场密集网格（用于下一步）
ffmpeg -v error -y -i "【Festival】舞萌转场复刻_16比9.mov" `
  -vf "select='between(n,85,149)',scale=320:180,tile=11x6" -frames:v 1 frames_grid/grid_out_dense.png
```

playwright 截图模板（在 vscode 浏览器工具内）：
```js
await page.setViewportSize({ width: 1280, height: 720 });
await page.goto('http://localhost:8765/index.html?t=' + Date.now(), { waitUntil: 'load' });
await page.waitForSelector('svg');
const start = Date.now();
for (const t of [60, 200, 400, 700, 1000]) {
  while (Date.now() - start < t) await page.waitForTimeout(5);
  await page.screenshot({ path: `e:/NewFiles/py训练/舞萌opus复刻_svg/cap_t${t}.png` });
}
```

## 10. 用户偏好（沟通约定）

- 用户用 zh-CN 沟通
- 用户在每轮结束时希望 agent 主动问"下一步要怎么做"（不要自己结束对话）
- 用户有 ffmpeg 可用，遇到缺失 PNG 资源会补
- 截图验证 / 工程化的中间产物（v2_*, v3_*, cmp_* 等）可以保留在根目录，不影响

---

**文件清单（根目录）：**
- `index.html` — 主页面
- `舞萌转场复刻_16比9.svg` — 原始 SVG 资源
- `【Festival】舞萌转场复刻_16比9.mov` — 参考视频
- `frames/` — 150 帧 PNG
- `frames_grid/` — 关键帧概览网格
- `cmp_final.png`, `cmp_final2.png` — 与参考视频上下行对比图
- `v6_grid.png` — 最新版动画进程网格（12 时刻）
- `shot_steady.png` — 稳态截图
- `HANDOFF.md` — 本文件
