# 轻提纲 · 项目交接文档（PROJECT_HANDOFF）

> 用途：当对话上下文丢失 / 换 AI / 换设备时，凭本文档可完整接手本项目。
> 最后更新：2026-09（v3 上线时）

## 一、项目是什么

「轻提纲」是一个**提肛（凯格尔）练习助手**网页 APP（PWA），用户为 iPhone 17 单人自用。
核心玩法：**手指按住屏幕上的蜜桃 = 收紧提肛，松开 = 放松**，一组 10 次，每天目标若干组（默认 3 组）。
风格参照小红书健康类 APP「轻护」的示范截图（浅绿小清新 + 可爱蜜桃吉祥物），示范图在本文件夹：`1.jpg`、`2.jpg`、`3.jpg`。

用户明确要求：**只做提肛训练，不要加其他功能**（喝水/大便记录等已被砍掉，勿恢复）。
优化方向只接受「现有功能的体验打磨」（丝滑度、动效、反馈）。

## 二、关键地址与账号

| 项目 | 值 |
|---|---|
| 网站地址（手机访问） | https://leowang2222.github.io/qingtigang/ |
| GitHub 仓库（公开） | https://github.com/LeoWang2222/qingtigang |
| GitHub 账号 | LeoWang2222（已在本机通过 `gh auth login` 登录，gh CLI 装在 `C:\Program Files\GitHub CLI`） |
| 本地文件夹 | `C:\Users\24509\Desktop\提纲小助手`（已 git init，main 分支，remote 已配好） |
| git 提交身份 | `LeoWang2222` / `LeoWang2222@users.noreply.github.com`（用 `git -c user.name=... -c user.email=...` 提交，未写全局配置） |

## 三、文件清单

| 文件 | 作用 |
|---|---|
| `index.html` | **整个 APP 本体**：HTML+CSS+JS 全部内嵌，单文件 |
| `sw.js` | Service Worker 离线缓存。**每次改代码必须升级里面的 `CACHE` 版本号**（当前 `qingtigang-v5`），同时把 `index.html` 里「我的」页的 `app-ver` 显示号同步改掉，否则用户手机不更新 |
| `manifest.webmanifest` | PWA 配置（standalone 全屏、图标、主题色 #edf5f0） |
| `apple-touch-icon.png` (180×180) | iOS 主屏幕图标（Python/PIL 生成的蜜桃图） |
| `icon-512.png` (512×512) | manifest 图标（由 180 的放大而来，如需重绘注意） |
| `1.jpg 2.jpg 3.jpg` | 用户提供的示范截图（设计参考，勿删） |

## 四、功能现状（v4）

三个标签页（底部 tabbar）：**首页 / 打卡 / 我的**，练习页为覆盖层（无 tab 高亮）。

- **首页**：今日状态卡（进度 X/N 组，数字滚动动画）、今日训练入口（点卡片或「开始」进练习页）、教程入口（`row-guide` → 教程页）、当前计划卡（点击跳「我的」）、下一条提醒倒计时（每天 9/11/13/15/17/19/21 点整点提醒，仅页面内展示）
- **教程页**（v4 新增，`pg-guide`）：提肛运动介绍、找准肌肉方法、标准动作流程、呼吸要领、常见错误、好处、注意事项；底部「学会了，开始练一组」直达练习页
- **练习页**（核心）：
  - 按住蜜桃 → 大字「收」+ 蜜桃挤压动画（`.hold` 类换 `><` 表情）+ 绿色进度环 rAF 丝滑填满 + 每秒轻震 + 1.6s 后开始轮换鼓励语（`ENCOURAGE` 数组，淡入淡出）
  - 保持 `S.holdSec` 秒（默认 5）→ 双震+高音提示「可以松开了」；**提前松手 = 该次作废**（toast 提示）
  - 松开 → 大字「放」+ 橙色进度环休息 4 秒 → 「再次按住」
  - 满 10 次 = 1 组 → 全屏庆祝动效（蜜桃弹跳+彩带飘落+三连音）→ 返回首页
  - 进入练习页时申请 **Wake Lock 屏幕常亮**，退出释放；页面锁滚动（`body.locked`）
- **打卡页**：月日历（周一起始），**完成当日目标组数自动打勾**（绿底✓，达标瞬间有弹出动画 `justCheckedKey`）；顶部统计：连续打卡 / 累计打卡 / 累计组数；‹ › 翻月
- **我的**：今日组数/连续天数/累计打卡；设置每日目标组数（1~10）、收紧秒数（2~10）；清空数据按钮

## 五、技术要点与坑（重要）

1. **数据存储**：localStorage，键 `qingtigang_v2`。结构：
   `{day, kegelGroups, kegelTarget, holdSec, checkins:{"2026-9-5":true,...}, totalGroups, _migrated}`
   - 键名里 `dstr()` 生成的日期格式是 `年-月-日`（月日不补零），改格式会导致历史打卡丢失
   - 旧版键 `qingtigang_v1` 有一次性迁移逻辑（`S._migrated` 标记）
2. **iOS 无 Vibration API**：震动用的是 `<input type="checkbox" switch>` + label.click() 的 iOS 17.4+ 触觉 hack（见 `haptic()`），另有 WebAudio `beep()` 提示音兜底。AudioContext 需在用户手势中 resume。
3. **iOS 网页无法后台推送**：提醒只是页面内倒计时，别承诺系统级通知。
4. **iOS 主屏幕图标不支持 SVG/data URI**：必须用真实 PNG 文件。
5. **进度环**：SVG circle r=120，周长 754（`RING_LEN`），改半径要同步改。
6. **按压交互**：用 Pointer Events（`pointerdown/up/cancel`），`up` 绑在 window 上；`touch-action:none` + `contextmenu` 阻止 + `-webkit-touch-callout:none` 防长按弹菜单。
7. **部署流程**（改完代码后）：
   ```bash
   cd "/c/Users/24509/Desktop/提纲小助手"
   # 1. sw.js 里 CACHE 版本号 +1（如 v3→v4）
   # 2. 语法检查：抽出 <script> 内容用 node --check 验证
   git add -A && git -c user.name="LeoWang2222" -c user.email="LeoWang2222@users.noreply.github.com" commit -m "说明" && git push
   # 3. 验证：curl https://leowang2222.github.io/qingtigang/index.html 确认新标记出现（Pages 构建约 30~60 秒）
   ```
   - 网络偶发 SSL 握手失败（用户有代理/VPN），push 失败就重试
   - 手机端更新：APP 关闭重开一两次（SW 换新缓存）；顽固就删主屏幕图标重装
8. **自动更新机制（v5 起）**：`index.html` 头部脚本在打开时和每 60 秒调用 `reg.update()`；新 SW 接管（controllerchange）时自动 `location.reload()`。若正在练习页则设 `window.__pendingReload`，回到其他页面时再刷新。首次安装（无 controller）不刷新。「我的」页有「检查更新」按钮（`btn-update`）手动触发。
9. **GitHub Pages 免费版要求公开仓库**：用户知情并接受现状（数据在手机本地，仓库无隐私）。若用户改主意想私有化，需迁移到 Cloudflare Pages / Vercel。

## 六、环境备忘（本机）

- Windows + Git Bash（`D:\Git\bin\bash.exe`）；Python 3.14 在 `D:\python`（有 PIL，可重新生成图标）；Node v24 在 `D:\NODE`
- 本机 IP 172.20.10.x 段是 iPhone 热点；曾用 `python -m http.server 8000` 做局域网调试（现在以 GitHub Pages 为主，非必需）
- iOS 添加到主屏幕：Safari 打开网址 → 分享 → 添加到主屏幕

## 七、用户偏好

- 中文交流；喜欢先讲计划再动手；在意隐私但接受公开仓库的现状
- 命名有谐音梗偏好（「提纲」谐「提肛」）
- 决策风格：砍功能求简洁、求手感，不要功能堆砌
