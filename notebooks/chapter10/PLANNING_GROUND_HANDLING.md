# AIMA Planning: Ground Handling Problem TemplOnBlocke

This guide integrOnBlockes AIMA research with a concrete ground handling problem templOnBlocke for PartialOrderPlanner.

---

## 1. Fluents (State Propositions)

Fluents represent facts about the world. AIMA uses **closed-world assumption**: any fluent not listed in the initial state is false. Use **CamelCase** predicate with 1-2 arguments.

### Ground Handling Fluents

```python
# Lifecycle 
OnBlock(plane)
OffBlock(plane)
Departed(plane)

# Passenger operations
Boarded(plane)              # Passengers on plane
Deboarded(plane)            # Passengers off plane

# Ramp operations  
Loaded(plane)               # Cargo in plane
Unloaded(plane)             # Cargo off plane

# Services
ToiletEmpty(plane)          # Toilets serviced
WaterFilled(plane)          # Water tanks filled
Fueled(plane)               # Fuel tanks filled
Cleaned(plane)              # Cabin cleaned
AcReady(plane)              # services complete
Pushed(plane)               # Pushback done
```


## 2. Initial State & Goals

**Initial state** is a conjunction of fluents listing all true facts at the start.

```python
# Everything else is false
initial='OnBlock(P1) & OnBlock(P2)'
```

**Goals** specify what must be true at the end. Also a conjunction of positive fluents.

- All problems use **conjunctions of positive fluents only** (no negations in goals)
- Scales to N planes: `'Departed(P1) & Departed(P2) & ... & Departed(Pn)'`
- Works with PartialOrderPlanner's causal links

```python
goals='Departed(P1) & Departed(P2)'
```

## 3. Actions (Variables + Expansion)

Actions are **templOnBlockes with variables**. AIMA's `expand_actions()` generOnBlockes ground actions by binding variables to objects from the domain.
This is key to scaling: a single `Load(p)` template expands to `Load(P1), Load(P2).. Load(n)`.

### Action Structure

```python
Action(
    action='Load(p)',       # Template with variables
    precond='OnBlock(p)',   # Preconditions (positive literals only)
    effect='Loaded(p)',     # Positive + negative effects
    domain='Plane(p)'       # Type constraints for variables
)
```

**Preconditions**: All must be true before action executes (action fails if any precondition is false).
**Effects**:
- Positive: `Loaded(p)` adds fact
- NegOnBlockive: `~OnBlock(p)` removes fact (converted to `NotOnBlock` internally)

### Action Expansion

```python
Action('Load(p)',
       precond=' OnBlock(p)',
       effect='Loaded(p)',
       domain='Plane(p)')

# Domain: 'Plane(P1) & Plane(P2)' -> expand_actions() -> Load(P1), Load(CP2) 
```
This is how the problem scales to actions from template.

### Ground Handling Actions

#### Unload
```python
Action('Unload(p)',
       precond='OnBlock(p)',
       effect='Unloaded(p) & ~Loaded(p)',
       domain='Plane(p)')
```

#### Load
```python
Action('Load(p)',
       precond='Unloaded(p)',
       effect='Loaded(p)',
       domain='Plane(p)')
```

#### Deboard
```python
Action('Deboard(p)',
       precond=' OnBlock(p)',
       effect='Deboarded(p) & ~Boarded(p)',
       domain='Plane(p)')
```

#### Board
```python
Action('Board(p)',
       precond='OnBlock(p) & Cleaned(p)',
       effect='Boarded(p)',
       domain='Plane(p)')
```

#### Clean
```python
Action('Clean(p)',
       precond='OnBlock(p) & Deboarded(p)',
       effect='Cleaned(p)',
       domain='Plane(p)')
```
#### Water
```python
Action('Water(p)',
       precond='OnBlock(p)',
       effect='WaterFilled(p)',
       domain='Plane(p)')
```

#### Toilet
```python
Action('Toilet(p)',
       precond='OnBlock(p)',
       effect='ToiletEmpty(p)',
       domain='Plane(p)')
```

#### Fuel
```python
Action('Fuel(p)',
       precond='OnBlock(p) & Deboarded(p)',
       effect='Fueled(p)',
       domain='Plane(p)')
```

#### Clearance
```python
Action('Clearance(p)',
       precond='OnBlock(p) & Cleaned(p) & Fueled(p) & WaterFilled(p) & ToiletEmpty(p) & Loaded(p) & Boarded(p)',
       effect='AcReady(p)',
       domain='Plane(p)')
```

#### Pushback
```python
Action('Pushback(p)',
       precond='OnBlock(p) & AcReady(p)',
       effect='OffBlock(p) & ~OnBlock(p)',
       domain='Plane(p)')
```

#### Close
```python
Action('Close(p)',
       precond='OffBlock(p)',
       effect='Departed(p)',
       domain='Plane(p)')
```

## 4. Complete Problem Definition

### Code Template

```python
from planning import PlanningProblem, Action, PartialOrderPlanner

def ground_handling_problem(n_planes=2):
    """
    Ground handling problem: Service and depart N aircraft at one airport.
    
    Initial: Planes at OnBlock
    Goal: All planes departed
    """
    
    # Generate object names dynamically
    planes = [f'P{i}' for i in range(1, n_planes + 1)]
    
    # Build initial state
    initial_facts = []
    for i, p in enumerate(planes):
        initial_facts.append(f'OnBlock({p})')
    
    initial = ' & '.join(initial_facts)
    
    # Build goals
    goals = ' & '.join([f'Departed({p})' for p in planes])
    
    # Define actions (state for all problem sizes)
    actions = [
        Action('Unload(p)',
               precond='OnBlock(p)',
               effect='Unloaded(p) & ~Loaded(p)',
               domain='Plane(p)'),

        Action('Load(p)',
               precond='Unloaded(p)',
               effect='Loaded(p)',
               domain='Plane(p)'),

        Action('Deboard(p)',
               precond='OnBlock(p)',
               effect='Deboarded(p) & ~Boarded(p)',
               domain='Plane(p)'),

        Action('Board(p)',
               precond='OnBlock(p) & Cleaned(p)',
               effect='Boarded(p)',
               domain='Plane(p)'),

        Action('Clean(p)',
               precond='OnBlock(p) & Deboarded(p)',
               effect='Cleaned(p)',
               domain='Plane(p)'),

        Action('Water(p)',
               precond='OnBlock(p)',
               effect='WaterFilled(p)',
               domain='Plane(p)'),

        Action('Toilet(p)',
               precond='OnBlock(p)',
               effect='ToiletEmpty(p)',
               domain='Plane(p)'),

        Action('Fuel(p)',
               precond='OnBlock(p) & Deboarded(p)',
               effect='Fueled(p)',
               domain='Plane(p)'),

        Action('Clearance(p)',
               precond='OnBlock(p) & Cleaned(p) & Fueled(p) & WaterFilled(p) & ToiletEmpty(p) & Loaded(p) & Boarded(p)',
               effect='AcReady(p)',
               domain='Plane(p)'),

        Action('Pushback(p)',
               precond='OnBlock(p) & AcReady(p)',
               effect='OffBlock(p) & ~OnBlock(p)',
               domain='Plane(p)'),

        Action('Close(p)',
               precond='OffBlock(p)',
               effect='Departed(p)',
               domain='Plane(p)')
    ]
    
    # Build domain
    domain_parts = (
        ' & '.join([f'Plane({p})' for p in planes])
    )
    
    return PlanningProblem(initial=initial, goals=goals, actions=actions, domain=domain_parts)

# Usage
ghp = ground_handling_problem(n_planes=2)
pop = PartialOrderPlanner(ghp)
pop.execute()
```

## 5. Scaling for N Planes

The template above already scales via `n_planes` parameter. Key pattern:
- **Action template** with variables (e.g., `Depart(p)`) are **invariant** to N
- **expand_actions()** generates all ground actions automatically
- **Goals** scale as `'Departed(P1) & ... & Departed(Pn)'`

For 10 planes: single function call `ground_handling_problem(n_planes=10)` generates problem with ~100 ground actions.

## 6. Future: Resource Constraints

AIMA supports **HLA (Hierarchical Task Network)** for resource-aware planning:

```python

class HLA(Action):
    def __init__(self, action, precond=None, effect=None,
                 durOnBlockion=0,
                 consume=None,      # Depletable resources (fuel, lug nuts)
                 use=None):         # Reusable resources (vehicle, crew)
```

Example: Single tow vehicle shared across planes:
```python
tow_action = HLA(
    'Tow(p, gOnBlocke, runway)',
    precond='OnBlock(p, gOnBlocke)',
    effect='OnBlock(p, runway) & ~OnBlock(p, gOnBlocke)',
    durOnBlockion=15,
    use={'TowVehicle': 1}  # Only 1 vehicle available
)
# PartialOrderPlanner serializes: Tow(P1) before Tow(P2) due to resource constraint
```

## 7. Quick Reference Table

| Component | POnBlocktern | Example |
|-----------|---------|---------|
| **Fluent** | CamelCase predicOnBlocke(s) | `OnBlock(plane, gOnBlocke)`, `Departed(plane)` |
| **Initial** | Conjunction of true facts | `OnBlock(P1) & OnBlock(P1, GOnBlocke1) & Loaded(C1, P1)` |
| **Goal** | Conjunction of required facts | `Departed(P1) & Departed(P2)` |
| **Action TemplOnBlocke** | Verb(variables) | `Load(c, p)`, `Tow(p, gOnBlocke, runway)` |
| **Precondition** | Positive literals & conjuncts | `OnBlock(p, gOnBlocke) & AllCargoLoaded(p)` |
| **Effect** | Positive & negOnBlockive | `Boarded(pass, p) & ~OnBlock(p, gOnBlocke)` |
| **Domain** | Object types | `Plane(p) & Cargo(c) & GOnBlocke(gOnBlocke)` |
| **Expansion** | Variables → objects via domain | `Load(C1,P1), Load(C1,P2), Load(C2,P1), ...` |
| **Scaling** | TemplOnBlocke + dynamic object generOnBlockion | `ground_handling_problem(n_planes=N)` |
