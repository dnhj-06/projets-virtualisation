# Projet 4 — Migration d'un serveur physique vers une VM (P2V)

**Cours :** Virtualisation — Mettre en place et exploiter une plateforme de virtualisation
**Classe :** E1A
**Auteur :** Drilon Nuhija
**Date :** 9 octobre 2026
**Outils :** VMware Workstation 17 Pro (17.5.2), Clonezilla live 3.3.3-15

---

## Contexte

Une entreprise possède un ancien serveur physique qu'elle veut virtualiser pour prolonger sa durée de vie logicielle et simplifier sa maintenance.

Faute de serveur physique disponible, la migration est simulée. La VM **SRV-IMPRESSION** (Windows 10) joue l'ancien serveur physique. Elle est traitée comme une vraie machine : on démarre sur un outil de clonage, on copie son disque sur un disque de destination, puis ce disque est branché sur une nouvelle VM, **SRV-IMPRESSION-P2V**.

### Résumé des résultats

| Élément | Résultat |
|---|---|
| Serveur source | SRV-IMPRESSION, Windows 10 Pro, serveur de fichiers et d'impression, IP 10.10.10.12 ✅ |
| Outil choisi | Clonezilla live 3.3.3-15, clonage disque à disque à froid ✅ |
| Migration | 21,8 Go copiés, environ 18 minutes de copie, aucune erreur ✅ |
| VM migrée | SRV-IMPRESSION-P2V démarre sur le disque cloné ✅ |
| Données | Fichier de test identique avant et après (même empreinte SHA-256) ✅ |
| Rôle | Même partage et même imprimante partagée ✅ |
| Problème rencontré | Adresse IP perdue à cause du changement de carte réseau ❌, corrigée ✅ |
| Test depuis un poste client | Ping, port 445 et ouverture du partage depuis SRV-FICHIERS ✅ |

> 📸 Les captures servent d'illustration. Toutes les valeurs utiles sont aussi écrites en texte, dans les tableaux.

---

## 1. Identifier le rôle et la configuration du serveur physique (réel ou simulé) à migrer

### 1.1 Le serveur choisi

| Élément | Valeur |
|---|---|
| Nom dans VMware | SRV-IMPRESSION |
| Nom Windows | DESKTOP-BUMKVM8 |
| Système | Windows 10 Professionnel, version 10.0.19045 (22H2), 64 bits |
| Démarrage | UEFI, Secure Boot désactivé |
| Pourquoi ce serveur | Windows 10 n'est plus suivi : c'est le cas typique d'un ancien serveur qu'on virtualise pour le garder en service |

La simulation a une limite : le « serveur physique » est déjà une VM VMware. Ses pilotes sont donc déjà des pilotes de matériel virtuel. Pour se rapprocher d'un vrai P2V, la VM cible a reçu un matériel différent (voir 3.4).

📸 *Capture 01 — SRV-IMPRESSION dans VMware : 2 Go de RAM, 2 processeurs, disque NVMe de 61 Go, réseau VMnet10*

![Capture 01 — SRV-IMPRESSION dans VMware : 2 Go de RAM, 2 processeurs, disque NVMe de 61 Go, réseau VMnet10](https://hackmd.io/_uploads/SJltWF9HsMg.png)

### 1.2 Rôle du serveur

Le serveur reçoit un rôle concret, pour pouvoir vérifier après la migration qu'il rend toujours le même service.

| Rôle | Configuration |
|---|---|
| Serveur de fichiers | Dossier `C:\Partage` partagé sous le nom `Partage`, en lecture pour « Tout le monde » |
| Fichier de contrôle | `C:\Partage\clients.txt` (104 octets, 3 lignes) |
| Serveur d'impression | Imprimante `IMPRIMANTE-PME`, pilote « Generic / Text Only », port TCP/IP `10.10.10.50`, partagée sous le nom `IMPRIMANTE-PME` |
| Adresse réseau | IP fixe `10.10.10.12/24` sur VMnet10, sans passerelle, DNS `8.8.8.8` et `1.1.1.1` |

L'imprimante est fictive : aucune imprimante réelle n'existe à l'adresse 10.10.10.50. Seule la file d'impression partagée compte ici.

Mise en place :

1. **Dossier partagé** : créer `C:\Partage`, y créer `clients.txt` avec le Bloc-notes, puis clic droit sur le dossier → **Propriétés → Partage → Partage avancé** → cocher **Partager ce dossier**, nom `Partage`, autorisation **Lecture** pour « Tout le monde ».
2. **Imprimante** : **Paramètres → Périphériques → Imprimantes et scanners → Ajouter une imprimante** → « L'imprimante que je veux n'est pas répertoriée » → **Ajouter une imprimante à l'aide d'une adresse TCP/IP** → `10.10.10.50` → pilote **Generic / Text Only** → nom `IMPRIMANTE-PME` → **Partager cette imprimante**.

Contenu du fichier de contrôle :

```
Fichier de test du serveur SRV-IMPRESSION
Cree avant la migration P2V
Ligne de controle : 0123456789
```

### 1.3 Configuration relevée avant la migration

| Domaine | Valeur relevée |
|---|---|
| Fabricant et modèle | VMware, Inc. — VMware20,1 |
| Processeur | AMD Ryzen 7 260, 2 processeurs logiques |
| Mémoire | 2 Go |
| Disque | NVMe 61 Go, GPT : partition système 0,1 Go, réservée 0,02 Go, `C:` 60,34 Go, récupération 0,55 Go |
| Espace sur C: | 37,38 Go libres, donc environ 23 Go utilisés |
| Carte réseau | Intel(R) 82574L Gigabit Network Connection, pilote Microsoft 12.17.10.8 |
| Adresse MAC | 00-0C-29-5F-04-A2 |
| Contrôleurs de stockage | Contrôleur NVM Express standard, contrôleur AHCI SATA standard |
| Identifiant matériel (UUID) | 237B4D56-B135-31C9-7EAC-DCDDD75F04A2 |
| Logiciels | Microsoft Edge, Visual C++ 2015-2022, VMware Tools 12.4.0 |
| Comptes actifs | PC01_DN |
| Services | Serveur (LanmanServer), Station de travail, Spouleur d'impression : démarrés, démarrage automatique |
| Empreinte de `clients.txt` | SHA-256 `2FAE7A63A25F1E49526A4AB24CDBF77F1894BB1D7E1BD5D11DEF9183D3BAEB1A` |
| Licence Windows | Professional, canal RETAIL, **non activée** (mode notification, code 0xC004F034) |
| Périphériques en erreur | Aucun |

Commandes et outils pour retrouver ces valeurs :

| Valeur | Où la trouver |
|---|---|
| Système, mémoire, fabricant | `systeminfo` |
| Carte réseau, MAC, IP, DNS | `ipconfig /all` |
| Partages | `net share` |
| Imprimantes | `powershell Get-Printer` |
| Disques et partitions | Gestion des disques (`diskmgmt.msc`) |
| Pilotes | Gestionnaire de périphériques (`devmgmt.msc`) |
| Empreinte du fichier | `powershell Get-FileHash C:\Partage\clients.txt` |
| Licence | `slmgr /dli` |

Le filigrane « Activer Windows » est visible sur l'écran de verrouillage du serveur.

📸 *Capture 02 — Écran de verrouillage de SRV-IMPRESSION avec le filigrane « Activer Windows »*

![Capture 02 — Écran de verrouillage de SRV-IMPRESSION avec le filigrane « Activer Windows »](https://hackmd.io/_uploads/BylqZF9BoGe.png)

---

## 2. Choisir un outil de migration P2V ou reconstruire équivalent en VM

### 2.1 Le principe d'une migration P2V

Un P2V (Physical to Virtual) copie le disque d'une machine physique, avec son système, ses programmes, ses réglages et ses données, dans un disque virtuel. Ce disque est ensuite démarré dans une VM. Le système ne se réinstalle pas : c'est le même Windows, qui découvre un nouveau matériel.

Il existe deux façons de faire :

| Méthode | Principe | Avantage | Inconvénient |
|---|---|---|---|
| À chaud | Un agent copie le système pendant qu'il tourne | Peu d'interruption de service | Les fichiers modifiés pendant la copie doivent être resynchronisés |
| À froid | On démarre la machine sur un autre système et on copie le disque à l'arrêt | Copie cohérente, rien ne bouge pendant la copie | Le serveur est indisponible pendant toute la copie |

La troisième possibilité, la **reconstruction**, consiste à installer une nouvelle VM et à y refaire le rôle à la main. Ce n'est plus une copie du serveur.

### 2.2 Comparaison des outils

| Outil | Méthode | Résultat | Contrainte |
|---|---|---|---|
| VMware vCenter Converter Standalone 6.6 | À chaud, agent installé sur la source | VM VMware directement, avec adaptation des pilotes | Téléchargement réservé aux comptes Broadcom |
| Disk2vhd (Microsoft Sysinternals) | À chaud | Fichier VHD/VHDX | Format Hyper-V : il faut ensuite le convertir pour VMware |
| Clonezilla live | À froid, disque à disque ou vers une image | Copie exacte du disque | Serveur arrêté pendant la copie, aucune adaptation des pilotes |
| Reconstruction | Nouvelle installation | VM propre | Tout le rôle est à refaire, risque d'oubli |

### 2.3 Choix retenu : Clonezilla

| Critère | Raison |
|---|---|
| Disponibilité | Gratuit, libre, sans compte. Converter n'a pas pu être utilisé faute de compte Broadcom. |
| Fidélité | Copie exacte du disque : même système, mêmes réglages, mêmes données |
| Méthode réaliste | Démarrer le serveur sur une clé de clonage et copier son disque vers un disque externe est une méthode courante pour un ancien serveur |
| Contrôle | Chaque étape est visible et confirmée, notamment le choix de la source et de la cible |

Dans cette simulation, le « disque externe » est un second disque virtuel de 80 Go ajouté au serveur. Après la copie, ce disque est débranché du serveur et branché sur la nouvelle VM.

---

## 3. Réaliser la migration/reconstruction

### 3.1 Préparation du serveur

| Étape | Manipulation | Résultat |
|---|---|---|
| Sauvegarde | Serveur éteint, **VM → Snapshot → Take Snapshot**, nom `avant-p2v` | Retour possible à l'état d'avant ✅ |
| Disque de destination | **VM → Settings → Add → Hard Disk → NVMe → Create a new virtual disk → 80 Go** | Second disque de 80 Go ✅ |
| Préparation du disque | Dans Windows, Gestion des disques : initialiser en GPT, nouveau volume NTFS nommé `P2V-DEST` | Disque prêt ✅ |

Le disque de destination fait 80 Go, plus que le disque source de 61 Go : Clonezilla demande une cible au moins aussi grande que la source.

📸 *Capture 03 — Snapshot « avant-p2v » du serveur*

![Capture 03 — Snapshot « avant-p2v » du serveur](https://hackmd.io/_uploads/r1lKWK9Sofe.png)

📸 *Capture 04 — Ajout du disque de destination : disque virtuel de 80 Go*

![Capture 04 — Ajout du disque de destination : disque virtuel de 80 Go](https://hackmd.io/_uploads/BJeY-F9Hizl.png)

📸 *Capture 05 — Paramètres de la VM : nouveau disque NVMe de 80 Go*

![Capture 05 — Paramètres de la VM : nouveau disque NVMe de 80 Go](https://hackmd.io/_uploads/SyWYZK5BjMg.png)

📸 *Capture 06 — SRV-IMPRESSION prête : snapshot avant-p2v, disque de 61 Go et disque de 80 Go*

![Capture 06 — SRV-IMPRESSION prête : snapshot avant-p2v, disque de 61 Go et disque de 80 Go](https://hackmd.io/_uploads/SyKZK5Bofg.png)

### 3.2 Démarrage du serveur sur Clonezilla

1. **VM → Settings → CD/DVD** : utiliser l'image `clonezilla-live-3.3.3-15-amd64.iso`.
2. **VM → Power → Power On to Firmware** : la VM s'arrête sur le menu de démarrage UEFI.
3. Dans le **Boot Manager**, choisir **EFI VMware Virtual SATA CDROM Drive (1.0)**.
4. Dans le menu de Clonezilla : **Other modes of Clonezilla live → Clonezilla live Safe graphic settings (vga=normal)**. Le mode normal s'est bloqué au premier essai (incident 1).

📸 *Capture 07 — Boot Manager : démarrage sur le lecteur CD qui contient Clonezilla*

![Capture 07 — Boot Manager : démarrage sur le lecteur CD qui contient Clonezilla](https://hackmd.io/_uploads/HJFWt5rjfg.png)

📸 *Capture 09 — Menu de démarrage de Clonezilla*

![Capture 09 — Menu de démarrage de Clonezilla](https://hackmd.io/_uploads/SyY-tcSjzg.png)

📸 *Capture 10 — Autres modes : choix « Safe graphic settings (vga=normal) »*

![Capture 10 — Autres modes : choix « Safe graphic settings (vga=normal) »](https://hackmd.io/_uploads/BktWF5Bifg.png)

### 3.3 Clonage disque à disque

| Écran de Clonezilla | Choix |
|---|---|
| Langue | fr_FR.UTF-8 Français |
| Clavier | Garder la disposition par défaut |
| Démarrage | Start_Clonezilla |
| Mode | device-device (disque vers disque) |
| Niveau | Beginner (débutant) |
| Type de copie | disk_to_local_disk (disque local vers disque local) |
| Disque source | `nvme0n1`, 61 Go, le disque du serveur |
| Disque cible | `nvme0n2`, 80 Go, le disque de destination |
| Vérification de la source | `-sfsck` : pas de vérification (choix par défaut) |
| Table de partitions | `-k0` : reprendre celle de la source |
| Action à la fin | `-p poweroff` |

Clonezilla affiche la commande qu'il va exécuter :

```
/usr/sbin/ocs-onthefly -g auto -e1 auto -e2 -r -j2 -sfsck -k0 -plu -p poweroff -f nvme0n1 -d nv...
```

`-f nvme0n1` indique la source (« from »), `-d` la destination.

Avant de copier, Clonezilla affiche le disque qui va être effacé :

```
ATTENTION! LES DONNÉES EXISTANTES SUR LE DISQUE OU LA PARTITION VONT ÊTRE ÉCRASÉES !
nvme0n2 (80G|VMware_Virtual_NVMe_Disk|...)
nvme0n2p1 (16M|No_file_system|...)
nvme0n2p2 (80G|ntfs|In_VMware_Virtual_NVMe_Disk|P2V-DEST|...)
Etes-vous sûr de vouloir continuer? [y/n]
```

C'est le point de contrôle le plus important : le disque écrasé est bien `nvme0n2`, celui de 80 Go nommé `P2V-DEST`. Le disque du serveur, `nvme0n1`, n'y apparaît pas. Une inversion de la source et de la cible aurait effacé le serveur.

📸 *Capture 11 — Action à la fin du clonage : « -p poweroff »*

![Capture 11 — Action à la fin du clonage : « -p poweroff »](https://hackmd.io/_uploads/SyK-tqrsfl.png)

📸 *Capture 12 — Commande générée par Clonezilla : source nvme0n1*

![Capture 12 — Commande générée par Clonezilla : source nvme0n1](https://hackmd.io/_uploads/rytbYcSizg.png)

📸 *Capture 13 — Confirmation : seul le disque de 80 Go (nvme0n2, P2V-DEST) sera écrasé*

![Capture 13 — Confirmation : seul le disque de 80 Go (nvme0n2, P2V-DEST) sera écrasé](https://hackmd.io/_uploads/HyKWYqSjGg.png)

Pendant la copie de la partition Windows, Clonezilla utilise partclone, qui ne copie que les blocs occupés :

```
Starting to back up device (/dev/nvme0n1p3) to device (/dev/nvme0n2p3)
File system:  NTFS
Device size:   64.8 GB = 15816926 Blocks
Space in use:  21.8 GB = 5323712 Blocks
Free Space:    43.0 GB = 10493214 Blocks
Elapsed: 00:00:40, Remaining: 00:14:03, Completed:   4.53%,   1.48GB/min,
```

Fin du clonage :

```
Ending /usr/sbin/ocs-sr at 2026-10-08 23:42:15 UTC...
"ocs-live-general" finished.
```

| Mesure | Valeur |
|---|---|
| Début de la copie | 23h24 (heure notée au lancement) |
| Fin | 23:42:15 (heure affichée par Clonezilla) |
| Durée | environ 18 minutes |
| Données copiées sur la partition Windows | 21,8 Go sur 64,8 Go |
| Débit affiché pendant la copie | 1,48 Go/min |
| Erreurs | aucune ✅ |

Le « 64,8 GB » de Clonezilla et les 60,34 Go de Windows désignent la même partition : Clonezilla compte en milliards d'octets, Windows en multiples de 1024.

📸 *Capture 14 — Copie de la partition Windows en cours : 21,8 Go utilisés, 1,48 Go/min*

![Capture 14 — Copie de la partition Windows en cours : 21,8 Go utilisés, 1,48 Go/min](https://hackmd.io/_uploads/BkWYWY5Ssze.png)

📸 *Capture 15 — Fin du clonage à 23:42:15, sans erreur*

![Capture 15 — Fin du clonage à 23:42:15, sans erreur](https://hackmd.io/_uploads/rkxY-tcHoGl.png)

### 3.4 Création de la VM migrée

1. Éteindre la VM source, puis dans ses paramètres retirer **Hard Disk 2** (bouton **Remove**). Le fichier du disque reste sur le PC.
2. Créer le dossier `Documents\Virtual Machines\SRV-IMPRESSION-P2V` et y déplacer les fichiers du disque cloné.
3. **File → New Virtual Machine → Custom** : Windows 10 x64, firmware **UEFI**, **Use an existing virtual disk** → choisir le disque cloné.
4. Régler le matériel de la nouvelle VM :

| Paramètre | Ancien serveur | VM migrée | Pourquoi |
|---|---|---|---|
| Processeurs | 2 | 2 | Charge identique |
| Mémoire | 2 Go | **4 Go** | Le minimum pour Windows 10 était juste, la virtualisation permet d'en donner plus |
| Disque | NVMe 61 Go | NVMe 80 Go (disque cloné) | Contrôleur de démarrage identique pour limiter le risque de non-démarrage |
| Carte réseau | Intel 82574L (e1000e) | **vmxnet3** | Carte virtuelle optimisée. Elle simule aussi le changement de carte d'un vrai P2V. |
| Réseau | VMnet10 | VMnet10 | Même réseau, même adresse prévue |
| Lecteur CD | oui | non | Inutile en production |

Le type de carte vmxnet3 ne se choisit pas dans l'assistant. Il se règle dans le fichier `.vmx` de la VM, VM éteinte, avec le Bloc-notes : `ethernet0.virtualDev = "vmxnet3"`.

5. Démarrer la nouvelle VM, ancien serveur éteint (les deux ont la même adresse IP).

Windows démarre sur le disque cloné jusqu'à l'écran de connexion, sans erreur ✅.

📸 *Capture 16 — SRV-IMPRESSION-P2V : 4 Go de RAM, 2 processeurs, disque NVMe 80 Go, VMnet10*

![Capture 16 — SRV-IMPRESSION-P2V : 4 Go de RAM, 2 processeurs, disque NVMe 80 Go, VMnet10](https://hackmd.io/_uploads/rJq-K5Hofx.png)

📸 *Capture 17 — Premier démarrage de la VM migrée : écran de connexion Windows*

![Capture 17 — Premier démarrage de la VM migrée : écran de connexion Windows](https://hackmd.io/_uploads/BJ5-YqSoGx.png)

---

## 4. Comparer le comportement avant/après migration

### 4.1 Comparatif

| Élément | Avant (SRV-IMPRESSION) | Après (SRV-IMPRESSION-P2V) | Résultat |
|---|---|---|---|
| Nom Windows | DESKTOP-BUMKVM8 | DESKTOP-BUMKVM8 | ✅ identique |
| Système | Windows 10 Pro 10.0.19045 | Windows 10 Pro 10.0.19045 | ✅ identique |
| Date d'installation de Windows | 11.09.2026 01:35:57 | 11.09.2026 01:35:57 | ✅ même installation |
| Démarrage | UEFI | UEFI | ✅ identique |
| Mémoire | 2 Go | 4 Go | modifiée volontairement |
| Disque | NVMe 61 Go | NVMe 80 Go | plus grand |
| Partition C: | 60,34 Go, 37,38 Go libres | 60,34 Go, 37,12 Go libres | ✅ même partition |
| Carte réseau | Intel 82574L | vmxnet3 | changée |
| Adresse MAC | 00-0C-29-5F-04-A2 | 00-0C-29-D4-04-3C | nouvelle |
| Identifiant matériel (UUID) | 237B4D56-…-DCDDD75F04A2 | EB3D4D56-…-7F15B6D4043C | nouveau |
| Adresse IP | 10.10.10.12 fixe | 169.254.152.254 au démarrage, puis 10.10.10.12 après correction | ❌ puis ✅ |
| Partage `Partage` | présent | présent | ✅ |
| Imprimante `IMPRIMANTE-PME` | partagée | partagée | ✅ |
| Services Serveur et Spouleur | démarrés | démarrés | ✅ |
| Empreinte SHA-256 de `clients.txt` | 2FAE7A63…D3BAEB1A | 2FAE7A63…D3BAEB1A | ✅ données intactes |
| Logiciels installés | 10 entrées | les mêmes 10 entrées | ✅ |
| Comptes | PC01_DN actif | PC01_DN actif | ✅ |
| Licence Windows | non activée (0xC004F034) | non activée (0xC004F034) | ✅ même état |

### 4.2 Test depuis un poste client

Le test est fait depuis **SRV-FICHIERS** (Windows 11, `10.10.10.11`), sur le même réseau VMnet10, après la correction de l'adresse IP (section 5.1). L'ancien serveur est éteint : seule la VM migrée peut répondre.

```
C:\Users\admin>ping 10.10.10.12

Réponse de 10.10.10.12 : octets=32 temps=2 ms TTL=128
Réponse de 10.10.10.12 : octets=32 temps<1ms TTL=128
Réponse de 10.10.10.12 : octets=32 temps=1 ms TTL=128
Réponse de 10.10.10.12 : octets=32 temps=1 ms TTL=128

Statistiques Ping pour 10.10.10.12:
    Paquets : envoyés = 4, reçus = 4, perdus = 0 (perte 0%),
```

```
C:\Users\admin>powershell Test-NetConnection 10.10.10.12 -Port 445

ComputerName     : 10.10.10.12
RemoteAddress    : 10.10.10.12
RemotePort       : 445
InterfaceAlias   : Ethernet0
SourceAddress    : 10.10.10.11
TcpTestSucceeded : True
```

Le port 445 est celui du partage de fichiers Windows (SMB).

Dans l'Explorateur de SRV-FICHIERS, `\\10.10.10.12\Partage` demande un identifiant d'un compte du serveur, puis affiche le fichier `clients`, daté du 07/10/2026 14:17, comme sur l'ancien serveur.

| Test depuis SRV-FICHIERS | Résultat |
|---|---|
| `ping 10.10.10.12` | ✅ 4 reçus sur 4 |
| Port 445 du serveur | ✅ ouvert (`TcpTestSucceeded : True`) |
| Ouverture de `\\10.10.10.12\Partage` | ✅ fichier `clients` visible |

📸 *Capture 18 — SRV-FICHIERS : ping vers le serveur migré, 4 reçus sur 4*

![Capture 18 — SRV-FICHIERS : ping vers le serveur migré, 4 reçus sur 4](https://hackmd.io/_uploads/HyZtZKcroGg.png)

📸 *Capture 19 — SRV-FICHIERS : port 445 du serveur migré ouvert*

![Capture 19 — SRV-FICHIERS : port 445 du serveur migré ouvert](https://hackmd.io/_uploads/HJet-tqHjzl.png)

📸 *Capture 20 — SRV-FICHIERS : le partage du serveur migré s'ouvre et affiche « clients »*

![Capture 20 — SRV-FICHIERS : le partage du serveur migré s'ouvre et affiche « clients »](https://hackmd.io/_uploads/H1eK-FqBszx.png)

---

## 5. Identifier les éventuels problèmes (pilotes, licences, performances)

### 5.1 Pilotes et réseau

| Constat | Détail | Gravité |
|---|---|---|
| Nouvelle carte réseau | Windows détecte une **nouvelle** carte, « Adaptateur Ethernet vmxnet3 » (pilote VMware 1.9.14.0), nommée `Ethernet0 2` | normal |
| Adresse IP perdue | La nouvelle carte est en DHCP. Sans serveur DHCP sur VMnet10, elle prend une adresse automatique `169.254.152.254`. Le serveur devient injoignable. | ❌ bloquant |
| Carte fantôme | L'ancienne carte Intel 82574L reste connue de Windows, mais absente | à nettoyer |
| Autres fantômes | Contrôleur AHCI SATA (lecteur CD retiré) et second disque NVMe (disque de destination) | sans effet |
| Stockage | Même contrôleur NVMe : Windows démarre sans problème | ✅ |
| Pilote vmxnet3 disponible | VMware Tools était déjà installé sur le serveur : le pilote était présent | ✅ |

Pourquoi l'adresse est perdue : dans Windows, la configuration IP est attachée à la carte réseau, pas à la machine. Une nouvelle carte repart donc de zéro, en DHCP.

Sur un vrai P2V, la carte change toujours (carte physique → carte virtuelle), ce problème est donc systématique. Le contrôleur de disque change aussi souvent. Si Windows n'a pas le pilote du nouveau contrôleur au démarrage, il s'arrête sur un écran bleu `INACCESSIBLE_BOOT_DEVICE`. Garder le même type de contrôleur (ici NVMe) a évité ce risque.

**Correction appliquée sur la VM migrée :**

1. Gestionnaire de périphériques → **Affichage → Afficher les périphériques cachés** → sous « Cartes réseau », clic droit sur **Intel(R) 82574L Gigabit Network Connection** (grisée) → **Désinstaller l'appareil**.
2. Panneau **Connexions réseau** (`ncpa.cpl`) : renommer `Ethernet0 2` en `Ethernet0`.
3. Propriétés de la carte → **Protocole Internet version 4** : adresse `10.10.10.12`, masque `255.255.255.0`, DNS `8.8.8.8` et `1.1.1.1`, comme avant la migration.

| Après correction | Valeur |
|---|---|
| Carte | `Ethernet0`, Adaptateur Ethernet vmxnet3, 00-0C-29-D4-04-3C |
| Adresse | `10.10.10.12/24`, configuration manuelle ✅ |
| Carte Intel fantôme | supprimée ✅ |

Le test de la section 4.2 confirme que le serveur est de nouveau joignable.

### 5.2 Licences

| Constat | Détail |
|---|---|
| État avant | Windows 10 Pro, canal RETAIL, non activé (mode notification, code 0xC004F034) |
| État après | Identique |
| Changement de l'identifiant matériel | UUID `…DCDDD75F04A2` → `…7F15B6D4043C`, MAC et carte réseau changées |

Le serveur n'étant pas activé avant la migration, la perte d'activation n'a pas pu être observée. Les risques restent réels sur un serveur en production :

- l'activation Windows est liée au matériel : après un P2V, Windows peut se considérer sur une nouvelle machine et demander une réactivation ;
- une licence **OEM**, vendue avec le serveur physique, n'est pas transférable sur une autre machine, ni sur une VM ;
- une licence **Retail** ou en volume peut être transférée, mais il faut parfois réactiver ;
- les logiciels dont la licence est liée au matériel (adresse MAC, numéro de série, clé USB de protection) peuvent cesser de fonctionner.

À vérifier avant toute migration réelle : le type de licence de chaque logiciel du serveur.

### 5.3 Performances

| Mesure | Avant | Après | Lecture |
|---|---|---|---|
| Lecture séquentielle du disque C: | 976,62 Mo/s | 969,91 Mo/s | identique |
| Lecture aléatoire du disque C: | 105,97 Mo/s | 192,82 Mo/s | meilleure, une seule mesure |
| Processeur, compression | 127,43 Mo/s | 126,82 Mo/s | identique |
| Durée de démarrage mesurée par Windows | 57,6 s | 69,7 s | plus longue au premier démarrage |

Commandes utilisées pour les débits : `winsat disk -seq -read -drive c`, `winsat disk -ran -read -drive c` et `winsat cpu -compression`. La durée de démarrage se lit dans l'**Observateur d'événements** : Journaux des applications et des services → Microsoft → Windows → Diagnostics-Performance → Opérationnel → événement **100**.

Les performances sont équivalentes. Le démarrage plus long s'explique probablement par le premier démarrage sur le nouveau matériel, pendant lequel Windows installe les nouveaux périphériques. Une seule mesure de chaque côté ne permet pas de conclure à une vraie différence.

### 5.4 Autres points relevés

| Point | Détail |
|---|---|
| Espace disque non utilisé | Le disque fait 80 Go mais C: garde 60,34 Go : environ 19 Go restent non alloués. On peut agrandir C: dans la Gestion des disques. |
| Interruption de service | Avec un clonage à froid, le serveur est arrêté pendant toute la copie : environ 18 minutes ici pour 21,8 Go |
| Doublon d'adresse | L'ancien serveur et la VM migrée ont la même IP et le même nom : ils ne doivent jamais tourner en même temps |

---

## 6. Rédiger un rapport de migration avec risques identifiés

### 6.1 Schéma de la migration

📸 *Capture 21 — Schéma annoté de la migration P2V : avant, outil, après et poste client*

[![Capture 21 — Schéma annoté de la migration P2V : avant, outil, après et poste client](https://hackmd.io/_uploads/rycZF9Sizl.png)](https://hackmd.io/_uploads/rycZF9Sizl.png)

Le même schéma en version texte :

```
PC HÔTE : Windows 11 + VMware Workstation 17 Pro (17.5.2)
|
+-- AVANT : SRV-IMPRESSION (« serveur physique »)
|     Windows 10 Pro 19045 · 2 processeurs · 2 Go
|     Disque NVMe 61 Go (C: 60,34 Go, ~23 Go utilisés)
|     Carte Intel 82574L · MAC 00-0C-29-5F-04-A2
|     IP fixe 10.10.10.12/24 · VMnet10
|     Rôle : Partage + IMPRIMANTE-PME
|     -> snapshot avant-p2v, éteint après la migration
|
|  [Clonezilla 3.3.3 · device-device · à froid]
|  nvme0n1 (61 Go) ---- copie 21,8 Go, ~18 min ----> nvme0n2 (80 Go)
|
+-- APRÈS : SRV-IMPRESSION-P2V
|     Même Windows · 2 processeurs · 4 Go
|     Disque NVMe 80 Go (C: 60,34 Go)
|     Carte vmxnet3 · MAC 00-0C-29-D4-04-3C
|     IP 169.254.x au démarrage -> remise à 10.10.10.12/24
|     Même rôle · même SHA-256 sur clients.txt
|
+-- VMnet10 · 10.10.10.0/24
      SRV-FICHIERS 10.10.10.11 (client)
      ping OK · port 445 OK · \\10.10.10.12\Partage OK
```

### 6.2 Rapport de migration

| Rubrique | Contenu |
|---|---|
| Serveur source | SRV-IMPRESSION (DESKTOP-BUMKVM8), Windows 10 Pro, serveur de fichiers et d'impression |
| Cible | VM SRV-IMPRESSION-P2V sous VMware Workstation 17 Pro |
| Méthode | P2V à froid, clonage disque à disque |
| Outil | Clonezilla live 3.3.3-15 |
| Sauvegarde avant migration | Snapshot `avant-p2v` du serveur source |
| Données copiées | 21,8 Go (partition Windows) |
| Durée de la copie | environ 18 minutes |
| Modifications après migration | Suppression de la carte fantôme, adresse IP fixe remise |
| Validation | Même système, même rôle, mêmes données (SHA-256), accès client OK |
| État final | ✅ VM migrée en service, ancien serveur éteint et conservé |

### 6.3 Risques identifiés

| Risque | Conséquence | Mesure prise ou conseillée |
|---|---|---|
| Inverser la source et la cible du clonage | Le serveur d'origine est écrasé, perte totale | Vérifier sur l'écran de confirmation la taille et le nom du disque écrasé ; snapshot avant la migration ✅ |
| Changement de carte réseau | Adresse IP perdue, serveur injoignable | Noter la configuration IP avant ; la remettre après ; supprimer la carte fantôme ✅ |
| Pilote de stockage absent | Écran bleu INACCESSIBLE_BOOT_DEVICE au démarrage | Garder un contrôleur compatible (NVMe ici) ✅ |
| Licence liée au matériel | Windows ou un logiciel à réactiver ; licence OEM non transférable | Vérifier les licences avant la migration |
| Deux serveurs identiques allumés | Conflit d'adresse IP et de nom sur le réseau | Ne jamais rallumer l'ancien serveur sur le réseau ✅ |
| Interruption de service | Serveur indisponible pendant la copie à froid (~18 min ici) | Migrer en dehors des heures de travail ; prévenir les utilisateurs |
| Coupure pendant la copie | Copie inutilisable, à recommencer | PC branché sur secteur (incident 2) ✅, mise en veille à désactiver |
| Système plus suivi | Windows 10 n'est plus pris en charge par Microsoft depuis octobre 2025 : la VM reste vulnérable | Isoler le serveur (réseau VMnet10), planifier une mise à jour du système |
| Espace disque perdu | 19 Go non alloués | Agrandir C: dans la Gestion des disques |

### 6.4 Limites

- **Migration simulée.** Le « serveur physique » est une VM VMware. Les pilotes d'origine étaient déjà virtuels, donc les problèmes de pilotes d'un vrai P2V (carte réseau, contrôleur RAID, outils du constructeur) n'ont pu être observés que pour la carte réseau.
- **Licence.** Windows n'était pas activé avant la migration : la perte d'activation n'a pas pu être vérifiée.
- **Performances.** Une seule mesure avant et après : les écarts ne sont pas significatifs.
- **Heure de début.** L'heure de début du clonage (23h24) a été notée à la main. L'heure de fin vient de Clonezilla.
- **Converter** n'a pas été testé, faute de compte Broadcom.

---

## Incidents rencontrés

> **Incident 1 — Clonezilla bloqué au démarrage**
> **Constat :** au premier démarrage sur Clonezilla, l'écran reste figé après `Enabling SMP-like behavior after udev was started... cpu1, done!`.
> **Cause :** probablement le mode d'affichage par défaut de Clonezilla avec la carte graphique virtuelle de VMware. Le mode « Safe graphic » a réglé le problème, ce qui va dans ce sens.
> **Résolution :** redémarrage de la VM et choix de **Other modes of Clonezilla live → Safe graphic settings (vga=normal)**. Clonezilla démarre normalement. ✅

📸 *Capture 08 — Clonezilla figé après « cpu1, done! »*

![Capture 08 — Clonezilla figé après « cpu1, done! »](https://hackmd.io/_uploads/HJFWYcSjMl.png)

> **Incident 2 — PC hôte en veille pendant la préparation**
> **Constat :** le lendemain, la VM est revenue sur le Boot Manager et le disque de destination est toujours vide.
> **Cause :** le PC était sur batterie (10 %) et s'est mis en veille avant le début du clonage. Le journal de VMware indique « System suspend detected » à 16:41.
> **Résolution :** PC branché sur secteur, puis démarrage de Clonezilla relancé depuis le Boot Manager. ✅

> **Incident 3 — Clonezilla ne s'éteint pas à la fin**
> **Constat :** malgré le choix `-p poweroff`, Clonezilla termine sur la ligne de commande `user@cl-3.3.3-15:~$` au lieu d'éteindre la VM.
> **Cause :** à la fin du travail, Clonezilla a exécuté l'étape « true » (« The next step: true ») au lieu de l'arrêt.
> **Résolution :** arrêt manuel avec la commande `sudo poweroff`. Le clonage était terminé et sans erreur. ✅

> **Incident 4 — Serveur migré injoignable**
> **Constat :** après la migration, le serveur a l'adresse `169.254.152.254` au lieu de `10.10.10.12`.
> **Cause :** la nouvelle carte vmxnet3 est vue comme une nouvelle carte, sans la configuration IP de l'ancienne.
> **Résolution :** suppression de la carte fantôme et remise de l'adresse fixe (section 5.1). Ping et partage fonctionnent depuis SRV-FICHIERS. ✅

### Tableau récapitulatif des incidents

| N° | Machine | Incident | Cause | Résolution | État |
|---|---|---|---|---|---|
| 1 | SRV-IMPRESSION | Clonezilla figé au démarrage | Mode d'affichage par défaut (cause probable) | Mode Safe graphic (vga=normal) | ✅ |
| 2 | PC hôte | Clonage pas commencé, PC en veille | Batterie faible | PC branché, démarrage relancé | ✅ |
| 3 | SRV-IMPRESSION | VM pas éteinte en fin de clonage | Étape finale « true » au lieu de l'arrêt | `sudo poweroff` | ✅ |
| 4 | SRV-IMPRESSION-P2V | Adresse IP perdue | Nouvelle carte réseau | Carte fantôme supprimée, IP fixe remise | ✅ |

---

## Livrables

| Livrable attendu | Où le trouver dans ce document | État |
|---|---|---|
| La VM migrée fonctionnelle | Sections 3.4 et 4.2 (démarrage, accès depuis un poste client) | ✅ |
| Un comparatif avant/après | Section 4.1, et section 5.3 pour les performances | ✅ |
| Un rapport de migration avec limites et risques | Sections 6.2, 6.3 et 6.4 | ✅ |
