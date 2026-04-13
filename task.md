## RPG inventory and battle system

You are building the core logic for a text-based RPG. The system manages a player's inventory of items, applies item effects during battle, enforces weight limits, handles item durability, simulates turn-based combat with enemies, and produces a detailed end-of-battle report. This must be a terminal project: everything should run in the terminal, and all actions/results must be shown with clear wording in terminal output.

## Part 1 - Data structures

### 1) Define the player and item data

Create the following data structures exactly as described. You will pass them into every function - no global variables allowed.

```python
player = {
    "name": "Aric",
    "hp": 100,
    "max_hp": 100,
    "attack": 18,
    "defense": 8,
    "gold": 120,
    "inventory": [],      # list of item dicts (see below)
    "status": None,       # None | "poisoned" | "burning" | "stunned"
    "kills": 0,
}

# Each item looks like:
item_template = {
    "name": str,
    "type": str,          # "weapon" | "potion" | "armor" | "scroll"
    "effect": str,        # e.g. "heal", "damage_boost", "shield", "burn"
    "value": int,         # potency of the effect
    "weight": float,      # in kg
    "durability": int,    # uses remaining; -1 means infinite
    "cost": int,
}

MAX_CARRY_WEIGHT = 20.0   # kg
```

Populate the player's inventory with at least 6 items of mixed types, including at least one item with durability that will run out during testing.

## Part 2 - Inventory functions

### 2) `add_item(player, item)` _(function, if/else)_

Adds an item to the player's inventory. Must enforce:

- Total carried weight must not exceed `MAX_CARRY_WEIGHT` after adding.
- The player cannot carry more than 10 items.
- No duplicate item names allowed.

Return `True` if added successfully, `False` with a specific printed reason if not.

> Trap: weight is a float - use rounding to 2 decimal places when summing, or floating-point errors will cause bugs.

### 3) `remove_item(player, item_name)` _(function, loop)_

Searches the inventory by name (case-insensitive) and removes the item. Returns the removed item dict, or `None` if not found. Do not use `list.remove()` - find by iterating and splice by index.

### 4) `get_items_by_type(player, item_type)` _(loop)_

Returns a new list of all items in the inventory matching the given type. The result must be sorted by `value` descending. Use a loop to build the list - no list comprehensions.

### 5) `total_inventory_value(player)` _(loop)_

Returns the total gold value of all items in the inventory. Additionally, print a formatted inventory table: each row shows item name, type, durability (or `"inf"` if `-1`), and cost - right-aligned columns with a fixed width of 16 characters each.

Hint: use Python's f-string formatting: `f"{value:>16}"` right-aligns a value in a 16-char field. For `"inf"` durability, use an if/else inside the loop.

## Part 3 - Item use and effects

### 6) `use_item(player, item_name, target=None)` _(match/case, if/else)_

Applies the item's effect using `match item["effect"]`. Handle all cases below. After use, reduce durability by 1; if durability reaches 0, remove it from inventory automatically. If durability was already 0, print a warning and do nothing.

```python
match item["effect"]:
    case "heal":
        # restore player["value"] HP, but never exceed max_hp
    case "damage_boost":
        # temporarily add item["value"] to player["attack"] (just for this battle turn)
    case "shield":
        # add item["value"] to player["defense"] for this turn
    case "burn":
        # set target["status"] = "burning", store burn_value = item["value"]
    case "poison":
        # set target["status"] = "poisoned"
    case "stun":
        # set target["status"] = "stunned" - target skips next turn
    case _:
        # print unknown effect warning
```

> Trap: healing must be capped at `max_hp`. New players often just do `hp += value` and overshoot.

## Part 4 - Battle system

### 7) `apply_status_effects(entity)` _(match/case)_

Called at the start of each entity's turn. Applies ongoing damage or effects based on `entity["status"]`:

- `"poisoned"` - deal 5 damage per turn. Lasts 3 turns then clears.
- `"burning"` - deal 8 damage per turn. Lasts 2 turns then clears.
- `"stunned"` - entity cannot act this turn (return `False` to signal skip). Clears after 1 turn.
- `None` - do nothing, return `True`.

You must track how many turns the status has been active. Add a `"status_turns"` key to the entity dict to manage this.

Hint: initialize `entity.setdefault("status_turns", 0)` safely. Increment each turn, then check against the limit and clear both `status` and `status_turns` when expired.

### 8) `enemy_turn(enemy, player)` _(if/else)_

Simulates an enemy attack. The enemy has `attack`, `hp`, and `status` keys. Damage formula:

```python
damage = max(0, enemy["attack"] - player["defense"])
player["hp"] -= damage
```

If the enemy is stunned, skip its attack (call `apply_status_effects` which returns `False`). If the enemy has a `special` key with a value like `"double_strike"`, it attacks twice. Print what happens each turn.

### 9) `run_battle(player, enemies)` _(loop, if/else)_

The main battle loop. `enemies` is a list of enemy dicts. Battle proceeds in rounds: player acts first, then each enemy. Continue until all enemies are dead or the player's HP drops to 0 or below. Rules:

- Each round, call `apply_status_effects` on the player first.
- Player automatically uses the best healing potion from inventory if HP falls below 30% of max.
- Player attacks the first living enemy (`hp > 0`). Track kills on the player dict.
- Dead enemies (`hp <= 0`) are skipped in subsequent rounds.
- Print a round header and every action taken each round.
- Battle ends if player `hp <= 0` (defeat) or all enemies `hp <= 0` (victory).

> Trap: looping over enemies while checking if they are alive - make sure you never attack an already-dead enemy or count it as living when checking end conditions.

## Part 5 - End-of-battle report

### 10) `battle_report(player, enemies, rounds)` _(function, loop)_

Print a formatted summary after the battle ends. Must include:

- Outcome: `"Victory"` or `"Defeat"`
- Rounds lasted
- Player final HP and kills
- Each enemy: name, final HP, and whether they survived
- Remaining inventory with durability of each item
- Total value of remaining items

All columns must be aligned using f-string padding. No plain `print("---")` separators - use `"-" * 40` dynamically.

## Test scenario

```python
enemies = [
    {"name": "Goblin",   "hp": 45,  "attack": 12, "defense": 3,  "status": None},
    {"name": "Troll",    "hp": 90,  "attack": 20, "defense": 6,  "status": None,
     "special": "double_strike"},
    {"name": "Wraith",   "hp": 60,  "attack": 15, "defense": 2,  "status": None},
]

# Expected behaviors to verify:
# - Player auto-heals when HP < 30
# - Troll attacks twice per turn
# - A poison/burn scroll gets used and tracks turns correctly
# - At least one item runs out of durability and is removed
# - Battle report prints aligned columns
```

## Bonus

- Add a `shop(player, shop_inventory)` function - player can browse, afford check, weight check, and buy items in a loop until they type `"done"`.
- Add a `level_up(player, xp_gained)` function - every 100 XP increases attack by 2, defense by 1, max_hp by 10 and fully heals the player. Handle multiple level-ups in one call with a while loop.
- Implement item `"type": "scroll"` with effect `"revive"` - if the player would die (`hp <= 0`), automatically trigger the scroll if one exists in inventory, restore to 25% max_hp, and remove the scroll. Requires intercepting the death condition inside `run_battle`.
