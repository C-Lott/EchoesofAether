Here is just a small snippet of my game.

[Screen recording 2025-10-16 09.56.15 (2).webm](https://github.com/user-attachments/assets/ab42373f-7d00-481c-bb75-b4ee3eb4ec1b)

"""
Echoes of the Aether — A Text-Based Fantasy RPG
Author: Caden & Shelbs
Run: python3 EchoesOfAether.py

Features:
- Branching story scenes with choices and consequences
- Player stats, inventory, experience, leveling
- Turn-based combat with enemies and special abilities
- Simple save/load (JSON file)
- Multiple endings based on key choices
- Journal that records important events

Notes:
- This is a single-file, terminal game intended for Python 3.8+
- Save file: echoes_save.json

Enjoy!
"""

import json
import random
import os
import sys
import textwrap
from typing import List, Dict, Any

# ------------------------- Utility functions -------------------------

def slow_print(text: str, width: int = 80):
    for line in textwrap.wrap(text, width=width):
        print(line)
    print()


def prompt(options: List[str]) -> str:
    """Prompt the player to choose an option from a list."""
    for i, opt in enumerate(options, 1):
        print(f"  {i}. {opt}")
    while True:
        choice = input("Choose: ").strip()
        if choice.isdigit() and 1 <= int(choice) <= len(options):
            return options[int(choice) - 1]
        # allow partial matching
        for opt in options:
            if choice.lower() == opt.lower() or opt.lower().startswith(choice.lower()):
                return opt
        print("Invalid choice. Enter the number or start typing an option.")


# ------------------------- Game data classes -------------------------

class Item:
    def __init__(self, name: str, description: str, heal: int = 0, power: int = 0):
        self.name = name
        self.description = description
        self.heal = heal
        self.power = power

    def to_dict(self):
        return {"name": self.name, "description": self.description, "heal": self.heal, "power": self.power}

    @staticmethod
    def from_dict(d):
        return Item(d["name"], d.get("description", ""), d.get("heal", 0), d.get("power", 0))


class Entity:
    def __init__(self, name: str, hp: int, max_hp: int, power: int, defense: int, speed: int):
        self.name = name
        self.hp = hp
        self.max_hp = max_hp
        self.power = power
        self.defense = defense
        self.speed = speed

    def is_alive(self):
        return self.hp > 0


class Player(Entity):
    def __init__(self, name: str):
        super().__init__(name, hp=30, max_hp=30, power=5, defense=2, speed=5)
        self.level = 1
        self.xp = 0
        self.next_xp = 50
        self.inventory: List[Item] = []
        self.gold = 10
        self.journal: List[str] = []
        self.flags: Dict[str, Any] = {}  # story flags

    def gain_xp(self, amount: int):
        self.xp += amount
        self.journal.append(f"Gained {amount} XP.")
        while self.xp >= self.next_xp:
            self.xp -= self.next_xp
            self.level_up()

    def level_up(self):
        self.level += 1
        self.max_hp += 8
        self.hp = self.max_hp
        self.power += 2
        self.defense += 1
        self.next_xp = int(self.next_xp * 1.6)
        self.journal.append(f"Leveled up to {self.level}!")
        slow_print(f"*** You reached level {self.level}! HP, Power, and Defense increased. ***")

    def add_item(self, item: Item):
        self.inventory.append(item)
        self.journal.append(f"Obtained: {item.name}")

    def use_item(self, index: int):
        if index < 0 or index >= len(self.inventory):
            return None
        item = self.inventory.pop(index)
        if item.heal > 0:
            healed = min(self.max_hp - self.hp, item.heal)
            self.hp += healed
            self.journal.append(f"Used {item.name} and healed {healed} HP.")
            return f"You used {item.name} and healed {healed} HP."
        return f"You used {item.name}."

    def to_dict(self):
        return {
            "name": self.name,
            "hp": self.hp,
            "max_hp": self.max_hp,
            "power": self.power,
            "defense": self.defense,
            "speed": self.speed,
            "level": self.level,
            "xp": self.xp,
            "next_xp": self.next_xp,
            "inventory": [it.to_dict() for it in self.inventory],
            "gold": self.gold,
            "journal": self.journal,
            "flags": self.flags,
        }

    @staticmethod
    def from_dict(d):
        p = Player(d.get("name", "Hero"))
        p.hp = d.get("hp", p.hp)
        p.max_hp = d.get("max_hp", p.max_hp)
        p.power = d.get("power", p.power)
        p.defense = d.get("defense", p.defense)
        p.speed = d.get("speed", p.speed)
        p.level = d.get("level", p.level)
        p.xp = d.get("xp", p.xp)
        p.next_xp = d.get("next_xp", p.next_xp)
        p.inventory = [Item.from_dict(it) for it in d.get("inventory", [])]
        p.gold = d.get("gold", p.gold)
        p.journal = d.get("journal", [])
        p.flags = d.get("flags", {})
        return p


class Enemy(Entity):
    def __init__(self, name: str, hp: int, power: int, defense: int, speed: int, xp_drop: int, gold_drop: int):
        super().__init__(name, hp=hp, max_hp=hp, power=power, defense=defense, speed=speed)
        self.xp_drop = xp_drop
        self.gold_drop = gold_drop


# ------------------------- Combat system -------------------------


def calculate_damage(attacker_power: int, defender_defense: int) -> int:
    base = max(0, attacker_power - defender_defense)
    variance = random.randint(int(base * 0.8), int(base * 1.2) if base > 0 else 1)
    return max(1, variance)


def combat(player: Player, enemies: List[Enemy]) -> bool:
    """Return True if player won, False if player died."""
    slow_print("Combat starts!")
    turn_order = sorted([player] + enemies, key=lambda e: getattr(e, "speed", 0), reverse=True)
    while player.is_alive() and any(e.is_alive() for e in enemies):
        for actor in turn_order:
            if isinstance(actor, Player):
                if not player.is_alive():
                    break
                # Player action
                options = ["Attack", "Use Item", "Defend", "Flee"]
                choice = prompt(options)
                if choice == "Attack":
                    target = choose_target([e for e in enemies if e.is_alive()])
                    dmg = calculate_damage(player.power, target.defense)
                    target.hp -= dmg
                    slow_print(f"You strike {target.name} for {dmg} damage.")
                elif choice == "Use Item":
                    if not player.inventory:
                        slow_print("You have no items.")
                    else:
                        item_choice = choose_item(player)
                        if item_choice is not None:
                            slow_print(player.use_item(item_choice))
                elif choice == "Defend":
                    slow_print("You brace yourself, reducing incoming damage this round.")
                    player.defense += 2
                elif choice == "Flee":
                    if random.random() < 0.5:
                        slow_print("You managed to escape!")
                        return True
                    else:
                        slow_print("You couldn't escape!")
                # end player action
            else:
                if not actor.is_alive():
                    continue
                # Enemy action
                if not player.is_alive():
                    break
                dmg = calculate_damage(actor.power, player.defense)
                player.hp -= dmg
                slow_print(f"{actor.name} hits you for {dmg} damage.")
        # reset temporary defense bonus
        player.defense = max(player.defense - 2, 0)

    if player.is_alive():
        total_xp = sum(e.xp_drop for e in enemies)
        total_gold = sum(e.gold_drop for e in enemies)
        player.gain_xp(total_xp)
        player.gold += total_gold
        slow_print(f"You defeated the enemies! Gained {total_xp} XP and {total_gold} gold.")
        return True
    else:
        slow_print("You have fallen in battle...")
        return False


def choose_target(targets: List[Enemy]) -> Enemy:
    print("Targets:")
    for i, t in enumerate(targets, 1):
        print(f"  {i}. {t.name} (HP: {t.hp}/{t.max_hp})")
    while True:
        choice = input("Target number: ").strip()
        if choice.isdigit() and 1 <= int(choice) <= len(targets):
            return targets[int(choice) - 1]
        print("Invalid target.")


def choose_item(player: Player) -> int:
    print("Inventory:")
    for i, it in enumerate(player.inventory, 1):
        print(f"  {i}. {it.name} - {it.description}")
    choice = input("Use which item (number) or press Enter to cancel: ").strip()
    if choice == "":
        return None
    if choice.isdigit() and 1 <= int(choice) <= len(player.inventory):
        return int(choice) - 1
    print("Invalid selection.")
    return None


# ------------------------- Scenes & Story -------------------------

class Scene:
    def __init__(self, id: str, title: str, text: str, options: Dict[str, Any]):
        self.id = id
        self.title = title
        self.text = text
        self.options = options  # mapping option_text -> handler


# Handlers return next_scene_id or None for special control

def intro(game):
    slow_print("A hush falls across the world of Aether. Old magics stir, and a single whisper reaches your ear: 'Wake.'")
    slow_print("You are a traveler of modest means, but tonight the sky flares with impossible lights.")
    name = input("What do they call you, wanderer? ").strip() or "Arin"
    game.player.name = name
    game.player.journal.append("Began the journey.")
    slow_print(f"Welcome, {name}. This night, choices will be heavier than they look.")
    return "village_gate"


def village_gate(game):
    slow_print("At the village gate, an old soldier argues with the mayor. A carriage smolders by the road.")
    options = ["Inspect the carriage", "Speak with the soldier", "Head into the tavern"]
    choice = prompt(options)
    if choice == options[0]:
        slow_print("You pry open the carriage. Inside, a small crystal glows faintly next to a map with red markings all around. You pocket them both.")
        game.player.add_item(Item("Aether Crystal", "A faintly glowing crystal. Mysterious.", heal=0, power=2))
        game.player.flags["has_crystal"] = True
        return "tavern"
    elif choice == options[1]:
        slow_print("The soldier mumbles of bandits to the north and a 'broken sigil'. He gives you a rusted blade.")
        game.player.add_item(Item("Rusted Blade", "An old blade. Better than bare hands.", power=1))
        return "tavern"
    else:
        return "tavern"


def tavern(game):
    slow_print("Later on, the tavern smells of ale and coal. A bard sings of a queen who vanished into the mist.")
    options = ["Listen to the bard", "Gamble with locals", "Ask about the queen"]
    choice = prompt(options)
    if choice == options[0]:
        slow_print("The song hints at the Shattered Spire beyond the marsh... The bard slips you a map.")
        game.player.flags["has_map"] = True
        return "market"
    elif choice == options[1]:
        outcome = random.random()
        if outcome < 0.4:
            slow_print("Luck smiles. You win 5 gold.")
            game.player.gold += 5
        elif outcome < 0.8:
            slow_print("You lose some coins.")
            game.player.gold = max(0, game.player.gold - 3)
        else:
            slow_print("The brawl that follows turns rowdy; you are tossed out into the night.")
        return "market"
    else:
        slow_print("An old woman leans in and warns: 'Do not wake what sleeps under the Spire.'")
        game.player.flags["warned_about_spire"] = True
        return "market"


def market(game):
    slow_print("At the market, stalls sell trinkets and potions. A cloaked figure meets your gaze.")
    options = ["Follow the cloaked figure", "Buy supplies", "Ignore and continue"]
    choice = prompt(options)
    if choice == options[0]:
        slow_print("The figure gives you a riddle and a key: 'The heart of the tower opens only to the patient.'")
        game.player.add_item(Item("Old Key", "A tarnished key engraved with runes."))
        game.player.flags["met_mysterious"] = True
        return "road"
    elif choice == options[1]:
        slow_print("You buy a healing potion and some jerky.")
        game.player.add_item(Item("Healing Tonic", "Restores 12 HP.", heal=12))
        game.player.gold = max(0, game.player.gold - 4)
        return "road"
    else:
        return "road"


def road(game):
    slow_print("Traveling north, the road narrows into mist. You hear distant howls.")
    if game.player.flags.get("has_map"):
        slow_print("Your map marks a short path through the marsh to the Spire. Do you take it?")
        options = ["Take the marsh path", "Stick to the main road"]
        choice = prompt(options)
        if choice == options[0]:
            if random.random() < 0.6:
                slow_print("Wraiths rise from the bog! Prepare to fight.")
                enemies = [Enemy("Bog Wraith", hp=12, power=4, defense=1, speed=4, xp_drop=20, gold_drop=3)]
                if not combat(game.player, enemies):
                    return "death"
            return "spire_approach"
        else:
            slow_print("The main road is longer but safer. You arrive weary at dusk.")
            return "waystation"
    else:
        slow_print("Without a map, you must choose your direction by instinct.")
        options = ["Venture through the marsh", "Follow main road"]
        choice = prompt(options)
        if choice == options[0]:
            if random.random() < 0.8:
                slow_print("A swamp hound bites! You fight.")
                enemies = [Enemy("Swamp Hound", hp=10, power=3, defense=0, speed=5, xp_drop=15, gold_drop=2)]
                if not combat(game.player, enemies):
                    return "death"
            return "spire_approach"
        else:
            return "waystation"


def waystation(game):
    slow_print("A waystation hosts travelers and a traveling knight in tarnished armor." )
    options = ["Speak with the knight", "Rest for a while", "Steal some food (risky)"]
    choice = prompt(options)
    if choice == options[0]:
        slow_print("The knight tests your resolve with a duel of words and offers you a mission to the Spire.")
        game.player.flags["knight_mission"] = True
        return "spire_approach"
    elif choice == options[1]:
        slow_print("You rest and recover your strength. HP restored.")
        game.player.hp = game.player.max_hp
        return "spire_approach"
    else:
        if random.random() < 0.5:
            slow_print("You are caught! You flee and hurt your pride.")
            game.player.hp = max(1, game.player.hp - 5)
        else:
            slow_print("You get away with food and a sour look from the cook.")
            game.player.gold += 2
        return "spire_approach"


def spire_approach(game):
    slow_print("The Shattered Spire rises like a broken tooth. The air hums.")
    if game.player.flags.get("met_mysterious"):
        slow_print("The key in your pack hums and points you to a hidden stair.")
        options = ["Use the key", "Search the base", "Turn back"]
        choice = prompt(options)
        if choice == options[0]:
            if game.player.flags.get("has_crystal"):
                slow_print("The crystal resonates with the key. A hidden door opens; you descend.")
                return "spire_depths"
            else:
                slow_print("The doorway resists; it requires a heart of Aether.")
                return "spire_outer"
        elif choice == options[1]:
            slow_print("You find sigils and a torn banner — the queen's mark. A memory echoes: 'Protect the heart.'")
            game.player.flags["saw_banner"] = True
            return "spire_outer"
        else:
            slow_print("You cannot ignore the pull. The Spire waits.")
            return "road"
    else:
        slow_print("The Spire seems to watch you. Do you enter?")
        options = ["Enter the Spire", "Return to the road"]
        choice = prompt(options)
        if choice == options[0]:
            return "spire_outer"
        else:
            return "road"


def spire_outer(game):
    slow_print("The outer halls are cracked, strewn with the bones of fallen guardians.")
    if random.random() < 0.7:
        slow_print("An ancient guardian stirs to life!")
        enemies = [Enemy("Stone Warden", hp=20, power=6, defense=3, speed=3, xp_drop=40, gold_drop=5)]
        if not combat(game.player, enemies):
            return "death"
    slow_print("A door sealed by a crystal socket stands before you.")
    if game.player.flags.get("has_crystal"):
        options = ["Place crystal", "Keep crystal", "Search elsewhere"]
        choice = prompt(options)
        if choice == options[0]:
            slow_print("The crystal melts into the socket, and the path deeper opens. The tower's pulse changes.")
            game.player.flags["opened_with_crystal"] = True
            return "spire_depths"
        elif choice == options[1]:
            slow_print("You decide to keep the crystal for now; its warmth hums in your pack.")
            return "spire_depths"
        else:
            return "spire_depths"
    else:
        slow_print("You need something to power the socket. Perhaps the heart of Aether?")
        return "spire_depths"


def spire_depths(game):
    slow_print("The depths are a maze of glowing sigils. The air tastes like old storms.")
    if not game.player.flags.get("opened_with_crystal"):
        slow_print("Without the crystal's song, shadows cling and whispers grow loud.")
    options = ["Follow the glow", "Listen to the whispers", "Search the side chambers"]
    choice = prompt(options)
    if choice == options[0]:
        slow_print("You follow a corridor to a vaulted hall. In the center floats an intricately-shaped relic: the Heart of Aether.")
        if game.player.flags.get("warned_about_spire"):
            slow_print("You remember the warning: some things are better left sleeping.")
        return "heart_choice"
    elif choice == options[1]:
        slow_print("The whispers promise power and revenge. They offer secrets for a cost.")
        game.player.flags["heard_whispers"] = True
        return "heart_choice"
    else:
        slow_print("You find inscriptions hinting the heart responds to compassion, not dominance.")
        game.player.flags["compassion_riddle"] = True
        return "heart_choice"


def heart_choice(game):
    slow_print("The Heart of Aether pulses slowly, light folding in on itself. Its surface shows visions.")
    options = ["Claim the Heart (take its power)", "Seal it (protect it)", "Destroy it"]
    choice = prompt(options)
    if choice == options[0]:
        slow_print("You raise your hands. The Heart leaps into you, burning and shining. Power floods your veins.")
        # buff player
        game.player.power += 5
        game.player.max_hp += 10
        game.player.hp = game.player.max_hp
        game.player.flags["claimed_heart"] = True
        game.player.journal.append("Claimed the Heart of Aether.")
        return "finale"
    elif choice == options[1]:
        slow_print("You cradle the Heart and weave protective sigils. Its light calms and settles deep within the Spire.")
        game.player.flags["sealed_heart"] = True
        game.player.journal.append("Sealed the Heart of Aether.")
        # reward: reputation and subtle magic
        game.player.gain_xp(80)
        return "finale"
    else:
        slow_print("You strike true. The Heart shatters, and a shockwave throws you back into shadow.")
        game.player.flags["destroyed_heart"] = True
        # consequences: corrupted echo
        game.player.power = max(1, game.player.power - 2)
        return "finale"


def finale(game):
    slow_print("Consequences ripple. The Spire groans; the world outside breathes differently.")
    if game.player.flags.get("claimed_heart"):
        slow_print("Power surges through the land. Some call you a savior; others fear what you have become.")
        if game.player.flags.get("heard_whispers"):
            slow_print("Whispers find purchase, and a shadow-queen rises, seeking your stolen heart.")
            enemies = [Enemy("Shadow-Queen", hp=40, power=9, defense=4, speed=6, xp_drop=120, gold_drop=25)]
            if not combat(game.player, enemies):
                return "death"
            slow_print("You defeat the shadow-queen, but the cost lingers in your bones.")
            game.player.journal.append("Vanquished the Shadow-Queen.")
            return "ending_power"
        else:
            slow_print("With careful rule, you mend fractured kingdoms and become a new kind of monarch.")
            game.player.journal.append("Became a ruler with the Heart's power.")
            return "ending_ruler"
    elif game.player.flags.get("sealed_heart"):
        slow_print("The Heart sleeps. The Spire becomes a ward for the ages. You are quietly honored.")
        game.player.journal.append("Sealed the Heart; Aether rests.")
        return "ending_guardian"
    elif game.player.flags.get("destroyed_heart"):
        slow_print("Shards of Aether scatter. The world shifts unpredictably; storms and strange beasts roam.")
        game.player.journal.append("The Heart was destroyed; the world changed.")
        return "ending_wilds"
    else:
        slow_print("You leave the Spire, neither claiming nor sealing. The mystery remains.")
        game.player.journal.append("Left the Heart untouched.")
        return "ending_wanderer"


# ------------------------- Endings (simple descriptions) -------------------------

endings = {
    "ending_power": "You walk the line between savior and tyrant. The world remembers you in both whispers and songs.",
    "ending_ruler": "Under your careful hand, the kingdoms knit back together. Your name is carved into new histories.",
    "ending_guardian": "You fade into legend as the one who kept the balance. Few know your face, but many reap the calm.",
    "ending_wilds": "Chaos blooms. New heroes must rise. You live on as an echo in the wild tales of a changed world.",
    "ending_wanderer": "You carry the memory like a faded map—always incomplete, and therefore beautiful."
}


# ------------------------- Game engine -------------------------

class Game:
    def __init__(self):
        self.player = Player("Hero")
        self.scenes: Dict[str, Scene] = {}
        self.current_scene = "intro"
        self.save_file = "echoes_save.json"
        self.setup_scenes()

    def setup_scenes(self):
        # register scenes linking ids to handler functions
        self.scenes = {
            "intro": Scene("intro", "Prologue", "", intro),
            "village_gate": Scene("village_gate", "Village Gate", "", village_gate),
            "tavern": Scene("tavern", "Tavern", "", tavern),
            "market": Scene("market", "Market", "", market),
            "road": Scene("road", "Road North", "", road),
            "waystation": Scene("waystation", "Waystation", "", waystation),
            "spire_approach": Scene("spire_approach", "Shattered Spire", "", spire_approach),
            "spire_outer": Scene("spire_outer", "Spire Outer", "", spire_outer),
            "spire_depths": Scene("spire_depths", "Spire Depths", "", spire_depths),
            "heart_choice": Scene("heart_choice", "Heart of Aether", "", heart_choice),
            "finale": Scene("finale", "Finale", "", finale),
        }

    def save(self):
        data = {
            "player": self.player.to_dict(),
            "current_scene": self.current_scene,
        }
        with open(self.save_file, "w") as f:
            json.dump(data, f, indent=2)
        slow_print(f"Game saved to {self.save_file}.")

    def load(self):
        if not os.path.exists(self.save_file):
            slow_print("No save file found.")
            return False
        with open(self.save_file, "r") as f:
            data = json.load(f)
        self.player = Player.from_dict(data.get("player", {}))
        self.current_scene = data.get("current_scene", "intro")
        slow_print("Game loaded.")
        return True

    def show_status(self):
        p = self.player
        slow_print(f"{p.name} — Level {p.level} | HP: {p.hp}/{p.max_hp} | Power: {p.power} | Defense: {p.defense} | Gold: {p.gold} | XP: {p.xp}/{p.next_xp}")

    def open_journal(self):
        slow_print("Journal:")
        if not self.player.journal:
            slow_print("(Empty)")
        else:
            for i, entry in enumerate(self.player.journal, 1):
                print(f"{i}. {entry}")
        print()

    def main_loop(self):
        slow_print("--- Echoes of Aether — Text Adventure RPG ---")
        while True:
            print("\n--- MENU ---")
            options = ["Continue", "Status", "Journal", "Save", "Load", "Quit"]
            choice = prompt(options)
            if choice == "Continue":
                current = self.scenes.get(self.current_scene)
                if current is None:
                    slow_print("The path is uncertain...")
                    break
                next_scene = current.options if isinstance(current.options, dict) and current.options else None
                # our scenes use handlers stored in Scene.text as function via the Scene object setup
                # Instead we map by id to functions
                handler = globals().get(current.id)
                if callable(handler):
                    result = handler(self)
                    if result == "death":
                        slow_print("Your story ends here. You may load a previous save or start anew.")
                        return
                    elif result in endings:
                        slow_print(endings[result])
                        slow_print("*** THE END ***")
                        return
                    elif result is None:
                        slow_print("The story stumbles. No path forward.")
                        return
                    else:
                        self.current_scene = result
                else:
                    slow_print("Scene handler missing. The weave frays.")
                    return
            elif choice == "Status":
                self.show_status()
            elif choice == "Journal":
                self.open_journal()
            elif choice == "Save":
                self.save()
            elif choice == "Load":
                self.load()
            elif choice == "Quit":
                slow_print("Farewell, traveler.")
                break


# ------------------------- Entry point -------------------------

if __name__ == '__main__':
    game = Game()
    # allow quick commands
    if len(sys.argv) > 1:
        arg = sys.argv[1].lower()
        if arg == "--load":
            game.load()
        elif arg == "--help":
            print("Usage: python EchoesOfAether.py [--load]")
            sys.exit(0)
    game.main_loop()
