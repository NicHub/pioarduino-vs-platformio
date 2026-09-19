# pioarduino vs PlatformIO

Ce projet utilise **pioarduino** (fork open-source de PlatformIO). Ce document récapitule le diagnostic établi sur cette machine : comment savoir quel core est réellement utilisé, comment gérer la coexistence des deux extensions, et les points d’attention.

## Comment savoir quel core est utilisé

### Le piège de `which pio`

```sh
which pio
# $HOME/.platformio/penv/bin/pio
```

Ce chemin est **trompeur** : `~/.platformio/penv/bin/pio` est le chemin historique partagé par les deux outils (pioarduino garde la compatibilité). Le fait qu’il affiche `PlatformIO Core` ne signifie pas que le PlatformIO officiel est utilisé : le fork conserve ce branding.

### La vraie méthode : vérifier le module Python installé

Le discriminateur est le **code** installé dans le venv :

```sh
python3 -c "import platformio; print(platformio.__title__, platformio.__version__)"
# pioarduino 6.2.0
```

Si `__title__` vaut `pioarduino`, c’est le fork qui est installé. Autres commandes utiles :

```sh
pio system info        # infos sur le core actif
grep __title__ ~/.platformio/penv/lib/python*/site-packages/platformio/__init__.py
```

État constaté sur cette machine : le module `platformio/__init__.py` a `__title__ = "pioarduino"` et `__url__ = "https://github.com/pioarduino"` → le core actif est **pioarduino-core 6.2.0**, installé par-dessus l’ancien core.

### Dans VS Code

Les deux extensions sont installées côte à côte :

-  `platformio.platformio-ide` (officiel)
-  `pioarduino.pioarduino-ide` (fork)

L’IDE ne « choisit » pas entre les deux : il exécute le `pio` présent sur le PATH. Le projet ici est configuré pour pioarduino via `.vscode/extensions.json`. Le réglage `platformio-ide.activateOnlyOnPlatformIOProject` du fork est conservé sous le même préfixe, c’est normal.

## Gérer la coexistence

1. **N’activer qu’une seule extension IDE à la fois** (de préférence pioarduino). Les deux fournissent les mêmes tâches, la même PIO Home et la même barre d’état → doublons et conflits. Désactiver `platformio.platformio-ide` depuis le panneau Extensions.
2. **Un seul core par projet** : deux cores lancés sur le même dossier `.pio` corrompent l’environnement de build.
3. **Fichiers de projet compatibles** : `platformio.ini`, `.pio/`, `.vscode/` ont le même format (fork drop-in). Un projet créé avec PlatformIO officiel fonctionne sans modification dans pioarduino et inversement.
4. **Séparation stricte** (optionnel) : créer un venv dédié, hors du PATH partagé :

   ```sh
   python -m venv ~/.pioarduino/penv
   ~/.pioarduino/penv/bin/pip install pioarduino
   ~/.pioarduino/penv/bin/pio run
   ```

> **À quoi sert ce venv ?** PlatformIO (et le fork pioarduino) ne vit pas dans un dossier unique : son *core* (la commande `pio`) est un paquet Python installé dans un environnement virtuel, et c’est lui qui pilote tout le reste. Au premier lancement, ce core télécharge dans `~/.platformio/` ses *packages* et *platforms* (compilateurs, toolchains, SDK des cartes…), puis invoque ces outils à chaque build ou upload. Sans venv dédié, les deux cores se partagent le même dossier d’installation (`~/.platformio/penv`) : ils s’écrasent mutuellement à chaque installation ou mise à jour. Un venv réservé à pioarduino donne à chaque core son propre « monde » (ses dépendances et sa commande `pio`), indépendant de l’autre — c’est la différence entre des dossiers séparés et des outils qui s’installent par-dessus leurs voisins.

## Points d’attention

-  **Partage de `~/.platformio`** : les deux cores écrivent dans les mêmes `packages/`, `platforms/`, `penv/`. **Le dernier installé ou mis à jour gagne.** Après un `pio upgrade` ou une réinstallation, revérifier `__title__`.
-  **Métadonnée orpheline** : `platformio-6.2.0.dist-info` traîne encore dans site-packages (vestige de l’installation d’origine). Sans danger, mais preuve que les deux packages ont été installés dans le même venv. En cas de souci :

  ```sh
  ~/.platformio/penv/bin/pip uninstall platformio pioarduino
  ~/.platformio/penv/bin/pip install pioarduino
  ```

-  **Fichiers auto-générés** : `.vscode/c_cpp_properties.json` et `.vscode/launch.json` contiennent des chemins absolus et sont régénérés par l’IDE. Ne pas les éditer à la main.
-  Cocher `platformio.platformio-ide` dans `unwantedRecommendations` de `.vscode/extensions.json` évite que l’autre extension soit re-suggérée.
