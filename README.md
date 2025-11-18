# RPG Maker MV 高级钩子管理插件 - 使用说明

## 第一部分：快速入门（5分钟上手）

### 1.1 什么是钩子系统

● 钩子的概念

**钩子（Hook）** 是一种在不修改原始代码的情况下，拦截并扩展函数功能的技术。就像在函数执行的"关键点"上挂了一个"钩子"，让你可以在函数执行前后插入自己的逻辑。

```
原始函数执行流程：
开始 → 执行函数 → 结束

使用钩子后：
开始 → 你的前置逻辑 → 执行函数 → 你的后置逻辑 → 结束
```

● 为什么需要钩子？

**传统方式的问题：**

```javascript
// ❌ 直接修改引擎源码
Game_Player.prototype.moveStraight = function(d) {
  // 原始代码...
  this.setMovementSuccess(this.canPass(x, y, d));
  
  // 你的修改
  console.log('玩家移动了');
  this.checkSpecialTile(); // 你添加的功能
};
```

**问题：**
- ❌ 引擎更新后，你的修改会丢失
- ❌ 多个插件修改同一函数会冲突
- ❌ 难以维护和调试
- ❌ 无法轻易移除功能

**使用钩子的优势：**

```javascript
// ✅ 使用钩子
HookManager.regHook('Game_Player.prototype.moveStraight', function(next, d) {
  // 你的前置逻辑
  console.log('玩家即将移动');
  
  // 调用原函数
  next();
  
  // 你的后置逻辑
  console.log('玩家移动完成');
  this.checkSpecialTile();
});
```

**优势：**
- ✅ 不修改引擎源码
- ✅ 多个插件可以和平共处
- ✅ 可以随时启用/禁用
- ✅ 易于维护和调试

● 实际应用场景

| 场景 | 传统方式 | 钩子方式 |
|------|----------|----------|
| **添加移动特效** | 修改 `moveStraight` 源码 | 注册钩子，在移动后播放特效 |
| **自定义伤害计算** | 修改 `makeDamageValue` 源码 | 注册钩子，修改伤害值 |
| **记录玩家行为** | 在每个函数里加日志 | 统一注册钩子记录 |
| **插件兼容性** | 手动协调执行顺序 | 使用优先级自动管理 |

---

### 1.2 第一个钩子

让我们从最简单的例子开始：在玩家移动时打印一条日志。

● 完整代码

```javascript
// 注册钩子：监听玩家移动
HookManager.regHook('Game_Player.prototype.moveStraight', function(next, direction) {
  // 在移动前执行
  console.log('玩家准备移动，方向:', direction);
  
  // 调用原始的移动函数
  next();
  
  // 在移动后执行
  console.log('玩家移动完成，当前位置:', this.x, this.y);
});
```

● 代码解析

**1. 目标函数路径**
```javascript
'Game_Player.prototype.moveStraight'
```
- `Game_Player`：玩家类
- `prototype`：原型对象
- `moveStraight`：直线移动方法

**2. 钩子函数**
```javascript
(next, direction) => { ... }
```
- `next`：一个函数，调用它会执行原始的 `moveStraight` 函数
- `direction`：原函数的参数（移动方向：2下/4左/6右/8上）

**3. `next()` 的作用**
```javascript
next(); // 调用原始函数
```
- 如果不调用 `next()`，原始函数不会执行
- 可以在 `next()` 前后添加自己的逻辑
- 可以选择不调用 `next()` 来阻止原函数执行

**4. `this` 上下文**
```javascript
console.log(this.x, this.y); // this 指向 Game_Player 实例
```

● 运行效果

当玩家在地图上移动时，控制台会输出：

```
玩家准备移动，方向: 2
玩家移动完成，当前位置: 10 15
玩家准备移动，方向: 6
玩家移动完成，当前位置: 11 15
```

● 实验：尝试修改

**实验1：阻止向下移动**
```javascript
HookManager.regHook('Game_Player.prototype.moveStraight', function(next, direction) {
  if (direction === 2) {
    console.log('禁止向下移动！');
    return; // 不调用 next()，阻止移动
  }
  next();
});
```

**实验2：移动后播放音效**
```javascript
HookManager.regHook('Game_Player.prototype.moveStraight', function(next, direction) {
  next();
  
  // 移动后播放脚步声
  AudioManager.playSe({
    name: 'Step',
    volume: 50,
    pitch: 100,
    pan: 0
  });
});
```

---

### 1.3 基础概念

● 钩子链的执行顺序

当多个钩子注册到同一个函数时，它们会形成一个"钩子链"：

```javascript
// 插件A注册
HookManager.regHook('Game_Player.prototype.moveStraight', function(next, d) {
  console.log('A: 开始');
  next();
  console.log('A: 结束');
});

// 插件B注册
HookManager.regHook('Game_Player.prototype.moveStraight', function(next, d) {
  console.log('B: 开始');
  next();
  console.log('B: 结束');
});

// 插件C注册
HookManager.regHook('Game_Player.prototype.moveStraight', function(next, d) {
  console.log('C: 开始');
  next();
  console.log('C: 结束');
});
```

**执行顺序（后注册的先执行）：**
```
C: 开始
  B: 开始
    A: 开始
      [原始函数执行]
    A: 结束
  B: 结束
C: 结束
```

**可视化：**
```
┌─────────────────────────────────┐
│ 钩子C                           │
│  ┌───────────────────────────┐  │
│  │ 钩子B                     │  │
│  │  ┌─────────────────────┐  │  │
│  │  │ 钩子A               │  │  │
│  │  │  ┌───────────────┐  │  │  │
│  │  │  │  原始函数     │  │  │  │
│  │  │  └───────────────┘  │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

● `next()` 的调用时机

**1. Before 模式（前置处理）**
```javascript
HookManager.regHook('someFunction', function(next, arg) {
  console.log('在原函数执行前做些什么');
  doSomethingBefore();
  
  next(); // 然后执行原函数
});
```

**2. After 模式（后置处理）**
```javascript
HookManager.regHook('someFunction', function(next, arg) {
  next(); // 先执行原函数
  
  console.log('在原函数执行后做些什么');
  doSomethingAfter();
});
```

**3. Around 模式（包裹处理）**
```javascript
HookManager.regHook('someFunction', function(next, arg) {
  console.log('前置处理');
  
  const result = next(); // 执行原函数并获取返回值
  
  console.log('后置处理，原函数返回:', result);
  return result; // 返回结果
});
```

**4. Replace 模式（替换原函数）**
```javascript
HookManager.regHook('someFunction', function(next, arg) {
  // 完全不调用 next()，使用自己的实现
  console.log('使用自定义逻辑');
  return myCustomImplementation(arg);
});
```

● 如何阻止原函数执行

**场景1：条件阻止**
```javascript
//注意: 如果使用this时务必确保使用function而不是箭头函数。
HookManager.regHook('Game_Actor.prototype.useItem', function(next, item) {
  // 检查自定义条件
  if (this.isConfused() && item.damage.type > 0) {
    console.log('混乱状态下无法使用攻击道具');
    return; // 不调用 next()，阻止使用道具
  }
  next(); // 条件满足时才执行
});
```

**场景2：权限检查**
```javascript
HookManager.regHook('Game_Party.prototype.gainGold', function(next, amount) {
  // 防作弊：限制单次获得金币上限
  if (amount > 9999) {
    console.warn('单次获得金币过多，已限制为9999');
    amount = 9999;
  }
  
  next(); // 使用修改后的参数
});
```

**场景3：完全替换**
```javascript
HookManager.regHook('Game_Player.prototype.encounterProgressValue', function(next) {
  // 不调用 next()，使用自定义的遇敌计算
  return this._encounterCount > 100 ? 2 : 1;
});
```

---
## 第二部分：核心功能详解

### 2.1 注册钩子的方式

● 基础语法

```javascript
HookManager.regHook(target, hookFunc, options);
```

**参数说明：**

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| target | String | ✓ | 目标函数路径 |
| hookFunc | Function | ✓ | 钩子函数 |
| options | Object | ✗ | 配置选项 |

● 字符串路径方式

```javascript
// 基础用法
HookManager.regHook('Game_Player.prototype.update', function(next) {
  console.log('玩家更新前');
  next();
  console.log('玩家更新后');
});

// 带配置选项
HookManager.regHook('Game_Actor.prototype.changeHp', function(next, value) {
  console.log('HP变化:', value);
  next();
}, {
  priority: 100,
  label: 'MyPlugin-HPChange'
});
```

---

● 新旧版本 API 说明

**HookManager 提供了两种调用方式：**

#● 1. 新版 API（推荐）

```javascript
// 直接使用 HookManager
const unbind = HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
  this.customUpdate();
}, {
  priority: 100,
  label: 'MyPlugin-PlayerUpdate'
});

// 解绑
unbind();
```

**特点：**
- ✓ 直接访问核心管理器
- ✓ 更清晰的命名空间
- ✓ 完整的功能支持
- ✓ 返回解绑函数

#● 2. 兼容旧版 API
**通过 PluginManager 调用：**
```javascript
// 通过 PluginManager 调用（兼容层）
const unbind = PluginManager.regHook('Game_Player.prototype.update', function(original) {
  return function() {
    original.call(this);
    // 逻辑
  };
});

```



---

### 2.2 钩子函数的编写

● 函数签名

```javascript
function(next, ...args) { return result }
```

**参数说明：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `next` | Function | 调用下一个钩子或原函数 |
| `...args` | any[] | 原函数的所有参数 |
| 返回值 | any | 返回给调用者的值 |

**`this` 上下文：**
```javascript
HookManager.regHook('Game_Actor.prototype.changeHp', function(next, value) {
  console.log(this); // this 指向 Game_Actor 实例
  console.log(this.name()); // 可以访问角色名称
  next();
});
```

⚠️ **注意：** 使用箭头函数时 `this` 会绑定到外层作用域：
```javascript
// ❌ 错误：箭头函数的 this 不是 Game_Actor
HookManager.regHook('Game_Actor.prototype.changeHp', (next, value) => {
  console.log(this.name()); // 错误！this 不是角色实例
});

// ✅ 正确：使用普通函数
HookManager.regHook('Game_Actor.prototype.changeHp', function(next, value) {
  console.log(this.name()); // 正确
  next();
});
```

● 返回值处理

**场景1：不需要返回值**
```javascript
HookManager.regHook('Game_Player.prototype.moveStraight', function(next, d) {
  next(); // moveStraight 没有返回值
});
```

**场景2：传递返回值**
```javascript
HookManager.regHook('Game_Actor.prototype.attackSkillId', function(next) {
  const skillId = next(); // 获取原函数返回的技能ID
  console.log('攻击技能ID:', skillId);
  return skillId; // 必须返回
});
```

**场景3：修改返回值**
```javascript
HookManager.regHook('Game_Action.prototype.makeDamageValue', function(next, target, critical) {
  let damage = next(); // 获取原始伤害
  
  // 暴击伤害翻倍
  if (critical) {
    damage *= 2;
  }
  
  return damage; // 返回修改后的伤害
});
```

**场景4：替换返回值**
```javascript
HookManager.regHook('Game_Player.prototype.encounterProgressValue', function(next) {
  // 不调用 next()，直接返回自定义值
  return this._encounterCount > 100 ? 2 : 1;
});
```

● 常见模式

**Before 模式（前置处理）**
```javascript
HookManager.regHook('Game_Actor.prototype.changeHp', function(next, value) {
  // 在HP变化前记录
  const oldHp = this.hp;
  
  next();
  
  // 可以在这里对比变化
  console.log(`HP: ${oldHp} -> ${this.hp}`);
});
```

**After 模式（后置处理）**
```javascript
HookManager.regHook('Game_Player.prototype.moveStraight', function(next, d) {
  next(); // 先移动
  
  // 移动后检查特殊地形
  if (this.isOnDamageTile()) {
    this.takeDamage(10);
  }
});
```

**Around 模式（包裹处理）**
```javascript
HookManager.regHook('Scene_Battle.prototype.update', function(next) {
  const startTime = performance.now();
  
  next(); // 执行原更新逻辑
  
  const duration = performance.now() - startTime;
  if (duration > 16.67) {
    console.warn('战斗场景更新过慢:', duration);
  }
});
```

**Replace 模式（替换原函数）**
```javascript
HookManager.regHook('Game_Actor.prototype.paramBase', function(next, paramId) {
  // 完全自定义属性计算，不调用原函数
  if (paramId === 2) { // 攻击力
    return this.level * 5 + 10;
  }
  return next(); // 其他属性使用原逻辑
});
```

**Guard 模式（条件守卫）**
```javascript
HookManager.regHook('Game_Party.prototype.gainGold', function(next, amount) {
  // 验证参数
  if (typeof amount !== 'number' || amount < 0) {
    console.error('无效的金币数量:', amount);
    return; // 阻止执行
  }
  
  // 防作弊检查
  if (amount > 999999) {
    console.warn('金币数量异常，已限制');
    amount = 999999;
  }
  
  next();
});
```

**Transform 模式（参数转换）**
```javascript
HookManager.regHook('Game_Actor.prototype.changeExp', function(next, exp, show) {
  // 经验值加成
  const bonusRate = this.hasExpBoost() ? 1.5 : 1.0;
  exp = Math.floor(exp * bonusRate);
  
  console.log(`经验值加成: ${bonusRate}x`);
  next(); // 使用修改后的参数
});
```

---

### 2.3 解绑钩子

● 为什么需要解绑

**问题1：内存泄漏**
```javascript
// ❌ 错误：临时钩子未解绑
function setupTemporaryEffect() {
  HookManager.regHook('Game_Player.prototype.update', function(next) {
    // 临时特效逻辑
    next();
  });
  // 特效结束后钩子仍在内存中！
}

// 调用100次后，有100个无用钩子在内存中
for (let i = 0; i < 100; i++) {
  setupTemporaryEffect();
}
```

**问题2：逻辑错误**
```javascript
// ❌ 错误：事件完成后仍在监听
function waitForPlayerMove() {
  HookManager.regHook('Game_Player.prototype.moveStraight', function(next, d) {
    next();
    console.log('玩家移动了');
    // 应该只监听一次，但会永远监听！
  });
}
```

● 使用解绑函数

**基础用法：**
```javascript
// 注册时保存解绑函数
const unbind = HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
});

// 需要时调用解绑
unbind();
```

**示例1：一次性钩子**
```javascript
function waitForNextMove(callback) {
  const unbind = HookManager.regHook('Game_Player.prototype.moveStraight', function(next, d) {
    next();
    
    callback(d); // 执行回调
    unbind(); // 立即解绑
  });
}

// 使用
waitForNextMove((direction) => {
  console.log('玩家移动了，方向:', direction);
});
```

**示例2：条件解绑**
```javascript
function trackMovementUntilPosition(targetX, targetY) {
  const unbind = HookManager.regHook('Game_Player.prototype.moveStraight', function(next, d) {
    next();
    
    if (this.x === targetX && this.y === targetY) {
      console.log('到达目标位置！');
      unbind(); // 到达后解绑
    }
  });
  
  return unbind; // 返回解绑函数，允许外部提前取消
}

// 使用
const cancel = trackMovementUntilPosition(10, 15);

// 如果需要提前取消
// cancel();
```

**示例3：插件生命周期管理**
```javascript
class MyPlugin {
  constructor() {
    this.hooks = [];
  }
  
  enable() {
    // 注册所有钩子并保存解绑函数
    this.hooks.push(
      HookManager.regHook('Game_Player.prototype.update', this.onPlayerUpdate.bind(this))
    );
    
    this.hooks.push(
      HookManager.regHook('Scene_Map.prototype.update', this.onMapUpdate.bind(this))
    );
  }
  
  disable() {
    // 解绑所有钩子
    this.hooks.forEach(unbind => unbind());
    this.hooks = [];
  }
  
  onPlayerUpdate(next) {
    next();
    // 插件逻辑
  }
  
  onMapUpdate(next) {
    next();
    // 插件逻辑
  }
}

// 使用
const plugin = new MyPlugin();
plugin.enable();

// 禁用插件时
plugin.disable();
```

● 内存泄漏风险

**风险场景：**
```javascript
// ❌ 危险：在循环中注册钩子
setInterval(() => {
  HookManager.regHook('update', function(next) {
    next();
  });
}, 1000);
// 每秒注册一个新钩子，永不解绑！
```

**正确做法：**
```javascript
// ✅ 正确：复用同一个钩子
let currentUnbind = null;

function setupHook() {
  // 先解绑旧钩子
  if (currentUnbind) {
    currentUnbind();
  }
  
  // 注册新钩子
  currentUnbind = HookManager.regHook('update', function(next) {
    next();
  });
}
```

● 最佳实践

**1. 始终保存解绑函数**
```javascript
// ✅ 好习惯
const unbind = HookManager.regHook(...);

// ❌ 坏习惯
HookManager.regHook(...); // 无法解绑
```

**2. 在合适的时机解绑**
```javascript
// 场景切换时
Scene_Map.prototype.terminate = function() {
  Scene_Base.prototype.terminate.call(this);
  
  // 解绑地图相关钩子
  this._mapHooks.forEach(unbind => unbind());
};
```

**3. 使用数组管理多个钩子**
```javascript
class FeatureManager {
  constructor() {
    this.unbinders = [];
  }
  
  registerHooks() {
    this.unbinders = [
      HookManager.regHook('hook1', ...),
      HookManager.regHook('hook2', ...),
      HookManager.regHook('hook3', ...)
    ];
  }
  
  cleanup() {
    this.unbinders.forEach(unbind => unbind());
    this.unbinders = [];
  }
}
```

**4. 避免在钩子内部注册新钩子**
```javascript
// ❌ 危险：递归注册
HookManager.regHook('update', function(next) {
  next();
  
  // 每次update都注册新钩子！
  HookManager.regHook('anotherHook', ...);
});

// ✅ 正确：在外部注册
HookManager.regHook('update', function(next) {
  next();
});

HookManager.regHook('anotherHook', function(next) {
  next();
});
```
### 2.5 钩子管理方法

● setHookEnabled - 启用/禁用钩子

动态控制钩子的启用状态，而无需解绑。

```javascript
// 注册钩子时保存 hookId
let myHookId;
const unbind = HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
  console.log('玩家更新');
}, {
  priority: 100,
  label: 'MyPlugin-PlayerUpdate'
});

// 获取 hookId（需要从内部访问）
// 或者通过 HookManager.hooks 查找

// 禁用钩子
HookManager.setHookEnabled('Game_Player.prototype.update', myHookId, false);

// 重新启用
HookManager.setHookEnabled('Game_Player.prototype.update', myHookId, true);
```

**参数：**
- `key` (string) - 钩子键（如 'Game_Player.prototype.update'）
- `hookId` (Symbol) - 钩子的唯一标识符
- `enabled` (boolean) - 是否启用

**返回值：**
- `boolean` - 操作是否成功

**使用场景：**
- 临时禁用某个功能
- 根据游戏状态动态控制钩子
- 调试时快速开关功能

---

● getStats - 获取性能统计

获取指定钩子的性能统计信息。

```javascript
// 启用统计
HookManager.globalOptions.enableStats = true;

// 注册钩子
const unbind = HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
  // 一些耗时操作
}, {
  label: 'MyPlugin-PlayerUpdate'
});

// 获取统计信息（需要 hookId）
const stats = HookManager.getStats(hookId);

if (stats) {
  console.log('调用次数:', stats.callCount);
  console.log('平均耗时:', stats.avgTime.toFixed(2), 'ms');
  console.log('最小耗时:', stats.minTime.toFixed(2), 'ms');
  console.log('最大耗时:', stats.maxTime.toFixed(2), 'ms');
  console.log('总耗时:', stats.totalTime.toFixed(2), 'ms');
  console.log('错误次数:', stats.errors);
}
```

**参数：**
- `hookId` (Symbol) - 钩子的唯一标识符

**返回值：**
- `Object|null` - 统计信息对象，包含：
  - `callCount` - 调用次数
  - `totalTime` - 总耗时（毫秒）
  - `maxTime` - 最大耗时（毫秒）
  - `minTime` - 最小耗时（毫秒）
  - `avgTime` - 平均耗时（毫秒）
  - `errors` - 错误次数

**注意：**
- 需要先启用 `globalOptions.enableStats`
- 返回 `null` 表示未找到对应钩子

---

● printStats - 打印所有统计信息

在控制台打印所有钩子的性能统计信息。

```javascript
// 启用统计
HookManager.globalOptions.enableStats = true;

// 注册多个钩子
HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
}, { label: 'PlayerUpdate' });

HookManager.regHook('Scene_Map.prototype.update', function(next) {
  next();
}, { label: 'MapUpdate' });

// 运行一段时间后，打印统计
HookManager.printStats();
```

**输出示例：**
```
=== Hook Performance Statistics ===

Game_Player.prototype.update:
  PlayerUpdate:
    Calls: 1523
    Avg: 0.15ms
    Min: 0.08ms
    Max: 2.34ms
    Total: 228.45ms
    Errors: 0

Scene_Map.prototype.update:
  MapUpdate:
    Calls: 1523
    Avg: 0.42ms
    Min: 0.21ms
    Max: 5.67ms
    Total: 639.66ms
    Errors: 0
```

**使用场景：**
- 性能分析
- 找出性能瓶颈
- 优化钩子执行效率

---

● clearAll - 清除所有钩子

移除所有已注册的钩子，恢复原始函数。

```javascript
// 注册多个钩子
HookManager.regHook('Game_Player.prototype.update', hookFunc1);
HookManager.regHook('Scene_Map.prototype.update', hookFunc2);
HookManager.regHook('Game_Actor.prototype.setup', hookFunc3);

// 清除所有钩子
HookManager.clearAll();

// 此时所有函数都恢复为原始状态
```

**功能：**
- 移除所有钩子链
- 恢复所有被代理的函数为原始函数
- 清空统计数据
- 清空条件缓存

**使用场景：**
- 插件卸载时清理
- 重置游戏状态
- 调试时快速清除所有钩子

**注意事项：**
- ⚠️ 此操作不可逆
- ⚠️ 会影响所有使用 HookManager 的插件
- ⚠️ 建议只在必要时使用

---

### 完整示例：钩子生命周期管理

```javascript
class MyPlugin {
  static initialize() {
    // 启用统计
    HookManager.globalOptions.enableStats = true;
    
    // 保存钩子信息
    this.hooks = new Map();
    
    // 注册钩子
    const unbind = HookManager.regHook('Game_Player.prototype.update', function(next) {
      next();
      this.customUpdate();
    }, {
      priority: 100,
      label: 'MyPlugin-PlayerUpdate'
    });
    
    // 保存解绑函数和 hookId（实际使用中需要从内部获取）
    this.hooks.set('playerUpdate', { unbind });
  }
  
  static toggleFeature(enabled) {
    // 动态启用/禁用功能
    const hook = this.hooks.get('playerUpdate');
    if (hook) {
      HookManager.setHookEnabled('Game_Player.prototype.update', hook.id, enabled);
    }
  }
  
  static showPerformance() {
    // 显示性能统计
    HookManager.printStats();
  }
  
  static terminate() {
    // 清理所有钩子
    this.hooks.forEach(hook => hook.unbind());
    this.hooks.clear();
  }
}

// 使用
MyPlugin.initialize();

// 临时禁用功能
MyPlugin.toggleFeature(false);

// 查看性能
MyPlugin.showPerformance();

// 卸载插件
MyPlugin.terminate();
```
## 第三部分：高级参数配置

### 3.1 优先级（priority）

● 概念说明

**什么是优先级？**

优先级决定了多个钩子的执行顺序。数字越大，优先级越高，越先执行。

```javascript
// 优先级 200 - 最先执行
HookManager.regHook('update', hook1, { priority: 200 });

// 优先级 100 - 第二执行
HookManager.regHook('update', hook2, { priority: 100 });

// 优先级 50（默认） - 最后执行
HookManager.regHook('update', hook3);
```

**执行顺序规则：**
- 按优先级**降序**执行（200 → 100 → 50）
- 优先级相同时，按注册顺序执行
- 默认优先级为 50

**可视化：**
```
优先级 200 ┐
           ├→ 优先级 100 ┐
           │             ├→ 优先级 50 ┐
           │             │            ├→ 原始函数
           │             │            ┘
           │             ┘
           ┘
```
● 使用场景

**场景1：数据验证 → 数据修改 → UI更新**

```javascript
// 优先级 200：先验证
HookManager.regHook('Game_Actor.prototype.changeHp', function(next, value) {
  if (this.isDead()) {
    console.log('角色已死亡，无法恢复HP');
    return; // 阻止执行
  }
  next();
}, { priority: 200 });

// 优先级 100：再修改数据
HookManager.regHook('Game_Actor.prototype.changeHp', function(next, value) {
  value = Math.floor(value * 1.5); // HP恢复+50%
  next();
}, { priority: 100 });

// 优先级 50：最后刷新UI
HookManager.regHook('Game_Actor.prototype.changeHp', function(next, value) {
  next();
  this.refreshStatusWindow();
}, { priority: 50 });
```

**场景2：插件依赖关系**

```javascript
// 基础战斗系统（优先级 100）
HookManager.regHook('Game_Action.prototype.apply', function(next, target) {
  this.setupBaseCombat();
  next();
}, { priority: 100, label: '基础战斗系统' });

// 技能特效系统（优先级 50，依赖基础系统）
HookManager.regHook('Game_Action.prototype.apply', function(next, target) {
  next();
  this.applySkillEffects(); // 必须在基础系统之后
}, { priority: 50, label: '技能特效' });
```

● 常见优先级建议

| 层级 | 优先级范围 | 用途 | 示例 |
|------|-----------|------|------|
| 验证层 | 150-200 | 参数验证、权限检查 | 防作弊、条件检查 |
| 业务层 | 50-100 | 核心逻辑修改 | 伤害计算、属性加成 |
| 展示层 | 0-50 | UI更新、特效播放 | 刷新窗口、播放动画 |

### 3.2 条件执行（condition）

● 概念说明

**什么是条件函数？**

条件函数决定钩子是否执行。返回 `true` 时执行，`false` 时跳过。

```javascript
HookManager.regHook('update', function(next) {
  next();
  // 钩子逻辑
}, {
  condition: () => $gameParty.inBattle() // 只在战斗中执行
});
```

**条件缓存机制：**

为了性能优化，条件结果会缓存 100ms（可配置）：

```javascript
// 条件函数不会每帧都调用
HookManager.globalOptions.conditionCacheTime = 100; // 默认100ms
```

● 使用场景

**场景1：只在战斗中生效**

```javascript
HookManager.regHook('Game_Actor.prototype.performAction', function(next, action) {
  console.log('战斗特效增强');
  next();
}, {
  condition: () => $gameParty.inBattle(),
  label: '战斗特效'
});
```

**场景2：只对特定角色生效**

```javascript
HookManager.regHook('Game_Actor.prototype.attackSkillId', function(next) {
  if (this.actorId() === 1) {
    return 10; // 主角使用特殊攻击
  }
  return next();
}, {
  condition: function() {
    return this.actorId() === 1; // 只对1号角色生效
  }
});
```

**场景3：基于游戏进度**

```javascript
HookManager.regHook('Game_Player.prototype.moveStraight', function(next, d) {
  this._moveSpeed += 1; // 移动加速
  next();
}, {
  condition: () => $gameVariables.value(10) > 50, // 变量10>50时生效
  label: '后期移动加速'
});
```

**场景4：多条件组合**

```javascript
HookManager.regHook('someFunction', function(next) {
  next();
}, {
  condition: function() {
    return $gameSwitches.value(5) &&  // 开关5开启
           !$gameParty.inBattle() &&   // 不在战斗中
           this.level >= 10;            // 等级>=10
  }
});
```

● 性能优化

**条件 vs 解绑的选择：**

```javascript
// ❌ 不推荐：频繁变化的条件
HookManager.regHook('update', hook, {
  condition: () => Math.random() > 0.5 // 每次都不同
});

// ✅ 推荐：稳定的条件
HookManager.regHook('update', hook, {
  condition: () => $gameSwitches.value(10) // 开关状态稳定
});

// ✅ 推荐：条件永久改变时使用解绑
const unbind = HookManager.regHook('update', hook);
if (conditionChanged) {
  unbind(); // 直接解绑，不再检查条件
}
```

**调整缓存时间：**

```javascript
// 条件变化频繁：缩短缓存时间
HookManager.globalOptions.conditionCacheTime = 50; // 50ms

// 条件变化缓慢：延长缓存时间
HookManager.globalOptions.conditionCacheTime = 500; // 500ms
```
### 3.3 性能监控（profile）

● 基础用法

**开启单个钩子的监控：**

```javascript
HookManager.regHook('Scene_Map.prototype.update', function(next) {
  next();
}, {
  profile: true,
  label: '地图更新'
});
```

**控制台输出：**
```
✓ [Profile] 地图更新: 8.32ms
✓ [Profile] 地图更新: 9.15ms
⚠️ [Profile] 地图更新: 18.45ms  // 超过阈值
```

● 阈值（threshold）设置

**默认阈值：16.67ms（60 FPS）**

```javascript
HookManager.regHook('expensiveFunction', function(next) {
  next();
}, {
  profile: true,
  threshold: 16.67, // 超过1帧时间就警告
  label: '昂贵的函数'
});
```

**自定义阈值：**

```javascript
// 严格模式：8.33ms（120 FPS）
{ profile: true, threshold: 8.33 }

// 宽松模式：33.33ms（30 FPS）
{ profile: true, threshold: 33.33 }

// 关键路径：5ms
{ profile: true, threshold: 5 }
```

● 慢函数回调（onSlow）

**基础用法：**

```javascript
HookManager.regHook('criticalFunction', function(next) {
  next();
}, {
  profile: true,
  threshold: 10,
  onSlow: (duration) => {
    console.error(`性能警告：函数耗时 ${duration}ms`);
  }
});
```

**实际应用：发送遥测数据**

```javascript
HookManager.regHook('Game_Map.prototype.update', function(next) {
  next();
}, {
  profile: true,
  threshold: 16.67,
  onSlow: (duration) => {
    // 发送性能数据到服务器
    Analytics.send('performance_warning', {
      function: 'Game_Map.update',
      duration: duration,
      timestamp: Date.now()
    });
    
    // 记录到本地日志
    PerformanceLogger.log({
      type: 'slow_function',
      duration: duration
    });
  }
});
```

**降级处理：**

```javascript
let slowCallCount = 0;

HookManager.regHook('heavyEffect', function(next) {
  next();
}, {
  profile: true,
  onSlow: (duration) => {
    slowCallCount++;
    
    if (slowCallCount > 10) {
      console.warn('特效过慢，已自动禁用');
      unbind(); // 自动解绑
    }
  }
});
```

● 标签（label）

**为钩子命名：**

```javascript
HookManager.regHook('update', hook1, {
  label: '粒子系统更新'
});

HookManager.regHook('update', hook2, {
  label: '天气系统更新'
});

HookManager.regHook('update', hook3, {
  label: 'UI动画更新'
});
```

**输出效果：**
```
✓ [Profile] 粒子系统更新: 2.34ms
✓ [Profile] 天气系统更新: 1.12ms
✓ [Profile] UI动画更新: 0.87ms
```

**调试时的价值：**

```javascript
// 没有标签
⚠️ [Profile] Hook-0: 25.67ms  // 不知道是哪个钩子

// 有标签
⚠️ [Profile] 碰撞检测系统: 25.67ms  // 一目了然
```

### 3.4 批量注册（regBatchHooks）

● 语法和结构

**基础语法：**

```javascript
HookManager.regBatchHooks({
  '目标路径1': {
    hook: 钩子函数,
    priority: 优先级,
    condition: 条件函数,
    // ... 其他选项
  },
  '目标路径2': {
    hook: 钩子函数,
    // ...
  }
});
```

**返回值：**

```javascript
const unbinders = HookManager.regBatchHooks({...});
// unbinders 是解绑函数数组
// unbinders[0]() - 解绑第一个钩子
// unbinders[1]() - 解绑第二个钩子
```

● 配置对象格式

**完整示例：**

```javascript
const hookConfig = {
  'Game_Player.prototype.moveStraight': {
    hook: function(next, d) {
      console.log('玩家移动');
      next();
    },
    priority: 100,
    label: '移动日志'
  },
  
  'Game_Actor.prototype.changeHp': {
    hook: function(next, value) {
      next();
      this.checkLowHp();
    },
    priority: 50,
    condition: function() {
      return this.hp < this.mhp * 0.3;
    },
    label: 'HP警告'
  },
  
  'Scene_Map.prototype.update': {
    hook: function(next) {
      next();
    },
    profile: true,
    threshold: 16.67,
    label: '地图性能监控'
  }
};

const unbinders = HookManager.regBatchHooks(hookConfig);
```

● 适用场景

**场景1：插件初始化**

```javascript
class MyPlugin {
  constructor() {
    this.unbinders = [];
  }
  
  initialize() {
    this.unbinders = HookManager.regBatchHooks({
      'Game_Player.prototype.update': {
        hook: this.onPlayerUpdate.bind(this),
        label: 'MyPlugin - 玩家更新'
      },
      
      'Scene_Map.prototype.update': {
        hook: this.onMapUpdate.bind(this),
        label: 'MyPlugin - 地图更新'
      },
      
      'Game_Action.prototype.apply': {
        hook: this.onActionApply.bind(this),
        priority: 100,
        label: 'MyPlugin - 技能应用'
      }
    });
  }
  
  terminate() {
    this.unbinders.forEach(unbind => unbind());
    this.unbinders = [];
  }
  
  onPlayerUpdate(next) { next(); }
  onMapUpdate(next) { next(); }
  onActionApply(next, target) { next(); }
}
```

**场景2：配置文件驱动**

```javascript
// config.json
const pluginConfig = {
  hooks: {
    'Game_Player.prototype.moveStraight': {
      hook: 'onPlayerMove',
      priority: 100,
      enabled: true
    },
    'Game_Actor.prototype.changeHp': {
      hook: 'onHpChange',
      priority: 50,
      enabled: false  // 可以禁用
    }
  }
};

// 加载配置
function loadHooksFromConfig(config) {
  const hookMap = {};
  
  for (const [path, settings] of Object.entries(config.hooks)) {
    if (!settings.enabled) continue;
    
    hookMap[path] = {
      hook: window[settings.hook], // 从全局获取函数
      priority: settings.priority
    };
  }
  
  return HookManager.regBatchHooks(hookMap);
}
```

**场景3：功能模块化**

```javascript
// 战斗模块
const battleHooks = {
  'Game_Action.prototype.apply': {
    hook: applyBattleLogic,
    priority: 100
  },
  'Game_Battler.prototype.regenerateAll': {
    hook: customRegeneration,
    priority: 50
  }
};

// UI模块
const uiHooks = {
  'Window_Base.prototype.drawText': {
    hook: enhanceTextDrawing,
    priority: 50
  },
  'Scene_Menu.prototype.create': {
    hook: customizeMenu,
    priority: 100
  }
};

// 分别注册
const battleUnbinders = HookManager.regBatchHooks(battleHooks);
const uiUnbinders = HookManager.regBatchHooks(uiHooks);

// 可以单独禁用某个模块
function disableBattleModule() {
  battleUnbinders.forEach(unbind => unbind());
}
```

● 完整示例

**技能增强系统：**

```javascript
const SkillEnhancementSystem = {
  unbinders: [],
  
  enable() {
    this.unbinders = HookManager.regBatchHooks({
      // 技能释放前验证
      'Game_Action.prototype.apply': {
        hook: function(next, target) {
          const skill = this.item();
          
          if (skill.meta.requiresFullHP && this.subject().hpRate() < 1.0) {
            console.log('需要满HP才能使用');
            return;
          }
          
          next();
        },
        priority: 200,
        condition: function() {
          return this.isSkill();
        },
        label: '技能条件检查'
      },
      
      // 伤害计算增强
      'Game_Action.prototype.makeDamageValue': {
        hook: function(next, target, critical) {
          let damage = next();
          
          // 连击加成
          if (this.subject()._comboCount > 0) {
            damage *= (1 + this.subject()._comboCount * 0.1);
          }
          
          return damage;
        },
        priority: 100,
        label: '连击伤害加成'
      },
      
      // 技能特效
      'Game_Action.prototype.apply': {
        hook: function(next, target) {
          next();
          
          const skill = this.item();
          if (skill.meta.specialEffect) {
            this.applySpecialEffect(target);
          }
        },
        priority: 50,
        label: '特殊技能特效'
      }
    });
    
    console.log('技能增强系统已启用');
  },
  
  disable() {
    this.unbinders.forEach(unbind => unbind());
    this.unbinders = [];
    console.log('技能增强系统已禁用');
  }
};

// 使用
SkillEnhancementSystem.enable();
```
## 第四部分：全局配置

### 4.1 全局选项说明

● 访问全局配置

```javascript
// 访问全局配置对象
HookManager.globalOptions

// 查看当前配置
console.log(HookManager.globalOptions);
/* 输出：
{
  enableProfiling: false,
  enableStats: false,
  defaultPriority: 50,
  conditionCacheTime: 100
}
*/
```

● enableProfiling - 全局性能监控开关

**作用：** 控制所有钩子的性能监控功能

```javascript
// 开启全局性能监控
HookManager.globalOptions.enableProfiling = true;

// 此后注册的钩子默认开启性能监控
HookManager.regHook('update', function(next) {
  next();
}, {
  label: '自动监控'
  // profile 默认为 true（继承全局设置）
});
```

**使用场景：**

```javascript
// 开发模式：开启监控
if (Utils.isOptionValid('test')) {
  HookManager.globalOptions.enableProfiling = true;
}

// 生产模式：关闭监控
if (!Utils.isOptionValid('test')) {
  HookManager.globalOptions.enableProfiling = false;
}
```

**单独控制：**

```javascript
// 全局开启
HookManager.globalOptions.enableProfiling = true;

// 但某个钩子不需要监控
HookManager.regHook('frequentFunction', function(next) {
  next();
}, {
  profile: false  // 显式关闭
});
```

● enableStats - 统计信息收集开关

**作用：** 控制是否收集钩子的统计数据

```javascript
// 开启统计收集
HookManager.globalOptions.enableStats = true;

// 运行一段时间后查看统计
setTimeout(() => {
  HookManager.printStats();
}, 60000);
```

**统计数据包含：**
- 调用次数（callCount）
- 总耗时（totalTime）
- 平均耗时（avgTime）
- 最大耗时（maxTime）
- 最小耗时（minTime）
- 错误次数（errors）

**性能影响：**

```javascript
// ⚠️ 注意：统计功能有性能开销（约5-10%）
HookManager.globalOptions.enableStats = true;  // 开发时使用
HookManager.globalOptions.enableStats = false; // 生产时关闭
```

● defaultPriority - 默认优先级

**作用：** 设置钩子的默认优先级

```javascript
// 修改默认优先级
HookManager.globalOptions.defaultPriority = 100;

// 此后注册的钩子默认优先级为100
HookManager.regHook('update', function(next) {
  next();
});
// 等同于 { priority: 100 }
```

**使用场景：**

```javascript
// 插件A：希望自己的钩子优先级较高
HookManager.globalOptions.defaultPriority = 100;
HookManager.regHook('someFunction', hookA);

// 插件B：希望自己的钩子优先级较低
HookManager.globalOptions.defaultPriority = 30;
HookManager.regHook('someFunction', hookB);

// 执行顺序：hookA(100) → hookB(30)
```

● conditionCacheTime - 条件缓存时间

**作用：** 设置条件函数结果的缓存时间（毫秒）

```javascript
// 默认缓存100ms
HookManager.globalOptions.conditionCacheTime = 100;

// 条件变化频繁：缩短缓存
HookManager.globalOptions.conditionCacheTime = 50;

// 条件变化缓慢：延长缓存
HookManager.globalOptions.conditionCacheTime = 500;
```

**性能权衡：**

```javascript
// 缓存时间短 = 更准确，但性能开销大
HookManager.globalOptions.conditionCacheTime = 10;

// 缓存时间长 = 性能好，但可能不及时
HookManager.globalOptions.conditionCacheTime = 1000;

// 推荐值：100-200ms
HookManager.globalOptions.conditionCacheTime = 150;
```

---

### 4.2 配置最佳实践

● 开发环境配置

```javascript
// 开发模式检测
const isDevelopment = Utils.isOptionValid('test') || Utils.isNwjs();

if (isDevelopment) {
  // 开启所有调试功能
  HookManager.globalOptions.enableProfiling = true;
  HookManager.globalOptions.enableStats = true;
  HookManager.globalOptions.conditionCacheTime = 50; // 更及时的条件检查
  
  console.log('钩子系统：开发模式已启用');
  
  // 定期输出统计
  setInterval(() => {
    HookManager.printStats();
  }, 30000); // 每30秒
}
```

● 生产环境配置

```javascript
// 生产模式
const isProduction = !Utils.isOptionValid('test');

if (isProduction) {
  // 关闭调试功能，优化性能
  HookManager.globalOptions.enableProfiling = false;
  HookManager.globalOptions.enableStats = false;
  HookManager.globalOptions.conditionCacheTime = 200; // 延长缓存
  
  // 只监控关键路径
  HookManager.regHook('Scene_Map.prototype.update', function(next) {
    next();
  }, {
    profile: true,  // 显式开启
    threshold: 33.33,
    onSlow: (duration) => {
      // 发送遥测数据
      sendTelemetry('slow_update', duration);
    }
  });
}
```

● 性能测试配置

```javascript
// 性能测试模式
function enablePerformanceTesting() {
  HookManager.globalOptions.enableProfiling = true;
  HookManager.globalOptions.enableStats = true;
  HookManager.globalOptions.conditionCacheTime = 100;
  
  console.log('=== 性能测试开始 ===');
  
  // 运行测试
  runGameForDuration(60000); // 运行1分钟
  
  // 输出报告
  setTimeout(() => {
    console.log('=== 性能测试结果 ===');
    HookManager.printStats();
    
    // 分析慢钩子
    analyzeSlowHooks();
  }, 61000);
}

function analyzeSlowHooks() {
  HookManager.hooks.forEach((hookData, key) => {
    hookData.chain.forEach(hook => {
      const stats = HookManager.getStats(hook.id);
      if (stats && stats.avgTime > 5) {
        console.warn(`⚠️ 慢钩子: ${hook.label}`);
        console.warn(`   平均: ${stats.avgTime.toFixed(2)}ms`);
        console.warn(`   最大: ${stats.maxTime.toFixed(2)}ms`);
      }
    });
  });
}
```

● 动态配置切换

```javascript
// 根据设备性能动态调整
function adjustConfigByPerformance() {
  const fps = Graphics._fpsMeter?.fps || 60;
  
  if (fps < 30) {
    // 低性能设备：关闭所有调试功能
    HookManager.globalOptions.enableProfiling = false;
    HookManager.globalOptions.enableStats = false;
    HookManager.globalOptions.conditionCacheTime = 500;
    console.log('检测到低性能，已优化配置');
  } else if (fps < 50) {
    // 中等性能：只保留关键监控
    HookManager.globalOptions.enableProfiling = false;
    HookManager.globalOptions.enableStats = false;
    HookManager.globalOptions.conditionCacheTime = 200;
  } else {
    // 高性能：可以开启调试
    HookManager.globalOptions.enableProfiling = true;
    HookManager.globalOptions.enableStats = true;
    HookManager.globalOptions.conditionCacheTime = 100;
  }
}

// 游戏启动后检测
Scene_Boot.prototype.start = function() {
  Scene_Base.prototype.start.call(this);
  adjustConfigByPerformance();
};
```

● 配置模板

```javascript
// 预设配置模板
const ConfigPresets = {
  // 开发模式
  development: {
    enableProfiling: true,
    enableStats: true,
    defaultPriority: 50,
    conditionCacheTime: 50
  },
  
  // 生产模式
  production: {
    enableProfiling: false,
    enableStats: false,
    defaultPriority: 50,
    conditionCacheTime: 200
  },
  
  // 性能测试
  performance: {
    enableProfiling: true,
    enableStats: true,
    defaultPriority: 50,
    conditionCacheTime: 100
  },
  
  // 低性能设备
  lowEnd: {
    enableProfiling: false,
    enableStats: false,
    defaultPriority: 50,
    conditionCacheTime: 500
  }
};

// 应用配置
function applyConfig(presetName) {
  const preset = ConfigPresets[presetName];
  if (!preset) {
    console.error('未知的配置预设:', presetName);
    return;
  }
  
  Object.assign(HookManager.globalOptions, preset);
  console.log(`已应用配置: ${presetName}`);
}

// 使用
applyConfig('development');
```
## 第五部分：调试和诊断

### 5.1 基础调试

● 使用 console.log 追踪执行流程

**基础追踪：**

```javascript
HookManager.regHook('Game_Player.prototype.moveStraight', function(next, direction) {
  console.log('=== 移动钩子开始 ===');
  console.log('方向:', direction);
  console.log('当前位置:', this.x, this.y);
  
  next();
  
  console.log('移动后位置:', this.x, this.y);
  console.log('=== 移动钩子结束 ===');
});
```

**追踪钩子链：**

```javascript
// 钩子A
HookManager.regHook('someFunction', function(next, arg) {
  console.log('→ 进入钩子A');
  next();
  console.log('← 离开钩子A');
}, { priority: 100, label: '钩子A' });

// 钩子B
HookManager.regHook('someFunction', function(next, arg) {
  console.log('  → 进入钩子B');
  next();
  console.log('  ← 离开钩子B');
}, { priority: 50, label: '钩子B' });

/* 输出：
→ 进入钩子A
  → 进入钩子B
    [原始函数执行]
  ← 离开钩子B
← 离开钩子A
*/
```

**追踪参数变化：**

```javascript
HookManager.regHook('Game_Actor.prototype.changeHp', function(next, value) {
  console.log('HP变化前:', this.hp);
  console.log('变化值:', value);
  
  next();
  
  console.log('HP变化后:', this.hp);
  console.log('实际变化:', this.hp - (this.hp - value));
});
```

● 检查钩子是否被调用

**方法1：计数器**

```javascript
let callCount = 0;

HookManager.regHook('Game_Player.prototype.update', function(next) {
  callCount++;
  console.log('update 被调用次数:', callCount);
  next();
});

// 检查
setTimeout(() => {
  console.log('总调用次数:', callCount);
}, 5000);
```

**方法2：时间戳**

```javascript
let lastCallTime = 0;

HookManager.regHook('someFunction', function(next) {
  const now = Date.now();
  const interval = now - lastCallTime;
  
  console.log('距离上次调用:', interval, 'ms');
  lastCallTime = now;
  
  next();
});
```

**方法3：堆栈追踪**

```javascript
HookManager.regHook('criticalFunction', function(next) {
  console.log('调用堆栈:');
  console.trace();
  next();
});
```

● 验证参数传递

**检查参数类型：**

```javascript
HookManager.regHook('Game_Party.prototype.gainGold', function(next, amount) {
  console.log('参数类型:', typeof amount);
  console.log('参数值:', amount);
  
  if (typeof amount !== 'number') {
    console.error('❌ 参数类型错误！期望 number，实际', typeof amount);
    return;
  }
  
  next();
});
```

**检查参数范围：**

```javascript
HookManager.regHook('Game_Actor.prototype.changeHp', function(next, value) {
  console.log('HP变化值:', value);
  
  if (Math.abs(value) > 9999) {
    console.warn('⚠️ HP变化值异常:', value);
  }
  
  next();
});
```

**检查对象状态：**

```javascript
HookManager.regHook('Game_Battler.prototype.performAction', function(next, action) {
  console.log('战斗者状态:');
  console.log('  HP:', this.hp, '/', this.mhp);
  console.log('  MP:', this.mp, '/', this.mmp);
  console.log('  是否死亡:', this.isDead());
  console.log('  状态:', this.states().map(s => s.name));
  
  next();
});
```

---

### 5.2 性能分析

● 启用统计功能

**基础启用：**

```javascript
// 1. 开启统计收集
HookManager.globalOptions.enableStats = true;

// 2. 注册需要监控的钩子
HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
}, {
  profile: true,
  label: '玩家更新'
});

// 3. 运行游戏一段时间

// 4. 查看统计
HookManager.printStats();
```

**批量启用：**

```javascript
// 开启全局监控
HookManager.globalOptions.enableProfiling = true;
HookManager.globalOptions.enableStats = true;

// 所有钩子自动监控
HookManager.regBatchHooks({
  'Game_Player.prototype.update': {
    hook: playerUpdateHook,
    label: '玩家更新'
  },
  'Scene_Map.prototype.update': {
    hook: mapUpdateHook,
    label: '地图更新'
  },
  'Game_Map.prototype.update': {
    hook: gameMapUpdateHook,
    label: '游戏地图更新'
  }
});
```

● 使用 printStats()

**基础用法：**

```javascript
// 运行游戏60秒后输出统计
setTimeout(() => {
  HookManager.printStats();
}, 60000);
```

**输出格式解读：**

```
=== Hook Performance Statistics ===

Game_Player.prototype.update:
  玩家更新:
    Calls: 3600              // 调用次数
    Avg: 0.12ms             // 平均耗时
    Min: 0.08ms             // 最小耗时
    Max: 2.34ms             // 最大耗时
    Total: 432.00ms         // 总耗时
    Errors: 0               // 错误次数

Scene_Map.prototype.update:
  地图更新:
    Calls: 3600
    Avg: 8.32ms             // ⚠️ 平均耗时较高
    Min: 5.12ms
    Max: 23.45ms            // ⚠️ 最大值超过1帧
    Total: 29952.00ms
    Errors: 2               // ⚠️ 有错误发生
```

**识别性能瓶颈：**

```javascript
function analyzePerformanceBottlenecks() {
  HookManager.printStats();
  
  console.log('\n=== 性能瓶颈分析 ===');
  
  HookManager.hooks.forEach((hookData, key) => {
    hookData.chain.forEach(hook => {
      const stats = HookManager.getStats(hook.id);
      if (!stats) return;
      
      // 平均耗时超过5ms
      if (stats.avgTime > 5) {
        console.warn(`⚠️ 慢钩子: ${hook.label}`);
        console.warn(`   路径: ${key}`);
        console.warn(`   平均: ${stats.avgTime.toFixed(2)}ms`);
      }
      
      // 最大耗时超过16.67ms（1帧）
      if (stats.maxTime > 16.67) {
        console.error(`❌ 卡顿钩子: ${hook.label}`);
        console.error(`   最大耗时: ${stats.maxTime.toFixed(2)}ms`);
      }
      
      // 有错误
      if (stats.errors > 0) {
        console.error(`❌ 错误钩子: ${hook.label}`);
        console.error(`   错误次数: ${stats.errors}`);
      }
    });
  });
}
```

**优化建议：**

```javascript
function suggestOptimizations() {
  const suggestions = [];
  
  HookManager.hooks.forEach((hookData, key) => {
    hookData.chain.forEach(hook => {
      const stats = HookManager.getStats(hook.id);
      if (!stats) return;
      
      // 调用频繁但耗时高
      if (stats.callCount > 1000 && stats.avgTime > 1) {
        suggestions.push({
          hook: hook.label,
          issue: '高频调用 + 高耗时',
          suggestion: '考虑优化算法或使用条件执行'
        });
      }
      
      // 耗时波动大
      const variance = stats.maxTime - stats.minTime;
      if (variance > 10) {
        suggestions.push({
          hook: hook.label,
          issue: `耗时波动大 (${variance.toFixed(2)}ms)`,
          suggestion: '检查是否有条件分支导致性能不稳定'
        });
      }
    });
  });
  
  console.log('\n=== 优化建议 ===');
  suggestions.forEach((s, i) => {
    console.log(`${i + 1}. ${s.hook}`);
    console.log(`   问题: ${s.issue}`);
    console.log(`   建议: ${s.suggestion}\n`);
  });
}
```

● 使用 getStats(hookId)

**获取单个钩子统计：**

```javascript
// 保存 hookId
let myHookId;

const unbind = HookManager.regHook('someFunction', function(next) {
  next();
}, {
  label: '我的钩子'
});

// 获取 hookId（需要从钩子链中查找）
HookManager.hooks.forEach((hookData) => {
  const hook = hookData.chain.find(h => h.label === '我的钩子');
  if (hook) {
    myHookId = hook.id;
  }
});

// 查看统计
setInterval(() => {
  const stats = HookManager.getStats(myHookId);
  if (stats) {
    console.log('当前统计:', stats);
  }
}, 5000);
```

**编程式性能检查：**

```javascript
function checkHookPerformance(hookLabel) {
  let hookId;
  
  // 查找钩子ID
  HookManager.hooks.forEach((hookData) => {
    const hook = hookData.chain.find(h => h.label === hookLabel);
    if (hook) hookId = hook.id;
  });
  
  if (!hookId) {
    console.error('未找到钩子:', hookLabel);
    return;
  }
  
  const stats = HookManager.getStats(hookId);
  if (!stats) {
    console.error('无统计数据:', hookLabel);
    return;
  }
  
  // 性能评级
  let rating;
  if (stats.avgTime < 1) {
    rating = '优秀';
  } else if (stats.avgTime < 5) {
    rating = '良好';
  } else if (stats.avgTime < 10) {
    rating = '一般';
  } else {
    rating = '差';
  }
  
  console.log(`钩子: ${hookLabel}`);
  console.log(`性能评级: ${rating}`);
  console.log(`平均耗时: ${stats.avgTime.toFixed(2)}ms`);
  console.log(`调用次数: ${stats.callCount}`);
  
  return { rating, stats };
}

// 使用
checkHookPerformance('玩家更新');
```
### 5.3 动态控制

● setHookEnabled() 的使用

**基础用法：**

```javascript
// 1. 注册钩子时保存必要信息
const hookLabel = '特效系统';
HookManager.regHook('Scene_Map.prototype.update', function(next) {
  next();
  this.updateSpecialEffects();
}, {
  label: hookLabel
});

// 2. 查找钩子ID
let hookId;
const hookData = HookManager.hooks.get('Scene_Map.prototype.update');
if (hookData) {
  const hook = hookData.chain.find(h => h.label === hookLabel);
  if (hook) hookId = hook.id;
}

// 3. 动态控制
HookManager.setHookEnabled('Scene_Map.prototype.update', hookId, false); // 禁用
HookManager.setHookEnabled('Scene_Map.prototype.update', hookId, true);  // 启用
```

● 临时禁用钩子

**场景1：性能对比测试**

```javascript
// 测试某个钩子的性能影响
function testHookImpact(hookKey, hookId) {
  console.log('=== 性能对比测试 ===');
  
  // 启用钩子，测试5秒
  HookManager.setHookEnabled(hookKey, hookId, true);
  const fpsWithHook = measureFPS(5000);
  console.log('启用钩子时 FPS:', fpsWithHook);
  
  // 禁用钩子，测试5秒
  HookManager.setHookEnabled(hookKey, hookId, false);
  const fpsWithoutHook = measureFPS(5000);
  console.log('禁用钩子时 FPS:', fpsWithoutHook);
  
  // 计算影响
  const impact = fpsWithHook - fpsWithoutHook;
  console.log('性能影响:', impact, 'FPS');
  
  // 恢复
  HookManager.setHookEnabled(hookKey, hookId, true);
}

function measureFPS(duration) {
  let frameCount = 0;
  const startTime = Date.now();
  
  const interval = setInterval(() => {
    frameCount++;
  }, 16.67);
  
  return new Promise(resolve => {
    setTimeout(() => {
      clearInterval(interval);
      const fps = frameCount / (duration / 1000);
      resolve(fps);
    }, duration);
  });
}
```

**场景2：故障排查**

```javascript
// 逐个禁用钩子，定位问题
function debugByDisablingHooks() {
  const hookData = HookManager.hooks.get('problematicFunction');
  if (!hookData) return;
  
  console.log('开始逐个禁用钩子...');
  
  hookData.chain.forEach((hook, index) => {
    console.log(`\n测试 ${index + 1}/${hookData.chain.length}: ${hook.label}`);
    
    // 禁用当前钩子
    hook.enabled = false;
    
    // 测试是否还有问题
    const hasIssue = testForIssue();
    
    if (!hasIssue) {
      console.log(`✓ 找到问题钩子: ${hook.label}`);
      return;
    }
    
    // 恢复钩子
    hook.enabled = true;
  });
}

function testForIssue() {
  // 执行测试逻辑
  try {
    // 触发可能有问题的代码
    return false; // 无问题
  } catch (e) {
    return true; // 有问题
  }
}
```

● 获取 hookId 的方法

**方法1：通过标签查找**

```javascript
function getHookIdByLabel(hookKey, label) {
  const hookData = HookManager.hooks.get(hookKey);
  if (!hookData) return null;
  
  const hook = hookData.chain.find(h => h.label === label);
  return hook ? hook.id : null;
}

// 使用
const hookId = getHookIdByLabel('Game_Player.prototype.update', '移动特效');
if (hookId) {
  HookManager.setHookEnabled('Game_Player.prototype.update', hookId, false);
}
```

**方法2：注册时保存引用**

```javascript
class FeatureManager {
  constructor() {
    this.hookRefs = new Map();
  }
  
  registerHook(key, func, options) {
    const unbind = HookManager.regHook(key, func, options);
    
    // 保存钩子引用
    const hookData = HookManager.hooks.get(key);
    const hook = hookData.chain[hookData.chain.length - 1]; // 最后一个是刚注册的
    
    this.hookRefs.set(options.label, {
      key: key,
      id: hook.id,
      unbind: unbind
    });
    
    return unbind;
  }
  
  enableHook(label) {
    const ref = this.hookRefs.get(label);
    if (ref) {
      HookManager.setHookEnabled(ref.key, ref.id, true);
    }
  }
  
  disableHook(label) {
    const ref = this.hookRefs.get(label);
    if (ref) {
      HookManager.setHookEnabled(ref.key, ref.id, false);
    }
  }
}

// 使用
const manager = new FeatureManager();
manager.registerHook('update', myHook, { label: '特效' });

manager.disableHook('特效'); // 禁用
manager.enableHook('特效');  // 启用
```

**方法3：遍历所有钩子**

```javascript
function listAllHooks() {
  const hooks = [];
  
  HookManager.hooks.forEach((hookData, key) => {
    hookData.chain.forEach(hook => {
      hooks.push({
        path: key,
        id: hook.id,
        label: hook.label,
        priority: hook.priority,
        enabled: hook.enabled
      });
    });
  });
  
  return hooks;
}

// 使用
const allHooks = listAllHooks();
console.table(allHooks);

// 禁用特定钩子
const targetHook = allHooks.find(h => h.label === '目标钩子');
if (targetHook) {
  HookManager.setHookEnabled(targetHook.path, targetHook.id, false);
}
```

---

### 5.4 常见问题排查

● 钩子未生效

**问题1：路径错误**

```javascript
// ❌ 错误：拼写错误
HookManager.regHook('Game_Player.prototype.moveStright', ...);
//                                              ^^^^^^^^ 应该是 moveStraight

// ✓ 检查方法
console.log(typeof Game_Player.prototype.moveStraight); // 'function'
console.log(typeof Game_Player.prototype.moveStright);  // 'undefined'

// ✓ 正确
HookManager.regHook('Game_Player.prototype.moveStraight', ...);
```

**问题2：注册时机过早**

```javascript
// ❌ 错误：对象还未定义
HookManager.regHook('Game_Player.prototype.update', ...);
// 此时 Game_Player 可能还未加载

// ✓ 正确：延迟到启动后
Scene_Boot.Totos.push(function() {
  HookManager.regHook('Game_Player.prototype.update', ...);
});
```

**问题3：条件函数返回 false**

```javascript
// ❌ 钩子注册了但不执行
HookManager.regHook('update', function(next) {
  console.log('这里不会执行');
  next();
}, {
  condition: () => false // 条件永远为 false
});

// ✓ 调试方法
HookManager.regHook('update', function(next) {
  next();
}, {
  condition: function() {
    const result = $gameSwitches.value(10);
    console.log('条件检查结果:', result); // 输出条件结果
    return result;
  }
});
```

**排查清单：**

```javascript
function diagnoseHookNotWorking(hookKey, hookLabel) {
  console.log('=== 钩子诊断 ===');
  console.log('目标:', hookKey);
  
  // 1. 检查路径是否存在
  const parts = hookKey.split('.');
  let obj = window;
  let valid = true;
  
  parts.forEach(part => {
    if (obj && obj[part]) {
      obj = obj[part];
    } else {
      console.error(`❌ 路径不存在: ${part}`);
      valid = false;
    }
  });
  
  if (valid) {
    console.log('✓ 路径有效');
  }
  
  // 2. 检查是否已注册
  const hookData = HookManager.hooks.get(hookKey);
  if (!hookData) {
    console.error('❌ 未找到钩子数据');
    return;
  }
  console.log('✓ 钩子已注册');
  
  // 3. 检查钩子是否在链中
  const hook = hookData.chain.find(h => h.label === hookLabel);
  if (!hook) {
    console.error('❌ 未找到指定钩子:', hookLabel);
    console.log('现有钩子:', hookData.chain.map(h => h.label));
    return;
  }
  console.log('✓ 钩子在链中');
  
  // 4. 检查是否启用
  if (!hook.enabled) {
    console.error('❌ 钩子已禁用');
    return;
  }
  console.log('✓ 钩子已启用');
  
  // 5. 检查条件
  if (hook.condition) {
    try {
      const result = hook.condition();
      console.log('条件函数返回:', result);
      if (!result) {
        console.warn('⚠️ 条件函数返回 false');
      }
    } catch (e) {
      console.error('❌ 条件函数错误:', e);
    }
  } else {
    console.log('✓ 无条件限制');
  }
  
  console.log('=== 诊断完成 ===');
}

// 使用
diagnoseHookNotWorking('Game_Player.prototype.update', '我的钩子');
```

● 钩子执行顺序错误

**问题：优先级设置不当**

```javascript
// 期望：验证 → 修改 → 显示
// 实际：显示 → 修改 → 验证（因为优先级相同，按注册顺序）

// ❌ 错误：都是默认优先级
HookManager.regHook('func', validate);  // 优先级 50
HookManager.regHook('func', modify);    // 优先级 50
HookManager.regHook('func', display);   // 优先级 50

// ✓ 正确：设置优先级
HookManager.regHook('func', validate, { priority: 200 });
HookManager.regHook('func', modify, { priority: 100 });
HookManager.regHook('func', display, { priority: 50 });
```

**调试方法：**

```javascript
function debugHookOrder(hookKey) {
  const hookData = HookManager.hooks.get(hookKey);
  if (!hookData) {
    console.error('未找到钩子:', hookKey);
    return;
  }
  
  console.log('=== 钩子执行顺序 ===');
  console.log('路径:', hookKey);
  console.log('钩子数量:', hookData.chain.length);
  console.log('\n执行顺序:');
  
  hookData.chain.forEach((hook, index) => {
    console.log(`${index + 1}. [优先级 ${hook.priority}] ${hook.label}`);
  });
}

// 使用
debugHookOrder('Game_Action.prototype.apply');
```
● 性能问题

**问题1：钩子耗时过长**

```javascript
// 识别慢钩子
function findSlowHooks(threshold = 5) {
  console.log(`=== 查找耗时超过 ${threshold}ms 的钩子 ===`);
  
  const slowHooks = [];
  
  HookManager.hooks.forEach((hookData, key) => {
    hookData.chain.forEach(hook => {
      const stats = HookManager.getStats(hook.id);
      if (stats && stats.avgTime > threshold) {
        slowHooks.push({
          path: key,
          label: hook.label,
          avgTime: stats.avgTime,
          maxTime: stats.maxTime,
          callCount: stats.callCount
        });
      }
    });
  });
  
  // 按平均耗时排序
  slowHooks.sort((a, b) => b.avgTime - a.avgTime);
  
  slowHooks.forEach((hook, index) => {
    console.log(`${index + 1}. ${hook.label}`);
    console.log(`   路径: ${hook.path}`);
    console.log(`   平均: ${hook.avgTime.toFixed(2)}ms`);
    console.log(`   最大: ${hook.maxTime.toFixed(2)}ms`);
    console.log(`   调用: ${hook.callCount} 次\n`);
  });
  
  return slowHooks;
}

// 使用
HookManager.globalOptions.enableStats = true;
// 运行游戏一段时间后
setTimeout(() => {
  findSlowHooks(5);
}, 60000);
```

**问题2：条件函数开销大**

```javascript
// ❌ 错误：条件函数中有复杂计算
HookManager.regHook('update', function(next) {
  next();
}, {
  condition: function() {
    // 每次都遍历整个数组！
    return $gameParty.members().some(actor => actor.hp < 100);
  }
});

// ✓ 优化1：缓存结果
let cachedCondition = false;
let lastCheck = 0;

HookManager.regHook('update', function(next) {
  next();
}, {
  condition: function() {
    const now = Date.now();
    if (now - lastCheck > 1000) { // 每秒检查一次
      cachedCondition = $gameParty.members().some(actor => actor.hp < 100);
      lastCheck = now;
    }
    return cachedCondition;
  }
});

// ✓ 优化2：使用简单条件
HookManager.regHook('update', function(next) {
  next();
}, {
  condition: () => $gameSwitches.value(10) // 简单的开关检查
});
```

**问题3：钩子链过长**

```javascript
// 检查钩子链长度
function checkHookChainLength() {
  console.log('=== 钩子链长度检查 ===');
  
  const longChains = [];
  
  HookManager.hooks.forEach((hookData, key) => {
    if (hookData.chain.length > 5) {
      longChains.push({
        path: key,
        length: hookData.chain.length,
        hooks: hookData.chain.map(h => h.label)
      });
    }
  });
  
  longChains.sort((a, b) => b.length - a.length);
  
  longChains.forEach(chain => {
    console.log(`⚠️ ${chain.path}: ${chain.length} 个钩子`);
    chain.hooks.forEach((label, i) => {
      console.log(`   ${i + 1}. ${label}`);
    });
    console.log('');
  });
  
  if (longChains.length === 0) {
    console.log('✓ 所有钩子链长度正常');
  }
}

// 使用
checkHookChainLength();
```

**优化建议：**

```javascript
// ❌ 避免：在高频函数上注册过多钩子
// update 每秒调用60次
HookManager.regHook('Scene_Map.prototype.update', hook1);
HookManager.regHook('Scene_Map.prototype.update', hook2);
HookManager.regHook('Scene_Map.prototype.update', hook3);
// ... 10个钩子

// ✓ 优化：合并钩子
HookManager.regHook('Scene_Map.prototype.update', function(next) {
  next();
  
  // 统一处理
  updateParticles();
  updateWeather();
  updateAnimations();
});
```

● 内存泄漏

**问题1：未解绑的钩子**

```javascript
// ❌ 错误：临时钩子未解绑
function showTemporaryEffect() {
  HookManager.regHook('Game_Player.prototype.update', function(next) {
    next();
    this.showEffect();
  });
  // 特效结束后钩子仍在内存中！
}

// 每次调用都注册新钩子
for (let i = 0; i < 100; i++) {
  showTemporaryEffect(); // 内存泄漏！
}

// ✓ 正确：使用解绑
function showTemporaryEffect(duration) {
  const unbind = HookManager.regHook('Game_Player.prototype.update', function(next) {
    next();
    this.showEffect();
  });
  
  // 定时解绑
  setTimeout(() => {
    unbind();
  }, duration);
}
```

**问题2：循环引用**

```javascript
// ❌ 错误：钩子引用外部对象
class EffectManager {
  constructor() {
    this.data = new Array(10000); // 大对象
    
    HookManager.regHook('update', function(next) {
      next();
      console.log(this.data); // 钩子持有 EffectManager 引用
    }.bind(this));
  }
}

// 即使 EffectManager 实例被销毁，钩子仍持有引用

// ✓ 正确：解绑钩子
class EffectManager {
  constructor() {
    this.data = new Array(10000);
    this.unbind = null;
  }
  
  enable() {
    this.unbind = HookManager.regHook('update', function(next) {
      next();
      console.log(this.data);
    }.bind(this));
  }
  
  destroy() {
    if (this.unbind) {
      this.unbind(); // 释放引用
      this.unbind = null;
    }
    this.data = null;
  }
}
```

**检测内存泄漏：**

```javascript
// 监控钩子数量
function monitorHookCount() {
  let lastCount = 0;
  
  setInterval(() => {
    let currentCount = 0;
    
    HookManager.hooks.forEach((hookData) => {
      currentCount += hookData.chain.length;
    });
    
    if (currentCount > lastCount) {
      console.warn(`⚠️ 钩子数量增加: ${lastCount} → ${currentCount}`);
      
      if (currentCount - lastCount > 10) {
        console.error('❌ 可能存在内存泄漏！');
        
        // 列出所有钩子
        HookManager.hooks.forEach((hookData, key) => {
          console.log(`${key}: ${hookData.chain.length} 个钩子`);
        });
      }
    }
    
    lastCount = currentCount;
  }, 5000);
}

// 启动监控
monitorHookCount();
```

**清理工具：**

```javascript
// 清理所有钩子
function emergencyCleanup() {
  console.log('=== 紧急清理 ===');
  
  let totalHooks = 0;
  HookManager.hooks.forEach((hookData) => {
    totalHooks += hookData.chain.length;
  });
  
  console.log(`清理前: ${totalHooks} 个钩子`);
  
  HookManager.clearAll();
  
  console.log('清理后: 0 个钩子');
  console.log('✓ 清理完成');
}

// 在控制台调用
// emergencyCleanup();
```

**最佳实践：**

```javascript
// 插件生命周期管理模板
class PluginLifecycleManager {
  constructor() {
    this.hooks = [];
    this.timers = [];
    this.listeners = [];
  }
  
  // 注册钩子
  registerHook(target, func, options) {
    const unbind = HookManager.regHook(target, func, options);
    this.hooks.push(unbind);
    return unbind;
  }
  
  // 注册定时器
  registerTimer(callback, interval) {
    const id = setInterval(callback, interval);
    this.timers.push(id);
    return id;
  }
  
  // 注册事件监听
  registerListener(element, event, handler) {
    element.addEventListener(event, handler);
    this.listeners.push({ element, event, handler });
  }
  
  // 清理所有资源
  cleanup() {
    // 解绑钩子
    this.hooks.forEach(unbind => unbind());
    this.hooks = [];
    
    // 清除定时器
    this.timers.forEach(id => clearInterval(id));
    this.timers = [];
    
    // 移除监听器
    this.listeners.forEach(({ element, event, handler }) => {
      element.removeEventListener(event, handler);
    });
    this.listeners = [];
    
    console.log('✓ 资源清理完成');
  }
}

// 使用
const manager = new PluginLifecycleManager();

// 注册资源
manager.registerHook('update', myHook);
manager.registerTimer(() => console.log('tick'), 1000);

// 插件卸载时
manager.cleanup();
```

---

### 故障排查清单

**快速检查清单：**

```javascript
function quickDiagnostic() {
  console.log('=== 快速诊断 ===\n');
  
  // 1. 钩子总数
  let totalHooks = 0;
  HookManager.hooks.forEach((hookData) => {
    totalHooks += hookData.chain.length;
  });
  console.log(`✓ 钩子总数: ${totalHooks}`);
  
  // 2. 性能状态
  if (HookManager.globalOptions.enableStats) {
    let slowHooks = 0;
    HookManager.hooks.forEach((hookData) => {
      hookData.chain.forEach(hook => {
        const stats = HookManager.getStats(hook.id);
        if (stats && stats.avgTime > 5) slowHooks++;
      });
    });
    console.log(`${slowHooks > 0 ? '⚠️' : '✓'} 慢钩子: ${slowHooks} 个`);
  } else {
    console.log('⚠️ 统计未启用');
  }
  
  // 3. 配置状态
  console.log(`✓ 性能监控: ${HookManager.globalOptions.enableProfiling ? '开启' : '关闭'}`);
  console.log(`✓ 统计收集: ${HookManager.globalOptions.enableStats ? '开启' : '关闭'}`);
  console.log(`✓ 默认优先级: ${HookManager.globalOptions.defaultPriority}`);
  console.log(`✓ 缓存时间: ${HookManager.globalOptions.conditionCacheTime}ms`);
  
  // 4. 钩子链长度
  let longChains = 0;
  HookManager.hooks.forEach((hookData) => {
    if (hookData.chain.length > 5) longChains++;
  });
  console.log(`${longChains > 0 ? '⚠️' : '✓'} 长钩子链: ${longChains} 个`);
  
  console.log('\n=== 诊断完成 ===');
}

// 在控制台运行
// quickDiagnostic();
```
## 第六部分：实战案例

### 6.1 案例1：自定义移动系统

● 需求分析

实现一个增强的移动系统，包含以下功能：
1. 移动时播放脚步声
2. 踩到特殊地形时触发事件
3. 记录移动历史（用于回退功能）
4. 移动速度根据地形变化

● 钩子设计

```javascript
// 功能分层设计
// 优先级 200: 数据记录（最先执行）
// 优先级 100: 地形检测
// 优先级 50:  音效播放（最后执行）
```

● 完整代码

```javascript
class EnhancedMovementSystem {
  constructor() {
    this.moveHistory = [];
    this.maxHistoryLength = 100;
    this.unbinders = [];
  }
  
  enable() {
    // 1. 记录移动历史（优先级最高）
    this.unbinders.push(
      HookManager.regHook('Game_Player.prototype.moveStraight', function(next, direction) {
        // 记录移动前的位置
        this._moveHistory = this._moveHistory || [];
        this._moveHistory.push({
          x: this.x,
          y: this.y,
          direction: this.direction(),
          timestamp: Date.now()
        });
        
        // 限制历史长度
        if (this._moveHistory.length > 100) {
          this._moveHistory.shift();
        }
        
        next();
      }, {
        priority: 200,
        label: '移动历史记录'
      })
    );
    
    // 2. 地形检测和速度调整（中等优先级）
    this.unbinders.push(
      HookManager.regHook('Game_Player.prototype.moveStraight', function(next, direction) {
        next();
        
        // 检测当前地形
        const terrainTag = $gameMap.terrainTag(this.x, this.y);
        
        // 根据地形调整速度
        switch(terrainTag) {
          case 1: // 沙地
            this.setMoveSpeed(3);
            break;
          case 2: // 沼泽
            this.setMoveSpeed(2);
            break;
          case 3: // 道路
            this.setMoveSpeed(5);
            break;
          default:
            this.setMoveSpeed(4);
        }
        
        // 特殊地形事件
        if (terrainTag === 4) { // 陷阱
          this.triggerTrapEvent();
        }
      }, {
        priority: 100,
        label: '地形检测'
      })
    );
    
    // 3. 音效播放（优先级最低）
    this.unbinders.push(
      HookManager.regHook('Game_Player.prototype.moveStraight', function(next, direction) {
        next();
        
        // 只在移动成功时播放音效
        if (this.isMovementSucceeded()) {
          const terrainTag = $gameMap.terrainTag(this.x, this.y);
          
          // 根据地形播放不同音效
          const soundMap = {
            1: 'Sand',      // 沙地
            2: 'Water1',    // 沼泽
            3: 'Stone',     // 道路
            default: 'Step' // 默认
          };
          
          const soundName = soundMap[terrainTag] || soundMap.default;
          
          AudioManager.playSe({
            name: soundName,
            volume: 50,
            pitch: 100 + Math.random() * 20 - 10, // 随机音调
            pan: 0
          });
        }
      }, {
        priority: 50,
        label: '移动音效',
        profile: true,
        threshold: 5
      })
    );
    
    console.log('✓ 增强移动系统已启用');
  }
  
  disable() {
    this.unbinders.forEach(unbind => unbind());
    this.unbinders = [];
    console.log('✓ 增强移动系统已禁用');
  }
  
  // 回退功能
  undoLastMove() {
    const player = $gamePlayer;
    if (!player._moveHistory || player._moveHistory.length === 0) {
      console.log('没有移动历史');
      return;
    }
    
    const lastPos = player._moveHistory.pop();
    player.locate(lastPos.x, lastPos.y);
    player.setDirection(lastPos.direction);
    
    console.log('已回退到:', lastPos);
  }
}

// 使用
const movementSystem = new EnhancedMovementSystem();
movementSystem.enable();

// 添加回退命令（可在控制台调用）
window.undoMove = () => movementSystem.undoLastMove();
```

● 测试验证

```javascript
// 测试脚本
function testMovementSystem() {
  console.log('=== 测试移动系统 ===');
  
  // 1. 测试移动历史
  const player = $gamePlayer;
  const initialHistory = player._moveHistory ? player._moveHistory.length : 0;
  
  // 模拟移动
  player.moveStraight(2); // 向下
  player.moveStraight(6); // 向右
  
  const newHistory = player._moveHistory ? player._moveHistory.length : 0;
  console.log(`✓ 移动历史: ${initialHistory} → ${newHistory}`);
  
  // 2. 测试回退
  const beforeUndo = { x: player.x, y: player.y };
  window.undoMove();
  const afterUndo = { x: player.x, y: player.y };
  
  console.log(`✓ 回退测试: (${beforeUndo.x},${beforeUndo.y}) → (${afterUndo.x},${afterUndo.y})`);
  
  // 3. 测试性能
  HookManager.globalOptions.enableStats = true;
  
  setTimeout(() => {
    const stats = [];
    HookManager.hooks.get('Game_Player.prototype.moveStraight').chain.forEach(hook => {
      const s = HookManager.getStats(hook.id);
      if (s) {
        stats.push({
          label: hook.label,
          avgTime: s.avgTime.toFixed(2)
        });
      }
    });
    
    console.log('✓ 性能统计:');
    console.table(stats);
  }, 5000);
}

// 运行测试
// testMovementSystem();
```

---

### 6.2 案例2：技能增强系统

● 需求分析

实现一个技能增强系统，包含：
1. 技能释放前的条件检查（MP、状态、冷却）
2. 伤害计算增强（暴击、连击、属性克制）
3. 技能特效（额外效果、动画）
4. 性能监控

● 多层钩子协作

```javascript
class SkillEnhancementSystem {
  constructor() {
    this.skillCooldowns = new Map(); // 技能冷却
    this.comboCounter = new Map();   // 连击计数
    this.unbinders = [];
  }
  
  enable() {
    // 层级1: 条件验证（优先级 200）
    this.unbinders.push(
      HookManager.regHook('Game_Action.prototype.apply', function(next, target) {
        const skill = this.item();
        const subject = this.subject();
        
        // 检查技能冷却
        const cooldownKey = `${subject.actorId()}_${skill.id}`;
        const cooldown = this.constructor._skillCooldowns.get(cooldownKey);
        
        if (cooldown && Date.now() < cooldown) {
          const remaining = Math.ceil((cooldown - Date.now()) / 1000);
          console.log(`技能冷却中，剩余 ${remaining} 秒`);
          return; // 阻止释放
        }
        
        // 检查自定义条件
        if (skill.meta.requireFullHP && subject.hpRate() < 1.0) {
          console.log('需要满HP才能使用此技能');
          return;
        }
        
        if (skill.meta.requireState) {
          const stateId = parseInt(skill.meta.requireState);
          if (!subject.isStateAffected(stateId)) {
            console.log('需要特定状态才能使用');
            return;
          }
        }
        
        next();
        
        // 设置冷却
        if (skill.meta.cooldown) {
          const cooldownTime = parseInt(skill.meta.cooldown) * 1000;
          this.constructor._skillCooldowns.set(cooldownKey, Date.now() + cooldownTime);
        }
      }, {
        priority: 200,
        condition: function() {
          return this.isSkill();
        },
        label: '技能条件检查'
      })
    );
    
    // 层级2: 伤害计算（优先级 100）
    this.unbinders.push(
      HookManager.regHook('Game_Action.prototype.makeDamageValue', function(next, target, critical) {
        let damage = next();
        const skill = this.item();
        const subject = this.subject();
        
        // 连击加成
        const comboKey = subject.actorId();
        let combo = this.constructor._comboCounter.get(comboKey) || 0;
        
        if (combo > 0) {
          const comboBonus = 1 + (combo * 0.15); // 每层+15%
          damage = Math.floor(damage * comboBonus);
          console.log(`连击 x${combo + 1}，伤害 x${comboBonus.toFixed(2)}`);
        }
        
        // 更新连击
        combo++;
        this.constructor._comboCounter.set(comboKey, combo);
        
        // 连击重置定时器
        setTimeout(() => {
          this.constructor._comboCounter.set(comboKey, 0);
        }, 3000);
        
        // 属性克制
        if (skill.meta.element && target.meta.weakness) {
          const elements = skill.meta.element.split(',');
          const weaknesses = target.meta.weakness.split(',');
          
          const hasAdvantage = elements.some(e => weaknesses.includes(e));
          if (hasAdvantage) {
            damage = Math.floor(damage * 1.5);
            console.log('属性克制！伤害 x1.5');
          }
        }
        
        // 暴击增强
        if (critical && skill.meta.criticalBonus) {
          const bonus = parseFloat(skill.meta.criticalBonus);
          damage = Math.floor(damage * (1 + bonus));
        }
        
        return damage;
      }, {
        priority: 100,
        condition: function() {
          return this.isSkill() && this.item().damage.type > 0;
        },
        label: '伤害计算增强',
        profile: true,
        threshold: 10
      })
    );
    
    // 层级3: 特效处理（优先级 50）
    this.unbinders.push(
      HookManager.regHook('Game_Action.prototype.apply', function(next, target) {
        next();
        
        const skill = this.item();
        
        // 额外效果
        if (skill.meta.extraEffect) {
          const effects = skill.meta.extraEffect.split(',');
          
          effects.forEach(effect => {
            switch(effect.trim()) {
              case 'burn':
                target.addState(10); // 灼烧状态
                break;
              case 'slow':
                target.addState(11); // 减速状态
                break;
              case 'heal':
                this.subject().gainHp(50);
                break;
            }
          });
        }
        
        // 自定义动画
        if (skill.meta.customAnimation) {
          const animId = parseInt(skill.meta.customAnimation);
          target.startAnimation(animId, false, 0);
        }
      }, {
        priority: 50,
        condition: function() {
          return this.isSkill();
        },
        label: '技能特效'
      })
    );
    
    // 静态属性初始化
    Game_Action._skillCooldowns = this.skillCooldowns;
    Game_Action._comboCounter = this.comboCounter;
    
    console.log('✓ 技能增强系统已启用');
  }
  
  disable() {
    this.unbinders.forEach(unbind => unbind());
    this.unbinders = [];
    this.skillCooldowns.clear();
    this.comboCounter.clear();
    console.log('✓ 技能增强系统已禁用');
  }
  
  // 查看冷却状态
  getCooldownStatus(actorId) {
    const cooldowns = [];
    
    this.skillCooldowns.forEach((time, key) => {
      if (key.startsWith(`${actorId}_`)) {
        const skillId = key.split('_')[1];
        const remaining = Math.max(0, Math.ceil((time - Date.now()) / 1000));
        
        if (remaining > 0) {
          cooldowns.push({
            skillId: skillId,
            remaining: remaining
          });
        }
      }
    });
    
    return cooldowns;
  }
}

// 使用
const skillSystem = new SkillEnhancementSystem();
skillSystem.enable();

// 查看冷却（控制台）
window.checkCooldown = (actorId) => {
  const cooldowns = skillSystem.getCooldownStatus(actorId);
  console.table(cooldowns);
};
```

● 技能备注标签示例

```
技能备注栏配置示例：

<requireFullHP>          // 需要满HP
<requireState:5>         // 需要5号状态
<cooldown:10>            // 冷却10秒
<element:fire,wind>      // 火、风属性
<criticalBonus:0.5>      // 暴击额外+50%
<extraEffect:burn,slow>  // 附加灼烧和减速
<customAnimation:50>     // 播放50号动画
```

● 性能优化

```javascript
// 性能监控
function monitorSkillPerformance() {
  HookManager.globalOptions.enableStats = true;
  
  setTimeout(() => {
    console.log('=== 技能系统性能报告 ===');
    
    const hookKey = 'Game_Action.prototype.makeDamageValue';
    const hookData = HookManager.hooks.get(hookKey);
    
    if (hookData) {
      hookData.chain.forEach(hook => {
        const stats = HookManager.getStats(hook.id);
        if (stats) {
          console.log(`${hook.label}:`);
          console.log(`  平均: ${stats.avgTime.toFixed(2)}ms`);
          console.log(`  调用: ${stats.callCount} 次`);
        }
      });
    }
  }, 60000);
}

// monitorSkillPerformance();
```
### 6.3 案例3：战斗日志系统

● 需求分析

实现一个完整的战斗日志系统：
1. 记录所有战斗行为（攻击、技能、道具）
2. 记录伤害、恢复、状态变化
3. 实时显示日志窗口
4. 支持日志导出和回放

● Before/After 模式应用

```javascript
class BattleLogSystem {
  constructor() {
    this.logs = [];
    this.maxLogs = 1000;
    this.unbinders = [];
    this.logWindow = null;
  }
  
  enable() {
    // 1. 记录行动开始（Before模式）
    this.unbinders.push(
      HookManager.regHook('Game_Action.prototype.apply', function(next, target) {
        const subject = this.subject();
        const item = this.item();
        
        // 记录行动开始
        const logEntry = {
          timestamp: Date.now(),
          type: 'action_start',
          subject: subject.name(),
          subjectId: subject.actorId() || subject.enemyId(),
          action: item.name,
          actionId: item.id,
          target: target.name(),
          targetId: target.actorId() || target.enemyId()
        };
        
        this.constructor._currentLog = logEntry;
        
        next();
      }, {
        priority: 200,
        label: '行动开始记录'
      })
    );
    
    // 2. 记录伤害/恢复（After模式）
    this.unbinders.push(
      HookManager.regHook('Game_Action.prototype.executeDamage', function(next, target, value) {
        next();
        
        // 记录伤害结果
        const logEntry = this.constructor._currentLog;
        if (logEntry) {
          logEntry.damage = value;
          logEntry.critical = target.result().critical;
          logEntry.targetHpBefore = target.hp + value;
          logEntry.targetHpAfter = target.hp;
          
          // 添加到日志
          this.constructor._battleLogs.push({ ...logEntry });
          
          // 显示日志
          this.constructor._displayLog(logEntry);
        }
      }, {
        priority: 50,
        label: '伤害记录'
      })
    );
    
    // 3. 记录状态变化
    this.unbinders.push(
      HookManager.regHook('Game_Battler.prototype.addState', function(next, stateId) {
        const stateName = $dataStates[stateId].name;
        
        next();
        
        // 记录状态添加
        const logEntry = {
          timestamp: Date.now(),
          type: 'state_add',
          target: this.name(),
          targetId: this.actorId() || this.enemyId(),
          state: stateName,
          stateId: stateId
        };
        
        this.constructor._battleLogs.push(logEntry);
        this.constructor._displayLog(logEntry);
      }, {
        priority: 50,
        label: '状态添加记录'
      })
    );
    
    // 4. 记录状态移除
    this.unbinders.push(
      HookManager.regHook('Game_Battler.prototype.removeState', function(next, stateId) {
        const stateName = $dataStates[stateId].name;
        
        next();
        
        const logEntry = {
          timestamp: Date.now(),
          type: 'state_remove',
          target: this.name(),
          targetId: this.actorId() || this.enemyId(),
          state: stateName,
          stateId: stateId
        };
        
        this.constructor._battleLogs.push(logEntry);
        this.constructor._displayLog(logEntry);
      }, {
        priority: 50,
        label: '状态移除记录'
      })
    );
    
    // 初始化静态属性
    Game_Action._currentLog = null;
    Game_Action._battleLogs = this.logs;
    Game_Action._displayLog = this.displayLog.bind(this);
    Game_Battler._battleLogs = this.logs;
    Game_Battler._displayLog = this.displayLog.bind(this);
    
    console.log('✓ 战斗日志系统已启用');
  }
  
  disable() {
    this.unbinders.forEach(unbind => unbind());
    this.unbinders = [];
    console.log('✓ 战斗日志系统已禁用');
  }
  
  // 显示日志
  displayLog(entry) {
    let message = '';
    
    switch(entry.type) {
      case 'action_start':
        message = `${entry.subject} 使用 ${entry.action} 对 ${entry.target}`;
        break;
        
      case 'state_add':
        message = `${entry.target} 获得状态: ${entry.state}`;
        break;
        
      case 'state_remove':
        message = `${entry.target} 失去状态: ${entry.state}`;
        break;
    }
    
    if (entry.damage !== undefined) {
      const dmgType = entry.damage > 0 ? '造成' : '恢复';
      const dmgValue = Math.abs(entry.damage);
      const critical = entry.critical ? ' [暴击]' : '';
      message += ` - ${dmgType} ${dmgValue} 伤害${critical}`;
    }
    
    console.log(`[战斗日志] ${message}`);
    
    // 限制日志数量
    if (this.logs.length > this.maxLogs) {
      this.logs.shift();
    }
  }
  
  // 导出日志
  exportLogs() {
    const data = JSON.stringify(this.logs, null, 2);
    const blob = new Blob([data], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    
    const a = document.createElement('a');
    a.href = url;
    a.download = `battle_log_${Date.now()}.json`;
    a.click();
    
    console.log('✓ 日志已导出');
  }
  
  // 获取统计
  getStatistics() {
    const stats = {
      totalActions: 0,
      totalDamage: 0,
      totalHealing: 0,
      criticalHits: 0,
      statesAdded: 0,
      statesRemoved: 0
    };
    
    this.logs.forEach(log => {
      switch(log.type) {
        case 'action_start':
          stats.totalActions++;
          if (log.damage > 0) {
            stats.totalDamage += log.damage;
          } else if (log.damage < 0) {
            stats.totalHealing += Math.abs(log.damage);
          }
          if (log.critical) {
            stats.criticalHits++;
          }
          break;
          
        case 'state_add':
          stats.statesAdded++;
          break;
          
        case 'state_remove':
          stats.statesRemoved++;
          break;
      }
    });
    
    return stats;
  }
  
  // 清空日志
  clearLogs() {
    this.logs = [];
    Game_Action._battleLogs = this.logs;
    Game_Battler._battleLogs = this.logs;
    console.log('✓ 日志已清空');
  }
}

// 使用
const battleLog = new BattleLogSystem();
battleLog.enable();

// 控制台命令
window.exportBattleLog = () => battleLog.exportLogs();
window.battleStats = () => {
  const stats = battleLog.getStatistics();
  console.table(stats);
};
```

● UI更新时机

```javascript
// 创建日志窗口（可选）
class Window_BattleLog extends Window_Base {
  initialize() {
    const width = 400;
    const height = 300;
    const x = Graphics.boxWidth - width;
    const y = 0;
    super.initialize(x, y, width, height);
    this.opacity = 200;
    this._logs = [];
    this._maxLines = 10;
  }
  
  addLog(message) {
    this._logs.push(message);
    if (this._logs.length > this._maxLines) {
      this._logs.shift();
    }
    this.refresh();
  }
  
  refresh() {
    this.contents.clear();
    
    this._logs.forEach((log, index) => {
      const y = index * this.lineHeight();
      this.drawText(log, 0, y, this.contentsWidth());
    });
  }
}

// 在战斗场景中添加窗口
const _Scene_Battle_createAllWindows = Scene_Battle.prototype.createAllWindows;
Scene_Battle.prototype.createAllWindows = function() {
  _Scene_Battle_createAllWindows.call(this);
  this._logWindow = new Window_BattleLog();
  this.addWindow(this._logWindow);
};
```

---

### 6.4 案例4：插件兼容性处理

● 需求分析

处理多个插件之间的兼容性问题：
1. 检测其他插件的钩子
2. 调整优先级避免冲突
3. 提供降级方案

● 检测其他插件

```javascript
class CompatibilityManager {
  constructor() {
    this.detectedPlugins = new Map();
  }
  
  // 检测已安装的插件
  detectPlugins() {
    console.log('=== 检测插件 ===');
    
    // 检测常见插件
    const knownPlugins = {
      'YEP_BattleEngineCore': () => typeof Yanfly !== 'undefined' && Yanfly.BEC,
      'MOG_BattleHud': () => typeof Moghunter !== 'undefined',
      'SRD_SuperToolsEngine': () => typeof SRD !== 'undefined',
      'GALV_LayerGraphics': () => typeof Galv !== 'undefined'
    };
    
    Object.entries(knownPlugins).forEach(([name, detector]) => {
      if (detector()) {
        this.detectedPlugins.set(name, true);
        console.log(`✓ 检测到: ${name}`);
      }
    });
    
    // 检测钩子冲突
    this.detectHookConflicts();
  }
  
  // 检测钩子冲突
  detectHookConflicts() {
    console.log('\n=== 检测钩子冲突 ===');
    
    const conflicts = [];
    
    HookManager.hooks.forEach((hookData, key) => {
      if (hookData.chain.length > 3) {
        conflicts.push({
          path: key,
          count: hookData.chain.length,
          hooks: hookData.chain.map(h => h.label)
        });
      }
    });
    
    if (conflicts.length > 0) {
      console.warn('⚠️ 发现潜在冲突:');
      conflicts.forEach(c => {
        console.log(`  ${c.path}: ${c.count} 个钩子`);
        c.hooks.forEach((label, i) => {
          console.log(`    ${i + 1}. ${label}`);
        });
      });
    } else {
      console.log('✓ 未发现冲突');
    }
  }
  
  // 调整优先级
  adjustPriorities() {
    // 如果检测到 YEP_BattleEngineCore
    if (this.detectedPlugins.has('YEP_BattleEngineCore')) {
      console.log('检测到 YEP 战斗引擎，调整优先级...');
      
      // YEP 的钩子通常优先级较高
      // 我们的钩子应该设置更低的优先级
      HookManager.globalOptions.defaultPriority = 30;
    }
  }
  
  // 提供兼容性补丁
  applyCompatibilityPatches() {
    // 补丁1: 与 MOG_BattleHud 兼容
    if (this.detectedPlugins.has('MOG_BattleHud')) {
      console.log('应用 MOG_BattleHud 兼容补丁...');
      
      // MOG 会修改 Sprite_Actor
      // 确保我们的钩子在 MOG 之后执行
      HookManager.regHook('Sprite_Actor.prototype.update', function(next) {
        next();
        // 我们的逻辑
      }, {
        priority: 20, // 低优先级
        label: 'MOG兼容层'
      });
    }
  }
}

// 使用
const compatibility = new CompatibilityManager();
compatibility.detectPlugins();
compatibility.adjustPriorities();
compatibility.applyCompatibilityPatches();
```

● 优先级协商

```javascript
// 插件优先级注册表
const PluginPriorityRegistry = {
  // 核心系统（最高优先级）
  'CoreSystem': 200,
  
  // 战斗引擎
  'YEP_BattleEngine': 150,
  'MOG_BattleHud': 140,
  
  // 功能增强
  'SkillEnhancement': 100,
  'ItemEnhancement': 90,
  
  // UI/特效（最低优先级）
  'UIEnhancement': 50,
  'VisualEffects': 40
};

// 根据插件类型获取推荐优先级
function getRecommendedPriority(pluginType) {
  return PluginPriorityRegistry[pluginType] || 50;
}

// 注册钩子时使用
HookManager.regHook('someFunction', myHook, {
  priority: getRecommendedPriority('SkillEnhancement'),
  label: '技能增强'
});
```

● 降级处理

```javascript
class GracefulDegradation {
  static registerWithFallback(target, hook, options) {
    try {
      // 尝试注册钩子
      return HookManager.regHook(target, hook, options);
    } catch (error) {
      console.warn(`钩子注册失败: ${target}`, error);
      
      // 降级方案1: 尝试别名方式
      try {
        return this.registerWithAlias(target, hook);
      } catch (aliasError) {
        console.error('别名方式也失败:', aliasError);
        
        // 降级方案2: 直接修改（最后手段）
        return this.registerDirect(target, hook);
      }
    }
  }
  
  static registerWithAlias(target, hook) {
    const parts = target.split('.');
    const method = parts.pop();
    const obj = parts.reduce((o, p) => o[p], window);
    
    const original = obj[method];
    obj[method] = function(...args) {
      hook.call(this, () => original.apply(this, args), ...args);
    };
    
    return () => {
      obj[method] = original;
    };
  }
  
  static registerDirect(target, hook) {
    console.warn('使用直接修改方式（不推荐）');
    // 实现直接修改逻辑
  }
}

// 使用
const unbind = GracefulDegradation.registerWithFallback(
  'Game_Player.prototype.update',
  myHook,
  { label: '我的钩子' }
);
```
### 6.5 案例5：性能优化实战

● 问题发现

```javascript
// 步骤1: 启用性能监控
function startPerformanceAnalysis() {
  console.log('=== 开始性能分析 ===');
  
  // 开启全局监控
  HookManager.globalOptions.enableProfiling = true;
  HookManager.globalOptions.enableStats = true;
  
  // 监控关键函数
  const criticalFunctions = [
    'Scene_Map.prototype.update',
    'Game_Player.prototype.update',
    'Game_Map.prototype.update',
    'Sprite_Character.prototype.update',
    'Game_CharacterBase.prototype.update'
  ];
  
  criticalFunctions.forEach(func => {
    HookManager.regHook(func, function(next) {
      next();
    }, {
      profile: true,
      threshold: 16.67, // 1帧时间
      label: `性能监控-${func}`,
      onSlow: (duration) => {
        console.warn(`⚠️ ${func} 耗时过长: ${duration.toFixed(2)}ms`);
      }
    });
  });
  
  console.log('✓ 性能监控已启动');
  console.log('运行游戏60秒后将输出报告...');
  
  // 60秒后输出报告
  setTimeout(() => {
    generatePerformanceReport();
  }, 60000);
}

// 步骤2: 生成性能报告
function generatePerformanceReport() {
  console.log('\n=== 性能分析报告 ===');
  
  HookManager.printStats();
  
  // 识别性能瓶颈
  const bottlenecks = [];
  
  HookManager.hooks.forEach((hookData, key) => {
    hookData.chain.forEach(hook => {
      const stats = HookManager.getStats(hook.id);
      if (!stats) return;
      
      // 平均耗时超过5ms
      if (stats.avgTime > 5) {
        bottlenecks.push({
          path: key,
          label: hook.label,
          avgTime: stats.avgTime,
          maxTime: stats.maxTime,
          callCount: stats.callCount,
          totalTime: stats.totalTime
        });
      }
    });
  });
  
  if (bottlenecks.length > 0) {
    console.log('\n⚠️ 性能瓶颈:');
    bottlenecks.sort((a, b) => b.totalTime - a.totalTime);
    
    bottlenecks.forEach((b, i) => {
      console.log(`\n${i + 1}. ${b.label}`);
      console.log(`   路径: ${b.path}`);
      console.log(`   平均耗时: ${b.avgTime.toFixed(2)}ms`);
      console.log(`   最大耗时: ${b.maxTime.toFixed(2)}ms`);
      console.log(`   调用次数: ${b.callCount}`);
      console.log(`   总耗时: ${b.totalTime.toFixed(2)}ms`);
      console.log(`   影响: ${((b.totalTime / 60000) * 100).toFixed(2)}% 的总时间`);
    });
  } else {
    console.log('✓ 未发现明显性能问题');
  }
  
  return bottlenecks;
}

// 使用
// startPerformanceAnalysis();
```

● 瓶颈定位

```javascript
// 深度分析特定函数
function analyzeFunction(functionPath) {
  console.log(`=== 分析函数: ${functionPath} ===`);
  
  let callStack = [];
  let maxDepth = 0;
  
  // 注册分析钩子
  const unbind = HookManager.regHook(functionPath, function(next) {
    const depth = callStack.length;
    maxDepth = Math.max(maxDepth, depth);
    
    const indent = '  '.repeat(depth);
    const startTime = performance.now();
    
    console.log(`${indent}→ 进入 (深度: ${depth})`);
    callStack.push({ startTime, depth });
    
    const result = next();
    
    const entry = callStack.pop();
    const duration = performance.now() - entry.startTime;
    
    console.log(`${indent}← 退出 (耗时: ${duration.toFixed(2)}ms)`);
    
    if (duration > 10) {
      console.warn(`${indent}⚠️ 耗时过长!`);
    }
    
    return result;
  }, {
    profile: true,
    label: '深度分析'
  });
  
  // 运行测试
  setTimeout(() => {
    console.log(`\n最大调用深度: ${maxDepth}`);
    unbind();
  }, 5000);
}

// 使用
// analyzeFunction('Game_Map.prototype.update');
```

● 优化方案

**优化1: 减少不必要的计算**

```javascript
// ❌ 优化前: 每帧都计算
HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
  
  // 每帧都遍历所有事件
  $gameMap.events().forEach(event => {
    const distance = Math.abs(this.x - event.x) + Math.abs(this.y - event.y);
    if (distance < 5) {
      event.checkProximity();
    }
  });
});

// ✅ 优化后: 使用条件执行
let lastCheckTime = 0;

HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
  
  const now = Date.now();
  if (now - lastCheckTime < 500) return; // 每500ms检查一次
  lastCheckTime = now;
  
  $gameMap.events().forEach(event => {
    const distance = Math.abs(this.x - event.x) + Math.abs(this.y - event.y);
    if (distance < 5) {
      event.checkProximity();
    }
  });
}, {
  condition: () => $gameMap.events().length > 0 // 有事件时才执行
});
```

**优化2: 缓存计算结果**

```javascript
// ❌ 优化前: 重复计算
HookManager.regHook('Game_CharacterBase.prototype.isMoving', function(next) {
  const result = next();
  
  // 每次都计算复杂的特效
  if (result) {
    this.updateComplexEffect();
  }
  
  return result;
});

// ✅ 优化后: 缓存结果
const effectCache = new Map();

HookManager.regHook('Game_CharacterBase.prototype.isMoving', function(next) {
  const result = next();
  
  if (result) {
    const cacheKey = `${this.x}_${this.y}_${this.direction()}`;
    
    if (!effectCache.has(cacheKey)) {
      const effect = this.calculateComplexEffect();
      effectCache.set(cacheKey, effect);
      
      // 限制缓存大小
      if (effectCache.size > 100) {
        const firstKey = effectCache.keys().next().value;
        effectCache.delete(firstKey);
      }
    }
    
    this.applyEffect(effectCache.get(cacheKey));
  }
  
  return result;
});
```

**优化3: 延迟执行**

```javascript
// ❌ 优化前: 立即执行所有逻辑
HookManager.regHook('Game_Player.prototype.moveStraight', function(next, d) {
  next();
  
  // 所有逻辑都在移动后立即执行
  this.checkTerrain();
  this.updateFootsteps();
  this.checkEvents();
  this.updateCamera();
});

// ✅ 优化后: 分帧执行
const taskQueue = [];
let taskIndex = 0;

HookManager.regHook('Game_Player.prototype.moveStraight', function(next, d) {
  next();
  
  // 添加任务到队列
  taskQueue.push(
    () => this.checkTerrain(),
    () => this.updateFootsteps(),
    () => this.checkEvents(),
    () => this.updateCamera()
  );
});

// 每帧执行一个任务
HookManager.regHook('Scene_Map.prototype.update', function(next) {
  next();
  
  if (taskQueue.length > 0) {
    const task = taskQueue.shift();
    task();
  }
});
```

**优化4: 使用对象池**

```javascript
// ❌ 优化前: 频繁创建对象
HookManager.regHook('Sprite_Character.prototype.update', function(next) {
  next();
  
  // 每帧创建新对象
  const effect = {
    x: this.x,
    y: this.y,
    opacity: 255,
    scale: 1.0
  };
  
  this.applyEffect(effect);
});

// ✅ 优化后: 对象池
class EffectPool {
  constructor(size = 50) {
    this.pool = [];
    this.active = [];
    
    for (let i = 0; i < size; i++) {
      this.pool.push({
        x: 0,
        y: 0,
        opacity: 255,
        scale: 1.0
      });
    }
  }
  
  acquire() {
    if (this.pool.length > 0) {
      const obj = this.pool.pop();
      this.active.push(obj);
      return obj;
    }
    return null;
  }
  
  release(obj) {
    const index = this.active.indexOf(obj);
    if (index > -1) {
      this.active.splice(index, 1);
      this.pool.push(obj);
    }
  }
}

const effectPool = new EffectPool();

HookManager.regHook('Sprite_Character.prototype.update', function(next) {
  next();
  
  const effect = effectPool.acquire();
  if (effect) {
    effect.x = this.x;
    effect.y = this.y;
    effect.opacity = 255;
    effect.scale = 1.0;
    
    this.applyEffect(effect);
    
    // 使用完后归还
    effectPool.release(effect);
  }
});
```

● 效果对比

```javascript
// 性能对比工具
class PerformanceComparison {
  static compare(name, originalFunc, optimizedFunc, iterations = 1000) {
    console.log(`=== 性能对比: ${name} ===`);
    
    // 测试原始版本
    const startOriginal = performance.now();
    for (let i = 0; i < iterations; i++) {
      originalFunc();
    }
    const durationOriginal = performance.now() - startOriginal;
    
    // 测试优化版本
    const startOptimized = performance.now();
    for (let i = 0; i < iterations; i++) {
      optimizedFunc();
    }
    const durationOptimized = performance.now() - startOptimized;
    
    // 输出结果
    console.log(`原始版本: ${durationOriginal.toFixed(2)}ms`);
    console.log(`优化版本: ${durationOptimized.toFixed(2)}ms`);
    
    const improvement = ((durationOriginal - durationOptimized) / durationOriginal * 100);
    console.log(`性能提升: ${improvement.toFixed(2)}%`);
    
    if (improvement > 0) {
      console.log(`✓ 优化成功!`);
    } else {
      console.log(`❌ 优化无效或性能下降`);
    }
    
    return {
      original: durationOriginal,
      optimized: durationOptimized,
      improvement: improvement
    };
  }
}

// 使用示例
function testOptimization() {
  // 原始实现
  const original = () => {
    const events = $gameMap.events();
    events.forEach(event => {
      const distance = Math.abs($gamePlayer.x - event.x) + 
                      Math.abs($gamePlayer.y - event.y);
      if (distance < 5) {
        // 处理逻辑
      }
    });
  };
  
  // 优化实现
  const optimized = () => {
    const events = $gameMap.events();
    const px = $gamePlayer.x;
    const py = $gamePlayer.y;
    
    for (let i = 0; i < events.length; i++) {
      const event = events[i];
      const dx = px - event.x;
      const dy = py - event.y;
      
      // 使用平方距离避免 Math.abs
      if (dx * dx + dy * dy < 25) {
        // 处理逻辑
      }
    }
  };
  
  PerformanceComparison.compare('事件距离检测', original, optimized, 10000);
}

// testOptimization();
```

● 优化清单

```javascript
// 性能优化检查清单
const OptimizationChecklist = {
  // 1. 减少计算频率
  reduceFrequency: [
    '✓ 使用条件执行',
    '✓ 增加检查间隔',
    '✓ 只在必要时执行'
  ],
  
  // 2. 缓存结果
  caching: [
    '✓ 缓存计算结果',
    '✓ 缓存查询结果',
    '✓ 限制缓存大小'
  ],
  
  // 3. 优化算法
  algorithm: [
    '✓ 使用更高效的算法',
    '✓ 减少循环嵌套',
    '✓ 提前退出循环'
  ],
  
  // 4. 内存管理
  memory: [
    '✓ 使用对象池',
    '✓ 及时释放引用',
    '✓ 避免内存泄漏'
  ],
  
  // 5. 异步处理
  async: [
    '✓ 分帧执行',
    '✓ 延迟加载',
    '✓ 后台处理'
  ]
};

// 打印清单
function printOptimizationChecklist() {
  console.log('=== 性能优化清单 ===\n');
  
  Object.entries(OptimizationChecklist).forEach(([category, items]) => {
    console.log(`${category}:`);
    items.forEach(item => console.log(`  ${item}`));
    console.log('');
  });
}

// printOptimizationChecklist();
```

## 第七部分：其它

### 7.1 从原始插件迁移

● API 差异对比

**原始别名方式：**

```javascript
// 旧方式
const _Game_Player_update = Game_Player.prototype.update;
Game_Player.prototype.update = function() {
  _Game_Player_update.call(this);
  // 自定义逻辑
  this.customUpdate();
};
```

**HookManager 方式：**

```javascript
// 新方式
HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
  this.customUpdate();
}, {
  label: 'MyPlugin-PlayerUpdate'
});
```

**对比表：**

| 特性 | 原始别名 | HookManager |
|------|---------|-------------|
| 代码量 | 多 | 少 |
| 优先级控制 | 无 | 有 |
| 条件执行 | 手动 | 内置 |
| 性能监控 | 无 | 内置 |
| 解绑 | 困难 | 简单 |
| 冲突处理 | 手动 | 自动 |

---

● 迁移步骤

**步骤1：识别所有别名**

```javascript
// 查找模式
const _OriginalFunction = ClassName.prototype.methodName;
ClassName.prototype.methodName = function() { ... };
```

**步骤2：转换为钩子**

```javascript
// 原始代码
const _Game_Actor_changeHp = Game_Actor.prototype.changeHp;
Game_Actor.prototype.changeHp = function(value) {
  console.log('HP变化:', value);
  _Game_Actor_changeHp.call(this, value);
  this.checkLowHp();
};

// 转换后
HookManager.regHook('Game_Actor.prototype.changeHp', function(next, value) {
  console.log('HP变化:', value);
  next();
  this.checkLowHp();
}, {
  label: 'MyPlugin-HPChange'
});
```

**步骤3：处理多层别名**

```javascript
// 原始代码（多层嵌套）
const _Game_Player_update = Game_Player.prototype.update;
Game_Player.prototype.update = function() {
  _Game_Player_update.call(this);
  this.updateA();
};

const _Game_Player_update2 = Game_Player.prototype.update;
Game_Player.prototype.update = function() {
  _Game_Player_update2.call(this);
  this.updateB();
};

// 转换后（使用优先级）
HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
  this.updateA();
}, {
  priority: 100,
  label: 'MyPlugin-UpdateA'
});

HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
  this.updateB();
}, {
  priority: 50,
  label: 'MyPlugin-UpdateB'
});
```

**步骤4：迁移条件逻辑**

```javascript
// 原始代码
const _Scene_Map_update = Scene_Map.prototype.update;
Scene_Map.prototype.update = function() {
  _Scene_Map_update.call(this);
  
  if ($gameSwitches.value(10)) {
    this.updateWeather();
  }
};

// 转换后
HookManager.regHook('Scene_Map.prototype.update', function(next) {
  next();
  this.updateWeather();
}, {
  condition: () => $gameSwitches.value(10),
  label: 'MyPlugin-Weather'
});
```

---

● 兼容性说明

**向后兼容：**

```javascript
// HookManager 不影响原有别名
// 可以混合使用

// 原始别名仍然有效
const _Game_Player_update = Game_Player.prototype.update;
Game_Player.prototype.update = function() {
  _Game_Player_update.call(this);
  // ...
};

// 同时使用钩子
HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
  // ...
});

// 执行顺序：钩子 → 别名
```

**注意事项：**

```javascript
// ⚠️ 避免对同一函数同时使用别名和钩子
// 可能导致执行顺序混乱

// ❌ 不推荐
const _update = Game_Player.prototype.update;
Game_Player.prototype.update = function() {
  _update.call(this);
};

HookManager.regHook('Game_Player.prototype.update', hook);

// ✓ 推荐：统一使用钩子
HookManager.regHook('Game_Player.prototype.update', hook1);
HookManager.regHook('Game_Player.prototype.update', hook2);
```

---

### 7.2 从其他钩子系统迁移

● vs YEP 别名系统

**YEP 方式：**

```javascript
// YEP_CoreEngine 别名
Yanfly.Core.Game_Player_update = Game_Player.prototype.update;
Game_Player.prototype.update = function() {
  Yanfly.Core.Game_Player_update.call(this);
  // 自定义逻辑
};
```

**迁移到 HookManager：**

```javascript
HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
  // 自定义逻辑
}, {
  priority: 100, // YEP 通常优先级较高
  label: 'MyPlugin-PlayerUpdate'
});
```

**优先级映射：**

```javascript
// YEP 插件通常在核心层
const YEP_PRIORITY = 150;

// 如果需要在 YEP 之前执行
HookManager.regHook('someFunction', hook, {
  priority: YEP_PRIORITY + 50 // 200
});

// 如果需要在 YEP 之后执行
HookManager.regHook('someFunction', hook, {
  priority: YEP_PRIORITY - 50 // 100
});
```

---

● vs FOSSIL

**FOSSIL 方式：**

```javascript
// FOSSIL 钩子系统
FOSSIL.hookBefore('Game_Player', 'update', function() {
  // 在 update 之前执行
});

FOSSIL.hookAfter('Game_Player', 'update', function() {
  // 在 update 之后执行
});
```

**迁移到 HookManager：**

```javascript
// Before 钩子
HookManager.regHook('Game_Player.prototype.update', function(next) {
  // 在 update 之前执行
  console.log('before');
  
  next();
}, {
  priority: 200, // 高优先级，先执行
  label: 'MyPlugin-Before'
});

// After 钩子
HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
  
  // 在 update 之后执行
  console.log('after');
}, {
  priority: 50, // 低优先级，后执行
  label: 'MyPlugin-After'
});
```

---

● 迁移工具脚本

**自动转换工具：**

```javascript
class MigrationHelper {
  // 转换别名为钩子
  static convertAlias(code) {
    // 匹配模式: const _Func = Class.prototype.method;
    const aliasPattern = /const\s+(_\w+)\s+=\s+([\w.]+);/g;
    const overridePattern = /([\w.]+)\s+=\s+function\((.*?)\)\s*{[\s\S]*?_\w+\.call\(this(?:,\s*(.*?))?\);([\s\S]*?)}/g;
    
    let result = code;
    
    // 提取别名信息
    const aliases = [];
    let match;
    
    while ((match = aliasPattern.exec(code)) !== null) {
      aliases.push({
        varName: match[1],
        target: match[2]
      });
    }
    
    // 转换为钩子
    aliases.forEach(alias => {
      const hookCode = `
HookManager.regHook('${alias.target}', function(next, ...args) {
  next();
  // 自定义逻辑
}, {
  label: 'Converted-${alias.target.replace(/\./g, '-')}'
});`;
      
      console.log('转换:', alias.target);
      console.log(hookCode);
    });
    
    return result;
  }
  
  // 分析插件依赖
  static analyzeDependencies(pluginCode) {
    const dependencies = [];
    
    // 查找别名引用
    const aliasRefs = pluginCode.match(/const\s+_\w+\s+=\s+([\w.]+);/g);
    
    if (aliasRefs) {
      aliasRefs.forEach(ref => {
        const target = ref.match(/([\w.]+);/)[1];
        dependencies.push(target);
      });
    }
    
    return dependencies;
  }
  
  // 生成迁移报告
  static generateReport(pluginCode) {
    console.log('=== 迁移分析报告 ===\n');
    
    const deps = this.analyzeDependencies(pluginCode);
    
    console.log('发现别名数量:', deps.length);
    console.log('需要转换的函数:');
    deps.forEach((dep, i) => {
      console.log(`  ${i + 1}. ${dep}`);
    });
    
    console.log('\n推荐优先级分配:');
    deps.forEach(dep => {
      let priority = 100;
      
      if (dep.includes('Scene')) priority = 50;
      if (dep.includes('Window')) priority = 20;
      if (dep.includes('Sprite')) priority = 30;
      
      console.log(`  ${dep}: ${priority}`);
    });
  }
}

// 使用示例
const oldPluginCode = `
const _Game_Player_update = Game_Player.prototype.update;
Game_Player.prototype.update = function() {
  _Game_Player_update.call(this);
  this.customUpdate();
};

const _Game_Actor_changeHp = Game_Actor.prototype.changeHp;
Game_Actor.prototype.changeHp = function(value) {
  _Game_Actor_changeHp.call(this, value);
  this.checkLowHp();
};
`;

MigrationHelper.generateReport(oldPluginCode);
MigrationHelper.convertAlias(oldPluginCode);
```

**批量迁移脚本：**

```javascript
class BatchMigration {
  constructor() {
    this.conversions = [];
  }
  
  // 添加转换规则
  addConversion(oldPattern, newHook) {
    this.conversions.push({ oldPattern, newHook });
  }
  
  // 执行迁移
  migrate(pluginCode) {
    let result = pluginCode;
    
    this.conversions.forEach(({ oldPattern, newHook }) => {
      result = result.replace(oldPattern, newHook);
    });
    
    return result;
  }
}

// 使用
const migration = new BatchMigration();

// 定义转换规则
migration.addConversion(
  /const _Game_Player_update = Game_Player\.prototype\.update;[\s\S]*?};/,
  `HookManager.regHook('Game_Player.prototype.update', function(next) {
  next();
  this.customUpdate();
});`
);

// 执行迁移
const migratedCode = migration.migrate(oldPluginCode);
console.log(migratedCode);
```

### 7.3 故障排查清单

● 检查项列表

**1. 钩子未生效**

```javascript
// ✓ 检查路径是否正确
console.log(typeof Game_Player.prototype.update); // 应该是 'function'

// ✓ 检查钩子是否注册成功
console.log(HookManager.hooks.has('Game_Player.prototype.update'));

// ✓ 检查条件函数
HookManager.regHook('update', hook, {
  condition: () => {
    const result = $gameSwitches.value(10);
    console.log('条件结果:', result);
    return result;
  }
});
```

**2. 性能问题**

```javascript
// ✓ 启用性能监控
HookManager.globalOptions.enableProfiling = true;

// ✓ 检查钩子链深度
HookManager.hooks.forEach((data, key) => {
  if (data.chain.length > 5) {
    console.warn(`${key}: ${data.chain.length} 个钩子`);
  }
});

// ✓ 查看统计
setTimeout(() => HookManager.printStats(), 60000);
```

**3. 执行顺序错误**

```javascript
// ✓ 检查优先级
HookManager.hooks.get('update').chain.forEach(h => {
  console.log(`[${h.priority}] ${h.label}`);
});

// ✓ 调整优先级
HookManager.regHook('update', hook, {
  priority: 200 // 提高优先级
});
```

---

### 7.4 常见错误代码

**错误1: 路径不存在**

```javascript
// 错误信息
// TypeError: Cannot read property 'prototype' of undefined

// 原因
HookManager.regHook('NonExistent.prototype.method', hook);

// 解决
if (window.NonExistent) {
  HookManager.regHook('NonExistent.prototype.method', hook);
}
```

**错误2: 无限递归**

```javascript
// 错误代码
HookManager.regHook('update', function(next) {
  this.update(); // ❌ 导致无限递归
});

// 正确代码
HookManager.regHook('update', function(next) {
  next(); // ✓ 调用原始函数
});
```

**错误3: this 指向错误**

```javascript
// 错误代码
HookManager.regHook('update', () => {
  this.customMethod(); // ❌ this 指向错误
});

// 正确代码
HookManager.regHook('update', function(next) {
  this.customMethod(); // ✓ this 指向正确
});
```

---



### 7.5 FAQ

**Q: 钩子和别名有什么区别？**

A: 钩子提供了更好的管理、优先级控制和性能监控，推荐使用钩子。

**Q: 可以同时使用多个钩子吗？**

A: 可以，钩子会按优先级顺序执行。

**Q: 如何调试钩子？**

A: 使用 `label` 标识钩子，启用 `profile` 监控性能。

**Q: 钩子会影响性能吗？**

A: 影响很小，通常 < 0.1ms。使用条件函数和合理的优先级可以进一步优化。

**Q: 如何解决钩子冲突？**

A: 使用优先级控制执行顺序，使用标签识别钩子来源。

---

## 结语

本文档涵盖了 HookManager 的完整使用方法，从基础概念到高级技巧。

**快速参考：**
- 基础用法：第二部分
- 实战案例：第六部分
- API参考：第八部分
- 故障排查：第十部分

**获取帮助：**
- 查看示例代码
- 使用性能监控工具
- 参考故障排查清单

祝您使用愉快！
` ` ` `





