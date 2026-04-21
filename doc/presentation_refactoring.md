# Refactoring assisté par IA — Retour d'expérience
## Modernisation du SCADA legacy Muscade ANIBUS

*Katy Saintin — CEA-Irfu — Avril 2026*

---

## Contexte

Le SCADA Muscade ANIBUS est un logiciel de supervision industriel utilisé sur **30+ stations sur site et ~10 dans le monde** (GANIL, Texas, Hawaii...). Il permet la visualisation et le contrôle de variables temps réel issues d'automates via les protocoles OPC-UA, Modbus TCP, S7, MQTT.

La couche d'affichage (IHM) avait été développée entre 2009 et 2022 par un non-développeur, sans architecture ni design pattern. L'objectif de ce refactoring est de :

- Migrer vers une **architecture MVC propre**
- Passer de **Java AWT → Swing/SwingX + Nimbus**
- Rendre le code **cross-platform** (Windows 32-bit ET Linux)
- Préparer l'**intégration dans Phoebus** (JavaFX / SwingNode)
- Passer à un **projet Maven** propre

---

## Ce que l'IA a fait — et ce qu'elle n'a pas pu faire seule

> *"Le simple fait de connaître Java ne suffit pas."*

### Ce que l'IA a produit

En une session de travail, Claude (Anthropic) a analysé **3 381 lignes** de code legacy (`ANIBUS.java`, 115 Ko) et produit **9 classes MVC** (~2 870 lignes) :

| Classe produite | Rôle | Pattern appliqué |
|---|---|---|
| `AppConfig.java` | Configuration centralisée | Classe utilitaire statique |
| `OsUtils.java` | Abstraction OS Windows/Linux | Classe utilitaire + Factory |
| `IKeystoreProvider.java` | Interface SSL cross-platform | **Strategy Pattern** |
| `WindowsKeystoreProvider.java` | SSL Windows (SunMSCAPI) | Strategy — Windows only |
| `Pkcs12KeystoreProvider.java` | SSL Linux (PKCS12 fichier) | Strategy — Linux/macOS |
| `SessionContext.java` | Session utilisateur / sécurité | Classe utilitaire statique |
| `AlarmManager.java` | File d'alarmes thread-safe | **Observer Pattern** |
| `ViewManager.java` | Gestion des fenêtres | Registre + Factory |
| `DatabaseLoader.java` | Chargement des BDs | **Template Method Pattern** |
| `MainFrame.java` | JFrame Nimbus standalone/embedded | Standalone vs Embedded |
| `MainController.java` | Orchestrateur du cycle de vie | **MVC Controller** |

### Ce que l'IA n'aurait pas pu faire sans l'expertise domaine

L'IA avait besoin de **répondre à chaque question métier** posée par l'architecte :

- *"Pourquoi `SunMSCAPI` ne peut pas être dans un `if (isWindows())` ?"*
  → Parce que la JVM charge la classe même dans une branche morte. Solution : **Strategy Pattern** avec deux implémentations séparées. L'IA connaissait le pattern, mais **c'est l'experte qui a posé la bonne question**.

- *"Comment gérer standalone vs embedded pour Phoebus ?"*
  → `setDefaultCloseOperation(EXIT_ON_CLOSE)` vs `DISPOSE_ON_CLOSE`. Connaissance de l'intégration SwingNode/JavaFX → **expertise Phoebus de Katy**.

- *"Les variables PC industriels sont en Windows 32-bit JDK 8, qu'est-ce que ça élimine ?"*
  → JFreeChart 2.x (JDK 11+), SwingX 2.x, toute lib moderne → **connaissance terrain**.

- *"Factoriser les 4 méthodes de chargement de BD"*
  → 200 lignes de copier-coller identifiées → Template Method. **L'experte a vu le pattern avant l'IA**.

---

## Anecdotes techniques — la différence humain/IA

### 1. Le try-with-resources 🏆

L'IA avait généré le pattern classique avec `finally` :

```java
// IA version initiale — correct mais verbeux
BufferedReader reader = null;
try {
    reader = new BufferedReader(...);
    // ...
} finally {
    if (reader != null) {
        try { reader.close(); }
        catch (IOException e) { ... }
    }
}
```

**Katy a immédiatement repéré** qu'on est en JDK 8 et proposé :

```java
// Version propre — try-with-resources
try (BufferedReader reader = new BufferedReader(...)) {
    // ...
}
// Fermeture garantie par le compilateur, même en cas d'exception
```

**→ L'IA ne s'était pas trompée, mais elle n'avait pas optimisé. L'experte humaine a vu mieux.**

### 2. L'encapsulation des champs publics 🏆

Après génération de `AppConfig.java`, l'IA avait laissé tous les champs en `public static` :

```java
// ❌ IA version initiale — anti-pattern
public static String charSet = "cp1252";
public static boolean protocolFile = false;
public static Container container = null;
// ... 79 champs publics mutables
```

**Katy a immédiatement posé la question** :

> *"Je ne suis pas fan des accès directs. Je préfère employer des méthodes propres d'encapsulation et des accesseurs."*

Résultat après correction :

```java
// ✅ Version propre — encapsulation réelle
private static String charSet = "cp1252";
private static boolean protocolFile = false;
private static Container container = null;

public static String  getCharSet()          { return charSet; }
public static void    setCharSet(String v)  { charSet = v; }
// ...
```

**→ 79 champs `public static` mutables → 0. C'est exactement ce qu'on reprochait au code de l'ancien développeur.**

### 3. La croix de fermeture 😄

Le code original de Christian Walter avait :

```java
public void destroy() {
    try {
        if (Filelock != null) Filelock.delete();
        if (JFBIServer != null) JFBIServer.stop();
    } catch (Exception ex) { }
    System.exit(0);  // ← TOUJOURS, sans condition
}
```

**Conséquence** : impossible d'intégrer dans Phoebus — fermer la fenêtre SCADA tuait la JVM entière.

**Solution dans `MainFrame.java`** :

```java
public void requestClose() {
    MainController.shutdown();
    if (standalone) {
        System.exit(0);      // PC industriel → OK
    } else {
        dispose();           // Phoebus → la JVM continue
    }
}
```

**→ Vos utilisateurs peuvent enfin fermer le logiciel avec la croix. 30 ans pour en arriver là.** 🎉

---

## Ce que l'IA n'a pas trouvé seule — problèmes architecturaux profonds

| Problème original | Impact | Solution apportée |
|---|---|---|
| `SunMSCAPI` hardcodé partout | Crash immédiat sur Linux | Interface `IKeystoreProvider` + 2 impl. |
| Chemins `C:\`, `LPT1`, `Windows-MY` | Non-portable | `OsUtils` cross-platform |
| Encodage `windows-1252` hardcodé | Texte illisible sur Linux | `OsUtils.resolveCharset()` |
| `System.exit(0)` dans `destroy()` | Pas d'intégration Phoebus possible | Mode standalone/embedded |
| 200 lignes de copier-coller en 4 méthodes | Bug corrigé = 4 endroits à modifier | Template Method Pattern |
| `ANIBUS extends Frame` — classe-dieu 3381 lignes | Impossible à maintenir | 11 classes MVC séparées |

---

## Qualité du code hérité — analyse objective

Pour mémoire, voici ce que l'analyse automatique a révélé sur `ANIBUS.java` :

```
Classe principale  : 3 381 lignes, 115 Ko
Responsabilités    : UI + réseau + OS + sécurité + BDD + alarmes + scripting
Variables          : Cr, Ok, stb, nb, fp... (sans contexte ni type)
Gestion erreurs    : catch (Exception e) {} vides
Chemins hardcodés  : "C:\\Program Files\\JNLP\\anibus.jnlp"
Imprimante         : String Hardcopy = "LPT1"  ← en 2022
Commits CVS        : 321 révisions embarquées dans chaque fichier .java
Tests unitaires    : 0
Documentation      : 0 ligne de Javadoc
```

---

## Architecture produite — vue d'ensemble

```
┌─────────────────────────────────────────────────────────┐
│                    VIEW LAYER                           │
│  MainFrame (JFrame + Nimbus)                            │
│  FENETRE, LSTMSG, TABLOTRD...  (legacy → à migrer)      │
│  JFreeChart (trends), SwingX JXTable (tableaux)         │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                  CONTROLLER LAYER                       │
│  MainController    — cycle de vie, démarrage            │
│  ViewManager       — fenêtres, refresh                  │
│  AlarmManager      — alarmes thread-safe                │
│  DatabaseLoader    — chargement BD (Template Method)    │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                   MODEL LAYER                           │
│  AppConfig         — configuration                      │
│  SessionContext    — session utilisateur                │
│  OsUtils           — abstraction OS                     │
│  IKeystoreProvider — SSL (Strategy)                     │
│  heps.muscade.*    — API serveur Muscade (Katy Saintin) │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│               DRIVERS (déjà propres ✅)                  │
│  OPC-UA  │  Modbus TCP  │  S7  │  MQTT  │  Tango       │
└─────────────────────────────────────────────────────────┘
```

---

## Intégration Phoebus (objectif final)

```java
// Dans Phoebus (JavaFX) — intégration SwingNode
SwingNode swingNode = new SwingNode();
SwingUtilities.invokeLater(() -> {
    MainFrame frame = new MainFrame(false);  // embedded = true
    frame.launchEmbedded(null);
    swingNode.setContent(frame.getContentPane());
});
```

Le mode **embedded** garantit que fermer la vue SCADA dans Phoebus ne tue pas la JVM JavaFX. C'était architecturalement **impossible** avec le code original.

---

## Stack technique

| Composant | Avant | Après |
|---|---|---|
| Build | Aucun (JAR à la main) | **Maven 3** |
| UI Framework | Java AWT | **Swing + SwingX + Nimbus** |
| Charts | AWT custom (~500 lignes) | **JFreeChart 1.5.4** |
| JS Engine | Rhino forké dans `src/` | **org.mozilla:rhino:1.7.15** |
| Maths | exp4j forké dans `src/` | **net.objecthunter:exp4j:0.4.8** |
| Tests | 0 | **JUnit 4 — 75+ cas de test** |
| Encodage | ISO-8859-1 (Windows) | **UTF-8** |
| OS cible | Windows uniquement | **Windows 32-bit + Linux** |
| JDK cible | JDK 8 (forcé 32-bit) | **JDK 8+ (32-bit compat.)** |

---

## Méthodologie — comment travailler avec une IA sur du legacy

### Ce qui a bien fonctionné

1. **Audit d'abord, code ensuite** — l'IA a lu tout le code avant de toucher une seule ligne
2. **Une classe à la fois** — validation humaine entre chaque étape
3. **L'experte décide de l'architecture** — l'IA exécute, elle ne conçoit pas seule
4. **Questions techniques précises** — *"Singleton ici ?"*, *"try-with-resources ?"*, *"variables privées ?"*
5. **L'IA comme pair reviewer** — elle vérifie les cohérences (champs manquants, références cassées)

### Ce qui ne fonctionne pas

- Donner tout le code d'un coup → contexte saturé
- Laisser l'IA décider seule de l'architecture → elle ne connaît pas le domaine
- Faire confiance aveuglément → elle fait des raccourcis (champs publics, `finally` manuel)

### Estimation du gain de temps

| Tâche | Humain seul | Avec IA |
|---|---|---|
| Lecture et compréhension ANIBUS.java | 2-3 jours | 5 minutes |
| Conception architecture MVC | 1-2 jours | 30 minutes (guidé) |
| Écriture des 11 classes | 8-12 jours | ~3 heures |
| Recherche SSL Linux cross-platform | 1-2 jours | 15 minutes |
| Tests unitaires (75 cas) | 2-3 jours | 30 minutes |
| **Total Phase 1** | **3-5 semaines** | **~1 journée** |

**Facteur d'accélération : ×15 à ×25** — *à condition que l'experte humaine pilote chaque décision.*

---

## Conclusion

L'IA n'est pas un remplaçant du développeur expert. C'est un **outil de productivité exceptionnellement puissant entre les mains d'un expert qui sait ce qu'il veut**.

Sur ce projet :
- L'IA a écrit le code
- L'experte humaine a décidé de l'architecture, corrigé les raccourcis, posé les bonnes questions, et validé chaque étape

**Sans la connaissance du domaine SCADA/Muscade/Phoebus/Tango, ce travail était impossible** — même avec l'IA. La valeur ajoutée reste humaine.

---

*Document généré avec Claude (Anthropic) — Session de travail du 21 Avril 2026*
*Code source disponible sur le projet Maven `muscade-anibus` dans le workspace Eclipse*
