---
name: paper-plugin-dev
description: >
  Minecraft Paper plugin geliştirme skill'i. Java 21 ve Paper 1.21.x (1.21.1 - 1.21.11) hedefli
  plugin kodlama, API kullanımı, event handling, command registration, NMS/internals erişimi,
  packet manipülasyonu, performans optimizasyonu ve best practice'ler için kullanılır.
  Bu skill'i şu durumlarda MUTLAKA kullan: Minecraft plugin yazma/düzenleme, Paper API kullanımı,
  Bukkit/Spigot/Paper event'leri, Brigadier command sistemi, Adventure/MiniMessage component'leri,
  Data Component API, PDC (Persistent Data Container), entity/block/item manipülasyonu,
  inventory GUI'leri, scheduler/async task'ler, packet dinleme/gönderme, anti-cheat mekanikleri,
  NMS (net.minecraft.server) erişimi, paperweight-userdev kullanımı, plugin.yml / paper-plugin.yml
  yapılandırması, Gradle/Maven proje kurulumu, ProtocolLib/PacketEvents entegrasyonu,
  world generation, scoreboard/team API, boss bar, title, action bar, tab list, resource pack,
  database entegrasyonu, config yönetimi, permission sistemi, PlaceholderAPI entegrasyonu,
  ve Minecraft sunucu eklentisi ile ilgili HER TÜRLÜ kodlama görevi. Kullanıcı "plugin", "eklenti",
  "Paper", "Bukkit", "Spigot", "Minecraft", "server mod", "sunucu", "SMP" gibi kelimeler
  kullandığında veya Java ile Minecraft geliştirme bağlamında bir şey sorduğunda bu skill tetiklenmeli.
---

# Minecraft Paper Plugin Development — Java 21 & Paper 1.21.x

Bu skill, Claude'un Minecraft Paper sunucusu için profesyonel kalitede Java 21 plugin'leri
yazabilmesini sağlayan kapsamlı bir referans ve kural setidir.

---

## 1. PROJE KURULUMU

### 1.1 Gradle (Kotlin DSL) — ÖNERİLEN

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
    id("xyz.jpenilla.run-paper") version "2.3.1"          // Test sunucusu
    // NMS gerekiyorsa:
    // id("io.papermc.paperweight.userdev") version "2.0.0-beta.19"
}

group = "com.example"
version = "1.0.0"

repositories {
    mavenCentral()
    maven("https://repo.papermc.io/repository/maven-public/")
}

dependencies {
    // Sadece API:
    compileOnly("io.papermc.paper:paper-api:1.21.4-R0.1-SNAPSHOT")
    // NMS gerekiyorsa API yerine:
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

### 1.3 Java 21 Önemli Özellikler (Plugin'lerde Kullan!)

- **Record sınıfları**: DTO/config veri yapıları için `public record PlayerData(UUID uuid, int level) {}`
- **Sealed sınıflar**: `sealed interface Reward permits CoinReward, ItemReward {}`
- **Pattern matching**: `if (entity instanceof Player player) { ... }`
- **Switch expressions**: `var msg = switch(rank) { case ADMIN -> "Yönetici"; case MOD -> "Moderatör"; default -> "Oyuncu"; };`
- **Text blocks**: Çok satırlı string'ler için `"""..."""`
- **Virtual threads** (preview): Ağır I/O işlemleri için `Thread.ofVirtual().start(() -> ...)`
- **SequencedCollection**: `list.getFirst()`, `list.getLast()`, `list.reversed()`

---

## 2. PLUGIN GİRİŞ NOKTASI VE YAŞAM DÖNGÜSÜ

### 2.1 Bukkit Plugin (Klasik — plugin.yml)

```java
public final class MyPlugin extends JavaPlugin {

    @Override
    public void onLoad() {
        // Dünya yüklenmeden ÖNCE çalışır. Config, bağımlılık kontrolü.
    }

    @Override
    public void onEnable() {
        // Sunucu açılınca. Event, command, scheduler kayıtları BURADA.
        saveDefaultConfig();
        getServer().getPluginManager().registerEvents(new MyListener(this), this);
        
        // Brigadier command kayıt (1.20.6+)
        getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
            final Commands commands = event.registrar();
            // command'ları burada kaydet
        });
    }

    @Override
    public void onDisable() {
        // Sunucu kapanınca. Veri kaydetme, cleanup.
    }
}
```

### 2.2 plugin.yml

```yaml
name: MyPlugin
version: '${version}'
main: com.example.myplugin.MyPlugin
api-version: '1.21.4'
description: Açıklama
author: AuthorName
website: https://example.com

load: POSTWORLD  # veya STARTUP

depend: [Vault, LuckPerms]        # Zorunlu bağımlılıklar
softdepend: [PlaceholderAPI]       # İsteğe bağlı
loadbefore: [AnotherPlugin]        # Bu plugin önce yüklensin

permissions:
  myplugin.admin:
    description: Admin izni
    default: op
  myplugin.use:
    description: Kullanım izni
    default: true

# plugin.yml'de command tanımlamak isteğe bağlı (Brigadier kullanıyorsan gereksiz)
commands:
  mycommand:
    description: Örnek komut
    usage: /<command> [args]
    aliases: [mc, mycmd]
    permission: myplugin.use
```

### 2.3 Paper Plugin (Deneysel — paper-plugin.yml)

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
// Paper Plugin Bootstrapper — sunucu başlamadan önce kod çalıştırma
public class MyBootstrap implements PluginBootstrap {
    @Override
    public void bootstrap(BootstrapContext context) {
        // Registry değişiklikleri, erken initialization
        context.getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
            // Komutları bootstrapper'dan kaydet — datapack function'larda kullanılabilir
        });
    }
}

// Paper Plugin Loader — classpath yönetimi
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

**Paper plugin vs Bukkit plugin farkları:**
- Paper plugin'ler classloading izolasyonu uygular (başka plugin'lerin sınıflarına erişemezsin)
- `join-classpath: true` ile izolasyonu by-pass edebilirsin
- Command kayıtları `paper-plugin.yml`'de yapılmaz, Brigadier API kullanılır
- Bukkit serileştirme sistemi desteklenir ama `ConfigurationSerialization.registerClass()` manuel çağrılmalı

---

## 3. EVENT SİSTEMİ

### 3.1 Event Listener Kayıt

```java
public class MyListener implements Listener {

    private final MyPlugin plugin;

    public MyListener(MyPlugin plugin) {
        this.plugin = plugin;
    }

    // Temel event dinleme
    @EventHandler
    public void onJoin(PlayerJoinEvent event) {
        event.joinMessage(Component.text("Hoşgeldin ")
            .append(event.getPlayer().displayName())
            .color(NamedTextColor.GREEN));
    }

    // Öncelik ve iptal kontrolü
    @EventHandler(priority = EventPriority.HIGH, ignoreCancelled = true)
    public void onBreak(BlockBreakEvent event) {
        // ignoreCancelled = true → zaten iptal edilmişse bu handler çalışmaz
    }

    // Paper'a özel event'ler
    @EventHandler
    public void onAsync(AsyncChatEvent event) {
        // Paper'ın async chat event'i — ASLA Bukkit API çağırma burada!
        Component message = event.message();
        // renderer ile mesaj formatı değiştirilebilir
        event.renderer(ChatRenderer.viewerUnaware((source, sourceDisplayName, msg) ->
            sourceDisplayName.append(Component.text(" > ")).append(msg)
        ));
    }
}
```

### 3.2 Event Öncelik Sırası (EventPriority)

```
LOWEST → LOW → NORMAL (varsayılan) → HIGH → HIGHEST → MONITOR
```

- `MONITOR`: Sadece izleme için. Event'i İPTAL ETME, değiştirme.
- İptal mantığı: Yüksek öncelikli handler düşük önceliklinin kararını geçersiz kılabilir.
- `ignoreCancelled = true` ekle ki iptal edilmiş event'leri boşuna işleme.

### 3.3 Önemli Paper-Specific Event'ler

```java
// Async — Ana thread'de değil, Bukkit API çağırma!
AsyncChatEvent              // Sohbet mesajı (Paper)
AsyncPlayerPreLoginEvent    // Login öncesi (IP ban, whitelist)

// Sunucu
ServerLoadEvent             // Sunucu tamamen yüklendiğinde
WhitelistStateUpdateEvent   // Whitelist değişikliği

// Oyuncu
PlayerArmorChangeEvent      // Zırh değişimi (Paper)
PlayerBedFailEnterEvent     // Yatağa girememe
PlayerDeathEvent            // Ölüm (getDeathMessage() → Component)
PlayerMoveEvent             // Hareket (çok sık çağrılır — dikkat!)
PlayerInteractEvent         // Sağ/sol tık
PlayerInteractEntityEvent   // Entity'ye tıklama
PlayerItemConsumeEvent      // Yeme/içme

// Blok
BlockBreakEvent
BlockPlaceEvent
BlockPhysicsEvent           // Redstone, kum düşmesi vb.
BlockFromToEvent            // Su/lav akışı

// Entity
EntityDamageByEntityEvent   // PvP/PvE hasar
EntityDeathEvent
CreatureSpawnEvent          // SpawnReason kontrolü
ProjectileHitEvent          // Ok/trident/snowball isabet

// İnventory
InventoryClickEvent
InventoryCloseEvent
InventoryDragEvent
PrepareItemCraftEvent       // Crafting önizleme

// Paper ekstra
PlayerArmSwingEvent
EntityMoveEvent             // Entity hareket (performans maliyetli)
PlayerChunkLoadEvent / PlayerChunkUnloadEvent
BeaconActivatedEvent / BeaconDeactivatedEvent
```

### 3.4 Custom Event Oluşturma

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

// Kullanım:
PlayerLevelUpEvent event = new PlayerLevelUpEvent(player, 10);
Bukkit.getPluginManager().callEvent(event);
if (!event.isCancelled()) {
    // Level up işlemi
}
```

---

## 4. COMMAND SİSTEMİ (Brigadier)

### 4.1 Brigadier Command Kaydı (Paper 1.20.6+)

```java
// onEnable() veya PluginBootstrap içinde:
getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
    final Commands commands = event.registrar();

    // Basit command
    commands.register(
        Commands.literal("heal")
            .requires(src -> src.getSender().hasPermission("myplugin.heal"))
            .executes(ctx -> {
                if (ctx.getSource().getSender() instanceof Player player) {
                    player.setHealth(player.getMaxHealth());
                    player.sendMessage(Component.text("İyileştirildin!", NamedTextColor.GREEN));
                }
                return Command.SINGLE_SUCCESS;
            })
            .build(),
        "Oyuncuyu iyileştirir",           // Açıklama
        List.of("h", "iyilestir")         // Alias'lar
    );

    // Argümanlı command
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
                            Component.text(target.getName() + " oyuncusuna " + amount + " XP verildi."));
                        return Command.SINGLE_SUCCESS;
                    })
                )
            )
            .build(),
        "XP verir",
        List.of("xp")
    );
});
```

### 4.2 BasicCommand (Basit Alternatif)

```java
commands.register("ping", "Gecikmeyi gösterir", List.of("gecikme"),
    new BasicCommand() {
        @Override
        public void execute(CommandSourceStack source, String[] args) {
            if (source.getSender() instanceof Player player) {
                player.sendMessage(Component.text("Gecikme: " + player.getPing() + "ms"));
            }
        }

        @Override
        public Collection<String> suggest(CommandSourceStack source, String[] args) {
            return List.of(); // Tab-complete önerileri
        }
    }
);
```

### 4.3 Brigadier Argüman Türleri

```java
// Vanilla Minecraft argümanları
ArgumentTypes.player()            // Tek oyuncu seçici
ArgumentTypes.players()           // Çoklu oyuncu seçici
ArgumentTypes.entity()            // Tek entity
ArgumentTypes.entities()          // Çoklu entity
ArgumentTypes.blockState()        // Blok durumu
ArgumentTypes.itemStack()         // Item stack
ArgumentTypes.resource(RegistryKey.ENCHANTMENT)  // Registry kaynağı
ArgumentTypes.namespacedKey()     // NamespacedKey

// Brigadier temel argümanları
IntegerArgumentType.integer()
IntegerArgumentType.integer(min, max)
DoubleArgumentType.doubleArg()
FloatArgumentType.floatArg()
BoolArgumentType.bool()
StringArgumentType.word()         // Tek kelime
StringArgumentType.string()       // Tırnaklı string
StringArgumentType.greedyString() // Satırın geri kalanı

// Paper-specific
ArgumentTypes.signedMessage()     // İmzalı mesaj (chat reporting)
ArgumentTypes.component()         // Component (JSON)
ArgumentTypes.key()               // Key (namespace:value)
ArgumentTypes.finePosition()      // Hassas konum (double x, y, z)
ArgumentTypes.blockPosition()     // Blok konumu (int x, y, z)
```

---

## 5. ADVENTURE COMPONENT API & MINIMESSAGE

### 5.1 Component Oluşturma

```java
// Temel text
Component text = Component.text("Merhaba Dünya!", NamedTextColor.GOLD, TextDecoration.BOLD);

// Zincirleme
Component msg = Component.text()
    .content("Sunucuya hoşgeldin! ")
    .color(NamedTextColor.GREEN)
    .append(Component.text("[TIKLA]", NamedTextColor.YELLOW, TextDecoration.BOLD)
        .clickEvent(ClickEvent.runCommand("/spawn"))
        .hoverEvent(HoverEvent.showText(Component.text("Spawn'a ışınlan"))))
    .build();

// Translatable (çok dilli)
Component translatable = Component.translatable("death.attack.player", 
    victim.displayName(), killer.displayName());

// Keybind
Component keybind = Component.keybind("key.inventory"); // Oyuncunun envanter tuşunu gösterir

// Gradient (MiniMessage ile)
Component gradient = MiniMessage.miniMessage().deserialize(
    "<gradient:gold:red>Bu yazı gradient'li!</gradient>");
```

### 5.2 MiniMessage (String ↔ Component)

```java
MiniMessage mm = MiniMessage.miniMessage();

// Parse
Component parsed = mm.deserialize(
    "<bold><red>UYARI:</red></bold> <gray>Bölgeye giremezsin!");

// Placeholder'lı parse
Component msg = mm.deserialize(
    "<green>Hoşgeldin <player>! Seviye: <level>",
    Placeholder.component("player", player.displayName()),
    Placeholder.unparsed("level", String.valueOf(playerLevel))
);

// TagResolver ile custom tag
TagResolver resolver = TagResolver.resolver("rank",
    Tag.selfClosingInserting(Component.text("[VIP]", NamedTextColor.GOLD)));
Component result = mm.deserialize("<rank> <green>Merhaba!", resolver);

// Serialize (Component → String)
String serialized = mm.serialize(someComponent);

// sendRichMessage kısayolu (Paper 1.19.4+)
player.sendRichMessage("<rainbow>Gökkuşağı mesaj!");
player.sendRichMessage("Merhaba <name>!", Placeholder.unparsed("name", player.getName()));
```

### 5.3 MiniMessage Tag Referansı

```
Renkler:      <red>, <#FF5555>, <color:#FF5555>, <color:red>
Dekorasyon:   <bold>, <italic>, <underlined>, <strikethrough>, <obfuscated>
Dekorasyon kapat: </bold> veya <!bold>
Gradient:     <gradient:red:blue>, <gradient:#FF0000:#0000FF>
Rainbow:      <rainbow>, <rainbow:phase>
Hover:        <hover:show_text:'Hover mesajı'>metin</hover>
Click:        <click:run_command:'/spawn'>Tıkla</click>
              <click:open_url:'https://...'>Link</click>
              <click:suggest_command:'/msg '>Yaz</click>
              <click:copy_to_clipboard:'metin'>Kopyala</click>
Insert:       <insert:metin>Shift+tıkla</insert>
Keybind:      <key:key.inventory>
Translatable: <lang:block.minecraft.diamond_block>
Selector:     <sel:@p>
Font:         <font:uniform>metin</font>
Reset:        <reset>
Newline:      <newline> veya <br>
Transition:   <transition:red:blue:0.5>
```

### 5.4 Mesaj Gönderme Yöntemleri

```java
// Chat mesajı
player.sendMessage(component);
player.sendRichMessage("<green>Başarılı!");

// Action bar
player.sendActionBar(Component.text("⚔ Savaş modu aktif!", NamedTextColor.RED));

// Title + Subtitle
player.showTitle(Title.title(
    Component.text("LEVEL UP!", NamedTextColor.GOLD),
    Component.text("Seviye 10'a ulaştın!", NamedTextColor.YELLOW),
    Title.Times.times(Duration.ofMillis(500), Duration.ofSeconds(3), Duration.ofMillis(500))
));

// Boss bar
BossBar bossBar = BossBar.bossBar(
    Component.text("Etkinlik: Boss Savaşı", NamedTextColor.RED),
    0.75f,                    // İlerleme (0.0 - 1.0)
    BossBar.Color.RED,
    BossBar.Overlay.PROGRESS
);
player.showBossBar(bossBar);
// İlerleme güncelle: bossBar.progress(0.5f);
// Kaldır: player.hideBossBar(bossBar);

// Tab list header/footer
player.sendPlayerListHeaderAndFooter(
    Component.text("✦ SunucuAdı ✦", NamedTextColor.GOLD),
    Component.text("Oyuncu sayısı: " + Bukkit.getOnlinePlayers().size())
);

// Scoreboard (sidebar)
// Bukkit Scoreboard API kullanılır — aşağıda detaylı

// Kitap
ItemStack book = ItemStack.of(Material.WRITTEN_BOOK);
book.setData(DataComponentTypes.WRITTEN_BOOK_CONTENT,
    WrittenBookContent.writtenBookContent("Rehber", "Sunucu")
        .addPage(Component.text("Sayfa 1 içeriği"))
        .addPage(Component.text("Sayfa 2 içeriği"))
        .build());
player.openBook(book);
```

---

## 6. DATA COMPONENT API (1.21.x, Deneysel)

Paper 1.21+ ItemStack'ların Vanilla data component'lerine doğrudan erişim sağlar.
ItemMeta'dan farkı: prototype (varsayılan) değerlere erişim, component kaldırma, daha performanslı.

```java
ItemStack sword = ItemStack.of(Material.DIAMOND_SWORD);

// Okuma
Integer maxDamage = sword.getData(DataComponentTypes.MAX_DAMAGE);
boolean hasTool = sword.hasData(DataComponentTypes.TOOL);
// Prototype (varsayılan) değer okuma
int defaultDurability = Material.DIAMOND_SWORD.getDefaultData(DataComponentTypes.MAX_DAMAGE);

// Yazma
sword.setData(DataComponentTypes.CUSTOM_NAME, Component.text("Ejder Kılıcı", NamedTextColor.LIGHT_PURPLE));
sword.setData(DataComponentTypes.LORE, ItemLore.lore()
    .addLine(Component.text("Efsanevi kılıç", NamedTextColor.GRAY).decoration(TextDecoration.ITALIC, false))
    .build());
sword.setData(DataComponentTypes.ENCHANTMENTS, ItemEnchantments.itemEnchantments()
    .add(Enchantment.SHARPNESS, 5)
    .add(Enchantment.UNBREAKING, 3)
    .showInTooltip(true)
    .build());
sword.setData(DataComponentTypes.RARITY, ItemRarity.EPIC);
sword.setData(DataComponentTypes.ENCHANTMENT_GLINT_OVERRIDE, true); // Parıldama
sword.setData(DataComponentTypes.MAX_DAMAGE, 2000);
sword.setData(DataComponentTypes.CUSTOM_MODEL_DATA, CustomModelData.customModelData()
    .addFloat(0.5f).addFlag(true).build());

// Component kaldırma (prototype'dan bile)
sword.unsetData(DataComponentTypes.TOOL); // Artık tool değil
// Prototype'a sıfırlama
sword.resetData(DataComponentTypes.MAX_STACK_SIZE);

// Non-valued (flag) component'ler
sword.setData(DataComponentTypes.GLIDER);        // Elytra gibi süzülme
sword.unsetData(DataComponentTypes.GLIDER);

// Builder pattern (karmaşık component'ler)
Equippable.Builder builder = sword.getData(DataComponentTypes.EQUIPPABLE).toBuilder();
builder.equipSound(SoundEventKeys.ENTITY_GHAST_HURT);
sword.setData(DataComponentTypes.EQUIPPABLE, builder);

// WrittenBookContent
ItemStack book = ItemStack.of(Material.WRITTEN_BOOK);
book.setData(DataComponentTypes.WRITTEN_BOOK_CONTENT,
    WrittenBookContent.writtenBookContent("Kitap", "Yazar")
        .addPage(Component.text("Sayfa 1"))
        .generation(0)
        .build());
```

**Önemli DataComponentTypes:**
`CUSTOM_NAME`, `LORE`, `ENCHANTMENTS`, `DAMAGE`, `MAX_DAMAGE`, `MAX_STACK_SIZE`,
`RARITY`, `UNBREAKABLE`, `CUSTOM_MODEL_DATA`, `DYED_COLOR`, `TOOL`, `FOOD`,
`POTION_CONTENTS`, `ATTRIBUTE_MODIFIERS`, `ENCHANTMENT_GLINT_OVERRIDE`, `FIRE_RESISTANT`,
`HIDE_TOOLTIP`, `WRITTEN_BOOK_CONTENT`, `EQUIPPABLE`, `GLIDER`, `CONTAINER`,
`BANNER_PATTERNS`, `CHARGED_PROJECTILES`, `PROFILE` (skull), `CONSUMABLE`, `USE_COOLDOWN`

---

## 7. PERSISTENT DATA CONTAINER (PDC)

PDC, item/entity/block entity'lere özel veri depolamak için kullanılır. NBT'ye doğrudan
erişimden çok daha güvenli ve sürümler arası stabil.

```java
// NamespacedKey (yeniden kullanılabilir tutulmalı!)
private static final NamespacedKey SOUL_KEY = new NamespacedKey(plugin, "soul-count");
private static final NamespacedKey OWNER_KEY = new NamespacedKey(plugin, "owner-uuid");

// === ITEM PDC ===
// 1.21.4+ doğrudan edit:
stack.editPersistentDataContainer(pdc -> {
    pdc.set(SOUL_KEY, PersistentDataType.INTEGER, 42);
    pdc.set(OWNER_KEY, PersistentDataType.STRING, player.getUniqueId().toString());
});
// 1.21.1+ sadece okuma (snapshot oluşturmadan):
Integer souls = stack.getPersistentDataContainer().get(SOUL_KEY, PersistentDataType.INTEGER);
// Eski yöntem (hala çalışır):
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
    chest.update(); // Block entity'de update() ZORUNLU!
}

// === OYUNCU PDC ===
player.getPersistentDataContainer().set(SOUL_KEY, PersistentDataType.INTEGER, 100);
// OfflinePlayer sadece read-only PDC sunar

// Kontrol & silme
boolean has = pdc.has(SOUL_KEY, PersistentDataType.INTEGER);
boolean hasAny = pdc.has(SOUL_KEY); // 1.20.5+ type-agnostic
pdc.remove(SOUL_KEY);
int valOrDefault = pdc.getOrDefault(SOUL_KEY, PersistentDataType.INTEGER, 0);

// Desteklenen tipler:
// BYTE, SHORT, INTEGER, LONG, FLOAT, DOUBLE, STRING, BYTE_ARRAY,
// INTEGER_ARRAY, LONG_ARRAY, BOOLEAN, TAG_CONTAINER (iç içe PDC),
// LIST (PersistentDataType.LIST.of(PersistentDataType.STRING))

// Nested PDC (Tag Container)
PersistentDataContainer inner = pdc.getAdapterContext().newPersistentDataContainer();
inner.set(new NamespacedKey(plugin, "x"), PersistentDataType.DOUBLE, 100.5);
inner.set(new NamespacedKey(plugin, "y"), PersistentDataType.DOUBLE, 64.0);
pdc.set(new NamespacedKey(plugin, "home"), PersistentDataType.TAG_CONTAINER, inner);

// List (1.20.5+)
pdc.set(new NamespacedKey(plugin, "friends"),
    PersistentDataType.LIST.strings(), List.of("Ahmet", "Mehmet"));
```

**DİKKAT:** PDC verisi holder'lar arasında otomatik kopyalanmaz! Block olarak yerleştirilen
ItemStack'ın PDC'si block entity'ye taşınmaz — manuel kopyalaman gerekir.

---

## 8. INVENTORY & GUI

### 8.1 Custom Inventory GUI

```java
public class ShopGUI implements InventoryHolder {
    private final Inventory inventory;

    public ShopGUI() {
        // 9'un katı olmalı (9, 18, 27, 36, 45, 54)
        this.inventory = Bukkit.createInventory(this, 54, 
            Component.text("✦ Mağaza", NamedTextColor.DARK_PURPLE));
        setupItems();
    }

    private void setupItems() {
        // Cam panel dolgu
        ItemStack filler = ItemStack.of(Material.BLACK_STAINED_GLASS_PANE);
        filler.editMeta(meta -> meta.displayName(Component.empty()));
        for (int i = 0; i < 54; i++) inventory.setItem(i, filler);

        // Satılık item
        ItemStack diamond = ItemStack.of(Material.DIAMOND);
        diamond.editMeta(meta -> {
            meta.displayName(Component.text("Elmas", NamedTextColor.AQUA)
                .decoration(TextDecoration.ITALIC, false));
            meta.lore(List.of(
                Component.text("Fiyat: 100 Coin", NamedTextColor.GOLD)
                    .decoration(TextDecoration.ITALIC, false),
                Component.empty(),
                Component.text("» Sol tıkla — Satın al", NamedTextColor.YELLOW)
                    .decoration(TextDecoration.ITALIC, false)
            ));
        });
        // PDC ile ID ekle
        diamond.editPersistentDataContainer(pdc ->
            pdc.set(new NamespacedKey(plugin, "shop-item"), PersistentDataType.STRING, "diamond_1"));
        inventory.setItem(22, diamond);
    }

    @Override
    public Inventory getInventory() { return inventory; }
}

// Açma:
player.openInventory(new ShopGUI().getInventory());

// Click handler:
@EventHandler
public void onClick(InventoryClickEvent event) {
    if (!(event.getInventory().getHolder() instanceof ShopGUI)) return;
    event.setCancelled(true); // Her zaman iptal et ki item çıkarılmasın

    if (event.getCurrentItem() == null) return;
    if (!(event.getWhoClicked() instanceof Player player)) return;

    ItemStack clicked = event.getCurrentItem();
    var pdc = clicked.getPersistentDataContainer();
    String itemId = pdc.get(new NamespacedKey(plugin, "shop-item"), PersistentDataType.STRING);
    if (itemId == null) return;

    // Satın alma mantığı...
    player.sendRichMessage("<green>Satın alındı!");
}

// Kapatma handler:
@EventHandler
public void onClose(InventoryCloseEvent event) {
    if (event.getInventory().getHolder() instanceof ShopGUI) {
        // Cleanup, veri kaydetme vb.
    }
}
```

### 8.2 Menu Type API (1.21.2+, Deneysel)

```java
// Daha type-safe yaklaşım
MenuType.GENERIC_9X6.create(Component.text("Envanter")).open(player);
```

---

## 9. SCHEDULER & ASYNC İŞLEMLER

### 9.1 BukkitScheduler

```java
// Bir sonraki tick'te çalıştır (1 tick = 50ms, 20 tick = 1 saniye)
Bukkit.getScheduler().runTask(plugin, () -> {
    player.teleport(location); // Bukkit API çağrıları MUTLAKA ana thread'de
});

// Gecikmeli (40 tick = 2 saniye sonra)
Bukkit.getScheduler().runTaskLater(plugin, () -> {
    player.sendMessage(Component.text("2 saniye geçti!"));
}, 40L);

// Tekrarlayan (her 20 tick = 1 saniye)
BukkitTask task = Bukkit.getScheduler().runTaskTimer(plugin, () -> {
    // Her saniye çalışır
}, 0L, 20L);
// Durdurma: task.cancel();

// ASYNC — Ağır I/O, veritabanı, HTTP istekleri için
Bukkit.getScheduler().runTaskAsynchronously(plugin, () -> {
    // Veritabanı sorgusu (async thread)
    String data = database.query("...");
    
    // Sonucu ana thread'e geri gönder
    Bukkit.getScheduler().runTask(plugin, () -> {
        player.sendMessage(Component.text("Veri: " + data));
    });
});

// Async tekrarlayan
Bukkit.getScheduler().runTaskTimerAsynchronously(plugin, () -> {
    // Periyodik async iş
}, 0L, 20L * 60); // Her dakika
```

### 9.2 Folia Desteği (İsteğe Bağlı)

```java
// Folia region-based scheduling
if (Bukkit.getServer().getName().contains("Folia")) {
    // Folia'da global scheduler:
    Bukkit.getGlobalRegionScheduler().run(plugin, task -> { ... });
    // Entity'nin region'ında:
    entity.getScheduler().run(plugin, task -> { ... }, null);
    // Async:
    Bukkit.getAsyncScheduler().runNow(plugin, task -> { ... });
}
```

### 9.3 ÖNEMLİ THREADING KURALLARI

```
╔══════════════════════════════════════════════════════════════╗
║  ANA THREAD (Tick Thread):                                  ║
║  • Bukkit/Paper API çağrıları (getPlayer, teleport, vb.)   ║
║  • Dünya değişiklikleri (blok koyma/kırma)                  ║
║  • Entity spawn/despawn                                      ║
║  • Envanter değişiklikleri                                   ║
║  • Event handling                                            ║
╠══════════════════════════════════════════════════════════════╣
║  ASYNC THREAD:                                               ║
║  • Veritabanı sorguları (MySQL, SQLite, Redis)              ║
║  • HTTP/API istekleri                                        ║
║  • Dosya I/O (büyük dosya okuma/yazma)                      ║
║  • Ağır hesaplamalar                                         ║
║  • ASLA Bukkit API çağırma!                                  ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 10. VERİTABANI ENTEGRASYONU

### 10.1 HikariCP + SQLite/MySQL

```java
// plugin.yml libraries bölümünde (auto-download):
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

    // Tablo oluştur
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
        getLogger().severe("Veritabanı hatası: " + e.getMessage());
    }
}

// Async veri okuma
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

// Kullanım:
getCoins(player.getUniqueId()).thenAcceptAsync(coins -> {
    Bukkit.getScheduler().runTask(plugin, () -> {
        player.sendMessage(Component.text("Coinlerin: " + coins));
    });
});
```

---

## 11. CONFIG YÖNETİMİ

```java
// config.yml oluştur/yükle
@Override
public void onEnable() {
    saveDefaultConfig(); // resources/config.yml → plugins/MyPlugin/config.yml

    // Okuma
    String prefix = getConfig().getString("prefix", "[Sunucu]");
    int maxHomes = getConfig().getInt("max-homes", 3);
    List<String> worlds = getConfig().getStringList("disabled-worlds");
    boolean enabled = getConfig().getBoolean("features.pvp-enabled", true);

    // Yazma
    getConfig().set("stats.total-joins", 100);
    saveConfig();

    // Yeniden yükleme
    reloadConfig();
}
```

```yaml
# resources/config.yml
prefix: "<gradient:gold:yellow>✦ SunucuAdı</gradient>"
max-homes: 5
disabled-worlds:
  - world_nether
  - world_the_end
features:
  pvp-enabled: true
  keep-inventory: false
database:
  type: sqlite  # sqlite veya mysql
  host: localhost
  port: 3306
  name: mydb
  username: root
  password: ""
messages:
  no-permission: "<red>Bu komutu kullanma iznin yok!"
  player-only: "<red>Bu komut sadece oyuncular tarafından kullanılabilir."
```

---

## 12. ENTITY, BLOCK, WORLD API

### 12.1 Entity İşlemleri

```java
// Spawn
Wolf wolf = (Wolf) world.spawnEntity(location, EntityType.WOLF);
wolf.setOwner(player);
wolf.customName(Component.text("Karabaş", NamedTextColor.GOLD));
wolf.setCustomNameVisible(true);
wolf.getPersistentDataContainer().set(key, PersistentDataType.STRING, "pet");

// Spawn with Consumer (daha güvenli)
world.spawn(location, Zombie.class, zombie -> {
    zombie.customName(Component.text("Boss Zombie"));
    zombie.setHealth(100);
    zombie.getAttribute(Attribute.MAX_HEALTH).setBaseValue(100);
    zombie.setRemoveWhenFarAway(false);
    zombie.addPotionEffect(new PotionEffect(PotionEffectType.SPEED, -1, 2));
});

// Display Entity (1.19.4+) — dekoratif, client-side render
world.spawn(location, TextDisplay.class, display -> {
    display.text(Component.text("Merhaba!", NamedTextColor.GOLD));
    display.setBillboard(Display.Billboard.CENTER); // Oyuncuya dönük
    display.setBackgroundColor(Color.fromARGB(128, 0, 0, 0)); // Yarı saydam arka plan
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
    if (success) player.sendMessage(Component.text("Işınlandın!"));
});
```

### 12.2 Block İşlemleri

```java
Block block = world.getBlockAt(x, y, z);
block.setType(Material.DIAMOND_BLOCK);

// BlockData ile detaylı kontrol
if (block.getBlockData() instanceof Stairs stairs) {
    stairs.setFacing(BlockFace.NORTH);
    stairs.setHalf(Bisected.Half.TOP);
    block.setBlockData(stairs);
}

// Chunk işlemleri
Chunk chunk = location.getChunk();
chunk.load();       // Yükle
chunk.isLoaded();   // Yüklü mü?
// Paper: async chunk yükleme
world.getChunkAtAsync(x, z).thenAccept(loadedChunk -> {
    // Chunk yüklendi
});
```

### 12.3 World İşlemleri

```java
World world = Bukkit.getWorld("world");
world.setTime(6000);       // Öğle
world.setStorm(false);     // Yağmuru durdur
world.setGameRule(GameRule.KEEP_INVENTORY, true);
world.setDifficulty(Difficulty.HARD);

// Patlama
world.createExplosion(location, 4.0f, false, false); // fire=false, breakBlocks=false

// Ses
world.playSound(location, Sound.ENTITY_ENDER_DRAGON_GROWL, 
    SoundCategory.MASTER, 1.0f, 1.0f);

// Parçacık
world.spawnParticle(Particle.FLAME, location, 50, 0.5, 0.5, 0.5, 0.02);
// Paper gelişmiş parçacık API:
player.spawnParticle(Particle.DUST, location, 1,
    new Particle.DustOptions(Color.RED, 2.0f));
```

---

## 13. SCOREBOARD & TEAM API

```java
// Sidebar scoreboard
Scoreboard scoreboard = Bukkit.getScoreboardManager().getNewScoreboard();
Objective obj = scoreboard.registerNewObjective("sidebar", Criteria.DUMMY,
    Component.text("✦ SUNUCU ADI ✦", NamedTextColor.GOLD, TextDecoration.BOLD));
obj.setDisplaySlot(DisplaySlot.SIDEBAR);

// Satırlar (score = sıralama, yüksek = üstte)
obj.getScore("§a» Coin: §f100").setScore(5);
obj.getScore("§e» Seviye: §f10").setScore(4);
obj.getScore("§7─────────").setScore(3);
obj.getScore("§b» Online: §f" + Bukkit.getOnlinePlayers().size()).setScore(2);
obj.getScore("§dplay.sunucu.net").setScore(1);

player.setScoreboard(scoreboard);

// Team (tab list sıralama, collision, friendly fire)
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

### 14.1 NMS Erişimi (paperweight-userdev ile)

```java
// build.gradle.kts
plugins {
    id("io.papermc.paperweight.userdev") version "2.0.0-beta.19"
}
dependencies {
    paperweight.paperDevBundle("1.21.4-R0.1-SNAPSHOT")
}

// Kullanım — Mojang-mapped isimler (1.20.5+)
import net.minecraft.server.level.ServerPlayer;
import net.minecraft.network.protocol.game.*;
import org.bukkit.craftbukkit.entity.CraftPlayer;

ServerPlayer nmsPlayer = ((CraftPlayer) player).getHandle();
// Paket gönderme örneği:
nmsPlayer.connection.send(new ClientboundSetTitleTextPacket(
    net.minecraft.network.chat.Component.literal("NMS Title!")));
```

### 14.2 PacketEvents Kütüphanesi (Önerilen)

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
// Packet dinleme
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
                // Anti-cheat mantığı...
            }
        }

        @Override
        public void onPacketSend(PacketSendEvent event) {
            // Sunucudan client'a giden paketler
        }
    }
);
```

### 14.3 Plugin Messaging (BungeeCord/Velocity Channel)

```java
// Kayıt
getServer().getMessenger().registerOutgoingPluginChannel(this, "BungeeCord");
getServer().getMessenger().registerIncomingPluginChannel(this, "BungeeCord", 
    (channel, player, message) -> {
        ByteArrayDataInput in = ByteStreams.newDataInput(message);
        String subchannel = in.readUTF();
        // ...
    });

// Gönderme (oyuncuyu başka sunucuya yönlendirme)
ByteArrayDataOutput out = ByteStreams.newDataOutput();
out.writeUTF("Connect");
out.writeUTF("lobby");
player.sendPluginMessage(plugin, "BungeeCord", out.toByteArray());
```

---

## 15. RECIPE API

```java
// Shaped recipe (şekilli)
NamespacedKey key = new NamespacedKey(plugin, "super_pickaxe");
ShapedRecipe recipe = new ShapedRecipe(key, superPickaxe);
recipe.shape("DDD", " S ", " S ");
recipe.setIngredient('D', Material.DIAMOND_BLOCK);
recipe.setIngredient('S', Material.STICK);
Bukkit.addRecipe(recipe);

// Shapeless recipe (şekilsiz)
ShapelessRecipe shapeless = new ShapelessRecipe(
    new NamespacedKey(plugin, "compressed_diamond"), compressedDiamond);
shapeless.addIngredient(9, Material.DIAMOND);
Bukkit.addRecipe(shapeless);

// Furnace recipe
FurnaceRecipe furnace = new FurnaceRecipe(
    new NamespacedKey(plugin, "custom_smelt"), result,
    Material.RAW_IRON, 0.7f, 200); // xp, cookTime (tick)
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

// Plugin disable'da kaldır
Bukkit.removeRecipe(key);
```

---

## 16. REGISTRIES (1.21.2+, Deneysel)

```java
// PluginBootstrap'ta registry değişiklikleri
@Override
public void bootstrap(BootstrapContext context) {
    context.getLifecycleManager().registerEventHandler(
        LifecycleEvents.REGISTRY_MODIFICATION.forRegistry(RegistryKey.ENCHANTMENT),
        event -> {
            // Mevcut enchantment'ı değiştir
            // Yeni enchantment ekle
        }
    );
}
```

---

## 17. PERFORMANS EN İYİ UYGULAMALARI

### 17.1 Yapılması Gerekenler

- **PlayerMoveEvent**: Sadece blok değişimini kontrol et, her tick çağrılır
  ```java
  @EventHandler
  public void onMove(PlayerMoveEvent event) {
      if (event.getFrom().getBlockX() == event.getTo().getBlockX()
          && event.getFrom().getBlockY() == event.getTo().getBlockY()
          && event.getFrom().getBlockZ() == event.getTo().getBlockZ()) return;
      // Sadece blok değişince çalışır
  }
  ```
- **NamespacedKey'leri static final olarak sakla**, her seferinde new yapma
- **Config değerlerini cache'le**, her erişimde dosya okuma
- **Async I/O**: Veritabanı, HTTP, dosya işlemleri → `runTaskAsynchronously`
- **Collection seçimi**: Çok arama → `HashMap`/`HashSet`, sıralı → `LinkedHashMap`
- **Scoreboard**: Paket tabanlı kütüphane kullan (FastBoard gibi), Bukkit Scoreboard API her güncelleme paket atar
- **Chunk yükleme**: `getChunkAtAsync()` kullan, senkron chunk yükleme TPS düşürür
- **Hologram/NPC**: Display Entity veya paket tabanlı (PacketEvents) kullan
- **Connection pooling**: HikariCP ile veritabanı bağlantılarını yönet

### 17.2 Yapılmaması Gerekenler

- `Bukkit.getOnlinePlayers()` döngüsünde `for` içinde ağır işlem yapma
- Async thread'den Bukkit API çağırma (ConcurrentModificationException!)
- `EntityMoveEvent` kullanma (çok ağır) — alternatif: scheduler ile kontrol
- Her tick'te config dosyası okuma
- `getItemMeta()` gereksiz yere çağırma (snapshot oluşturur)
- `/reload` desteklemeye çalışma — Paper resmi olarak deprecated etti

---

## 18. PLACEHOLDERAPI ENTEGRASYONU

```java
// Expansion sınıfı
public class MyExpansion extends PlaceholderExpansion {
    private final MyPlugin plugin;

    public MyExpansion(MyPlugin plugin) { this.plugin = plugin; }

    @Override public String getIdentifier() { return "myplugin"; }
    @Override public String getAuthor() { return "AuthorName"; }
    @Override public String getVersion() { return plugin.getDescription().getVersion(); }
    @Override public boolean persist() { return true; } // /reload'da kaybolmasın
    @Override public boolean canRegister() { return true; }

    @Override
    public String onPlaceholderRequest(Player player, String params) {
        if (player == null) return "";
        return switch (params) {
            case "coins" -> String.valueOf(getCoins(player));
            case "level" -> String.valueOf(getLevel(player));
            case "rank" -> getRank(player);
            default -> null; // Bilinmeyen placeholder
        };
    }
}

// Kayıt (onEnable):
if (Bukkit.getPluginManager().isPluginEnabled("PlaceholderAPI")) {
    new MyExpansion(this).register();
}

// MiniMessage ile PlaceholderAPI kullanma:
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

## 19. PERMISSION SİSTEMİ (Vault / LuckPerms)

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
// Kullanım:
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
    .prompt(Component.text("Lütfen kaynak paketini indir!", NamedTextColor.YELLOW))
    .required(true) // Zorunlu
    .build());

// Event
@EventHandler
public void onResourcePackStatus(PlayerResourcePackStatusEvent event) {
    switch (event.getStatus()) {
        case ACCEPTED -> { /* İndirme başladı */ }
        case SUCCESSFULLY_LOADED -> { /* Başarılı */ }
        case DECLINED -> { player.kick(Component.text("Kaynak paketi zorunlu!")); }
        case FAILED_DOWNLOAD -> { player.kick(Component.text("İndirme başarısız!")); }
    }
}
```

---

## 21. DIALOG API (1.21.7+, Deneysel)

Paper 1.21.7 ile gelen yeni dialog sistemi — özel form/diyalog gösterme.

```java
// Dialog oluştur ve göster (API henüz deneysel, değişebilir)
// Güncel kullanım için: https://docs.papermc.io/paper/dev/dialogs/
```

---

## 22. YAPISAL PATTERN'LER

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

## 23. SIK YAPILAN HATALAR & ÇÖZÜMLER

| Hata | Sebep | Çözüm |
|------|-------|-------|
| `NoClassDefFoundError` at runtime | Bağımlılık shade edilmemiş | Shadow plugin ile fat JAR oluştur veya `libraries:` kullan |
| `Plugin is not associated with this PluginManager` | Farklı plugin loader'dan event kayıt | Doğru plugin instance'ı kullan |
| Item lore italic | Lore varsayılan olarak italic | `.decoration(TextDecoration.ITALIC, false)` ekle |
| `IllegalPluginAccessException` async'te | Async thread'den Bukkit API | `runTask()` ile ana thread'e dön |
| PDC verisi kayboluyor | `BlockState.update()` çağrılmamış | Block entity PDC sonrası `state.update()` çağır |
| Command tab-complete çalışmıyor | Eski Bukkit command sistemi | Brigadier API'ye geç |
| `api-version` eksik | plugin.yml'de belirtilmemiş | `api-version: '1.21.4'` ekle, yoksa legacy uyarısı verir |
| Config Türkçe karakter bozuk | Encoding sorunu | `filteringCharset = "UTF-8"` Gradle'da, config dosyaları UTF-8 |

---

## 24. PROJE YAPISI ÖNERİSİ

```
src/main/java/com/example/myplugin/
├── MyPlugin.java              # Ana sınıf (extends JavaPlugin)
├── MyBootstrap.java           # Paper plugin bootstrap (isteğe bağlı)
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
├── paper-plugin.yml           # Paper plugin kullanıyorsan
├── config.yml
└── lang/
    ├── tr.yml
    └── en.yml
```

---

## 25. HIZLI REFERANS — IMPORT'LAR

```java
// Paper API (her zaman)
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

## 26. REGISTRY API (Detaylı — 1.21.2+, Deneysel)

Registry'ler Minecraft'ın veri tabanıdır: enchantment, biome, damage type, potion effect vb.
Paper 1.21+ ile plugin'ler bootstrap aşamasında registry'lere yeni kayıt ekleyebilir veya
mevcut kayıtları değiştirebilir.

### 26.1 Registry'den Değer Okuma

```java
// Modern yöntem (1.21+)
Registry<Enchantment> enchReg = RegistryAccess.registryAccess()
    .getRegistry(RegistryKey.ENCHANTMENT);
Enchantment sharpness = enchReg.get(NamespacedKey.minecraft("sharpness"));

// TypedKey ile
TypedKey<Enchantment> sharpKey = TypedKey.create(
    RegistryKey.ENCHANTMENT, Key.key("minecraft:sharpness"));

// Tüm kayıtları dolaşma
for (Enchantment ench : enchReg) {
    Key key = enchReg.getKey(ench);
    // ...
}
```

### 26.2 Custom Enchantment Oluşturma (Bootstrap)

```java
// MyBootstrap.java (paper-plugin.yml gerektirir)
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

### 26.3 Mevcut Enchantment Değiştirme

```java
context.getLifecycleManager().registerEventHandler(
    RegistryEvents.ENCHANTMENT.entryAdd()
        .forKey(TypedKey.create(RegistryKey.ENCHANTMENT, Key.key("minecraft:sharpness")))
        .newHandler(event -> {
            event.builder().maxLevel(10); // Sharpness max level → 10
        })
);
```

### 26.4 Kullanılabilir RegistryKey'ler

```
RegistryKey.ENCHANTMENT          // Büyüler
RegistryKey.BIOME                // Biyomlar
RegistryKey.DAMAGE_TYPE          // Hasar türleri
RegistryKey.TRIM_MATERIAL        // Zırh trim malzemeleri
RegistryKey.TRIM_PATTERN         // Zırh trim desenleri
RegistryKey.BANNER_PATTERN       // Bayrak desenleri
RegistryKey.PAINTING_VARIANT     // Tablo varyantları
RegistryKey.INSTRUMENT           // Keçi boynuzu enstrümanları
RegistryKey.STRUCTURE            // Yapılar (⚠ client'ta yok, command arg olarak kullanma)
RegistryKey.WOLF_VARIANT         // Kurt varyantları
RegistryKey.JUKEBOX_SONG         // Müzik diskleri
RegistryKey.MENU                 // Menu türleri (1.21.2+)
RegistryKey.DIALOG               // Dialog türleri (1.21.7+)
```

### 26.5 RegistryKeySet & Tag

```java
// Belirli enchantment'ları grupla
RegistryKeySet<Enchantment> swordEnchants = RegistrySet.keySet(
    RegistryKey.ENCHANTMENT,
    TypedKey.create(RegistryKey.ENCHANTMENT, Key.key("minecraft:sharpness")),
    TypedKey.create(RegistryKey.ENCHANTMENT, Key.key("minecraft:smite")),
    TypedKey.create(RegistryKey.ENCHANTMENT, Key.key("myplugin:lifesteal"))
);

// Vanilla tag'lerini kullan (ItemTypeTagKeys, BlockTypeTagKeys, vb.)
// Tag'ler Minecraft wiki'de listelenmiştir
```

---

## 27. CUSTOM WORLD GENERATION

### 27.1 ChunkGenerator

```java
public class VoidWorldGenerator extends ChunkGenerator {

    // Noise aşaması — ana arazi şekli
    @Override
    public void generateNoise(WorldInfo worldInfo, Random random,
                              int chunkX, int chunkZ, ChunkData chunkData) {
        // Boş dünya — hiçbir şey yapma
        // Normal dünya için: chunkData.setBlock(x, y, z, Material.STONE);
    }

    // Yüzey aşaması — çim, kum vb.
    @Override
    public void generateSurface(WorldInfo worldInfo, Random random,
                                int chunkX, int chunkZ, ChunkData chunkData) {
        // Sadece spawn platformu
        if (chunkX == 0 && chunkZ == 0) {
            for (int x = 0; x < 16; x++) {
                for (int z = 0; z < 16; z++) {
                    chunkData.setBlock(x, 64, z, Material.BEDROCK);
                    chunkData.setBlock(x, 65, z, Material.GRASS_BLOCK);
                }
            }
        }
    }

    // Vanilla aşamalarını kontrol et
    @Override public boolean shouldGenerateNoise() { return false; }
    @Override public boolean shouldGenerateSurface() { return false; }
    @Override public boolean shouldGenerateCaves() { return false; }
    @Override public boolean shouldGenerateDecorations() { return false; }
    @Override public boolean shouldGenerateMobs() { return false; }
    @Override public boolean shouldGenerateStructures() { return false; }

    // Sabit spawn noktası
    @Override
    public Location getFixedSpawnLocation(World world, Random random) {
        return new Location(world, 8, 66, 8);
    }
}

// plugin.yml'de: generator: MyPlugin
// bukkit.yml'de veya WorldCreator ile:
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
        return Biome.PLAINS; // Her yer plains
    }

    @Override
    public List<Biome> getBiomes(WorldInfo worldInfo) {
        return List.of(Biome.PLAINS);
    }
}

// Kullanım:
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

        if (random.nextDouble() < 0.05) { // %5 ihtimalle ağaç
            Location loc = new Location(null, worldX, worldY + 1, worldZ);
            if (limitedRegion.isInRegion(loc)) {
                limitedRegion.generateTree(loc, random, TreeType.TREE);
            }
        }
    }
}

// ChunkGenerator'a ekle:
@Override
public List<BlockPopulator> getDefaultPopulators(World world) {
    return List.of(new TreePopulator());
}
```

---

## 28. ATTRIBUTE API (1.21.x Güncel)

```java
// Attribute okuma/değiştirme
AttributeInstance maxHealth = player.getAttribute(Attribute.MAX_HEALTH);
maxHealth.setBaseValue(40.0); // 20 kalp

// Modifier ekleme (1.21+ Key-based, UUID deprecated!)
AttributeModifier modifier = new AttributeModifier(
    Key.key("myplugin", "strength_boost"),  // Unique key
    10.0,                                    // Miktar
    AttributeModifier.Operation.ADD_NUMBER   // İşlem türü
);
maxHealth.addModifier(modifier);

// Modifier kaldırma
maxHealth.removeModifier(Key.key("myplugin", "strength_boost"));

// Item attribute modifier (Data Component API ile)
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

// Operation türleri:
// ADD_NUMBER      → base + amount
// ADD_SCALAR      → base * (1 + amount)  (yüzde artış)
// MULTIPLY_SCALAR_1 → toplam * (1 + amount) (tüm modifier'lardan sonra çarpma)

// Tüm Attribute'lar:
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
// Potion effect verme
player.addPotionEffect(new PotionEffect(
    PotionEffectType.SPEED,
    20 * 60,            // Süre (tick) — 60 saniye
    1,                  // Amplifier (0=Level I, 1=Level II)
    true,               // Ambient (beacon efekti gibi)
    true,               // Parçacık göster
    true                // İkon göster
));
// Sonsuz süre:
player.addPotionEffect(new PotionEffect(PotionEffectType.NIGHT_VISION, -1, 0));

// Effect kaldır
player.removePotionEffect(PotionEffectType.SPEED);
// Tüm efektleri kaldır
player.clearActivePotionEffects();

// Kontrol
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

## 30. PARTICLE API (Paper Gelişmiş)

```java
// Temel
world.spawnParticle(Particle.HEART, location, 5);

// Hız ve offset ile
world.spawnParticle(Particle.FLAME, location, 
    50,     // count
    0.5,    // offsetX
    0.5,    // offsetY
    0.5,    // offsetZ
    0.02    // extra (hız)
);

// Renkli dust
player.spawnParticle(Particle.DUST, location, 1,
    new Particle.DustOptions(Color.fromRGB(255, 100, 0), 2.0f));

// Dust transition (renk geçişi)
player.spawnParticle(Particle.DUST_COLOR_TRANSITION, location, 1,
    new Particle.DustTransition(
        Color.RED, Color.BLUE, 1.5f));

// Blok parçacığı
player.spawnParticle(Particle.BLOCK, location, 30,
    Material.DIAMOND_BLOCK.createBlockData());

// Item parçacığı
player.spawnParticle(Particle.ITEM, location, 10,
    new ItemStack(Material.GOLDEN_APPLE));

// Vibration (sculk sensör efekti)
player.spawnParticle(Particle.VIBRATION, location, 1,
    new Vibration(
        new Vibration.Destination.BlockDestination(targetLocation),
        20  // arrival tick
    ));

// Sadece belirli oyuncuya göster
player.spawnParticle(Particle.HEART, location, 5,
    0, 0, 0, 0, null);

// Daire çizme
for (double angle = 0; angle < Math.PI * 2; angle += Math.PI / 16) {
    double x = Math.cos(angle) * radius;
    double z = Math.sin(angle) * radius;
    Location point = center.clone().add(x, 0, z);
    world.spawnParticle(Particle.DUST, point, 1,
        new Particle.DustOptions(Color.PURPLE, 1.0f));
}

// Helix (sarmal)
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
border.setSize(1000);               // 1000x1000 alan
border.setSize(500, 60);            // 60 saniyede 500'e küçült
border.setDamageBuffer(5);          // 5 blok tampon (hasar başlamadan önce)
border.setDamageAmount(1.0);        // Saniyede 1 hasar
border.setWarningDistance(10);       // 10 blok uyarı mesafesi
border.setWarningTime(15);          // 15 saniye uyarı süresi

// Per-player world border (Paper)
player.setWorldBorder(
    world.getWorldBorder()  // Veya null ile varsayılana dön
);
```

---

## 32. MAP RENDERER

```java
public class CustomMapRenderer extends MapRenderer {
    @Override
    public void render(MapView map, MapCanvas canvas, Player player) {
        // Piksel boyama (0-127 x, 0-127 y)
        for (int x = 0; x < 128; x++) {
            for (int y = 0; y < 128; y++) {
                canvas.setPixel(x, y, MapPalette.matchColor(Color.BLUE));
            }
        }

        // Resim çizme
        BufferedImage image = ...; // 128x128 piksel
        canvas.drawImage(0, 0, image);

        // Metin
        canvas.drawText(10, 10, MinecraftFont.Font, "Merhaba!");
    }
}

// Kullanım
MapView mapView = Bukkit.createMap(world);
mapView.getRenderers().clear();
mapView.addRenderer(new CustomMapRenderer());
mapView.setTrackingPosition(false);

ItemStack map = ItemStack.of(Material.FILLED_MAP);
map.editMeta(MapMeta.class, meta -> meta.setMapView(mapView));
player.getInventory().addItem(map);
```

---

## 33. ÇOK DİLLİ PLUGIN PATTERN

```java
public class MessageManager {
    private final Map<String, YamlConfiguration> languages = new HashMap<>();
    private final String defaultLang = "tr";

    public void loadLanguages(JavaPlugin plugin) {
        for (String lang : List.of("tr", "en")) {
            plugin.saveResource("lang/" + lang + ".yml", false);
            File file = new File(plugin.getDataFolder(), "lang/" + lang + ".yml");
            languages.put(lang, YamlConfiguration.loadConfiguration(file));
        }
    }

    public Component getMessage(Player player, String key, TagResolver... resolvers) {
        String locale = player.locale().getLanguage(); // "tr", "en", vb.
        YamlConfiguration lang = languages.getOrDefault(locale, languages.get(defaultLang));
        String raw = lang.getString(key, "<red>Missing: " + key);
        return MiniMessage.miniMessage().deserialize(raw, resolvers);
    }

    public void send(Player player, String key, TagResolver... resolvers) {
        player.sendMessage(getMessage(player, key, resolvers));
    }
}

// lang/tr.yml:
// welcome: "<green>Hoşgeldin <player>! Sunucuya tekrar hoşgeldin."
// no-permission: "<red>Bu işlem için yetkin yok!"

// lang/en.yml:
// welcome: "<green>Welcome <player>! Welcome back to the server."
// no-permission: "<red>You don't have permission for this!"

// Kullanım:
messageManager.send(player, "welcome",
    Placeholder.component("player", player.displayName()));
```

---

## 34. LIFECYCLE API (Detaylı)

```java
// Lifecycle event'ler sunucunun yaşam döngüsüne bağlıdır
// ve reload-safe çalışır (her reload'da otomatik yeniden kayıt)

// Ana event türleri:
// LifecycleEvents.COMMANDS          → Command kayıtları
// RegistryEvents.<REGISTRY>.compose() → Yeni registry entry
// RegistryEvents.<REGISTRY>.entryAdd() → Mevcut entry değiştirme

// Bootstrap'tan (PluginBootstrap — sunucu başlamadan önce):
context.getLifecycleManager().registerEventHandler(
    LifecycleEvents.COMMANDS, event -> { /* ... */ });

// Plugin main class'tan (onEnable — sunucu açıkken):
this.getLifecycleManager().registerEventHandler(
    LifecycleEvents.COMMANDS, event -> { /* ... */ });

// ÖNEMLİ FARK:
// - Bootstrap'tan kaydedilen komutlar datapack function'larda çalışabilir
// - Plugin main class'tan kaydedilenler çalışamaz (plugin yüklenmesi daha geç)
```

---

## 35. ANTI-CHEAT / GÜVENLİK PATTERN'LERİ

```java
// Hareket doğrulama (temel)
@EventHandler(priority = EventPriority.LOWEST)
public void onMove(PlayerMoveEvent event) {
    Player player = event.getPlayer();
    Location from = event.getFrom();
    Location to = event.getTo();

    // Fly check (basit)
    if (!player.isFlying() && !player.getAllowFlight()
        && !player.getGameMode().equals(GameMode.SPECTATOR)) {
        double dy = to.getY() - from.getY();
        if (dy > 0 && !player.isOnGround() && player.getVelocity().getY() <= 0) {
            // Potansiyel fly — velocity check yap
            // DİKKAT: Elytra, riptide, slime block, bed bounce vb. false positive!
        }
    }

    // Speed check (basit)
    double distSq = Math.pow(to.getX() - from.getX(), 2)
                  + Math.pow(to.getZ() - from.getZ(), 2);
    double maxSpeed = getMaxAllowedSpeed(player); // Hız efekti, ice, soul sand vb.
    if (distSq > maxSpeed * maxSpeed) {
        // Potansiyel speed hack
        event.setCancelled(true); // Geri ışınla
    }
}

private double getMaxAllowedSpeed(Player player) {
    double base = player.isSprinting() ? 0.36 : 0.22; // Yaklaşık blok/tick
    PotionEffect speed = player.getPotionEffect(PotionEffectType.SPEED);
    if (speed != null) base *= 1 + (speed.getAmplifier() + 1) * 0.2;
    AttributeInstance attr = player.getAttribute(Attribute.MOVEMENT_SPEED);
    if (attr != null) base *= attr.getValue() / 0.1;
    return base * 1.25; // %25 tolerans
}

// Cooldown bazlı anti-spam
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
        if (distance > maxReach + 0.5) { // +0.5 tolerans
            event.setCancelled(true);
        }
    }
}
```

---

## 36. MINECRAFT 1.21.x SÜRÜM DEĞİŞİKLİKLERİ

### Protokol Sürümleri

```
1.21    → Protocol 767
1.21.1  → Protocol 767 (aynı)
1.21.2  → Protocol 768
1.21.3  → Protocol 768 (aynı)
1.21.4  → Protocol 769
1.21.5  → Protocol 770
```

### Sürümler Arası Önemli API Değişiklikleri

```
1.21 (Haziran 2024):
  • Trial Chamber, Breeze, Mace eklendi
  • Yeni enchantment'lar: Wind Burst, Density, Breach
  • Ominous events & Trial Spawner

1.21.2 (Eylül 2024):
  • Bundles
  • Yeni attribute'lar (TEMPT_RANGE)
  • Menu Type API (deneysel)

1.21.4 (Aralık 2024):
  • Paper hardforked from Spigot
  • Mojang-mapped runtime (CraftBukkit paketleri relocated değil!)
  • ItemStack.editPersistentDataContainer() eklendi
  • /reload resmi olarak deprecated

1.21.5 (Mart 2025):
  • Leaf Litter, Test Blocks
  • Protocol 770

1.21.7:
  • Dialog API (deneysel) — client'a özel form/diyalog gösterme
```

### paperweight-userdev Sürüm Uyumu

```
Paper 1.20.5+  → Mojang-mapped (CraftBukkit artık relocated değil)
Paper 1.21.4+  → Spigot'tan hardforked (Spigot patch'leri artık bağımsız)
                  paperweight 2.x kullanılmalı
Paper <1.20.5  → Spigot-mapped (eski obfuscation)
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
        // Event otomatik tetiklenir, mesajları kontrol et
        // player.assertSaid("Hoşgeldin Newcomer!");
    }
}
```

---

## 38. LOGGING EN İYİ UYGULAMALARI

```java
// Plugin logger kullan (java.util.logging)
getLogger().info("Plugin yüklendi!");
getLogger().warning("Config dosyası bulunamadı, varsayılan oluşturuluyor.");
getLogger().severe("Veritabanı bağlantısı başarısız: " + e.getMessage());

// ComponentLogger (Paper 1.19+) — MiniMessage destekli
getComponentLogger().info(Component.text("Yüklenen oyuncu: ")
    .append(Component.text(count, NamedTextColor.GREEN)));

// SLF4J (Paper bundled — önerilen yeni yöntem)
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
private static final Logger log = LoggerFactory.getLogger(MyPlugin.class);
log.info("Yüklendi");
log.warn("Uyarı: {}", message);  // Placeholder desteği
log.error("Hata", exception);     // Stacktrace otomatik eklenir

// DEBUG modunu config'e bağla
if (getConfig().getBoolean("debug", false)) {
    getLogger().info("[DEBUG] Detaylı bilgi...");
}
```

---

## 39. SERVİS SAĞLAYICI (ServiceManager)

```java
// Kendi API'ni diğer plugin'lere sun
public interface MyPluginAPI {
    int getPlayerLevel(Player player);
    void setPlayerLevel(Player player, int level);
}

// Kayıt (onEnable):
getServer().getServicesManager().register(
    MyPluginAPI.class, new MyPluginAPIImpl(),
    this, ServicePriority.Normal);

// Başka plugin'den erişim:
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
// Anonim BukkitRunnable
new BukkitRunnable() {
    int seconds = 10;
    @Override
    public void run() {
        if (seconds <= 0) {
            // Zaman doldu
            cancel();
            return;
        }
        player.sendActionBar(Component.text("Kalan: " + seconds + "s", NamedTextColor.RED));
        seconds--;
    }
}.runTaskTimer(plugin, 0L, 20L);

// CompletableFuture pattern (async veri + sync kullanım)
CompletableFuture.supplyAsync(() -> {
    // Async: Veritabanından veri çek
    return database.getPlayerData(uuid);
}).thenAccept(data -> {
    // Hala async thread'deyiz — ana thread'e geç
    Bukkit.getScheduler().runTask(plugin, () -> {
        player.sendMessage(Component.text("Veriler yüklendi!"));
    });
}).exceptionally(ex -> {
    plugin.getLogger().severe("Veri çekme hatası: " + ex.getMessage());
    return null;
});
```

---

## 41. DATAPACK ENTEGRASYONU

```java
// Lifecycle API ile datapack keşfi (Paper plugin gerektirir)
// paper-plugin.yml bootstrapper'ında:
context.getLifecycleManager().registerEventHandler(
    LifecycleEvents.DATAPACK_DISCOVERY, event -> {
        // Plugin JAR'ından datapack yükle
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

// Plugin JAR içinde datapack yapısı:
// src/main/resources/datapack/
//   pack.mcmeta
//   data/myplugin/recipe/...
//   data/myplugin/advancement/...
//   data/myplugin/loot_table/...
```

---

## 42. NBT OKUMA (Paper Convenience API)

```java
// 1.20.5+ — UnsafeValues aracılığıyla (gerçekten gerekmedikçe kullanma)
// PDC veya Data Component API tercih edilmeli

// ItemStack → SNBT (debug amaçlı)
// NMS ile:
import net.minecraft.nbt.CompoundTag;
import org.bukkit.craftbukkit.inventory.CraftItemStack;

net.minecraft.world.item.ItemStack nmsItem = CraftItemStack.asNMSCopy(stack);
CompoundTag tag = (CompoundTag) nmsItem.save(
    ((CraftWorld) world).getHandle().registryAccess());
String snbt = tag.toString();

// UYARI: NMS kullanımı paperweight-userdev gerektirir ve sürümler arası kırılabilir!
// Mümkünse her zaman PDC veya Data Component API kullan.
```

---

## 43. SENTRY / HATA İZLEME PATTERN

```java
// Global exception handler
Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> {
    plugin.getLogger().severe("Beklenmeyen hata [" + thread.getName() + "]: "
        + throwable.getMessage());
    throwable.printStackTrace();
});

// Event handler'larda güvenli sarmalama
public static void safeRun(JavaPlugin plugin, Runnable task) {
    try {
        task.run();
    } catch (Exception e) {
        plugin.getLogger().severe("Hata: " + e.getMessage());
        e.printStackTrace();
    }
}
```

---

## 44. YARATIK (MOB) AI & GOAL API (NMS)

```java
// NMS ile custom AI (paperweight-userdev gerektirir)
import net.minecraft.world.entity.ai.goal.*;
import net.minecraft.world.entity.monster.Zombie;
import org.bukkit.craftbukkit.entity.CraftZombie;

Zombie nmsZombie = ((CraftZombie) zombie).getHandle();
// Mevcut goal'ları temizle
nmsZombie.goalSelector.getAvailableGoals().clear();
// Yeni goal ekle
nmsZombie.goalSelector.addGoal(1, new FloatGoal(nmsZombie));
nmsZombie.goalSelector.addGoal(2, new MeleeAttackGoal(nmsZombie, 1.2, false));
nmsZombie.goalSelector.addGoal(3, new RandomStrollGoal(nmsZombie, 0.8));

// Paper API ile basit mob davranışı:
// Pathfinding
mob.getPathfinder().moveTo(targetLocation, 1.0); // hız
mob.getPathfinder().stopPathfinding();

// Hedef
mob.setTarget(player);  // LivingEntity hedef
mob.setAware(false);     // AI kapat
mob.setAI(false);        // Tamamen dondur
```

---

## 45. SINIRLAMA & BÖLGE KORUMA PATTERN

```java
// Basit WorldGuard-benzeri bölge sistemi
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

// Event'lerde kontrol
@EventHandler(priority = EventPriority.LOW)
public void onBlockBreak(BlockBreakEvent event) {
    for (Region region : regionManager.getRegions()) {
        if (region.contains(event.getBlock().getLocation())) {
            if (!region.getFlag("block-break", true)) {
                event.setCancelled(true);
                event.getPlayer().sendRichMessage("<red>Bu bölgede blok kıramazsın!");
                return;
            }
        }
    }
}
```

---

## 46. PROTOCOL DETAYLARI — HANDSHAKE & LOGIN AKIŞI

```
Client → Server bağlantı akışı:

1. HANDSHAKE (Paket 0x00)
   - Protocol Version (VarInt): 769 (1.21.4)
   - Server Address (String): play.example.net
   - Server Port (Unsigned Short): 25565
   - Next State (VarInt): 1=Status, 2=Login, 3=Transfer

2. LOGIN
   - Client → LoginStart (0x00): username, UUID
   - Server → EncryptionRequest (0x01): server ID, public key, verify token
   - Client → EncryptionResponse (0x02): shared secret, verify token
   - Server → LoginSuccess (0x02): UUID, username, properties
   - [Sıkıştırma etkinse: SetCompression paketi]

3. CONFIGURATION (1.20.2+)
   - Server → RegistryData, FeatureFlags, KnownPacks
   - Client → AcknowledgeFinish
   
4. PLAY
   - Server → JoinGame, ChunkData, PlayerInfo, vb.
   - Paketler artık çift yönlü akar

// Bu bilgi anti-bot, protocol analiz ve custom handshake
// doğrulama plugin'leri için kritiktir.
```

---

## 47. GRIMac / ANTI-CHEAT ENTEGRASYON PATTERN

```java
// GrimAC ile entegrasyon (event-based)
// ConditionalEvents veya Grim'in kendi event'leri üzerinden

// Grim flag event dinleme (3rd party API)
@EventHandler
public void onGrimFlag(FlagEvent event) {
    // Grim'in tespit ettiği hack türü
    String checkName = event.getCheckName();
    Player player = event.getPlayer();
    
    // Ceza sistemi
    int violations = getViolationCount(player, checkName);
    if (violations > 10) {
        player.kick(Component.text("Anti-Cheat: Şüpheli aktivite", NamedTextColor.RED));
    }
}

// Genel anti-cheat plugin uyumluluğu için:
// - Event'leri LOWEST priority'de dinle
// - Başka plugin iptal etmişse (ignoreCancelled=true) atla
// - Teleport/velocity event'lerinden sonra cooldown uygula
// - Elytra, riptide, slime block, piston vb. edge case'leri handle et
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
