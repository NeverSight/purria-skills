# Game UI Patterns

UI component library patterns, responsive design, animation systems, and accessibility for game interfaces.

---

## Component Architecture

### Compound Component Pattern

Build complex game UI from composable pieces that share context internally.

```typescript
import { createContext, useContext, useState, useCallback, useRef, useEffect, ReactNode } from 'react';

// Generic compound component factory
function createCompoundContext<T>(name: string) {
  const Context = createContext<T | null>(null);
  
  function useCompoundContext(): T {
    const ctx = useContext(Context);
    if (!ctx) throw new Error(`${name} components must be used within <${name}>`);
    return ctx;
  }
  
  return { Provider: Context.Provider, useContext: useCompoundContext };
}
```

### Game Card Component

```typescript
interface GameCardContext {
  flipped: boolean;
  flip: () => void;
  selected: boolean;
  select: () => void;
  disabled: boolean;
}

const { Provider: CardProvider, useContext: useCardContext } = 
  createCompoundContext<GameCardContext>('GameCard');

function GameCard({ 
  children,
  disabled = false,
  onFlip,
  onSelect,
}: {
  children: ReactNode;
  disabled?: boolean;
  onFlip?: (flipped: boolean) => void;
  onSelect?: () => void;
}) {
  const [flipped, setFlipped] = useState(false);
  const [selected, setSelected] = useState(false);
  
  const flip = useCallback(() => {
    if (disabled) return;
    setFlipped(prev => {
      onFlip?.(!prev);
      return !prev;
    });
  }, [disabled, onFlip]);
  
  const select = useCallback(() => {
    if (disabled) return;
    setSelected(prev => !prev);
    onSelect?.();
  }, [disabled, onSelect]);
  
  return (
    <CardProvider value={{ flipped, flip, selected, select, disabled }}>
      <div 
        className={cn(
          'game-card',
          flipped && 'game-card--flipped',
          selected && 'game-card--selected',
          disabled && 'game-card--disabled',
        )}
        role="button"
        tabIndex={disabled ? -1 : 0}
        aria-label="Game card"
        aria-pressed={selected}
        onKeyDown={(e) => {
          if (e.key === 'Enter' || e.key === ' ') {
            e.preventDefault();
            select();
          }
        }}
      >
        {children}
      </div>
    </CardProvider>
  );
}

GameCard.Front = function Front({ children }: { children: ReactNode }) {
  const { flipped } = useCardContext();
  return (
    <div className={cn('game-card__face game-card__front', !flipped && 'visible')}>
      {children}
    </div>
  );
};

GameCard.Back = function Back({ children }: { children: ReactNode }) {
  const { flipped } = useCardContext();
  return (
    <div className={cn('game-card__face game-card__back', flipped && 'visible')}>
      {children}
    </div>
  );
};

GameCard.FlipButton = function FlipButton() {
  const { flip, disabled } = useCardContext();
  return (
    <button onClick={flip} disabled={disabled} aria-label="Flip card">
      🔄
    </button>
  );
};
```

### Game HUD (Heads-Up Display)

```typescript
interface HUDLayout {
  topLeft?: ReactNode;
  topCenter?: ReactNode;
  topRight?: ReactNode;
  bottomLeft?: ReactNode;
  bottomCenter?: ReactNode;
  bottomRight?: ReactNode;
  center?: ReactNode;
}

function GameHUD({ layout, className }: { layout: HUDLayout; className?: string }) {
  return (
    <div className={cn('game-hud', className)} aria-label="Game interface">
      <div className="game-hud__top">
        <div className="game-hud__slot game-hud__top-left">{layout.topLeft}</div>
        <div className="game-hud__slot game-hud__top-center">{layout.topCenter}</div>
        <div className="game-hud__slot game-hud__top-right">{layout.topRight}</div>
      </div>
      
      {layout.center && (
        <div className="game-hud__center">{layout.center}</div>
      )}
      
      <div className="game-hud__bottom">
        <div className="game-hud__slot game-hud__bottom-left">{layout.bottomLeft}</div>
        <div className="game-hud__slot game-hud__bottom-center">{layout.bottomCenter}</div>
        <div className="game-hud__slot game-hud__bottom-right">{layout.bottomRight}</div>
      </div>
    </div>
  );
}

// CSS (Tailwind)
const HUD_STYLES = `
  .game-hud {
    @apply fixed inset-0 pointer-events-none z-50;
    @apply flex flex-col justify-between p-4;
  }
  .game-hud__top, .game-hud__bottom {
    @apply flex justify-between items-start;
  }
  .game-hud__center {
    @apply flex justify-center items-center;
  }
  .game-hud__slot {
    @apply pointer-events-auto;
  }
`;
```

---

## Resource Display Components

### Animated Currency Display

```typescript
import { motion, useSpring, useTransform, useMotionValue } from 'framer-motion';

function CurrencyDisplay({ 
  value, 
  icon, 
  label,
  color = 'gold',
  showDelta = true,
}: { 
  value: number;
  icon: string;
  label: string;
  color?: string;
  showDelta?: boolean;
}) {
  const prevValue = useRef(value);
  const [delta, setDelta] = useState<number | null>(null);
  
  const springValue = useSpring(value, { stiffness: 100, damping: 30 });
  const displayValue = useTransform(springValue, (v) => Math.floor(v).toLocaleString());
  
  useEffect(() => {
    if (showDelta && value !== prevValue.current) {
      setDelta(value - prevValue.current);
      const timeout = setTimeout(() => setDelta(null), 2000);
      prevValue.current = value;
      return () => clearTimeout(timeout);
    }
    springValue.set(value);
  }, [value, springValue, showDelta]);
  
  return (
    <div className="currency-display" aria-label={`${label}: ${value}`}>
      <span className="currency-display__icon">{icon}</span>
      <motion.span className="currency-display__value">
        {displayValue}
      </motion.span>
      
      {delta !== null && (
        <motion.span
          className={cn(
            'currency-display__delta',
            delta > 0 ? 'text-green-400' : 'text-red-400'
          )}
          initial={{ opacity: 1, y: 0 }}
          animate={{ opacity: 0, y: -20 }}
          transition={{ duration: 1.5 }}
        >
          {delta > 0 ? '+' : ''}{delta.toLocaleString()}
        </motion.span>
      )}
    </div>
  );
}

// Health/Energy Bar
function ResourceBar({
  current,
  max,
  color = '#4ade80',
  backgroundColor = '#1a1a2e',
  label,
  showText = true,
  animate = true,
}: {
  current: number;
  max: number;
  color?: string;
  backgroundColor?: string;
  label: string;
  showText?: boolean;
  animate?: boolean;
}) {
  const percentage = Math.min(100, Math.max(0, (current / max) * 100));
  
  return (
    <div 
      className="resource-bar"
      role="progressbar"
      aria-valuenow={current}
      aria-valuemin={0}
      aria-valuemax={max}
      aria-label={label}
    >
      <div 
        className="resource-bar__track"
        style={{ backgroundColor }}
      >
        <motion.div
          className="resource-bar__fill"
          style={{ backgroundColor: color }}
          initial={false}
          animate={{ width: `${percentage}%` }}
          transition={animate ? { type: 'spring', stiffness: 100, damping: 20 } : { duration: 0 }}
        />
      </div>
      
      {showText && (
        <span className="resource-bar__text">
          {current} / {max}
        </span>
      )}
    </div>
  );
}
```

### Countdown Timer

```typescript
function CountdownTimer({
  targetTime,
  onComplete,
  format = 'hh:mm:ss',
}: {
  targetTime: Date;
  onComplete?: () => void;
  format?: 'hh:mm:ss' | 'mm:ss' | 'compact';
}) {
  const [remaining, setRemaining] = useState(0);
  
  useEffect(() => {
    const update = () => {
      const diff = Math.max(0, targetTime.getTime() - Date.now());
      setRemaining(diff);
      if (diff <= 0) onComplete?.();
    };
    
    update();
    const interval = setInterval(update, 1000);
    return () => clearInterval(interval);
  }, [targetTime, onComplete]);
  
  const hours = Math.floor(remaining / 3600000);
  const minutes = Math.floor((remaining % 3600000) / 60000);
  const seconds = Math.floor((remaining % 60000) / 1000);
  
  const formatTime = () => {
    switch (format) {
      case 'hh:mm:ss':
        return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
      case 'mm:ss':
        return `${(hours * 60 + minutes).toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
      case 'compact':
        if (hours > 0) return `${hours}h ${minutes}m`;
        if (minutes > 0) return `${minutes}m ${seconds}s`;
        return `${seconds}s`;
    }
  };
  
  return (
    <span className="countdown-timer" aria-label={`Time remaining: ${formatTime()}`}>
      {formatTime()}
    </span>
  );
}
```

---

## Responsive Game Layout

### Mobile-First Game Scaling

```typescript
interface GameDimensions {
  designWidth: number;
  designHeight: number;
  minScale: number;
  maxScale: number;
}

function useGameScale(dimensions: GameDimensions = {
  designWidth: 390,
  designHeight: 844,
  minScale: 0.5,
  maxScale: 2.0,
}) {
  const containerRef = useRef<HTMLDivElement>(null);
  const [scale, setScale] = useState(1);
  const [orientation, setOrientation] = useState<'portrait' | 'landscape'>('portrait');
  
  useEffect(() => {
    const observer = new ResizeObserver(([entry]) => {
      const { width, height } = entry.contentRect;
      
      const scaleX = width / dimensions.designWidth;
      const scaleY = height / dimensions.designHeight;
      const newScale = Math.min(
        Math.max(Math.min(scaleX, scaleY), dimensions.minScale),
        dimensions.maxScale
      );
      
      setScale(newScale);
      setOrientation(width > height ? 'landscape' : 'portrait');
    });
    
    if (containerRef.current?.parentElement) {
      observer.observe(containerRef.current.parentElement);
    }
    
    return () => observer.disconnect();
  }, [dimensions]);
  
  return { scale, orientation, containerRef };
}

// Safe area insets for notched devices
function useSafeAreaInsets() {
  const [insets, setInsets] = useState({
    top: 0, right: 0, bottom: 0, left: 0,
  });
  
  useEffect(() => {
    const update = () => {
      const style = getComputedStyle(document.documentElement);
      setInsets({
        top: parseInt(style.getPropertyValue('env(safe-area-inset-top)') || '0'),
        right: parseInt(style.getPropertyValue('env(safe-area-inset-right)') || '0'),
        bottom: parseInt(style.getPropertyValue('env(safe-area-inset-bottom)') || '0'),
        left: parseInt(style.getPropertyValue('env(safe-area-inset-left)') || '0'),
      });
    };
    
    update();
    window.addEventListener('resize', update);
    return () => window.removeEventListener('resize', update);
  }, []);
  
  return insets;
}

// Responsive breakpoints for game UI
const GAME_BREAKPOINTS = {
  mobile: { max: 480 },
  tablet: { min: 481, max: 1024 },
  desktop: { min: 1025 },
} as const;

function useGameBreakpoint(): 'mobile' | 'tablet' | 'desktop' {
  const [breakpoint, setBreakpoint] = useState<'mobile' | 'tablet' | 'desktop'>('mobile');
  
  useEffect(() => {
    const mediaQueries = {
      mobile: window.matchMedia('(max-width: 480px)'),
      tablet: window.matchMedia('(min-width: 481px) and (max-width: 1024px)'),
      desktop: window.matchMedia('(min-width: 1025px)'),
    };
    
    const update = () => {
      if (mediaQueries.desktop.matches) setBreakpoint('desktop');
      else if (mediaQueries.tablet.matches) setBreakpoint('tablet');
      else setBreakpoint('mobile');
    };
    
    update();
    Object.values(mediaQueries).forEach(mq => mq.addEventListener('change', update));
    return () => {
      Object.values(mediaQueries).forEach(mq => mq.removeEventListener('change', update));
    };
  }, []);
  
  return breakpoint;
}
```

### Adaptive Layout Component

```typescript
function GameLayout({ children }: { children: ReactNode }) {
  const breakpoint = useGameBreakpoint();
  const { scale, orientation, containerRef } = useGameScale();
  const insets = useSafeAreaInsets();
  
  return (
    <div
      ref={containerRef}
      className={cn(
        'game-layout',
        `game-layout--${breakpoint}`,
        `game-layout--${orientation}`,
      )}
      style={{
        '--game-scale': scale,
        '--safe-top': `${insets.top}px`,
        '--safe-bottom': `${insets.bottom}px`,
        '--safe-left': `${insets.left}px`,
        '--safe-right': `${insets.right}px`,
      } as React.CSSProperties}
    >
      {children}
    </div>
  );
}

const LAYOUT_CSS = `
  .game-layout {
    width: 100vw;
    height: 100dvh;
    overflow: hidden;
    padding-top: var(--safe-top);
    padding-bottom: var(--safe-bottom);
    padding-left: var(--safe-left);
    padding-right: var(--safe-right);
  }
  
  .game-layout--mobile {
    font-size: calc(16px * var(--game-scale));
  }
  
  .game-layout--tablet {
    font-size: calc(18px * var(--game-scale));
    max-width: 768px;
    margin: 0 auto;
  }
  
  .game-layout--desktop {
    font-size: calc(16px * var(--game-scale));
    max-width: 1024px;
    margin: 0 auto;
  }
`;
```

---

## Animation System

### Animation Queue for Sequenced Game Events

```typescript
interface GameAnimation {
  id: string;
  duration: number;
  execute: () => Promise<void>;
  priority?: number;
  blocking?: boolean; // If true, blocks subsequent animations
}

class AnimationQueue {
  private queue: GameAnimation[] = [];
  private running = false;
  private currentAnimation: GameAnimation | null = null;
  
  enqueue(animation: GameAnimation): void {
    this.queue.push(animation);
    this.queue.sort((a, b) => (b.priority || 0) - (a.priority || 0));
    
    if (!this.running) {
      this.processQueue();
    }
  }
  
  enqueueParallel(animations: GameAnimation[]): void {
    const parallel: GameAnimation = {
      id: `parallel-${Date.now()}`,
      duration: Math.max(...animations.map(a => a.duration)),
      execute: () => Promise.all(animations.map(a => a.execute())).then(() => {}),
      blocking: true,
    };
    
    this.enqueue(parallel);
  }
  
  private async processQueue(): Promise<void> {
    this.running = true;
    
    while (this.queue.length > 0) {
      const animation = this.queue.shift()!;
      this.currentAnimation = animation;
      
      try {
        await Promise.race([
          animation.execute(),
          this.timeout(animation.duration + 500), // Safety timeout
        ]);
      } catch (error) {
        console.error(`Animation "${animation.id}" failed:`, error);
      }
      
      this.currentAnimation = null;
    }
    
    this.running = false;
  }
  
  clear(): void {
    this.queue = [];
  }
  
  private timeout(ms: number): Promise<never> {
    return new Promise((_, reject) => {
      setTimeout(() => reject(new Error('Animation timeout')), ms);
    });
  }
}

// React hook for animation queue
function useAnimationQueue() {
  const queueRef = useRef(new AnimationQueue());
  
  const animate = useCallback((animation: GameAnimation) => {
    queueRef.current.enqueue(animation);
  }, []);
  
  const animateParallel = useCallback((animations: GameAnimation[]) => {
    queueRef.current.enqueueParallel(animations);
  }, []);
  
  return { animate, animateParallel, clear: () => queueRef.current.clear() };
}
```

### Particle System (CSS-based)

```typescript
interface Particle {
  id: string;
  x: number;
  y: number;
  emoji: string;
  size: number;
  duration: number;
  delay: number;
}

function ParticleEmitter({
  particles,
  origin,
}: {
  particles: Particle[];
  origin: { x: number; y: number };
}) {
  return (
    <div className="particle-emitter" style={{ left: origin.x, top: origin.y }}>
      {particles.map(p => (
        <motion.span
          key={p.id}
          className="particle"
          style={{ fontSize: p.size }}
          initial={{ x: 0, y: 0, opacity: 1, scale: 1 }}
          animate={{
            x: p.x,
            y: p.y,
            opacity: 0,
            scale: 0,
            rotate: Math.random() * 360,
          }}
          transition={{
            duration: p.duration / 1000,
            delay: p.delay / 1000,
            ease: 'easeOut',
          }}
        >
          {p.emoji}
        </motion.span>
      ))}
    </div>
  );
}

function createCelebrationParticles(count: number = 20): Particle[] {
  return Array.from({ length: count }, (_, i) => ({
    id: `particle-${i}`,
    x: (Math.random() - 0.5) * 300,
    y: -Math.random() * 200 - 50,
    emoji: ['✨', '🌟', '⭐', '💫', '🎉'][Math.floor(Math.random() * 5)],
    size: Math.random() * 16 + 12,
    duration: Math.random() * 1000 + 500,
    delay: Math.random() * 300,
  }));
}

function createCoinShowerParticles(count: number = 15): Particle[] {
  return Array.from({ length: count }, (_, i) => ({
    id: `coin-${i}`,
    x: (Math.random() - 0.5) * 200,
    y: Math.random() * 200 + 100,
    emoji: '🪙',
    size: Math.random() * 12 + 16,
    duration: Math.random() * 800 + 400,
    delay: Math.random() * 500,
  }));
}
```

---

## Accessibility Patterns

### Focus Management for Games

```typescript
function useFocusTrap(enabled: boolean = true) {
  const containerRef = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    if (!enabled || !containerRef.current) return;
    
    const container = containerRef.current;
    const focusableSelectors = [
      'button:not([disabled])',
      '[tabindex]:not([tabindex="-1"])',
      'input:not([disabled])',
      'select:not([disabled])',
      'a[href]',
    ].join(', ');
    
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      
      const focusableElements = container.querySelectorAll<HTMLElement>(focusableSelectors);
      if (focusableElements.length === 0) return;
      
      const firstElement = focusableElements[0];
      const lastElement = focusableElements[focusableElements.length - 1];
      
      if (e.shiftKey && document.activeElement === firstElement) {
        e.preventDefault();
        lastElement.focus();
      } else if (!e.shiftKey && document.activeElement === lastElement) {
        e.preventDefault();
        firstElement.focus();
      }
    };
    
    container.addEventListener('keydown', handleKeyDown);
    
    // Auto-focus first element
    const firstFocusable = container.querySelector<HTMLElement>(focusableSelectors);
    firstFocusable?.focus();
    
    return () => container.removeEventListener('keydown', handleKeyDown);
  }, [enabled]);
  
  return containerRef;
}

// Screen reader announcer for game events
function useGameAnnouncer() {
  const [message, setMessage] = useState('');
  
  const announce = useCallback((text: string, priority: 'polite' | 'assertive' = 'polite') => {
    setMessage('');
    requestAnimationFrame(() => setMessage(text));
  }, []);
  
  const Announcer = useCallback(() => (
    <>
      <div aria-live="polite" aria-atomic="true" className="sr-only">
        {message}
      </div>
    </>
  ), [message]);
  
  return { announce, Announcer };
}

// Reduced motion preference
function useReducedMotion(): boolean {
  const [reduced, setReduced] = useState(false);
  
  useEffect(() => {
    const mq = window.matchMedia('(prefers-reduced-motion: reduce)');
    setReduced(mq.matches);
    
    const handler = (e: MediaQueryListEvent) => setReduced(e.matches);
    mq.addEventListener('change', handler);
    return () => mq.removeEventListener('change', handler);
  }, []);
  
  return reduced;
}

// High contrast mode detection
function useHighContrast(): boolean {
  const [highContrast, setHighContrast] = useState(false);
  
  useEffect(() => {
    const mq = window.matchMedia('(forced-colors: active)');
    setHighContrast(mq.matches);
    
    const handler = (e: MediaQueryListEvent) => setHighContrast(e.matches);
    mq.addEventListener('change', handler);
    return () => mq.removeEventListener('change', handler);
  }, []);
  
  return highContrast;
}
```

### ARIA Patterns for Game Elements

```typescript
// Accessible game grid (hex grid, board)
function AccessibleGameGrid({
  rows,
  cols,
  cells,
  onCellSelect,
  label,
}: {
  rows: number;
  cols: number;
  cells: { row: number; col: number; content: string; disabled?: boolean }[];
  onCellSelect: (row: number, col: number) => void;
  label: string;
}) {
  const [focusedCell, setFocusedCell] = useState({ row: 0, col: 0 });
  
  const handleKeyDown = (e: React.KeyboardEvent) => {
    const { row, col } = focusedCell;
    let newRow = row;
    let newCol = col;
    
    switch (e.key) {
      case 'ArrowUp': newRow = Math.max(0, row - 1); break;
      case 'ArrowDown': newRow = Math.min(rows - 1, row + 1); break;
      case 'ArrowLeft': newCol = Math.max(0, col - 1); break;
      case 'ArrowRight': newCol = Math.min(cols - 1, col + 1); break;
      case 'Enter':
      case ' ':
        e.preventDefault();
        onCellSelect(row, col);
        return;
      default:
        return;
    }
    
    e.preventDefault();
    setFocusedCell({ row: newRow, col: newCol });
  };
  
  return (
    <div
      role="grid"
      aria-label={label}
      onKeyDown={handleKeyDown}
    >
      {Array.from({ length: rows }, (_, r) => (
        <div key={r} role="row">
          {Array.from({ length: cols }, (_, c) => {
            const cell = cells.find(cell => cell.row === r && cell.col === c);
            const isFocused = focusedCell.row === r && focusedCell.col === c;
            
            return (
              <div
                key={c}
                role="gridcell"
                tabIndex={isFocused ? 0 : -1}
                aria-disabled={cell?.disabled}
                aria-label={cell?.content || `Row ${r + 1}, Column ${c + 1}`}
                onClick={() => onCellSelect(r, c)}
                ref={el => isFocused && el?.focus()}
              >
                {cell?.content}
              </div>
            );
          })}
        </div>
      ))}
    </div>
  );
}
```

---

## Touch & Gesture Handling

```typescript
interface GestureHandlers {
  onTap?: (point: { x: number; y: number }) => void;
  onDoubleTap?: (point: { x: number; y: number }) => void;
  onLongPress?: (point: { x: number; y: number }) => void;
  onSwipe?: (direction: 'up' | 'down' | 'left' | 'right', velocity: number) => void;
  onPinch?: (scale: number) => void;
}

function useGameGestures(
  ref: React.RefObject<HTMLElement>,
  handlers: GestureHandlers
) {
  useEffect(() => {
    const element = ref.current;
    if (!element) return;
    
    let touchStart: { x: number; y: number; time: number } | null = null;
    let lastTap = 0;
    let longPressTimer: ReturnType<typeof setTimeout> | null = null;
    
    const onTouchStart = (e: TouchEvent) => {
      const touch = e.touches[0];
      touchStart = { x: touch.clientX, y: touch.clientY, time: Date.now() };
      
      longPressTimer = setTimeout(() => {
        if (touchStart) {
          handlers.onLongPress?.({ x: touchStart.x, y: touchStart.y });
          touchStart = null;
        }
      }, 500);
    };
    
    const onTouchEnd = (e: TouchEvent) => {
      if (longPressTimer) clearTimeout(longPressTimer);
      if (!touchStart) return;
      
      const touch = e.changedTouches[0];
      const dx = touch.clientX - touchStart.x;
      const dy = touch.clientY - touchStart.y;
      const distance = Math.sqrt(dx * dx + dy * dy);
      const duration = Date.now() - touchStart.time;
      
      if (distance < 10) {
        // Tap or double tap
        const now = Date.now();
        if (now - lastTap < 300) {
          handlers.onDoubleTap?.({ x: touch.clientX, y: touch.clientY });
        } else {
          handlers.onTap?.({ x: touch.clientX, y: touch.clientY });
        }
        lastTap = now;
      } else if (distance > 50 && duration < 300) {
        // Swipe
        const velocity = distance / duration;
        const absDx = Math.abs(dx);
        const absDy = Math.abs(dy);
        
        if (absDx > absDy) {
          handlers.onSwipe?.(dx > 0 ? 'right' : 'left', velocity);
        } else {
          handlers.onSwipe?.(dy > 0 ? 'down' : 'up', velocity);
        }
      }
      
      touchStart = null;
    };
    
    element.addEventListener('touchstart', onTouchStart, { passive: true });
    element.addEventListener('touchend', onTouchEnd, { passive: true });
    
    return () => {
      element.removeEventListener('touchstart', onTouchStart);
      element.removeEventListener('touchend', onTouchEnd);
      if (longPressTimer) clearTimeout(longPressTimer);
    };
  }, [ref, handlers]);
}
```

---

*Game UI Patterns Reference — Game Engineering Team*
