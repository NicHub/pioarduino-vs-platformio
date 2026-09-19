# HELLO WORLD PIOARDUINO

Projet Arduino Uno avec PlatformIO (CLI).

## Commandes utiles

### Build (compilation)

```shell
pio run
```

### Upload (téléversement sur la carte)

```shell
pio run -t upload
```

### Erase flash (efface uniquement la flash)

```shell
pio run -t erase
```

### Monitor (console série)

```shell
pio device monitor
```

### Clean (nettoyage du build)

```shell
pio run -t clean
```

### Build + upload + monitor (enchaînés)

```shell
pio run -t upload -t monitor
```
