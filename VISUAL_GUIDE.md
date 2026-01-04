# 1-CNF Belief State Agent - Visual Guide

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                  SimplifiedWumpusAgent                           │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ execute(percept)                                         │   │
│  │                                                          │   │
│  │  1. _encode_percepts(percept)                           │   │
│  │     └→ Add facts to KB.clauses                          │   │
│  │                                                          │   │
│  │  2. belief_state = kb.update_belief_state(candidates)  │   │
│  │     └→ Test 3n²+4 symbols with pl_resolution()         │   │
│  │        Return set of proven literals                    │   │
│  │                                                          │   │
│  │  3. _update_position_from_belief()                      │   │
│  │     └→ Find location(x,y) in belief_state (O(n²))      │   │
│  │        Find orientation symbol (O(1))                   │   │
│  │                                                          │   │
│  │  4. safe_points = _get_safe_cells()                     │   │
│  │     └→ Filter cells where ~pit AND ~wumpus (O(n²))     │   │
│  │                                                          │   │
│  │  5. Plan actions (grab → explore → shoot → return)     │   │
│  │     └→ Use plan_route(astar_search)                    │   │
│  │                                                          │   │
│  │  6. Return action; t += 1                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Private Attributes:                                            │
│  ├─ kb: SimpleBeliefStateKB                                     │
│  ├─ belief_state: Set[Expr]        ← 1-CNF beliefs             │
│  ├─ current_position: WumpusPosition                            │
│  ├─ plan: List[str]                ← Action queue              │
│  ├─ t: int                          ← Timestep                  │
│  └─ dimrow: int                     ← Grid dimension            │
└─────────────────────────────────────────────────────────────────┘
         │
         │ uses
         ▼
┌─────────────────────────────────────────────────────────────────┐
│              SimpleBeliefStateKB(PropKB)                         │
│                                                                   │
│  Static Rules (initialized once, never changed):               │
│  ├─ ~Wumpus(1,1)                                               │
│  ├─ ~Pit(1,1)                                                  │
│  ├─ Breeze(x,y) ↔ (Pit(x±1,y) ∨ Pit(x,y±1))                  │
│  ├─ Stench(x,y) ↔ (Wumpus(x±1,y) ∨ Wumpus(x,y±1))            │
│  ├─ Wumpus(1,1) ∨ ... ∨ Wumpus(n,n)  [At least one]          │
│  └─ ¬(Wumpus(i,j) ∧ Wumpus(i',j')) for all pairs [At most one]│
│                                                                   │
│  Per-Timestep Process:                                          │
│  1. update_belief_state(candidates)                            │
│     ├─ candidates = enumerate_grid_candidate_symbols(dimrow)   │
│     ├─ For each symbol S in candidates:                        │
│     │  ├─ IF pl_resolution(self, S):   add S to beliefs       │
│     │  ├─ ELSE IF pl_resolution(self, ~S):  add ~S            │
│     │  └─ ELSE: skip (unknown)                                │
│     └─ Return belief_state as Set[Expr]                       │
│                                                                   │
│  2. is_believed(literal) → bool                                │
│     └─ return literal in self.belief_state   [O(1)]            │
│                                                                   │
│  Internal Attributes:                                           │
│  ├─ belief_state: Set[Expr]  ← 1-CNF beliefs                  │
│  ├─ dimrow: int              ← Grid size                       │
│  └─ clauses: List[Expr]      ← Inherited from PropKB           │
└─────────────────────────────────────────────────────────────────┘
         │
         │ generates candidates for
         ▼
┌─────────────────────────────────────────────────────────────────┐
│       enumerate_grid_candidate_symbols(dimrow) → List[Expr]    │
│                                                                   │
│  Returns flat list of 3n²+4 symbols to test:                   │
│                                                                   │
│  Location Layer (n²):                                          │
│  ├─ location(1,1), location(1,2), ... location(n,n)           │
│  └─ [represents "where is agent now?"]                         │
│                                                                   │
│  Pit Layer (n²):                                               │
│  ├─ pit(1,1), pit(1,2), ..., pit(n,n)                         │
│  └─ [represents "pit locations"]                               │
│                                                                   │
│  Wumpus Layer (n²):                                            │
│  ├─ wumpus(1,1), wumpus(1,2), ..., wumpus(n,n)                │
│  └─ [represents "where is wumpus?"]                            │
│                                                                   │
│  Orientation Layer (4):                                        │
│  ├─ Expr('FacingNorth')                                        │
│  ├─ Expr('FacingSouth')                                        │
│  ├─ Expr('FacingEast')                                         │
│  ├─ Expr('FacingWest')                                         │
│  └─ [represents "which direction?"]                            │
│                                                                   │
│  Total for 4×4: 16+16+16+4 = 52 symbols                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Belief State Extraction Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ Timestep t: Belief State Extraction                             │
└─────────────────────────────────────────────────────────────────┘

Input:  SimpleBeliefStateKB with static rules + current percepts
        Candidate symbols: [location(x,y), pit(x,y), wumpus(x,y), facing_*]

Process:

  OLD belief_state = {L(1,2), ~P(2,3), W(4,1), ...}   ← Discarded
                      │
                      ▼ (fully replaced)
  
  new_beliefs = {}  ← Empty set to build
  
  FOR each symbol S in candidates:
    │
    ├─ Test positive:  pl_resolution(KB, S)?
    │  │
    │  ├─ YES → new_beliefs.add(S)
    │  │        Example: pl_resolution proves location(1,2)
    │  │        Result: location(1,2) ∈ new_beliefs
    │  │
    │  └─ NO → Continue to negation test
    │
    ├─ Test negative: pl_resolution(KB, ~S)?
    │  │
    │  ├─ YES → new_beliefs.add(~S)
    │  │        Example: pl_resolution proves ~pit(2,3)
    │  │        Result: ~pit(2,3) ∈ new_beliefs
    │  │
    │  └─ NO → S is unknown (skip it)
    │
    └─ Move to next symbol
  
  DONE
  
Output: NEW belief_state = {location(1,2), ~pit(2,3), wumpus(4,1), FacingEast, ...}
        
Query Example:
  ├─ is_believed(location(1,2))     → True  (in set)
  ├─ is_believed(pit(2,3))          → False (not in set)
  ├─ is_believed(~pit(2,3))         → True  (in set)
  └─ is_believed(location(2,2))     → False (unknown, not proven)
```

---

## Comparison: Temporal Explosion

### HybridWumpusAgent (KB Growing)

```
Timestep 0: KB = [static rules + initial state]
           Size = 50 clauses

Timestep 1: KB = [static rules + initial state + t=0 transition rules]
           Size = 50 + 50 = 100 clauses

Timestep 2: KB = [static rules + initial state + t=0 rules + t=1 rules]
           Size = 50 + 50 + 50 = 150 clauses

...

Timestep 20: KB = [static rules + initial state + t=0,...,t=19 rules]
            Size = 50 + 20×50 = 1050 clauses

Complexity: Query cost ∝ KB size
           → Timestep 20 query ~20x slower than Timestep 1
           → Memory grows linearly
           → Becomes impractical after ~50 timesteps
```

### SimplifiedWumpusAgent (Fixed KB)

```
Timestep 0: KB = [static rules]
           Size = 50 clauses
           Query: Test 52 symbols × pl_resolution

Timestep 1: KB = [static rules]  ← SAME KB
           Size = 50 clauses
           Query: Test 52 symbols × pl_resolution

Timestep 2: KB = [static rules]  ← SAME KB
           Size = 50 clauses
           Query: Test 52 symbols × pl_resolution

...

Timestep 20: KB = [static rules]  ← SAME KB
            Size = 50 clauses
            Query: Test 52 symbols × pl_resolution

Complexity: Query cost = constant per timestep
           → All timesteps have same cost
           → Memory stays constant
           → Scales indefinitely
```

---

## Memory Usage Comparison

```
Memory (MB)
    │
 60 │                                    ┌─ HybridWumpus
    │                                   /  (Grows linearly)
 50 │                                  /
    │                                 /
 40 │                                /
    │                               /
 30 │                              /
    │                             /
 20 │                            /
    │                           /
 10 │         ┌─ Simplified    /
    │         │ (Constant)    /
  0 │─────────┴──────────────/──────────────── Timesteps
    0        10       20        30       40      50
```

---

## Decision Tree: When to use which agent

```
                          ┌─ Wumpus Agent Choice ─┐
                          │                       │
                          ▼
                   Problem constraints?
                   
              ┌──────────────────────────┬──────────────────────────┐
              │ LARGE grid (n > 10)      │ SMALL grid (n ≤ 4)      │
              │ or                       │ and                      │
              │ LONG episodes (t > 50)   │ SINGLE episode (t < 50)  │
              └──────────────────────────┴──────────────────────────┘
                        │                              │
                        ▼                              ▼
                   [SIMPLIFIED]              [Either is fine]
            SimplifiedWumpusAgent                │
         (1-CNF, constant cost)         ┌───────┴────────┐
                                        │                │
                                        ▼                ▼
                                   Need full          Prefer
                                   temporal          simplicity?
                                   reasoning?        
                                        │                │
                                   YES │            NO  │
                                        │                │
                                        ▼                ▼
                                   [HYBRID]         [SIMPLIFIED]
                              HybridWumpusAgent  SimplifiedWumpusAgent
                           (Full inference,       (1-CNF, O(1) lookup)
                            expensive but sound)
```

---

## Symbol Testing Example (4×4 Grid)

```
Candidate symbols tested each timestep:

┌──────────────────────────────────────────────────────────────┐
│ LOCATION SYMBOLS (16)                                        │
├──────────────────────────────────────────────────────────────┤
│ location(1,1) location(1,2) location(1,3) location(1,4)      │
│ location(2,1) location(2,2) location(2,3) location(2,4)      │
│ location(3,1) location(3,2) location(3,3) location(3,4)      │
│ location(4,1) location(4,2) location(4,3) location(4,4)      │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ PIT SYMBOLS (16)                                             │
├──────────────────────────────────────────────────────────────┤
│ pit(1,1)     pit(1,2)     pit(1,3)     pit(1,4)             │
│ pit(2,1)     pit(2,2)     pit(2,3)     pit(2,4)             │
│ pit(3,1)     pit(3,2)     pit(3,3)     pit(3,4)             │
│ pit(4,1)     pit(4,2)     pit(4,3)     pit(4,4)             │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ WUMPUS SYMBOLS (16)                                          │
├──────────────────────────────────────────────────────────────┤
│ wumpus(1,1) wumpus(1,2) wumpus(1,3) wumpus(1,4)             │
│ wumpus(2,1) wumpus(2,2) wumpus(2,3) wumpus(2,4)             │
│ wumpus(3,1) wumpus(3,2) wumpus(3,3) wumpus(3,4)             │
│ wumpus(4,1) wumpus(4,2) wumpus(4,3) wumpus(4,4)             │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ ORIENTATION SYMBOLS (4)                                      │
├──────────────────────────────────────────────────────────────┤
│ FacingNorth    FacingSouth    FacingEast    FacingWest       │
└──────────────────────────────────────────────────────────────┘

Total: 52 symbols tested with pl_resolution per timestep
```

---

## Code Flow Diagram

```
SimplifiedWumpusAgent.execute(percept)
│
├─→ _encode_percepts(percept)
│   └─→ KB.tell(...) [add percepts]
│
├─→ candidates = enumerate_grid_candidate_symbols(dimrow)
│   └─→ Return [location(...), pit(...), wumpus(...), facing_*]
│
├─→ belief_state = kb.update_belief_state(candidates)
│   │
│   ├─→ FOR each symbol S in candidates:
│   │   ├─→ IF pl_resolution(KB, S): beliefs.add(S)
│   │   └─→ ELSE IF pl_resolution(KB, ~S): beliefs.add(~S)
│   │
│   └─→ RETURN beliefs
│
├─→ _update_position_from_belief()
│   ├─→ FOR x,y: IF kb.is_believed(location(x,y)): set position
│   └─→ FOR facing: IF kb.is_believed(FacingX): set orientation
│
├─→ safe_points = _get_safe_cells()
│   ├─→ FOR x,y: IF is_believed(~pit) AND is_believed(~wumpus): append
│   └─→ RETURN safe_points
│
├─→ IF glitter: _plan_grab_and_return(safe_points)
├─→ ELIF unvisited: _plan_explore_unvisited(safe_points)
├─→ ELIF arrow: _plan_shoot_wumpus(safe_points)
├─→ ELSE: _plan_explore_risky(safe_points) or _plan_return_to_start
│
├─→ action = plan[0]
├─→ plan = plan[1:]
├─→ t += 1
│
└─→ RETURN action
```

---

## Belief State Visualization Example

For a 4×4 world at some timestep:

```
Grid Map (A = Agent, W = Wumpus, P = Pit, . = Safe)

  1   2   3   4
1 A . . .
2 . . P .
3 . . . .
4 . . . W

Corresponding Belief State:

✓ Provable (in belief_state):
  ├─ location(1, 1)       [Agent is here]
  ├─ ~pit(1, 1)           [Not a pit]
  ├─ ~wumpus(1, 1)        [Not wumpus]
  ├─ ~pit(1, 2)           [Not pits adjacent to agent]
  ├─ ~pit(2, 1)
  ├─ pit(2, 3)            [Pit detected from breeze]
  ├─ wumpus(4, 4)         [Wumpus inferred from stench]
  ├─ ~wumpus(1, 2)        [Not wumpus elsewhere]
  ├─ ... [more negations]
  └─ FacingEast           [Orientation]

✗ Unknown (not in belief_state):
  ├─ location(2, 2)       [Could be here after move]
  ├─ pit(1, 3)            [Not directly observed]
  └─ pit(3, 3)            [Not directly observed]

Query Examples:
  kb.is_believed(location(1, 1))      → True
  kb.is_believed(~pit(1, 1))          → True
  kb.is_believed(pit(2, 3))           → True
  kb.is_believed(~wumpus(4, 4))       → False
  kb.is_believed(pit(1, 3))           → False (unknown)
```

---

This visual guide should help understand the architecture, algorithm, and performance characteristics of the 1-CNF Belief State Agent.
