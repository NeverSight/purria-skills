# Extended Game Programming Patterns

Deep-dive implementations for Entity Component System (ECS), behavior trees, scene management, and other advanced game programming patterns.

---

## Entity Component System (ECS)

ECS separates game objects into entities (IDs), components (data), and systems (logic). This data-oriented approach enables massive scalability and cache-friendly memory access.

### Core Architecture

```typescript
type EntityId = number & { readonly __brand: unique symbol };

interface World {
  nextId: number;
  entities: Set<EntityId>;
  components: Map<string, Map<EntityId, unknown>>;
  systems: System[];
}

function createWorld(): World {
  return {
    nextId: 0,
    entities: new Set(),
    components: new Map(),
    systems: [],
  };
}

function spawnEntity(world: World): EntityId {
  const id = world.nextId++ as EntityId;
  world.entities.add(id);
  return id;
}

function despawnEntity(world: World, entity: EntityId): void {
  world.entities.delete(entity);
  for (const store of world.components.values()) {
    store.delete(entity);
  }
}
```

### Type-Safe Component Storage

```typescript
class ComponentStore<T> {
  private data = new Map<EntityId, T>();
  
  set(entity: EntityId, value: T): void {
    this.data.set(entity, value);
  }
  
  get(entity: EntityId): T | undefined {
    return this.data.get(entity);
  }
  
  has(entity: EntityId): boolean {
    return this.data.has(entity);
  }
  
  remove(entity: EntityId): void {
    this.data.delete(entity);
  }
  
  *entries(): IterableIterator<[EntityId, T]> {
    yield* this.data.entries();
  }
  
  get size(): number {
    return this.data.size;
  }
}

// Define components as plain data
interface Position { x: number; y: number }
interface Velocity { dx: number; dy: number }
interface Sprite { texture: string; frame: number; visible: boolean }
interface Health { current: number; max: number }
interface Disabled {}

// Component registry with type safety
class ComponentRegistry {
  private stores = new Map<string, ComponentStore<any>>();
  
  register<T>(name: string): ComponentStore<T> {
    const store = new ComponentStore<T>();
    this.stores.set(name, store);
    return store;
  }
  
  get<T>(name: string): ComponentStore<T> {
    const store = this.stores.get(name);
    if (!store) throw new Error(`Component "${name}" not registered`);
    return store;
  }
}

const registry = new ComponentRegistry();
const positions = registry.register<Position>('position');
const velocities = registry.register<Velocity>('velocity');
const sprites = registry.register<Sprite>('sprite');
const health = registry.register<Health>('health');
const disabled = registry.register<Disabled>('disabled');
```

### Archetype-Based Query System

```typescript
interface Query {
  with: string[];
  without?: string[];
}

function* queryEntities(
  world: World,
  registry: ComponentRegistry,
  query: Query
): Generator<EntityId> {
  const requiredStores = query.with.map(name => registry.get(name));
  const excludedStores = (query.without || []).map(name => registry.get(name));
  
  // Use smallest required store for iteration
  const smallest = requiredStores.reduce(
    (min, store) => store.size < min.size ? store : min
  );
  
  for (const [entity] of smallest.entries()) {
    if (!world.entities.has(entity)) continue;
    
    const hasAll = requiredStores.every(s => s.has(entity));
    const hasNone = excludedStores.every(s => !s.has(entity));
    
    if (hasAll && hasNone) {
      yield entity;
    }
  }
}
```

### System Definition and Scheduling

```typescript
type SystemStage = 'preUpdate' | 'update' | 'postUpdate' | 'render';

interface System {
  name: string;
  stage: SystemStage;
  query: Query;
  execute: (entities: EntityId[], dt: number) => void;
}

class SystemScheduler {
  private systems: Map<SystemStage, System[]> = new Map([
    ['preUpdate', []],
    ['update', []],
    ['postUpdate', []],
    ['render', []],
  ]);
  
  add(system: System): void {
    this.systems.get(system.stage)!.push(system);
  }
  
  tick(world: World, registry: ComponentRegistry, dt: number): void {
    for (const stage of ['preUpdate', 'update', 'postUpdate', 'render'] as SystemStage[]) {
      for (const system of this.systems.get(stage)!) {
        const entities = [...queryEntities(world, registry, system.query)];
        system.execute(entities, dt);
      }
    }
  }
}

// Movement system: processes entities with position + velocity, skips disabled
const movementSystem: System = {
  name: 'movement',
  stage: 'update',
  query: { with: ['position', 'velocity'], without: ['disabled'] },
  execute(entities, dt) {
    for (const entity of entities) {
      const pos = positions.get(entity)!;
      const vel = velocities.get(entity)!;
      pos.x += vel.dx * dt;
      pos.y += vel.dy * dt;
    }
  },
};

// Health system: removes entities at zero health
const healthSystem: System = {
  name: 'health',
  stage: 'postUpdate',
  query: { with: ['health'] },
  execute(entities) {
    for (const entity of entities) {
      const hp = health.get(entity)!;
      if (hp.current <= 0) {
        disabled.set(entity, {});
      }
    }
  },
};
```

### Culling with Disabled Component

Toggle entities with a `Disabled` component instead of destroying them. This allows 100x+ scaling by disabling physics and rendering for off-screen entities.

```typescript
class CullingSystem {
  constructor(
    private viewportBounds: { x: number; y: number; w: number; h: number }
  ) {}
  
  execute(entities: EntityId[]): void {
    for (const entity of entities) {
      const pos = positions.get(entity);
      if (!pos) continue;
      
      const inView = 
        pos.x >= this.viewportBounds.x &&
        pos.x <= this.viewportBounds.x + this.viewportBounds.w &&
        pos.y >= this.viewportBounds.y &&
        pos.y <= this.viewportBounds.y + this.viewportBounds.h;
      
      if (inView) {
        disabled.remove(entity);
      } else {
        disabled.set(entity, {});
      }
    }
  }
}
```

---

## Behavior Trees

Industry-standard AI architecture used in AAA titles (Halo, The Last of Us). Organizes decision-making hierarchically through composable nodes.

### Node Types

```typescript
type NodeStatus = 'running' | 'success' | 'failure';

interface BehaviorNode {
  name: string;
  tick(blackboard: Blackboard): NodeStatus;
  reset(): void;
}

type Blackboard = Map<string, unknown>;
```

### Composite Nodes

```typescript
// Sequence: runs children in order, fails on first failure
class SequenceNode implements BehaviorNode {
  private currentIndex = 0;
  
  constructor(
    public name: string,
    private children: BehaviorNode[]
  ) {}
  
  tick(blackboard: Blackboard): NodeStatus {
    while (this.currentIndex < this.children.length) {
      const status = this.children[this.currentIndex].tick(blackboard);
      
      if (status === 'running') return 'running';
      if (status === 'failure') {
        this.reset();
        return 'failure';
      }
      
      this.currentIndex++;
    }
    
    this.reset();
    return 'success';
  }
  
  reset(): void {
    this.currentIndex = 0;
    this.children.forEach(c => c.reset());
  }
}

// Selector: tries children in order, succeeds on first success
class SelectorNode implements BehaviorNode {
  private currentIndex = 0;
  
  constructor(
    public name: string,
    private children: BehaviorNode[]
  ) {}
  
  tick(blackboard: Blackboard): NodeStatus {
    while (this.currentIndex < this.children.length) {
      const status = this.children[this.currentIndex].tick(blackboard);
      
      if (status === 'running') return 'running';
      if (status === 'success') {
        this.reset();
        return 'success';
      }
      
      this.currentIndex++;
    }
    
    this.reset();
    return 'failure';
  }
  
  reset(): void {
    this.currentIndex = 0;
    this.children.forEach(c => c.reset());
  }
}

// Parallel: runs all children simultaneously
class ParallelNode implements BehaviorNode {
  constructor(
    public name: string,
    private children: BehaviorNode[],
    private successThreshold: number = Infinity // how many must succeed
  ) {}
  
  tick(blackboard: Blackboard): NodeStatus {
    let successCount = 0;
    let failureCount = 0;
    
    for (const child of this.children) {
      const status = child.tick(blackboard);
      if (status === 'success') successCount++;
      if (status === 'failure') failureCount++;
    }
    
    if (successCount >= this.successThreshold) return 'success';
    if (failureCount > this.children.length - this.successThreshold) return 'failure';
    return 'running';
  }
  
  reset(): void {
    this.children.forEach(c => c.reset());
  }
}
```

### Decorator Nodes

```typescript
// Inverts child result
class InverterNode implements BehaviorNode {
  constructor(
    public name: string,
    private child: BehaviorNode
  ) {}
  
  tick(blackboard: Blackboard): NodeStatus {
    const status = this.child.tick(blackboard);
    if (status === 'success') return 'failure';
    if (status === 'failure') return 'success';
    return 'running';
  }
  
  reset(): void { this.child.reset(); }
}

// Repeats child N times
class RepeatNode implements BehaviorNode {
  private count = 0;
  
  constructor(
    public name: string,
    private child: BehaviorNode,
    private times: number
  ) {}
  
  tick(blackboard: Blackboard): NodeStatus {
    while (this.count < this.times) {
      const status = this.child.tick(blackboard);
      
      if (status === 'running') return 'running';
      if (status === 'failure') {
        this.reset();
        return 'failure';
      }
      
      this.child.reset();
      this.count++;
    }
    
    this.reset();
    return 'success';
  }
  
  reset(): void {
    this.count = 0;
    this.child.reset();
  }
}

// Conditional gate
class GuardNode implements BehaviorNode {
  constructor(
    public name: string,
    private condition: (bb: Blackboard) => boolean,
    private child: BehaviorNode
  ) {}
  
  tick(blackboard: Blackboard): NodeStatus {
    if (!this.condition(blackboard)) return 'failure';
    return this.child.tick(blackboard);
  }
  
  reset(): void { this.child.reset(); }
}

// Cooldown decorator
class CooldownNode implements BehaviorNode {
  private lastRun = 0;
  
  constructor(
    public name: string,
    private child: BehaviorNode,
    private cooldownMs: number
  ) {}
  
  tick(blackboard: Blackboard): NodeStatus {
    const now = Date.now();
    if (now - this.lastRun < this.cooldownMs) return 'failure';
    
    const status = this.child.tick(blackboard);
    if (status === 'success') this.lastRun = now;
    return status;
  }
  
  reset(): void {
    this.lastRun = 0;
    this.child.reset();
  }
}
```

### Leaf Nodes (Actions & Conditions)

```typescript
class ActionNode implements BehaviorNode {
  constructor(
    public name: string,
    private action: (bb: Blackboard) => NodeStatus
  ) {}
  
  tick(blackboard: Blackboard): NodeStatus {
    return this.action(blackboard);
  }
  
  reset(): void {}
}

class ConditionNode implements BehaviorNode {
  constructor(
    public name: string,
    private condition: (bb: Blackboard) => boolean
  ) {}
  
  tick(blackboard: Blackboard): NodeStatus {
    return this.condition(blackboard) ? 'success' : 'failure';
  }
  
  reset(): void {}
}
```

### Practical Example: Farm NPC Behavior

```typescript
function createFarmerAI(): BehaviorNode {
  return new SelectorNode('farmer-root', [
    // Priority 1: Respond to trouble
    new SequenceNode('handle-trouble', [
      new ConditionNode('trouble-nearby', (bb) => {
        const trouble = bb.get('nearestTrouble') as any;
        return trouble && trouble.distance < 5;
      }),
      new ActionNode('move-to-trouble', (bb) => {
        const trouble = bb.get('nearestTrouble') as any;
        const self = bb.get('self') as any;
        return moveToward(self, trouble.position) ? 'success' : 'running';
      }),
      new ActionNode('clear-trouble', (bb) => {
        const trouble = bb.get('nearestTrouble') as any;
        return clearTrouble(trouble) ? 'success' : 'running';
      }),
    ]),
    
    // Priority 2: Tend to crops
    new SequenceNode('tend-crops', [
      new ConditionNode('crops-need-care', (bb) => {
        return (bb.get('untendedCrops') as any[])?.length > 0;
      }),
      new ActionNode('move-to-crop', (bb) => {
        const crop = (bb.get('untendedCrops') as any[])[0];
        const self = bb.get('self') as any;
        return moveToward(self, crop.position) ? 'success' : 'running';
      }),
      new ActionNode('tend-crop', (bb) => {
        const crop = (bb.get('untendedCrops') as any[])[0];
        return tendCrop(crop) ? 'success' : 'running';
      }),
    ]),
    
    // Priority 3: Idle/wander
    new ActionNode('wander', (bb) => {
      const self = bb.get('self') as any;
      return wander(self) ? 'success' : 'running';
    }),
  ]);
}
```

---

## Scene Management

### Scene Graph with Transition System

```typescript
type SceneId = string;

interface Scene {
  id: SceneId;
  onEnter(params?: Record<string, unknown>): void | Promise<void>;
  onUpdate(dt: number): void;
  onRender(): void;
  onExit(): void | Promise<void>;
  onPause?(): void;
  onResume?(): void;
}

type TransitionType = 'fade' | 'slide' | 'instant' | 'crossfade';

interface TransitionConfig {
  type: TransitionType;
  durationMs: number;
  easing?: (t: number) => number;
}

class SceneManager {
  private scenes = new Map<SceneId, Scene>();
  private stack: Scene[] = [];
  private transitioning = false;
  
  register(scene: Scene): void {
    this.scenes.set(scene.id, scene);
  }
  
  get current(): Scene | undefined {
    return this.stack[this.stack.length - 1];
  }
  
  async goto(
    sceneId: SceneId,
    params?: Record<string, unknown>,
    transition?: TransitionConfig
  ): Promise<void> {
    if (this.transitioning) return;
    
    const nextScene = this.scenes.get(sceneId);
    if (!nextScene) throw new Error(`Scene "${sceneId}" not found`);
    
    this.transitioning = true;
    
    const prevScene = this.current;
    
    if (transition && prevScene) {
      await this.playTransition(prevScene, nextScene, transition);
    }
    
    if (prevScene) {
      await prevScene.onExit();
    }
    
    // Replace entire stack
    this.stack = [nextScene];
    await nextScene.onEnter(params);
    
    this.transitioning = false;
  }
  
  async push(
    sceneId: SceneId,
    params?: Record<string, unknown>
  ): Promise<void> {
    if (this.transitioning) return;
    
    const nextScene = this.scenes.get(sceneId);
    if (!nextScene) throw new Error(`Scene "${sceneId}" not found`);
    
    this.current?.onPause?.();
    this.stack.push(nextScene);
    await nextScene.onEnter(params);
  }
  
  async pop(): Promise<void> {
    if (this.stack.length <= 1 || this.transitioning) return;
    
    const current = this.stack.pop()!;
    await current.onExit();
    this.current?.onResume?.();
  }
  
  update(dt: number): void {
    if (this.transitioning) return;
    this.current?.onUpdate(dt);
  }
  
  render(): void {
    this.current?.onRender();
  }
  
  private async playTransition(
    from: Scene,
    to: Scene,
    config: TransitionConfig
  ): Promise<void> {
    return new Promise(resolve => {
      const start = performance.now();
      const easing = config.easing || ((t: number) => t);
      
      const animate = (now: number) => {
        const elapsed = now - start;
        const progress = Math.min(elapsed / config.durationMs, 1);
        const easedProgress = easing(progress);
        
        this.renderTransitionFrame(config.type, from, to, easedProgress);
        
        if (progress < 1) {
          requestAnimationFrame(animate);
        } else {
          resolve();
        }
      };
      
      requestAnimationFrame(animate);
    });
  }
  
  private renderTransitionFrame(
    type: TransitionType,
    _from: Scene,
    _to: Scene,
    _progress: number
  ): void {
    switch (type) {
      case 'fade':
        // Render from scene with (1 - progress) opacity
        // Render to scene with progress opacity
        break;
      case 'slide':
        // Translate from scene out, to scene in
        break;
      case 'crossfade':
        // Both visible, blend
        break;
      case 'instant':
        break;
    }
  }
}
```

### React-Based Scene System

```typescript
import { createContext, useContext, useState, useCallback, ReactNode } from 'react';
import { AnimatePresence, motion } from 'framer-motion';

interface SceneConfig {
  id: string;
  component: React.ComponentType<any>;
}

interface SceneContextType {
  currentScene: string;
  goto: (sceneId: string, params?: Record<string, unknown>) => void;
  params: Record<string, unknown>;
}

const SceneContext = createContext<SceneContextType | null>(null);

export function useScene() {
  const ctx = useContext(SceneContext);
  if (!ctx) throw new Error('useScene must be used within SceneProvider');
  return ctx;
}

export function SceneProvider({ 
  scenes, 
  initialScene,
  children 
}: { 
  scenes: SceneConfig[];
  initialScene: string;
  children?: ReactNode;
}) {
  const [currentScene, setCurrentScene] = useState(initialScene);
  const [params, setParams] = useState<Record<string, unknown>>({});
  
  const goto = useCallback((sceneId: string, newParams?: Record<string, unknown>) => {
    setParams(newParams || {});
    setCurrentScene(sceneId);
  }, []);
  
  const ActiveScene = scenes.find(s => s.id === currentScene)?.component;
  
  return (
    <SceneContext.Provider value={{ currentScene, goto, params }}>
      <AnimatePresence mode="wait">
        <motion.div
          key={currentScene}
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          exit={{ opacity: 0, y: -20 }}
          transition={{ duration: 0.3 }}
        >
          {ActiveScene && <ActiveScene {...params} />}
        </motion.div>
      </AnimatePresence>
      {children}
    </SceneContext.Provider>
  );
}
```

---

## Spatial Data Structures

### Hex Grid System

```typescript
interface HexCoord {
  q: number; // column
  r: number; // row
  s: number; // q + r + s = 0
}

function hexCoord(q: number, r: number): HexCoord {
  return { q, r, s: -q - r };
}

function hexDistance(a: HexCoord, b: HexCoord): number {
  return Math.max(
    Math.abs(a.q - b.q),
    Math.abs(a.r - b.r),
    Math.abs(a.s - b.s)
  );
}

function hexNeighbors(hex: HexCoord): HexCoord[] {
  const directions = [
    [1, 0], [1, -1], [0, -1],
    [-1, 0], [-1, 1], [0, 1],
  ];
  return directions.map(([dq, dr]) => hexCoord(hex.q + dq, hex.r + dr));
}

function hexRing(center: HexCoord, radius: number): HexCoord[] {
  if (radius === 0) return [center];
  
  const results: HexCoord[] = [];
  let hex = hexCoord(
    center.q + radius,
    center.r - radius
  );
  
  const directions = [
    [-1, 1], [-1, 0], [0, -1],
    [1, -1], [1, 0], [0, 1],
  ];
  
  for (const [dq, dr] of directions) {
    for (let i = 0; i < radius; i++) {
      results.push(hex);
      hex = hexCoord(hex.q + dq, hex.r + dr);
    }
  }
  
  return results;
}

function hexToPixel(hex: HexCoord, size: number): { x: number; y: number } {
  const x = size * (Math.sqrt(3) * hex.q + Math.sqrt(3) / 2 * hex.r);
  const y = size * (3 / 2 * hex.r);
  return { x, y };
}

function pixelToHex(point: { x: number; y: number }, size: number): HexCoord {
  const q = (Math.sqrt(3) / 3 * point.x - 1 / 3 * point.y) / size;
  const r = (2 / 3 * point.y) / size;
  return hexRound({ q, r, s: -q - r });
}

function hexRound(frac: { q: number; r: number; s: number }): HexCoord {
  let rq = Math.round(frac.q);
  let rr = Math.round(frac.r);
  let rs = Math.round(frac.s);
  
  const dq = Math.abs(rq - frac.q);
  const dr = Math.abs(rr - frac.r);
  const ds = Math.abs(rs - frac.s);
  
  if (dq > dr && dq > ds) {
    rq = -rr - rs;
  } else if (dr > ds) {
    rr = -rq - rs;
  } else {
    rs = -rq - rr;
  }
  
  return { q: rq, r: rr, s: rs };
}
```

### A* Pathfinding on Hex Grid

```typescript
function hexPathfind(
  start: HexCoord,
  goal: HexCoord,
  isWalkable: (hex: HexCoord) => boolean,
  moveCost: (from: HexCoord, to: HexCoord) => number = () => 1
): HexCoord[] | null {
  const openSet = new PriorityQueue<HexCoord>();
  const cameFrom = new Map<string, HexCoord>();
  const gScore = new Map<string, number>();
  const key = (h: HexCoord) => `${h.q},${h.r}`;
  
  openSet.enqueue(start, 0);
  gScore.set(key(start), 0);
  
  while (!openSet.isEmpty()) {
    const current = openSet.dequeue()!;
    
    if (current.q === goal.q && current.r === goal.r) {
      return reconstructPath(cameFrom, current, key);
    }
    
    for (const neighbor of hexNeighbors(current)) {
      if (!isWalkable(neighbor)) continue;
      
      const tentativeG = (gScore.get(key(current)) || Infinity) + moveCost(current, neighbor);
      
      if (tentativeG < (gScore.get(key(neighbor)) || Infinity)) {
        cameFrom.set(key(neighbor), current);
        gScore.set(key(neighbor), tentativeG);
        const f = tentativeG + hexDistance(neighbor, goal);
        openSet.enqueue(neighbor, f);
      }
    }
  }
  
  return null;
}

function reconstructPath(
  cameFrom: Map<string, HexCoord>,
  current: HexCoord,
  key: (h: HexCoord) => string
): HexCoord[] {
  const path = [current];
  while (cameFrom.has(key(current))) {
    current = cameFrom.get(key(current))!;
    path.unshift(current);
  }
  return path;
}

class PriorityQueue<T> {
  private items: { value: T; priority: number }[] = [];
  
  enqueue(value: T, priority: number): void {
    this.items.push({ value, priority });
    this.items.sort((a, b) => a.priority - b.priority);
  }
  
  dequeue(): T | undefined {
    return this.items.shift()?.value;
  }
  
  isEmpty(): boolean {
    return this.items.length === 0;
  }
}
```

---

## Game Loop Patterns

### Fixed Timestep Loop

```typescript
class GameLoop {
  private lastTime = 0;
  private accumulator = 0;
  private running = false;
  
  constructor(
    private readonly fixedDt: number = 1 / 60, // 60 updates/sec
    private readonly maxFrameTime: number = 0.25 // prevent spiral of death
  ) {}
  
  start(
    update: (dt: number) => void,
    render: (interpolation: number) => void
  ): void {
    this.running = true;
    this.lastTime = performance.now() / 1000;
    
    const tick = (now: number) => {
      if (!this.running) return;
      
      const currentTime = now / 1000;
      let frameTime = currentTime - this.lastTime;
      this.lastTime = currentTime;
      
      // Clamp to prevent spiral
      if (frameTime > this.maxFrameTime) {
        frameTime = this.maxFrameTime;
      }
      
      this.accumulator += frameTime;
      
      while (this.accumulator >= this.fixedDt) {
        update(this.fixedDt);
        this.accumulator -= this.fixedDt;
      }
      
      const interpolation = this.accumulator / this.fixedDt;
      render(interpolation);
      
      requestAnimationFrame(tick);
    };
    
    requestAnimationFrame(tick);
  }
  
  stop(): void {
    this.running = false;
  }
}
```

### Frame Budget Monitor

```typescript
class FrameBudgetMonitor {
  private frameTimes: number[] = [];
  private readonly budgetMs: number;
  
  constructor(targetFps: number = 60) {
    this.budgetMs = 1000 / targetFps;
  }
  
  measure<T>(label: string, fn: () => T): T {
    const start = performance.now();
    const result = fn();
    const elapsed = performance.now() - start;
    
    if (elapsed > this.budgetMs * 0.5) {
      console.warn(`[FrameBudget] ${label}: ${elapsed.toFixed(2)}ms (${((elapsed / this.budgetMs) * 100).toFixed(0)}% of budget)`);
    }
    
    return result;
  }
  
  recordFrame(frameTimeMs: number): void {
    this.frameTimes.push(frameTimeMs);
    if (this.frameTimes.length > 120) {
      this.frameTimes.shift();
    }
  }
  
  getStats(): { avg: number; p95: number; p99: number; dropped: number } {
    const sorted = [...this.frameTimes].sort((a, b) => a - b);
    const len = sorted.length;
    
    return {
      avg: sorted.reduce((a, b) => a + b, 0) / len,
      p95: sorted[Math.floor(len * 0.95)] || 0,
      p99: sorted[Math.floor(len * 0.99)] || 0,
      dropped: this.frameTimes.filter(t => t > this.budgetMs).length,
    };
  }
}
```

---

*Extended Game Patterns Reference — Game Engineering Team*
