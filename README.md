# Paper Plugin Dev

A comprehensive **Claude Code plugin** that gives Claude deep knowledge of the Minecraft Paper API, Java 21 patterns, and server internals — so it can write production-quality Paper plugins right out of the box.

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Paper 1.21.x](https://img.shields.io/badge/Paper-1.21.x-blue.svg)](https://papermc.io)
[![Java 21](https://img.shields.io/badge/Java-21-orange.svg)](https://openjdk.org/projects/jdk/21/)

**[View Website](https://atomgameraga.github.io/Paper-ClaudePlugin/)**

---

## What Is This?

Paper Plugin Dev is a Claude Code plugin that equips Claude with a curated 2,500+ line reference covering 47 major sections of the Paper API. When installed, Claude stops guessing at deprecated APIs and instead uses correct, modern patterns for Paper 1.21.x and Java 21.

## Features

- **Project Setup & Lifecycle** — Gradle KTS, Maven, paper-plugin.yml, Lifecycle API
- **Core API Systems** — Events, Brigadier commands, Adventure/MiniMessage, Data Components
- **Data & Storage** — PDC, HikariCP, SQLite/MySQL, config management, NBT
- **Gameplay Systems** — Inventory GUIs, entities, recipes, registries, particles, mob AI
- **Advanced & Internals** — NMS access, packet manipulation, anti-cheat patterns, world generation
- **Quality & Patterns** — Performance best practices, design patterns, MockBukkit testing, Sentry
- **Integrations** — PlaceholderAPI, Vault, LuckPerms, ProtocolLib, PacketEvents, GrimAC

## Installation

### Quick Install

```bash
claude install-plugin ATOMGAMERAGA/Paper-ClaudePlugin
```

### Manual Install

```bash
git clone https://github.com/ATOMGAMERAGA/Paper-ClaudePlugin.git
claude --plugin-dir ./Paper-ClaudePlugin
```

## Usage

Once installed, simply ask Claude to build a Paper plugin. The skill activates automatically when your prompt involves Minecraft, Paper, Bukkit, Spigot, or plugin development.

```
You: Create a Paper plugin that adds custom enchantments with the Registry API
```

Claude will use correct Java 21 syntax, modern Paper 1.21.x APIs, and production-ready patterns.

## Plugin Structure

```
Paper-ClaudePlugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   └── paper-plugin-dev/
│       └── SKILL.md             # Comprehensive skill reference (2,500+ lines)
├── docs/                        # GitHub Pages website
│   ├── index.html
│   ├── style.css
│   └── script.js
├── README.md
└── LICENSE
```

## License

MIT License — Copyright (c) 2026 Atom Gamer Arda A.G.A
