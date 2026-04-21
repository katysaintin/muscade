# Refactoring avec Claude AI

20 avril 

Est-ce tu sais faire du refactoring de code java ?

Claude a répondu : Oui, absolument !Oui, absolument ! Je peux t'aider avec le refactoring de code Java. Voici ce que je sais faire :
Améliorations structurelles

Extraction de méthodes, classes ou interfaces
Renommage pour améliorer la lisibilité
Élimination du code dupliqué (principe DRY)
Réorganisation des responsabilités (principe SRP)

Patterns et bonnes pratiques

Application des design patterns (Factory, Builder, Strategy, etc.)
Respect des principes SOLID
Amélioration de la gestion des exceptions
Optimisation des collections et streams Java

Modernisation du code

Migration vers des fonctionnalités Java modernes (lambdas, streams, records, etc.)
Remplacement de code verbeux par des alternatives concises
Amélioration de la gestion de la nullité (Optional, etc.)

Lisibilité et maintenabilité

Simplification de conditions complexes
Réduction de la complexité cyclomatique
Amélioration des noms de variables/méthodes/classes

Envoie-moi ton code et dis-moi ce que tu veux améliorer (ou je peux analyser et proposer des améliorations moi-même) !

Après envoie du zip

TON ARCHITECTURE CIBLE :

┌─────────────────────────────────────────┐
│              VIEW (à refactorer)         │
│   AWT → Swing/SwingX + Nimbus LAF       │
│   (propre, découplée, sans logique)     │
└──────────────┬──────────────────────────┘
               │ interfaces
┌──────────────▼──────────────────────────┐
│           CONTROLLER                    │
│   (binding View ↔ Model)               │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│         TON MODEL (déjà propre ✅)       │
│   Interface DataSource / Variable API   │
├─────────────────────────────────────────┤
│  OPC-UA │ Modbus TCP │ S7 │ MQTT ...    │
│  (tes drivers, déjà Maven ✅)           │
└─────────────────────────────────────────┘


Le vrai contexte maintenant : SITUATION RÉELLE :

PC Industriels (Windows 32 bits, vieux)
    → JDK 8 32 bits (seule option viable)
    → SCADA legacy AWT/Swing (à refactorer)
    → Temporaire le temps de migrer les users

PC modernes / Linux
    → OpenJDK 21
    → Phoebus (ton fork ✅)
    → Sources : OPC-UA, Modbus, S7, MQTT, Muscade ✅


🔬 Audit du code source — Résultats
📊 Inventaire global
CatégorieFichiersNotesFichiers .java total592Forks à virer~237org.mozilla.javascript + net.objecthunter.exp4jCode métier réel~355Ce qu'on va travaillerClasse monstreANIBUS.java3381 lignes, 115 Ko 😱

📦 Packages identifiés
PackageFichiersRôleActionmuscade/anibus290UI + tout le reste mélangéRefactoring principalfbi/*38Protocole réseau/commandes→ Model/Servicejbarchiv13Archivage historique→ Modelprocedure8Scripting JavaScript→ Service isolécea/sslsocketfactory4SSL/TLS connexion→ Infrastructurecwzip1Zip utilitaire→ Util

🔄 Librairies forkées → Maven
Fork présentDépendance MavenVersion JDK8 okorg/mozilla/javascriptorg.mozilla:rhino:1.7.14✅net/objecthunter/exp4jnet.objecthunter:exp4j:0.4.8✅

🪟 Code Windows à isoler (18 fichiers)
ANIBUS.java, FENETRE.java, LOADCFG.java, MACFILE.java, SocketFactoryMain.java et 13 autres — tous avec Runtime.exec, SunMSCAPI, Windows-MY, charset windows-1252 hardcodé...

📐 Composants graphiques à migrer

30 fichiers avec Graphics.draw*() pur AWT → JPanel + painting Swing
16 fichiers avec Button, TextField, Label AWT → JButton, JTextField...
Charts/Plots : COURBE.java, TREND.java, BARGRAF.java, NB_DIALTRD.java (63Ko!) → JFreeChart
Nombreux DIAL*.java = dialogues → JDialog + Nimbus


🏗️ Architecture MVC cible proposée
com.muscade.scada
├── model/
│   ├── api/          ← IDataSource, IVariable (tes interfaces)
│   ├── archive/      ← jbarchiv refactoré
│   └── alarm/        ← AlarmHandler, etc.
├── view/
│   ├── main/         ← ANIBUS → MainFrame (JFrame + Nimbus)
│   ├── trend/        ← TREND, COURBE → JFreeChart
│   ├── table/        ← TABLOTRD, TABLODAT → JXTable
│   ├── alarm/        ← LSTMSG, LSTSURV
│   └── dialog/       ← tous les DIAL* → JDialog
├── controller/
│   ├── MainController.java
│   └── ...
├── service/
│   ├── script/       ← JavaScript/Rhino isolé
│   └── archive/      ← jbarchiv
└── infrastructure/
    ├── os/           ← OsUtils (Windows/Linux abstraction)
    └── ssl/          ← SocketFactory

⚠️ Encodage
Tous les fichiers sont en ISO-8859-1 (French Windows). → Migration vers UTF-8 dans le projet Maven.

🏗️ Plan de refactoring — ANIBUS.java (3381 lignes, 8 responsabilités mélangées)
Ce que contient ANIBUS.java aujourd'hui
ANIBUS extends Frame  ← déjà un problème : la classe principale EST une Frame AWT
    ├── 1. Configuration application (params JNLP, XML, charset, paths)
    ├── 2. Détection OS (Windows/Linux/Mac) → à isoler dans OsUtils
    ├── 3. Gestion SSL/Sécurité (SunMSCAPI, Windows-MY keystore)
    ├── 4. Gestion utilisateurs (UserName, SuperUser, password)
    ├── 5. Chargement base de données (LoadBD_ANA, LoadBD_TOR, LoadBD)
    ├── 6. Gestion alarmes (AddMsg, Ack, ClearMsg...)
    ├── 7. Gestion vues/fenêtres (AddView, RepaintAllViews, FENETRE)
    └── 8. main() + actionPerformed() + run()  ← tout dans la même classe

Architecture cible — découpage en classes

MainFrame.java          ← JFrame + Nimbus (remplace ANIBUS extends Frame)
    │
    ├── AppConfig.java          ← tous les static String/int de config
    ├── OsUtils.java            ← détection OS + paths Windows/Linux
    ├── SessionContext.java     ← User, SuperUser, Password, SecurityLevel
    ├── AlarmManager.java       ← AddMsg, Ack, ClearMsg, LSTMSG
    ├── ViewManager.java        ← AddView, RepaintAllViews, openFenetre
    ├── DatabaseLoader.java     ← LoadBD_ANA, LoadBD_TOR, LoadBD
    └── MainController.java     ← run(), init(), actionPerformed()

Ordre de création (une classe à la fois, tu valides)

AppConfig.java — le plus simple, zéro logique, que des constantes/config
OsUtils.java — isolation Windows/Linux
SessionContext.java — utilisateurs/sécurité
AlarmManager.java — alarmes
ViewManager.java — gestion vues
DatabaseLoader.java — chargement BD
MainFrame.java — la nouvelle JFrame Nimbus propre
MainController.java — le chef d'orchestre
-------------------------
AppConfig.java — 330 lignes, propre. Voici ce qui a été fait :
Extraction depuis ANIBUS.java :

Tous les static champs dispersés sur ~200 lignes → regroupés en sections logiques
static final pour les vraies constantes (ports, timeouts, codes alarmes, palettes)
Champs mutables runtime séparés clairement des constantes
Button ViewList, Connexion... → non repris car ce sont des widgets UI, ils iront dans MainFrame.java
FENETRE MainImage, ACQUIS TacheAcquis, CWBASE BaseD → non repris car ce sont des dépendances vers d'autres classes, elles iront dans le Controller

Choix de design :

Classe final avec constructeur privé → utilitaire pur, non instanciable
Nommage camelCase propre (socketPort au lieu de SocketPort)
Commentaires en anglais sur chaque champ
@SuppressWarnings("rawtypes") sur les Vector/HashMap non typés legacy — on les gardera pour l'instant pour ne pas casser la compatibilité, on les typera lors du refactoring suivant
Une instruction par ligne ✅
-------------------------
OsUtils.java — 300 lignes, voici le bilan de ce qui a été fait :
Ce qui a été extrait et unifié :
Code original Où c'était Dans OsUtilsos.name + MAC_OS/LINUX booleansANIBUS.java ligne 1194isWindows(),
isLinux(), isMac()USERPROFILE env var WindowsGetEnv.getUserHome()getUserHome()LOCALAPPDATA\Temp WindowsGetEnv.getTmpDir()getTmpDir()
user.home + muscade\GetEnv.getWorkDir()getWorkDir()
Chemin .certs Windows vs LinuxGetEnv.getUserCertsFile()getUserCertsFile() +
getUserCertsFileLocal()SunMSCAPI / Windows-MY hardcodésSocketFactoryMain.javagetDefaultKeystoreType()
+ getDefaultSecurityProvider()Chemin.replace("\\", "/")ANIBUS.java ligne 1858toUrlPath()
+ ensureTrailingSeparator()Lock file dans TmpDirANIBUS.java ligne 3275getLockFile()Charset.defaultCharset() + fallbackANIBUS.java ligne 1325resolveCharset()

Points clés :

GetEnv.java est maintenant entièrement absorbée — elle pourra être supprimée
getDefaultKeystoreType() retourne "Windows-MY" sur Windows, "PKCS12" sur Linux → cross-platform SSL propre
logEnvironment() remplace les dizaines de System.out.println de debug dispersés
Détection OS en static {} → une seule fois au démarrage, zéro overhead
-------------------------
Pour SSL/Certificats → Interface + 2 implémentations
C'est là que la différence est fondamentale et métier :
IKeystoreProvider  (interface)
    │
    ├── WindowsKeystoreProvider
    │     ├── type     = "Windows-MY"
    │     ├── provider = "SunMSCAPI"  (sun.security.mscapi — Windows only !)
    │     └── accède au store Windows natif (gestionnaire de certificats Windows)
    │
    └── Pkcs12KeystoreProvider
          ├── type     = "PKCS12"
          ├── provider = "SunJSSE"   (standard JDK)
          └── accède à un fichier .p12 / .pfx sur disque

Pourquoi c'est obligatoire ici et pas juste un if :

SunMSCAPI est une classe qui n'existe pas sur Linux — si tu l'instancies même dans un if (isWindows()), la JVM plante au classloading sur Linux
Le comportement est fondamentalement différent : Windows délègue au gestionnaire OS, Linux lit un fichier
C'est exactement le Strategy Pattern — même interface, comportement interchangeable à l'initialisation

Sur Linux en pratique pour ton SCADA :

Tu crées un fichier .p12 avec keytool (fourni avec JDK 8)
Tu le déposes dans ~/.muscade/security/muscade.p12
Pkcs12KeystoreProvider le lit avec le mot de passe configuré
C'est standard, portable, bien documenté


Schéma final proposé
OsUtils.java                    ← classe statique (détection + chemins)
    │
    └── crée au démarrage ──→  IKeystoreProvider  (interface)
                                    │
                            OsUtils.createKeystoreProvider()
                                    │
                          ┌─────────┴──────────┐
                          │                    │
              WindowsKeystoreProvider    Pkcs12KeystoreProvider
              (Windows-MY + SunMSCAPI)   (fichier .p12 + SunJSSE)
OsUtils.createKeystoreProvider() fait le if isWindows() une seule fois au démarrage et retourne la bonne implémentation. Tout le reste du code ne voit que IKeystoreProvider.

On génère IKeystoreProvider + les 2 implémentations maintenant, ou tu veux d'abord continuer avec SessionContext.java et on revient sur SSL après ? Les deux sont indépendants. 🎯

J'ai tout ce qu'il faut. Je génère les 3 fichiers : IKeystoreProvider, WindowsKeystoreProvider, Pkcs12KeystoreProvider, puis OsUtils remanié.3 fichiers créés3 fichiers créésMaintenant OsUtils remanié — plus léger, sans la logique SSL (déléguée aux providers), avec la factory createKeystoreProvider().2 fichiers créés, exécuté 2 commandes, lu un fichier2 fichiers créés, exécuté 2 commandes, lu un fichierVoilà les 4 fichiers. Récap de ce qui a été produit et pourquoi :

Bilan architectural
IKeystoreProvider — l'interface
Définit le contrat commun : openKeystore(), buildSslContext(), installDefaultSocketFactory(). Tout le code appelant ne connaît que cette interface.
WindowsKeystoreProvider — Windows uniquement

Type Windows-MY, provider SunMSCAPI
Accès au gestionnaire de certificats Windows natif
Jamais chargé sur Linux — la JVM ne résout SunMSCAPI que si cette classe est instanciée

Pkcs12KeystoreProvider — Linux / macOS / portable

Type PKCS12, provider SunJSSE (standard JDK 8)
Lit un fichier .p12 dans ~/.muscade/security/muscade-client.p12
Le commentaire Javadoc en tête explique exactement comment créer ce .p12 avec openssl — tu pourras donner ça directement à tes admins Linux

OsUtils remanié

Plus léger : toute la logique SSL en est sortie
createKeystoreProvider() = le seul if isWindows() de toute l'application pour le SSL
GetEnv.java est maintenant entièrement remplacée et peut être supprimée

Usage dans le code après refactoring
java// Au démarrage (MainController) — une seule fois :
IKeystoreProvider keystoreProvider = OsUtils.createKeystoreProvider(
    AppConfig.p12FilePath,   // null = chemin par défaut
    AppConfig.certAlias      // null = auto-sélection
);
keystoreProvider.installDefaultSocketFactory(alias, password, gui);

// Partout ailleurs : zero if/else sur l'OS pour le SSL
-----------------------------

SessionContext.java — 250 lignes. Voici ce qui a été extrait et les décisions prises :
Extrait de ANIBUS.java :
OriginalNouveauNoteUserNameosUserNameRenommé pour distinguer du login OS vs login MuscadeUseruserAvec sa logique de trailing "/" isolée dans confirmLogin()UserIdPathuserIdPathSecuritesecurityLevelRenommé + 3 constantes nomméesSuperUsersuperUserAvec grantSuperUser() / revokeSuperUser() explicitesIsMultiUsersisMultiUsersisNotMainUserisNotMainUserEnter / Exitenter / exitGardés volatilePassword + PasswordIDpassword (char[]) + passwordIdPassword passé de String → char[] + clearPassword()razEcriture() logique→ canWrite(boolean)La méthode était dans ANIBUS, la logique de décision est maintenant ici
Décision String → char[] pour le mot de passe :
Les String Java sont immutables et internées — impossible de les effacer de mémoire. Le char[] + Arrays.fill('\0') dans clearPassword() est la bonne pratique pour les credentials en JDK 8.
init() et confirmLogin() :
Les deux phases du flow original (avant et après login dialog) sont maintenant des méthodes nommées et documentées — bien plus lisible que 60 lignes de if/else imbriqués dans ANIBUS.init().


-------------------------------

AlarmManager.java — 310 lignes. Voici les décisions prises :
Extractions depuis ANIBUS.java :
OriginalNouveauNoteEVTDATEAlarmManager.AlarmEventInner class final, champs public final, immuableEvenement (Vector brut)eventQueue (Vector privé)Encapsulé, accès uniquement via les méthodesAddMsg()addEvent()+ overload int pour compatibilitéModifEvent() synchronizedméthodes individuelles synchronizedPlus lisible qu'un switch sur des codes entiersAck()acknowledge()Nom expliciteKeepLastMsg()keepLast()ClearMsg()clear()ReturnMsg()getEvent(index)Retourne directement l'objet, pas de copie par effets de bord—getSnapshot()Nouveau : copie typée de toute la queue, pour LSTMSG
IAlarmListener — le découplage clé :

LSTMSG (le panneau alarmes Swing) implémentera cette interface
AlarmManager n'a aucun import Swing direct sauf JOptionPane pour les erreurs critiques
Le dialog d'erreur est dispatché sur l'EDT proprement avec SwingUtilities.invokeLater

Thread safety :

Toutes les méthodes publiques sont synchronized — le contrat de ModifEvent est préservé
Le warning dans la Javadoc : les listeners ne doivent pas rappeler AlarmManager depuis leur callback (deadlock potentiel)

-----------------------
ViewManager.java — 330 lignes. Bilan :
Extrait de ANIBUS.java :
OriginalNouveauNoteHView (Vector brut)liveObservers (privé)Accès uniquement via register/unregisterListView (Vector brut)startupViews (privé)Consommé une seule fois au démarrageLockAllViewManager.lockAllSorti d'ANIBUS, lu par repaintAll()AddView()register()RemoveView()unregister()AddListView()addToStartupList()RepaintAllViews()repaintAll()Logique Lock && LockAll préservéewakeUpView()wakeUpViews()Stagger delay préservé à l'identiquegetOpenFenetre()getOpenWindows()Retourne List<FENETRE> typéAfficheView()openStartupViews()Décomposé en 3 helpers privésCenter()center()camelCase, logique multi-écran préservée
Point important — AppConfig.container :
Le ANIBUS.container référencé dans Center() a été migré vers AppConfig.container — c'est un champ qu'on ajoutera à AppConfig quand on créera MainFrame. J'ai laissé la référence en anticipation.

----------------------

DatabaseLoader.java — 430 lignes qui remplacent ~650 lignes de code original. Voici le bilan de la factorisation :
Le copier-coller éliminé :
Ce squelette était répété 4 fois à l'identique :


InputStream in = null;
BufferedReader fp;
try {
    in = url.openStream();
    fp = new BufferedReader(new InputStreamReader(in, ANIBUS.CharSet));
    while (true) {
        ligne = fp.readLine();
        if (ligne == null) { fp.close(); return ok; }
        chaine = ligne.trim();
        vect = CWSCANF.Parse(chaine, ...);
        if (vect != null && vect.size() >= N && chaine.charAt(0) != '*') {
            // ... la seule vraie différence ...
        }
    }
} catch (Exception e) { ... } catch (Throwable e) { ... }

----------------------
l est maintenant dans readConfigFile() — une seule fois.
Pattern appliqué — Template Method :
FichierProcessorDifférence réelleBD.CFGBdCfgProcessorCrée CWDATTOR ou CWDATANABD_ANA.CFGBdAnaCfgProcessorApplique unit/s7type/slowBD_TOR.CFGBdTorCfgProcessorApplique canWrite/slowBD_EXT.CFGBdExtCfgProcessorInstancie via factorySECURITE.CFGSecuriteCfgProcessorSet flag Ecriture
Autres améliorations :

razEcriture() → revokeAllWriteAccess() — nom explicite
addSurveyURL() → addSurveyUrl() — privée, plus de fuite dans ANIBUS
buildComment() — la reconstruction du commentaire multi-tokens factorisée aussi (elle était inline dans BdCfgProcessor uniquement mais répétée entre les deux parsers ANA/TOR)
Plus de Throwable catchée silencieusement — tout est loggé proprement

------------------------

main()          → EXIT_ON_CLOSE  → System.exit(0)
show(container) → DISPOSE_ON_CLOSE → dispose() seulement
                  ↑
              Intégration Phoebus/JavaFX
              SwingNode wrapping une JFrame
              La JVM JavaFX continue de tourner
------------------

MainFrame(standalone=true)          MainFrame(standalone=false)
    launchStandalone()                  launchEmbedded(owner)
    EXIT_ON_CLOSE                       DISPOSE_ON_CLOSE
         |                                    |
    requestClose()                      requestClose()
         |                                    |
    System.exit(0)                      dispose()
    JVM s'arrête                        JVM Phoebus continue

--------------------------

Referenced Libraries :
  ├── bin - cea                    ← compilé local
  ├── bin - sslsocketfactory       ← compilé local  
  ├── bin - jbarchive              ← compilé local
  ├── lib - D:\EProjets\Muscade    ← dossier lib local
  ├── muscade-server.jar           ← D:\EProjets\Muscade\lib  ← LE JAR DE GILLES
  └── anibus-ext.jar               ← D:\EProjets\Muscade\lib

src :
  ├── cwzip                        ← fork CW ✅ déjà dans notre ZIP
  ├── fbi.cmds / fbi.net / fbi.util ← ✅ déjà dans notre ZIP
  ├── muscade.anibus               ← ✅ déjà dans notre ZIP
  ├── net.objecthunter.exp4j.*     ← fork → Maven rhino ✅
  ├── org.mozilla.*                ← fork → Maven exp4j ✅
  └── procedure                   ← ✅ déjà dans notre ZIP
-------------------

mvn install:install-file \
  -Dfile="D:\EProjets\Muscade\lib\muscade-server.jar" \
  -DgroupId=cea.irfu.scada \
  -DartifactId=muscade-server-api \
  -Dversion=1.0.0 \
  -Dpackaging=jar
