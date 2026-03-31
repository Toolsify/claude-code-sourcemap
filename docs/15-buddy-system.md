## 十五、AI 伴侣系统 (Buddy)

### 15.1 Buddy 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Buddy AI Companion System                     │
├─────────────────────────────────────────────────────────────────┤
│  数据模型                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │ Companion   │  │   Bones     │  │    Soul     │            │
│  │ (完整伴侣)  │  │ (外观骨架)  │  │ (个性灵魂)  │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
├─────────────────────────────────────────────────────────────────┤
│  生成机制                                                       │
│  userId + SALT → hashString() → mulberry32() → roll()          │
│  ├── rollRarity()   → common/uncommon/rare/epic/legendary      │
│  ├── rollSpecies()  → 鸭子/猫/狗等                             │
│  ├── rollStats()    → 智力/魅力/活力等属性                     │
│  └── rollAppearance() → 眼睛/帽子/闪亮                         │
├─────────────────────────────────────────────────────────────────┤
│  UI 渲染                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │  Companion  │  │  Floating   │  │  Sprite     │            │
│  │  Sprite     │  │  Bubble     │  │  Animation  │            │
│  │  (ASCII画)  │  │  (对话气泡) │  │  (表情动画) │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
├─────────────────────────────────────────────────────────────────┤
│  触发机制                                                       │
│  ├── /buddy 命令触发                                           │
│  ├── 启动时 teaser 通知                                        │
│  └── companionIntroText 对话注入                               │
└─────────────────────────────────────────────────────────────────┘
```

### 15.2 伴侣生成算法

```typescript
// buddy/companion.ts
const SALT = 'friend-2026-401';

export function roll(userId: string): Roll {
  // 确定性哈希
  const seed = hashString(userId + SALT);
  const rng = mulberry32(seed);
  
  // 稀有度（加权随机）
  const rarity = rollRarity(rng);  // common → legendary
  
  // 骨架生成
  const bones: CompanionBones = {
    rarity,
    species: pick(rng, SPECIES),    // 鸭子、猫、狗等
    eye: pick(rng, EYES),           // 眼睛样式
    hat: rarity === 'common' ? 'none' : pick(rng, HATS),
    shiny: rng() < 0.01,            // 1% 闪亮几率
    stats: rollStats(rng, rarity),  // 属性生成
  };
  
  return { bones, inspirationSeed: seed };
}

// 属性生成（一高一低，其余随机）
function rollStats(
  rng: () => number,
  rarity: Rarity,
): Record<StatName, number> {
  const floor = RARITY_FLOOR[rarity];  // 稀有度下限
  const peak = pick(rng, STAT_NAMES);  // 高属性
  const dump = pick(rng, STAT_NAMES);  // 低属性
  
  const stats = {} as Record<StatName, number>;
  for (const name of STAT_NAMES) {
    if (name === peak) {
      stats[name] = Math.min(100, floor + 50 + Math.floor(rng() * 30));
    } else if (name === dump) {
      stats[name] = Math.max(1, floor - 10 + Math.floor(rng() * 15));
    } else {
      stats[name] = floor + Math.floor(rng() * 40);
    }
  }
  return stats;
}
```

### 15.3 UI 渲染与交互

```typescript
// buddy/CompanionSprite.tsx
export function CompanionSprite({ companion, state }: Props) {
  const cols = useTerminalSize().columns;
  
  // 终端宽度自适应
  if (cols < MIN_COLS_FOR_FULL_SPRITE) {
    return <CompactView companion={companion} />;
  }
  
  // 完整 ASCII 精灵
  return (
    <Box flexDirection="column">
      <Text color={RARITY_COLORS[companion.rarity]}>
        {renderSprite(SPRITES[companion.species], state.frame)}
      </Text>
      {state.showBubble && (
        <CompanionFloatingBubble
          text={state.bubbleText}
          emotion={state.emotion}
        />
      )}
    </Box>
  );
}

// 5 帧动画
const SPRITES = {
  duck: [
    '    __\n   (  >\n    )/\n   (_',
    // ... 其他帧
  ],
  // ...
};
```

---

## 十六、查询配置与 Token 预算
