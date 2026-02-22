# Architecture & Code Quality

Code organization, refactoring patterns, documentation standards, and engineering practices.

---

## Project Structure

### Monorepo Layout

```
purria/
├── apps/
│   ├── web/                    # Main web application
│   │   ├── src/
│   │   │   ├── routes/         # TanStack Router pages
│   │   │   ├── components/     # UI components
│   │   │   ├── features/       # Feature modules
│   │   │   ├── hooks/          # Custom React hooks
│   │   │   ├── stores/         # Zustand stores
│   │   │   ├── lib/            # Utilities
│   │   │   └── styles/         # Global styles
│   │   ├── public/
│   │   └── package.json
│   │
│   └── api/                    # Backend API
│       ├── src/
│       │   ├── routes/         # Hono route handlers
│       │   ├── services/       # Business logic
│       │   ├── middleware/      # Auth, rate limiting
│       │   ├── db/             # Drizzle schema & queries
│       │   └── lib/            # Shared utilities
│       └── package.json
│
├── packages/
│   ├── game-engine/            # Core game logic (platform-agnostic)
│   │   ├── src/
│   │   │   ├── systems/        # Game systems (betting, farming, etc.)
│   │   │   ├── entities/       # Entity definitions
│   │   │   ├── events/         # Event bus & types
│   │   │   ├── math/           # RNG, probability, curves
│   │   │   └── types/          # Shared types
│   │   └── package.json
│   │
│   ├── ui/                     # Shared UI component library
│   │   ├── src/
│   │   │   ├── primitives/     # Button, Input, Dialog
│   │   │   ├── game/           # GameCard, HexGrid, PotDisplay
│   │   │   ├── feedback/       # Toast, Particles, Announcer
│   │   │   └── layout/         # GameHUD, ResponsiveContainer
│   │   └── package.json
│   │
│   ├── shared/                 # Shared types and utilities
│   │   ├── src/
│   │   │   ├── types/          # TypeScript types
│   │   │   ├── schemas/        # Zod schemas
│   │   │   ├── constants/      # Game constants
│   │   │   └── utils/          # Pure utility functions
│   │   └── package.json
│   │
│   └── config/                 # Shared configs
│       ├── tsconfig/
│       ├── eslint/
│       └── tailwind/
│
├── drizzle/                    # Database migrations
├── turbo.json
├── package.json
└── pnpm-workspace.yaml
```

### Feature Module Structure

Each feature is self-contained with its own components, hooks, and logic:

```
features/
├── betting/
│   ├── components/
│   │   ├── BetPanel.tsx
│   │   ├── PotDisplay.tsx
│   │   ├── BetButton.tsx
│   │   └── BetResultModal.tsx
│   ├── hooks/
│   │   ├── useBetting.ts
│   │   ├── usePotAnimation.ts
│   │   └── useBetHistory.ts
│   ├── stores/
│   │   └── bettingStore.ts
│   ├── utils/
│   │   └── betCalculations.ts
│   ├── types.ts
│   └── index.ts                # Public API barrel export
│
├── farming/
│   ├── components/
│   ├── hooks/
│   ├── stores/
│   └── index.ts
│
└── simulin/
    ├── components/
    ├── hooks/
    ├── stores/
    └── index.ts
```

### Barrel Export Convention

```typescript
// features/betting/index.ts
export { BetPanel } from './components/BetPanel';
export { PotDisplay } from './components/PotDisplay';
export { useBetting } from './hooks/useBetting';
export type { BetType, BetResult } from './types';

// Usage elsewhere:
import { BetPanel, useBetting } from '@/features/betting';
```

---

## Naming Conventions

### Files and Directories

| Pattern | Convention | Example |
|---------|-----------|---------|
| React components | PascalCase | `BetPanel.tsx` |
| Hooks | camelCase with `use` prefix | `useBetting.ts` |
| Stores | camelCase with `Store` suffix | `bettingStore.ts` |
| Utilities | camelCase | `betCalculations.ts` |
| Types | camelCase or PascalCase | `types.ts` |
| Constants | camelCase | `gameConstants.ts` |
| Test files | Same name with `.test` | `BetPanel.test.tsx` |

### Code Identifiers

```typescript
// Types: PascalCase
type GamePhase = 'morning' | 'action' | 'resolution' | 'night';
interface BetResult { won: boolean; payout: number }

// Constants: SCREAMING_SNAKE_CASE for true constants
const MAX_BET_AMOUNT = 10000;
const DEFAULT_STARTING_COINS = 500;

// Config objects: camelCase
const gameConfig = { maxPlayers: 4, startingCoins: 500 };

// Functions: camelCase, verb-first
function calculatePayout(bet: Bet, pot: Pot): number { ... }
function isValidBet(amount: number, balance: number): boolean { ... }

// Boolean variables/props: is/has/should/can prefix
const isLoading = true;
const hasPermission = false;
const shouldAnimate = !reducedMotion;

// Event handlers: handle/on prefix
function handleBetPlaced(bet: Bet) { ... }
const onDayComplete = () => { ... };

// React components: PascalCase, descriptive
function BetConfirmationDialog({ ... }) { ... }
function PotFillIndicator({ ... }) { ... }
```

---

## Design Patterns

### Repository Pattern for Data Access

```typescript
interface Repository<T, ID = string> {
  findById(id: ID): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  create(data: Omit<T, 'id' | 'createdAt'>): Promise<T>;
  update(id: ID, data: Partial<T>): Promise<T>;
  delete(id: ID): Promise<void>;
}

class SeasonRepository implements Repository<Season> {
  constructor(private db: DrizzleClient) {}
  
  async findById(id: string): Promise<Season | null> {
    const result = await this.db
      .select()
      .from(seasons)
      .where(eq(seasons.id, id))
      .limit(1);
    return result[0] || null;
  }
  
  async findAll(filter?: Partial<Season>): Promise<Season[]> {
    let query = this.db.select().from(seasons);
    
    if (filter?.userId) {
      query = query.where(eq(seasons.userId, filter.userId));
    }
    
    return query.orderBy(desc(seasons.startedAt));
  }
  
  async create(data: Omit<Season, 'id' | 'createdAt'>): Promise<Season> {
    const id = createSeasonId();
    const now = new Date();
    
    const [season] = await this.db
      .insert(seasons)
      .values({ id, ...data, startedAt: now })
      .returning();
    
    return season;
  }
  
  async update(id: string, data: Partial<Season>): Promise<Season> {
    const [updated] = await this.db
      .update(seasons)
      .set(data)
      .where(eq(seasons.id, id))
      .returning();
    
    return updated;
  }
  
  async delete(id: string): Promise<void> {
    await this.db.delete(seasons).where(eq(seasons.id, id));
  }
  
  // Domain-specific queries
  async findActiveSeason(userId: string): Promise<Season | null> {
    const result = await this.db
      .select()
      .from(seasons)
      .where(
        and(
          eq(seasons.userId, userId),
          isNull(seasons.completedAt)
        )
      )
      .limit(1);
    return result[0] || null;
  }
}
```

### Service Layer Pattern

```typescript
class BettingService {
  constructor(
    private seasonRepo: SeasonRepository,
    private betRepo: BetRepository,
    private eventBus: GameEventBus
  ) {}
  
  async placeBet(
    userId: string,
    seasonId: string,
    potId: string,
    betType: BetType,
    amount: number
  ): Promise<Result<Bet, BettingError>> {
    // 1. Validate
    const season = await this.seasonRepo.findById(seasonId);
    if (!season) return err('SEASON_NOT_FOUND');
    if (season.userId !== userId) return err('UNAUTHORIZED');
    if (season.phase !== 'action') return err('WRONG_PHASE');
    
    // 2. Check balance
    const balance = await this.getBalance(userId);
    if (amount > balance) return err('INSUFFICIENT_FUNDS');
    
    // 3. Check duplicate bet
    const existingBet = await this.betRepo.findByPot(seasonId, season.currentDay, potId);
    if (existingBet) return err('ALREADY_BET');
    
    // 4. Create bet
    const bet = await this.betRepo.create({
      seasonId,
      dayNumber: season.currentDay,
      potId,
      betType,
      amount,
    });
    
    // 5. Emit event
    this.eventBus.emit({
      type: 'BET_PLACED',
      userId,
      seasonId,
      potId,
      betType,
      amount,
    });
    
    return ok(bet);
  }
  
  private async getBalance(userId: string): Promise<number> {
    // Implementation
    return 0;
  }
}

// Result type for explicit error handling
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
function ok<T>(value: T): Result<T, never> { return { ok: true, value }; }
function err<E>(error: E): Result<never, E> { return { ok: false, error }; }

type BettingError = 
  | 'SEASON_NOT_FOUND'
  | 'UNAUTHORIZED'
  | 'WRONG_PHASE'
  | 'INSUFFICIENT_FUNDS'
  | 'ALREADY_BET'
  | 'INVALID_AMOUNT';
```

### Dependency Injection via Factory Functions

```typescript
// Compose services with explicit dependencies
function createServices(db: DrizzleClient, eventBus: GameEventBus) {
  const seasonRepo = new SeasonRepository(db);
  const betRepo = new BetRepository(db);
  const triggerRepo = new TriggerRepository(db);
  
  const bettingService = new BettingService(seasonRepo, betRepo, eventBus);
  const farmingService = new FarmingService(seasonRepo, triggerRepo, eventBus);
  const progressionService = new ProgressionService(seasonRepo, eventBus);
  
  return {
    betting: bettingService,
    farming: farmingService,
    progression: progressionService,
  };
}

// Usage in API routes
const services = createServices(db, eventBus);

app.post('/api/bet', async (c) => {
  const body = await c.req.json();
  const result = await services.betting.placeBet(
    c.get('userId'),
    body.seasonId,
    body.potId,
    body.betType,
    body.amount
  );
  
  if (!result.ok) {
    return c.json({ error: result.error }, 400);
  }
  
  return c.json(result.value);
});
```

---

## Refactoring Patterns

### Extract Hook Pattern

Before:
```typescript
function BetPanel() {
  const [bets, setBets] = useState<Bet[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const balance = useStore(s => s.balance);
  
  const placeBet = async (potId: string, betType: BetType, amount: number) => {
    setLoading(true);
    setError(null);
    try {
      const result = await api.placeBet(potId, betType, amount);
      setBets(prev => [...prev, result]);
    } catch (e) {
      setError(e.message);
    } finally {
      setLoading(false);
    }
  };
  
  const removeBet = (potId: string) => {
    setBets(prev => prev.filter(b => b.potId !== potId));
  };
  
  // ... 200 more lines of component code
}
```

After:
```typescript
function useBetting() {
  const [bets, setBets] = useState<Bet[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const balance = useStore(s => s.balance);
  
  const placeBet = useCallback(async (potId: string, betType: BetType, amount: number) => {
    setLoading(true);
    setError(null);
    try {
      const result = await api.placeBet(potId, betType, amount);
      setBets(prev => [...prev, result]);
    } catch (e: any) {
      setError(e.message);
    } finally {
      setLoading(false);
    }
  }, []);
  
  const removeBet = useCallback((potId: string) => {
    setBets(prev => prev.filter(b => b.potId !== potId));
  }, []);
  
  return { bets, loading, error, balance, placeBet, removeBet };
}

function BetPanel() {
  const { bets, loading, error, placeBet, removeBet } = useBetting();
  // Component is now pure rendering logic
}
```

### Replace Conditional with Polymorphism

Before:
```typescript
function calculatePayout(betType: BetType, amount: number, threshold: number): number {
  if (betType === 'fold') return 0;
  if (betType === 'call') return amount * (threshold >= 50 ? 1.5 : 0);
  if (betType === 'raise') return amount * (threshold >= 80 ? 2.5 : threshold >= 50 ? 1.5 : 0);
  if (betType === 'all-in') return amount * (threshold >= 100 ? 5 : threshold >= 80 ? 2 : 0);
  return 0;
}
```

After:
```typescript
interface PayoutStrategy {
  calculate(amount: number, threshold: number): number;
}

const PAYOUT_STRATEGIES: Record<BetType, PayoutStrategy> = {
  fold: {
    calculate: () => 0,
  },
  call: {
    calculate: (amount, threshold) => threshold >= 50 ? amount * 1.5 : 0,
  },
  raise: {
    calculate: (amount, threshold) => {
      if (threshold >= 80) return amount * 2.5;
      if (threshold >= 50) return amount * 1.5;
      return 0;
    },
  },
  'all-in': {
    calculate: (amount, threshold) => {
      if (threshold >= 100) return amount * 5;
      if (threshold >= 80) return amount * 2;
      return 0;
    },
  },
};

function calculatePayout(betType: BetType, amount: number, threshold: number): number {
  return PAYOUT_STRATEGIES[betType].calculate(amount, threshold);
}
```

### Guard Clause Pattern

Before:
```typescript
function processDay(season: Season, bets: Bet[]): DayResult {
  if (season) {
    if (season.phase === 'action') {
      if (bets.length > 0) {
        if (bets.every(b => b.amount <= season.balance)) {
          // 4 levels deep...
          return calculateResults(season, bets);
        } else {
          throw new Error('Insufficient balance');
        }
      } else {
        throw new Error('No bets placed');
      }
    } else {
      throw new Error('Wrong phase');
    }
  } else {
    throw new Error('Season not found');
  }
}
```

After:
```typescript
function processDay(season: Season | null, bets: Bet[]): DayResult {
  if (!season) throw new Error('Season not found');
  if (season.phase !== 'action') throw new Error('Wrong phase');
  if (bets.length === 0) throw new Error('No bets placed');
  if (!bets.every(b => b.amount <= season.balance)) throw new Error('Insufficient balance');
  
  return calculateResults(season, bets);
}
```

---

## Error Handling Strategy

### Layered Error Types

```typescript
// Base error with code and context
class AppError extends Error {
  constructor(
    public code: string,
    message: string,
    public statusCode: number = 500,
    public context?: Record<string, unknown>
  ) {
    super(message);
    this.name = 'AppError';
  }
}

class ValidationError extends AppError {
  constructor(message: string, context?: Record<string, unknown>) {
    super('VALIDATION_ERROR', message, 400, context);
    this.name = 'ValidationError';
  }
}

class NotFoundError extends AppError {
  constructor(entity: string, id: string) {
    super('NOT_FOUND', `${entity} with id "${id}" not found`, 404, { entity, id });
    this.name = 'NotFoundError';
  }
}

class BusinessRuleError extends AppError {
  constructor(rule: string, message: string, context?: Record<string, unknown>) {
    super(`RULE_${rule}`, message, 422, context);
    this.name = 'BusinessRuleError';
  }
}

// Global error handler
function handleApiError(error: unknown): { status: number; body: object } {
  if (error instanceof AppError) {
    return {
      status: error.statusCode,
      body: {
        error: error.code,
        message: error.message,
        ...(process.env.NODE_ENV === 'development' && { context: error.context }),
      },
    };
  }
  
  console.error('Unhandled error:', error);
  return {
    status: 500,
    body: { error: 'INTERNAL_ERROR', message: 'An unexpected error occurred' },
  };
}
```

---

## Testing Patterns

### Test Organization

```
__tests__/
├── unit/
│   ├── game-engine/
│   │   ├── betting.test.ts
│   │   ├── combo-engine.test.ts
│   │   └── progression.test.ts
│   └── utils/
│       └── math.test.ts
│
├── integration/
│   ├── api/
│   │   ├── betting-flow.test.ts
│   │   └── season-lifecycle.test.ts
│   └── db/
│       └── migrations.test.ts
│
└── e2e/
    ├── onboarding.spec.ts
    ├── betting-round.spec.ts
    └── season-complete.spec.ts
```

### Test Factories

```typescript
function createTestSeason(overrides?: Partial<Season>): Season {
  return {
    id: `season_${Math.random().toString(36).slice(2)}`,
    userId: 'user_test',
    seasonNumber: 1,
    currentDay: 1,
    phase: 'action',
    jackpotContribution: 0,
    startedAt: new Date(),
    completedAt: null,
    ...overrides,
  };
}

function createTestBet(overrides?: Partial<Bet>): Bet {
  return {
    id: `bet_${Math.random().toString(36).slice(2)}`,
    seasonId: 'season_test',
    dayNumber: 1,
    potId: 'water',
    betType: 'call',
    amount: 100,
    placedAt: new Date(),
    ...overrides,
  };
}

// Composable test setup
function createTestGame(config?: {
  balance?: number;
  day?: number;
  phase?: GamePhase;
}) {
  const season = createTestSeason({
    currentDay: config?.day || 1,
    phase: config?.phase || 'action',
  });
  
  const eventBus = new GameEventBus();
  const bettingSystem = new BettingSystem();
  
  return { season, eventBus, bettingSystem };
}
```

### Snapshot Testing for Game State

```typescript
describe('Day Resolution', () => {
  it('produces consistent results with same seed', () => {
    const rng = new SeededRNG(12345);
    const result1 = simulateDay(rng, createTestSeason());
    
    const rng2 = new SeededRNG(12345);
    const result2 = simulateDay(rng2, createTestSeason());
    
    expect(result1).toEqual(result2);
  });
  
  it('matches expected state snapshot', () => {
    const rng = new SeededRNG(42);
    const result = simulateDay(rng, createTestSeason());
    
    expect(result).toMatchSnapshot();
  });
});
```

---

## Documentation Standards

### JSDoc for Public APIs

```typescript
/**
 * Places a bet on a specific pot for the current day.
 * 
 * @param potId - The pot to bet on ('water' | 'sun' | 'pest' | 'growth')
 * @param betType - The type of bet to place
 * @param amount - Bet amount in coins (must be positive for non-fold bets)
 * @returns The created bet record
 * @throws {ValidationError} If the bet amount is invalid
 * @throws {BusinessRuleError} If the player has already bet on this pot
 * 
 * @example
 * ```typescript
 * const bet = await bettingService.placeBet('water', 'raise', 100);
 * console.log(bet.potId); // 'water'
 * ```
 */
async placeBet(potId: string, betType: BetType, amount: number): Promise<Bet> {
  // ...
}
```

### Type-Level Documentation

```typescript
/**
 * Represents a single game day's betting phase.
 * 
 * Flow: Bets placed → Day simulated → Pots fill → Bets resolved
 */
interface DayPhase {
  /** Current day number within the season (1-28) */
  dayNumber: number;
  
  /** Bets placed by the player for this day */
  bets: Bet[];
  
  /** Current fill levels of all four pots (0-100) */
  potLevels: Record<PotId, number>;
  
  /** Whether the day has been resolved */
  resolved: boolean;
}
```

---

*Architecture & Quality Reference — Game Engineering Team*
