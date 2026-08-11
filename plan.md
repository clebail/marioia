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
un moteur par cœur.

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
4. Adaptateur simulateur, puis solveur.

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

- Approche du solveur. Piste évoquée, non tranchée : recherche dans le
  simulateur (save states de moins de 3 Ko) pour produire des trajectoires
  gagnantes, puis distillation supervisée vers un réseau pixels vers action,
  qui seul est déployé sur la boîte noire.

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
- SuperMarioBros-C n'a aucun fichier de licence, donc tous droits réservés.
  C'est la relation de fork GitHub qui couvre la copie, il faut la conserver.
- Le dépôt de base d'une PR se vérifie à la main, ou se fixe avec
  `gh repo set-default`.
