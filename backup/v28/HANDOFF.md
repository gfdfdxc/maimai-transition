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

## 4. 入场动画规则（来自用户口述 + 多轮迭代澄清）

1. **滴拉熊（chip）**：
   - **始终居中**（SVG viewBox 中心 = 1920, 1080）
   - **旋转**：**逆时针 CCW 半圈**，从 **180°（倒置）→ 0°（正立）**。不是多圈。
   - **缩放**：起始 **scale 4.5**（面包似铺满全屏）→ 0.86（缩过头 undershoot）→ 1.04（回弹 overshoot）→ 1
   - **透明度**：0 → 1（前期半透明作为大圆盘叠加效果）
   - **入场期间需置于绘制顺序最顶层**（已在 JS `elevateChip()` 处理）
   - **出场期间（待实现）**：用户说 “出场是倒置 180°然后转正到 0°”——出场同样从 0°转到 180°且放大？需在实现出场时再确认。
   - **transform 实现**：不要依赖 `transform-box: fill-box` + `transform-origin`（在 SVG `<use>` 上浏览器行为不一致），改用显式 `transform: translate(1920px,1080px) rotate(α) scale(s) translate(-1920px,-1080px)`。
2. **左上 / 右下 子层**（底板小方块、底板、两角淡紫、两角白色 各 4 组）：
   - 从远处对角线方向飞入
   - 带 **overshoot**：越过最终位置一点（多靠拢）→ 再回弹至最终位置
3. **Slide / HOLD**：每条单独从 **左下 → 右上** 滑入；每条 duration / delay 略有差异，由 JS `jitterSlides()` 随机注入（0.40-0.70s）

## 5. 当前实现要点（v9b，最新）

文件：[index.html](index.html)

- 用 `fetch()` 加载 SVG，把整段 SVG 文本 `innerHTML` 注入 `.frame` 容器
- CSS 关键帧均使用 `linear` timing-function + 关键帧百分比布置缓动（避免 `cubic-bezier` 与关键帧 % 互相干扰导致“前期一闪而过”）
- chip 用 **显式 `translate(1920px,1080px) rotate scale translate(-1920px,-1080px)`** 让变换围绕 viewBox 中心（不再依赖 transform-box: fill-box）
- 角组用 SVG userspace 的 `translate(...)` 做对角线滑入
- `elevateChip()` 在加载后立刻把 chip 移到 SVG 末尾，使其在所有图层之上绘制；稳态时角组三角不覆盖中心区域，所以永久置顶视觉无副作用
- `jitterSlides()` 给每个 `#HOLD > use` 和 `#Slide > use` 注入随机 `animation-duration` / `animation-delay`

当前 duration 配置（CSS 变量）：

| 变量 | 值 | 含义 |
|------|----|------|
| `--dur-chip` | **0.70s** | 滴拉熊整体（+ delay 0.10s，end@800ms） |
| `--dur-base` | **0.80s** | 底板（end@800ms） |
| `--dur-purple` | **0.77s** | 两角淡紫，delay **0.03s**（end@800ms） |
| `--dur-white` | **0.75s** | 两角白色，delay **0.05s**（end@800ms） |
| `--dur-bg` | **0.75s** | slide/hold 默认（jitter 后 0.55-0.80s） |

**v12 关键设计**：所有元素动画在 ~800ms 一起结束，避免"chip 比其他元素晚 350ms 结束"的不同步感。

chip 的 keyframes（v11）：
```
0%   rotate(-180) scale(4.50)  opacity 0     // 倒置、铺满、不可见
5%   rotate(-170) scale(4.30)  opacity 0.55
18%  rotate(-140) scale(3.60)  opacity 0.85
32%  rotate(-100) scale(2.50)  opacity 0.95
48%  rotate( -55) scale(1.40)  opacity 1
62%  rotate( -22) scale(0.96)  opacity 1
74%  rotate(  -8) scale(0.91)  opacity 1     // 轻 undershoot
86%  rotate(  -2) scale(0.97)  opacity 1
94%  rotate(   0) scale(1.015) opacity 1     // 轻 overshoot
100% rotate(   0) scale(1.00)  opacity 1
```

> ⚠️ **CSS rotate 方向陷阱**：正角度 = CW（顺时针）。从 `rotate(180deg) → rotate(0deg)` 角度递减，但视觉上是 CW（经过 90°→45°→0°）。**真正的 CCW 必须用负角度 `rotate(-180deg) → rotate(0deg)`**，经过 `-90°`（头朝左）→ `-45°` → 正立。v8/v9/v10 都搞错了，v11 修正。

角组 keyframes（v15，**底板不回弹 + 两角保留 v12 回弹 + 向远端放大**）：
```
/* 底板 · 单调缓出 */
settle-TL: 0%(-2400,-1500) → 100%(0,0)      (cubic-bezier(0.22,1,0.36,1))
settle-BR: 0%(+2400,+1500) → 100%(0,0)

/* 两角白色/淡紫 · v12 回弹 + scale(1.06) 以 (1920,1080) 为锚点 */
converge-TL: 0%(-2400,-1500) → 60%(+100,+60) → 82%(-25,-15) → 100%(0,0)
converge-BR: 0%(+2400,+1500) → 60%(-100,-60) → 82%(+25,+15) → 100%(0,0)
transform = translate(1920,1080) scale(1.06) translate(-1920,-1080) translate(animX, animY)
timing-function: linear  // 由关键帧控制节奏
```

> ⚠️ **为什么 scale(1.06)**：回弹偏移最大 100/60 px，会让三角远端（朝画面外的那一側）从屏幕外缘缩回 100/60 px露背景。以画面中心 (1920,1080) 为锚点 scale 1.06，让左上角 (0,0) 扩到 (-115.2, -64.8)，右下角 (3840,2160) 扩到 (3955.2, 2224.8)，出屏幕冷量 ≥ 115/65 px > 100/60，后面三角面包住屏幕边缘。因锚点在画面中心，三角趛中心的“显示边”几乎不变。

> ⚠️ **为什么底板单独不回弹**：底板覆盖整个半屏区域，一旦回弹露背景，露出面积不可接受；而两角只是装饰三角，金额尝试了护镕问题可控。

Slides 缓动（v10）：`animation-timing-function: cubic-bezier(0.16, 1, 0.3, 1)`（easeOutExpo-ish 由快到慢）

## 6. 验证结果

最新对比图：[cmp_final5.png](cmp_final5.png)（上=参考视频帧 10/15/22/30/40/55，下=我方对应时刻）

- ✓ 稳态匹配（参考 f55 vs `shot_steady.png`）
- ✓ 三轴叠加层级正确：底板 < 角组 < chip
- ✓ chip 旋转方向正确（CCW 半圈 180°→20°）
- ✓ chip 起始 scale 足够大（4.5x 铺满屏幕）
- ✓ chip f10 不可见、f15 半透明铺满、f22 大半透明——节奏与参考重合
- ⚠ f30/f40 我方 chip 还在旋转/缩小过程，参考已接近稳态（略偏慢 ~50-80ms）——可微调 dur 到0.70s 或调关键帧百分比
- ⚠ 参考 f10/f15 里的“散落灰色像素”是角组进入时的运动模糊拖尾（视频压缩产生），静态代码不会出现

## 7. 用户标注的已知问题（已澄清）

- ✅ "白色扫光"误判：参考视频里的"白色对角条"其实是 PNG 透明部分露出白底，**不是额外动画元素**。所有素材都在 SVG 里
- ✅ chip 入场是"大→小→略放大"的 overshoot，不是单调缩小
- ✅ 角组是"靠拢 + overshoot + 回弹"，不是单调滑入
- ✅ slide 方向是 BL→TR，不是 TR→BL
- ⚠️ v5_grid.png 中 1400ms 帧异常空白 → 已澄清：那次是用户手动点了"重播"按钮干扰了截图，**不是 bug**
- ✅ **chip 旋转方向**（v8 → v9 修正）：早期版本用 380°→0°（顺时针 CW + 多转一圈），用户明确否定。正确是 **180°→0°（逆时针 CCW 半圈）**
- ✅ **chip 起始尺寸**（v8 → v9 修正）：早期 3.2x 不够大，正确是 **4.5x（开场铺满屏幕）**
- ✅ **chip 入场延迟**（v9 → v9b 修正）：原本 delay=0 导致 chip 比角组先出现且早 ~150ms，正确节奏是 **delay 0.18s + dur 0.78s**（让角组/底板先动一点点再 chip 入场）

## 7.5. 演进历史

| 版本 | chip 旋转 | chip scale 起 | chip delay | chip dur | 备注 |
|------|----------|--------------|-----------|---------|------|
| v1-v7 | 多次迭代 (cubic-bezier / linear, 380°/410°/360°) | 3.0~3.2 | 0 | 0.95s | chip 不居中、被角组挡、节奏不准 |
| v8 | 380°→0° (CW, 多转 ~1 圈) | 3.20 | 0 | 0.95s | 用显式 translate trick 修复居中；但用户否定旋转方向+量+起始大小 |
| **v9b** | 180°→0° (CCW, 半圈) | 4.50 | 0.18s | 0.78s | 节奏与参考重合度大幅提升 |
| **v10** | 180°→0° (CCW, 半圈) | 4.50 | 0.18s | 0.78s | 末尾 overshoot 平滑化 (0.86→0.91→1.02→1)；角组回弹弱化 (220/130→100/60)；slides 改 cubic-bezier(0.16,1,0.3,1) 易出曲线 |
| **v11** | -180°→0°（真·CCW，半圈） | 4.50 | 0.18s | 0.78s | 关键修正：CSS rotate 角度递减实际是 CW，必须用负角度才是 CCW；同时缩短 dur-base/purple/white 0.78→0.55s 让角组在 ~500ms 内到位（用户反馈"偏晚"）；delay 微调 0.05/0.10 → 0.03/0.06 |
| **v12** | -180°→0°（真·CCW，半圈） | 4.50 | 0.10s | 0.70s | 对齐所有动画在 ~800ms 一起结束（用户反馈"chip 慢、其他快、没一起结束"）：chip 0.78→0.70s + delay 0.18→0.10s；base 0.55→0.80s；purple 0.55→0.77s + delay 0.03；white 0.55→0.75s + delay 0.05；slides jitter 0.40-0.70s → 0.55-0.80s |
| **v13** | -180°→0°（真·CCW，半圈） | 4.50 | 0.10s | 0.70s | **角组移除 overshoot 改 ease-out**（用户反馈："角组 overshoot 越过最终位置时，远端边缘从屏幕边缘缩回，露出网页黑色底色 + 中间白色"）：converge-TL/BR 由 4 关键帧（含 60%、82% overshoot/回弹）简化为 0%/100% 两帧；timing-function 从 `linear` 改为 `cubic-bezier(0.22, 1, 0.36, 1)` 保持"由快到慢"的视觉感 |
| **v14** | -180°→0°（真·CCW，半圈） | 4.50 | 0.10s | 0.70s | **重新加回角组回弹，但反向振荡**（用户："我需要回弹，只是想让你帮我解决回弹会显示背景或底色的问题"）：converge-TL/BR 变为 0%→70%(0,0)→85%(⁄40px,⁄24px)→100%(0,0)，即“到终点后反跳向左上/右下”而不是“冲过终点到中心”，这样远端边缘始纶覆盖屏幕外不露背景；timing-function 改回 `linear` 由关键帧控制节奏 |
| **v15** | -180°→0°（真·CCW，半圈） | 4.50 | 0.10s | 0.70s | **底板取消回弹 + 两角恢复 v12 回弹 + scale(1.06) 向远端放大** |
| **v16** | -180°→0°（真·CCW，半圈） | 4.50 | 0.10s | 0.70s | 两角分段 timing-function 修复"匀速感" |
| **v17** | -180°→0°（真·CCW，半圈） | 4.50 | 0.10s | 0.70s | 两角 + 底板提前到位（缩短 dur、取消 delay、底板 keyframes 30% 已到位） |
| **v18** | -180°→0°（真·CCW，半圈） | 4.50 | t0+0.10s | 0.70s | 全局 170ms 起始延迟修复"动画过快" |
| **v19** | -180°→0°（真·CCW，半圈） | 4.50 | t0+0.10s | 0.70s | **底板/两角去掉"提前到位"锚点 + 延长 dur**（用户反馈："滴拉熊入场速度合适，但其他元素，特别是背景，感觉太快"）：v18 中底板用 keyframes 0%→30%(已到位)→100%(保持) + easeOutCubic，导致 0→30% 段前 60ms 就走完一半位移；v19 改成纯 0%→100% 单调缓出 (整段 dur 才到位)，timing-function 由内联 easeOutCubic 提到 animation 上；两角 keyframes 60%/85% 谷值从 v17 的 40%/65% 往后推（让前段更慢），dur 0.55→0.70s/0.68s |
| **v20** | -180°→0°（真·CCW，半圈） | 4.50 | t0+0.10s | 0.70s | **修正 HOLD/Slide 滑入方向**（用户发现："右上角的 4 个 HOLD 在入场时应该是从右上到左下进入的，现在好像反了"）：对照参考视频 f15→f24 右上 1/4 区，确认 HOLD 端头先从**右上方**冒出再向左下铺展。原 `slide-BL-to-TR` 起点 (-2400, 1500) 是从左下飞向右上，方向错了。改为 `slide-TR-to-BL` 起点 (2400, -1500) → (0,0) |
| **v21** | -180°→0°（真·CCW，半圈） | 4.50 | t0+0.10s | 0.70s | **HOLD 与 SLIDE 方向分离**（用户进一步澄清："HOLD 是右上到左下，但 SLIDE 都是左下到右上"）：v20 把两者都改成右上→左下了。v21 恢复 `slide-BL-to-TR` keyframe 并将 `#Slide > use` 单独绑定它；`#HOLD > use` 继续用 `slide-TR-to-BL`。jitterSlides 只覆盖 duration/delay，不影响 animation-name，两组方向独立 |
| **v22** | 入场同 v21；**新增出场动画**（chip f95-115 放大淡出；底板/两角 f110-128 飞回角外；HOLD/SLIDE f120-140 维持入场方向继续飞出） | 4.50 | t0+0.10s | 0.70s | **出场动画实现**。关键技术坑：1) 每个元素挂入场+出场两段 animation，必须用 longhand `animation-name/duration/timing-function/delay/fill-mode` 列表声明（CSS shorthand 中混合 `var()` 时间值有解析歧义，导致 fill-mode 被吞或时间值错位）。2) 出场 animation 用 fill-mode `forwards`（不含 backwards），避免其 0% 关键帧在入场 active phase 中覆盖入场动画的插值。3) `keyframes from{}/to{}` 显式声明两端，不要省 from，因为 Chrome 多 animation 合成时空 from 会用 element 初始值而非入场动画末态。4) **chip 出场不是缩小到 0**，而是从 scale 1 放大到 3.5（参考视频是消散感的"大圆环展开"），是入场 scale 4.5→1 的真正逆向。5) Playwright 验证仍用 WAAPI pause+currentTime（因为 SVG fetch 有 700ms 延迟，page time 与 animation time 错位）。CSS 变量 `--t-exit-chip 1.58s / --t-exit-base 1.83s / --t-exit-bg 2.00s` 对应 f95/f110/f120 |
| **v23** | 同 v22，**出场动画修正三处**（用户反馈"出场不是入场反着来，到处都是问题"） | 同上 | 同上 | 同上 | 1) **chip-out 改为"先缩后放大淡出，无旋转"**：0%(scale 1)→25%(scale 0.82, undershoot)→100%(scale 3.80, opacity 0)；rotate 全程 0deg。2) **底板/两角出场加入"先内收再外飞"**：expand-TL/BR 与 converge-TL/BR-out 都加 20% 关键帧 (TL=+100,+60 / BR=-100,-60) 模拟入场 overshoot 的反向。timing-function 改 linear 由关键帧控速。3) **HOLD/SLIDE 出场提前到 1.83s**（与底板/两角同时起）：原 2.00s 太晚，jitter delay 1.83+0.08s、dur 0.40-0.52s。chip dur 0.33→0.50s、base 0.30→0.42s、bg 0.40→0.45s |
| **v24** | 同上 | 同上 | 同上 | 同上 | 用户反馈“f115 chip 仍小密图、f125 背景退场太快、参考 f109 角组已部分退场”。调整：chip dur 0.55→0.75s且加多 keyframe (29%/50%/65%/85%/100%) 让前段快放大为圆环后段慢淡出；底板/两角 dur 0.58→0.80s；HOLD/SLIDE 0.48-0.62s → 0.65-0.85s，起点 1.85→1.80s |
| **v25** | 同上 | 同上 | 同上 | 同上 | v24 反而背景退场太慢。重看参考视频 grid_out_v24（8 × 7）发现：背景仅 f117→f128 约 170ms 已几乎全部退场，chip 仅 f95→f120 约 420ms。重新校准：t-exit-base 1.94s、dur 0.30s；chip dur 0.45s、keyframes 0%(1)→37%(0.78)→63%(2.4)→85%(3.4,0.25)→100%(4.0,0) |
| **v26** | 同上 | 同上 | 同上 | 同上 | v25 背景 col4 (f117) 仍偏慢。提前 t-exit-base 1.94→1.87s；内收 keyframe 20%→10%、内收幅度 100/60→40/24（减少“在同一帧看上去是静止”的感觉） |
| **v27** | 同上 | 同上 | 同上 | 同上 | v26 仍偏慢。缩短 dur 0.30→0.22s；内收 keyframe 10%→15%、内收幅度 40/24→30/18；HOLD/SLIDE jitter dur 0.20-0.28s |
| **v28** | 同上 | 同上 | 同上 | 同上 | 微调最终：t-exit-base 1.87→1.83s、t-exit-bg 1.88→1.84s。与参考视频 col1–col6 (f95/f105/f112/f117/f120/f125) 几乎完全同步，仅 col4 chip 圆环偏清晰 ≈30ms |
| **v29** | 同上 | 同上 | 同上 | 同上 | 用户反馈：SLIDE 出场偏右（不是 45°）且比其他元素消失晚。修复：slide-BL-to-TR-out / slide-TR-to-BL-out 终点 (2400,-1500) → (2200,-2200) 沿 45° 对角；jitter dur 0.20-0.28 → 0.18-0.22、delay 1.84+0.04 → 1.83+0.02 |
| **v30** | 同上 | 同上 | 同上 | 同上 | v29 退场反而过快（f120 参考仍有大量背景，我方已退完）。调整：t-exit-base 1.83→1.88s、dur 0.22→0.28s；SLIDE/HOLD dur 0.18-0.22 → 0.24-0.28、delay 1.83+0.02 → 1.88+0.02。所有元素最晚于 ~2.18s 全完成，与参考 f135 全空对齐 |
| **v31-v40（实验）** | 多次反复 | 同上 | 同上 | 同上 | 一系列尝试：v31-v33 反复改 chip dur；v34-v36 误判 ref 帧 (把模糊读成大 chip) 把所有 dur 缩到 0.15-0.30s 太快；v36-v38 误以为 chip 不旋转；v39-v40 缩短入场。最终用户否定全部，要求恢复 v33 速度+旋转 |
| **v41** | calc(t0-0.02s) | t0+0+~0.05s | 1.67s | 1.88s | **关键修复**：发现 `transform-box: view-box; transform-origin: 50% 50%` 才能让 chip 围绕 SVG viewBox 中心 (1920,1080) 缩放/旋转。之前用 `translate(1920px,...) rotate scale translate(-1920px,...)` 因 CSS px ≠ SVG user units 在 viewBox 3840x2160 中导致 chip 在缩放时偏移到角落（稳态正确，缩放时错）。<br>恢复 v33 的所有时长：dur-chip 0.62s（含 -180°→0° CCW 旋转）、dur-base 0.80s、dur-purple 0.70s、dur-white 0.68s、dur-bg 0.75s（jitter 0.55-0.80s）；出场恢复 v32 时长（1.88s 起、0.28s 退场），chip-out 改为直接放大无"先缩"段 |
| **v42** | 同 v41 | 同 v41 | 1.62s | 1.88s | 用户纠正：chip 出场原本有"先回往内收一点再往外放大"，恢复 chip-out keyframes 0%(1)→30%(0.82)→50%(2.2)→70%(3.4)→85%(4.2,op0.05)→100%(4.6,op0)。出场时序对齐 ref：t-exit-chip 1.67→1.62s、dur 0.36→0.42s |
| **v43** | 同 v41 | 同 v41 | 1.62s | 同 chip（1.62s + dur 0.42s） | 用户纠正："chip 收缩时左上/右下子层也跟着收缩；chip 放大时子层+背景也跟着放大"。<br>实现：新增 `bg-out` keyframes (scale 1→0.94→1.90→2.90→3.70→4.20, op 1→1→1→0.55→0.15→0)；`#底板, #两角淡紫, #两角白色` 父组挂 bg-out（transform-box: view-box; origin 50% 50%）与 chip-out 同步起停；子组旧 expand-TL/BR、converge-TL-out/BR-out 改为保留稳态的空 keyframes（父组负责整体出场缩放）。SLIDE/HOLD 仍走原 45° 飞出（装饰条独立行为）。<br>**typo bug**：first commit 把 selector 写成 `#底板, #两角淑紫, #两角白色`（错字"淑"应为"淡"），导致 #两角淡紫 没匹配。用户发现并提示"两角淡紫为什么没跟着两角白色一起缩放"，修正后 #两角淡紫 正确同步 |
| **v44** | 同 v41 | 同 v41 | 1.62s | 收缩 1.62s / 放大+退场 1.746s | 用户精确描述："出场顺序：(1) 收缩阶段 chip + 两角淡紫 + 两角白色 一起收缩；(2) 放大阶段 chip + 两角 + 底板 一起放大；(3) 同时 HOLD/Slide 开始退场；(4) 所有东西同一时刻 2.04s 全部退出"。<br>实现：<br>• 两角淡紫/白色 用 bg-out (1.62-2.04s, 收缩+放大)<br>• #底板 独立用 **bg-out-expand**（只放大，无收缩），1.746-2.04s<br>• HOLD/Slide jitter delayOut 1.88→1.746s、durOut 0.24-0.28→0.27-0.31s，与放大阶段同步<br>• 所有元素在 2.04s 同步退完。<br>**血泪 typo 教训**：v43 写错 `两角淡紫` 为 `两角淑紫`；v44 我又重蹈覆辙，且 PowerShell `Set-Content` 不指定编码会污染中文文件（产生 `?` 字节）。最终用 Python utf-8 读写 + 从 backup 恢复修正 |
| **v45** | 同 v41 | 同 v41 | 1.62s | 同 v44 | 用户要求：HOLD/Slide 入场改为严格 45° 对角。slide-TR-to-BL 起点 `translate(2400px,-1500px)` → `translate(2400px,-2400px)`、slide-BL-to-TR 起点 `translate(-2400px,1500px)` → `translate(-2400px,2400px)`（dx:dy 1:1） |
| **v46（当前）** | 同 v41 | 同 v41 | 1.62s | 同 v44 | 用户指出：底板出场不应直接淡出，应像入场一样飞向角落。<br>**纠正 v43 过度简化**：删除 `#底板` 父组 `bg-out-expand` scale，恢复 `expand-TL/BR` 子层关键帧：左上方块 (0,0)→(-2400,-1500)、右下方块 (0,0)→(2400,1500)（与入场 settle-TL/BR 完全对称），delay 1.746s / dur 0.294s。出场时左上、右下方块分别飞回角落外，2.04s 同时退完 |

## 8. 下一步建议

v41 已对齐参考视频的速度感与旋转。当前关键参数：

| 元素 | 起点 (s) | dur (s) | 关键帧要点 |
|------|----------|---------|-----------|
| chip-in | calc(t0-0.02s)=0.15 | 0.62 | rotate -180°→0° CCW + scale 4.5→1（含 undershoot 0.91 + overshoot 1.015） |
| chip-out | 1.62 | 0.42 | scale 1→30%(0.82 微内收)→50%(2.2)→70%(3.4)→85%(4.2,op0.05)→100%(4.6,op0)，无旋转 |
| 底板 (settle) | t0=0.17 | 0.80 | translate ±2400/±1500 → 0,0 ease-out |
| 两角淡紫/白 (converge) | t0=0.17 | 0.70/0.68 | scale(1.06) 锚点 + 回弹 (±100/60 → ±25/15 → 0) |
| HOLD/SLIDE | t0+jitter=0.17~0.23 | 0.55~0.80 | 沿 45° 对角入 (±2400, ∓1500) → 0,0 |
| 出场底板/两角 (expand/converge-out) | 1.88 | 0.28 | 0,0 → 15% 内收(±30/±18) → ±2400/±1500 |
| 出场 HOLD/SLIDE | 1.88+jitter | 0.24-0.28 | 沿 45° 对角飞出 (±2200, ∓2200) opacity 1→0 |

**关键技术点（v41 修复）**：
- chip 必须用 `transform-box: view-box; transform-origin: 50% 50%;` （不是 fill-box，因 `<use>` getBBox 返回空）
- keyframes 不再需要 `translate(1920px,...) ... translate(-1920px,...)` 包裹，直接 `transform: rotate(...) scale(...)` 即可

**验证脚本注意**（重要 bug 修复）：
`CSSAnimation.currentTime` **包含 delay**（即 page-time-since-startTime）。验证时设 `a.currentTime = pageTimeMs` 即可，**不要**减去 delay。之前 v33fix 错误地减了 delay 导致截图全部失真。

下一步可选方向：

1. 端到端 2.5s mp4 对比（尚未做）
2. 微调入场 chip 起始时间（参考 f12 已有微弱 chip 出现，我方 f15 才出现）
3. chip 出场加速到 ref 节奏（mine f120 仍可见 chip 残留）

## 10. 用户偏好与工作约定（必读）

- **每次修改完代码后，必须更新 HANDOFF.md**（用户在 v8→v9 修正时明确要求）
- **对话结束时不要自己结束对话**，必须用 `vscode_askQuestions` 问下一步要做什么
- 用户用 zh-CN 沟通；回复保持简短
- 验证流程：playwright 截图（用 `document.getAnimations()` 暂停+seek currentTime）→ ffmpeg vstack 与 `frames/f_NNN.png` 对比 → `view_image` 看 cmp_finalN.png
- playwright 截图小坑：
  - `page.screenshot()` 偶尔会在第一帧 timeout（"waiting for fonts" 完成后挂起几秒）——重试即可
  - 用 `page.evaluate(()=>{for(const a of document.getAnimations()){a.pause();a.currentTime=tMs}})` 比 `animation-delay: -Xms` 可靠（后者在动画已完成的元素上不会回拨）
  - 截图前 `await page.evaluate(()=>new Promise(r=>requestAnimationFrame(()=>requestAnimationFrame(r))))` 让浏览器完成一次合成

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
