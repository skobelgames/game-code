# AI Decision-Making Code Quick Reference

## File Location
- **Main File**: `/home/user/game-code/ratix` (HTML/JavaScript, ~950KB)
- **Type**: Single-page application with embedded JavaScript

---

## Key Function Locations (Line Numbers)

### Top-Level AI Control
| Function | Line | Purpose |
|----------|------|---------|
| `triggerAiTurn()` | 3248 | Initiates AI's turn, prepares state |
| `aiTurn()` | 16948 | Main AI turn execution loop |
| `makeNextMove()` | 17134 | Executes individual moves in sequence |
| `completeAiTurn()` | 3279 | Cleans up after AI turn ends |

### Decision Making (Core Algorithm)
| Function | Line | Purpose |
|----------|------|---------|
| `minimax()` | 16081 | Main decision algorithm with alpha-beta pruning |
| `evaluateBoard()` | 15725 | Scores board position for minimax |
| `findSimpleAIAction()` | 15946 | Fallback greedy AI (faster, less strategic) |

### Special Move Evaluation
| Function | Line | Purpose |
|----------|------|---------|
| (Inferno in minimax) | 16156 | Dragon area-effect capture logic |
| (Strafe in minimax) | 16186 | Wizard/Dragon positioning moves |
| (Dark Void in minimax) | 16381 | Necromancer/Lich void placement |
| (Barrage in minimax) | 16406 | Artillery special move (Supreme only) |
| (Summoning in minimax) | 16218-16350 | All summoning ability evaluation |

### Special Move Execution (AI)
| Function | Line | Purpose |
|----------|------|---------|
| `executeAiElephantryTripleShot()` | 17046 | Triple shot execution with targeting |
| `handleAiElephantryMoveShoot()` | 17025 | Move then auto-shoot logic |
| `selectAiElephantryMoveShootTarget()` | (search) | Picks best target for Elephantry shoot |
| `getBestKingEvadePlan()` | (search) | Defensive King movement |
| `getBestKingShotPlan()` | (search) | King ranged attack evaluation |
| `getAiMoralBoostTarget()` | (search) | Moral Boost usage decision |

### Board & State Management
| Variable | Line | Type |
|----------|------|------|
| `board` | - | 2D array of piece objects |
| `aiDifficulty` | 4545 | String ('Easy', 'Medium', 'Hard') |
| `aiMaxDepth` | 4547 | Number (25-91 based on variant) |
| `movedPieces` | - | Set tracking pieces already moved |
| `movesLeft` | - | Number of action points remaining |
| `aiPending` | 2267 | Boolean flag preventing duplicate AI |
| `plannedActions` | - | Array of pre-computed actions to execute |

### AI Configuration
| Constant | Line | Value |
|----------|------|-------|
| `AI_PROFILES` | 4561 | Object defining 5 personality types |
| `AI_TIME_LIMIT_DEFAULT` | 4557 | 5400ms (5.4 seconds) |
| `AI_TIME_LIMIT_LARGE` | 4558 | 1800ms (1.8 seconds) |
| `AI_LEFTOVER_PENALTY` | 4555 | 0.75 (penalizes unspent moves) |

### Piece Value System
| Variable | Line | Purpose |
|----------|------|---------|
| `basePieceValues` | (search) | Base piece values for calculation |
| `aiPieceValues` | 4871 | Difficulty-adjusted piece values |
| `pieceValueScale` | (search) | Multiplier based on difficulty |

---

## Code Structure Overview

```
ratix (HTML file)
├── HTML/CSS Section (lines 1-1800)
│   └── Game board UI, modals, styling
│
└── JavaScript Section (lines 1800+)
    ├── Constants & Configuration (lines 2150-6500)
    │   ├── Game rules & mechanics
    │   ├── Piece definitions
    │   ├── AI_PROFILES & difficulty settings
    │   └── Special ability costs/thresholds
    │
    ├── Game State Management (lines 6500-8000)
    │   ├── Board initialization
    │   ├── Piece tracking variables
    │   └── Score/capture management
    │
    ├── AI System (lines 3248-17000+)
    │   ├── Turn Control
    │   │   ├── triggerAiTurn() [3248]
    │   │   ├── aiTurn() [16948]
    │   │   ├── makeNextMove() [17134]
    │   │   └── completeAiTurn() [3279]
    │   │
    │   ├── Decision Making
    │   │   ├── minimax() [16081] ← CORE ALGORITHM
    │   │   ├── evaluateBoard() [15725]
    │   │   └── findSimpleAIAction() [15946]
    │   │
    │   ├── Special Move Logic
    │   │   ├── (Lines 16102-16595: action generation)
    │   │   ├── (Lines 16598-16623: action sorting)
    │   │   └── (Lines 16625-16748: action evaluation)
    │   │
    │   ├── Execution Helpers
    │   │   ├── simulateMove() [15836]
    │   │   ├── simulateShoot() [15867]
    │   │   ├── simulateTurn() [15880]
    │   │   └── (Many more simulation functions)
    │   │
    │   └── AI Personality System [4561-4617]
    │       ├── Berserker profile
    │       ├── Guardian profile
    │       ├── Tactician profile
    │       ├── Sorcerer profile
    │       └── Nomad profile
    │
    └── UI & Rendering (lines 8000+)
        ├── Board display functions
        ├── Move highlighting
        ├── Score/status updates
        └── Animation/visual feedback
```

---

## Algorithm Flow at a Glance

```
AI's Turn Begins
    │
    ├─→ triggerAiTurn() [3248]
    │   └─→ aiTurn() [16948]
    │
    └─→ Loop: makeNextMove() [17134]
        │
        ├─ Check King Evade needed? [17155]
        │  └─ YES → Execute + recurse
        │
        ├─ Check King Shot available? [17182]
        │  └─ YES → Execute + recurse
        │
        ├─ Check Moral Boost target? [17213]
        │  └─ YES → Execute + recurse
        │
        ├─ If no planned actions:
        │  └─→ minimax() [16081] ← DECISION POINT
        │      │
        │      ├─ Generate all possible actions [16102]
        │      │  ├─ Normal moves/shoots
        │      │  ├─ Special abilities (Inferno, Strafe, Dark Void, Barrage, Summons)
        │      │  └─ Turn actions + compounds
        │      │
        │      ├─ Sort by priority [16598]
        │      │  1. Capture value (highest first)
        │      │  2. Action cost (spend more points)
        │      │  3. Special abilities (preferred)
        │      │  4. Type (avoid unnecessary turns)
        │      │
        │      ├─ Evaluate top N actions [16625]
        │      │  For each action:
        │      │    • Simulate on board clone
        │      │    • Recurse: minimax(..., depth-1, ...) 
        │      │    • Score = evaluateBoard() [15725]
        │      │    • Track best (alpha-beta pruning)
        │      │
        │      └─→ Return best action + evaluation
        │
        ├─ Execute chosen action
        ├─ Update visuals/scores
        └─ scheduleNextMove() [delay 3-7 seconds]
            └─ LOOP until movesLeft <= 0
                │
                └─→ completeAiTurn() [3279]
                    ├─ Apply end-turn effects
                    ├─ Reset state
                    └─ Return to player turn
```

---

## Debugging & Key Variables to Monitor

### Game State Snapshots
```javascript
// Current board state
console.log(board);

// Current AI's remaining moves
console.log(movesLeft);

// Pieces already moved this turn
console.log(movedPieces);

// Current/captured pieces
console.log(capturedPieces);
console.log(scores);

// AI configuration
console.log('Difficulty:', aiDifficulty);
console.log('AI Profile:', currentAiProfile);
console.log('Max Search Depth:', aiMaxDepth);
```

### During minimax Execution
```javascript
// Time limit
console.log('AI deadline:', new Date(aiSearchDeadline));
console.log('Time remaining:', aiSearchDeadline - Date.now());

// Search state
console.log('Current depth:', depth);
console.log('Maximizing player:', isMaximizing);
console.log('Possible actions:', allPossibleActions.length);
```

### Action Selection
```javascript
// Best action found
console.log('Action:', bestAction);
console.log('Action type:', bestAction.type); // 'move', 'shoot', 'summon', etc.
console.log('Special ability:', bestAction.special); // 'inferno', 'darkVoid', etc.
console.log('Score:', eval);
```

---

## Adding New Special Move Support

To add a new special move to AI decision-making:

1. **Add to minimax action generation** (after line 16218, before turn actions)
   ```javascript
   if (piece.type === 'YourPiece' && conditionsMet && movesLeft >= cost) {
       let targetValue = baseValue;
       if (aiDifficulty === 'Hard') targetValue *= 1.5;
       else if (aiDifficulty === 'Medium') targetValue *= 1.3;
       allPossibleActions.push({
           type: 'move',
           pos: [row, col],
           from: [pRow, pCol],
           cost: costValue,
           special: 'yourSpecialMove',
           targetValue
       });
   }
   ```

2. **Add simulation logic** (after line 16655)
   ```javascript
   if (special === 'yourSpecialMove') {
       // Clone board, apply effects
       newBoard = cloneBoard(board);
       // Update captured pieces/scores as needed
   }
   ```

3. **Add execution handler** (in aiTurn's completeAiMove function, after line 17007)
   ```javascript
   if (special === 'yourSpecialMove') {
       // Execute on actual game board
       // Update tracking variables
       // Log action
   }
   ```

