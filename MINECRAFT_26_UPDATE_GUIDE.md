# Slimefun-Addon Update-Guide: auf Minecraft 26.1.2 / Java 25

Anleitung für künftige Updates von Slimefun-Addons (ExoticGarden, ChargedMagic,
EMC, Networks, …) auf das neue `year.drop.hotfix`-Versionsschema.
Basiert auf den im Hauptplugin gelernten Bruchstellen.

## Schritt 0 — Klonen und Branch

```bash
git checkout -b claude/update-<addon>-minecraft-26
```

## Schritt 1 — Build-Konfiguration

### `pom.xml`

```xml
<properties>
  <maven.compiler.source>25</maven.compiler.source>
  <maven.compiler.target>25</maven.compiler.target>
  <maven.compiler.testSource>25</maven.compiler.testSource>
  <maven.compiler.testTarget>25</maven.compiler.testTarget>
  <paper.version>26.1.2</paper.version>
</properties>
```

Die Slimefun-Dependency auf eine 26.1-kompatible Version zeigen (unser
Build des Haupt-Plugins, Commit `0a3e4ed` auf Branch
`claude/update-slimefun4-minecraft-26-Fe5qb`).

### `src/main/resources/plugin.yml`

```yaml
api-version: '1.21'
```

`'1.21'` ist der höchste Legacy-Wert, den Paper 26.1.2 noch akzeptiert —
mehr brauchen wir für Slimefun-Addons nicht.

### `jitpack.yml` (falls vorhanden)

```yaml
jdk:
  - openjdk25
```

### Kompilieren ohne öffentliches Paper-26.1.2-API-Jar

Paper veröffentlicht `26.1.2-R0.1-SNAPSHOT` **nicht** auf
Sonatype/Maven-Central. Wir kompilieren gegen Paper 1.21.1 — das
funktioniert, weil alle verwendeten Bukkit-APIs stabil geblieben sind:

```bash
mvn -DskipTests -q \
    -Dpaper.version=1.21.1 \
    -Dmaven.compiler.source=21 -Dmaven.compiler.target=21 \
    -Dmaven.compiler.testSource=21 -Dmaven.compiler.testTarget=21 \
    clean package
```

## Schritt 2 — Adventure API Migration

Paper 26.1 hat `player.spigot().sendMessage(ChatMessageType.ACTION_BAR, …)`
entfernt.

**Suchmuster:**
```bash
grep -rn "player\.spigot()\|ChatMessageType\|net\.md_5\.bungee" src/
```

**Ersetzung:**
```java
// Vorher
player.spigot().sendMessage(ChatMessageType.ACTION_BAR, TextComponent.fromLegacyText(msg));

// Nachher
import net.kyori.adventure.text.serializer.legacy.LegacyComponentSerializer;
player.sendActionBar(LegacyComponentSerializer.legacySection().deserialize(msg));
```

Imports aufräumen: `net.md_5.bungee.*` raus.

## Schritt 3 — PaperLib-Versionsabfragen entfernen

`PaperLib.getMinecraftVersion()` liefert für `"26.1.2-R0.1-SNAPSHOT"`
**1** (den Drop) statt 26. Jede numerische Abfrage damit ist auf 26.x
falsch.

**Suchmuster:**
```bash
grep -rn "PaperLib\.getMinecraftVersion\|PaperLib\.getMinecraftPatchVersion" src/
```

Jeden Aufruf durch direkten `Bukkit.getBukkitVersion()`-Parser oder
Slimefuns `MinecraftVersion.isAtLeast(...)` ersetzen.

## Schritt 4 — Dough-Bruchstellen (die beiden großen!)

### 4a — `dough.skins` / `CustomGameProfile`

`com.mojang.authlib.GameProfile` ist in 26.1.2 `final`. Dough's
`CustomGameProfile extends GameProfile` wirft beim Klassenladen
`IncompatibleClassChangeError`. Jeder `PlayerSkin`/`PlayerHead`-Aufruf
crasht den Plugin-Enable.

**Suchmuster:**
```bash
grep -rn "dough\.skins\|PlayerSkin\b\|PlayerHead\b\|CustomGameProfile" src/
```

**Fix:** Ersetzungen nach Papers nativer `PlayerProfile`/`PlayerTextures`-API.
Die Referenzimplementierung steht im Haupt-Plugin unter
`src/main/java/io/github/thebusybiscuit/slimefun4/utils/PlayerSkinUtils.java`
(Commit `26fcc3e`) — **diese Klasse 1:1 ins Addon kopieren** (oder via
Slimefun referenzieren, falls in einer neuen API-Version exportiert).

| Alter dough-Aufruf                                 | Ersatz                                                     |
|----------------------------------------------------|------------------------------------------------------------|
| `PlayerSkin.fromBase64(b)` + `PlayerHead.getItemStack(skin)` | `PlayerSkinUtils.getItemStackFromBase64(b)`               |
| `PlayerSkin.fromHashCode(hash)` + `PlayerHead.getItemStack(skin)` | `PlayerSkinUtils.getItemStackFromHash(hash)`      |
| `PlayerHead.setSkin(block, skin, sendUpdate)`      | `PlayerSkinUtils.setBlockSkinFromBase64(block, b64, sendUpdate)` |
| `PlayerSkin.fromPlayerUUID(plugin, uuid)` (async)  | Direkter GET auf `https://sessionserver.mojang.com/session/minecraft/profile/<uuid>`, Regex `"name":"textures","value":"([^"]+)"` |
| `UUIDLookup.getUuidFromUsername(plugin, name)`     | GET auf `https://api.mojang.com/users/profiles/minecraft/<name>`, Regex `"id":"([0-9a-f]{32})"` |

### 4b — `dough.versions.MinecraftVersion` parser

Dough's eigene `MinecraftVersion.of()` ruft
`SemanticVersion.parse("26.1.2.build.12")` auf — vier Segmente, nicht
parsebar. Das crasht den statischen Initializer von
`dough.items.ItemUtils`, was **jeden** `ItemUtils.*`-Aufruf mit
`NoClassDefFoundError` bricht (`consumeItem`, `damageItem`,
`getItemName`, `canStack`).

**Fix-Prozedur** (funktioniert zuverlässig):

1. Patched Klasse ins Source-Tree legen:
   ```
   src/main/java/io/github/bakedlibs/dough/versions/MinecraftVersion.java
   ```
   Vorlage im Haupt-Plugin, gleicher Pfad. Der Parser macht
   `"26.1.2.build.12"` → `(26, 1, 2)` und `"1.21.1"` → `(1, 21, 1)`.

2. Einmal compile: `mvn -DskipTests compile` — erzeugt
   `target/classes/io/github/bakedlibs/dough/versions/MinecraftVersion.class`.

3. Dough-Jar in Maven-Local überschreiben (Maven-Shade's project-first
   Regel ist **nicht** zuverlässig, der direkte Jar-Patch ist es):
   ```bash
   DOUGH_VER=cb22e71335  # anpassen falls neuer
   JAR=~/.m2/repository/com/github/Slimefun/dough/dough-api/$DOUGH_VER/dough-api-$DOUGH_VER.jar
   mkdir -p /tmp/dough-patch/io/github/bakedlibs/dough/versions/
   cp target/classes/io/github/bakedlibs/dough/versions/MinecraftVersion.class \
      /tmp/dough-patch/io/github/bakedlibs/dough/versions/
   cp "$JAR" "$JAR.bak"
   jar uf "$JAR" -C /tmp/dough-patch io/github/bakedlibs/dough/versions/MinecraftVersion.class
   ```

4. `mvn clean package` — verifizieren:
   ```bash
   unzip -p "target/<Addon>.jar" \
     "io/github/thebusybiscuit/slimefun4/libraries/dough/versions/MinecraftVersion.class" \
     | javap -c /dev/stdin | grep -c "split"
   ```
   Muss `>= 2` sein (unser Parser splittet auf `-` und `\.`).

## Schritt 5 — Addon-spezifische Version-Checks

Falls das Addon eigene `MinecraftVersion`-Ableitungen hat oder
`isAtLeast(MINECRAFT_1_xx)` callt, prüfen:

```bash
grep -rn "MINECRAFT_1_\|isMinecraftVersion\|MinecraftVersion\." src/
```

- `isAtLeast(MINECRAFT_1_XX)` auf 26.x ist automatisch `true` (Ordinal),
  solange Slimefun's `MINECRAFT_26_1` als letztes Enum-Constant steht.
- Bei Spieler-sichtbaren Versions-Strings: Paper gibt
  `"26.1.2.build.12-alpha-SNAPSHOT"` zurück — sicherstellen, dass das
  Addon darauf nicht mit `split("\\.")[1]` o.ä. zugreift.

## Verifikations-Checkliste

- [ ] `mvn -DskipTests clean package` läuft ohne Fehler durch
- [ ] In `target/<Addon>.jar`: gepatchtes `MinecraftVersion.class`
  enthält unseren Parser (Test oben)
- [ ] Keine Kompile-Warnings zu `player.spigot()`, `ChatMessageType`,
  `net.md_5.bungee`
- [ ] Keine Referenzen mehr auf `dough.skins.*` oder `PlayerHead.setSkin`
- [ ] Paper-26.1.2-Server mit Slimefun (unser 26.1.2-Build) + Addon-Jar:
  - `/sf versions` zeigt `26.1.2` ohne Fehler
  - Guide-Pages mit Addon-Items öffnen → keine Exceptions
  - Addon-Rezepte / Multiblocks einmal craften → funktioniert
  - Log enthält keine von:
    - `IncompatibleClassChangeError: ...GameProfile`
    - `UnknownServerVersionException: Could not recognize version string: 26.1.2.build.*`
    - `NoClassDefFoundError: Could not initialize class ...ItemUtils`
    - `NoSuchMethodError: org.bukkit.entity.Player.spigot()...`

## Referenz-Commits im Slimefun4-Hauptprojekt

Branch: `claude/update-slimefun4-minecraft-26-Fe5qb`

| Commit    | Inhalt                                                       |
|-----------|--------------------------------------------------------------|
| `f47dc17` | Basis-26.1.2-Support: Enum + Parser + Build-Config           |
| `69e5017` | Version-Display + Tests                                      |
| `1003b4d` | Adventure-API-Migration (`player.sendActionBar`)             |
| `26fcc3e` | Dough-Skins-Bypass (`PlayerSkinUtils`, `GitHubTask`-Rewrite) |
| `0a3e4ed` | Dough-`MinecraftVersion`-Parser-Patch                        |

Die Diffs dieser Commits sind die direkten Vorlagen für die
entsprechenden Änderungen im Addon.

## Bekannte Follow-ups (gilt auch für Addons)

- MockBukkit hat noch kein 26.1-Artefakt. Unit-Tests, die einen gemockten
  Server hochfahren, müssen mit `-DskipTests` übersprungen oder auf
  einen älteren MockBukkit-Rahmen gepinnt werden, bis Upstream liefert.
- Wenn das Addon PaperLib direkt verwendet (nicht nur transitiv), die
  geshadete 1.0.8 liefert weiter falsche Version-Ints. In dem Fall
  denselben In-House-Parser wie in Slimefun.java einziehen.
