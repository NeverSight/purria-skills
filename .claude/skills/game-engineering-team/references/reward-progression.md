# Reward & Progression Systems

Complete reward systems, economy balancing, progression curves, and virtual economy design.

---

## Economy Architecture

### Faucet-Sink Model

Every healthy game economy is a flow system: resources enter through **faucets** (quest rewards, loot, daily bonuses) and exit through **sinks** (purchases, crafting, upgrades). Equilibrium prevents both inflation and player frustration.

```typescript
interface EconomyConfig {
  currencies: CurrencyDefinition[];
  faucets: FaucetDefinition[];
  sinks: SinkDefinition[];
  inflationTargetPerDay: number; // e.g., 0.02 for 2% growth
}

interface CurrencyDefinition {
  id: string;
  name: string;
  type: 'soft' | 'hard' | 'seasonal' | 'energy';
  cap?: number;
  decayRate?: number; // per day, for seasonal currencies
}

interface FaucetDefinition {
  id: string;
  currencyId: string;
  source: 'quest' | 'daily' | 'achievement' | 'drop' | 'purchase';
  amountRange: [number, number];
  frequency: 'once' | 'daily' | 'per_action' | 'weekly';
}

interface SinkDefinition {
  id: string;
  currencyId: string;
  target: 'upgrade' | 'cosmetic' | 'consumable' | 'crafting' | 'fee';
  cost: number;
  scalingFactor?: number;
}

class EconomySimulator {
  private balances: Map<string, number[]> = new Map();
  
  constructor(private config: EconomyConfig) {}
  
  simulateDays(days: number, playerCount: number): SimulationResult {
    // Initialize player balances
    for (const currency of this.config.currencies) {
      this.balances.set(
        currency.id,
        Array(playerCount).fill(0)
      );
    }
    
    const dailySnapshots: DailySnapshot[] = [];
    
    for (let day = 0; day < days; day++) {
      // Apply faucets
      for (const faucet of this.config.faucets) {
        if (faucet.frequency === 'once' && day > 0) continue;
        if (faucet.frequency === 'weekly' && day % 7 !== 0) continue;
        
        const balances = this.balances.get(faucet.currencyId)!;
        for (let p = 0; p < playerCount; p++) {
          const amount = this.randomInRange(faucet.amountRange);
          balances[p] += amount;
        }
      }
      
      // Apply sinks (probabilistic spending)
      for (const sink of this.config.sinks) {
        const balances = this.balances.get(sink.currencyId)!;
        for (let p = 0; p < playerCount; p++) {
          if (balances[p] >= sink.cost && Math.random() < 0.3) {
            balances[p] -= sink.cost;
          }
        }
      }
      
      // Apply currency decay
      for (const currency of this.config.currencies) {
        if (currency.decayRate) {
          const balances = this.balances.get(currency.id)!;
          for (let p = 0; p < playerCount; p++) {
            balances[p] = Math.floor(balances[p] * (1 - currency.decayRate));
          }
        }
      }
      
      // Record snapshot
      dailySnapshots.push(this.takeSnapshot(day));
    }
    
    return { snapshots: dailySnapshots };
  }
  
  private takeSnapshot(day: number): DailySnapshot {
    const currencies: Record<string, CurrencySnapshot> = {};
    
    for (const [id, balances] of this.balances) {
      const sorted = [...balances].sort((a, b) => a - b);
      currencies[id] = {
        mean: sorted.reduce((a, b) => a + b, 0) / sorted.length,
        median: sorted[Math.floor(sorted.length / 2)],
        p10: sorted[Math.floor(sorted.length * 0.1)],
        p90: sorted[Math.floor(sorted.length * 0.9)],
        total: sorted.reduce((a, b) => a + b, 0),
        gini: this.calculateGini(sorted),
      };
    }
    
    return { day, currencies };
  }
  
  private calculateGini(sorted: number[]): number {
    const n = sorted.length;
    const sum = sorted.reduce((a, b) => a + b, 0);
    if (sum === 0) return 0;
    
    let cumulativeSum = 0;
    let giniNumerator = 0;
    
    for (let i = 0; i < n; i++) {
      cumulativeSum += sorted[i];
      giniNumerator += (2 * (i + 1) - n - 1) * sorted[i];
    }
    
    return giniNumerator / (n * sum);
  }
  
  private randomInRange([min, max]: [number, number]): number {
    return Math.floor(Math.random() * (max - min + 1)) + min;
  }
}

interface SimulationResult {
  snapshots: DailySnapshot[];
}

interface DailySnapshot {
  day: number;
  currencies: Record<string, CurrencySnapshot>;
}

interface CurrencySnapshot {
  mean: number;
  median: number;
  p10: number;
  p90: number;
  total: number;
  gini: number; // 0 = perfect equality, 1 = total inequality
}
```

---

## Progression Curves

### Comprehensive XP System

```typescript
interface LevelConfig {
  maxLevel: number;
  curveType: 'linear' | 'quadratic' | 'exponential' | 'polynomial' | 'custom';
  baseXP: number;
  growthRate: number;
  customCurve?: (level: number) => number;
}

class LevelSystem {
  private xpTable: number[];
  private cumulativeXP: number[];
  
  constructor(private config: LevelConfig) {
    this.xpTable = this.buildXPTable();
    this.cumulativeXP = this.buildCumulativeTable();
  }
  
  private buildXPTable(): number[] {
    const table: number[] = [0]; // Level 1 needs 0 XP
    
    for (let level = 2; level <= this.config.maxLevel; level++) {
      let xpNeeded: number;
      
      switch (this.config.curveType) {
        case 'linear':
          xpNeeded = this.config.baseXP * level;
          break;
        case 'quadratic':
          xpNeeded = this.config.baseXP * level * level;
          break;
        case 'exponential':
          xpNeeded = Math.floor(this.config.baseXP * Math.pow(this.config.growthRate, level - 1));
          break;
        case 'polynomial':
          xpNeeded = Math.floor(this.config.baseXP * Math.pow(level, this.config.growthRate));
          break;
        case 'custom':
          xpNeeded = this.config.customCurve!(level);
          break;
      }
      
      table.push(Math.floor(xpNeeded));
    }
    
    return table;
  }
  
  private buildCumulativeTable(): number[] {
    const cumulative: number[] = [0];
    for (let i = 1; i < this.xpTable.length; i++) {
      cumulative.push(cumulative[i - 1] + this.xpTable[i]);
    }
    return cumulative;
  }
  
  getLevelFromXP(totalXP: number): number {
    for (let i = this.cumulativeXP.length - 1; i >= 0; i--) {
      if (totalXP >= this.cumulativeXP[i]) return i + 1;
    }
    return 1;
  }
  
  getProgress(totalXP: number): LevelProgress {
    const level = this.getLevelFromXP(totalXP);
    const xpIntoLevel = totalXP - this.cumulativeXP[level - 1];
    const xpForNextLevel = level < this.config.maxLevel ? this.xpTable[level] : 0;
    
    return {
      level,
      currentXP: xpIntoLevel,
      requiredXP: xpForNextLevel,
      totalXP,
      percentage: xpForNextLevel > 0 ? (xpIntoLevel / xpForNextLevel) * 100 : 100,
      isMaxLevel: level >= this.config.maxLevel,
    };
  }
  
  getXPTable(): { level: number; xpNeeded: number; cumulative: number }[] {
    return this.xpTable.map((xp, i) => ({
      level: i + 1,
      xpNeeded: xp,
      cumulative: this.cumulativeXP[i],
    }));
  }
  
  estimateTimeToLevel(
    currentXP: number,
    targetLevel: number,
    xpPerHour: number
  ): number {
    const targetXP = this.cumulativeXP[targetLevel - 1];
    const xpNeeded = Math.max(0, targetXP - currentXP);
    return xpNeeded / xpPerHour;
  }
}

interface LevelProgress {
  level: number;
  currentXP: number;
  requiredXP: number;
  totalXP: number;
  percentage: number;
  isMaxLevel: boolean;
}

// Presets for common game systems
const LEVEL_PRESETS = {
  // Gentle curve for casual games (100 XP to ~5000 XP per level)
  casual: {
    maxLevel: 50,
    curveType: 'polynomial' as const,
    baseXP: 100,
    growthRate: 1.3,
  },
  
  // Standard RPG curve
  rpg: {
    maxLevel: 100,
    curveType: 'polynomial' as const,
    baseXP: 50,
    growthRate: 1.8,
  },
  
  // Companion bonding (short, sweet)
  bonding: {
    maxLevel: 25,
    curveType: 'quadratic' as const,
    baseXP: 30,
    growthRate: 1,
  },
  
  // Seasonal prestige
  seasonal: {
    maxLevel: 30,
    curveType: 'linear' as const,
    baseXP: 200,
    growthRate: 1,
  },
};
```

---

## Achievement System

### Flexible Achievement Engine

```typescript
type AchievementCategory = 'farming' | 'betting' | 'social' | 'collection' | 'mastery' | 'secret';

interface AchievementDefinition {
  id: string;
  name: string;
  description: string;
  category: AchievementCategory;
  icon: string;
  tiers?: AchievementTier[];
  condition: AchievementCondition;
  reward: AchievementReward;
  hidden?: boolean; // Secret achievements
}

interface AchievementTier {
  name: string;
  threshold: number;
  reward: AchievementReward;
}

type AchievementCondition =
  | { type: 'counter'; event: string; target: number }
  | { type: 'threshold'; stat: string; value: number }
  | { type: 'composite'; operator: 'and' | 'or'; conditions: AchievementCondition[] }
  | { type: 'streak'; event: string; days: number }
  | { type: 'collection'; items: string[]; requiredCount?: number };

interface AchievementReward {
  coins?: number;
  xp?: number;
  items?: { id: string; quantity: number }[];
  title?: string;
  badge?: string;
}

interface PlayerAchievementProgress {
  achievementId: string;
  currentValue: number;
  currentTier: number;
  completed: boolean;
  completedAt?: Date;
}

class AchievementEngine {
  private definitions: Map<string, AchievementDefinition> = new Map();
  private progress: Map<string, PlayerAchievementProgress> = new Map();
  private listeners: ((achievement: AchievementDefinition, tier?: number) => void)[] = [];
  
  register(definition: AchievementDefinition): void {
    this.definitions.set(definition.id, definition);
  }
  
  onUnlock(listener: (achievement: AchievementDefinition, tier?: number) => void): () => void {
    this.listeners.push(listener);
    return () => {
      const idx = this.listeners.indexOf(listener);
      if (idx >= 0) this.listeners.splice(idx, 1);
    };
  }
  
  incrementCounter(event: string, amount: number = 1): AchievementDefinition[] {
    const unlocked: AchievementDefinition[] = [];
    
    for (const [id, def] of this.definitions) {
      if (!this.matchesEvent(def.condition, event)) continue;
      
      const progress = this.getOrCreateProgress(id);
      if (progress.completed) continue;
      
      progress.currentValue += amount;
      
      // Check tiers
      if (def.tiers) {
        for (let i = progress.currentTier; i < def.tiers.length; i++) {
          if (progress.currentValue >= def.tiers[i].threshold) {
            progress.currentTier = i + 1;
            this.notifyListeners(def, i);
            
            if (i === def.tiers.length - 1) {
              progress.completed = true;
              progress.completedAt = new Date();
              unlocked.push(def);
            }
          }
        }
      } else {
        const target = this.getTarget(def.condition);
        if (target && progress.currentValue >= target) {
          progress.completed = true;
          progress.completedAt = new Date();
          unlocked.push(def);
          this.notifyListeners(def);
        }
      }
    }
    
    return unlocked;
  }
  
  checkStatThreshold(stat: string, value: number): AchievementDefinition[] {
    const unlocked: AchievementDefinition[] = [];
    
    for (const [id, def] of this.definitions) {
      if (def.condition.type !== 'threshold') continue;
      if (def.condition.stat !== stat) continue;
      
      const progress = this.getOrCreateProgress(id);
      if (progress.completed) continue;
      
      progress.currentValue = value;
      
      if (value >= def.condition.value) {
        progress.completed = true;
        progress.completedAt = new Date();
        unlocked.push(def);
        this.notifyListeners(def);
      }
    }
    
    return unlocked;
  }
  
  getProgress(achievementId: string): PlayerAchievementProgress | undefined {
    return this.progress.get(achievementId);
  }
  
  getAllProgress(): PlayerAchievementProgress[] {
    return Array.from(this.progress.values());
  }
  
  getCompletionRate(): number {
    const total = this.definitions.size;
    const completed = Array.from(this.progress.values()).filter(p => p.completed).length;
    return total > 0 ? completed / total : 0;
  }
  
  private getOrCreateProgress(achievementId: string): PlayerAchievementProgress {
    let progress = this.progress.get(achievementId);
    if (!progress) {
      progress = {
        achievementId,
        currentValue: 0,
        currentTier: 0,
        completed: false,
      };
      this.progress.set(achievementId, progress);
    }
    return progress;
  }
  
  private matchesEvent(condition: AchievementCondition, event: string): boolean {
    if (condition.type === 'counter') return condition.event === event;
    if (condition.type === 'streak') return condition.event === event;
    if (condition.type === 'composite') {
      return condition.conditions.some(c => this.matchesEvent(c, event));
    }
    return false;
  }
  
  private getTarget(condition: AchievementCondition): number | null {
    if (condition.type === 'counter') return condition.target;
    if (condition.type === 'threshold') return condition.value;
    if (condition.type === 'streak') return condition.days;
    return null;
  }
  
  private notifyListeners(def: AchievementDefinition, tier?: number): void {
    this.listeners.forEach(l => l(def, tier));
  }
}

// Example: Farming achievements
const FARMING_ACHIEVEMENTS: AchievementDefinition[] = [
  {
    id: 'first_harvest',
    name: 'Green Thumb',
    description: 'Harvest your first tulip',
    category: 'farming',
    icon: '🌷',
    condition: { type: 'counter', event: 'tulip_harvested', target: 1 },
    reward: { coins: 50, xp: 25 },
  },
  {
    id: 'harvest_master',
    name: 'Harvest Master',
    description: 'Harvest tulips across many seasons',
    category: 'farming',
    icon: '🌻',
    tiers: [
      { name: 'Bronze', threshold: 10, reward: { coins: 100 } },
      { name: 'Silver', threshold: 50, reward: { coins: 500, xp: 200 } },
      { name: 'Gold', threshold: 200, reward: { coins: 2000, xp: 1000, title: 'Master Gardener' } },
      { name: 'Diamond', threshold: 1000, reward: { coins: 10000, xp: 5000, badge: 'diamond_harvester' } },
    ],
    condition: { type: 'counter', event: 'tulip_harvested', target: 1000 },
    reward: { coins: 10000, xp: 5000 },
  },
  {
    id: 'lucky_streak',
    name: 'Lady Luck',
    description: 'Win bets 7 days in a row',
    category: 'betting',
    icon: '🎰',
    condition: { type: 'streak', event: 'bet_won', days: 7 },
    reward: { coins: 1000, title: 'Lucky Star' },
  },
];
```

---

## Daily Reward / Login Streak System

```typescript
interface DailyRewardConfig {
  cycle: DailyRewardDay[];
  streakBonusMultiplier: number; // e.g., 1.5 for 50% bonus
  maxStreakBonus: number; // Cap the multiplier
  missedDayGracePeriod: number; // Hours before streak resets
}

interface DailyRewardDay {
  day: number;
  rewards: AchievementReward;
  isMilestone: boolean;
}

interface PlayerDailyRewardState {
  lastClaimDate: string | null; // ISO date
  currentStreak: number;
  cyclePosition: number;
  totalDaysClaimed: number;
}

class DailyRewardSystem {
  constructor(private config: DailyRewardConfig) {}
  
  canClaim(state: PlayerDailyRewardState, now: Date = new Date()): boolean {
    if (!state.lastClaimDate) return true;
    
    const lastClaim = new Date(state.lastClaimDate);
    const hoursSinceClaim = (now.getTime() - lastClaim.getTime()) / (1000 * 60 * 60);
    
    // Must wait at least 20 hours
    if (hoursSinceClaim < 20) return false;
    
    return true;
  }
  
  claim(state: PlayerDailyRewardState, now: Date = new Date()): ClaimResult {
    if (!this.canClaim(state, now)) {
      return { success: false, reason: 'Too early to claim' };
    }
    
    // Check streak continuation
    const streakBroken = this.isStreakBroken(state, now);
    
    if (streakBroken) {
      state.currentStreak = 0;
      state.cyclePosition = 0;
    }
    
    state.currentStreak++;
    state.totalDaysClaimed++;
    state.lastClaimDate = now.toISOString();
    
    const dayConfig = this.config.cycle[state.cyclePosition];
    state.cyclePosition = (state.cyclePosition + 1) % this.config.cycle.length;
    
    // Calculate streak multiplier
    const streakMultiplier = Math.min(
      1 + (state.currentStreak - 1) * (this.config.streakBonusMultiplier - 1) / 7,
      this.config.maxStreakBonus
    );
    
    // Apply multiplier to rewards
    const adjustedRewards: AchievementReward = {
      ...dayConfig.rewards,
      coins: dayConfig.rewards.coins 
        ? Math.floor(dayConfig.rewards.coins * streakMultiplier)
        : undefined,
      xp: dayConfig.rewards.xp
        ? Math.floor(dayConfig.rewards.xp * streakMultiplier)
        : undefined,
    };
    
    return {
      success: true,
      rewards: adjustedRewards,
      streakDay: state.currentStreak,
      isMilestone: dayConfig.isMilestone,
      streakMultiplier,
      streakBroken,
    };
  }
  
  private isStreakBroken(state: PlayerDailyRewardState, now: Date): boolean {
    if (!state.lastClaimDate) return false;
    
    const lastClaim = new Date(state.lastClaimDate);
    const hoursSinceClaim = (now.getTime() - lastClaim.getTime()) / (1000 * 60 * 60);
    
    return hoursSinceClaim > 24 + this.config.missedDayGracePeriod;
  }
  
  getUpcomingRewards(state: PlayerDailyRewardState, count: number): DailyRewardDay[] {
    const upcoming: DailyRewardDay[] = [];
    let pos = state.cyclePosition;
    
    for (let i = 0; i < count; i++) {
      upcoming.push(this.config.cycle[pos]);
      pos = (pos + 1) % this.config.cycle.length;
    }
    
    return upcoming;
  }
}

interface ClaimResult {
  success: boolean;
  reason?: string;
  rewards?: AchievementReward;
  streakDay?: number;
  isMilestone?: boolean;
  streakMultiplier?: number;
  streakBroken?: boolean;
}

// Example: 7-day reward cycle
const WEEKLY_REWARDS: DailyRewardConfig = {
  cycle: [
    { day: 1, rewards: { coins: 100 }, isMilestone: false },
    { day: 2, rewards: { coins: 150 }, isMilestone: false },
    { day: 3, rewards: { coins: 200, xp: 50 }, isMilestone: false },
    { day: 4, rewards: { coins: 250 }, isMilestone: false },
    { day: 5, rewards: { coins: 300, xp: 100 }, isMilestone: false },
    { day: 6, rewards: { coins: 400 }, isMilestone: false },
    { day: 7, rewards: { coins: 1000, xp: 500, items: [{ id: 'rare_seed', quantity: 1 }] }, isMilestone: true },
  ],
  streakBonusMultiplier: 1.5,
  maxStreakBonus: 3.0,
  missedDayGracePeriod: 8,
};
```

---

## Loot Table System

```typescript
interface LootTable {
  id: string;
  entries: LootEntry[];
  guaranteedDrops?: string[];
  maxDrops?: number;
}

interface LootEntry {
  itemId: string;
  weight: number;
  minQuantity: number;
  maxQuantity: number;
  rarity: 'common' | 'uncommon' | 'rare' | 'epic' | 'legendary';
  conditions?: LootCondition[];
}

type LootCondition =
  | { type: 'min_level'; level: number }
  | { type: 'season'; season: string }
  | { type: 'luck_bonus'; multiplier: number };

interface LootDrop {
  itemId: string;
  quantity: number;
  rarity: string;
}

class LootTableEngine {
  constructor(private rng: { next(): number; nextInt(min: number, max: number): number }) {}
  
  roll(table: LootTable, context?: { level?: number; season?: string; luckBonus?: number }): LootDrop[] {
    const drops: LootDrop[] = [];
    
    // Add guaranteed drops
    if (table.guaranteedDrops) {
      for (const itemId of table.guaranteedDrops) {
        drops.push({ itemId, quantity: 1, rarity: 'common' });
      }
    }
    
    // Filter eligible entries
    const eligible = table.entries.filter(entry => 
      this.meetsConditions(entry, context)
    );
    
    // Apply luck bonus to weights
    const adjustedEntries = eligible.map(entry => {
      let weight = entry.weight;
      if (context?.luckBonus && entry.rarity !== 'common') {
        weight *= context.luckBonus;
      }
      return { ...entry, weight };
    });
    
    // Calculate total weight
    const totalWeight = adjustedEntries.reduce((sum, e) => sum + e.weight, 0);
    
    // Roll for each drop slot
    const maxRolls = table.maxDrops || 3;
    for (let i = 0; i < maxRolls; i++) {
      let roll = this.rng.next() * totalWeight;
      
      for (const entry of adjustedEntries) {
        roll -= entry.weight;
        if (roll <= 0) {
          const quantity = this.rng.nextInt(entry.minQuantity, entry.maxQuantity);
          drops.push({
            itemId: entry.itemId,
            quantity,
            rarity: entry.rarity,
          });
          break;
        }
      }
    }
    
    return drops;
  }
  
  // Pity system: increase rare drop rates after N misses
  rollWithPity(
    table: LootTable,
    pityCounter: number,
    pityThreshold: number = 50,
    context?: { level?: number; season?: string; luckBonus?: number }
  ): { drops: LootDrop[]; newPityCounter: number } {
    const luckBonus = 1 + Math.floor(pityCounter / pityThreshold) * 0.5;
    const adjustedContext = { ...context, luckBonus: (context?.luckBonus || 1) * luckBonus };
    
    const drops = this.roll(table, adjustedContext);
    
    const hasRare = drops.some(d => 
      ['rare', 'epic', 'legendary'].includes(d.rarity)
    );
    
    return {
      drops,
      newPityCounter: hasRare ? 0 : pityCounter + 1,
    };
  }
  
  private meetsConditions(entry: LootEntry, context?: any): boolean {
    if (!entry.conditions) return true;
    
    return entry.conditions.every(condition => {
      switch (condition.type) {
        case 'min_level':
          return (context?.level || 0) >= condition.level;
        case 'season':
          return context?.season === condition.season;
        default:
          return true;
      }
    });
  }
}

// Example: Harvest loot table
const HARVEST_LOOT: LootTable = {
  id: 'harvest_rewards',
  guaranteedDrops: ['harvest_xp_token'],
  maxDrops: 3,
  entries: [
    { itemId: 'common_seeds', weight: 40, minQuantity: 1, maxQuantity: 5, rarity: 'common' },
    { itemId: 'fertilizer', weight: 25, minQuantity: 1, maxQuantity: 3, rarity: 'common' },
    { itemId: 'rare_seeds', weight: 15, minQuantity: 1, maxQuantity: 2, rarity: 'uncommon' },
    { itemId: 'golden_water', weight: 10, minQuantity: 1, maxQuantity: 1, rarity: 'rare' },
    { itemId: 'simulin_treat', weight: 7, minQuantity: 1, maxQuantity: 1, rarity: 'epic' },
    { itemId: 'legendary_bloom', weight: 3, minQuantity: 1, maxQuantity: 1, rarity: 'legendary' },
  ],
};
```

---

## Battle Pass / Season Pass

```typescript
interface SeasonPass {
  id: string;
  name: string;
  durationDays: number;
  maxTier: number;
  xpPerTier: number;
  freeTierRewards: Map<number, AchievementReward>;
  premiumTierRewards: Map<number, AchievementReward>;
}

interface PlayerSeasonPassState {
  passId: string;
  isPremium: boolean;
  currentTier: number;
  currentXP: number;
  claimedFree: Set<number>;
  claimedPremium: Set<number>;
}

class SeasonPassSystem {
  constructor(private pass: SeasonPass) {}
  
  addXP(state: PlayerSeasonPassState, xp: number): TierUpResult {
    state.currentXP += xp;
    const previousTier = state.currentTier;
    
    while (
      state.currentXP >= this.pass.xpPerTier &&
      state.currentTier < this.pass.maxTier
    ) {
      state.currentXP -= this.pass.xpPerTier;
      state.currentTier++;
    }
    
    // Cap XP at max tier
    if (state.currentTier >= this.pass.maxTier) {
      state.currentXP = 0;
    }
    
    return {
      previousTier,
      newTier: state.currentTier,
      tiersGained: state.currentTier - previousTier,
      unclaimedRewards: this.getUnclaimedRewards(state),
    };
  }
  
  claimReward(
    state: PlayerSeasonPassState,
    tier: number,
    track: 'free' | 'premium'
  ): AchievementReward | null {
    if (tier > state.currentTier) return null;
    
    if (track === 'premium' && !state.isPremium) return null;
    
    const rewardsMap = track === 'free'
      ? this.pass.freeTierRewards
      : this.pass.premiumTierRewards;
    
    const claimedSet = track === 'free'
      ? state.claimedFree
      : state.claimedPremium;
    
    if (claimedSet.has(tier)) return null;
    
    const reward = rewardsMap.get(tier);
    if (!reward) return null;
    
    claimedSet.add(tier);
    return reward;
  }
  
  getUnclaimedRewards(state: PlayerSeasonPassState): { tier: number; track: string }[] {
    const unclaimed: { tier: number; track: string }[] = [];
    
    for (let tier = 1; tier <= state.currentTier; tier++) {
      if (this.pass.freeTierRewards.has(tier) && !state.claimedFree.has(tier)) {
        unclaimed.push({ tier, track: 'free' });
      }
      if (
        state.isPremium &&
        this.pass.premiumTierRewards.has(tier) &&
        !state.claimedPremium.has(tier)
      ) {
        unclaimed.push({ tier, track: 'premium' });
      }
    }
    
    return unclaimed;
  }
  
  getProgressToNextTier(state: PlayerSeasonPassState): number {
    if (state.currentTier >= this.pass.maxTier) return 100;
    return (state.currentXP / this.pass.xpPerTier) * 100;
  }
}

interface TierUpResult {
  previousTier: number;
  newTier: number;
  tiersGained: number;
  unclaimedRewards: { tier: number; track: string }[];
}
```

---

## Anti-Inflation Mechanisms

```typescript
class EconomyRegulator {
  private metrics: EconomyMetrics = {
    totalCurrencyInCirculation: 0,
    dailyFaucetTotal: 0,
    dailySinkTotal: 0,
    averagePlayerBalance: 0,
    medianPlayerBalance: 0,
    priceIndex: 100,
  };
  
  // Dynamic pricing: adjust costs based on inflation
  adjustPrice(basePrice: number): number {
    const inflationFactor = this.metrics.priceIndex / 100;
    return Math.floor(basePrice * inflationFactor);
  }
  
  // Tax on transactions (currency sink)
  calculateTax(amount: number, transactionType: string): number {
    const taxRates: Record<string, number> = {
      trade: 0.05,      // 5% on player-to-player trades
      auction: 0.10,    // 10% on auction house
      withdraw: 0.02,   // 2% on bank withdrawals
    };
    
    return Math.floor(amount * (taxRates[transactionType] || 0));
  }
  
  // Diminishing returns on repeated farming
  calculateDiminishingReward(
    baseReward: number,
    timesCompleted: number,
    decayRate: number = 0.1
  ): number {
    const multiplier = Math.max(0.1, 1 - decayRate * Math.log2(timesCompleted + 1));
    return Math.floor(baseReward * multiplier);
  }
  
  // Limited-time currency that prevents hoarding
  decayCurrency(balance: number, daysSinceEarned: number, halfLife: number): number {
    return Math.floor(balance * Math.pow(0.5, daysSinceEarned / halfLife));
  }
  
  updateMetrics(newMetrics: Partial<EconomyMetrics>): void {
    Object.assign(this.metrics, newMetrics);
    
    // Auto-adjust price index
    const netFlow = this.metrics.dailyFaucetTotal - this.metrics.dailySinkTotal;
    if (netFlow > 0) {
      this.metrics.priceIndex += 0.5; // Gentle inflation adjustment
    } else if (netFlow < 0) {
      this.metrics.priceIndex = Math.max(80, this.metrics.priceIndex - 0.3);
    }
  }
}

interface EconomyMetrics {
  totalCurrencyInCirculation: number;
  dailyFaucetTotal: number;
  dailySinkTotal: number;
  averagePlayerBalance: number;
  medianPlayerBalance: number;
  priceIndex: number;
}
```

---

*Reward & Progression Reference — Game Engineering Team*
