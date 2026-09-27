# 🔴 Critical Issues Found - Must Fix Before Deployment

## Issue #1: Attacker STAY/SHOOT Buttons Not Appearing ❌

**Problem**: When attacker reaches the final row (goal zone), the STAY/SHOOT choice buttons do NOT appear.

**Root Cause Analysis**:
```typescript
// Line 626-640: Condition checks for goal zone
if (moved.type === 'A' && triggersShoot(moved) && !defender) {
  // Show STAY/SHOOT choice
  attackerScoringChoice: moved.id
}
```

**The bug**: This condition only checks for `defender` (type 'D'), but does NOT check if there are other pieces (like goalkeeper, another attacker, or seeker) on the same square!

**Example Scenario**:
1. Attacker moves to row 1 (Team 1's goal zone)
2. Goalkeeper is also on that square (common in Quidditch)
3. Condition `!defender` is TRUE (no defender there)
4. But the attacker is **sharing the square with goalkeeper**
5. Code falls through to default case → ends turn WITHOUT showing STAY/SHOOT UI ❌

**Console Logs Added**:
- `[ATTACKER MOVE] Reached goal zone without defender` - should fire when buttons appear
- `[MOVE] No special condition met, ending turn normally` - fires when buttons don't appear
- Check `piecesOnSameSquare` array to see what's blocking

**Fix Needed**:
The condition is CORRECT - multiple pieces CAN occupy the same square (up to 5). The real issue might be:
1. UI not rendering due to React state issue?
2. `attackerScoringChoice` getting cleared immediately?
3. Race condition with turn switching?

**Test Case**:
1. Move attacker to final row (row 1 for Team 1, row 5 for Team 2)
2. Check browser console for logs
3. Verify `gs.attackerScoringChoice` is set in React DevTools
4. Check if UI component is rendering

---

## Issue #2: Coin Flip Winner Not Starting Game ⚠️

**Problem**: User reports "the one who starts do not work successfully"

**Current Implementation**:
1. ✅ Room page: Coin flip sets `coinFlipResult` (1 or 2)
2. ✅ Room page: Navigate with `?starter=${coinFlipResult}` parameter
3. ✅ Game page: Read `starter` from URL → `starterTeam`
4. ✅ Game page: Pass `starterTeam` to `initGS()`
5. ✅ Game page: Set `turn: starterTeam` in initial state
6. ✅ Game page: Override database `turn` during deployment phase

**Potential Issues**:
1. **Multiple clients race condition**: If 2 clients load simultaneously, one might overwrite the other's state
2. **Database persists wrong starter**: If database was created BEFORE the fix, it has old turn value
3. **URL parameter lost**: If page refreshes or user navigates back, parameter might be lost
4. **Team number confusion**: Code uses 1=Purple, 2=Yellow - verify this matches coin flip display

**Console Logs to Check**:
```
[COIN FLIP] Server returned starting_team: X (1=Purple, 2=Yellow)
[NAVIGATE] Going to game with starter: X (1=Purple, 2=Yellow)
[INITIAL LOAD] Deployment phase - using starter team from URL: X
```

**Test Case**:
1. Run coin flip 5 times
2. For each flip, verify winner matches who starts deployment
3. Check console logs for all 3 messages above
4. Verify `gs.turn` in React DevTools matches coin flip winner

---

## Additional Feature Verification Needed

### ✅ Features That Look Correct (Need Live Testing):

1. **Attacker Combat Flow**:
   - Line 608-626: Combat triggers when attacker meets defender ✅
   - Line 773-819: Combat resolution (win/lose) ✅
   - Line 782-811: After combat win in goal zone → STAY/SHOOT choice ✅

2. **Attacker SHOOT from STAY**:
   - Line 719: STAY sets `readyToShoot: true` ✅
   - Line 730-757: ATTACKER_SHOOT triggers duel ✅
   - Line 3796-3807: UI shows SHOOT button when `readyToShoot` ✅

3. **Duel (RPS)**:
   - Line 851-901: Both players choose, resolve goal/save ✅
   - Line 868-871: Streak bonus (2 consecutive goals = 20 pts) ✅

4. **Bludger**:
   - Line 937-951: Fire bludger, detect hits ✅
   - Line 959-980: Resolve, knock back, disable 1 turn ✅

5. **Seeker Bonus Move**:
   - Line 646-657: After seeker move → offer bonus move ✅
   - Line 670-687: SEEKER_CONTINUE or END_BONUS_TURN ✅
   - Line 3243-3246: Block seeker selection during bonus ✅

6. **Snitch**:
   - Line 518: Spawn after 4 moves (2 complete turns) ✅
   - Line 2450-2507: Team 1 generates wheel (2s animation) ✅
   - Line 421-468: Wait 2 moves after landing → encounter ✅
   - Line 1147-1206: Catch wheel, winner gets +50 pts ✅

7. **Captain Permissions**:
   - Line 3199, 3237, 3276, 3318: Frontend blocks non-captains ✅
   - Migration 030: Backend validates captain_id ✅

8. **Player Action Permissions**:
   - Line 3177-3179: Only piece controller sees duel UI ✅
   - Line 3740: Only attacker controller sees STAY/SHOOT ✅
   - Line 3794: Only piece controller sees SHOOT button ✅

9. **Realtime Updates**:
   - Line 2139: ALL actions broadcast immediately ✅
   - Line 2932-2940: Spectators receive all actions ✅
   - Line 2142-2147: All actions in needsSave list ✅

10. **Deployment Movement**:
    - Line 3199-3214: Captain selects piece ✅
    - Line 3276-3296: Captain moves piece ✅
    - Line 3288-3303: Captain places new piece ✅

---

## Testing Priority

### 🔴 CRITICAL (Must Fix):
1. ❌ Attacker STAY/SHOOT buttons at final row
2. ⚠️ Coin flip starter verification

### 🟡 HIGH (Should Test):
3. Combat → STAY → SHOOT flow
4. Duel RPS with both outcomes
5. Bludger hit detection
6. Seeker bonus move blocking
7. Snitch 2-move wait and encounter
8. Captain permission enforcement

### 🟢 MEDIUM (Verify Working):
9. Deployment piece movement
10. Realtime spectator updates
11. Player-specific action UI
12. Streak bonus scoring

---

## Next Steps

1. **Deploy to test environment** with logging enabled
2. **Test attacker movement** to final row - check console logs
3. **Run 5 coin flip tests** - verify starter matches winner
4. **Full playthrough test** - verify all features work end-to-end
5. **Fix critical issues** based on log output
6. **Redeploy and retest**

---

## Console Log Patterns to Look For

### If Attacker Buttons Don't Appear:
```
[MOVE] Piece moved: { id: 'X', type: 'A', to: 'A1' }
[ATTACKER MOVE] Defender found, triggering combat    // Combat path
  OR
[ATTACKER MOVE] Reached goal zone without defender   // Should show buttons
  OR
[MOVE] No special condition met, ending turn normally // BUG - buttons should show
```

### If Coin Flip Wrong:
```
[COIN FLIP] Server returned starting_team: 1 (1=Purple, 2=Yellow)
[NAVIGATE] Going to game with starter: 1
[INITIAL LOAD] Deployment phase - using starter team from URL: 1
// But deployment shows Team 2 starting → BUG
```

---

**Status**: Logging added, build passing ✅  
**Ready for**: Live testing with console monitoring 🧪
