# 《接住小团子》开发指南

## 📦 项目文件说明

```
catch-the-puppy/
├── prototype.html          # HTML5 可玩原型（可直接打开测试）
├── docs/
│   └── game-design.md      # 完整游戏设计文档
└── README.md               # 本文件
```

---

## 🎮 第一步：测试 HTML5 原型

**立即体验核心玩法！**

1. 在浏览器中打开 `prototype.html`
   - Mac: 右键文件 → 打开方式 → Google Chrome
   - 或用命令行：`open prototype.html`

2. 游戏玩法：
   - 点击「开始游戏」
   - 当小狗落入底部圆圈时，点击屏幕
   - 连续接住，得分越高速度越快
   - 漏接一个 → 游戏结束

3. 测试重点：
   - 手感是否流畅？
   - 难度曲线是否合理？
   - 是否需要调整速度/判定圈大小？

**如有调整建议，告诉我，我立即修改原型！**

---

## 🛠️ 第二步：安装 Cocos Creator（正式开发）

### 下载地址
- 官网：https://www.cocos.com/products
- 推荐版本：**Cocos Creator 3.8.x**（最新 LTS 版本）

### 安装步骤
1. 下载安装包（Mac 选 macOS 版）
2. 拖入 Applications 文件夹
3. 首次启动需登录 Cocos 账号（免费注册）

### 学习资源（新手必看）
- **官方入门教程**：https://docs.cocos.com/creator/3.8/manual/
- **视频教程**：B 站搜索「Cocos Creator 入门」
- **小游戏专项**：https://docs.cocos.com/creator/3.8/manual/zh/wechat-game/

**预计学习时间**：4-6 小时（有编程基础更快）

---

## 📁 第三步：创建 Cocos Creator 项目

### 1. 新建项目
1. 打开 Cocos Creator
2. 点击「新建项目」
3. 选择模板：**Empty**（空项目）
4. 项目名称：`catch-the-puppy`
5. 项目路径：选择现有文件夹或新建
6. 点击「创建」

### 2. 项目设置
1. 菜单栏 → 项目 → 项目设置
2. 基础设置：
   - 游戏名称：接住小团子
   - 包名：`com.yourname.catchpuppy`
   - 横竖屏：**竖屏**
3. 平台设置 → 微信小游戏：
   - AppID: （后续在微信后台获取）
   - 域名配置：（后续配置）

### 3. 创建目录结构
在 Cocos Creator 资源管理器中创建：
```
assets/
├── scripts/        # 脚本代码
├── textures/       # 图片资源
├── audio/          # 音频资源
├── fonts/          # 字体资源
└── scenes/         # 场景文件
```

---

## 💻 第四步：核心代码实现

### 4.1 创建 GameManager（游戏主控制器）

在 `assets/scripts/` 下创建 `GameManager.ts`：

```typescript
import { _decorator, Component, Node, instantiate, Prefab, Label, Button } from 'cc';
import { Puppy } from './Puppy';

const { ccclass, property } = _decorator;

@ccclass('GameManager')
export class GameManager extends Component {
    @property(Prefab)
    puppyPrefab: Prefab = null!;
    
    @property(Label)
    scoreLabel: Label = null!;
    
    @property(Label)
    comboLabel: Label = null!;
    
    score: number = 0;
    combo: number = 0;
    gameState: 'menu' | 'playing' | 'gameover' = 'menu';
    
    speed: number = 200; // px/s
    spawnInterval: number = 1500; // ms
    lastSpawnTime: number = 0;
    
    start() {
        this.resetGame();
    }
    
    resetGame() {
        this.score = 0;
        this.combo = 0;
        this.speed = 200;
        this.spawnInterval = 1500;
        this.updateUI();
    }
    
    startGame() {
        this.gameState = 'playing';
        this.lastSpawnTime = Date.now();
    }
    
    update(deltaTime: number) {
        if (this.gameState !== 'playing') return;
        
        // 生成小狗
        if (Date.now() - this.lastSpawnTime > this.spawnInterval) {
            this.spawnPuppy();
            this.lastSpawnTime = Date.now();
        }
    }
    
    spawnPuppy() {
        const puppy = instantiate(this.puppyPrefab);
        this.node.addChild(puppy);
    }
    
    catchPuppy(isPerfect: boolean) {
        this.score += isPerfect ? 2 : 1;
        this.combo++;
        
        if (this.combo % 10 === 0) {
            this.score += 5; // 连击奖励
        }
        
        // 增加难度
        this.speed = Math.min(this.speed + 20, 800);
        this.spawnInterval = Math.max(this.spawnInterval - 50, 400);
        
        this.updateUI();
    }
    
    gameOver() {
        this.gameState = 'gameover';
        // 保存最高分到本地
        const best = wx.getStorageSync('bestScore') || 0;
        if (this.score > best) {
            wx.setStorageSync('bestScore', this.score);
        }
    }
    
    updateUI() {
        this.scoreLabel.string = this.score.toString();
        if (this.combo >= 5) {
            this.comboLabel.string = `Combo x${this.combo}!`;
            this.comboLabel.node.active = true;
        } else {
            this.comboLabel.node.active = false;
        }
    }
}
```

### 4.2 创建 Puppy（小狗逻辑）

创建 `assets/scripts/Puppy.ts`：

```typescript
import { _decorator, Component, Node, Collider2D, ICollider2DContact } from 'cc';

const { ccclass, property } = _decorator;

@ccclass('Puppy')
export class Puppy extends Component {
    speed: number = 200;
    caught: boolean = false;
    missed: boolean = false;
    
    targetCircleY: number = -300; // 判定圈 Y 坐标
    
    start() {
        // 获取 GameManager 的速度
        const gameManager = this.node.parent?.getComponent('GameManager') as any;
        if (gameManager) {
            this.speed = gameManager.speed;
        }
    }
    
    update(deltaTime: number) {
        if (this.caught || this.missed) return;
        
        // 下落
        this.node.setPosition(
            this.node.position.x + Math.sin(this.node.position.y / 100) * 0.5,
            this.node.position.y - this.speed * deltaTime
        );
        
        // 检查是否漏掉
        if (this.node.position.y < -this.targetCircleY - 100) {
            this.missed = true;
            // 通知 GameManager 游戏结束
            const gameManager = this.node.parent?.getComponent('GameManager') as any;
            if (gameManager) {
                gameManager.gameOver();
            }
        }
    }
    
    checkCatch(clickY: number): boolean {
        if (this.caught || this.missed) return false;
        
        const distance = Math.abs(this.node.position.y - this.targetCircleY);
        if (distance < 60) { // 判定圈半径
            this.caught = true;
            const isPerfect = distance < 20;
            
            // 通知 GameManager
            const gameManager = this.node.parent?.getComponent('GameManager') as any;
            if (gameManager) {
                gameManager.catchPuppy(isPerfect);
            }
            
            // 播放捕获动画
            this.node.destroy();
            return true;
        }
        return false;
    }
}
```

---

## 🔌 第五步：微信 SDK 接入

### 5.1 配置微信小游戏

1. 在 Cocos Creator 中：
   - 菜单栏 → 项目 → 项目设置
   - 服务 → 微信小游戏 → 启用

2. 构建发布：
   - 菜单栏 → 项目 → 构建发布
   - 平台：微信小游戏
   - 点击「构建」

3. 用微信开发者工具打开构建目录

### 5.2 接入微信功能

创建 `assets/scripts/WechatSDK.ts`：

```typescript
// 微信登录
function login() {
    wx.login({
        success: (res) => {
            console.log('登录成功', res.code);
            // 发送 code 到后端换取 openid
        }
    });
}

// 分享
function shareAppMessage() {
    wx.shareAppMessage({
        title: '我接住了 32 只小团子，你能超过我吗？',
        imageUrl: ''
    });
}

// 排行榜
function getFriendRank() {
    const openDataContext = wx.getOpenDataContext();
    openDataContext.postMessage({
        type: 'getFriendRank'
    });
}

// 激励视频广告
let rewardedAd = null;

function initAd() {
    rewardedAd = wx.createRewardedVideoAd({
        adUnitId: 'adunit-xxxxx' // 在微信后台创建
    });
    
    rewardedAd.onClose(res => {
        if (res && res.isEnded) {
            // 广告观看完成，给予奖励
            giveReviveReward();
        } else {
            console.log('广告中途退出');
        }
    });
}

function showAd() {
    if (rewardedAd) {
        rewardedAd.show().catch(() => {
            // 失败重试
            rewardedAd.load()
                .then(() => rewardedAd.show())
                .catch(err => console.log('广告加载失败', err));
        });
    }
}

function giveReviveReward() {
    // 复活逻辑
    const gameManager = cc.find('Canvas/GameManager')?.getComponent('GameManager');
    if (gameManager) {
        gameManager.revive();
    }
}
```

---

## 🎨 第六步：美术资源

### 方案 A：自己制作（免费）
- **工具**：Procreate / Photoshop / Aseprite
- **风格参考**：搜索「治愈系 手绘 小狗」
- **尺寸要求**：
  - 角色：128x128 px
  - UI 按钮：300x100 px
  - 背景：750x1334 px（适配 iPhone）

### 方案 B：使用免费素材
- **itch.io**：https://itch.io/game-assets/free
- **Kenney.nl**：https://kenney.nl/assets
- **OpenGameArt**：https://opengameart.org/

### 方案 C：AI 生成（推荐快速验证）
- **Midjourney**：生成高质量插画
- **Stable Diffusion**：本地部署免费
- 提示词示例：
  ```
  cute cartoon puppy, round chubby body, 
  simple flat design, pastel colors, 
  game asset, white background --v 5
  ```

---

## 📱 第七步：测试与发布

### 本地测试
1. Cocos Creator 中点击「预览」
2. 浏览器中测试玩法
3. 调整难度曲线和手感

### 微信开发者工具测试
1. 构建微信小游戏
2. 用微信开发者工具打开
3. 真机预览（扫码）

### 提交审核
1. 登录微信小程序后台：https://mp.weixin.qq.com/
2. 创建小程序（个人主体）
3. 上传代码
4. 提交审核（1-3 个工作日）

---

## 💰 第八步：商业化配置

### 申请广告位
1. 登录微信小程序后台
2. 流量主 → 开通（需要累计 1000 用户）
3. 创建广告位 → 复制 adUnitId

### 广告点位建议
| 场景 | 频率 | 预期 eCPM |
|------|------|-----------|
| 复活 | 每日 3 次 | ¥30-50 |
| 解锁皮肤 | 一次性 | ¥20-30 |
| 双倍积分 | 每局可选 | ¥15-25 |

**注意**：初期不要放太多广告，优先保证用户体验！

---

## 🚀 快速启动清单

- [ ] 打开 `prototype.html` 测试核心玩法
- [ ] 反馈调整建议（速度/难度/手感）
- [ ] 下载并安装 Cocos Creator
- [ ] 完成官方入门教程（4-6 小时）
- [ ] 创建 Cocos Creator 项目
- [ ] 复制粘贴我提供的代码
- [ ] 制作/获取美术资源
- [ ] 构建并在微信开发者工具测试
- [ ] 申请微信小程序账号
- [ ] 提交审核上线

---

## ❓ 常见问题

### Q: 我没有编程经验，能学会吗？
A: 可以！Cocos Creator 是可视化编辑器，大部分操作是拖拽。TypeScript 代码我已经写好，你主要是学习和调整参数。

### Q: 多久能上线？
A: 
- 快速验证版（用现成素材）：3-5 天
- 完整版（定制美术）：2-3 周

### Q: 需要版号吗？
A: 个人开发者的小游戏，选择「休闲」分类，**不需要版号**。

### Q: 怎么赚钱？
A: 主要靠广告（激励视频）。1000 日活大概每天¥50-200 收入，看广告点位设计。

---

**下一步行动**：

1. **先玩 prototype.html**，告诉我手感如何
2. 我根据你的反馈调整参数
3. 确认玩法 OK 后，我帮你搭建完整的 Cocos Creator 项目

有任何问题随时问我！🚀
