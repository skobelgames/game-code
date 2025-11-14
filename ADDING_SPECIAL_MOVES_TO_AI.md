# Adding New Special Moves to AI Decision-Making

## Overview
When you add a new special move to the Wizard (or any piece), you need to integrate it into 3 places in the AI system:
1. **Action Generation** - Make AI aware the move exists
2. **Action Simulation** - Evaluate what happens if AI uses it
3. **Action Execution** - Actually perform it during AI's turn

---

## Step 1: Generate the Action in minimax()

**Location**: Lines 16102-16595 in `/home/user/game-code/ratix`

Find the section for your piece type and add action generation logic:

```javascript
// EXAMPLE: Wizard new special move "Arcane Burst"
// Add this AFTER the Wizard Spectre Summoning code (after line 16237)
// and BEFORE the Necromancer summoning code (line 16239)

// Check for Wizard Arcane Burst
if (piece.type === 'Wizard' && arcaneBlastUsed[player] < ARCANE_BLAST_LIMIT && movesLeft >= 2) {
    // Get valid targets for Arcane Burst
    const targets = getArcaneBlastTargets(r, c, player);
    
    for (let [tr, tc] of targets) {
        // Calculate value of hitting this target
        let burstValue = 0;
        const targetPiece = board[tr][tc];
        
        if (targetPiece && targetPiece.player === opponent) {
            burstValue = aiPieceValues[targetPiece.type] * 2.0;  // 200% of piece value
            
            // KING TARGETING: Massively increase value
            if (targetPiece.type === 'King') {
                burstValue = aiPieceValues['King'] * 30.0;  // Game-winning move
            } else if (targetPiece.type === 'Champion') {
                burstValue *= 1.5;  // Extra bonus for high-value pieces
            }
        }
        
        // Apply difficulty scaling
        if (aiDifficulty === 'Hard') {
            burstValue *= 1.5;  // 50% better evaluation
        } else if (aiDifficulty === 'Medium') {
            burstValue *= 1.3;  // 30% better evaluation
        }
        
        allPossibleActions.push({
            type: 'move',                    // Or 'shoot', 'summon' depending on mechanic
            pos: [tr, tc],                   // Target position
            from: [r, c],                    // Wizard's position
            cost: 2,                         // Action point cost
            special: 'arcaneBurst',          // Unique identifier
            pieceType: 'Wizard',
            targetValue: burstValue          // Used for sorting
        });
    }
}
```

### Key Variables to Define Elsewhere

You'll need constants defined earlier in the file (around line 2280):

```javascript
// Add near other special move constants
const ARCANE_BLAST_LIMIT = 2;  // Max uses per game
let arcaneBlastUsed = { 'W': 0, 'B': 0 };  // Track uses

// Add helper function
function getArcaneBlastTargets(row, col, player) {
    const targets = [];
    const opponent = player === 'W' ? 'B' : 'W';
    
    // Example: Returns all opponent pieces within 5 squares
    for (let r = 0; r < ROWS; r++) {
        for (let c = 0; c < COLS; c++) {
            const piece = board[r][c];
            if (piece && piece.player === opponent) {
                const distance = Math.hypot(r - row, c - col);
                if (distance <= 5 && distance > 0) {
                    targets.push([r, c]);
                }
            }
        }
    }
    return targets;
}
```

---

## Step 2: Simulate the Action in minimax()

**Location**: Lines 16642-16731 (in the evaluation section)

Add a new condition to handle your special move's board state:

```javascript
// EXAMPLE: Handling Arcane Burst in the action evaluation loop

if (type === 'move') {
    if (special === 'darkVoid') {
        // ... existing Dark Void code ...
    } else if (special === 'arcaneBurst') {
        // NEW: Handle Arcane Burst
        newBoard = cloneBoard(board);
        const targetRow = pos[0];
        const targetCol = pos[1];
        const targetPiece = newBoard[targetRow][targetCol];
        
        if (targetPiece && targetPiece.player === opponent) {
            // Remove captured piece
            newCapturedPieces[player].push(targetPiece);
            newScores[player] += aiPieceValues[targetPiece.type];
            newBoard[targetRow][targetCol] = null;
            
            // Arcane Burst might affect adjacent pieces too
            const directions = [[0,1], [0,-1], [1,0], [-1,0], [1,1], [1,-1], [-1,1], [-1,-1]];
            for (let d of directions) {
                const ar = targetRow + d[0];
                const ac = targetCol + d[1];
                if (isValid(ar, ac) && newBoard[ar][ac]) {
                    const adjPiece = newBoard[ar][ac];
                    // Damage but don't capture adjacent pieces (or capture, your choice)
                    // Example: reduce durability if pieces have health
                }
            }
        }
        newMovedPieces.add(`${r},${c}`);
    } else {
        // ... existing move simulation code ...
    }
}
```

---

## Step 3: Execute the Action in aiTurn()

**Location**: Lines 17977-17023 (in `completeAiMove` function)

Add execution logic after the Ogre Rage handling:

```javascript
// EXAMPLE: Execute Arcane Burst
// Add this AFTER the existing special move handling (after line 17007)

if (special === 'arcaneBurst') {
    // Track usage
    arcaneBlastUsed['B']++;
    
    // Apply the actual effects
    const targetRow = action.pos[0];
    const targetCol = action.pos[1];
    const targetPiece = board[targetRow][targetCol];
    
    if (targetPiece && targetPiece.player === 'W') {
        // Capture the target
        capturedPieces['B'].push(targetPiece);
        scores['B'] += aiPieceValues[targetPiece.type];
        board[targetRow][targetCol] = null;
        
        // Log the action
        const aiLabel = getOpponentDisplayName();
        const targetLabel = formatBoardCoordinates(targetRow, targetCol);
        gameLog.push(`${aiLabel} cast Arcane Burst at ${targetLabel}!`);
        
        // Update UI
        updateCapturedPiecesDisplay();
        updateScoreDisplay();
        
        // Handle adjacent effects if any
        // ...
    }
}
```

---

## Step 4: Handle Player-Side Execution (if needed)

If the player can also use this move, add similar logic to the player-side action handlers:

**Location**: Search for where other special moves handle player input (search for "activateEnergyBlast", etc. around line 10000+)

```javascript
// EXAMPLE: Player can also use Arcane Burst
// Add button in HTML (around line 1900):
// <button id="arcane-burst-button" onclick="activateArcaneBurst(this)" style="display: none;">Arcane Burst</button>

// Add JavaScript handler:
function activateArcaneBurst(button) {
    // Validate preconditions
    if (arcaneBlastUsed['W'] >= ARCANE_BLAST_LIMIT) {
        alert('Arcane Burst already used maximum times');
        return;
    }
    
    if (movesLeft < 2) {
        alert('Need 2 moves remaining for Arcane Burst');
        return;
    }
    
    // Highlight valid targets
    arcaneBlastMode = true;
    validMoves = getArcaneBlastTargets(selectedPiece.row, selectedPiece.col, 'W')
        .map(([tr, tc]) => ({ type: 'special', pos: [tr, tc], special: 'arcaneBurst' }));
    
    // User clicks a target, which calls playerCastArcaneBurst(targetRow, targetCol)
    renderBoard();
}

function playerCastArcaneBurst(targetRow, targetCol) {
    // Apply effects same as AI does
    const targetPiece = board[targetRow][targetCol];
    
    if (targetPiece && targetPiece.player === 'B') {
        capturedPieces['W'].push(targetPiece);
        scores['W'] += aiPieceValues[targetPiece.type];
        board[targetRow][targetCol] = null;
        
        gameLog.push(`Player cast Arcane Burst at ${formatBoardCoordinates(targetRow, targetCol)}!`);
        arcaneBlastUsed['W']++;
        movedPieces.add(`${selectedPiece.row},${selectedPiece.col}`);
        movesLeft -= 2;
        
        arcaneBlastMode = false;
        selectedPiece = null;
        validMoves = [];
        
        updateCapturedPiecesDisplay();
        updateScoreDisplay();
        renderBoard();
        updateStatus();
        ensureAITurnIfNeeded();
    }
}
```

---

## Step 5: Add Difficulty Scaling

The AI adjusts special move values based on difficulty. Add scaling throughout:

```javascript
// SCALABLE DIFFICULTY BONUSES (used in action generation):

// Easy: Base value
// Medium: 1.3x multiplier
// Hard: 1.5x multiplier

let burstValue = 5.0;  // Base value

if (aiDifficulty === 'Hard') {
    burstValue *= 1.5;  // Hard difficulty AI values special moves 50% more
} else if (aiDifficulty === 'Medium') {
    burstValue *= 1.3;  // Medium values them 30% more
}
// Easy uses base value (1.0x)

allPossibleActions.push({
    // ... action details ...
    targetValue: burstValue
});
```

---

## Complete Integration Checklist

```
[ ] Add action generation code to minimax() [16102-16595]
    [ ] Check piece type and conditions
    [ ] Calculate targetValue based on effects
    [ ] Apply difficulty multiplier
    [ ] Push to allPossibleActions

[ ] Add simulation code to minimax() [16642-16731]
    [ ] Clone board
    [ ] Apply effects to cloned board
    [ ] Update captured pieces/scores
    [ ] Return modified board state

[ ] Add execution code to aiTurn() [17977-17023]
    [ ] Apply actual effects to game board
    [ ] Track usage counter
    [ ] Log action
    [ ] Update UI (captures, scores, board)

[ ] If player-accessible:
    [ ] Add HTML button
    [ ] Add activation function
    [ ] Add click handler
    [ ] Add validation logic
    [ ] Add board state update

[ ] Add constants [around line 2280]:
    [ ] MAX_USES limit constant
    [ ] Usage tracking dictionary
    [ ] Helper functions if needed

[ ] Add difficulty scaling throughout
    [ ] Evaluation phase (1.3x-1.5x)
    [ ] Execution phase (logging only)
```

---

## Testing Your New Special Move

### Test AI Usage
1. Set AI to `Hard` difficulty
2. Create a board position where the special move is valuable
3. Step through makeNextMove() in debugger
4. Verify:
   - Action appears in `allPossibleActions`
   - Action is scored with difficulty multiplier
   - Action is selected if it's the highest-value option
   - Action executes correctly

### Test Player Usage
1. Play as human player
2. Use the special move on opponent
3. Verify:
   - Button only appears when conditions met
   - Click highlights valid targets
   - Move executes correctly
   - AI responds appropriately next turn

### Debug Output
```javascript
// Log action generation:
console.log('Action added:', {
    special: 'arcaneBurst',
    targetValue: burstValue,
    from: [r, c],
    to: [tr, tc]
});

// Log evaluation:
console.log('Action evaluated:', {
    action,
    eval: eval,
    rank: allPossibleActions.indexOf(action)
});

// Log execution:
console.log('Action executed:', {
    special: 'arcaneBurst',
    captured: targetPiece.type,
    newScore: scores['B']
});
```

---

## Common Pitfalls

1. **Forgetting difficulty multiplier** - AI special moves MUST be scaled by difficulty
2. **Not tracking usage** - Many moves have limited uses, need to track and check
3. **Incorrect piece values in evaluation** - Use `aiPieceValues`, not `pieceValues`
4. **Missing alpha-beta pruning checks** - Always call `aiTimeExpired()` in loops
5. **Board mutation** - Always `cloneBoard()` before simulation, never mutate original
6. **Unhandled edge cases** - Check piece type exists before accessing properties

---

## Related Code Areas

- **Board state**: `board[row][col]` is a piece object with `.type`, `.player`, `.facing`
- **Piece evaluation**: `aiPieceValues[pieceType]` gives difficulty-adjusted value
- **Board updates**: Use `updateCapturedPiecesDisplay()`, `updateScoreDisplay()`, `renderBoard()`
- **Logging**: `gameLog.push()` adds to game message history
- **UI state**: `selectedPiece`, `validMoves`, `movedPieces` track interaction state

