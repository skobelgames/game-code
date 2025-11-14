# AI Player Decision-Making Logic - RATIX Game

## Location: `/home/user/game-code/ratix` (HTML file, ~950KB)

---

## 1. WHERE AI PLAYERS MAKE MOVE DECISIONS

### Main AI Turn Entry Point
- **Function**: `aiTurn()` (Line 16948)
  - Called by `triggerAiTurn()` (Line 3248)
  - Sets `currentPlayer = 'B'` (AI player)
  - Initializes move state and timers
  - Calls `makeNextMove()` recursively to execute actions

### Move Decision Flow
1. **ensureAITurnIfNeeded()** (Line 3325)
   - Checks if player has valid moves remaining
   - If no valid actions available, triggers AI turn

2. **minimax()** (Line 16081)
   - Core decision-making algorithm
   - Parameters: board state, depth, isMaximizing flag, player, movesLeft, movedPieces, capturedPieces, scores, alpha-beta pruning values
   - Time-controlled search with deadline: `aiSearchDeadline = Date.now() + getAiTimeBudget()`

3. **makeNextMove()** (Line 17134)
   - Executes planned actions in sequence
   - Checks for special cases (King Evade, King Shot, Moral Boost)
   - Calls `minimax()` to evaluate available actions if no planned actions exist
   - Recursively calls itself after each move completes

---

## 2. HOW AI PLAYERS CHOOSE ACTIONS (Attacks, Special Moves, etc.)

### Core Decision Algorithm: Minimax with Alpha-Beta Pruning

**Phase 1: Action Generation** (Line 16102)
- Generates ALL possible actions for the AI player:
  - Regular moves (cost 1-2)
  - Shoots/ranged attacks
  - Special abilities (see section 3)
  - Turn actions (rotate facing)
  - Compound actions (turn + move, turn + shoot)
  
**Phase 2: Action Sorting** (Line 16598)
Actions are sorted by priority:
1. **Capture Value**: Higher targeted piece value = higher priority
2. **Action Cost**: Prefer spending more move points (encourages full utilization)
3. **Special Abilities**: Higher priority than regular moves
4. **Minimize Turns**: Avoid standalone turn moves unless necessary

**Phase 3: Action Evaluation** (Line 16625)
For each action:
1. Simulate the action on a board clone
2. Recursively evaluate opponent's best response (minimax depth - 1)
3. Score = `evaluateBoard()` result
4. Track best action with highest evaluation score

**Alpha-Beta Pruning**: Cuts off branches where score won't change best outcome
- Significantly reduces search space for deeper analysis

### Board Evaluation Function: evaluateBoard() (Line 15725)

Calculates a composite score based on:

```
Score = (PlayerPieceValue - OpponentPieceValue)
       + (PlayerThreats - OpponentThreats)
       + MOBILITY_WEIGHT * (PlayerMobility - OpponentMobility)
       + CENTER_WEIGHT * (PlayerCenterControl - OpponentCenterControl)
```

**Components**:
- **Piece Values**: Assigned by AI profile weights
- **Threats**: Pieces capable of capturing opponent pieces (KING = 50x multiplier!)
- **Mobility**: Number of available moves for each side
- **Center Control**: Proximity to board center (weighted negative distance)
- **Advancement Bonus**: 8x multiplier for pieces approaching opponent's back row (reinforcement opportunity)
- **Back Row Bonuses**: 5-11 points for summoning-capable pieces at back row

### Piece Value System

Base values (scaled by difficulty):
- King: 30 points (HIGHEST - losing = game loss)
- Champion: 13 points
- Dragon/Troll/Ogre: 8-9 points
- Necromancer/Wizard/Lich: 10-11 points
- Cavalry/Guard/Archer: 5-6 points
- Infantry/Mercenary: 3 points
- Spectre/Zombie: 7-9 points

**Difficulty Multipliers**:
- Easy: 1.0x baseline
- Medium: 1.2x (20% better piece evaluation)
- Hard: 1.3x (30% better piece evaluation)

---

## 3. EXISTING SPECIAL MOVE USAGE BY AI

### Offensive Special Moves

#### Dragon Inferno (Line 16156)
- **Cost**: 2 moves
- **Value Calculation**: 1.4x multiplier base, then 1.44x (Medium) or 1.82x (Hard)
- **Decision**: AI checks all 8 adjacent squares for capture opportunities
- **Usage**: Calculates combined value of direct target + adjacent enemy pieces

#### Wizard/Dragon Strafe (Line 16186)
- **Cost**: 2 moves (Limit: 2 uses each)
- **Value**: 2.5x base, multiplied by difficulty (1.38x Medium, 1.69x Hard)
- **Evaluation**: Positional move for tactical repositioning
- **Execution**: `wizardStrafeUsed[player]` tracks usage

#### Dark Void (Line 16381)
- **Cost**: DARK_VOID_COST moves
- **Value Base**: 5.5 points + occupant value
- **Multiplier for Difficulty**: 1.35x (Medium), 1.5x (Hard)
- **Logic**: `getDarkVoidTargets()` returns valid placement locations
- **Occupant Handling**:
  - Own pieces: -0.35x multiplier (negative value)
  - Enemy pieces: +1.5x multiplier (positive)

#### Barrage (Artillery) (Line 16406)
- **Cost**: BARRAGE_COST moves
- **Requirements**: 
  - Champion or King piece
  - Supreme variant only
  - 3 successful Moral Boosts (BARRAGE_REQUIRED_MORAL_BOOSTS)
  - At least one enemy on board
- **Value**: Min(enemyCandidates, BARRAGE_TARGET_COUNT) * 6.0
- **Difficulty**: 1.3x (Medium), 1.5x (Hard)

#### Inferno & Rage Moves
- **Dragon Inferno** & **Ogre Rage**: Special movement with AOE capture
  - Values all pieces in capture area
  - Multiplies value by 1.3x if multiple captures

### Summoning Special Moves

#### Wizard Spectre Summoning (Line 16218)
- **Cost**: 2 moves
- **Conditions**: Total captured >= 5, limit 2 spectres
- **Value**: 15.0 base (encourages usage)
- **Difficulty**: 1.56x (Medium), 1.95x (Hard)
- **Expected Success**: 50% chance (factored into evaluation)

#### Necromancer Summoning (Line 16239)
- **Zombie Summoning**: Value 12.0
- **Lich Summoning**: Value 22.0
- **Spectre Summoning**: Value 16.0
- **Undead Summoning**: Value 14.0
- **All variants**: Difficulty scaled (1.3x Hard, 1.15x Medium)

#### Champion Summoning (Line 16319)
- **Necromancer**: Value 18.0 (limit 2, for Elite/Supreme)
- **Huntsman**: Value 12.0 (Supreme only, with active limit)
- **Usage**: Champion at back row can summon

#### King Summoning (Line 16350)
- **Pistolier**: Value 10.0 (diagonal shooting specialist)
- **Fusilier**: Value 10.0 (forward shooting specialist)
- **Cost**: 2 moves each
- **Variants**: Expert, Elite, Supreme

### Defensive/Utility Special Moves

#### King Evade (Line 17155)
- **Trigger**: `getBestKingEvadePlan('B', movedPieces)`
- **Effect**: Move King to safe position
- **Cost**: -2 moves next round
- **Priority**: Checked FIRST before other moves if King is threatened

#### King Shot (Line 17182)
- **Trigger**: `getBestKingShotPlan('B', movedPieces, movesLeft)`
- **Effect**: King executes ranged attack
- **Cost**: 1 move
- **Value Threshold**: KING_SHOT_AI_THRESHOLD = 5
- **Priority**: Second priority after King Evade

#### Moral Boost (Line 17213)
- **Triggering Function**: `getAiMoralBoostTarget()`
- **Cost**: 1 move
- **Effect**: +2 moves next round on success (50% success rate)
- **Maximum Uses**: 3 (unlocks Barrage)
- **Coin Flip**: AI uses coin toss for success determination

#### Elephantry Charge (Line 16438)
- **Cost**: 2 moves
- **Conditions**: Not yet used this game, specific facing/direction
- **Value**: Sum of captured pieces + 1.3x if multiple captures
- **Path Evaluation**: Can charge up to 3 squares, stopping at heavy units

#### Ogre Rage (Line 16491)
- **Cost**: 2 moves
- **Conditions**: Not used this turn
- **Value**: Sum of all pieces in capture area pattern
- **Movement**: 1 square in cardinal direction with AOE capture

### Specialized Piece Actions

#### Elephantry Triple Shot (Line 17046)
- **Execution**: `executeAiElephantryTripleShot()`
- **Mechanic**: 3 shots with coin flip for success each
- **AI Targeting**: `selectAiElephantryMoveShootTarget()` picks highest-value targets
- **Value Evaluation**: Uses `aiPieceValues` for each potential target

#### Elephantry Move+Shoot (Line 17025)
- **Function**: `handleAiElephantryMoveShoot()`
- **Flow**: Move piece, then auto-select best target to shoot
- **Target Selection**: `selectAiElephantryMoveShootTarget()`

#### Archer Move + Diagonal Shoot (Line 16569)
- **Cost**: 2 moves
- **Condition**: Limit 2 uses, archer can move then shoot diagonally
- **Evaluation**: Targets all diagonal enemies from new position

#### Fusilier Strafe (Line 16195)
- **Value**: 2.5x base, difficulty scaled (1.38x Medium, 1.69x Hard)
- **Cost**: 2 moves
- **Limit**: 2 uses per player

---

## 4. STRUCTURE OF AI DECISION-MAKING CODE

### Constants & Configuration (Lines 4545-4617)

```javascript
// Core difficulty settings
let aiDifficulty = 'Medium';  // Easy, Medium, Hard
let aiMaxDepth = 53;  // Effective search depth after variant adjustments

// Time budgets (5400ms base for Medium, +2400ms bonus)
const AI_TIME_LIMIT_DEFAULT = 5400ms;
const AI_TIME_LIMIT_LARGE = 1800ms;

// Evaluation penalties
const AI_LEFTOVER_PENALTY = 0.75;  // Penalize unspent moves
const AI_TIE_EPSILON = 0.0001;  // Tie-breaking threshold
```

### AI Profile System (Lines 4561-4617)

5 distinct AI personalities with weighted attributes:

**Berserker** (Ragnar)
- threatWeight: 1.8 (highly aggressive)
- captureBonus: 1.4
- mobilityWeight: 0.25 (focuses on attack)
- specialAbilityBonus: 1.6

**Guardian** (Thorne)
- threatWeight: 0.6 (defensive)
- mobilityWeight: 0.7
- centerWeight: 0.6 (values stability)
- captureBonus: 0.8 (cautious)

**Tactician** (Astra) - BALANCED
- All weights at ~1.0x (neutral, adapts to board)
- specialAbilityBonus: 1.4

**Sorcerer** (Zephyr)
- specialAbilityBonus: 2.1 (HIGHEST - favors special moves)
- threatWeight: 0.85
- captureBonus: 0.95

**Nomad** (Kira)
- mobilityWeight: 0.85 (HIGHEST - mobility focused)
- threatWeight: 0.75
- specialAbilityBonus: 1.3

**Profile Application** (Line 15741):
```javascript
const profile = (player === 'B' && currentAiProfile) ? currentAiProfile : null;
const MOBILITY_WEIGHT = profile ? (0.45 * profile.mobilityWeight) : 0.45;
const CENTER_WEIGHT = profile ? (0.25 * profile.centerWeight) : 0.25;
const THREAT_MULTIPLIER = profile ? profile.threatWeight : 1.0;
```

### Key State Variables

```javascript
// Board & pieces
let board = [];  // 2D array of piece objects
let movedPieces = new Set();  // Pieces already moved this turn
let movesLeft = 3;  // Action points remaining

// AI planning
let aiPending = false;  // Prevents duplicate AI scheduling
let aiTurnTimeoutForced = false;  // Forces turn end
let aiSearchDeadline = 0;  // Time limit for minimax search
let plannedActions = [];  // Pre-computed action sequence

// Captured pieces & scoring
let capturedPieces = {'W': [], 'B': []};
let scores = {'W': 0, 'B': 0};

// Special move tracking
let dragonInfernoUsed = {'W': false, 'B': false};
let darkVoidUses = {'W': 0, 'B': 0};
let moralBoostSuccesses = {'W': 0, 'B': 0};
let necromancersSummoned = {'W': 0, 'B': 0};
// ... (many more tracking variables for each ability)
```

### Execution Flow Diagram

```
triggerAiTurn()
    ↓
  aiTurn()
    ├─ Apply start-of-turn modifiers
    ├─ Initialize plannedActions = []
    └─ makeNextMove() [LOOP]
        ├─ Check for forced timeout
        ├─ Check movesLeft <= 0 (end turn if true)
        ├─ Priority checks:
        │  ├─ King Evade plan? → Execute
        │  ├─ King Shot plan? → Execute
        │  └─ Moral Boost target? → Execute
        ├─ If no planned actions:
        │  └─ Call minimax() → Get best action
        │      ├─ Generate all possible actions
        │      ├─ Sort by: capture value, cost, special, type
        │      ├─ Evaluate top N actions (cap: 80-220)
        │      └─ Return highest-scoring action
        ├─ Execute chosen action
        ├─ Update board, scores, graphics
        └─ setTimeout(makeNextMove, 3-7 seconds)
            [Loop continues until movesLeft <= 0]

    ↓
completeAiTurn()
    ├─ Apply end-of-turn effects
    ├─ Reset AI state
    └─ Return control to player
```

### Time Management

**AI Time Budget Calculation** (Line 4877):
```javascript
function getAiTimeBudget() {
    let base = isLargeBoardVariant() ? 1800ms : 5400ms;
    if (aiDifficulty === 'Hard') {
        base += isLargeBoardVariant() ? 1200ms : 3600ms;  // +3600ms = ~9s total
    } else if (aiDifficulty === 'Medium') {
        base += isLargeBoardVariant() ? 800ms : 2400ms;   // +2400ms = ~7.8s total
    } else if (aiDifficulty === 'Easy') {
        base += isLargeBoardVariant() ? 400ms : 1200ms;   // +1200ms = ~6.6s total
    }
    return base;
}
```

**Time Expiration Check** (Line 16087):
```javascript
if (aiTimeExpired()) {
    return { score: leafScore(), sequence: [], sequenceCost: 0 };
}
```
- Called frequently during action generation and evaluation
- Returns immediately if time budget exceeded
- Allows graceful early termination of search

### Action Execution Pipeline

**Simple AI** (fallback, Line 15946):
- `findSimpleAIAction()` - Linear scan of pieces, greedy evaluation
- Used if minimax times out or for quick moves
- Much faster but less strategic

**Complex AI** (primary, Line 16081):
- `minimax()` with full evaluation
- Considers 2-3+ moves ahead
- Alpha-beta pruning for efficiency
- Adjusts depth based on board size and difficulty

---

## Summary of AI Capabilities

### Strategic Level
✓ Plans 2-3+ moves ahead (depth 25-91 depending on difficulty/board)
✓ Understands piece values and board control
✓ Evaluates threats to own King
✓ Pursues reinforcement opportunities
✓ Adapts playstyle based on AI profile (aggressive, defensive, special-move focused)

### Tactical Level
✓ Prioritizes captures over repositioning
✓ Values threatening opponent's King at 50x normal weight
✓ Chains special abilities (move → shoot, charge with capture, etc.)
✓ Manages limited resources (special move uses, cooldowns)
✓ Balances offense vs. defense (King Evade checks before attacking)

### Limitations
✗ Cannot see beyond current turn's move budget
✗ Grenade/mine mechanics evaluated as area captures only (no chain reactions)
✗ Multi-turn strategic plans limited by time budget
✗ No opening/endgame-specific strategies hardcoded

