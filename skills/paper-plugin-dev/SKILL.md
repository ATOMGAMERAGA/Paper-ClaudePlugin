---
name: paper-plugin-dev
description: >
  Minecraft Paper plugin development skill. Used for plugin coding targeting Java 21 and Paper 1.21.x
  (1.21.1 - 1.21.11), API usage, event handling, command registration, NMS/internals access,
  packet manipulation, performance optimization, and best practices.
  ALWAYS use this skill in the following situations: writing/editing Minecraft plugins, Paper API usage,
  Bukkit/Spigot/Paper events, Brigadier command system, Adventure/MiniMessage components,
  Data Component API, PDC (Persistent Data Container), entity/block/item manipulation,
  inventory GUIs, scheduler/async tasks, packet listening/sending, anti-cheat mechanics,
  NMS (net.minecraft.server) access, paperweight-userdev usage, plugin.yml / paper-plugin.yml
  configuration, Gradle/Maven project setup, ProtocolLib/PacketEvents integration,
  world generation, scoreboard/team API, boss bar, title, action bar, tab list, resource pack,
  database integration, config management, permission system, PlaceholderAPI integration,
  and ANY coding task related to Minecraft server plugins. This skill should trigger when the user
  uses words like "plugin", "addon", "Paper", "Bukkit", "Spigot", "Minecraft", "server mod",
  "server", "SMP", or asks anything in the context of Java and Minecraft development.
---

# Minecraft Paper Plugin Development — Java 21 & Paper 1.21.x

This skill is a comprehensive reference and rule set that enables Claude to write
professional-quality Java 21 plugins for Minecraft Paper servers.

---

## 1. PROJECT SETUP

### 1.1 Gradle (Kotlin DSL) — RECOMMENDED

```kotlin
// settings.gradle.kts
pluginManagement {
    repositories {
        gradlePluginPortal()
        maven("https://repo.papermc.io/repository/maven-public/")
    }
}

// build.gradle.kts
plugins {
    java
    id("com.gradleup.shadow") version "8.3.0"            // Fat JAR (shade)
    id("xyz.jpenilla.run-paper") version "2.3.1"          // Test server
    // If NMS is needed:
    // id("io.papermc.paperweight.userdev") version "2.0.0-beta.19"
}

group = "com.example"
version = "1.0.0"

repositories {
    mavenCentral()
    maven("https://repo.papermc.io/repository/maven-public/")
}

dependencies {
    // API only:
    compileOnly("io.papermc.paper:paper-api:1.21.4-R0.1-SNAPSHOT")
    // If NMS is needed, use instead of API:
    // paperweight.paperDevBundle("1.21.4-R0.1-SNAPSHOT")
}

java {
    toolchain.languageVersion.set(JavaLanguageVersion.of(21))
}

tasks.processResources {
    filteringCharset = "UTF-8"
    filesMatching("plugin.yml") {
        expand("version" to project.version)
    }
}
```

### 1.2 Maven

```xml
<repositories>
    <repository>
        <id>papermc</id>
        <url>https://repo.papermc.io/repository/maven-public/</url>
    </repository>
</repositories>
<dependencies>
    <dependency>
        <groupId>io.papermc.paper</groupId>
        <artifactId>paper-api</artifactId>
        <version>1.21.4-R0.1-SNAPSHOT</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.13.0</version>
            <configuration>
                <release>21</release>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.6.0</version>
        </plugin>
    </plugins>
</build>
```

### 1.3 Important Java 21 Features (Use in Plugins!)

- **Record classes**: For DTO/config data structures `public record PlayerData(UUID uuid, int level) {}`
- **Sealed classes**: `sealed interface Reward permits CoinReward, ItemReward {}`
- **Pattern matching**: `if (entity instanceof Player player) { ... }`
- **Switch expressions**: `var msg = switch(rank) { case ADMIN -> "Administrator"; case MOD -> "Moderator"; default -> "Player"; };`
- **Text blocks**: For multiline strings `"""..."""`
- **Virtual threads** (preview): For heavy I/O operations `Thread.ofVirtual().start(() -> ...)`
- **SequencedCollection**: `list.getFirst()`, `list.getLast()`, `list.reversed()`

---

## 2. PLUGIN ENTRY POINT AND LIFECYCLE

### 2.1 Bukkit Plugin (Classic — plugin.yml)

```java
public final class MyPlugin extends JavaPlugin {

    @Override
    public void onLoad() {
        // Runs BEFORE world loading. Config, dependency checks.
    }

    @Override
    public void onEnable() {
        // When server starts. Event, command, scheduler registrations go HERE.
        saveDefaultConfig();
        getServer().getPluginManager().registerEvents(new MyListener(this), this);
        
        // Brigadier command registration (1.20.6+)
        getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
            final Commands commands = event.registrar();
            // Register commands here
        });
    }

    @Override
    public void onDisable() {
        // When server shuts down. Save data, cleanup.
    }
}
```

### 2.2 plugin.yml

```yaml
name: MyPlugin
version: '${version}'
main: com.example.myplugin.MyPlugin
api-version: '1.21.4'
description: Description
author: AuthorName
website: https://example.com

load: POSTWORLD  # or STARTUP

depend: [Vault, LuckPerms]        # Required dependencies
softdepend: [PlaceholderAPI]       # Optional
loadbefore: [AnotherPlugin]        # This plugin loads first

permissions:
  myplugin.admin:
    description: Admin permission
    default: op
  myplugin.use:
    description: Usage permission
    default: true

# Defining commands in plugin.yml is optional (unnecessary if using Brigadier)
commands:
  mycommand:
    description: Example command
    usage: /<command> [args]
    aliases: [mc, mycmd]
    permission: myplugin.use
```

### 2.3 Paper Plugin (Experimental — paper-plugin.yml)

```yaml
name: MyPaperPlugin
version: '1.0.0'
main: com.example.myplugin.MyPlugin
api-version: '1.21.4'
bootstrapper: com.example.myplugin.MyBootstrap
loader: com.example.myplugin.MyLoader

dependencies:
  server:
    Vault:
      load: BEFORE
      required: true
      join-classpath: true
    PlaceholderAPI:
      load: BEFORE
      required: false
  bootstrap: {}
```

```java
// Paper Plugin Bootstrapper — execute code before server starts
public class MyBootstrap implements PluginBootstrap {
    @Override
    public void bootstrap(BootstrapContext context) {
        // Registry modifications, early initialization
        context.getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
            // Register commands from bootstrapper — usable in datapack functions
        });
    }
}

// Paper Plugin Loader — classpath management
public class MyLoader implements PluginLoader {
    @Override
    public void classloader(PluginClasspathBuilder classpathBuilder) {
        MavenLibraryResolver resolver = new MavenLibraryResolver();
        resolver.addDependency(new Dependency(
            new DefaultArtifact("com.zaxxer:HikariCP:5.1.0"), null));
        resolver.addRepository(new RemoteRepository.Builder(
            "central", "default",
            MavenLibraryResolver.MAVEN_CENTRAL_DEFAULT_MIRROR).build());
        classpathBuilder.addLibrary(resolver);
    }
}
```

**Paper plugin vs Bukkit plugin differences:**
- Paper plugins enforce classloading isolation (you can't access other plugins' classes)
- You can bypass isolation with `join-classpath: true`
- Command registrations are not done in `paper-plugin.yml`, use Brigadier API
- Bukkit serialization system is supported but `ConfigurationSerialization.registerClass()` must be called manually

---

## 3. EVENT SYSTEM

### 3.1 Event Listener Registration

```java
public class MyListener implements Listener {

    private final MyPlugin plugin;

    public MyListener(MyPlugin plugin) {
        this.plugin = plugin;
    }

    // Basic event listening
    @EventHandler
    public void onJoin(PlayerJoinEvent event) {
        event.joinMessage(Component.text("Welcome ")
            .append(event.getPlayer().displayName())
            .color(NamedTextColor.GREEN));
    }

    // Priority and cancellation control
    @EventHandler(priority = EventPriority.HIGH, ignoreCancelled = true)
    public void onBreak(BlockBreakEvent event) {
        // ignoreCancelled = true → this handler won't run if already cancelled
    }

    // Paper-specific events
    @EventHandler
    public void onAsync(AsyncChatEvent event) {
        // Paper's async chat event — NEVER call Bukkit API here!
        Component message = event.message();
        // Message format can be changed with renderer
        event.renderer(ChatRenderer.viewerUnaware((source, sourceDisplayName, msg) ->
            sourceDisplayName.append(Component.text(" > ")).append(msg)
        ));
    }
}
```

### 3.2 Event Priority Order (EventPriority)

```
LOWEST → LOW → NORMAL (default) → HIGH → HIGHEST → MONITOR
```

- `MONITOR`: For monitoring only. Do NOT cancel or modify events.
- Cancellation logic: Higher priority handlers can override lower priority decisions.
- Add `ignoreCancelled = true` to avoid unnecessarily processing cancelled events.

### 3.3 Important Paper-Specific Events

```java
// Async — NOT on the main thread, do NOT call Bukkit API!
AsyncChatEvent              // Chat message (Paper)
AsyncPlayerPreLoginEvent    // Before login (IP ban, whitelist)

// Server
ServerLoadEvent             // When server is fully loaded
WhitelistStateUpdateEvent   // Whitelist change

// Player
PlayerArmorChangeEvent      // Armor change (Paper)
PlayerBedFailEnterEvent     // Failed bed entry
PlayerDeathEvent            // Death (getDeathMessage() → Component)
PlayerMoveEvent             // Movement (called very frequently — be careful!)
PlayerInteractEvent         // Right/left click
PlayerInteractEntityEvent   // Entity click
PlayerItemConsumeEvent      // Eating/drinking

// Block
BlockBreakEvent
BlockPlaceEvent
BlockPhysicsEvent           // Redstone, falling sand, etc.
BlockFromToEvent            // Water/lava flow

// Entity
EntityDamageByEntityEvent   // PvP/PvE damage
EntityDeathEvent
CreatureSpawnEvent          // SpawnReason check
ProjectileHitEvent          // Arrow/trident/snowball hit

// Inventory
InventoryClickEvent
InventoryCloseEvent
InventoryDragEvent
PrepareItemCraftEvent       // Crafting preview

// Paper extras
PlayerArmSwingEvent
EntityMoveEvent             // Entity movement (performance-costly)
PlayerChunkLoadEvent / PlayerChunkUnloadEvent
BeaconActivatedEvent / BeaconDeactivatedEvent
```

### 3.4 Creating Custom Events

```java
public class PlayerLevelUpEvent extends Event implements Cancellable {
    private static final HandlerList HANDLER_LIST = new HandlerList();
    private final Player player;
    private final int newLevel;
    private boolean cancelled;

    public PlayerLevelUpEvent(Player player, int newLevel) {
        this.player = player;
        this.newLevel = newLevel;
    }

    public Player getPlayer() { return player; }
    public int getNewLevel() { return newLevel; }

    @Override public boolean isCancelled() { return cancelled; }
    @Override public void setCancelled(boolean cancel) { this.cancelled = cancel; }

    @Override public HandlerList getHandlers() { return HANDLER_LIST; }
    public static HandlerList getHandlerList() { return HANDLER_LIST; }
}

// Usage:
PlayerLevelUpEvent event = new PlayerLevelUpEvent(player, 10);
Bukkit.getPluginManager().callEvent(event);
if (!event.isCancelled()) {
    // Perform level up
}
```

---

## 4. COMMAND SYSTEM (Brigadier)

### 4.1 Brigadier Command Registration (Paper 1.20.6+)

```java
// In onEnable() or PluginBootstrap:
getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
    final Commands commands = event.registrar();

    // Simple command
    commands.register(
        Commands.literal("heal")
            .requires(src -> src.getSender().hasPermission("myplugin.heal"))
            .executes(ctx -> {
                if (ctx.getSource().getSender() instanceof Player player) {
                    player.setHealth(player.getMaxHealth());
                    player.sendMessage(Component.text("You have been healed!", NamedTextColor.GREEN));
                }
                return Command.SINGLE_SUCCESS;
            })
            .build(),
        "Heals the player",              // Description
        List.of("h", "restore")          // Aliases
    );

    // Command with arguments
    commands.register(
        Commands.literal("give-xp")
            .requires(src -> src.getSender().hasPermission("myplugin.givexp"))
            .then(Commands.argument("target", ArgumentTypes.player())
                .then(Commands.argument("amount", IntegerArgumentType.integer(1, 10000))
                    .executes(ctx -> {
                        Player target = ctx.getArgument("target", PlayerSelectorArgumentResolver.class)
                            .resolve(ctx.getSource()).getFirst();
                        int amount = IntegerArgumentType.getInteger(ctx, "amount");
                        target.giveExp(amount);
                        ctx.getSource().getSender().sendMessage(
                            Component.text("Gave " + amount + " XP to " + target.getName() + "."));
                        return Command.SINGLE_SUCCESS;
                    })
                )
            )
            .build(),
        "Gives XP",
        List.of("xp")
    );
});
```

### 4.2 BasicCommand (Simple Alternative)

```java
commands.register("ping", "Shows latency", List.of("latency"),
    new BasicCommand() {
        @Override
        public void execute(CommandSourceStack source, String[] args) {
            if (source.getSender() instanceof Player player) {
                player.sendMessage(Component.text("Latency: " + player.getPing() + "ms"));
            }
        }

        @Override
        public Collection<String> suggest(CommandSourceStack source, String[] args) {
            return List.of(); // Tab-complete suggestions
        }
    }
);
```

### 4.3 Brigadier Argument Types

```java
// Vanilla Minecraft arguments
ArgumentTypes.player()            // Single player selector
ArgumentTypes.players()           // Multiple player selector
ArgumentTypes.entity()            // Single entity
ArgumentTypes.entities()          // Multiple entities
ArgumentTypes.blockState()        // Block state
ArgumentTypes.itemStack()         // Item stack
ArgumentTypes.resource(RegistryKey.ENCHANTMENT)  // Registry resource
ArgumentTypes.namespacedKey()     // NamespacedKey

// Brigadier basic arguments
IntegerArgumentType.integer()
IntegerArgumentType.integer(min, max)
DoubleArgumentType.doubleArg()
FloatArgumentType.floatArg()
BoolArgumentType.bool()
StringArgumentType.word()         // Single word
StringArgumentType.string()       // Quoted string
StringArgumentType.greedyString() // Rest of the line

// Paper-specific
ArgumentTypes.signedMessage()     // Signed message (chat reporting)
ArgumentTypes.component()         // Component (JSON)
ArgumentTypes.key()               // Key (namespace:value)
ArgumentTypes.finePosition()      // Fine position (double x, y, z)
ArgumentTypes.blockPosition()     // Block position (int x, y, z)
```

---

## 5. ADVENTURE COMPONENT API & MINIMESSAGE

### 5.1 Creating Components

```java
// Basic text
Component text = Component.text("Hello World!", NamedTextColor.GOLD, TextDecoration.BOLD);

// Chaining
Component msg = Component.text()
    .content("Welcome to the server! ")
    .color(NamedTextColor.GREEN)
    .append(Component.text("[CLICK]", NamedTextColor.YELLOW, TextDecoration.BOLD)
        .clickEvent(ClickEvent.runCommand("/spawn"))
        .hoverEvent(HoverEvent.showText(Component.text("Teleport to spawn"))))
    .build();

// Translatable (multilingual)
Component translatable = Component.translatable("death.attack.player", 
    victim.displayName(), killer.displayName());

// Keybind
Component keybind = Component.keybind("key.inventory"); // Shows the player's inventory key

// Gradient (with MiniMessage)
Component gradient = MiniMessage.miniMessage().deserialize(
    "<gradient:gold:red>This text has a gradient!</gradient>");
```

### 5.2 MiniMessage (String ↔ Component)

```java
MiniMessage mm = MiniMessage.miniMessage();

// Parse
Component parsed = mm.deserialize(
    "<bold><red>WARNING:</red></bold> <gray>You cannot enter this region!");

// Parse with placeholders
Component msg = mm.deserialize(
    "<green>Welcome <player>! Level: <level>",
    Placeholder.component("player", player.displayName()),
    Placeholder.unparsed("level", String.valueOf(playerLevel))
);

// Custom tag with TagResolver
TagResolver resolver = TagResolver.resolver("rank",
    Tag.selfClosingInserting(Component.text("[VIP]", NamedTextColor.GOLD)));
Component result = mm.deserialize("<rank> <green>Hello!", resolver);

// Serialize (Component → String)
String serialized = mm.serialize(someComponent);

// sendRichMessage shortcut (Paper 1.19.4+)
player.sendRichMessage("<rainbow>Rainbow message!");
player.sendRichMessage("Hello <n>!", Placeholder.unparsed("name", player.getName()));
```

### 5.3 MiniMessage Tag Reference

```
Colors:       <red>, <#FF5555>, <color:#FF5555>, <color:red>
Decorations:  <bold>, <italic>, <underlined>, <strikethrough>, <obfuscated>
Close deco:   </bold> or <!bold>
Gradient:     <gradient:red:blue>, <gradient:#FF0000:#0000FF>
Rainbow:      <rainbow>, <rainbow:phase>
Hover:        <hover:show_text:'Hover message'>text</hover>
Click:        <click:run_command:'/spawn'>Click</click>
              <click:open_url:'https://...'>Link</click>
              <click:suggest_command:'/msg '>Type</click>
              <click:copy_to_clipboard:'text'>Copy</click>
Insert:       <insert:text>Shift+click</insert>
Keybind:      <key:key.inventory>
Translatable: <lang:block.minecraft.diamond_block>
Selector:     <sel:@p>
Font:         <font:uniform>text</font>
Reset:        <reset>
Newline:      <newline> or <br>
Transition:   <transition:red:blue:0.5>
```

### 5.4 Message Sending Methods

```java
// Chat message
player.sendMessage(component);
player.sendRichMessage("<green>Success!");

// Action bar
player.sendActionBar(Component.text("⚔ Combat mode active!", NamedTextColor.RED));

// Title + Subtitle
player.showTitle(Title.title(
    Component.text("LEVEL UP!", NamedTextColor.GOLD),
    Component.text("You reached level 10!", NamedTextColor.YELLOW),
    Title.Times.times(Duration.ofMillis(500), Duration.ofSeconds(3), Duration.ofMillis(500))
));

// Boss bar
BossBar bossBar = BossBar.bossBar(
    Component.text("Event: Boss Fight", NamedTextColor.RED),
    0.75f,                    // Progress (0.0 - 1.0)
    BossBar.Color.RED,
    BossBar.Overlay.PROGRESS
);
player.showBossBar(bossBar);
// Update progress: bossBar.progress(0.5f);
// Remove: player.hideBossBar(bossBar);

// Tab list header/footer
player.sendPlayerListHeaderAndFooter(
    Component.text("✦ ServerName ✦", NamedTextColor.GOLD),
    Component.text("Players online: " + Bukkit.getOnlinePlayers().size())
);

// Scoreboard (sidebar)
// Uses Bukkit Scoreboard API — detailed below

// Book
ItemStack book = ItemStack.of(Material.WRITTEN_BOOK);
book.setData(DataComponentTypes.WRITTEN_BOOK_CONTENT,
    WrittenBookContent.writtenBookContent("Guide", "Server")
        .addPage(Component.text("Page 1 content"))
        .addPage(Component.text("Page 2 content"))
        .build());
player.openBook(book);
```

---

## 6. DATA COMPONENT API (1.21.x, Experimental)

Paper 1.21+ provides direct access to Vanilla data components of ItemStacks.
Difference from ItemMeta: access to prototype (default) values, component removal, more performant.

```java
ItemStack sword = ItemStack.of(Material.DIAMOND_SWORD);

// Reading
Integer maxDamage = sword.getData(DataComponentTypes.MAX_DAMAGE);
boolean hasTool = sword.hasData(DataComponentTypes.TOOL);
// Read prototype (default) value
int defaultDurability = Material.DIAMOND_SWORD.getDefaultData(DataComponentTypes.MAX_DAMAGE);

// Writing
sword.setData(DataComponentTypes.CUSTOM_NAME, Component.text("Dragon Slayer", NamedTextColor.LIGHT_PURPLE));
sword.setData(DataComponentTypes.LORE, ItemLore.lore()
    .addLine(Component.text("Legendary sword", NamedTextColor.GRAY).decoration(TextDecoration.ITALIC, false))
    .build());
sword.setData(DataComponentTypes.ENCHANTMENTS, ItemEnchantments.itemEnchantments()
    .add(Enchantment.SHARPNESS, 5)
    .add(Enchantment.UNBREAKING, 3)
    .showInTooltip(true)
    .build());
sword.setData(DataComponentTypes.RARITY, ItemRarity.EPIC);
sword.setData(DataComponentTypes.ENCHANTMENT_GLINT_OVERRIDE, true); // Glint
sword.setData(DataComponentTypes.MAX_DAMAGE, 2000);
sword.setData(DataComponentTypes.CUSTOM_MODEL_DATA, CustomModelData.customModelData()
    .addFloat(0.5f).addFlag(true).build());

// Removing component (even from prototype)
sword.unsetData(DataComponentTypes.TOOL); // No longer a tool
// Reset to prototype
sword.resetData(DataComponentTypes.MAX_STACK_SIZE);

// Non-valued (flag) components
sword.setData(DataComponentTypes.GLIDER);        // Glide like elytra
sword.unsetData(DataComponentTypes.GLIDER);

// Builder pattern (complex components)
Equippable.Builder builder = sword.getData(DataComponentTypes.EQUIPPABLE).toBuilder();
builder.equipSound(SoundEventKeys.ENTITY_GHAST_HURT);
sword.setData(DataComponentTypes.EQUIPPABLE, builder);

// WrittenBookContent
ItemStack book = ItemStack.of(Material.WRITTEN_BOOK);
book.setData(DataComponentTypes.WRITTEN_BOOK_CONTENT,
    WrittenBookContent.writtenBookContent("Book", "Author")
        .addPage(Component.text("Page 1"))
        .generation(0)
        .build());
```

**Important DataComponentTypes:**
`CUSTOM_NAME`, `LORE`, `ENCHANTMENTS`, `DAMAGE`, `MAX_DAMAGE`, `MAX_STACK_SIZE`,
`RARITY`, `UNBREAKABLE`, `CUSTOM_MODEL_DATA`, `DYED_COLOR`, `TOOL`, `FOOD`,
`POTION_CONTENTS`, `ATTRIBUTE_MODIFIERS`, `ENCHANTMENT_GLINT_OVERRIDE`, `FIRE_RESISTANT`,
`HIDE_TOOLTIP`, `WRITTEN_BOOK_CONTENT`, `EQUIPPABLE`, `GLIDER`, `CONTAINER`,
`BANNER_PATTERNS`, `CHARGED_PROJECTILES`, `PROFILE` (skull), `CONSUMABLE`, `USE_COOLDOWN`

---

## 7. PERSISTENT DATA CONTAINER (PDC)

PDC is used to store custom data on items/entities/block entities. Much safer than
direct NBT access and stable across versions.

```java
// NamespacedKey (should be kept reusable!)
private static final NamespacedKey SOUL_KEY = new NamespacedKey(plugin, "soul-count");
private static final NamespacedKey OWNER_KEY = new NamespacedKey(plugin, "owner-uuid");

// === ITEM PDC ===
// 1.21.4+ direct edit:
stack.editPersistentDataContainer(pdc -> {
    pdc.set(SOUL_KEY, PersistentDataType.INTEGER, 42);
    pdc.set(OWNER_KEY, PersistentDataType.STRING, player.getUniqueId().toString());
});
// 1.21.1+ read-only (without creating a snapshot):
Integer souls = stack.getPersistentDataContainer().get(SOUL_KEY, PersistentDataType.INTEGER);
// Old method (still works):
ItemMeta meta = stack.getItemMeta();
meta.getPersistentDataContainer().set(SOUL_KEY, PersistentDataType.INTEGER, 42);
stack.setItemMeta(meta);

// === ENTITY PDC ===
entity.getPersistentDataContainer().set(SOUL_KEY, PersistentDataType.INTEGER, 10);
Integer val = entity.getPersistentDataContainer().get(SOUL_KEY, PersistentDataType.INTEGER);

// === BLOCK ENTITY PDC ===
Block block = ...;
if (block.getState() instanceof Chest chest) {
    chest.getPersistentDataContainer().set(SOUL_KEY, PersistentDataType.STRING, "locked");
    chest.update(); // update() is REQUIRED for block entities!
}

// === PLAYER PDC ===
player.getPersistentDataContainer().set(SOUL_KEY, PersistentDataType.INTEGER, 100);
// OfflinePlayer only offers read-only PDC

// Check & delete
boolean has = pdc.has(SOUL_KEY, PersistentDataType.INTEGER);
boolean hasAny = pdc.has(SOUL_KEY); // 1.20.5+ type-agnostic
pdc.remove(SOUL_KEY);
int valOrDefault = pdc.getOrDefault(SOUL_KEY, PersistentDataType.INTEGER, 0);

// Supported types:
// BYTE, SHORT, INTEGER, LONG, FLOAT, DOUBLE, STRING, BYTE_ARRAY,
// INTEGER_ARRAY, LONG_ARRAY, BOOLEAN, TAG_CONTAINER (nested PDC),
// LIST (PersistentDataType.LIST.of(PersistentDataType.STRING))

// Nested PDC (Tag Container)
PersistentDataContainer inner = pdc.getAdapterContext().newPersistentDataContainer();
inner.set(new NamespacedKey(plugin, "x"), PersistentDataType.DOUBLE, 100.5);
inner.set(new NamespacedKey(plugin, "y"), PersistentDataType.DOUBLE, 64.0);
pdc.set(new NamespacedKey(plugin, "home"), PersistentDataType.TAG_CONTAINER, inner);

// List (1.20.5+)
pdc.set(new NamespacedKey(plugin, "friends"),
    PersistentDataType.LIST.strings(), List.of("Alice", "Bob"));
```

**CAUTION:** PDC data is NOT automatically copied between holders! An ItemStack's PDC
placed as a block does not transfer to the block entity — you must copy it manually.

---

## 8. INVENTORY & GUI

### 8.1 Custom Inventory GUI

```java
public class ShopGUI implements InventoryHolder {
    private final Inventory inventory;

    public ShopGUI() {
        // Must be a multiple of 9 (9, 18, 27, 36, 45, 54)
        this.inventory = Bukkit.createInventory(this, 54, 
            Component.text("✦ Shop", NamedTextColor.DARK_PURPLE));
        setupItems();
    }

    private void setupItems() {
        // Glass pane filler
        ItemStack filler = ItemStack.of(Material.BLACK_STAINED_GLASS_PANE);
        filler.editMeta(meta -> meta.displayName(Component.empty()));
        for (int i = 0; i < 54; i++) inventory.setItem(i, filler);

        // Item for sale
        ItemStack diamond = ItemStack.of(Material.DIAMOND);
        diamond.editMeta(meta -> {
            meta.displayName(Component.text("Diamond", NamedTextColor.AQUA)
                .decoration(TextDecoration.ITALIC, false));
            meta.lore(List.of(
                Component.text("Price: 100 Coins", NamedTextColor.GOLD)
                    .decoration(TextDecoration.ITALIC, false),
                Component.empty(),
                Component.text("» Left click — Buy", NamedTextColor.YELLOW)
                    .decoration(TextDecoration.ITALIC, false)
            ));
        });
        // Add ID via PDC
        diamond.editPersistentDataContainer(pdc ->
            pdc.set(new NamespacedKey(plugin, "shop-item"), PersistentDataType.STRING, "diamond_1"));
        inventory.setItem(22, diamond);
    }

    @Override
    public Inventory getInventory() { return inventory; }
}

// Opening:
player.openInventory(new ShopGUI().getInventory());

// Click handler:
@EventHandler
public void onClick(InventoryClickEvent event) {
    if (!(event.getInventory().getHolder() instanceof ShopGUI)) return;
    event.setCancelled(true); // Always cancel so items can't be taken

    if (event.getCurrentItem() == null) return;
    if (!(event.getWhoClicked() instanceof Player player)) return;

    ItemStack clicked = event.getCurrentItem();
    var pdc = clicked.getPersistentDataContainer();
    String itemId = pdc.get(new NamespacedKey(plugin, "shop-item"), PersistentDataType.STRING);
    if (itemId == null) return;

    // Purchase logic...
    player.sendRichMessage("<green>Purchased!");
}

// Close handler:
@EventHandler
public void onClose(InventoryCloseEvent event) {
    if (event.getInventory().getHolder() instanceof ShopGUI) {
        // Cleanup, save data, etc.
    }
}
```

### 8.2 Menu Type API (1.21.2+, Experimental)

```java
// More type-safe approach
MenuType.GENERIC_9X6.create(Component.text("Inventory")).open(player);
```

---

## 9. SCHEDULER & ASYNC OPERATIONS

### 9.1 BukkitScheduler

```java
// Run on the next tick (1 tick = 50ms, 20 ticks = 1 second)
Bukkit.getScheduler().runTask(plugin, () -> {
    player.teleport(location); // Bukkit API calls MUST be on the main thread
});

// Delayed (40 ticks = 2 seconds later)
Bukkit.getScheduler().runTaskLater(plugin, () -> {
    player.sendMessage(Component.text("2 seconds have passed!"));
}, 40L);

// Repeating (every 20 ticks = 1 second)
BukkitTask task = Bukkit.getScheduler().runTaskTimer(plugin, () -> {
    // Runs every second
}, 0L, 20L);
// Stop: task.cancel();

// ASYNC — For heavy I/O, database, HTTP requests
Bukkit.getScheduler().runTaskAsynchronously(plugin, () -> {
    // Database query (async thread)
    String data = database.query("...");
    
    // Send result back to the main thread
    Bukkit.getScheduler().runTask(plugin, () -> {
        player.sendMessage(Component.text("Data: " + data));
    });
});

// Async repeating
Bukkit.getScheduler().runTaskTimerAsynchronously(plugin, () -> {
    // Periodic async work
}, 0L, 20L * 60); // Every minute
```

### 9.2 Folia Support (Optional)

```java
// Folia region-based scheduling
if (Bukkit.getServer().getName().contains("Folia")) {
    // Folia global scheduler:
    Bukkit.getGlobalRegionScheduler().run(plugin, task -> { ... });
    // In entity's region:
    entity.getScheduler().run(plugin, task -> { ... }, null);
    // Async:
    Bukkit.getAsyncScheduler().runNow(plugin, task -> { ... });
}
```

### 9.3 IMPORTANT THREADING RULES

```
╔══════════════════════════════════════════════════════════════╗
║  MAIN THREAD (Tick Thread):                                  ║
║  • Bukkit/Paper API calls (getPlayer, teleport, etc.)       ║
║  • World changes (block place/break)                         ║
║  • Entity spawn/despawn                                      ║
║  • Inventory changes                                         ║
║  • Event handling                                            ║
╠══════════════════════════════════════════════════════════════╣
║  ASYNC THREAD:                                               ║
║  • Database queries (MySQL, SQLite, Redis)                   ║
║  • HTTP/API requests                                         ║
║  • File I/O (large file read/write)                          ║
║  • Heavy computations                                        ║
║  • NEVER call Bukkit API!                                    ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 10. DATABASE INTEGRATION

### 10.1 HikariCP + SQLite/MySQL

```java
// In plugin.yml libraries section (auto-download):
// libraries:
//   - com.zaxxer:HikariCP:5.1.0

private HikariDataSource dataSource;

public void setupDatabase() {
    HikariConfig config = new HikariConfig();
    // SQLite:
    config.setJdbcUrl("jdbc:sqlite:" + getDataFolder() + "/data.db");
    // MySQL:
    // config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
    // config.setUsername("root");
    // config.setPassword("pass");
    config.setMaximumPoolSize(10);
    config.setMinimumIdle(2);
    config.setConnectionTimeout(5000);
    config.setMaxLifetime(600000);
    dataSource = new HikariDataSource(config);

    // Create table
    try (Connection conn = dataSource.getConnection();
         Statement stmt = conn.createStatement()) {
        stmt.execute("""
            CREATE TABLE IF NOT EXISTS player_data (
                uuid VARCHAR(36) PRIMARY KEY,
                coins INTEGER DEFAULT 0,
                level INTEGER DEFAULT 1,
                last_seen BIGINT
            )
            """);
    } catch (SQLException e) {
        getLogger().severe("Database error: " + e.getMessage());
    }
}

// Async data reading
public CompletableFuture<Integer> getCoins(UUID uuid) {
    return CompletableFuture.supplyAsync(() -> {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "SELECT coins FROM player_data WHERE uuid = ?")) {
            ps.setString(1, uuid.toString());
            ResultSet rs = ps.executeQuery();
            return rs.next() ? rs.getInt("coins") : 0;
        } catch (SQLException e) {
            throw new CompletionException(e);
        }
    });
}

// Usage:
getCoins(player.getUniqueId()).thenAcceptAsync(coins -> {
    Bukkit.getScheduler().runTask(plugin, () -> {
        player.sendMessage(Component.text("Your coins: " + coins));
    });
});
```

---

## 11. CONFIG MANAGEMENT

```java
// Create/load config.yml
@Override
public void onEnable() {
    saveDefaultConfig(); // resources/config.yml → plugins/MyPlugin/config.yml

    // Reading
    String prefix = getConfig().getString("prefix", "[Server]");
    int maxHomes = getConfig().getInt("max-homes", 3);
    List<String> worlds = getConfig().getStringList("disabled-worlds");
    boolean enabled = getConfig().getBoolean("features.pvp-enabled", true);

    // Writing
    getConfig().set("stats.total-joins", 100);
    saveConfig();

    // Reload
    reloadConfig();
}
```

```yaml
# resources/config.yml
prefix: "<gradient:gold:yellow>✦ ServerName</gradient>"
max-homes: 5
disabled-worlds:
  - world_nether
  - world_the_end
features:
  pvp-enabled: true
  keep-inventory: false
database:
  type: sqlite  # sqlite or mysql
  host: localhost
  port: 3306
  name: mydb
  username: root
  password: ""
messages:
  no-permission: "<red>You don't have permission for this command!"
  player-only: "<red>This command can only be used by players."
```

---

## 12. ENTITY, BLOCK, WORLD API

### 12.1 Entity Operations

```java
// Spawn
Wolf wolf = (Wolf) world.spawnEntity(location, EntityType.WOLF);
wolf.setOwner(player);
wolf.customName(Component.text("Rex", NamedTextColor.GOLD));
wolf.setCustomNameVisible(true);
wolf.getPersistentDataContainer().set(key, PersistentDataType.STRING, "pet");

// Spawn with Consumer (safer)
world.spawn(location, Zombie.class, zombie -> {
    zombie.customName(Component.text("Boss Zombie"));
    zombie.setHealth(100);
    zombie.getAttribute(Attribute.MAX_HEALTH).setBaseValue(100);
    zombie.setRemoveWhenFarAway(false);
    zombie.addPotionEffect(new PotionEffect(PotionEffectType.SPEED, -1, 2));
});

// Display Entity (1.19.4+) — decorative, client-side render
world.spawn(location, TextDisplay.class, display -> {
    display.text(Component.text("Hello!", NamedTextColor.GOLD));
    display.setBillboard(Display.Billboard.CENTER); // Faces the player
    display.setBackgroundColor(Color.fromARGB(128, 0, 0, 0)); // Semi-transparent background
});

world.spawn(location, ItemDisplay.class, display -> {
    display.setItemStack(new ItemStack(Material.DIAMOND_SWORD));
    display.setTransformation(new Transformation(
        new Vector3f(0, 1, 0), new AxisAngle4f(), new Vector3f(2, 2, 2), new AxisAngle4f()));
});

// Teleportation
entity.teleport(newLocation);
// Async teleport (Paper)
entity.teleportAsync(newLocation).thenAccept(success -> {
    if (success) player.sendMessage(Component.text("Teleported!"));
});
```

### 12.2 Block Operations

```java
Block block = world.getBlockAt(x, y, z);
block.setType(Material.DIAMOND_BLOCK);

// Detailed control with BlockData
if (block.getBlockData() instanceof Stairs stairs) {
    stairs.setFacing(BlockFace.NORTH);
    stairs.setHalf(Bisected.Half.TOP);
    block.setBlockData(stairs);
}

// Chunk operations
Chunk chunk = location.getChunk();
chunk.load();       // Load
chunk.isLoaded();   // Is loaded?
// Paper: async chunk loading
world.getChunkAtAsync(x, z).thenAccept(loadedChunk -> {
    // Chunk loaded
});
```

### 12.3 World Operations

```java
World world = Bukkit.getWorld("world");
world.setTime(6000);       // Noon
world.setStorm(false);     // Stop rain
world.setGameRule(GameRule.KEEP_INVENTORY, true);
world.setDifficulty(Difficulty.HARD);

// Explosion
world.createExplosion(location, 4.0f, false, false); // fire=false, breakBlocks=false

// Sound
world.playSound(location, Sound.ENTITY_ENDER_DRAGON_GROWL, 
    SoundCategory.MASTER, 1.0f, 1.0f);

// Particle
world.spawnParticle(Particle.FLAME, location, 50, 0.5, 0.5, 0.5, 0.02);
// Paper advanced particle API:
player.spawnParticle(Particle.DUST, location, 1,
    new Particle.DustOptions(Color.RED, 2.0f));
```

---

## 13. SCOREBOARD & TEAM API

```java
// Sidebar scoreboard
Scoreboard scoreboard = Bukkit.getScoreboardManager().getNewScoreboard();
Objective obj = scoreboard.registerNewObjective("sidebar", Criteria.DUMMY,
    Component.text("✦ SERVER NAME ✦", NamedTextColor.GOLD, TextDecoration.BOLD));
obj.setDisplaySlot(DisplaySlot.SIDEBAR);

// Lines (score = order, higher = top)
obj.getScore("§a» Coins: §f100").setScore(5);
obj.getScore("§e» Level: §f10").setScore(4);
obj.getScore("§7─────────").setScore(3);
obj.getScore("§b» Online: §f" + Bukkit.getOnlinePlayers().size()).setScore(2);
obj.getScore("§dplay.server.net").setScore(1);

player.setScoreboard(scoreboard);

// Team (tab list sorting, collision, friendly fire)
Team team = scoreboard.registerNewTeam("vip");
team.displayName(Component.text("VIP", NamedTextColor.GOLD));
team.prefix(Component.text("[VIP] ", NamedTextColor.GOLD));
team.suffix(Component.text(" ✦", NamedTextColor.GOLD));
team.color(NamedTextColor.GOLD);
team.setOption(Team.Option.COLLISION_RULE, Team.OptionStatus.NEVER);
team.setAllowFriendlyFire(false);
team.addPlayer(player);
```

---

## 14. PACKET & PROTOCOL (NMS / ProtocolLib / PacketEvents)

### 14.1 NMS Access (with paperweight-userdev)

```java
// build.gradle.kts
plugins {
    id("io.papermc.paperweight.userdev") version "2.0.0-beta.19"
}
dependencies {
    paperweight.paperDevBundle("1.21.4-R0.1-SNAPSHOT")
}

// Usage — Mojang-mapped names (1.20.5+)
import net.minecraft.server.level.ServerPlayer;
import net.minecraft.network.protocol.game.*;
import org.bukkit.craftbukkit.entity.CraftPlayer;

ServerPlayer nmsPlayer = ((CraftPlayer) player).getHandle();
// Packet sending example:
nmsPlayer.connection.send(new ClientboundSetTitleTextPacket(
    net.minecraft.network.chat.Component.literal("NMS Title!")));
```

### 14.2 PacketEvents Library (Recommended)

```xml
<!-- Maven -->
<dependency>
    <groupId>com.github.retrooper</groupId>
    <artifactId>packetevents-spigot</artifactId>
    <version>2.7.0</version>
    <scope>provided</scope>
</dependency>
```

```java
// Packet listening
PacketEvents.getAPI().getEventManager().registerListener(
    new PacketListenerAbstract() {
        @Override
        public void onPacketReceive(PacketReceiveEvent event) {
            if (event.getPacketType() == PacketType.Play.Client.PLAYER_POSITION) {
                WrapperPlayClientPlayerPosition packet = 
                    new WrapperPlayClientPlayerPosition(event);
                double x = packet.getLocation().getX();
                double y = packet.getLocation().getY();
                double z = packet.getLocation().getZ();
                boolean onGround = packet.isOnGround();
                // Anti-cheat logic...
            }
        }

        @Override
        public void onPacketSend(PacketSendEvent event) {
            // Packets going from server to client
        }
    }
);
```

### 14.3 Plugin Messaging (BungeeCord/Velocity Channel)

```java
// Registration
getServer().getMessenger().registerOutgoingPluginChannel(this, "BungeeCord");
getServer().getMessenger().registerIncomingPluginChannel(this, "BungeeCord", 
    (channel, player, message) -> {
        ByteArrayDataInput in = ByteStreams.newDataInput(message);
        String subchannel = in.readUTF();
        // ...
    });

// Sending (redirect player to another server)
ByteArrayDataOutput out = ByteStreams.newDataOutput();
out.writeUTF("Connect");
out.writeUTF("lobby");
player.sendPluginMessage(plugin, "BungeeCord", out.toByteArray());
```

---

## 15. RECIPE API

```java
// Shaped recipe
NamespacedKey key = new NamespacedKey(plugin, "super_pickaxe");
ShapedRecipe recipe = new ShapedRecipe(key, superPickaxe);
recipe.shape("DDD", " S ", " S ");
recipe.setIngredient('D', Material.DIAMOND_BLOCK);
recipe.setIngredient('S', Material.STICK);
Bukkit.addRecipe(recipe);

// Shapeless recipe
ShapelessRecipe shapeless = new ShapelessRecipe(
    new NamespacedKey(plugin, "compressed_diamond"), compressedDiamond);
shapeless.addIngredient(9, Material.DIAMOND);
Bukkit.addRecipe(shapeless);

// Furnace recipe
FurnaceRecipe furnace = new FurnaceRecipe(
    new NamespacedKey(plugin, "custom_smelt"), result,
    Material.RAW_IRON, 0.7f, 200); // xp, cookTime (ticks)
Bukkit.addRecipe(furnace);

// Smithing recipe (1.20+)
SmithingTransformRecipe smithing = new SmithingTransformRecipe(
    new NamespacedKey(plugin, "custom_upgrade"),
    result,
    new RecipeChoice.MaterialChoice(Material.NETHERITE_UPGRADE_SMITHING_TEMPLATE),
    new RecipeChoice.MaterialChoice(Material.DIAMOND_SWORD),
    new RecipeChoice.MaterialChoice(Material.NETHERITE_INGOT)
);
Bukkit.addRecipe(smithing);

// Remove on plugin disable
Bukkit.removeRecipe(key);
```

---

## 16. REGISTRIES (1.21.2+, Experimental)

```java
// Registry modifications in PluginBootstrap
@Override
public void bootstrap(BootstrapContext context) {
    context.getLifecycleManager().registerEventHandler(
        LifecycleEvents.REGISTRY_MODIFICATION.forRegistry(RegistryKey.ENCHANTMENT),
        event -> {
            // Modify existing enchantment
            // Add new enchantment
        }
    );
}
```

---

## 17. PERFORMANCE BEST PRACTICES

### 17.1 Do's

- **PlayerMoveEvent**: Only check block changes, called every tick
  ```java
  @EventHandler
  public void onMove(PlayerMoveEvent event) {
      if (event.getFrom().getBlockX() == event.getTo().getBlockX()
          && event.getFrom().getBlockY() == event.getTo().getBlockY()
          && event.getFrom().getBlockZ() == event.getTo().getBlockZ()) return;
      // Only runs on block change
  }
  ```
- **Store NamespacedKeys as static final**, don't create new ones every time
- **Cache config values**, don't read from file on every access
- **Async I/O**: Database, HTTP, file operations → `runTaskAsynchronously`
- **Collection choice**: Frequent lookups → `HashMap`/`HashSet`, ordered → `LinkedHashMap`
- **Scoreboard**: Use packet-based libraries (like FastBoard), Bukkit Scoreboard API sends packets on every update
- **Chunk loading**: Use `getChunkAtAsync()`, synchronous chunk loading drops TPS
- **Holograms/NPCs**: Use Display Entities or packet-based (PacketEvents)
- **Connection pooling**: Manage database connections with HikariCP

### 17.2 Don'ts

- Don't do heavy operations inside a `for` loop over `Bukkit.getOnlinePlayers()`
- Don't call Bukkit API from async threads (ConcurrentModificationException!)
- Don't use `EntityMoveEvent` (very heavy) — alternative: check with scheduler
- Don't read config file every tick
- Don't call `getItemMeta()` unnecessarily (creates a snapshot)
- Don't try to support `/reload` — Paper officially deprecated it

---

## 18. PLACEHOLDERAPI INTEGRATION

```java
// Expansion class
public class MyExpansion extends PlaceholderExpansion {
    private final MyPlugin plugin;

    public MyExpansion(MyPlugin plugin) { this.plugin = plugin; }

    @Override public String getIdentifier() { return "myplugin"; }
    @Override public String getAuthor() { return "AuthorName"; }
    @Override public String getVersion() { return plugin.getDescription().getVersion(); }
    @Override public boolean persist() { return true; } // Don't lose on /reload
    @Override public boolean canRegister() { return true; }

    @Override
    public String onPlaceholderRequest(Player player, String params) {
        if (player == null) return "";
        return switch (params) {
            case "coins" -> String.valueOf(getCoins(player));
            case "level" -> String.valueOf(getLevel(player));
            case "rank" -> getRank(player);
            default -> null; // Unknown placeholder
        };
    }
}

// Registration (onEnable):
if (Bukkit.getPluginManager().isPluginEnabled("PlaceholderAPI")) {
    new MyExpansion(this).register();
}

// Using PlaceholderAPI with MiniMessage:
TagResolver papiResolver = TagResolver.resolver("papi", (args, ctx) -> {
    String placeholder = args.popOr("papi tag needs an argument").value();
    String parsed = PlaceholderAPI.setPlaceholders(player, "%" + placeholder + "%");
    Component comp = LegacyComponentSerializer.legacySection().deserialize(parsed);
    return Tag.selfClosingInserting(comp);
});
player.sendMessage(MiniMessage.miniMessage().deserialize(
    "<green>Rank: <papi:luckperms_prefix>", papiResolver));
```

---

## 19. PERMISSION SYSTEM (Vault / LuckPerms)

```java
// Vault Economy
private Economy economy;

public boolean setupEconomy() {
    RegisteredServiceProvider<Economy> rsp = 
        getServer().getServicesManager().getRegistration(Economy.class);
    if (rsp == null) return false;
    economy = rsp.getProvider();
    return true;
}
// Usage:
economy.getBalance(player);
economy.withdrawPlayer(player, 100.0);
economy.depositPlayer(player, 50.0);

// LuckPerms API
LuckPerms luckPerms = LuckPermsProvider.get();
User user = luckPerms.getUserManager().getUser(player.getUniqueId());
String primaryGroup = user.getPrimaryGroup();
boolean hasNode = user.getCachedData().getPermissionData()
    .checkPermission("myplugin.vip").asBoolean();
String prefix = user.getCachedData().getMetaData().getPrefix();
```

---

## 20. RESOURCE PACK

```java
// 1.20.3+ API
player.sendResourcePacks(ResourcePackRequest.resourcePackRequest()
    .packs(ResourcePackInfo.resourcePackInfo()
        .uri(URI.create("https://example.com/pack.zip"))
        .hash("sha1hashhere")
        .build())
    .prompt(Component.text("Please download the resource pack!", NamedTextColor.YELLOW))
    .required(true) // Required
    .build());

// Event
@EventHandler
public void onResourcePackStatus(PlayerResourcePackStatusEvent event) {
    switch (event.getStatus()) {
        case ACCEPTED -> { /* Download started */ }
        case SUCCESSFULLY_LOADED -> { /* Success */ }
        case DECLINED -> { player.kick(Component.text("Resource pack is required!")); }
        case FAILED_DOWNLOAD -> { player.kick(Component.text("Download failed!")); }
    }
}
```

---

## 21. DIALOG API (1.21.7+, Experimental)

New dialog system introduced with Paper 1.21.7 — showing custom forms/dialogs.

```java
// Create and show dialog (API is still experimental, subject to change)
// For current usage see: https://docs.papermc.io/paper/dev/dialogs/
```

---

## 22. STRUCTURAL PATTERNS

### 22.1 Manager Pattern

```java
public class CooldownManager {
    private final Map<UUID, Long> cooldowns = new ConcurrentHashMap<>();

    public boolean hasCooldown(Player player) {
        Long expiry = cooldowns.get(player.getUniqueId());
        if (expiry == null) return false;
        if (System.currentTimeMillis() >= expiry) {
            cooldowns.remove(player.getUniqueId());
            return false;
        }
        return true;
    }

    public void setCooldown(Player player, Duration duration) {
        cooldowns.put(player.getUniqueId(), System.currentTimeMillis() + duration.toMillis());
    }

    public Duration getRemainingCooldown(Player player) {
        Long expiry = cooldowns.get(player.getUniqueId());
        if (expiry == null) return Duration.ZERO;
        long remaining = expiry - System.currentTimeMillis();
        return remaining > 0 ? Duration.ofMillis(remaining) : Duration.ZERO;
    }
}
```

### 22.2 Config Wrapper

```java
public record DatabaseConfig(String type, String host, int port, String name, String user, String pass) {
    public static DatabaseConfig fromConfig(FileConfiguration config) {
        return new DatabaseConfig(
            config.getString("database.type", "sqlite"),
            config.getString("database.host", "localhost"),
            config.getInt("database.port", 3306),
            config.getString("database.name", "mydb"),
            config.getString("database.username", "root"),
            config.getString("database.password", "")
        );
    }
}
```

---

## 23. COMMON MISTAKES & SOLUTIONS

| Error | Cause | Solution |
|-------|-------|----------|
| `NoClassDefFoundError` at runtime | Dependency not shaded | Create fat JAR with Shadow plugin or use `libraries:` |
| `Plugin is not associated with this PluginManager` | Event registered from different plugin loader | Use the correct plugin instance |
| Item lore is italic | Lore is italic by default | Add `.decoration(TextDecoration.ITALIC, false)` |
| `IllegalPluginAccessException` in async | Bukkit API called from async thread | Return to main thread with `runTask()` |
| PDC data disappearing | `BlockState.update()` not called | Call `state.update()` after block entity PDC changes |
| Command tab-complete not working | Old Bukkit command system | Switch to Brigadier API |
| `api-version` missing | Not specified in plugin.yml | Add `api-version: '1.21.4'`, otherwise legacy warning |
| Config special characters broken | Encoding issue | `filteringCharset = "UTF-8"` in Gradle, config files as UTF-8 |

---

## 24. RECOMMENDED PROJECT STRUCTURE

```
src/main/java/com/example/myplugin/
├── MyPlugin.java              # Main class (extends JavaPlugin)
├── MyBootstrap.java           # Paper plugin bootstrap (optional)
├── command/
│   ├── SpawnCommand.java
│   └── HomeCommand.java
├── listener/
│   ├── PlayerListener.java
│   ├── BlockListener.java
│   └── ChatListener.java
├── manager/
│   ├── HomeManager.java
│   ├── CooldownManager.java
│   └── EconomyManager.java
├── gui/
│   ├── ShopGUI.java
│   └── SettingsGUI.java
├── storage/
│   ├── DatabaseManager.java
│   └── ConfigManager.java
├── model/
│   ├── PlayerData.java        # record
│   └── Home.java              # record
├── util/
│   ├── MessageUtil.java
│   └── LocationUtil.java
└── hook/
    ├── VaultHook.java
    └── PlaceholderAPIHook.java

src/main/resources/
├── plugin.yml
├── paper-plugin.yml           # If using Paper plugin
├── config.yml
└── lang/
    ├── en.yml
    └── tr.yml
```

---

## 25. QUICK REFERENCE — IMPORTS

```java
// Paper API (always)
import org.bukkit.*;
import org.bukkit.entity.*;
import org.bukkit.event.*;
import org.bukkit.event.player.*;
import org.bukkit.event.block.*;
import org.bukkit.event.entity.*;
import org.bukkit.event.inventory.*;
import org.bukkit.inventory.*;
import org.bukkit.plugin.java.JavaPlugin;
import org.bukkit.scheduler.BukkitRunnable;
import org.bukkit.persistence.*;
import org.bukkit.attribute.*;
import org.bukkit.potion.*;
import org.bukkit.scoreboard.*;

// Adventure (Paper bundled)
import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.format.NamedTextColor;
import net.kyori.adventure.text.format.TextDecoration;
import net.kyori.adventure.text.minimessage.MiniMessage;
import net.kyori.adventure.text.minimessage.tag.resolver.Placeholder;
import net.kyori.adventure.text.minimessage.tag.resolver.TagResolver;
import net.kyori.adventure.text.minimessage.tag.Tag;
import net.kyori.adventure.text.event.ClickEvent;
import net.kyori.adventure.text.event.HoverEvent;
import net.kyori.adventure.title.Title;
import net.kyori.adventure.bossbar.BossBar;
import net.kyori.adventure.sound.Sound;

// Paper-specific
import io.papermc.paper.event.player.*;
import io.papermc.paper.plugin.lifecycle.event.types.LifecycleEvents;
import io.papermc.paper.command.brigadier.Commands;
import io.papermc.paper.command.brigadier.argument.ArgumentTypes;
import io.papermc.paper.datacomponent.DataComponentTypes;
import io.papermc.paper.datacomponent.item.*;
import io.papermc.paper.plugin.bootstrap.PluginBootstrap;
import io.papermc.paper.plugin.bootstrap.BootstrapContext;
import io.papermc.paper.registry.RegistryKey;

// Brigadier
import com.mojang.brigadier.Command;
import com.mojang.brigadier.arguments.*;
```

---

## 26. REGISTRY API (Detailed — 1.21.2+, Experimental)

Registries are Minecraft's data store: enchantments, biomes, damage types, potion effects, etc.
With Paper 1.21+, plugins can add new entries to registries or modify existing ones
during the bootstrap phase.

### 26.1 Reading Values from Registry

```java
// Modern method (1.21+)
Registry<Enchantment> enchReg = RegistryAccess.registryAccess()
    .getRegistry(RegistryKey.ENCHANTMENT);
Enchantment sharpness = enchReg.get(NamespacedKey.minecraft("sharpness"));

// With TypedKey
TypedKey<Enchantment> sharpKey = TypedKey.create(
    RegistryKey.ENCHANTMENT, Key.key("minecraft:sharpness"));

// Iterating all entries
for (Enchantment ench : enchReg) {
    Key key = enchReg.getKey(ench);
    // ...
}
```

### 26.2 Creating Custom Enchantments (Bootstrap)

```java
// MyBootstrap.java (requires paper-plugin.yml)
public class MyBootstrap implements PluginBootstrap {
    @Override
    public void bootstrap(BootstrapContext context) {
        context.getLifecycleManager().registerEventHandler(
            RegistryEvents.ENCHANTMENT.compose().newHandler(event -> {
                event.registry().register(
                    TypedKey.create(RegistryKey.ENCHANTMENT,
                        Key.key("myplugin", "lifesteal")),
                    builder -> builder
                        .description(Component.text("Lifesteal"))
                        .supportedItems(event.getOrCreateTag(ItemTypeTagKeys.SWORDS))
                        .weight(5)
                        .maxLevel(3)
                        .minimumCost(EnchantmentRegistryEntry.EnchantmentCost.of(10, 8))
                        .maximumCost(EnchantmentRegistryEntry.EnchantmentCost.of(30, 8))
                        .anvilCost(4)
                        .activeSlots(EquipmentSlotGroup.MAINHAND)
                );
            })
        );
    }
}
```

### 26.3 Modifying Existing Enchantments

```java
context.getLifecycleManager().registerEventHandler(
    RegistryEvents.ENCHANTMENT.entryAdd()
        .forKey(TypedKey.create(RegistryKey.ENCHANTMENT, Key.key("minecraft:sharpness")))
        .newHandler(event -> {
            event.builder().maxLevel(10); // Sharpness max level → 10
        })
);
```

### 26.4 Available RegistryKeys

```
RegistryKey.ENCHANTMENT          // Enchantments
RegistryKey.BIOME                // Biomes
RegistryKey.DAMAGE_TYPE          // Damage types
RegistryKey.TRIM_MATERIAL        // Armor trim materials
RegistryKey.TRIM_PATTERN         // Armor trim patterns
RegistryKey.BANNER_PATTERN       // Banner patterns
RegistryKey.PAINTING_VARIANT     // Painting variants
RegistryKey.INSTRUMENT           // Goat horn instruments
RegistryKey.STRUCTURE            // Structures (⚠ not on client, don't use as command arg)
RegistryKey.WOLF_VARIANT         // Wolf variants
RegistryKey.JUKEBOX_SONG         // Music discs
RegistryKey.MENU                 // Menu types (1.21.2+)
RegistryKey.DIALOG               // Dialog types (1.21.7+)
```

### 26.5 RegistryKeySet & Tag

```java
// Group specific enchantments
RegistryKeySet<Enchantment> swordEnchants = RegistrySet.keySet(
    RegistryKey.ENCHANTMENT,
    TypedKey.create(RegistryKey.ENCHANTMENT, Key.key("minecraft:sharpness")),
    TypedKey.create(RegistryKey.ENCHANTMENT, Key.key("minecraft:smite")),
    TypedKey.create(RegistryKey.ENCHANTMENT, Key.key("myplugin:lifesteal"))
);

// Use Vanilla tags (ItemTypeTagKeys, BlockTypeTagKeys, etc.)
// Tags are listed on the Minecraft wiki
```

---

## 27. CUSTOM WORLD GENERATION

### 27.1 ChunkGenerator

```java
public class VoidWorldGenerator extends ChunkGenerator {

    // Noise phase — main terrain shape
    @Override
    public void generateNoise(WorldInfo worldInfo, Random random,
                              int chunkX, int chunkZ, ChunkData chunkData) {
        // Void world — do nothing
        // For normal world: chunkData.setBlock(x, y, z, Material.STONE);
    }

    // Surface phase — grass, sand, etc.
    @Override
    public void generateSurface(WorldInfo worldInfo, Random random,
                                int chunkX, int chunkZ, ChunkData chunkData) {
        // Only spawn platform
        if (chunkX == 0 && chunkZ == 0) {
            for (int x = 0; x < 16; x++) {
                for (int z = 0; z < 16; z++) {
                    chunkData.setBlock(x, 64, z, Material.BEDROCK);
                    chunkData.setBlock(x, 65, z, Material.GRASS_BLOCK);
                }
            }
        }
    }

    // Control Vanilla phases
    @Override public boolean shouldGenerateNoise() { return false; }
    @Override public boolean shouldGenerateSurface() { return false; }
    @Override public boolean shouldGenerateCaves() { return false; }
    @Override public boolean shouldGenerateDecorations() { return false; }
    @Override public boolean shouldGenerateMobs() { return false; }
    @Override public boolean shouldGenerateStructures() { return false; }

    // Fixed spawn point
    @Override
    public Location getFixedSpawnLocation(World world, Random random) {
        return new Location(world, 8, 66, 8);
    }
}

// In plugin.yml: generator: MyPlugin
// In bukkit.yml or with WorldCreator:
World voidWorld = new WorldCreator("void_world")
    .generator(new VoidWorldGenerator())
    .environment(World.Environment.NORMAL)
    .createWorld();
```

### 27.2 BiomeProvider

```java
public class SingleBiomeProvider extends BiomeProvider {
    @Override
    public Biome getBiome(WorldInfo worldInfo, int x, int y, int z) {
        return Biome.PLAINS; // Everything is plains
    }

    @Override
    public List<Biome> getBiomes(WorldInfo worldInfo) {
        return List.of(Biome.PLAINS);
    }
}

// Usage:
new WorldCreator("custom_world")
    .generator(new VoidWorldGenerator())
    .biomeProvider(new SingleBiomeProvider())
    .createWorld();
```

### 27.3 BlockPopulator

```java
public class TreePopulator extends BlockPopulator {
    @Override
    public void populate(WorldInfo worldInfo, Random random,
                         int chunkX, int chunkZ, LimitedRegion limitedRegion) {
        int worldX = chunkX * 16 + random.nextInt(16);
        int worldZ = chunkZ * 16 + random.nextInt(16);
        int worldY = limitedRegion.getHighestBlockYAt(worldX, worldZ);

        if (random.nextDouble() < 0.05) { // 5% chance for tree
            Location loc = new Location(null, worldX, worldY + 1, worldZ);
            if (limitedRegion.isInRegion(loc)) {
                limitedRegion.generateTree(loc, random, TreeType.TREE);
            }
        }
    }
}

// Add to ChunkGenerator:
@Override
public List<BlockPopulator> getDefaultPopulators(World world) {
    return List.of(new TreePopulator());
}
```

---

## 28. ATTRIBUTE API (1.21.x Current)

```java
// Reading/modifying attributes
AttributeInstance maxHealth = player.getAttribute(Attribute.MAX_HEALTH);
maxHealth.setBaseValue(40.0); // 20 hearts

// Adding modifier (1.21+ Key-based, UUID deprecated!)
AttributeModifier modifier = new AttributeModifier(
    Key.key("myplugin", "strength_boost"),  // Unique key
    10.0,                                    // Amount
    AttributeModifier.Operation.ADD_NUMBER   // Operation type
);
maxHealth.addModifier(modifier);

// Removing modifier
maxHealth.removeModifier(Key.key("myplugin", "strength_boost"));

// Item attribute modifier (with Data Component API)
ItemStack boots = ItemStack.of(Material.DIAMOND_BOOTS);
boots.setData(DataComponentTypes.ATTRIBUTE_MODIFIERS,
    ItemAttributeModifiers.itemAttributes()
        .addModifier(Attribute.MOVEMENT_SPEED,
            new AttributeModifier(
                Key.key("myplugin", "speed_boots"),
                0.05,
                AttributeModifier.Operation.ADD_NUMBER
            ),
            EquipmentSlotGroup.FEET)
        .addModifier(Attribute.MAX_HEALTH,
            new AttributeModifier(
                Key.key("myplugin", "health_boots"),
                4.0,
                AttributeModifier.Operation.ADD_NUMBER
            ),
            EquipmentSlotGroup.FEET)
        .showInTooltip(true)
        .build()
);

// Operation types:
// ADD_NUMBER      → base + amount
// ADD_SCALAR      → base * (1 + amount)  (percentage increase)
// MULTIPLY_SCALAR_1 → total * (1 + amount) (multiply after all modifiers)

// All Attributes:
// MAX_HEALTH, FOLLOW_RANGE, KNOCKBACK_RESISTANCE, MOVEMENT_SPEED,
// FLYING_SPEED, ATTACK_DAMAGE, ATTACK_KNOCKBACK, ATTACK_SPEED,
// ARMOR, ARMOR_TOUGHNESS, LUCK, MAX_ABSORPTION,
// BLOCK_INTERACTION_RANGE, ENTITY_INTERACTION_RANGE,     (1.20.5+)
// BLOCK_BREAK_SPEED, GRAVITY, SAFE_FALL_DISTANCE,        (1.20.5+)
// FALL_DAMAGE_MULTIPLIER, JUMP_STRENGTH, SCALE,           (1.20.5+)
// STEP_HEIGHT, BURNING_TIME, EXPLOSION_KNOCKBACK_RESISTANCE, (1.21+)
// MINING_EFFICIENCY, MOVEMENT_EFFICIENCY, OXYGEN_BONUS,    (1.21+)
// SNEAKING_SPEED, SUBMERGED_MINING_SPEED, SWEEPING_DAMAGE_RATIO, (1.21+)
// WATER_MOVEMENT_EFFICIENCY, TEMPT_RANGE                   (1.21.2+)
```

---

## 29. POTION & EFFECT API

```java
// Giving potion effects
player.addPotionEffect(new PotionEffect(
    PotionEffectType.SPEED,
    20 * 60,            // Duration (ticks) — 60 seconds
    1,                  // Amplifier (0=Level I, 1=Level II)
    true,               // Ambient (like beacon effect)
    true,               // Show particles
    true                // Show icon
));
// Infinite duration:
player.addPotionEffect(new PotionEffect(PotionEffectType.NIGHT_VISION, -1, 0));

// Remove effect
player.removePotionEffect(PotionEffectType.SPEED);
// Remove all effects
player.clearActivePotionEffects();

// Check
boolean hasSpeed = player.hasPotionEffect(PotionEffectType.SPEED);
PotionEffect effect = player.getPotionEffect(PotionEffectType.SPEED);

// Custom Potion item (Data Component API)
ItemStack potion = ItemStack.of(Material.POTION);
potion.setData(DataComponentTypes.POTION_CONTENTS, PotionContents.potionContents()
    .potion(PotionType.HEALING)
    .addCustomEffect(new PotionEffect(PotionEffectType.REGENERATION, 200, 1))
    .customColor(Color.fromRGB(255, 0, 0))
    .build());
```

---

## 30. PARTICLE API (Paper Advanced)

```java
// Basic
world.spawnParticle(Particle.HEART, location, 5);

// With speed and offset
world.spawnParticle(Particle.FLAME, location, 
    50,     // count
    0.5,    // offsetX
    0.5,    // offsetY
    0.5,    // offsetZ
    0.02    // extra (speed)
);

// Colored dust
player.spawnParticle(Particle.DUST, location, 1,
    new Particle.DustOptions(Color.fromRGB(255, 100, 0), 2.0f));

// Dust transition (color transition)
player.spawnParticle(Particle.DUST_COLOR_TRANSITION, location, 1,
    new Particle.DustTransition(
        Color.RED, Color.BLUE, 1.5f));

// Block particle
player.spawnParticle(Particle.BLOCK, location, 30,
    Material.DIAMOND_BLOCK.createBlockData());

// Item particle
player.spawnParticle(Particle.ITEM, location, 10,
    new ItemStack(Material.GOLDEN_APPLE));

// Vibration (sculk sensor effect)
player.spawnParticle(Particle.VIBRATION, location, 1,
    new Vibration(
        new Vibration.Destination.BlockDestination(targetLocation),
        20  // arrival tick
    ));

// Show only to specific player
player.spawnParticle(Particle.HEART, location, 5,
    0, 0, 0, 0, null);

// Drawing a circle
for (double angle = 0; angle < Math.PI * 2; angle += Math.PI / 16) {
    double x = Math.cos(angle) * radius;
    double z = Math.sin(angle) * radius;
    Location point = center.clone().add(x, 0, z);
    world.spawnParticle(Particle.DUST, point, 1,
        new Particle.DustOptions(Color.PURPLE, 1.0f));
}

// Helix (spiral)
for (double y = 0; y < height; y += 0.1) {
    double angle = y * 4;
    double x = Math.cos(angle) * radius;
    double z = Math.sin(angle) * radius;
    world.spawnParticle(Particle.FLAME, center.clone().add(x, y, z), 1,
        0, 0, 0, 0);
}
```

---

## 31. WORLDBORDER API

```java
WorldBorder border = world.getWorldBorder();
border.setCenter(0, 0);
border.setSize(1000);               // 1000x1000 area
border.setSize(500, 60);            // Shrink to 500 over 60 seconds
border.setDamageBuffer(5);          // 5 block buffer (before damage starts)
border.setDamageAmount(1.0);        // 1 damage per second
border.setWarningDistance(10);       // 10 block warning distance
border.setWarningTime(15);          // 15 second warning time

// Per-player world border (Paper)
player.setWorldBorder(
    world.getWorldBorder()  // Or null to reset to default
);
```

---

## 32. MAP RENDERER

```java
public class CustomMapRenderer extends MapRenderer {
    @Override
    public void render(MapView map, MapCanvas canvas, Player player) {
        // Pixel painting (0-127 x, 0-127 y)
        for (int x = 0; x < 128; x++) {
            for (int y = 0; y < 128; y++) {
                canvas.setPixel(x, y, MapPalette.matchColor(Color.BLUE));
            }
        }

        // Draw image
        BufferedImage image = ...; // 128x128 pixels
        canvas.drawImage(0, 0, image);

        // Text
        canvas.drawText(10, 10, MinecraftFont.Font, "Hello!");
    }
}

// Usage
MapView mapView = Bukkit.createMap(world);
mapView.getRenderers().clear();
mapView.addRenderer(new CustomMapRenderer());
mapView.setTrackingPosition(false);

ItemStack map = ItemStack.of(Material.FILLED_MAP);
map.editMeta(MapMeta.class, meta -> meta.setMapView(mapView));
player.getInventory().addItem(map);
```

---

## 33. MULTILINGUAL PLUGIN PATTERN

```java
public class MessageManager {
    private final Map<String, YamlConfiguration> languages = new HashMap<>();
    private final String defaultLang = "en";

    public void loadLanguages(JavaPlugin plugin) {
        for (String lang : List.of("en", "tr")) {
            plugin.saveResource("lang/" + lang + ".yml", false);
            File file = new File(plugin.getDataFolder(), "lang/" + lang + ".yml");
            languages.put(lang, YamlConfiguration.loadConfiguration(file));
        }
    }

    public Component getMessage(Player player, String key, TagResolver... resolvers) {
        String locale = player.locale().getLanguage(); // "en", "tr", etc.
        YamlConfiguration lang = languages.getOrDefault(locale, languages.get(defaultLang));
        String raw = lang.getString(key, "<red>Missing: " + key);
        return MiniMessage.miniMessage().deserialize(raw, resolvers);
    }

    public void send(Player player, String key, TagResolver... resolvers) {
        player.sendMessage(getMessage(player, key, resolvers));
    }
}

// lang/en.yml:
// welcome: "<green>Welcome <player>! Welcome back to the server."
// no-permission: "<red>You don't have permission for this!"

// lang/tr.yml:
// welcome: "<green>Hoşgeldin <player>! Sunucuya tekrar hoşgeldin."
// no-permission: "<red>Bu işlem için yetkin yok!"

// Usage:
messageManager.send(player, "welcome",
    Placeholder.component("player", player.displayName()));
```

---

## 34. LIFECYCLE API (Detailed)

```java
// Lifecycle events are tied to the server's lifecycle
// and work reload-safe (automatic re-registration on each reload)

// Main event types:
// LifecycleEvents.COMMANDS          → Command registrations
// RegistryEvents.<REGISTRY>.compose() → New registry entry
// RegistryEvents.<REGISTRY>.entryAdd() → Modify existing entry

// From Bootstrap (PluginBootstrap — before server starts):
context.getLifecycleManager().registerEventHandler(
    LifecycleEvents.COMMANDS, event -> { /* ... */ });

// From Plugin main class (onEnable — while server is running):
this.getLifecycleManager().registerEventHandler(
    LifecycleEvents.COMMANDS, event -> { /* ... */ });

// IMPORTANT DIFFERENCE:
// - Commands registered from Bootstrap can be used in datapack functions
// - Commands registered from Plugin main class cannot (plugin loads later)
```

---

## 35. ANTI-CHEAT / SECURITY PATTERNS

```java
// Movement validation (basic)
@EventHandler(priority = EventPriority.LOWEST)
public void onMove(PlayerMoveEvent event) {
    Player player = event.getPlayer();
    Location from = event.getFrom();
    Location to = event.getTo();

    // Fly check (simple)
    if (!player.isFlying() && !player.getAllowFlight()
        && !player.getGameMode().equals(GameMode.SPECTATOR)) {
        double dy = to.getY() - from.getY();
        if (dy > 0 && !player.isOnGround() && player.getVelocity().getY() <= 0) {
            // Potential fly — do velocity check
            // CAUTION: Elytra, riptide, slime block, bed bounce, etc. cause false positives!
        }
    }

    // Speed check (simple)
    double distSq = Math.pow(to.getX() - from.getX(), 2)
                  + Math.pow(to.getZ() - from.getZ(), 2);
    double maxSpeed = getMaxAllowedSpeed(player); // Speed effect, ice, soul sand, etc.
    if (distSq > maxSpeed * maxSpeed) {
        // Potential speed hack
        event.setCancelled(true); // Teleport back
    }
}

private double getMaxAllowedSpeed(Player player) {
    double base = player.isSprinting() ? 0.36 : 0.22; // Approximate blocks/tick
    PotionEffect speed = player.getPotionEffect(PotionEffectType.SPEED);
    if (speed != null) base *= 1 + (speed.getAmplifier() + 1) * 0.2;
    AttributeInstance attr = player.getAttribute(Attribute.MOVEMENT_SPEED);
    if (attr != null) base *= attr.getValue() / 0.1;
    return base * 1.25; // 25% tolerance
}

// Cooldown-based anti-spam
private final Map<UUID, Long> lastAction = new ConcurrentHashMap<>();

public boolean checkCooldown(Player player, long cooldownMs) {
    long now = System.currentTimeMillis();
    Long last = lastAction.get(player.getUniqueId());
    if (last != null && now - last < cooldownMs) return false;
    lastAction.put(player.getUniqueId(), now);
    return true;
}

// Reach check
@EventHandler
public void onDamage(EntityDamageByEntityEvent event) {
    if (event.getDamager() instanceof Player attacker) {
        double distance = attacker.getLocation().distance(event.getEntity().getLocation());
        double maxReach = attacker.getGameMode() == GameMode.CREATIVE ? 6.0 : 3.5;
        if (distance > maxReach + 0.5) { // +0.5 tolerance
            event.setCancelled(true);
        }
    }
}
```

---

## 36. MINECRAFT 1.21.x VERSION CHANGES

### Protocol Versions

```
1.21    → Protocol 767
1.21.1  → Protocol 767 (same)
1.21.2  → Protocol 768
1.21.3  → Protocol 768 (same)
1.21.4  → Protocol 769
1.21.5  → Protocol 770
```

### Important API Changes Across Versions

```
1.21 (June 2024):
  • Added Trial Chamber, Breeze, Mace
  • New enchantments: Wind Burst, Density, Breach
  • Ominous events & Trial Spawner

1.21.2 (September 2024):
  • Bundles
  • New attributes (TEMPT_RANGE)
  • Menu Type API (experimental)

1.21.4 (December 2024):
  • Paper hardforked from Spigot
  • Mojang-mapped runtime (CraftBukkit packages not relocated!)
  • ItemStack.editPersistentDataContainer() added
  • /reload officially deprecated

1.21.5 (March 2025):
  • Leaf Litter, Test Blocks
  • Protocol 770

1.21.7:
  • Dialog API (experimental) — showing custom forms/dialogs to client
```

### paperweight-userdev Version Compatibility

```
Paper 1.20.5+  → Mojang-mapped (CraftBukkit no longer relocated)
Paper 1.21.4+  → Hardforked from Spigot (Spigot patches now independent)
                  Must use paperweight 2.x
Paper <1.20.5  → Spigot-mapped (old obfuscation)
```

---

## 37. TESTING (MockBukkit)

```xml
<!-- pom.xml test dependency -->
<dependency>
    <groupId>com.github.seeseemelk</groupId>
    <artifactId>MockBukkit-v1.21</artifactId>
    <version>4.0.0</version>
    <scope>test</scope>
</dependency>
```

```java
// JUnit 5 test
class MyPluginTest {
    private ServerMock server;
    private MyPlugin plugin;

    @BeforeEach
    void setUp() {
        server = MockBukkit.mock();
        plugin = MockBukkit.load(MyPlugin.class);
    }

    @AfterEach
    void tearDown() {
        MockBukkit.unmock();
    }

    @Test
    void testHealCommand() {
        PlayerMock player = server.addPlayer("TestPlayer");
        player.setHealth(5.0);
        player.performCommand("heal");
        assertEquals(20.0, player.getHealth());
    }

    @Test
    void testJoinEvent() {
        PlayerMock player = server.addPlayer("Newcomer");
        // Event fires automatically, check messages
        // player.assertSaid("Welcome Newcomer!");
    }
}
```

---

## 38. LOGGING BEST PRACTICES

```java
// Use plugin logger (java.util.logging)
getLogger().info("Plugin loaded!");
getLogger().warning("Config file not found, creating defaults.");
getLogger().severe("Database connection failed: " + e.getMessage());

// ComponentLogger (Paper 1.19+) — MiniMessage supported
getComponentLogger().info(Component.text("Players loaded: ")
    .append(Component.text(count, NamedTextColor.GREEN)));

// SLF4J (Paper bundled — recommended new approach)
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
private static final Logger log = LoggerFactory.getLogger(MyPlugin.class);
log.info("Loaded");
log.warn("Warning: {}", message);  // Placeholder support
log.error("Error", exception);     // Stacktrace automatically appended

// Tie DEBUG mode to config
if (getConfig().getBoolean("debug", false)) {
    getLogger().info("[DEBUG] Detailed info...");
}
```

---

## 39. SERVICE PROVIDER (ServiceManager)

```java
// Expose your own API to other plugins
public interface MyPluginAPI {
    int getPlayerLevel(Player player);
    void setPlayerLevel(Player player, int level);
}

// Registration (onEnable):
getServer().getServicesManager().register(
    MyPluginAPI.class, new MyPluginAPIImpl(),
    this, ServicePriority.Normal);

// Access from another plugin:
RegisteredServiceProvider<MyPluginAPI> rsp =
    Bukkit.getServicesManager().getRegistration(MyPluginAPI.class);
if (rsp != null) {
    MyPluginAPI api = rsp.getProvider();
    int level = api.getPlayerLevel(player);
}
```

---

## 40. BukkitRunnable PATTERN

```java
// Anonymous BukkitRunnable
new BukkitRunnable() {
    int seconds = 10;
    @Override
    public void run() {
        if (seconds <= 0) {
            // Time's up
            cancel();
            return;
        }
        player.sendActionBar(Component.text("Remaining: " + seconds + "s", NamedTextColor.RED));
        seconds--;
    }
}.runTaskTimer(plugin, 0L, 20L);

// CompletableFuture pattern (async data + sync usage)
CompletableFuture.supplyAsync(() -> {
    // Async: Fetch data from database
    return database.getPlayerData(uuid);
}).thenAccept(data -> {
    // Still on async thread — switch to main thread
    Bukkit.getScheduler().runTask(plugin, () -> {
        player.sendMessage(Component.text("Data loaded!"));
    });
}).exceptionally(ex -> {
    plugin.getLogger().severe("Data fetch error: " + ex.getMessage());
    return null;
});
```

---

## 41. DATAPACK INTEGRATION

```java
// Datapack discovery with Lifecycle API (requires Paper plugin)
// In paper-plugin.yml bootstrapper:
context.getLifecycleManager().registerEventHandler(
    LifecycleEvents.DATAPACK_DISCOVERY, event -> {
        // Load datapack from plugin JAR
        event.registrar().discoverPack(
            URI.create("file:" + myDatapackPath),
            "myplugin_datapack",
            pack -> {
                pack.title(Component.text("My Plugin Data"));
                pack.description(Component.text("Custom recipes and advancements"));
                pack.required(true);
            }
        );
    }
);

// Datapack structure inside plugin JAR:
// src/main/resources/datapack/
//   pack.mcmeta
//   data/myplugin/recipe/...
//   data/myplugin/advancement/...
//   data/myplugin/loot_table/...
```

---

## 42. NBT READING (Paper Convenience API)

```java
// 1.20.5+ — via UnsafeValues (don't use unless truly needed)
// Prefer PDC or Data Component API

// ItemStack → SNBT (for debugging)
// With NMS:
import net.minecraft.nbt.CompoundTag;
import org.bukkit.craftbukkit.inventory.CraftItemStack;

net.minecraft.world.item.ItemStack nmsItem = CraftItemStack.asNMSCopy(stack);
CompoundTag tag = (CompoundTag) nmsItem.save(
    ((CraftWorld) world).getHandle().registryAccess());
String snbt = tag.toString();

// WARNING: NMS usage requires paperweight-userdev and can break across versions!
// Always use PDC or Data Component API when possible.
```

---

## 43. SENTRY / ERROR TRACKING PATTERN

```java
// Global exception handler
Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> {
    plugin.getLogger().severe("Unexpected error [" + thread.getName() + "]: "
        + throwable.getMessage());
    throwable.printStackTrace();
});

// Safe wrapping for event handlers
public static void safeRun(JavaPlugin plugin, Runnable task) {
    try {
        task.run();
    } catch (Exception e) {
        plugin.getLogger().severe("Error: " + e.getMessage());
        e.printStackTrace();
    }
}
```

---

## 44. MOB AI & GOAL API (NMS)

```java
// Custom AI with NMS (requires paperweight-userdev)
import net.minecraft.world.entity.ai.goal.*;
import net.minecraft.world.entity.monster.Zombie;
import org.bukkit.craftbukkit.entity.CraftZombie;

Zombie nmsZombie = ((CraftZombie) zombie).getHandle();
// Clear existing goals
nmsZombie.goalSelector.getAvailableGoals().clear();
// Add new goals
nmsZombie.goalSelector.addGoal(1, new FloatGoal(nmsZombie));
nmsZombie.goalSelector.addGoal(2, new MeleeAttackGoal(nmsZombie, 1.2, false));
nmsZombie.goalSelector.addGoal(3, new RandomStrollGoal(nmsZombie, 0.8));

// Simple mob behavior with Paper API:
// Pathfinding
mob.getPathfinder().moveTo(targetLocation, 1.0); // speed
mob.getPathfinder().stopPathfinding();

// Target
mob.setTarget(player);  // LivingEntity target
mob.setAware(false);     // Disable AI
mob.setAI(false);        // Freeze completely
```

---

## 45. RESTRICTION & REGION PROTECTION PATTERN

```java
// Simple WorldGuard-like region system
public record Region(String name, Location min, Location max, Map<String, Boolean> flags) {

    public boolean contains(Location loc) {
        return loc.getWorld().equals(min.getWorld())
            && loc.getBlockX() >= min.getBlockX() && loc.getBlockX() <= max.getBlockX()
            && loc.getBlockY() >= min.getBlockY() && loc.getBlockY() <= max.getBlockY()
            && loc.getBlockZ() >= min.getBlockZ() && loc.getBlockZ() <= max.getBlockZ();
    }

    public boolean getFlag(String flag, boolean defaultValue) {
        return flags.getOrDefault(flag, defaultValue);
    }
}

// Check in events
@EventHandler(priority = EventPriority.LOW)
public void onBlockBreak(BlockBreakEvent event) {
    for (Region region : regionManager.getRegions()) {
        if (region.contains(event.getBlock().getLocation())) {
            if (!region.getFlag("block-break", true)) {
                event.setCancelled(true);
                event.getPlayer().sendRichMessage("<red>You cannot break blocks in this region!");
                return;
            }
        }
    }
}
```

---

## 46. PROTOCOL DETAILS — HANDSHAKE & LOGIN FLOW

```
Client → Server connection flow:

1. HANDSHAKE (Packet 0x00)
   - Protocol Version (VarInt): 769 (1.21.4)
   - Server Address (String): play.example.net
   - Server Port (Unsigned Short): 25565
   - Next State (VarInt): 1=Status, 2=Login, 3=Transfer

2. LOGIN
   - Client → LoginStart (0x00): username, UUID
   - Server → EncryptionRequest (0x01): server ID, public key, verify token
   - Client → EncryptionResponse (0x02): shared secret, verify token
   - Server → LoginSuccess (0x02): UUID, username, properties
   - [If compression enabled: SetCompression packet]

3. CONFIGURATION (1.20.2+)
   - Server → RegistryData, FeatureFlags, KnownPacks
   - Client → AcknowledgeFinish
   
4. PLAY
   - Server → JoinGame, ChunkData, PlayerInfo, etc.
   - Packets now flow bidirectionally

// This info is critical for anti-bot, protocol analysis,
// and custom handshake validation plugins.
```

---

## 47. GrimAC / ANTI-CHEAT INTEGRATION PATTERN

```java
// Integration with GrimAC (event-based)
// Via ConditionalEvents or Grim's own events

// Listening to Grim flag events (3rd party API)
@EventHandler
public void onGrimFlag(FlagEvent event) {
    // Hack type detected by Grim
    String checkName = event.getCheckName();
    Player player = event.getPlayer();
    
    // Punishment system
    int violations = getViolationCount(player, checkName);
    if (violations > 10) {
        player.kick(Component.text("Anti-Cheat: Suspicious activity", NamedTextColor.RED));
    }
}

// For general anti-cheat plugin compatibility:
// - Listen to events at LOWEST priority
// - Skip if another plugin cancelled (ignoreCancelled=true)
// - Apply cooldown after teleport/velocity events
// - Handle edge cases: elytra, riptide, slime block, piston, etc.
```

---

**Javadoc:** https://jd.papermc.io/paper/1.21.4/  
**Paper Docs:** https://docs.papermc.io/paper/dev/  
**Adventure Docs:** https://docs.papermc.io/adventure/  
**MiniMessage Viewer:** https://webui.advntr.dev/  
**Paper GitHub:** https://github.com/PaperMC/Paper  
**paperweight-userdev:** https://docs.papermc.io/paper/dev/userdev/  
**PacketEvents:** https://github.com/retrooper/packetevents  
**MockBukkit:** https://github.com/MockBukkit/MockBukkit  
