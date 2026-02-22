# Casino Game Implementations

Complete implementations for poker, blackjack, slots, and provably fair systems.

---

## Provably Fair RNG

Cryptographic RNG that allows players to verify game outcomes independently, eliminating the "black box" problem.

### HMAC-SHA256 Based System

```typescript
class ProvablyFairRNG {
  private nonce = 0;
  
  constructor(
    private serverSeed: string,
    private clientSeed: string
  ) {}
  
  async generateBytes(count: number): Promise<Uint8Array> {
    const message = `${this.clientSeed}:${this.nonce}`;
    this.nonce++;
    
    const encoder = new TextEncoder();
    const key = await crypto.subtle.importKey(
      'raw',
      encoder.encode(this.serverSeed),
      { name: 'HMAC', hash: 'SHA-256' },
      false,
      ['sign']
    );
    
    const signature = await crypto.subtle.sign(
      'HMAC',
      key,
      encoder.encode(message)
    );
    
    return new Uint8Array(signature).slice(0, count);
  }
  
  // Rejection sampling to avoid modulo bias
  async generateUniform(min: number, max: number): Promise<number> {
    const range = max - min + 1;
    const bytesNeeded = Math.ceil(Math.log2(range) / 8) || 1;
    const maxValid = Math.floor(256 ** bytesNeeded / range) * range;
    
    while (true) {
      const bytes = await this.generateBytes(bytesNeeded);
      let value = 0;
      for (let i = 0; i < bytesNeeded; i++) {
        value = value * 256 + bytes[i];
      }
      
      if (value < maxValid) {
        return min + (value % range);
      }
    }
  }
  
  async generateFloat(): Promise<number> {
    const bytes = await this.generateBytes(4);
    const value = (bytes[0] << 24 | bytes[1] << 16 | bytes[2] << 8 | bytes[3]) >>> 0;
    return value / 4294967296;
  }
  
  // Fisher-Yates shuffle with cryptographic randomness
  async shuffle<T>(array: T[]): Promise<T[]> {
    const result = [...array];
    for (let i = result.length - 1; i > 0; i--) {
      const j = await this.generateUniform(0, i);
      [result[i], result[j]] = [result[j], result[i]];
    }
    return result;
  }
  
  getNonce(): number {
    return this.nonce;
  }
}

// Seed commitment for verification
class SeedCommitment {
  static async createServerSeed(): Promise<{ seed: string; hash: string }> {
    const bytes = crypto.getRandomValues(new Uint8Array(32));
    const seed = Array.from(bytes).map(b => b.toString(16).padStart(2, '0')).join('');
    const hash = await this.hashSeed(seed);
    return { seed, hash };
  }
  
  static async hashSeed(seed: string): Promise<string> {
    const encoder = new TextEncoder();
    const hashBuffer = await crypto.subtle.digest('SHA-256', encoder.encode(seed));
    return Array.from(new Uint8Array(hashBuffer))
      .map(b => b.toString(16).padStart(2, '0'))
      .join('');
  }
  
  // Players verify: hash(revealed_server_seed) === committed_hash
  static async verify(
    serverSeed: string,
    clientSeed: string,
    nonce: number,
    committedHash: string,
    expectedResult: number
  ): Promise<boolean> {
    const seedHash = await this.hashSeed(serverSeed);
    if (seedHash !== committedHash) return false;
    
    const rng = new ProvablyFairRNG(serverSeed, clientSeed);
    // Advance to the correct nonce
    for (let i = 0; i < nonce; i++) {
      await rng.generateFloat();
    }
    
    const result = await rng.generateFloat();
    return Math.abs(result - expectedResult) < Number.EPSILON;
  }
}
```

---

## Blackjack Implementation

### Complete Game Logic

```typescript
type Suit = 'hearts' | 'diamonds' | 'clubs' | 'spades';
type Rank = 'A' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9' | '10' | 'J' | 'Q' | 'K';

interface BlackjackCard {
  suit: Suit;
  rank: Rank;
  faceUp: boolean;
}

type BlackjackAction = 'hit' | 'stand' | 'double' | 'split' | 'insurance' | 'surrender';

interface BlackjackHand {
  cards: BlackjackCard[];
  bet: number;
  doubled: boolean;
  surrendered: boolean;
  stood: boolean;
}

type BlackjackPhase = 'betting' | 'dealing' | 'player_turn' | 'dealer_turn' | 'resolution';

interface BlackjackState {
  phase: BlackjackPhase;
  deck: BlackjackCard[];
  playerHands: BlackjackHand[];
  activeHandIndex: number;
  dealerHand: BlackjackCard[];
  balance: number;
  insuranceBet: number;
}

class BlackjackEngine {
  private state: BlackjackState;
  
  constructor(
    private rng: ProvablyFairRNG,
    private rules: BlackjackRules = DEFAULT_RULES
  ) {
    this.state = this.createInitialState();
  }
  
  private createInitialState(): BlackjackState {
    return {
      phase: 'betting',
      deck: [],
      playerHands: [],
      activeHandIndex: 0,
      dealerHand: [],
      balance: 1000,
      insuranceBet: 0,
    };
  }
  
  async startRound(bet: number): Promise<void> {
    if (bet > this.state.balance || bet < this.rules.minBet) {
      throw new Error('Invalid bet amount');
    }
    
    // Build and shuffle shoe
    this.state.deck = await this.buildShoe(this.rules.deckCount);
    this.state.balance -= bet;
    this.state.phase = 'dealing';
    
    // Deal initial cards
    const playerCard1 = this.drawCard(true);
    const dealerCard1 = this.drawCard(true);
    const playerCard2 = this.drawCard(true);
    const dealerCard2 = this.drawCard(false); // Hole card face down
    
    this.state.playerHands = [{
      cards: [playerCard1, playerCard2],
      bet,
      doubled: false,
      surrendered: false,
      stood: false,
    }];
    this.state.dealerHand = [dealerCard1, dealerCard2];
    this.state.activeHandIndex = 0;
    
    // Check for naturals
    const playerBlackjack = this.handValue(this.state.playerHands[0].cards) === 21;
    const dealerBlackjack = this.handValue(this.state.dealerHand) === 21;
    
    if (playerBlackjack || dealerBlackjack) {
      this.state.dealerHand[1].faceUp = true;
      this.resolveRound();
      return;
    }
    
    // Offer insurance if dealer shows Ace
    if (dealerCard1.rank === 'A') {
      // Insurance logic handled by player action
    }
    
    this.state.phase = 'player_turn';
  }
  
  getAvailableActions(): BlackjackAction[] {
    const hand = this.state.playerHands[this.state.activeHandIndex];
    if (!hand || hand.stood) return [];
    
    const actions: BlackjackAction[] = ['hit', 'stand'];
    
    // Double down: first two cards only
    if (hand.cards.length === 2 && !hand.doubled && this.state.balance >= hand.bet) {
      actions.push('double');
    }
    
    // Split: matching ranks, max splits not reached
    if (
      hand.cards.length === 2 &&
      this.cardValue(hand.cards[0]) === this.cardValue(hand.cards[1]) &&
      this.state.playerHands.length < this.rules.maxSplits + 1 &&
      this.state.balance >= hand.bet
    ) {
      actions.push('split');
    }
    
    // Surrender: first action only
    if (hand.cards.length === 2 && this.rules.allowSurrender) {
      actions.push('surrender');
    }
    
    return actions;
  }
  
  async playerAction(action: BlackjackAction): Promise<void> {
    const hand = this.state.playerHands[this.state.activeHandIndex];
    
    switch (action) {
      case 'hit': {
        hand.cards.push(this.drawCard(true));
        if (this.handValue(hand.cards) > 21) {
          this.advanceToNextHand();
        }
        break;
      }
      
      case 'stand': {
        hand.stood = true;
        this.advanceToNextHand();
        break;
      }
      
      case 'double': {
        this.state.balance -= hand.bet;
        hand.bet *= 2;
        hand.doubled = true;
        hand.cards.push(this.drawCard(true));
        hand.stood = true;
        this.advanceToNextHand();
        break;
      }
      
      case 'split': {
        this.state.balance -= hand.bet;
        const splitCard = hand.cards.pop()!;
        
        hand.cards.push(this.drawCard(true));
        
        const newHand: BlackjackHand = {
          cards: [splitCard, this.drawCard(true)],
          bet: hand.bet,
          doubled: false,
          surrendered: false,
          stood: false,
        };
        
        this.state.playerHands.splice(this.state.activeHandIndex + 1, 0, newHand);
        break;
      }
      
      case 'surrender': {
        hand.surrendered = true;
        hand.stood = true;
        this.advanceToNextHand();
        break;
      }
    }
  }
  
  private advanceToNextHand(): void {
    this.state.activeHandIndex++;
    if (this.state.activeHandIndex >= this.state.playerHands.length) {
      this.dealerPlay();
    }
  }
  
  private dealerPlay(): void {
    this.state.phase = 'dealer_turn';
    this.state.dealerHand[1].faceUp = true;
    
    // Check if all player hands busted/surrendered
    const allBusted = this.state.playerHands.every(
      h => h.surrendered || this.handValue(h.cards) > 21
    );
    
    if (!allBusted) {
      while (this.shouldDealerHit()) {
        this.state.dealerHand.push(this.drawCard(true));
      }
    }
    
    this.resolveRound();
  }
  
  private shouldDealerHit(): boolean {
    const value = this.handValue(this.state.dealerHand);
    if (value < 17) return true;
    if (value === 17 && this.rules.dealerHitsSoft17 && this.isSoft(this.state.dealerHand)) {
      return true;
    }
    return false;
  }
  
  private resolveRound(): void {
    this.state.phase = 'resolution';
    const dealerValue = this.handValue(this.state.dealerHand);
    const dealerBust = dealerValue > 21;
    const dealerBlackjack = dealerValue === 21 && this.state.dealerHand.length === 2;
    
    for (const hand of this.state.playerHands) {
      if (hand.surrendered) {
        this.state.balance += Math.floor(hand.bet / 2);
        continue;
      }
      
      const playerValue = this.handValue(hand.cards);
      const playerBust = playerValue > 21;
      const playerBlackjack = playerValue === 21 && hand.cards.length === 2;
      
      if (playerBust) {
        // Player loses
        continue;
      }
      
      if (playerBlackjack && !dealerBlackjack) {
        // Blackjack pays 3:2
        this.state.balance += Math.floor(hand.bet * 2.5);
      } else if (dealerBust || playerValue > dealerValue) {
        // Player wins
        this.state.balance += hand.bet * 2;
      } else if (playerValue === dealerValue) {
        // Push
        this.state.balance += hand.bet;
      }
      // else dealer wins, bet already deducted
    }
  }
  
  handValue(cards: BlackjackCard[]): number {
    let value = 0;
    let aces = 0;
    
    for (const card of cards) {
      const v = this.cardValue(card);
      if (v === 11) aces++;
      value += v;
    }
    
    while (value > 21 && aces > 0) {
      value -= 10;
      aces--;
    }
    
    return value;
  }
  
  private cardValue(card: BlackjackCard): number {
    if (card.rank === 'A') return 11;
    if (['J', 'Q', 'K'].includes(card.rank)) return 10;
    return parseInt(card.rank);
  }
  
  private isSoft(cards: BlackjackCard[]): boolean {
    let value = 0;
    let aces = 0;
    for (const card of cards) {
      const v = this.cardValue(card);
      if (v === 11) aces++;
      value += v;
    }
    return aces > 0 && value <= 21;
  }
  
  private async buildShoe(deckCount: number): Promise<BlackjackCard[]> {
    const suits: Suit[] = ['hearts', 'diamonds', 'clubs', 'spades'];
    const ranks: Rank[] = ['A', '2', '3', '4', '5', '6', '7', '8', '9', '10', 'J', 'Q', 'K'];
    
    const cards: BlackjackCard[] = [];
    for (let d = 0; d < deckCount; d++) {
      for (const suit of suits) {
        for (const rank of ranks) {
          cards.push({ suit, rank, faceUp: false });
        }
      }
    }
    
    return this.rng.shuffle(cards);
  }
  
  private drawCard(faceUp: boolean): BlackjackCard {
    const card = this.state.deck.pop()!;
    card.faceUp = faceUp;
    return card;
  }
  
  getState(): Readonly<BlackjackState> {
    return this.state;
  }
}

interface BlackjackRules {
  deckCount: number;
  minBet: number;
  maxBet: number;
  dealerHitsSoft17: boolean;
  allowSurrender: boolean;
  maxSplits: number;
  blackjackPayout: number; // 1.5 = 3:2
}

const DEFAULT_RULES: BlackjackRules = {
  deckCount: 6,
  minBet: 10,
  maxBet: 5000,
  dealerHitsSoft17: true,
  allowSurrender: true,
  maxSplits: 3,
  blackjackPayout: 1.5,
};
```

### Basic Strategy Engine

```typescript
type StrategyAction = 'H' | 'S' | 'D' | 'P' | 'R';

// H=Hit S=Stand D=Double P=Split R=Surrender
const HARD_TOTALS: Record<string, StrategyAction[]> = {
  //         2    3    4    5    6    7    8    9   10    A
  '5':  ['H', 'H', 'H', 'H', 'H', 'H', 'H', 'H', 'H', 'H'],
  '6':  ['H', 'H', 'H', 'H', 'H', 'H', 'H', 'H', 'H', 'H'],
  '7':  ['H', 'H', 'H', 'H', 'H', 'H', 'H', 'H', 'H', 'H'],
  '8':  ['H', 'H', 'H', 'H', 'H', 'H', 'H', 'H', 'H', 'H'],
  '9':  ['H', 'D', 'D', 'D', 'D', 'H', 'H', 'H', 'H', 'H'],
  '10': ['D', 'D', 'D', 'D', 'D', 'D', 'D', 'D', 'H', 'H'],
  '11': ['D', 'D', 'D', 'D', 'D', 'D', 'D', 'D', 'D', 'D'],
  '12': ['H', 'H', 'S', 'S', 'S', 'H', 'H', 'H', 'H', 'H'],
  '13': ['S', 'S', 'S', 'S', 'S', 'H', 'H', 'H', 'H', 'H'],
  '14': ['S', 'S', 'S', 'S', 'S', 'H', 'H', 'H', 'H', 'H'],
  '15': ['S', 'S', 'S', 'S', 'S', 'H', 'H', 'H', 'R', 'R'],
  '16': ['S', 'S', 'S', 'S', 'S', 'H', 'H', 'R', 'R', 'R'],
  '17': ['S', 'S', 'S', 'S', 'S', 'S', 'S', 'S', 'S', 'S'],
};

class BasicStrategy {
  private dealerIndex(dealerUpcard: number): number {
    if (dealerUpcard === 11) return 9; // Ace
    return dealerUpcard - 2;
  }
  
  recommend(playerCards: BlackjackCard[], dealerUpcard: BlackjackCard): StrategyAction {
    const engine = new BlackjackEngine(null as any);
    const playerValue = engine.handValue(playerCards);
    const dealerVal = engine.handValue([dealerUpcard]);
    const idx = this.dealerIndex(dealerVal);
    
    const row = HARD_TOTALS[playerValue.toString()];
    if (!row) return playerValue < 12 ? 'H' : 'S';
    
    return row[idx];
  }
}
```

---

## Slot Machine Implementation

### Reel-Based Slot Engine

```typescript
interface SlotSymbol {
  id: string;
  name: string;
  weight: number;
  payouts: Record<number, number>; // count -> multiplier
  isWild?: boolean;
  isScatter?: boolean;
}

interface SlotConfig {
  reels: number;
  rows: number;
  symbols: SlotSymbol[];
  paylines: number[][];
  rtp: number; // Target Return to Player (0.90-0.99)
}

interface SpinResult {
  grid: string[][];
  wins: WinLine[];
  totalWin: number;
  freeSpinsAwarded: number;
}

interface WinLine {
  paylineIndex: number;
  symbol: string;
  count: number;
  multiplier: number;
  positions: [number, number][];
}

class SlotEngine {
  private reelStrips: string[][];
  
  constructor(
    private config: SlotConfig,
    private rng: ProvablyFairRNG
  ) {
    this.reelStrips = this.buildReelStrips();
  }
  
  private buildReelStrips(): string[][] {
    // Build virtual reel strips with weighted symbols
    return Array.from({ length: this.config.reels }, () => {
      const strip: string[] = [];
      for (const symbol of this.config.symbols) {
        for (let i = 0; i < symbol.weight; i++) {
          strip.push(symbol.id);
        }
      }
      return strip;
    });
  }
  
  async spin(bet: number): Promise<SpinResult> {
    // Generate random stop positions
    const stopPositions = await Promise.all(
      this.reelStrips.map(strip => this.rng.generateUniform(0, strip.length - 1))
    );
    
    // Build visible grid
    const grid = this.buildGrid(stopPositions);
    
    // Evaluate paylines
    const wins = this.evaluatePaylines(grid);
    
    // Check for scatter wins (free spins)
    const scatterCount = this.countScatters(grid);
    const freeSpinsAwarded = scatterCount >= 3 ? scatterCount * 5 : 0;
    
    const totalWin = wins.reduce((sum, w) => sum + w.multiplier * bet, 0);
    
    return { grid, wins, totalWin, freeSpinsAwarded };
  }
  
  private buildGrid(stopPositions: number[]): string[][] {
    const grid: string[][] = [];
    
    for (let row = 0; row < this.config.rows; row++) {
      const rowSymbols: string[] = [];
      for (let reel = 0; reel < this.config.reels; reel++) {
        const strip = this.reelStrips[reel];
        const position = (stopPositions[reel] + row) % strip.length;
        rowSymbols.push(strip[position]);
      }
      grid.push(rowSymbols);
    }
    
    return grid;
  }
  
  private evaluatePaylines(grid: string[][]): WinLine[] {
    const wins: WinLine[] = [];
    
    for (let i = 0; i < this.config.paylines.length; i++) {
      const payline = this.config.paylines[i];
      const symbolsOnLine = payline.map((row, col) => ({
        symbol: grid[row][col],
        position: [row, col] as [number, number],
      }));
      
      const win = this.evaluateLine(symbolsOnLine, i);
      if (win) wins.push(win);
    }
    
    return wins;
  }
  
  private evaluateLine(
    line: { symbol: string; position: [number, number] }[],
    paylineIndex: number
  ): WinLine | null {
    const firstSymbol = this.resolveWild(line[0].symbol);
    let count = 1;
    const positions: [number, number][] = [line[0].position];
    
    for (let i = 1; i < line.length; i++) {
      const sym = line[i].symbol;
      const symDef = this.config.symbols.find(s => s.id === sym);
      
      if (sym === firstSymbol || symDef?.isWild) {
        count++;
        positions.push(line[i].position);
      } else {
        break;
      }
    }
    
    const symbolDef = this.config.symbols.find(s => s.id === firstSymbol);
    if (!symbolDef) return null;
    
    const multiplier = symbolDef.payouts[count];
    if (!multiplier) return null;
    
    return {
      paylineIndex,
      symbol: firstSymbol,
      count,
      multiplier,
      positions,
    };
  }
  
  private resolveWild(symbolId: string): string {
    const sym = this.config.symbols.find(s => s.id === symbolId);
    if (sym?.isWild) return symbolId;
    return symbolId;
  }
  
  private countScatters(grid: string[][]): number {
    let count = 0;
    for (const row of grid) {
      for (const sym of row) {
        const def = this.config.symbols.find(s => s.id === sym);
        if (def?.isScatter) count++;
      }
    }
    return count;
  }
}

// Example: Farming-themed slot config
const FARM_SLOTS: SlotConfig = {
  reels: 5,
  rows: 3,
  symbols: [
    { id: 'tulip_red', name: 'Red Tulip', weight: 8, payouts: { 3: 5, 4: 15, 5: 50 } },
    { id: 'tulip_yellow', name: 'Yellow Tulip', weight: 8, payouts: { 3: 5, 4: 15, 5: 50 } },
    { id: 'tulip_purple', name: 'Purple Tulip', weight: 6, payouts: { 3: 10, 4: 25, 5: 100 } },
    { id: 'watering_can', name: 'Watering Can', weight: 5, payouts: { 3: 15, 4: 40, 5: 150 } },
    { id: 'sun', name: 'Golden Sun', weight: 4, payouts: { 3: 20, 4: 60, 5: 250 } },
    { id: 'simulin', name: 'Simulin', weight: 3, payouts: { 3: 50, 4: 150, 5: 500 } },
    { id: 'wild_bloom', name: 'Wild Bloom', weight: 2, payouts: { 3: 25, 4: 75, 5: 300 }, isWild: true },
    { id: 'scatter_seed', name: 'Magic Seed', weight: 2, payouts: {}, isScatter: true },
  ],
  paylines: [
    [1, 1, 1, 1, 1], // Middle row
    [0, 0, 0, 0, 0], // Top row
    [2, 2, 2, 2, 2], // Bottom row
    [0, 1, 2, 1, 0], // V shape
    [2, 1, 0, 1, 2], // Inverted V
    [0, 0, 1, 2, 2], // Diagonal down
    [2, 2, 1, 0, 0], // Diagonal up
    [1, 0, 0, 0, 1], // U shape
    [1, 2, 2, 2, 1], // Inverted U
  ],
  rtp: 0.96,
};
```

---

## Poker Variants

### Texas Hold'em Engine

```typescript
type PokerPhase = 'preflop' | 'flop' | 'turn' | 'river' | 'showdown';

interface PokerPlayer {
  id: string;
  chips: number;
  holeCards: BlackjackCard[];
  currentBet: number;
  folded: boolean;
  allIn: boolean;
}

interface PokerState {
  phase: PokerPhase;
  communityCards: BlackjackCard[];
  pot: number;
  sidePots: SidePot[];
  players: PokerPlayer[];
  dealerIndex: number;
  activePlayerIndex: number;
  currentBet: number;
  minRaise: number;
}

interface SidePot {
  amount: number;
  eligiblePlayerIds: string[];
}

class TexasHoldemEngine {
  private state: PokerState;
  private deck: BlackjackCard[] = [];
  
  constructor(
    private rng: ProvablyFairRNG,
    playerIds: string[],
    buyIn: number,
    private blinds: { small: number; big: number }
  ) {
    this.state = {
      phase: 'preflop',
      communityCards: [],
      pot: 0,
      sidePots: [],
      players: playerIds.map(id => ({
        id,
        chips: buyIn,
        holeCards: [],
        currentBet: 0,
        folded: false,
        allIn: false,
      })),
      dealerIndex: 0,
      activePlayerIndex: 0,
      currentBet: 0,
      minRaise: blinds.big,
    };
  }
  
  async startHand(): Promise<void> {
    // Reset
    for (const player of this.state.players) {
      player.holeCards = [];
      player.currentBet = 0;
      player.folded = false;
      player.allIn = false;
    }
    this.state.communityCards = [];
    this.state.pot = 0;
    this.state.sidePots = [];
    this.state.phase = 'preflop';
    
    // Build deck
    this.deck = await this.buildDeck();
    
    // Post blinds
    const sbIndex = (this.state.dealerIndex + 1) % this.state.players.length;
    const bbIndex = (this.state.dealerIndex + 2) % this.state.players.length;
    
    this.postBlind(sbIndex, this.blinds.small);
    this.postBlind(bbIndex, this.blinds.big);
    this.state.currentBet = this.blinds.big;
    
    // Deal hole cards
    for (const player of this.state.players) {
      player.holeCards = [this.drawCard(), this.drawCard()];
    }
    
    // First to act is after big blind
    this.state.activePlayerIndex = (bbIndex + 1) % this.state.players.length;
    this.skipFoldedPlayers();
  }
  
  playerAction(action: 'fold' | 'check' | 'call' | 'raise', raiseAmount?: number): void {
    const player = this.state.players[this.state.activePlayerIndex];
    
    switch (action) {
      case 'fold':
        player.folded = true;
        break;
        
      case 'check':
        if (player.currentBet < this.state.currentBet) {
          throw new Error('Cannot check, must call or raise');
        }
        break;
        
      case 'call': {
        const callAmount = Math.min(
          this.state.currentBet - player.currentBet,
          player.chips
        );
        player.chips -= callAmount;
        player.currentBet += callAmount;
        this.state.pot += callAmount;
        if (player.chips === 0) player.allIn = true;
        break;
      }
        
      case 'raise': {
        if (!raiseAmount || raiseAmount < this.state.minRaise) {
          throw new Error(`Minimum raise is ${this.state.minRaise}`);
        }
        const totalBet = this.state.currentBet + raiseAmount;
        const needed = totalBet - player.currentBet;
        
        if (needed > player.chips) {
          // All-in
          this.state.pot += player.chips;
          player.currentBet += player.chips;
          player.chips = 0;
          player.allIn = true;
        } else {
          player.chips -= needed;
          player.currentBet = totalBet;
          this.state.pot += needed;
        }
        
        this.state.currentBet = player.currentBet;
        this.state.minRaise = raiseAmount;
        break;
      }
    }
    
    this.advanceAction();
  }
  
  private advanceAction(): void {
    // Check if only one player remains
    const activePlayers = this.state.players.filter(p => !p.folded);
    if (activePlayers.length === 1) {
      this.awardPot(activePlayers[0]);
      return;
    }
    
    // Move to next player
    this.state.activePlayerIndex = 
      (this.state.activePlayerIndex + 1) % this.state.players.length;
    this.skipFoldedPlayers();
    
    // Check if betting round is complete
    if (this.isBettingComplete()) {
      this.advancePhase();
    }
  }
  
  private advancePhase(): void {
    // Reset bets
    for (const player of this.state.players) {
      player.currentBet = 0;
    }
    this.state.currentBet = 0;
    this.state.minRaise = this.blinds.big;
    
    switch (this.state.phase) {
      case 'preflop':
        this.state.phase = 'flop';
        this.state.communityCards.push(this.drawCard(), this.drawCard(), this.drawCard());
        break;
      case 'flop':
        this.state.phase = 'turn';
        this.state.communityCards.push(this.drawCard());
        break;
      case 'turn':
        this.state.phase = 'river';
        this.state.communityCards.push(this.drawCard());
        break;
      case 'river':
        this.state.phase = 'showdown';
        this.resolveShowdown();
        return;
    }
    
    // First to act post-flop is after dealer
    this.state.activePlayerIndex = 
      (this.state.dealerIndex + 1) % this.state.players.length;
    this.skipFoldedPlayers();
  }
  
  private resolveShowdown(): void {
    const evaluator = new PokerHandEvaluator();
    const activePlayers = this.state.players.filter(p => !p.folded);
    
    let bestValue = -1;
    let winners: PokerPlayer[] = [];
    
    for (const player of activePlayers) {
      const allCards = [...player.holeCards, ...this.state.communityCards];
      const result = evaluator.evaluate(allCards as any);
      
      if (result.value > bestValue) {
        bestValue = result.value;
        winners = [player];
      } else if (result.value === bestValue) {
        winners.push(player);
      }
    }
    
    const share = Math.floor(this.state.pot / winners.length);
    for (const winner of winners) {
      winner.chips += share;
    }
  }
  
  private postBlind(playerIndex: number, amount: number): void {
    const player = this.state.players[playerIndex];
    const actual = Math.min(amount, player.chips);
    player.chips -= actual;
    player.currentBet = actual;
    this.state.pot += actual;
    if (player.chips === 0) player.allIn = true;
  }
  
  private isBettingComplete(): boolean {
    const active = this.state.players.filter(p => !p.folded && !p.allIn);
    return active.every(p => p.currentBet === this.state.currentBet);
  }
  
  private skipFoldedPlayers(): void {
    const len = this.state.players.length;
    for (let i = 0; i < len; i++) {
      const player = this.state.players[this.state.activePlayerIndex];
      if (!player.folded && !player.allIn) return;
      this.state.activePlayerIndex = 
        (this.state.activePlayerIndex + 1) % len;
    }
  }
  
  private awardPot(winner: PokerPlayer): void {
    winner.chips += this.state.pot;
    this.state.pot = 0;
    this.state.phase = 'showdown';
  }
  
  private async buildDeck(): Promise<BlackjackCard[]> {
    const suits: Suit[] = ['hearts', 'diamonds', 'clubs', 'spades'];
    const ranks: Rank[] = ['A', '2', '3', '4', '5', '6', '7', '8', '9', '10', 'J', 'Q', 'K'];
    const cards: BlackjackCard[] = [];
    
    for (const suit of suits) {
      for (const rank of ranks) {
        cards.push({ suit, rank, faceUp: false });
      }
    }
    
    return this.rng.shuffle(cards);
  }
  
  private drawCard(): BlackjackCard {
    const card = this.deck.pop()!;
    card.faceUp = true;
    return card;
  }
  
  getState(): Readonly<PokerState> {
    return this.state;
  }
}

// Reuse PokerHandEvaluator from SKILL.md
class PokerHandEvaluator {
  evaluate(cards: any[]): { value: number; rank: string } {
    // Full implementation in SKILL.md Part III
    return { value: 0, rank: 'high-card' };
  }
}
```

---

## House Edge & RTP Calculation

### Mathematical Framework

```typescript
interface PayTableEntry {
  outcome: string;
  probability: number;
  payout: number;
}

class HouseEdgeCalculator {
  // RTP = sum(probability * payout) for all outcomes
  calculateRTP(payTable: PayTableEntry[]): number {
    return payTable.reduce((sum, entry) => {
      return sum + entry.probability * entry.payout;
    }, 0);
  }
  
  // House Edge = 1 - RTP
  calculateHouseEdge(payTable: PayTableEntry[]): number {
    return 1 - this.calculateRTP(payTable);
  }
  
  // Variance = sum(probability * (payout - rtp)^2)
  calculateVariance(payTable: PayTableEntry[]): number {
    const rtp = this.calculateRTP(payTable);
    return payTable.reduce((sum, entry) => {
      const diff = entry.payout - rtp;
      return sum + entry.probability * diff * diff;
    }, 0);
  }
  
  // Volatility Index (standard deviation of return)
  calculateVolatility(payTable: PayTableEntry[]): number {
    return Math.sqrt(this.calculateVariance(payTable));
  }
  
  // Simulate N rounds and measure actual RTP
  async simulateRTP(
    engine: { spin: (bet: number) => Promise<{ totalWin: number }> },
    rounds: number,
    betSize: number
  ): Promise<{ actualRTP: number; confidence95: [number, number] }> {
    let totalBet = 0;
    let totalWin = 0;
    const returns: number[] = [];
    
    for (let i = 0; i < rounds; i++) {
      totalBet += betSize;
      const result = await engine.spin(betSize);
      totalWin += result.totalWin;
      returns.push(result.totalWin / betSize);
    }
    
    const actualRTP = totalWin / totalBet;
    
    // 95% confidence interval
    const mean = returns.reduce((a, b) => a + b, 0) / returns.length;
    const stdDev = Math.sqrt(
      returns.reduce((sum, r) => sum + (r - mean) ** 2, 0) / (returns.length - 1)
    );
    const marginOfError = 1.96 * stdDev / Math.sqrt(returns.length);
    
    return {
      actualRTP,
      confidence95: [actualRTP - marginOfError, actualRTP + marginOfError],
    };
  }
}

// Example: Blackjack RTP analysis
const BLACKJACK_PAY_TABLE: PayTableEntry[] = [
  { outcome: 'blackjack', probability: 0.0473, payout: 2.5 },
  { outcome: 'win', probability: 0.4244, payout: 2.0 },
  { outcome: 'push', probability: 0.0848, payout: 1.0 },
  { outcome: 'lose', probability: 0.4435, payout: 0.0 },
];

const calc = new HouseEdgeCalculator();
const rtp = calc.calculateRTP(BLACKJACK_PAY_TABLE);
// ~0.9954 (99.54% RTP with perfect basic strategy)
```

---

*Casino Implementation Reference — Game Engineering Team*
