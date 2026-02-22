# Tutorial & Onboarding Systems

Tutorial flows, contextual help, progressive disclosure, and player guidance design.

---

## Onboarding Philosophy

### Core Principles

1. **Play First, Learn Second**: Players should be doing something fun within 30 seconds
2. **Contextual, Not Front-Loaded**: Teach mechanics when they become relevant
3. **Show, Don't Tell**: Demonstrate through interaction, not text walls
4. **Respect Player Intelligence**: Experienced players should be able to skip
5. **Invisible When Mastered**: Tutorial UI should vanish once the player understands

---

## Tutorial Flow Engine

### State Machine Approach

```typescript
type TutorialPhase = 
  | 'inactive'
  | 'awaiting_trigger'
  | 'showing_prompt'
  | 'awaiting_action'
  | 'showing_feedback'
  | 'completed';

interface TutorialStep {
  id: string;
  phase: TutorialPhase;
  trigger: StepTrigger;
  prompt: StepPrompt;
  validation: StepValidation;
  feedback: StepFeedback;
  next?: string | null;
  prerequisites?: string[];
  skipCondition?: (state: GameState) => boolean;
}

type StepTrigger =
  | { type: 'screen_enter'; screen: string }
  | { type: 'action_complete'; action: string }
  | { type: 'state_change'; predicate: (state: GameState) => boolean }
  | { type: 'delay'; ms: number; after?: string }
  | { type: 'manual' };

interface StepPrompt {
  style: 'tooltip' | 'modal' | 'spotlight' | 'coach_mark' | 'inline_hint';
  title?: string;
  message: string;
  character?: {
    id: string;
    emotion: 'happy' | 'neutral' | 'excited' | 'thinking';
    position: 'left' | 'right' | 'center';
  };
  highlight?: {
    selector: string;
    padding?: number;
    shape?: 'circle' | 'rectangle' | 'pill';
  };
  position?: 'auto' | 'top' | 'bottom' | 'left' | 'right' | 'center';
  dismissable?: boolean;
  showArrow?: boolean;
}

type StepValidation =
  | { type: 'click'; target: string }
  | { type: 'action'; action: string; params?: Record<string, any> }
  | { type: 'state'; predicate: (state: GameState) => boolean }
  | { type: 'dismiss' }
  | { type: 'timer'; ms: number }
  | { type: 'any_of'; conditions: StepValidation[] };

interface StepFeedback {
  type: 'success' | 'info' | 'celebration';
  message?: string;
  reward?: { coins?: number; xp?: number; item?: string };
  animation?: 'confetti' | 'sparkle' | 'bounce' | 'none';
  duration?: number;
}

class TutorialFlowEngine {
  private steps: Map<string, TutorialStep> = new Map();
  private completedSteps: Set<string> = new Set();
  private activeStep: TutorialStep | null = null;
  private phase: TutorialPhase = 'inactive';
  
  private callbacks = {
    onShow: null as ((step: TutorialStep) => void) | null,
    onHide: null as (() => void) | null,
    onFeedback: null as ((feedback: StepFeedback) => void) | null,
    onComplete: null as ((stepId: string) => void) | null,
  };
  
  constructor(steps: TutorialStep[]) {
    for (const step of steps) {
      this.steps.set(step.id, step);
    }
    this.loadProgress();
  }
  
  on<K extends keyof typeof this.callbacks>(
    event: K,
    handler: NonNullable<typeof this.callbacks[K]>
  ): void {
    (this.callbacks as any)[event] = handler;
  }
  
  processEvent(event: {
    type: 'screen_enter' | 'action_complete' | 'state_change' | 'click';
    payload: any;
    gameState: GameState;
  }): void {
    // If showing prompt, check validation
    if (this.activeStep && this.phase === 'awaiting_action') {
      if (this.checkValidation(this.activeStep.validation, event)) {
        this.completeStep();
        return;
      }
    }
    
    // Check for new trigger
    if (!this.activeStep || this.phase === 'inactive') {
      for (const [id, step] of this.steps) {
        if (this.completedSteps.has(id)) continue;
        if (!this.prerequisitesMet(step)) continue;
        if (step.skipCondition?.(event.gameState)) {
          this.completedSteps.add(id);
          continue;
        }
        
        if (this.matchesTrigger(step.trigger, event)) {
          this.activateStep(step);
          break;
        }
      }
    }
  }
  
  private activateStep(step: TutorialStep): void {
    this.activeStep = step;
    this.phase = 'showing_prompt';
    this.callbacks.onShow?.(step);
    
    if (step.prompt.dismissable || step.validation.type === 'dismiss') {
      // Wait for user interaction
      this.phase = 'awaiting_action';
    } else {
      this.phase = 'awaiting_action';
    }
  }
  
  private completeStep(): void {
    if (!this.activeStep) return;
    
    const step = this.activeStep;
    this.phase = 'showing_feedback';
    
    // Show feedback
    this.callbacks.onFeedback?.(step.feedback);
    
    // Mark complete
    this.completedSteps.add(step.id);
    this.callbacks.onComplete?.(step.id);
    this.saveProgress();
    
    // Transition to next step after feedback
    const feedbackDuration = step.feedback.duration || 1500;
    setTimeout(() => {
      this.callbacks.onHide?.();
      this.activeStep = null;
      this.phase = 'inactive';
      
      // Auto-chain to next step
      if (step.next) {
        const nextStep = this.steps.get(step.next);
        if (nextStep && !this.completedSteps.has(nextStep.id)) {
          if (nextStep.trigger.type === 'manual') {
            this.activateStep(nextStep);
          }
        }
      }
    }, feedbackDuration);
  }
  
  dismiss(): void {
    if (this.activeStep && this.activeStep.prompt.dismissable) {
      if (this.activeStep.validation.type === 'dismiss') {
        this.completeStep();
      } else {
        this.callbacks.onHide?.();
        this.activeStep = null;
        this.phase = 'inactive';
      }
    }
  }
  
  skip(stepId: string): void {
    this.completedSteps.add(stepId);
    this.saveProgress();
    if (this.activeStep?.id === stepId) {
      this.callbacks.onHide?.();
      this.activeStep = null;
      this.phase = 'inactive';
    }
  }
  
  skipAll(): void {
    for (const id of this.steps.keys()) {
      this.completedSteps.add(id);
    }
    this.saveProgress();
    this.callbacks.onHide?.();
    this.activeStep = null;
    this.phase = 'inactive';
  }
  
  reset(): void {
    this.completedSteps.clear();
    this.activeStep = null;
    this.phase = 'inactive';
    this.saveProgress();
  }
  
  getProgress(): { completed: number; total: number; percentage: number } {
    const total = this.steps.size;
    const completed = this.completedSteps.size;
    return { completed, total, percentage: (completed / total) * 100 };
  }
  
  private matchesTrigger(trigger: StepTrigger, event: any): boolean {
    switch (trigger.type) {
      case 'screen_enter':
        return event.type === 'screen_enter' && event.payload === trigger.screen;
      case 'action_complete':
        return event.type === 'action_complete' && event.payload === trigger.action;
      case 'state_change':
        return trigger.predicate(event.gameState);
      default:
        return false;
    }
  }
  
  private checkValidation(validation: StepValidation, event: any): boolean {
    switch (validation.type) {
      case 'click':
        return event.type === 'click' && event.payload === validation.target;
      case 'action':
        return event.type === 'action_complete' && event.payload === validation.action;
      case 'state':
        return validation.predicate(event.gameState);
      case 'dismiss':
        return false; // Handled separately
      case 'any_of':
        return validation.conditions.some(c => this.checkValidation(c, event));
      default:
        return false;
    }
  }
  
  private prerequisitesMet(step: TutorialStep): boolean {
    if (!step.prerequisites) return true;
    return step.prerequisites.every(id => this.completedSteps.has(id));
  }
  
  private loadProgress(): void {
    try {
      const saved = localStorage.getItem('tutorial_progress');
      if (saved) {
        this.completedSteps = new Set(JSON.parse(saved));
      }
    } catch {}
  }
  
  private saveProgress(): void {
    try {
      localStorage.setItem(
        'tutorial_progress',
        JSON.stringify([...this.completedSteps])
      );
    } catch {}
  }
}
```

---

## Tutorial UI Components

### Spotlight Overlay

```typescript
import { motion, AnimatePresence } from 'framer-motion';

interface SpotlightProps {
  target: string; // CSS selector
  padding?: number;
  shape?: 'circle' | 'rectangle' | 'pill';
  children: ReactNode;
  onClickOutside?: () => void;
}

function SpotlightOverlay({ target, padding = 8, shape = 'rectangle', children, onClickOutside }: SpotlightProps) {
  const [rect, setRect] = useState<DOMRect | null>(null);
  
  useEffect(() => {
    const element = document.querySelector(target);
    if (!element) return;
    
    const updateRect = () => {
      setRect(element.getBoundingClientRect());
    };
    
    updateRect();
    
    const observer = new ResizeObserver(updateRect);
    observer.observe(element);
    window.addEventListener('scroll', updateRect, true);
    
    return () => {
      observer.disconnect();
      window.removeEventListener('scroll', updateRect, true);
    };
  }, [target]);
  
  if (!rect) return null;
  
  const cutout = {
    x: rect.x - padding,
    y: rect.y - padding,
    width: rect.width + padding * 2,
    height: rect.height + padding * 2,
  };
  
  const borderRadius = shape === 'circle'
    ? Math.max(cutout.width, cutout.height) / 2
    : shape === 'pill'
    ? cutout.height / 2
    : 8;
  
  return (
    <div className="spotlight-overlay" onClick={onClickOutside}>
      <svg className="spotlight-overlay__mask" width="100%" height="100%">
        <defs>
          <mask id="spotlight-mask">
            <rect width="100%" height="100%" fill="white" />
            <rect
              x={cutout.x}
              y={cutout.y}
              width={cutout.width}
              height={cutout.height}
              rx={borderRadius}
              fill="black"
            />
          </mask>
        </defs>
        <rect
          width="100%"
          height="100%"
          fill="rgba(0,0,0,0.7)"
          mask="url(#spotlight-mask)"
        />
      </svg>
      
      <motion.div
        className="spotlight-overlay__content"
        initial={{ opacity: 0, y: 10 }}
        animate={{ opacity: 1, y: 0 }}
        style={{
          position: 'absolute',
          top: cutout.y + cutout.height + 16,
          left: Math.max(16, cutout.x),
        }}
        onClick={(e) => e.stopPropagation()}
      >
        {children}
      </motion.div>
    </div>
  );
}

const SPOTLIGHT_CSS = `
  .spotlight-overlay {
    position: fixed;
    inset: 0;
    z-index: 9999;
  }
  .spotlight-overlay__mask {
    position: absolute;
    inset: 0;
  }
  .spotlight-overlay__content {
    max-width: 320px;
  }
`;
```

### Coach Mark / Tooltip

```typescript
interface CoachMarkProps {
  title?: string;
  message: string;
  character?: { id: string; emotion: string };
  position: 'top' | 'bottom' | 'left' | 'right';
  showArrow?: boolean;
  onDismiss?: () => void;
  onNext?: () => void;
  stepIndicator?: { current: number; total: number };
}

function CoachMark({
  title,
  message,
  character,
  position,
  showArrow = true,
  onDismiss,
  onNext,
  stepIndicator,
}: CoachMarkProps) {
  return (
    <motion.div
      className={`coach-mark coach-mark--${position}`}
      initial={{ opacity: 0, scale: 0.9 }}
      animate={{ opacity: 1, scale: 1 }}
      exit={{ opacity: 0, scale: 0.9 }}
      role="dialog"
      aria-label={title || 'Tutorial tip'}
    >
      {showArrow && <div className={`coach-mark__arrow coach-mark__arrow--${position}`} />}
      
      <div className="coach-mark__body">
        {character && (
          <div className="coach-mark__character">
            <img
              src={`/characters/${character.id}_${character.emotion}.png`}
              alt={`${character.id}`}
              className="coach-mark__avatar"
            />
          </div>
        )}
        
        <div className="coach-mark__content">
          {title && <h3 className="coach-mark__title">{title}</h3>}
          <p className="coach-mark__message">{message}</p>
        </div>
      </div>
      
      <div className="coach-mark__footer">
        {stepIndicator && (
          <span className="coach-mark__step-indicator">
            {stepIndicator.current} / {stepIndicator.total}
          </span>
        )}
        
        <div className="coach-mark__actions">
          {onDismiss && (
            <button className="coach-mark__skip" onClick={onDismiss}>
              Skip
            </button>
          )}
          {onNext && (
            <button className="coach-mark__next" onClick={onNext}>
              Got it!
            </button>
          )}
        </div>
      </div>
    </motion.div>
  );
}
```

---

## Contextual Help System

### Just-in-Time Help Engine

```typescript
interface HelpEntry {
  id: string;
  title: string;
  summary: string;
  content: string; // Markdown
  category: 'mechanics' | 'ui' | 'strategy' | 'social' | 'settings';
  relatedEntries?: string[];
  contextTriggers?: ContextTrigger[];
  videoUrl?: string;
}

type ContextTrigger =
  | { type: 'element_hover'; selector: string; delayMs: number }
  | { type: 'error_state'; errorCode: string }
  | { type: 'repeated_failure'; action: string; threshold: number }
  | { type: 'idle'; screen: string; durationMs: number };

class ContextualHelpEngine {
  private entries: Map<string, HelpEntry> = new Map();
  private dismissedHints: Set<string> = new Set();
  private failureCounters: Map<string, number> = new Map();
  
  constructor(entries: HelpEntry[]) {
    for (const entry of entries) {
      this.entries.set(entry.id, entry);
    }
    this.loadDismissals();
  }
  
  registerFailure(action: string): HelpEntry | null {
    const count = (this.failureCounters.get(action) || 0) + 1;
    this.failureCounters.set(action, count);
    
    for (const entry of this.entries.values()) {
      if (this.dismissedHints.has(entry.id)) continue;
      
      for (const trigger of entry.contextTriggers || []) {
        if (
          trigger.type === 'repeated_failure' &&
          trigger.action === action &&
          count >= trigger.threshold
        ) {
          return entry;
        }
      }
    }
    
    return null;
  }
  
  getHelpForError(errorCode: string): HelpEntry | null {
    for (const entry of this.entries.values()) {
      for (const trigger of entry.contextTriggers || []) {
        if (trigger.type === 'error_state' && trigger.errorCode === errorCode) {
          return entry;
        }
      }
    }
    return null;
  }
  
  searchHelp(query: string): HelpEntry[] {
    const queryLower = query.toLowerCase();
    const results: { entry: HelpEntry; score: number }[] = [];
    
    for (const entry of this.entries.values()) {
      let score = 0;
      
      if (entry.title.toLowerCase().includes(queryLower)) score += 10;
      if (entry.summary.toLowerCase().includes(queryLower)) score += 5;
      if (entry.content.toLowerCase().includes(queryLower)) score += 1;
      
      if (score > 0) {
        results.push({ entry, score });
      }
    }
    
    return results
      .sort((a, b) => b.score - a.score)
      .map(r => r.entry);
  }
  
  getByCategory(category: HelpEntry['category']): HelpEntry[] {
    return Array.from(this.entries.values()).filter(e => e.category === category);
  }
  
  getRelated(entryId: string): HelpEntry[] {
    const entry = this.entries.get(entryId);
    if (!entry?.relatedEntries) return [];
    
    return entry.relatedEntries
      .map(id => this.entries.get(id))
      .filter((e): e is HelpEntry => !!e);
  }
  
  dismissHint(entryId: string): void {
    this.dismissedHints.add(entryId);
    this.saveDismissals();
  }
  
  private loadDismissals(): void {
    try {
      const saved = localStorage.getItem('help_dismissed');
      if (saved) this.dismissedHints = new Set(JSON.parse(saved));
    } catch {}
  }
  
  private saveDismissals(): void {
    try {
      localStorage.setItem('help_dismissed', JSON.stringify([...this.dismissedHints]));
    } catch {}
  }
}

// Example: Help entries for Farming in Purria
const GAME_HELP: HelpEntry[] = [
  {
    id: 'betting_basics',
    title: 'How Betting Works',
    summary: 'Place bets on which pots will fill their thresholds',
    content: `
# Betting Basics

Each day, the four **Meta-Pots** fill based on events on your farm.

## Bet Types
- **Fold**: Skip this pot (safe, no reward)
- **Call**: Small bet, small reward
- **Raise**: Medium risk, medium reward
- **All-In**: High risk, high reward!

## Tips
- Watch pot trends over multiple days
- All-In on pots that consistently hit 80%+
- Fold on pots with unpredictable patterns
    `,
    category: 'mechanics',
    relatedEntries: ['pot_thresholds', 'trigger_combos'],
    contextTriggers: [
      { type: 'element_hover', selector: '.bet-panel', delayMs: 3000 },
      { type: 'repeated_failure', action: 'bet_lost', threshold: 3 },
    ],
  },
  {
    id: 'pot_thresholds',
    title: 'Pot Thresholds',
    summary: 'Understanding the 50%, 80%, and 100% threshold system',
    content: `
# Pot Thresholds

Each pot has three reward tiers:

| Threshold | Difficulty | Payout Bonus |
|-----------|-----------|--------------|
| 50%       | Common    | 1.5x         |
| 80%       | Moderate  | 2.5x         |
| 100%      | Rare      | 5x           |

Your payout multiplier increases with higher thresholds!
    `,
    category: 'mechanics',
    relatedEntries: ['betting_basics'],
  },
];
```

---

## Guided Tour System

### Multi-Step Interactive Tour

```typescript
interface TourConfig {
  id: string;
  name: string;
  steps: TourStep[];
  completionReward?: { coins?: number; xp?: number };
  autoStartCondition?: (state: GameState) => boolean;
}

interface TourStep {
  target: string; // CSS selector
  title: string;
  content: string;
  placement: 'top' | 'bottom' | 'left' | 'right' | 'auto';
  allowInteraction?: boolean;
  waitForAction?: string;
  beforeShow?: () => void | Promise<void>;
  afterShow?: () => void | Promise<void>;
}

class GuidedTourManager {
  private tours: Map<string, TourConfig> = new Map();
  private completedTours: Set<string> = new Set();
  private activeTour: { config: TourConfig; stepIndex: number } | null = null;
  
  private renderCallback: ((step: TourStep | null, progress: { current: number; total: number }) => void) | null = null;
  
  register(tour: TourConfig): void {
    this.tours.set(tour.id, tour);
  }
  
  onRender(callback: typeof this.renderCallback): void {
    this.renderCallback = callback;
  }
  
  async startTour(tourId: string): Promise<void> {
    const config = this.tours.get(tourId);
    if (!config || this.completedTours.has(tourId)) return;
    
    this.activeTour = { config, stepIndex: 0 };
    await this.showCurrentStep();
  }
  
  async nextStep(): Promise<void> {
    if (!this.activeTour) return;
    
    const step = this.activeTour.config.steps[this.activeTour.stepIndex];
    await step.afterShow?.();
    
    this.activeTour.stepIndex++;
    
    if (this.activeTour.stepIndex >= this.activeTour.config.steps.length) {
      this.completeTour();
      return;
    }
    
    await this.showCurrentStep();
  }
  
  previousStep(): void {
    if (!this.activeTour || this.activeTour.stepIndex <= 0) return;
    this.activeTour.stepIndex--;
    this.showCurrentStep();
  }
  
  endTour(): void {
    this.activeTour = null;
    this.renderCallback?.(null, { current: 0, total: 0 });
  }
  
  private async showCurrentStep(): Promise<void> {
    if (!this.activeTour) return;
    
    const step = this.activeTour.config.steps[this.activeTour.stepIndex];
    await step.beforeShow?.();
    
    // Scroll target into view
    const targetElement = document.querySelector(step.target);
    if (targetElement) {
      targetElement.scrollIntoView({ behavior: 'smooth', block: 'center' });
    }
    
    this.renderCallback?.(step, {
      current: this.activeTour.stepIndex + 1,
      total: this.activeTour.config.steps.length,
    });
  }
  
  private completeTour(): void {
    if (!this.activeTour) return;
    
    this.completedTours.add(this.activeTour.config.id);
    this.saveTourProgress();
    
    this.activeTour = null;
    this.renderCallback?.(null, { current: 0, total: 0 });
  }
  
  checkAutoStart(state: GameState): void {
    for (const [id, tour] of this.tours) {
      if (this.completedTours.has(id)) continue;
      if (tour.autoStartCondition?.(state)) {
        this.startTour(id);
        break;
      }
    }
  }
  
  private saveTourProgress(): void {
    try {
      localStorage.setItem('tours_completed', JSON.stringify([...this.completedTours]));
    } catch {}
  }
}

// Example tour
const FARM_TOUR: TourConfig = {
  id: 'farm_basics',
  name: 'Farm Basics',
  steps: [
    {
      target: '.hex-grid',
      title: 'Your Farm',
      content: 'This is your tulip farm! Each hexagon is a plot where you can plant.',
      placement: 'bottom',
    },
    {
      target: '.pot-display',
      title: 'The Meta-Pots',
      content: 'These four pots track farm conditions: Water, Sun, Pest Control, and Growth.',
      placement: 'top',
    },
    {
      target: '.bet-panel',
      title: 'Place Your Bets',
      content: 'Before each day, bet on which pots will hit their thresholds. Higher risk = higher reward!',
      placement: 'left',
      allowInteraction: true,
      waitForAction: 'bet_placed',
    },
    {
      target: '.day-button',
      title: 'Start the Day',
      content: 'Once your bets are placed, tap here to begin the day and watch events unfold!',
      placement: 'top',
    },
    {
      target: '.simulin-panel',
      title: 'Your Simulin Companion',
      content: 'Your Simulin helper gives bonuses and grows with you. Take care of them!',
      placement: 'right',
    },
  ],
  completionReward: { coins: 200, xp: 100 },
  autoStartCondition: (state) => state.dayNumber === 1 && !state.tutorialCompleted,
};
```

---

## Adaptive Difficulty Hints

### Performance-Based Help System

```typescript
interface PlayerPerformanceMetrics {
  winsLast10: number;
  averageBetROI: number;
  tutorialsCompleted: number;
  sessionsPlayed: number;
  averageSessionMinutes: number;
  lastFiveOutcomes: ('win' | 'loss')[];
}

class AdaptiveHintSystem {
  private readonly STRUGGLE_THRESHOLD = 0.3;
  private readonly MASTERY_THRESHOLD = 0.8;
  
  constructor(
    private helpEngine: ContextualHelpEngine,
    private tutorialEngine: TutorialFlowEngine
  ) {}
  
  evaluateAndSuggest(metrics: PlayerPerformanceMetrics): HintSuggestion | null {
    const winRate = metrics.winsLast10 / 10;
    
    // Losing streak detection
    const recentLosses = metrics.lastFiveOutcomes.filter(o => o === 'loss').length;
    if (recentLosses >= 4) {
      return {
        type: 'encouragement',
        message: "Tough streak! Here's a tip to turn things around.",
        helpEntryId: 'betting_basics',
        priority: 'high',
      };
    }
    
    // Struggling player
    if (winRate < this.STRUGGLE_THRESHOLD && metrics.sessionsPlayed > 3) {
      return {
        type: 'tutorial_replay',
        message: 'Want to revisit the betting tutorial?',
        tutorialId: 'betting_basics_tour',
        priority: 'medium',
      };
    }
    
    // New player not engaging with features
    if (metrics.sessionsPlayed < 5 && metrics.averageSessionMinutes < 3) {
      return {
        type: 'feature_discovery',
        message: 'Did you know you can check pot trends?',
        helpEntryId: 'pot_trends',
        priority: 'low',
      };
    }
    
    // Mastery player: offer advanced tips
    if (winRate > this.MASTERY_THRESHOLD && metrics.sessionsPlayed > 10) {
      return {
        type: 'advanced_tip',
        message: 'Ready for advanced strategies?',
        helpEntryId: 'advanced_combos',
        priority: 'low',
      };
    }
    
    return null;
  }
  
  shouldShowHint(lastHintShown: Date | null, cooldownMinutes: number = 30): boolean {
    if (!lastHintShown) return true;
    const elapsed = Date.now() - lastHintShown.getTime();
    return elapsed > cooldownMinutes * 60 * 1000;
  }
}

interface HintSuggestion {
  type: 'encouragement' | 'tutorial_replay' | 'feature_discovery' | 'advanced_tip';
  message: string;
  helpEntryId?: string;
  tutorialId?: string;
  priority: 'low' | 'medium' | 'high';
}
```

---

## First-Time User Experience (FTUE) Checklist

```typescript
interface FTUEChecklist {
  items: FTUEItem[];
}

interface FTUEItem {
  id: string;
  label: string;
  description: string;
  condition: (state: GameState) => boolean;
  reward?: { coins?: number; xp?: number };
  order: number;
}

class FTUETracker {
  private completed: Set<string> = new Set();
  
  constructor(private checklist: FTUEChecklist) {
    this.loadProgress();
  }
  
  check(state: GameState): { newlyCompleted: FTUEItem[]; allDone: boolean } {
    const newlyCompleted: FTUEItem[] = [];
    
    for (const item of this.checklist.items) {
      if (this.completed.has(item.id)) continue;
      
      if (item.condition(state)) {
        this.completed.add(item.id);
        newlyCompleted.push(item);
      }
    }
    
    if (newlyCompleted.length > 0) {
      this.saveProgress();
    }
    
    return {
      newlyCompleted,
      allDone: this.completed.size >= this.checklist.items.length,
    };
  }
  
  getItems(): (FTUEItem & { completed: boolean })[] {
    return this.checklist.items
      .sort((a, b) => a.order - b.order)
      .map(item => ({
        ...item,
        completed: this.completed.has(item.id),
      }));
  }
  
  getCompletionPercentage(): number {
    return (this.completed.size / this.checklist.items.length) * 100;
  }
  
  private loadProgress(): void {
    try {
      const saved = localStorage.getItem('ftue_progress');
      if (saved) this.completed = new Set(JSON.parse(saved));
    } catch {}
  }
  
  private saveProgress(): void {
    try {
      localStorage.setItem('ftue_progress', JSON.stringify([...this.completed]));
    } catch {}
  }
}

// Example FTUE checklist
const PURRIA_FTUE: FTUEChecklist = {
  items: [
    {
      id: 'plant_first_tulip',
      label: 'Plant Your First Tulip',
      description: 'Tap a hex to plant a tulip seed',
      condition: (state) => state.totalTulipsPlanted > 0,
      reward: { coins: 50 },
      order: 1,
    },
    {
      id: 'place_first_bet',
      label: 'Place Your First Bet',
      description: 'Choose a bet type on any pot',
      condition: (state) => state.totalBetsPlaced > 0,
      reward: { coins: 100 },
      order: 2,
    },
    {
      id: 'complete_first_day',
      label: 'Complete Your First Day',
      description: 'Play through an entire game day',
      condition: (state) => state.dayNumber > 1,
      reward: { coins: 200, xp: 50 },
      order: 3,
    },
    {
      id: 'earn_first_trigger',
      label: 'Earn Your First Trigger',
      description: 'Achieve a special trigger during a day',
      condition: (state) => state.totalTriggersEarned > 0,
      reward: { coins: 150 },
      order: 4,
    },
    {
      id: 'bond_with_simulin',
      label: 'Bond With Your Simulin',
      description: 'Interact with your Simulin companion',
      condition: (state) => state.simulinBondLevel > 1,
      reward: { coins: 200, xp: 100 },
      order: 5,
    },
    {
      id: 'win_first_bet',
      label: 'Win Your First Bet',
      description: 'Successfully predict a pot threshold',
      condition: (state) => state.totalBetsWon > 0,
      reward: { coins: 300 },
      order: 6,
    },
  ],
};
```

---

*Tutorial & Onboarding Reference — Game Engineering Team*
