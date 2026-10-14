# Projet 5 — Comparatif d'hyperviseurs pour un cahier des charges client

**Cours :** Virtualisation — Mettre en place et exploiter une plateforme de virtualisation
**Classe :** E1A
**Auteur :** Drilon Nuhija
**Date :** 9 octobre 2026
**Outils testés :** VMware Workstation 17 Pro (17.5.2), VMware ESXi 8.0 Update 3 et ESXi 6.5 Update 3

---

## Contexte

Un client hésite entre Proxmox VE, Hyper-V, VMware et VirtualBox pour virtualiser ses serveurs. Il attend une recommandation argumentée.

Deux hyperviseurs ont été testés en pratique avec le même cas d'usage : **VMware Workstation** (type 2) sur mon PC et **VMware ESXi** (type 1) sur deux serveurs de l'école. Proxmox VE, Hyper-V et VirtualBox sont évalués à partir de leur documentation et de leurs conditions de licence.

### Résumé des résultats

| Élément | Résultat |
|---|---|
| Critères | 5 critères pondérés + 1 critère éliminatoire ✅ |
| Hyperviseurs testés | VMware Workstation 17 Pro (PC) et VMware ESXi 8.0 U3 / 6.5 U3 (serveurs HP et Dell) ✅ |
| Cas d'usage commun | Une VM serveur Windows : création, accès console, snapshot, administration ✅ |
| Différence la plus visible | Workstation dépend de Windows : un ESXi installé dedans n'a pas pu lancer de VM ❌ |
| Mesure comparable | Snapshot avec mémoire : environ 10 min (Workstation, 8 Go) contre 1 h 12 min 45 s (ESXi 6.5, 23,5 Go) |
| Recommandation | **Proxmox VE** avec abonnement Basic, Hyper-V en solution de repli |
| Présentation | Support de 6 diapositives pour 5 minutes ✅ |

> 📸 Les captures servent d'illustration. Toutes les valeurs utiles sont aussi écrites en texte, dans les tableaux.

---

## 1. Définir les critères de comparaison (coût, licences, facilité d'administration, performances, écosystème)

### 1.1 Scénario client retenu

Le formateur n'ayant pas encore donné de scénario, le comparatif est fait pour le cas suivant. **C'est une hypothèse.**

| Élément | Hypothèse |
|---|---|
| Client | PME genevoise d'environ 20 postes |
| Budget | Petit, pas de dépense récurrente importante |
| Compétences | Pas de spécialiste de la virtualisation en interne, un prestataire informatique externe |
| Matériel | Un serveur physique, 2 processeurs de 8 cœurs (16 cœurs) |
| Charges | 2 VM Windows Server (annuaire et fichiers, impression) et 1 VM Linux (application métier) |
| Besoin | Fonctionner en continu, sauvegarder et restaurer facilement |

### 1.2 Critères et pondération

| Critère | Poids | Ce qu'on regarde | Pourquoi ce poids |
|---|---|---|---|
| **Type 1 (éliminatoire)** | — | L'hyperviseur s'installe sur le serveur et tourne sans poste de travail | Un serveur de PME doit tourner jour et nuit, sans session Windows ouverte |
| Coût | 25 % | Prix du logiciel et du support pour le serveur du scénario | Petit budget |
| Licences | 15 % | Simplicité du modèle, risque d'erreur ou de hausse | Personne en interne pour suivre des contrats complexes |
| Facilité d'administration | 25 % | Interface, tâches courantes (créer une VM, snapshot), compétences demandées | Pas de spécialiste |
| Performances | 15 % | Surcoût de l'hyperviseur, adapté à une petite charge | 3 VM légères : tous les type 1 suffisent |
| Écosystème | 20 % | Sauvegarde, support, documentation, prestataires disponibles | Le prestataire doit pouvoir dépanner vite |

Échelle de notes : 1 (mauvais) à 5 (très bon). La note finale est la moyenne pondérée.

---

## 2. Installer et tester au moins 2 hyperviseurs différents avec un cas d'usage identique

### 2.1 Banc de test

| | Plateforme A | Plateforme B | Plateforme C |
|---|---|---|---|
| Machine | Mon PC, Acer Nitro ANV17-41 | Serveur HP ProLiant DL380p Gen8 | Serveur Dell DR4000 |
| Processeur | AMD Ryzen 7 260, 8 cœurs / 16 threads | 2 × Intel Xeon E5-2630 v2, 12 cœurs | 12 CPU Intel Xeon E5645 |
| Mémoire | 32 Go | 192 Go (191,97 Go vus par ESXi) | 47,98 Go |
| Stockage | NVMe 954 Go | Volume logique HP de 558,88 GiB | Datastore VMFS-5 de 2,72 To |
| Système / hyperviseur | Windows 11 Famille + **VMware Workstation 17 Pro 17.5.2** | **VMware ESXi 8.0 Update 3** | **VMware ESXi 6.5 Update 3** (image Dell) |
| Type | 2 | 1 | 1 |
| Adresse de gestion | — | `10.40.10.180` | `10.40.10.181` |

Les serveurs B et C ont été installés à l'école dans le cadre d'un projet de cluster, du 21 au 25 septembre 2026.

**Cas d'usage identique** : héberger une VM serveur Windows et faire les tâches courantes d'un administrateur.

1. Installer l'hyperviseur.
2. Préparer le stockage et l'image ISO.
3. Créer la VM (processeur, mémoire, disque, réseau).
4. Accéder à la console et installer les outils invités (VMware Tools).
5. Prendre un snapshot.
6. Administrer à distance.

Les deux hyperviseurs testés sont du même éditeur (VMware). Ils permettent de comparer clairement un type 2 et un type 1, mais pas deux éditeurs différents (voir 3.4).

📸 *Capture 01 — Schéma du banc de test : les trois plateformes, leurs VM et la limite observée*

![Capture 01 — Schéma du banc de test : les trois plateformes, leurs VM et la limite observée](https://hackmd.io/_uploads/HkB4lE8jMe.png)

Le même schéma en texte :

```
PLATEFORME A : PC (Ryzen 7 260, 32 Go)
  Windows 11 Famille (VBS active)
  └─ VMware Workstation 17 Pro (type 2)
       ├─ SRV-IMPRESSION  Win10   2 vCPU, 2 Go, 61 Go
       ├─ SRV-APPLI       Ubuntu  4 vCPU, 8 Go, 40 Go
       └─ ESXi imbriqué   8.0.3   2 vCPU, 8 Go, 70+50 Go
            └─ VM : ne démarre pas (AMD-V) ❌

PLATEFORME B : HP DL380p Gen8 (2x E5-2630 v2, 192 Go)
  └─ VMware ESXi 8.0 U3 (type 1), 10.40.10.180

PLATEFORME C : Dell DR4000 (12 CPU E5645, 48 Go)
  └─ VMware ESXi 6.5 U3 (type 1), 10.40.10.181
       ├─ Windows Server 2025 181  1 vCPU, 4 Go, 40 Go
       └─ WinServ25_DA  disque 600 Go, mémoire 23,5 Go
```

### 2.2 Test sur VMware Workstation 17 Pro (type 2)

**Installation de l'hyperviseur.** Workstation s'installe comme un logiciel sur Windows (fichier `.exe`, assistant d'installation). Le PC reste utilisable normalement.

| Élément relevé | Valeur |
|---|---|
| Produit | VMware Workstation 17 Pro, version 17.5.2 build-23775571 |
| Licence affichée | Licensed, Commercial Use, No expiration |
| Système hôte | Windows 11 Home 64 bits (build 26200.9457) |
| Mémoire de l'hôte | 32 060 Mo |
| Dossier des VM | `C:\Users\dn040\Documents\Virtual Machines\` |

📸 *Capture 02 — « About VMware Workstation 17 Pro » : version 17.5.2, licence Commercial Use sans expiration*

![Capture 02 — « About VMware Workstation 17 Pro » : version 17.5.2, licence Commercial Use sans expiration](https://hackmd.io/_uploads/Hk2mxE8jfe.png)

📸 *Capture 03 — Ressources du PC hôte : Ryzen 7 260, 32 Go de RAM, 954 Go de stockage*

![Capture 03 — Ressources du PC hôte : Ryzen 7 260, 32 Go de RAM, 954 Go de stockage](https://hackmd.io/_uploads/S13Qx4UsMg.png)

**Stockage et ISO.** Chaque VM est un dossier Windows qui contient son fichier de configuration `.vmx` et son disque `.vmdk`. L'ISO est un fichier du PC choisi directement dans l'assistant.

**Création de la VM.** File → New Virtual Machine → choix du système invité, du processeur, de la mémoire, du disque et du réseau (VMnet).

| VM Windows de test | Valeur |
|---|---|
| Nom | SRV-IMPRESSION |
| Système | Windows 10 Professionnel 64 bits |
| Processeurs | 2 |
| Mémoire | 2 Go |
| Disque | NVMe 61 Go |
| Réseau | Custom (VMnet10) |

📸 *Capture 04 — VM Windows dans Workstation : 2 Go, 2 processeurs, disque NVMe 61 Go, réseau VMnet10*

![Capture 04 — VM Windows dans Workstation : 2 Go, 2 processeurs, disque NVMe 61 Go, réseau VMnet10](https://hackmd.io/_uploads/Hy44eVLiGg.png)

**Console et outils.** La console est la fenêtre de Workstation, avec un onglet par VM. Les VMware Tools s'installent par VM → Install VMware Tools.

📸 *Capture 05 — Trois VM ouvertes en onglets dans Workstation (SRV-IMPRESSION, SRV-APPLI, SRV-FICHIERS)*

![Capture 05 — Trois VM ouvertes en onglets dans Workstation (SRV-IMPRESSION, SRV-APPLI, SRV-FICHIERS)](https://hackmd.io/_uploads/HJ2XgN8jzx.png)

**Snapshot.** VM → Snapshot → Take Snapshot, puis Snapshot Manager pour revenir en arrière.

📸 *Capture 06 — Snapshot Manager : snapshot « avant-mise-a-jour » du 06.10.2026 à 15:24:32*

![Capture 06 — Snapshot Manager : snapshot « avant-mise-a-jour » du 06.10.2026 à 15:24:32](https://hackmd.io/_uploads/rJ6XeEIifg.png)

**Administration à distance.** Aucune dans Workstation : les VM se gèrent depuis le PC où le logiciel est installé.

### 2.3 Test sur VMware ESXi (type 1)

#### Installation de l'hyperviseur (serveur HP)

ESXi s'installe **à la place** du système du serveur.

1. Démarrer le serveur sur l'ISO d'ESXi 8.0.3.
2. Écran d'accueil → **Enter (Continue)**, accepter la licence.
3. Choisir le disque d'installation : `HP LOGICAL VOLUME`, 558,88 GiB.
4. Choisir le clavier, saisir le mot de passe `root`, confirmer avec **F11 (Install)**.
5. Redémarrer, puis dans la console de l'hôte (DCUI, touche **F2**) régler l'adresse IP, le DNS (`10.40.10.50`) et le nom d'hôte `ESXi-GIT-DA`.

| Élément relevé au démarrage du serveur | Valeur |
|---|---|
| BIOS | HP ProLiant System BIOS P70 (07/01/2015) |
| Processeurs | 2 détectés, 12 cœurs au total, Hyper-Threading activé |
| Mémoire | 192 GB |
| Écran ESXi | VMware ESXi 8.0.3 (VMKernel Release Build 24585383), HP ProLiant DL380p Gen8, 2 × Xeon E5-2630 v2, 192 GiB Memory |

📸 *Capture 07 — Démarrage du serveur HP : 192 Go installés, 2 processeurs, 12 cœurs*

![Capture 07 — Démarrage du serveur HP : 192 Go installés, 2 processeurs, 12 cœurs](https://hackmd.io/_uploads/SkNEg48iMl.jpg)

📸 *Capture 08 — Installation d'ESXi : choix du disque HP LOGICAL VOLUME de 558,88 GiB*

![Capture 08 — Installation d'ESXi : choix du disque HP LOGICAL VOLUME de 558,88 GiB](https://hackmd.io/_uploads/HkQExNIsGe.jpg)

📸 *Capture 09 — Console ESXi 8.0.3 sur le serveur HP, avec l'adresse de gestion*

![Capture 09 — Console ESXi 8.0.3 sur le serveur HP, avec l'adresse de gestion](https://hackmd.io/_uploads/r1NExVUsfe.jpg)

Une fois installé, le serveur s'administre depuis n'importe quel PC du réseau, par un navigateur : `https://10.40.10.180/ui`.

| Client web de l'hôte HP | Valeur |
|---|---|
| Version | 8.0 Update 3 |
| CPU | 12 CPUs × Intel Xeon E5-2630 v2 @ 2.60 GHz, 1 % utilisé (197 MHz) |
| Mémoire | 10,2 Go utilisés sur 191,97 Go |
| Stockage | 131,22 Go utilisés sur 430,75 Go |
| Licence | « Vous utilisez actuellement ESXi en mode d'évaluation. Cette licence expirera dans 60 jours. » |

📸 *Capture 10 — Client web ESXi du serveur HP : version 8.0 Update 3, mode d'évaluation de 60 jours*

![Capture 10 — Client web ESXi du serveur HP : version 8.0 Update 3, mode d'évaluation de 60 jours](https://hackmd.io/_uploads/rkmVgNLifg.png)

#### Cas d'usage complet (serveur Dell, ESXi 6.5)

| Client web de l'hôte Dell | Valeur |
|---|---|
| Version | 6.5.0 Update 3 (Build 14320405), image `DellEMC-ESXi-6.5U3-14320405-A03` |
| Matériel | Dell DR4000, 12 CPUs × Intel Xeon E5645 @ 2.40 GHz, 47,98 Go |
| Stockage | 661,68 Go utilisés sur 2,72 To |
| Licence | Mode d'évaluation, 60 jours |

📸 *Capture 11 — Client web ESXi 6.5 du serveur Dell : matériel, version et licence d'évaluation*

![Capture 11 — Client web ESXi 6.5 du serveur Dell : matériel, version et licence d'évaluation](https://hackmd.io/_uploads/rkEVxELsMg.png)

**Stockage et ISO.** Stockage → Navigateur de banque de données → créer un dossier `iso` dans `datastore1` → **Télécharger** l'ISO depuis le PC. L'ISO `windows-server-2025.iso` (7,71 Go) est copiée sur le serveur.

📸 *Capture 12 — Navigateur de banque de données : windows-server-2025.iso (7,71 Go) dans datastore1*

![Capture 12 — Navigateur de banque de données : windows-server-2025.iso (7,71 Go) dans datastore1](https://hackmd.io/_uploads/Hy6mgELsMe.png)

**Création de la VM.** Machines virtuelles → **Créer/Enregistrer une machine virtuelle** : 5 étapes (type, nom et système invité, stockage, paramètres, résumé).

| Paramètre de l'assistant | Valeur |
|---|---|
| Nom | Windows Server 2025 181 |
| Compatibilité | Machine virtuelle ESXi 6.5 |
| CPU | 1 |
| Mémoire | 4096 Mo |
| Disque dur 1 | 40 Go |
| Contrôleur SCSI | LSI Logic SAS |
| Réseau | VM Network, connecté |
| Lecteur CD/DVD | Connecté au démarrage |

📸 *Capture 13 — Assistant de création : 1 CPU, 4096 Mo, 40 Go, LSI Logic SAS, VM Network*

![Capture 13 — Assistant de création : 1 CPU, 4096 Mo, 40 Go, LSI Logic SAS, VM Network](https://hackmd.io/_uploads/r1MNlVUifg.png)

📸 *Capture 14 — La VM apparaît dans la liste, tâche « Create VM » terminée*

![Capture 14 — La VM apparaît dans la liste, tâche « Create VM » terminée](https://hackmd.io/_uploads/SJWElN8sGl.png)

**Console et outils.** La console s'ouvre dans le navigateur ou avec **VMware Remote Console (VMRC)**. Windows Server 2025 y a été installé, puis les VMware Tools (tâche « Mount Tools Installer »).

📸 *Capture 15 — Console VMRC : Gestionnaire de serveur de Windows Server dans la VM*

![Capture 15 — Console VMRC : Gestionnaire de serveur de Windows Server dans la VM](https://hackmd.io/_uploads/r1mEgEIsGg.png)

**Snapshot.** Clic droit sur la VM → **Snapshots → Prendre un snapshot**.

📸 *Capture 16 — Menu Snapshots de la VM dans le client web ESXi*

![Capture 16 — Menu Snapshots de la VM dans le client web ESXi](https://hackmd.io/_uploads/rybVg4UoGl.png)

| Tâches relevées le 24.09.2026 | Heure | Résultat |
|---|---|---|
| Auto Start Power On | 10:53:46 | ✅ Terminé |
| Power On VM | 10:57:41 | ✅ Terminé |
| Mount Tools Installer | 10:58:18 | ✅ Terminé |
| Create Snapshot | 11:15:18 | en cours, 4 % |

📸 *Capture 17 — Tâches récentes : démarrage de la VM, VMware Tools, snapshot en cours*

![Capture 17 — Tâches récentes : démarrage de la VM, VMware Tools, snapshot en cours](https://hackmd.io/_uploads/Sk1NlVIjMg.png)

**Administration à distance.** ESXi s'administre aussi en SSH (service à activer sur l'hôte) :

```
C:\Users\dn040>ssh root@10.40.10.181
[root@GIT-DA:~] df -h | grep -i vmfs
VMFS-5   2.7T  998.0G   1.7T  36% /vmfs/volumes/datastore1
[root@GIT-DA:~] vim-cmd vmsvc/getallvms
Vmid  Name          File                                     Guest OS                 Version
1     WinServ25_DA  [datastore1] WinServ25_DA/WinServ25_DA.vmx windows9Server64Guest  vmx-13
```

📸 *Capture 18 — Session SSH sur l'hôte Dell : datastore VMFS-5 et liste des VM*

![Capture 18 — Session SSH sur l'hôte Dell : datastore VMFS-5 et liste des VM](https://hackmd.io/_uploads/Hyz4xNIsGx.png)

Le journal de la VM donne la durée exacte d'un snapshot avec mémoire, sur la VM `WinServ25_DA` :

```
2026-09-25T00:57:20.310Z| vmx| SnapshotVMX_TakeSnapshot start: 'snapshot avant màj 00:41'
2026-09-25T00:57:22.066Z| vcpu-0| MainMem: Writing full memory image
2026-09-25T02:10:05.491Z| vmx| SnapshotVMXTakeSnapshotComplete: Done with snapshot
[root@GIT-DA:~] vim-cmd vmsvc/snapshot.get 1
--Snapshot Name        : snapshot avant màj 00:41
--Snapshot Created On  : 9/25/2026 0:57:20
--Snapshot State       : powered on
```

Fichiers de la VM après le snapshot : disque `WinServ25_DA-flat.vmdk` de 600,0 Go, image mémoire `WinServ25_DA-Snapshot1.vmem` de 23,5 Go, disque delta de 5,8 Go.

📸 *Capture 19 — Journal de la VM : début du snapshot à 00:57:20, fin à 02:10:05*

![Capture 19 — Journal de la VM : début du snapshot à 00:57:20, fin à 02:10:05](https://hackmd.io/_uploads/Hy7VeEUozx.png)

### 2.4 Essai complémentaire : ESXi dans Workstation

Pour avoir les deux hyperviseurs sur la même machine, ESXi 8.0.3 a été installé dans une VM de Workstation (virtualisation imbriquée).

| Paramètre de la VM ESXi | Valeur |
|---|---|
| Processeurs | 2, option **Virtualize Intel VT-x/EPT or AMD-V/RVI** cochée |
| Mémoire | 8 Go |
| Disques | 70 Go (système) + 50 Go (datastore) |
| Réseau | NAT |

| Étape | Résultat |
|---|---|
| Démarrage de l'installation | ✅ ESXi 8.0.3 voit « VMware20,1 », 2 × AMD Ryzen 7 260, 8 GiB |
| Analyse du système | ⚠️ `HARDWARE_VIRTUALIZATION WARNING: Hardware Virtualization is not a feature of the CPU, or is not enabled in the BIOS` |
| Installation | ✅ « ESXi 8.0.3 has been installed successfully », évaluation de 60 jours |
| Client web | ✅ ESXi à vide (0 VM) : 310 MHz de CPU et 1,66 Go de mémoire utilisés sur 8 Go |
| Démarrage d'une VM dans ESXi | ❌ « Cet hôte ne prend pas en charge la technologie AMD-V » |

📸 *Capture 20 — ESXi 8.0.3 démarre dans une VM Workstation : VMware20,1, Ryzen 7 260, 8 GiB*

![Capture 20 — ESXi 8.0.3 démarre dans une VM Workstation : VMware20,1, Ryzen 7 260, 8 GiB](https://hackmd.io/_uploads/rkk4xNUsGx.png)

📸 *Capture 21 — Avertissement de l'installateur : virtualisation matérielle non disponible*

![Capture 21 — Avertissement de l'installateur : virtualisation matérielle non disponible](https://hackmd.io/_uploads/r1CmlN8iMg.png)

📸 *Capture 22 — Installation d'ESXi terminée dans Workstation*

![Capture 22 — Installation d'ESXi terminée dans Workstation](https://hackmd.io/_uploads/B1CXgVUsMe.png)

📸 *Capture 23 — Client web de l'ESXi imbriqué à vide : 1,66 Go de mémoire utilisés*

![Capture 23 — Client web de l'ESXi imbriqué à vide : 1,66 Go de mémoire utilisés](https://hackmd.io/_uploads/rJV4eNUoGl.png)

📸 *Capture 24 — Échec du démarrage d'une VM : « Cet hôte ne prend pas en charge la technologie AMD-V »*

![Capture 24 — Échec du démarrage d'une VM : « Cet hôte ne prend pas en charge la technologie AMD-V »](https://hackmd.io/_uploads/ryyEeNUizl.png)

La cause est du côté de Windows (voir l'incident 1) : Workstation affiche « VMware Workstation does not support nested virtualization on this host. Module 'HV' power on failed. »

📸 *Capture 25 — Workstation : virtualisation imbriquée non prise en charge sur cet hôte*

![Capture 25 — Workstation : virtualisation imbriquée non prise en charge sur cet hôte](https://hackmd.io/_uploads/rk1VlVUoMl.png)

📸 *Capture 26 — Informations système du PC : Windows 11 Famille, sécurité basée sur la virtualisation « En cours d'exécution »*

![Capture 26 — Informations système du PC : Windows 11 Famille, sécurité basée sur la virtualisation « En cours d'exécution »](https://hackmd.io/_uploads/BkNEeV8oGl.png)

---

## 3. Mesurer/observer les différences notables

### 3.1 Déroulement du cas d'usage

| Étape | Workstation (type 2) | ESXi (type 1) |
|---|---|---|
| Installation | Logiciel sur Windows, le PC reste un poste de travail | Remplace le système du serveur, réglages dans la console DCUI |
| Matériel visé | PC de bureau ou portable | Serveur certifié (image standard sur le HP, image Dell sur le Dell) |
| Stockage | Dossier Windows, fichiers `.vmx` et `.vmdk` | Banque de données VMFS, à créer sur un disque libre |
| ISO | Fichier local choisi dans l'assistant | Copié d'abord dans le datastore (7,71 Go) |
| Création de VM | Assistant de Workstation | Assistant web en 5 étapes, très proche |
| Console | Onglet dans la fenêtre de Workstation | Navigateur ou VMRC, depuis n'importe quel PC |
| Snapshots | Snapshot Manager | Menu Snapshots ou `vim-cmd` en SSH |
| Administration à distance | Non | Oui : navigateur, SSH |
| Fonctionnement en continu | Non : si le PC se met en veille, les VM s'arrêtent | Oui : serveur dédié, démarrage automatique des VM |
| Licence observée | Commercial Use, sans expiration | Évaluation de 60 jours sur les trois hôtes ESXi |

### 3.2 Mesures relevées

| Mesure | Workstation | ESXi | Source |
|---|---|---|---|
| Mémoire de l'hôte | 32 Go | 192 Go (HP), 47,98 Go (Dell) | Captures 03, 10, 11 |
| Mémoire prise par l'hyperviseur sans VM | non mesurée (Windows + Workstation) | 1,66 Go (ESXi imbriqué, 0 VM) | Capture 23 |
| Snapshot d'une VM allumée, mémoire comprise | environ 10 min, VM de 8 Go de RAM | 1 h 12 min 45 s, image mémoire de 23,5 Go | Capture 19 et mesures de mes VM Workstation |
| Restauration d'un snapshot | 16 à 17 s | non mesurée | Mesure de mes VM Workstation |
| Démarrage de Windows dans la VM | 57,6 s (Windows 10) | non mesuré | Observateur d'événements, événement 100 |
| Lecture séquentielle du disque dans la VM | 976,62 Mo/s (NVMe du PC) | non mesurée | `winsat disk -seq -read -drive c` |
| Lancer une VM dans une VM | ❌ impossible sur ce PC | sans objet | Captures 24 à 26 |

### 3.3 Différences notables

1. **Dépendance au système hôte.** Workstation passe par Windows. Sur ce PC, la sécurité basée sur la virtualisation de Windows 11 occupe la virtualisation matérielle : Workstation ne peut plus la donner à une VM, et l'ESXi imbriqué ne lance aucune VM. ESXi, lui, a un accès direct au processeur.
2. **Accès.** Workstation ne se gère que devant le PC. ESXi se gère depuis le réseau (navigateur, VMRC, SSH), ce qui convient à un serveur rangé dans une baie.
3. **Disponibilité.** Avec Workstation, les VM s'arrêtent quand le PC se met en veille ou redémarre. Un hôte ESXi tourne en continu.
4. **Snapshots.** Le snapshot avec mémoire prend du temps sur les deux hyperviseurs, et la durée dépend surtout de la quantité de RAM à écrire et des disques : environ 10 minutes pour 8 Go sur le NVMe du PC, 1 h 12 pour 23,5 Go sur le serveur Dell.
5. **Surcoût de l'hyperviseur.** ESXi à vide n'utilise que 1,66 Go de mémoire. Sur le PC, Windows 11 et ses programmes restent chargés en plus des VM.
6. **Prise en main.** L'assistant de création de VM est presque le même dans Workstation et dans le client web d'ESXi. Passer de l'un à l'autre est facile.

### 3.4 Limites des tests

- **Même éditeur.** Les deux hyperviseurs testés sont des produits VMware. Proxmox VE, Hyper-V et VirtualBox n'ont pas été installés : Hyper-V n'existe pas sur Windows 11 Famille, et Proxmox n'a pas été installé : le projet s'appuie sur les tests ESXi déjà faits sur de vrais serveurs.
- **Matériels différents.** PC portable récent contre serveurs de plus de dix ans : les chiffres de performance ne se comparent pas directement.
- **Mesures uniques.** Une seule mesure par cas, sans répétition.
- **Snapshots différents.** 8 Go de RAM d'un côté, 23,5 Go de l'autre : la comparaison montre un ordre de grandeur, pas un écart de performance.
- **ESXi imbriqué.** Le problème de virtualisation imbriquée n'a pas été résolu (incident 1).

---

## 4. Construire un tableau comparatif

### 4.1 Licences et coûts (vérifiés le 9 octobre 2026)

| Hyperviseur | Type | Licence | Coût |
|---|---|---|---|
| **VMware vSphere / ESXi** (Broadcom) | 1 | Abonnement uniquement. Depuis le 10.04.2025, minimum de **72 cœurs par commande** et 16 cœurs par processeur. Les offres Essentials ont été retirées | Pas de prix public, devis via un revendeur. ESXi gratuit revenu avec la 8.0 U3e (avril 2025) : sans support, 8 vCPU maximum par VM |
| **VMware Workstation Pro** | 2 | Gratuit pour tous, y compris en entreprise, depuis novembre 2024 | 0, sans support par ticket |
| **Proxmox VE** | 1 | Logiciel libre (AGPLv3), basé sur Debian, KVM et conteneurs LXC | 0. Abonnement facultatif par processeur et par an : Community 120 €, Basic 370 €, Standard 550 €, Premium 1 100 € |
| **Hyper-V** (Microsoft) | 1 | Rôle inclus dans Windows Server. Pas sur Windows 11 Famille. Le Hyper-V Server gratuit n'a pas eu de version 2022 | Windows Server 2025 Standard, 16 cœurs : environ 1 176 $ (2 VM Windows). Datacenter : environ 6 771 $ (VM illimitées). Licences d'accès client (CAL) en plus |
| **VirtualBox** (Oracle) | 2 | Base libre (GPL). Extension Pack sous licence PUEL : gratuit pour un usage personnel, scolaire ou 30 jours d'essai | Extension Pack payant en entreprise |

Remarque importante : les VM Windows Server du client demandent des licences Windows Server **quel que soit l'hyperviseur**. Avec Hyper-V, le rôle est compris dans cette licence. Avec Proxmox ou ESXi, on paie la licence Windows Server pour les VM, plus l'éventuel abonnement de l'hyperviseur.

### 4.2 Coût pour le scénario (1 serveur, 2 processeurs, 16 cœurs)

| Hyperviseur | Coût de l'hyperviseur | Remarque |
|---|---|---|
| Proxmox VE | 0 €, ou 240 €/an (Community), ou **740 €/an (Basic, 2 processeurs)** | Le support Basic donne 3 tickets par an |
| Hyper-V | 0 € en plus de Windows Server Standard | Standard couvre 2 VM Windows : juste assez pour le scénario |
| VMware vSphere | Abonnement de 72 cœurs minimum pour un serveur qui en a 16 | 56 cœurs payés sans être utilisés, prix sur devis |
| VMware ESXi gratuit | 0 € | Sans support : risqué pour une entreprise |
| Workstation, VirtualBox | — | Éliminés : type 2 |

### 4.3 Tableau comparatif noté

T = testé en pratique, D = évalué sur la documentation.

| Critère (poids) | VMware vSphere (T) | Proxmox VE (D) | Hyper-V (D) | Workstation (T) | VirtualBox (D) |
|---|---|---|---|---|---|
| Type 1 (éliminatoire) | ✅ | ✅ | ✅ | ❌ type 2 | ❌ type 2 |
| Coût (25 %) | 1 — 72 cœurs minimum | 5 — gratuit, support à 370 €/processeur | 4 — compris dans Windows Server | — | — |
| Licences (15 %) | 1 — abonnement, règles qui changent | 5 — libre, abonnement facultatif | 2 — cœurs, nombre de VM, CAL | — | — |
| Administration (25 %) | 4 — client web simple, testé | 3 — interface web, base Linux | 4 — outils Windows connus | — | — |
| Performances (15 %) | 4 | 4 | 4 | — | — |
| Écosystème (20 %) | 5 — le plus répandu, beaucoup d'outils | 4 — sauvegarde intégrée, grande communauté | 4 — Microsoft, nombreux prestataires | — | — |
| **Note pondérée / 5** | **3,00** | **4,15** | **3,70** | éliminé | éliminé |

Calcul pour Proxmox VE : 5 × 0,25 + 5 × 0,15 + 3 × 0,25 + 4 × 0,15 + 4 × 0,20 = 4,15.

Les performances ont la même note pour les trois type 1 : pour 3 VM légères, aucun ne sera limitant, et les tests ne permettent pas de les départager.

---

## 5. Rédiger une recommandation argumentée pour un scénario client donné par le formateur

### Recommandation : Proxmox VE, avec l'abonnement Basic

**Pour le client** (PME de 20 postes, petit budget, sans spécialiste, hypothèse de la section 1.1) :

1. **Le coût le plus bas pour un vrai hyperviseur de serveur.** Le logiciel est gratuit. L'abonnement Basic (740 €/an pour 2 processeurs) ajoute le dépôt stable et 3 demandes de support par an. VMware demanderait un abonnement de 72 cœurs pour un serveur de 16 cœurs.
2. **Une licence simple.** Pas de comptage de cœurs ni de nombre de VM pour l'hyperviseur. Le risque de hausse brutale, vécu par les clients VMware depuis le rachat par Broadcom, n'existe pas.
3. **Une administration proche de ce qui a été testé.** Comme ESXi, Proxmox est un type 1 géré par une interface web : création de VM par assistant, console dans le navigateur, snapshots. Les tests sur ESXi montrent que ce mode de travail convient même sans spécialiste.
4. **La sauvegarde comprise.** Proxmox sauvegarde les VM sans logiciel en plus, ce qui couvre le besoin de restauration.
5. **Les VM Linux et Windows.** L'application métier sous Linux et les serveurs Windows tournent sur le même hôte.

**Points d'attention :**

- Proxmox repose sur Debian : le prestataire doit connaître Linux pour les mises à jour et les dépannages.
- Les 2 VM Windows Server ont toujours besoin d'une licence Windows Server Standard pour les 16 cœurs.
- Proxmox n'a pas été testé dans ce projet. Avant la mise en production, je conseille un essai de deux semaines sur le serveur du client avec une VM de test.

**Solution de repli : Hyper-V**, si le prestataire du client travaille surtout avec Microsoft ou si le client ne veut que du Windows. Le rôle est compris dans Windows Server Standard, déjà nécessaire pour les VM. Mais au-delà de 2 VM Windows, il faut racheter une licence Standard complète, et le modèle de licence est plus compliqué.

**Pourquoi pas VMware.** C'est la solution la plus aboutie, et celle que j'ai testée sur deux serveurs. Mais le minimum de 72 cœurs et l'abonnement obligatoire la rendent trop chère pour une PME d'un seul serveur. L'ESXi gratuit n'a pas de support.

**Pourquoi pas VirtualBox ni Workstation.** Ce sont des hyperviseurs de type 2, faits pour un poste de travail. Les tests le montrent : ils dépendent du système hôte (incident 1) et les VM s'arrêtent avec le PC.

---

## 6. Préparer une courte présentation (5 min) des résultats

Le support de présentation est en ligne : **[Comparatif d'hyperviseurs — présentation](https://claude.ai/artifact/FLit6UtTvss2tGqMfJXqY4)**

| Diapositive | Contenu | Durée |
|---|---|---|
| 1 | Titre, besoin du client et scénario | 30 s |
| 2 | Critères et pondération | 45 s |
| 3 | Banc de test : Workstation contre ESXi, même cas d'usage | 1 min |
| 4 | Différences observées et mesures | 1 min |
| 5 | Tableau comparatif noté | 45 s |
| 6 | Recommandation : Proxmox VE, repli Hyper-V | 1 min |

---

## Incidents rencontrés

> **Incident 1 — Aucune VM ne démarre dans l'ESXi installé dans Workstation**
> **Constat :** dans l'ESXi imbriqué, la mise sous tension d'une VM échoue : « Cet hôte ne prend pas en charge la technologie AMD-V ». Workstation affiche ensuite « does not support nested virtualization on this host. Module 'HV' power on failed. »
> **Cause probable :** sur Windows 11, la sécurité basée sur la virtualisation est « En cours d'exécution ». Windows garde alors la virtualisation matérielle du processeur pour lui, et Workstation ne peut plus la donner à une VM.
> **Résolution :** option **Virtualize Intel VT-x/EPT or AMD-V/RVI** cochée dans la VM, puis tentative de désactivation de la sécurité basée sur la virtualisation. Elle restait « En cours d'exécution » après la tentative. ❌ Non résolu : le test complet a été fait sur les serveurs physiques à la place.

📸 *Capture 27 — Paramètres de la VM ESXi : 8 Go, 2 processeurs, deux disques, option AMD-V cochée*

![Capture 27 — Paramètres de la VM ESXi : 8 Go, 2 processeurs, deux disques, option AMD-V cochée](https://hackmd.io/_uploads/H1JVgNLsMg.png)

> **Incident 2 — Pas de disque pour créer la banque de données**
> **Constat :** dans l'ESXi imbriqué, l'assistant « Nouvelle banque de données » affiche « Aucun périphérique disposant d'espace libre ».
> **Cause :** le seul disque de la VM (70 Go) contient déjà ESXi.
> **Résolution :** ajout d'un second disque virtuel de 50 Go à la VM. La banque de données « Stockage ESXi-01 » a pu être créée. ✅

📸 *Capture 28 — Nouvelle banque de données : aucun périphérique disposant d'espace libre*

![Capture 28 — Nouvelle banque de données : aucun périphérique disposant d'espace libre](https://hackmd.io/_uploads/SymVxVUsGg.png)

> **Incident 3 — Option des compteurs de performance refusée**
> **Constat :** Workstation refuse de démarrer la VM : « does not support virtualized performance counters on this host. Module 'VPMC' power on failed. »
> **Cause :** l'option **Virtualize CPU performance counters** n'est pas utilisable sur cet hôte.
> **Résolution :** option décochée dans les paramètres du processeur de la VM. ✅

📸 *Capture 29 — Workstation : compteurs de performance virtualisés non pris en charge*

![Capture 29 — Workstation : compteurs de performance virtualisés non pris en charge](https://hackmd.io/_uploads/H1y4eV8ofe.png)

> **Incident 4 — Tâche bloquée sur l'hôte ESXi Dell**
> **Constat :** une tâche de copie (`vim.FileManager.copyFile`) reste « running » à 51 % dans la liste des tâches de l'hôte.
> **Cause :** la tâche ne progresse plus dans l'agent de gestion de l'hôte (`hostd`).
> **Résolution :** en SSH, redémarrage de l'agent avec `/etc/init.d/hostd restart`. La tâche disparaît de la liste, et la VM reste allumée (`vim-cmd vmsvc/power.getstate 1` → `Powered on`) : redémarrer l'agent de gestion n'arrête pas les VM. ✅

📸 *Capture 30 — Redémarrage de hostd en SSH, la liste des tâches est vidée*

![Capture 30 — Redémarrage de hostd en SSH, la liste des tâches est vidée](https://hackmd.io/_uploads/r1bNlNUsfx.png)

### Tableau récapitulatif des incidents

| N° | Machine | Incident | Cause | Résolution | État |
|---|---|---|---|---|---|
| 1 | ESXi dans Workstation | VM impossible à démarrer | Sécurité basée sur la virtualisation de Windows 11 | Option AMD-V cochée, désactivation de VBS tentée | ❌ |
| 2 | ESXi dans Workstation | Pas de disque pour le datastore | Disque unique occupé par ESXi | Second disque de 50 Go | ✅ |
| 3 | VM ESXi dans Workstation | Refus de démarrer (VPMC) | Option non prise en charge | Option décochée | ✅ |
| 4 | Hôte ESXi Dell | Tâche de copie bloquée | Agent hostd | `hostd restart` en SSH | ✅ |

---

## Livrables

| Livrable attendu | Où le trouver dans ce document | État |
|---|---|---|
| Tableau comparatif | Section 4 (licences et coûts, coût du scénario, tableau noté) | ✅ |
| Preuves de test (captures, mesures) | Sections 2 et 3, captures 01 à 30 | ✅ |
| Recommandation argumentée écrite | Section 5 | ✅ (sur un scénario supposé) |
| Support de présentation court | Section 6 et lien vers la présentation | ✅ |

---

## Sources

- [Proxmox VE — Pricing](https://www.proxmox.com/en/proxmox-ve/pricing)
- [Proxmox Virtual Environment — Wikipedia](https://en.wikipedia.org/wiki/Proxmox_Virtual_Environment)
- [The Register — VMware revives its free ESXi hypervisor](https://www.theregister.com/2025/04/14/vmware_free_esxi_returns/)
- [Virtualization Howto — VMware ESXi free edition available again with 8.0U3e](https://www.virtualizationhowto.com/2025/04/vmware-esxi-free-edition-available-again-with-8-0u3e-dow)
- [Dutch IT Channel — VMware verhoogt minimum aantal cores van 16 naar 72](https://dutchitchannel.nl/news/603494/vmware-verhoogt-minimum-aantal-cores-voor-licenties-van-16-naar-72)
- [V2 Cloud — VMware vSphere Pricing](https://v2cloud.com/blog/vmware-vsphere-licensing-and-costs)
- [Virtualization Howto — VMware Workstation Pro now free for all users](https://virtualizationhowto.com/2024/11/vmware-workstation-pro-now-free-for-all-users-even-businesses)
- [Licenseware — Windows Server 2025 Licensing Guide](https://licenseware.io/microsoft-windows-server-2025-licensing-guide/)
- [Microsoft Tech Community — Hyper-V Server 2022](https://techcommunity.microsoft.com/discussions/windowsserverinsiders/hyper-v-server-2022/2652790)
- [Microsoft Q&A — Hyper-V absent des fonctionnalités Windows](https://learn.microsoft.com/en-us/answers/questions/1494424/dont-have-hyper-v-on-windows-features)
- [Licenseware — Oracle VirtualBox Licensing Guide](https://licenseware.io/oracle-virtualbox-licensing-guide/)
