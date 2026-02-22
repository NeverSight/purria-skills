# Data Infrastructure

Schema design, database migrations, telemetry, analytics, and security infrastructure.

---

## Schema Design

### Branded ID Types

Prevent accidental ID mixing across entity types with TypeScript branded types.

```typescript
// Branded ID types for compile-time safety
declare const __brand: unique symbol;
type Brand<T, B> = T & { readonly [__brand]: B };

type UserId = Brand<string, 'UserId'>;
type SeasonId = Brand<string, 'SeasonId'>;
type SimulinId = Brand<string, 'SimulinId'>;
type BetId = Brand<string, 'BetId'>;
type TriggerId = Brand<string, 'TriggerId'>;

function createUserId(): UserId {
  return `usr_${crypto.randomUUID()}` as UserId;
}

function createSeasonId(): SeasonId {
  return `ssn_${crypto.randomUUID()}` as SeasonId;
}

function createSimulinId(): SimulinId {
  return `sim_${crypto.randomUUID()}` as SimulinId;
}

// Compile-time error: Type 'UserId' is not assignable to type 'SeasonId'
// findSeason(userId) ← caught at compile time
```

### Complete Drizzle Schema

```typescript
import { sqliteTable, text, integer, real, index, uniqueIndex } from 'drizzle-orm/sqlite-core';
import { relations } from 'drizzle-orm';

// ── Users ──────────────────────────────────────────────────────────────────
export const users = sqliteTable('users', {
  id: text('id').primaryKey(),
  email: text('email').notNull(),
  displayName: text('display_name').notNull(),
  avatarUrl: text('avatar_url'),
  role: text('role', { enum: ['player', 'admin', 'moderator'] }).notNull().default('player'),
  createdAt: integer('created_at', { mode: 'timestamp' }).notNull(),
  lastLoginAt: integer('last_login_at', { mode: 'timestamp' }),
}, (table) => ({
  emailIdx: uniqueIndex('idx_users_email').on(table.email),
}));

export const usersRelations = relations(users, ({ many }) => ({
  seasons: many(seasons),
  simulins: many(simulins),
  sessions: many(sessions),
}));

// ── Sessions ───────────────────────────────────────────────────────────────
export const sessions = sqliteTable('sessions', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  expiresAt: integer('expires_at', { mode: 'timestamp' }).notNull(),
  createdAt: integer('created_at', { mode: 'timestamp' }).notNull(),
  userAgent: text('user_agent'),
  ipAddress: text('ip_address'),
}, (table) => ({
  userIdx: index('idx_sessions_user').on(table.userId),
  expiresIdx: index('idx_sessions_expires').on(table.expiresAt),
}));

// ── Seasons ────────────────────────────────────────────────────────────────
export const seasons = sqliteTable('seasons', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().references(() => users.id),
  seasonNumber: integer('season_number').notNull(),
  currentDay: integer('current_day').notNull().default(1),
  phase: text('phase', { enum: ['planting', 'growth', 'harvest'] }).notNull().default('planting'),
  coins: integer('coins').notNull().default(500),
  jackpotContribution: real('jackpot_contribution').notNull().default(0),
  startedAt: integer('started_at', { mode: 'timestamp' }).notNull(),
  completedAt: integer('completed_at', { mode: 'timestamp' }),
  finalScore: integer('final_score'),
}, (table) => ({
  userIdx: index('idx_seasons_user').on(table.userId),
  activeIdx: index('idx_seasons_active').on(table.userId, table.completedAt),
}));

export const seasonsRelations = relations(seasons, ({ one, many }) => ({
  user: one(users, { fields: [seasons.userId], references: [users.id] }),
  days: many(days),
  bets: many(bets),
  triggers: many(dailyTriggers),
}));

// ── Days ───────────────────────────────────────────────────────────────────
export const days = sqliteTable('days', {
  id: text('id').primaryKey(),
  seasonId: text('season_id').notNull().references(() => seasons.id, { onDelete: 'cascade' }),
  dayNumber: integer('day_number').notNull(),
  seed: text('seed').notNull(),
  potLevels: text('pot_levels', { mode: 'json' }).notNull(), // { water: 75, sun: 60, pest: 90, growth: 45 }
  events: text('events', { mode: 'json' }).notNull(),         // DayEvent[]
  weather: text('weather', { enum: ['sunny', 'cloudy', 'rainy', 'stormy'] }).notNull(),
  resolved: integer('resolved', { mode: 'boolean' }).notNull().default(false),
  createdAt: integer('created_at', { mode: 'timestamp' }).notNull(),
}, (table) => ({
  seasonDayIdx: uniqueIndex('idx_days_season_day').on(table.seasonId, table.dayNumber),
}));

// ── Bets ───────────────────────────────────────────────────────────────────
export const bets = sqliteTable('bets', {
  id: text('id').primaryKey(),
  seasonId: text('season_id').notNull().references(() => seasons.id, { onDelete: 'cascade' }),
  dayNumber: integer('day_number').notNull(),
  potId: text('pot_id', { enum: ['water', 'sun', 'pest', 'growth'] }).notNull(),
  betType: text('bet_type', { enum: ['fold', 'call', 'raise', 'all-in'] }).notNull(),
  amount: integer('amount').notNull(),
  payout: integer('payout'),
  won: integer('won', { mode: 'boolean' }),
  resolvedAt: integer('resolved_at', { mode: 'timestamp' }),
  placedAt: integer('placed_at', { mode: 'timestamp' }).notNull(),
}, (table) => ({
  seasonDayIdx: index('idx_bets_season_day').on(table.seasonId, table.dayNumber),
  seasonDayPotIdx: uniqueIndex('idx_bets_unique').on(table.seasonId, table.dayNumber, table.potId),
}));

// ── Daily Triggers ─────────────────────────────────────────────────────────
export const dailyTriggers = sqliteTable('daily_triggers', {
  id: text('id').primaryKey(),
  seasonId: text('season_id').notNull().references(() => seasons.id, { onDelete: 'cascade' }),
  dayNumber: integer('day_number').notNull(),
  triggerType: text('trigger_type', { 
    enum: ['first_bloom', 'perfect_read', 'trouble_slayer', 'growth_spurt', 'simulin_sync'] 
  }).notNull(),
  earnedAt: integer('earned_at', { mode: 'timestamp' }).notNull(),
  metadata: text('metadata', { mode: 'json' }),
}, (table) => ({
  seasonIdx: index('idx_triggers_season').on(table.seasonId),
  seasonDayIdx: index('idx_triggers_season_day').on(table.seasonId, table.dayNumber),
}));

// ── Simulins ───────────────────────────────────────────────────────────────
export const simulins = sqliteTable('simulins', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().references(() => users.id),
  name: text('name').notNull(),
  species: text('species').notNull(),
  stage: text('stage', { enum: ['sproutling', 'cultivar', 'bloomform', 'legendary'] }).notNull(),
  path: text('path', { enum: ['tender', 'watcher', 'lucky'] }),
  bondLevel: integer('bond_level').notNull().default(1),
  bondXP: integer('bond_xp').notNull().default(0),
  personality: text('personality', { mode: 'json' }).notNull(),
  abilities: text('abilities', { mode: 'json' }).notNull().default('[]'),
  createdAt: integer('created_at', { mode: 'timestamp' }).notNull(),
  retiredAt: integer('retired_at', { mode: 'timestamp' }),
}, (table) => ({
  userIdx: index('idx_simulins_user').on(table.userId),
}));

export const simulinsRelations = relations(simulins, ({ one }) => ({
  user: one(users, { fields: [simulins.userId], references: [users.id] }),
}));

// ── Leaderboard ────────────────────────────────────────────────────────────
export const leaderboardEntries = sqliteTable('leaderboard_entries', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().references(() => users.id),
  seasonId: text('season_id').notNull().references(() => seasons.id),
  score: integer('score').notNull(),
  rank: integer('rank'),
  period: text('period', { enum: ['weekly', 'seasonal', 'all_time'] }).notNull(),
  calculatedAt: integer('calculated_at', { mode: 'timestamp' }).notNull(),
}, (table) => ({
  periodScoreIdx: index('idx_leaderboard_period_score').on(table.period, table.score),
  userPeriodIdx: index('idx_leaderboard_user_period').on(table.userId, table.period),
}));
```

---

## Migration Management

### Drizzle Kit Configuration

```typescript
// drizzle.config.ts
import type { Config } from 'drizzle-kit';

export default {
  schema: './src/db/schema.ts',
  out: './drizzle/migrations',
  dialect: 'sqlite',
  dbCredentials: {
    url: process.env.DATABASE_URL || './local.db',
  },
} satisfies Config;
```

### Migration Workflow

```bash
# Generate migration from schema changes
pnpm drizzle-kit generate

# Apply migrations
pnpm drizzle-kit migrate

# Quick prototype without migration files (dev only)
pnpm drizzle-kit push

# Pull existing DB schema to TypeScript
pnpm drizzle-kit pull

# View current schema visually
pnpm drizzle-kit studio
```

### Safe Migration Patterns

```typescript
// Custom migration for data transformation
import { sql } from 'drizzle-orm';

export async function migrateCoinsToNewFormat(db: DrizzleClient) {
  // Step 1: Add new column
  await db.run(sql`ALTER TABLE seasons ADD COLUMN coins_v2 INTEGER DEFAULT 0`);
  
  // Step 2: Migrate data
  await db.run(sql`UPDATE seasons SET coins_v2 = coins * 100`); // Convert to cents
  
  // Step 3: Verify
  const mismatch = await db.get(sql`
    SELECT COUNT(*) as count FROM seasons WHERE coins_v2 != coins * 100
  `);
  
  if (mismatch && (mismatch as any).count > 0) {
    throw new Error('Migration verification failed');
  }
  
  // Step 4: Swap columns (SQLite requires table recreation)
  // For production: use a phased approach with read/write to both columns
}

// Rollback support
interface Migration {
  version: string;
  up: (db: DrizzleClient) => Promise<void>;
  down: (db: DrizzleClient) => Promise<void>;
}

const migrations: Migration[] = [
  {
    version: '001_add_coins_v2',
    up: async (db) => {
      await db.run(sql`ALTER TABLE seasons ADD COLUMN coins_v2 INTEGER DEFAULT 0`);
      await db.run(sql`UPDATE seasons SET coins_v2 = coins * 100`);
    },
    down: async (db) => {
      // SQLite doesn't support DROP COLUMN in older versions
      // Use table recreation pattern
    },
  },
];
```

---

## Telemetry & Analytics

### Event Schema

```typescript
interface TelemetryEvent {
  eventId: string;
  eventType: string;
  timestamp: number;
  sessionId: string;
  userId?: string;
  properties: Record<string, string | number | boolean>;
  context: EventContext;
}

interface EventContext {
  appVersion: string;
  platform: 'web' | 'ios' | 'android';
  screenWidth: number;
  screenHeight: number;
  locale: string;
  timezone: string;
  connectionType?: string;
}

// Strongly typed event catalog
type GameTelemetryEvent =
  | { type: 'session_start'; properties: { referrer?: string } }
  | { type: 'session_end'; properties: { duration_ms: number; days_played: number } }
  | { type: 'season_start'; properties: { season_number: number } }
  | { type: 'season_complete'; properties: { season_number: number; final_score: number; days_played: number } }
  | { type: 'day_start'; properties: { day_number: number; phase: string } }
  | { type: 'day_complete'; properties: { day_number: number; triggers_earned: number } }
  | { type: 'bet_placed'; properties: { pot_id: string; bet_type: string; amount: number } }
  | { type: 'bet_resolved'; properties: { pot_id: string; won: boolean; payout: number } }
  | { type: 'trigger_earned'; properties: { trigger_type: string; day_number: number } }
  | { type: 'combo_evaluated'; properties: { hand_type: string; multiplier: number } }
  | { type: 'simulin_bond'; properties: { simulin_id: string; new_level: number } }
  | { type: 'tutorial_step'; properties: { step_id: string; action: 'shown' | 'completed' | 'skipped' } }
  | { type: 'error'; properties: { error_code: string; message: string; stack?: string } }
  | { type: 'performance'; properties: { metric: string; value: number; unit: string } };
```

### Telemetry Client

```typescript
class TelemetryClient {
  private queue: TelemetryEvent[] = [];
  private flushTimer: ReturnType<typeof setInterval> | null = null;
  private sessionId: string;
  private context: EventContext;
  
  constructor(
    private config: {
      endpoint: string;
      batchSize: number;
      flushIntervalMs: number;
      maxQueueSize: number;
      enabled: boolean;
    }
  ) {
    this.sessionId = this.initSession();
    this.context = this.buildContext();
    
    if (config.enabled) {
      this.flushTimer = setInterval(() => this.flush(), config.flushIntervalMs);
      
      // Flush on page unload
      if (typeof window !== 'undefined') {
        window.addEventListener('beforeunload', () => this.flush());
        document.addEventListener('visibilitychange', () => {
          if (document.visibilityState === 'hidden') this.flush();
        });
      }
    }
  }
  
  track<T extends GameTelemetryEvent>(
    type: T['type'],
    properties: T['properties']
  ): void {
    if (!this.config.enabled) return;
    
    const event: TelemetryEvent = {
      eventId: crypto.randomUUID(),
      eventType: type,
      timestamp: Date.now(),
      sessionId: this.sessionId,
      userId: this.getUserId(),
      properties: properties as Record<string, string | number | boolean>,
      context: this.context,
    };
    
    this.queue.push(event);
    
    // Auto-flush when batch size reached
    if (this.queue.length >= this.config.batchSize) {
      this.flush();
    }
    
    // Prevent unbounded growth
    if (this.queue.length > this.config.maxQueueSize) {
      this.queue = this.queue.slice(-this.config.maxQueueSize);
    }
  }
  
  private async flush(): Promise<void> {
    if (this.queue.length === 0) return;
    
    const batch = this.queue.splice(0, this.config.batchSize);
    
    try {
      // Use sendBeacon for reliability during page unload
      if (typeof navigator?.sendBeacon === 'function') {
        const blob = new Blob(
          [JSON.stringify({ events: batch })],
          { type: 'application/json' }
        );
        const success = navigator.sendBeacon(this.config.endpoint, blob);
        if (!success) throw new Error('sendBeacon failed');
      } else {
        await fetch(this.config.endpoint, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ events: batch }),
          keepalive: true,
        });
      }
    } catch (error) {
      // Re-queue on failure (with limit)
      if (this.queue.length < this.config.maxQueueSize) {
        this.queue.unshift(...batch);
      }
    }
  }
  
  private initSession(): string {
    const existing = sessionStorage.getItem('telemetry_session');
    if (existing) return existing;
    
    const id = crypto.randomUUID();
    sessionStorage.setItem('telemetry_session', id);
    return id;
  }
  
  private buildContext(): EventContext {
    return {
      appVersion: import.meta.env.VITE_APP_VERSION || 'dev',
      platform: 'web',
      screenWidth: typeof window !== 'undefined' ? window.innerWidth : 0,
      screenHeight: typeof window !== 'undefined' ? window.innerHeight : 0,
      locale: typeof navigator !== 'undefined' ? navigator.language : 'en',
      timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
      connectionType: (navigator as any)?.connection?.effectiveType,
    };
  }
  
  private getUserId(): string | undefined {
    // Integrate with auth system
    return undefined;
  }
  
  destroy(): void {
    this.flush();
    if (this.flushTimer) clearInterval(this.flushTimer);
  }
}

// Singleton instance
export const telemetry = new TelemetryClient({
  endpoint: '/api/telemetry',
  batchSize: 10,
  flushIntervalMs: 5000,
  maxQueueSize: 200,
  enabled: import.meta.env.PROD,
});
```

### Server-Side Telemetry Ingestion

```typescript
import { Hono } from 'hono';

const telemetryRouter = new Hono();

telemetryRouter.post('/api/telemetry', async (c) => {
  const { events } = await c.req.json<{ events: TelemetryEvent[] }>();
  
  // Validate events
  if (!Array.isArray(events) || events.length === 0 || events.length > 100) {
    return c.json({ error: 'Invalid batch' }, 400);
  }
  
  // Enrich with server-side context
  const enriched = events.map(event => ({
    ...event,
    serverTimestamp: Date.now(),
    ip: c.req.header('cf-connecting-ip') || c.req.header('x-forwarded-for'),
    country: c.req.header('cf-ipcountry'),
  }));
  
  // Batch insert (use queue for high throughput)
  await insertTelemetryBatch(enriched);
  
  return c.json({ accepted: events.length });
});

// Analytics queries
async function getDailyActiveUsers(db: DrizzleClient, date: Date): Promise<number> {
  const startOfDay = new Date(date);
  startOfDay.setHours(0, 0, 0, 0);
  const endOfDay = new Date(date);
  endOfDay.setHours(23, 59, 59, 999);
  
  const result = await db.get(sql`
    SELECT COUNT(DISTINCT user_id) as dau
    FROM telemetry_events
    WHERE timestamp >= ${startOfDay.getTime()}
    AND timestamp <= ${endOfDay.getTime()}
    AND user_id IS NOT NULL
  `);
  
  return (result as any)?.dau || 0;
}

async function getRetentionRate(
  db: DrizzleClient,
  cohortDate: Date,
  dayN: number
): Promise<number> {
  const result = await db.get(sql`
    WITH cohort AS (
      SELECT DISTINCT user_id
      FROM telemetry_events
      WHERE event_type = 'session_start'
      AND date(timestamp / 1000, 'unixepoch') = ${cohortDate.toISOString().split('T')[0]}
    ),
    retained AS (
      SELECT DISTINCT e.user_id
      FROM telemetry_events e
      JOIN cohort c ON e.user_id = c.user_id
      WHERE e.event_type = 'session_start'
      AND date(e.timestamp / 1000, 'unixepoch') = date(${cohortDate.toISOString().split('T')[0]}, '+' || ${dayN} || ' days')
    )
    SELECT
      (SELECT COUNT(*) FROM cohort) as cohort_size,
      (SELECT COUNT(*) FROM retained) as retained_count
  `);
  
  const { cohort_size, retained_count } = result as any;
  return cohort_size > 0 ? retained_count / cohort_size : 0;
}
```

### Performance Monitoring

```typescript
class PerformanceMonitor {
  private metrics: Map<string, number[]> = new Map();
  
  measure(metric: string, fn: () => void): void {
    const start = performance.now();
    fn();
    const duration = performance.now() - start;
    this.record(metric, duration);
  }
  
  async measureAsync<T>(metric: string, fn: () => Promise<T>): Promise<T> {
    const start = performance.now();
    const result = await fn();
    const duration = performance.now() - start;
    this.record(metric, duration);
    return result;
  }
  
  private record(metric: string, value: number): void {
    if (!this.metrics.has(metric)) {
      this.metrics.set(metric, []);
    }
    
    const values = this.metrics.get(metric)!;
    values.push(value);
    
    // Keep last 100 samples
    if (values.length > 100) values.shift();
    
    // Alert on threshold breach
    if (value > this.getThreshold(metric)) {
      telemetry.track('performance', {
        metric,
        value: Math.round(value * 100) / 100,
        unit: 'ms',
      });
    }
  }
  
  getStats(metric: string): {
    avg: number;
    p50: number;
    p95: number;
    p99: number;
    min: number;
    max: number;
  } | null {
    const values = this.metrics.get(metric);
    if (!values || values.length === 0) return null;
    
    const sorted = [...values].sort((a, b) => a - b);
    const len = sorted.length;
    
    return {
      avg: sorted.reduce((a, b) => a + b, 0) / len,
      p50: sorted[Math.floor(len * 0.5)],
      p95: sorted[Math.floor(len * 0.95)],
      p99: sorted[Math.floor(len * 0.99)],
      min: sorted[0],
      max: sorted[len - 1],
    };
  }
  
  private getThreshold(metric: string): number {
    const thresholds: Record<string, number> = {
      'render_frame': 16.67,     // 60fps budget
      'game_tick': 8,            // Half frame budget
      'api_response': 200,       // 200ms
      'db_query': 50,            // 50ms
      'state_update': 5,         // 5ms
    };
    return thresholds[metric] || 100;
  }
}

export const perfMonitor = new PerformanceMonitor();
```

---

## Security Infrastructure

### Authentication Flow

```typescript
import { Hono } from 'hono';
import { z } from 'zod';

const authRouter = new Hono();

const LoginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(128),
});

authRouter.post('/api/auth/login', async (c) => {
  const body = LoginSchema.parse(await c.req.json());
  
  const user = await findUserByEmail(body.email);
  if (!user) return c.json({ error: 'Invalid credentials' }, 401);
  
  const valid = await verifyPassword(body.password, user.passwordHash);
  if (!valid) {
    await recordFailedLogin(user.id, c.req.header('cf-connecting-ip'));
    return c.json({ error: 'Invalid credentials' }, 401);
  }
  
  // Check if account is locked
  if (await isAccountLocked(user.id)) {
    return c.json({ error: 'Account temporarily locked' }, 423);
  }
  
  const session = await createSession(user.id, {
    userAgent: c.req.header('user-agent'),
    ip: c.req.header('cf-connecting-ip'),
  });
  
  // Set HTTP-only cookie
  c.header('Set-Cookie', [
    `session=${session.id}`,
    'HttpOnly',
    'Secure',
    'SameSite=Strict',
    `Max-Age=${60 * 60 * 24 * 7}`, // 7 days
    'Path=/',
  ].join('; '));
  
  return c.json({ user: sanitizeUser(user) });
});

// Auth middleware
function requireAuth() {
  return async (c: any, next: any) => {
    const sessionId = getCookieValue(c.req.header('cookie'), 'session');
    if (!sessionId) return c.json({ error: 'Unauthorized' }, 401);
    
    const session = await validateSession(sessionId);
    if (!session) return c.json({ error: 'Session expired' }, 401);
    
    c.set('userId', session.userId);
    c.set('sessionId', session.id);
    
    await next();
  };
}
```

### Input Validation Middleware

```typescript
import { z } from 'zod';

function validate<T>(schema: z.ZodSchema<T>) {
  return async (c: any, next: any) => {
    try {
      const body = await c.req.json();
      const validated = schema.parse(body);
      c.set('body', validated);
      await next();
    } catch (error) {
      if (error instanceof z.ZodError) {
        return c.json({
          error: 'VALIDATION_ERROR',
          details: error.errors.map(e => ({
            field: e.path.join('.'),
            message: e.message,
          })),
        }, 400);
      }
      throw error;
    }
  };
}

// Game-specific validation schemas
const PlaceBetSchema = z.object({
  seasonId: z.string().startsWith('ssn_'),
  potId: z.enum(['water', 'sun', 'pest', 'growth']),
  betType: z.enum(['fold', 'call', 'raise', 'all-in']),
  amount: z.number().int().min(0).max(10000),
});

const AdvanceDaySchema = z.object({
  seasonId: z.string().startsWith('ssn_'),
  expectedDay: z.number().int().min(1).max(28),
});
```

### Rate Limiting

```typescript
interface RateLimitConfig {
  windowMs: number;
  maxRequests: number;
  keyGenerator?: (c: any) => string;
}

class TokenBucketRateLimiter {
  private buckets: Map<string, { tokens: number; lastRefill: number }> = new Map();
  
  constructor(
    private maxTokens: number,
    private refillRate: number, // tokens per second
    private cleanupIntervalMs: number = 60000
  ) {
    setInterval(() => this.cleanup(), cleanupIntervalMs);
  }
  
  consume(key: string, tokens: number = 1): { allowed: boolean; remaining: number; retryAfter?: number } {
    this.refill(key);
    const bucket = this.buckets.get(key) || { tokens: this.maxTokens, lastRefill: Date.now() };
    
    if (bucket.tokens >= tokens) {
      bucket.tokens -= tokens;
      this.buckets.set(key, bucket);
      return { allowed: true, remaining: bucket.tokens };
    }
    
    const retryAfter = Math.ceil((tokens - bucket.tokens) / this.refillRate);
    return { allowed: false, remaining: 0, retryAfter };
  }
  
  private refill(key: string): void {
    const bucket = this.buckets.get(key);
    if (!bucket) return;
    
    const now = Date.now();
    const elapsed = (now - bucket.lastRefill) / 1000;
    const newTokens = Math.min(this.maxTokens, bucket.tokens + elapsed * this.refillRate);
    
    bucket.tokens = newTokens;
    bucket.lastRefill = now;
  }
  
  private cleanup(): void {
    const now = Date.now();
    for (const [key, bucket] of this.buckets) {
      if (now - bucket.lastRefill > 300000) { // 5 minutes idle
        this.buckets.delete(key);
      }
    }
  }
}

// Per-route rate limiters
const rateLimiters = {
  api: new TokenBucketRateLimiter(100, 10),         // 100 burst, 10/sec refill
  auth: new TokenBucketRateLimiter(5, 0.1),          // 5 burst, 1 per 10sec
  telemetry: new TokenBucketRateLimiter(50, 20),     // 50 burst, 20/sec refill
  bet: new TokenBucketRateLimiter(10, 2),             // 10 burst, 2/sec refill
};

function rateLimit(limiter: TokenBucketRateLimiter) {
  return async (c: any, next: any) => {
    const key = c.get('userId') || c.req.header('cf-connecting-ip') || 'anonymous';
    const result = limiter.consume(key);
    
    c.header('X-RateLimit-Remaining', result.remaining.toString());
    
    if (!result.allowed) {
      c.header('Retry-After', result.retryAfter!.toString());
      return c.json({ error: 'Rate limit exceeded' }, 429);
    }
    
    await next();
  };
}
```

### Anti-Cheat Validation

```typescript
class ServerAuthorityValidator {
  // All critical state transitions must be validated server-side
  validateBetPlacement(
    request: { seasonId: string; potId: string; betType: string; amount: number },
    serverState: { season: Season; existingBets: Bet[] }
  ): ValidationResult {
    const errors: string[] = [];
    
    // Phase check
    if (serverState.season.phase !== 'action') {
      errors.push('Cannot bet outside action phase');
    }
    
    // Balance check (server-side balance, not client-reported)
    if (request.amount > serverState.season.coins) {
      errors.push('Insufficient server-validated balance');
    }
    
    // Duplicate check
    if (serverState.existingBets.some(b => b.potId === request.potId)) {
      errors.push('Duplicate bet on pot');
    }
    
    // Amount bounds
    if (request.amount < 0 || request.amount > 10000) {
      errors.push('Bet amount out of bounds');
    }
    
    // Timing check (prevent automated rapid betting)
    const lastBet = serverState.existingBets[serverState.existingBets.length - 1];
    if (lastBet) {
      const timeSinceLastBet = Date.now() - lastBet.placedAt.getTime();
      if (timeSinceLastBet < 100) { // Less than 100ms between bets
        errors.push('Suspicious rapid betting detected');
        this.flagSuspicious(serverState.season.userId, 'rapid_bet');
      }
    }
    
    return {
      valid: errors.length === 0,
      errors,
    };
  }
  
  validateDayResult(
    clientResult: DayResult,
    serverSeed: string,
    season: Season
  ): ValidationResult {
    const errors: string[] = [];
    
    // Recalculate expected results from seed
    const rng = new SeededRNG(hashString(`${serverSeed}-day-${season.currentDay}`));
    const expectedResult = this.simulateDay(rng, season);
    
    // Compare pot levels
    for (const potId of ['water', 'sun', 'pest', 'growth'] as const) {
      if (Math.abs(clientResult.potLevels[potId] - expectedResult.potLevels[potId]) > 0.01) {
        errors.push(`Pot level mismatch for ${potId}`);
      }
    }
    
    // Compare coin totals
    if (clientResult.coinsEarned > expectedResult.maxPossibleCoins) {
      errors.push('Coin total exceeds maximum possible');
      this.flagSuspicious(season.userId, 'coin_manipulation');
    }
    
    return { valid: errors.length === 0, errors };
  }
  
  private flagSuspicious(userId: string, type: string): void {
    telemetry.track('error', {
      error_code: `ANTI_CHEAT_${type.toUpperCase()}`,
      message: `Suspicious activity: ${type}`,
    });
  }
  
  private simulateDay(rng: SeededRNG, season: Season): DayResult {
    // Server-side deterministic simulation
    return {} as DayResult;
  }
}

interface ValidationResult {
  valid: boolean;
  errors: string[];
}
```

---

## Database Performance

### Query Optimization Patterns

```typescript
// Batch loading to avoid N+1 queries
async function loadSeasonWithDetails(db: DrizzleClient, seasonId: string) {
  const [season, seasonBets, seasonTriggers] = await Promise.all([
    db.select().from(seasons).where(eq(seasons.id, seasonId)).limit(1),
    db.select().from(bets).where(eq(bets.seasonId, seasonId)),
    db.select().from(dailyTriggers).where(eq(dailyTriggers.seasonId, seasonId)),
  ]);
  
  return {
    ...season[0],
    bets: seasonBets,
    triggers: seasonTriggers,
  };
}

// Prepared statements for hot paths
const findActiveSeasonStmt = db
  .select()
  .from(seasons)
  .where(and(eq(seasons.userId, sql.placeholder('userId')), isNull(seasons.completedAt)))
  .limit(1)
  .prepare();

// Usage: const season = await findActiveSeasonStmt.execute({ userId: 'usr_xxx' });

// Connection pooling for Turso
import { createClient } from '@libsql/client';

const tursoClient = createClient({
  url: process.env.TURSO_URL!,
  authToken: process.env.TURSO_AUTH_TOKEN,
});
```

---

*Data Infrastructure Reference — Game Engineering Team*
