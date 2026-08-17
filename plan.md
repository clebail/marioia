# Plan — IA Super Mario Bros

## Objectif

Une IA qui gagne à Super Mario Bros 1 (NES), en jouant sur un émulateur réel
comme le ferait un humain : manette physique en entrée, image de l'écran en
sortie. Aucun accès privilégié au jeu au moment de jouer.

## Décisions

**Jeu** : Super Mario Bros 1, version NES.

**Émulateur cible** : Mesen, traité en boîte noire.

**OS** : Linux à l'arrivée. La base peut se développer sur macOS.

**Langage** : C++, from scratch.

**Build** : CMake.

**Manette** : microcontrôleur Pico en USB HID. Vraie manette pour l'émulateur,
pilotée en série (termios). Identique sur les deux OS, seule la chaîne du
périphérique change (`/dev/ttyACM0` / `/dev/cu.usbmodem*`).

**Capture** : modèle poussé — le backend appelle un rappel à chaque frame avec un
horodatage.
- macOS : ScreenCaptureKit, dans un `.mm`.
- Linux : X11/XComposite. Mesen tourne sous un serveur X dédié (Xvfb ou
  Xorg+dummy), pas sur le bureau. Redirection de la fenêtre hors écran par
  `XCompositeNameWindowPixmap`, signal poussé par `XDamage` à chaque frame
  produite, pixels tirés en mémoire partagée par `XShmGetImage` (~1 Mo en
  512x480 RGBA).

  PipeWire écarté : le portail xdg-desktop-portal exige un consentement par
  session et une session graphique complète (dbus, compositeur, démon), ce qui
  ne tient pas dans une boucle non surveillée ; et chaque couche traversée
  ajoute de la gigue, or la gigue est le livrable de l'étape 3. Le serveur X
  dédié évacue en prime toute la question Wayland, et permet N instances sur
  N affichages.

**Horodatage** : toujours pris à la réception chez nous, même point de mesure sur
les deux plateformes, pour que les latences restent comparables.

**Observation** : pixels uniquement, 60 fps.
La RAM sert au calcul de la récompense et au diagnostic, en canal latéral propre
au simulateur, en dehors de l'interface.

**Interface** : minimale. `reset() -> observation`, `step(action) -> observation`.
Rien de plus.

**Simulateur d'entraînement** : fork de SuperMarioBros-C, en headless
(`update()` sans `render()`), quelques milliers de frames par seconde,
un moteur par cœur. `render(uint32_t*)` écrit un 256x240 ARGB dans un buffer
fourni, sans passer par SDL : l'observation du simulateur est un `memcpy`, pas
une capture.

**Save states** : tout l'état est en champs membres plats, donc save/restore par
`memcpy`. Deux tailles, à ne pas confondre :
- recherche seule, qui ne rend rien : `ram[0x800]` + registres +
  `returnIndexStack[100]`, soit ~2,5 Ko ;
- restore *puis* rendu, ce qu'exige DAgger : il faut en plus l'état PPU
  (`nametable[2048]`, `oam[256]`, `palette[32]`, registres), soit ~4,8 Ko.
Le budget mémoire de la recherche double donc si l'on veut pouvoir rendre depuis
n'importe quel nœud. Un save state n'est cohérent qu'en frontière de frame : le
`returnIndexStack` et le dispatch par `goto` du code transpilé interdisent une
sauvegarde en milieu de frame.

**Objectif de victoire** : les 32 niveaux, de 1-1 à 8-4, sans zone de warp, avec
un réseau généraliste unique. Décision du 12 août 2026. C'est la cible la plus
exigeante des options envisagées, assumée comme telle : le repli documenté plus
bas est admis si la généralisation bloque.

**Solveur** : deux étages, dont un seul est déployé.

1. *Oracle de recherche*, dans le simulateur. Best-first sur `(x, area)` — pas
   sur x seul — avec frameskip 4 et jeu d'actions réduit à 6-8 combinaisons.
   Doit résoudre les 32 niveaux avant que la distillation ait un sens.
2. *Réseau pixels vers action*, généraliste sur les 32 niveaux, seul déployé sur
   la boîte noire. Entrée : pile de 4 frames pour donner la vitesse, que la
   frame seule ne contient pas. La pile vit dans l'agent, pas dans l'interface.

**Transfert : DAgger, pas distillation en un coup.** Une trajectoire gagnante ne
visite qu'une séquence d'états ; un réseau entraîné dessus n'a jamais rien vu à
côté du chemin et n'y revient jamais après le premier écart. Or l'écart est
garanti par la gigue du déploiement. On fait donc rouler l'élève, on restaure le
simulateur aux états qu'il a visités, on relance la recherche depuis ces états
pour obtenir la bonne action, et on agrège. L'oracle reste donc disponible
pendant tout l'entraînement, il ne produit pas un jeu de trajectoires en amont.

**Injection de latence à l'entraînement.** Dans le simulateur `step()` est
instantané ; sur le vrai montage il y a la capture plus le série, avec de la
gigue, et SMB demande des sauts à la frame près au-dessus des trous. Une
politique entraînée à latence nulle est accordée sur un monde qui n'existe pas.
Une fois la distribution mesurée à l'étape 3, on l'injecte : action appliquée k
frames plus tard, observation vieille de k' frames, gigue échantillonnée dans la
distribution réelle. La mesure de l'étape 3 est un paramètre d'entraînement, pas
un livrable décoratif.

**Jumeau synchronisé : écarté.** Les deux moteurs étant déterministes, on
pourrait faire tourner SMB-C en jumeau de Mesen, planifier dedans et n'utiliser
la capture que pour vérifier la synchro — sans aucun réseau. Écarté parce que
son mode d'échec est irrattrapable : à la première désynchronisation due à la
gigue, le jumeau n'a aucun moyen de se réamorcer depuis les pixels. Une
politique réactive est robuste au même incident par construction, puisqu'elle
réobserve à chaque frame. C'est le meilleur argument en faveur de
l'architecture retenue.

**Repli admis** si la généralisation bloque, par ordre de préférence : plus de
tours de DAgger ciblés sur les niveaux en échec, puis plus de capacité, puis en
dernier recours une politique par niveau — qui gagne mais n'est presque que du
rejeu appris.

**Dépôt** : unique. `clebail/marioia` est le fork de SuperMarioBros-C et porte
aussi l'interface, l'adaptateur Mesen, l'adaptateur simulateur et le solveur.
Décision du 11 août 2026, qui remplace le découpage fork + nouveau dépôt en
sous-module initialement prévu.

Conséquence à tenir : le code du projet cohabite avec les 63 commits d'upstream.
Garder les ajouts en dehors de `source/` et `codegen/` pour qu'un futur rebase
sur SuperMarioBros-C reste lisible, et le patch du moteur (accès RAM, boucle
headless) aussi minimal que prévu à l'origine.

## Étapes

1. Fork SuperMarioBros-C — fait
2. Installer et configurer Mesen — fait
3. Interface + adaptateur Mesen (capture + Pico), dans marioia.
   Livrable : latence moyenne et gigue du pipeline complet.
   Premier test bloquant, à faire sur la machine Linux avant d'écrire le
   backend : lancer Mesen sous Xvfb et vérifier qu'il tient 60 fps. Xvfb tombe
   sur du GL logiciel (llvmpipe) ; si la cadence ne suit pas, basculer sur
   Xorg+dummy avec le GPU réel. C'est le seul risque sérieux de l'approche X11.
4. Adaptateur simulateur, puis solveur, dans cet ordre :
   a. Harnais de comparaison de pixels SMB-C / Mesen. Bloquant : le réseau
      apprend sur les pixels de `PPU.cpp` et joue sur ceux de Mesen.
   b. Oracle de recherche, jusqu'à résoudre les 32 niveaux.
   c. Boucle DAgger avec injection de la latence mesurée à l'étape 3.
   d. Évaluation sur la boîte noire.

L'ordre compte : l'interface se conçoit contre le côté contraint, Mesen et la
manette physique, qui imposent leurs conditions.

## Config Mesen figée

Déjà appliqué :
- `DisableOsd` = true
- `AllowBackgroundInput` = true
- `PauseWhenInBackground` = false
- overscan NTSC à 0 sur les quatre côtés
- `VideoFilter` = None
- `UseBilinearInterpolation` = false
- `AspectRatio` = NoStretching
- `ScanlineIntensity` et paramètres NTSC à 0
- `ShowFps` = false
- écran macOS en échelle exactement 2x

À revalider empiriquement à l'étape 3 : la taille de fenêtre. Les valeurs
enregistrées ne correspondent à aucun facteur entier de 256x240. Premier test
à écrire : capturer une frame et vérifier ses dimensions réelles.

Copier le `settings.json` dans le dépôt une fois figé.

## À trancher

- Résolution et format d'entrée du réseau (niveaux de gris, sous-échantillonnage).
  Dépend du harnais de comparaison de pixels : inutile de choisir avant de savoir
  si SMB-C et Mesen sont identiques au pixel près.
- Ensemencement des épisodes d'entraînement sur les 32 niveaux : save states
  collectés en début de chaque niveau, ou écriture directe de `WorldNumber`
  ($075f), `AreaNumber` ($0760) et `LevelNumber` ($075c). La seconde est plus
  simple mais contourne des routines d'initialisation, donc à valider.

## Pièges connus

- `codegen` réécrit `SMB.cpp`, `SMBData.cpp`, `SMBConstants.hpp` et
  `SMBDataPointers.hpp` dans l'arbre source à chaque build. Toute modification
  de ces fichiers passe par `codegen/`.
- Press et release doivent être séparés par un sync frame, sinon l'entrée est
  ignorée par l'émulateur.
- Capturer à 60 fps, la cadence du jeu.
- macOS : l'autorisation d'enregistrement d'écran s'attache au programme qui
  lance. Depuis un terminal, c'est le terminal qui doit l'avoir.
- Un serveur X dédié ne coupe pas l'entrée physique : Mesen lit les manettes via
  SDL, qui passe par evdev sur `/dev/input/event*` sans transiter par X. La
  manette Pico reste donc opérante sous Xvfb.
- `XCompositeNameWindowPixmap` doit être rappelé si la fenêtre est redimensionnée,
  l'ancien pixmap devenant invalide.
- Les châteaux-labyrinthes 4-4, 7-4 et 8-4 rebouclent sur le mauvais chemin. Une
  heuristique gloutonne en x y tourne indéfiniment : chercher sur `(x, area)` et
  détecter les boucles. Corollaire côté réseau : le bon chemin n'y est pas
  déductible des pixels locaux, le généraliste devra les mémoriser. C'est admis.
- Les niveaux aquatiques (2-2, 7-2) ont une physique et un usage du bouton A
  entièrement différents. Ils doivent être représentés dans l'échantillonnage
  d'entraînement, sous peine d'être les premiers à échouer.
- Le prétraitement doit être rigoureusement identique entre simulateur et
  déploiement. Ça rend la taille de fenêtre Mesen (voir plus haut) non plus
  souhaitable mais obligatoire en facteur entier de 256x240 : un
  sous-échantillonnage depuis une taille non entière introduit des artefacts de
  rééchantillonnage qui n'existent pas dans le simulateur.
- L'évaluation sur la boîte noire tourne en temps réel : une partie complète
  coûte une quinzaine de minutes. L'essentiel de l'évaluation se fait dans le
  simulateur, la boîte noire ne sert qu'à la validation périodique.
- SuperMarioBros-C n'a aucun fichier de licence, donc tous droits réservés.
  C'est la relation de fork GitHub qui couvre la copie, il faut la conserver.
- Le dépôt de base d'une PR se vérifie à la main, ou se fixe avec
  `gh repo set-default`.
