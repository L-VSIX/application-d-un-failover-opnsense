# Bascule automatique du routeur OPNsense (Hyperviseur MSI → ACE)

## Contexte

Dans le cadre du plan de reprise d'activité (PRA) du homelab RAID-A-PORTER, le pare-feu **OPNsense** — qui assure le rôle de routeur, passerelle et serveur DHCP — tourne en VM sur l'hyperviseur **MSI**.

Ce nœud MSI subit extinctions fréquentes et parfois inattendues de l'hyperviseur. Lorsque MSI est éteint, tout le réseau perd sa passerelle et son DHCP.

Pour pallier ce point de défaillance unique, un **clone de la VM OPNsense** est maintenu à jour sur l'hyperviseur **ACE**. Ce script permet à ACE de détecter automatiquement l'indisponibilité de MSI et de démarrer ce clone en relais, avant de l'éteindre de lui-même dès que MSI revient en ligne.

## Fonctionnement

Le script s'exécute via une tâche cron sur l'hyperviseur ACE, toutes les minutes :

1. **Test de connectivité** — un ping est envoyé vers l'adresse IP de l'hyperviseur MSI.
2. **Détection de panne** — si le ping échoue plusieurs fois de suite (seuil configurable, 3 par défaut), MSI est considéré comme indisponible.
3. **Bascule** — le clone OPNsense est démarré sur ACE via `qm start`, et un fichier d'état marque la bascule active pour éviter les démarrages en double.
4. **Retour à la normale** — dès que MSI répond de nouveau au ping, le clone est arrêté via `qm stop` pour éviter d'avoir deux instances OPNsense actives simultanément sur le réseau, et le fichier d'état est supprimé.
5. **Maintenance du log** — le fichier de log est purgé automatiquement tous les 4 passages du script, pour éviter qu'il ne grossisse indéfiniment.

## Installation

Placer le script sur l'hyperviseur ACE :

```bash
cd /usr/local/bin/
```

Créer le fichier `check-failover.sh` :

```bash
#!/bin/bash

# ==== VARIABLES A REMPLIR ====
IP_HYPERVISEUR_A="192.168.1.30"      # Adresse IP d'hyperviseur MSI
VMID_CLONE="120"                      # ID de la VM clone OPNsense sur ACE

# ==== VARIABLES INTERNES (ne pas modifier) ====
FICHIER_ETAT="/tmp/failover_actif"
FICHIER_COMPTEUR="/tmp/failover_compteur"
FICHIER_COMPTEUR_LOG="/tmp/failover_compteur_log"
SEUIL_ECHECS=3
LOG="/var/log/failover.log"

# ---- Nettoyage du log tous les 4 passages ----
if [ ! -f "$FICHIER_COMPTEUR_LOG" ]; then
    echo 0 > "$FICHIER_COMPTEUR_LOG"
fi

COMPTEUR_LOG=$(cat "$FICHIER_COMPTEUR_LOG")
COMPTEUR_LOG=$((COMPTEUR_LOG + 1))

if [ "$COMPTEUR_LOG" -ge 4 ]; then
    echo "$(date): Nettoyage automatique du log (4 exécutions atteintes)." > "$LOG"
    COMPTEUR_LOG=0
fi

echo "$COMPTEUR_LOG" > "$FICHIER_COMPTEUR_LOG"

# ---- Logique de failover ----
if [ ! -f "$FICHIER_COMPTEUR" ]; then
    echo 0 > "$FICHIER_COMPTEUR"
fi

COMPTEUR=$(cat "$FICHIER_COMPTEUR")

if ping -c 1 -W 2 "$IP_HYPERVISEUR_A" > /dev/null 2>&1; then
    echo 0 > "$FICHIER_COMPTEUR"

    if [ -f "$FICHIER_ETAT" ]; then
        echo "$(date): Hyperviseur MSI de nouveau en ligne, arrêt du clone." >> "$LOG"
        /usr/sbin/qm stop "$VMID_CLONE"
        rm -f "$FICHIER_ETAT"
    fi
else
    COMPTEUR=$((COMPTEUR + 1))
    echo "$COMPTEUR" > "$FICHIER_COMPTEUR"
    echo "$(date): Echec ping hyperviseur MSI ($COMPTEUR/$SEUIL_ECHECS)" >> "$LOG"

    if [ "$COMPTEUR" -ge "$SEUIL_ECHECS" ] && [ ! -f "$FICHIER_ETAT" ]; then
        echo "$(date): Hyperviseur MSI injoignable, démarrage du clone." >> "$LOG"
        /usr/sbin/qm start "$VMID_CLONE"
        touch "$FICHIER_ETAT"
    fi
fi
```

Rendre le script exécutable :

```bash
chmod +x /usr/local/bin/check-failover.sh
```

Ajouter la tâche cron (dans le crontab **root**, `sudo crontab -e`) pour une exécution toutes les minutes :

```
* * * * * /usr/local/bin/check-failover.sh
```

## Points d'attention

- **Droits d'exécution** — `qm start`/`qm stop` nécessitent d'être root. Le crontab doit donc être celui de l'utilisateur root, pas d'un utilisateur standard.
- **Chemin absolu** — l'environnement cron n'a pas le même `$PATH` que le shell interactif ; le script utilise `/usr/sbin/qm` en chemin complet pour éviter les échecs silencieux.
- **Synchronisation du clone** — ce script suppose que le clone OPNsense sur ACE est maintenu manuellement à jour (config réseau, règles de pare-feu). Il ne gère pas la réplication de configuration entre les deux VM.
- **Fichiers temporaires** — en cas de comportement anormal (blocage de la bascule), vérifier et nettoyer si besoin :
  ```bash
  rm -f /tmp/failover_actif /tmp/failover_compteur /tmp/failover_compteur_log
  ```

## Difficultés rencontrées et débogage

Cette section retrace les problèmes réels rencontrés pendant la mise au point du script, pour garder une trace utile en cas de réapparition d'un symptôme similaire.

### 1. Boucle infinie et syntaxe invalide dans une version intermédiaire

Une première réécriture du script (tentative d'ajout d'un délai manuel entre les vérifications) introduisait une boucle `while true` combinée à une syntaxe de test invalide en bash :

```bash
elif [ $(date)" + "$DELAY" minutes ago ]; then
```

Cette construction n'existe pas en bash — elle ne provoque pas d'erreur bloquante mais ne teste jamais rien de valable, ce qui rendait toute la logique de bascule inopérante. La logique `qm start` / `qm stop` était en plus inversée : le `qm stop` s'exécutait dès que l'hyperviseur MSI répondait, sans condition sur un état de bascule préalable, et le `qm start` restait bloqué derrière une condition qui ne pouvait jamais devenir vraie au premier passage.

**Correction** : suppression complète de la boucle `while`. Le script ne fait plus qu'**un seul test par exécution**, en laissant le cron gérer la répétition. C'est cette version qui est documentée plus haut.

### 2. Le démarrage du clone ne fonctionnait pas, alors que la commande seule fonctionnait

Une fois le script corrigé et en place via cron, le symptôme suivant est apparu : lancer `qm start 120` manuellement dans le terminal fonctionnait, mais le script ne démarrait jamais le clone tout seul.

Trois causes possibles ont été identifiées, par ordre de probabilité :

- **Droits root** : `qm start` nécessite d'être root. Si la tâche cron est déclarée dans le crontab d'un utilisateur standard (`crontab -e` sans `sudo`), la commande échoue silencieusement.
- **`$PATH` incomplet dans l'environnement cron** : le cron n'hérite pas du `$PATH` complet du shell interactif. `qm` se trouve dans `/usr/sbin`, absent du `$PATH` minimal de cron.
- **Log muet sur l'étape concernée** : vérification via `tail -f /var/log/failover.log` pour confirmer si le script atteignait seulement la ligne « injoignable » sans jamais exécuter réellement la commande.

**Correction appliquée** : passage à `sudo crontab -e` (tâche en crontab root) et remplacement de `qm start` / `qm stop` par les chemins absolus `/usr/sbin/qm start` / `/usr/sbin/qm stop` dans le script.

### 3. Compteur d'échecs qui grimpe indéfiniment sans jamais déclencher la bascule

Après correction du point précédent, le log affichait une progression du compteur bien au-delà du seuil sans qu'aucune bascule ne se déclenche :

```
Thu Aug 13 12:25:03 PM AST 2026: Echec ping hyperviseur A (13/3)
Thu Aug 13 12:26:03 PM AST 2026: Echec ping hyperviseur A (14/3)
Thu Aug 13 12:27:03 PM AST 2026: Echec ping hyperviseur A (15/3)
Thu Aug 13 12:28:03 PM AST 2026: Echec ping hyperviseur A (16/3)
```

**Diagnostic** : la condition de démarrage est `[ "$COMPTEUR" -ge "$SEUIL_ECHECS" ] && [ ! -f "$FICHIER_ETAT" ]`. Le compteur dépassait largement le seuil, ce qui signifiait que `$FICHIER_ETAT` (`/tmp/failover_actif`) existait déjà — probablement créé par un `touch` exécuté après un `qm start` ayant échoué silencieusement lors d'un test antérieur (avant la correction du point 2). Résultat : le fichier d'état bloquait indéfiniment tout nouveau démarrage, sans que le clone ait jamais réellement tourné.

**Correction** :
```bash
rm -f /tmp/failover_actif /tmp/failover_compteur /tmp/failover_compteur_log
```
pour repartir sur un état propre, combinée à une amélioration du script pour ne créer `$FICHIER_ETAT` qu'en cas de succès réel de `qm start` :

```bash
if [ "$COMPTEUR" -ge "$SEUIL_ECHECS" ] && [ ! -f "$FICHIER_ETAT" ]; then
    echo "$(date): Hyperviseur MSI injoignable, démarrage du clone." >> "$LOG"
    if /usr/sbin/qm start "$VMID_CLONE"; then
        touch "$FICHIER_ETAT"
        echo "$(date): Clone démarré avec succès." >> "$LOG"
    else
        echo "$(date): ERREUR - échec du démarrage du clone." >> "$LOG"
    fi
fi
```

Cette version évite qu'un échec de `qm start` (droits, VM déjà démarrée, ressources insuffisantes, etc.) ne bloque silencieusement toutes les tentatives suivantes.

## Améliorations possibles

- Vérifier le code de retour de `qm start` avant de créer le fichier d'état, pour ne pas marquer une bascule comme "active" si la commande a réellement échoué.
- Ajouter une notification (email, webhook) au moment de la bascule et du retour à la normale.
- Basculer le test de disponibilité sur l'API Proxmox plutôt qu'un simple ping, pour distinguer un hôte injoignable réseau d'un service Proxmox arrêté mais hôte allumé.

## Repos liés

- [`pbs-plan-de-reprise-pra`](https://github.com/L-VSIX/pbs-plan-de-reprise-pra) — second site de sauvegarde

## Auteur

**Lilian Vertueux** — [LinkedIn](https://www.linkedin.com/in/lilian-vertueux/)
