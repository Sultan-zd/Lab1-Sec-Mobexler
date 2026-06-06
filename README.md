# 🔬 Lab 1 — Mise en place de l'environnement de test mobile (Mobexler)

> **Module :** Sécurité des applications mobiles  
> **Objectif :** Installer, configurer et valider un laboratoire de pentesting mobile complet et reproductible.

---

## 📑 Table des matières

- [Glossaire](#-glossaire)
- [Objectifs pédagogiques](#-objectifs-pédagogiques)
- [Prérequis](#-prérequis)
- [Architecture du lab](#-architecture-du-lab)
- [Étape 1 — Téléchargement de Mobexler (OVA)](#étape-1--téléchargement-de-mobexler-ova)
- [Étape 2 — Import dans VirtualBox / VMware](#étape-2--import-dans-virtualbox--vmware)
- [Étape 3 — Premier démarrage et connexion](#étape-3--premier-démarrage-et-connexion)
- [Étape 4 — Vérification réseau (tests de santé)](#étape-4--vérification-réseau-tests-de-santé)
- [Étape 5 — Création du snapshot CLEAN](#étape-5--création-du-snapshot-clean)
- [Étape 6 — Préparation de la cible Android](#étape-6--préparation-de-la-cible-android)
- [Dépannage](#-dépannage)
- [Checklist finale](#-checklist-finale)
- [Références](#-références)

---

## 📖 Glossaire

| Terme | Définition |
|---|---|
| **Machine virtuelle (VM)** | Ordinateur « logiciel » exécuté dans VirtualBox/VMware, isolé du système hôte. |
| **OVA / OVF** | Format d'export/import d'une VM — image prête à l'emploi contenant la configuration matérielle + disque virtuel. |
| **Snapshot** | « Photo » de l'état complet d'une VM à un instant T, permettant un retour arrière immédiat. |
| **NAT** | Mode réseau où la VM accède à Internet via l'hôte (IP privée, transparent et stable). |
| **Host-Only** | Réseau privé entre l'hôte et la VM (et éventuellement d'autres VMs/appareils), sans accès Internet direct. |
| **ADB (Android Debug Bridge)** | Outil CLI officiel Android pour communiquer avec un appareil/émulateur (installer APK, lire logs, shell distant). |
| **Proxy** | Intermédiaire réseau configurable sur la cible pour observer, intercepter et modifier le trafic HTTP(S). |
| **Mobexler** | Distribution Linux préconfigurée dédiée au pentesting d'applications mobiles (Android & iOS). |

---

## 🎯 Objectifs pédagogiques

À la fin de ce lab, l'environnement doit permettre de :

1. ✅ **Démarrer Mobexler** sans erreur et accéder à Internet (via NAT).
2. ✅ **Communiquer avec une cible Android** sur un réseau isolé « lab » (Host-Only).
3. ✅ **Revenir à un état propre** grâce au snapshot `CLEAN_BASELINE_TP1`.
4. ✅ **Documenter les éléments critiques** (versions, IP, ADB, proxy) pour reproductibilité.

---

## ⚙️ Prérequis

| Composant | Exigence |
|---|---|
| **Virtualisation** | VT-x (Intel) ou AMD-V activé dans le BIOS/UEFI |
| **Hyperviseur** | VirtualBox ≥ 7.x **ou** VMware Workstation/Player |
| **RAM** | ≥ 4 Go (8 Go recommandé) |
| **Disque** | ~25 Go d'espace libre |
| **ADB** | Inclus dans Mobexler ; sinon installer Android SDK Platform Tools |
| **Cible Android** | Appareil test dédié **ou** émulateur (Genymotion recommandé) |

> [!WARNING]
> **Toujours utiliser une cible dédiée test** (pas un téléphone personnel).  
> Certaines manipulations (proxy, certificats CA custom, etc.) modifient le comportement système et peuvent compromettre la sécurité de l'appareil.

---

## 🏗 Architecture du lab

```
┌──────────────────────────────────────────────────────┐
│                    MACHINE HÔTE                      │
│                                                      │
│  ┌──────────────────────┐   ┌─────────────────────┐  │
│  │     Mobexler VM      │   │   Cible Android     │  │
│  │  ┌────────────────┐  │   │  (Appareil / Ému.)  │  │
│  │  │  Adapter 1     │──┼───┼──► Internet (NAT)   │  │
│  │  │  (NAT)         │  │   │                     │  │
│  │  ├────────────────┤  │   │                     │  │
│  │  │  Adapter 2     │◄─┼───┼──► Réseau Lab       │  │
│  │  │  (Host-Only)   │  │   │  (Host-Only)        │  │
│  │  └────────────────┘  │   └─────────────────────┘  │
│  │                      │                            │
│  │  Outils : ADB,      │                            │
│  │  Burp, Frida, drozer │                            │
│  └──────────────────────┘                            │
└──────────────────────────────────────────────────────┘
```

| Interface | Rôle | Plage IP typique |
|---|---|---|
| **NAT** | Accès Internet (mises à jour, téléchargements) | `10.0.2.x` |
| **Host-Only** | Communication lab isolée (hôte ↔ VM ↔ cible) | `192.168.56.x` |

---

## Étape 1 — Téléchargement de Mobexler (OVA)

### 1.1 Télécharger l'image

📥 **Lien officiel (Google Drive) :**  
[Mobexler OVA — Téléchargement direct](https://drive.google.com/file/d/1rd8g3bmK_XMTtb6PlcfIwjyoJ-mEhAk5/view?usp=sharing)

### 1.2 Vérifier l'intégrité du fichier (recommandé)

Calculer le hash SHA-256 et le comparer au hash officiel (si fourni) :

**Windows (PowerShell) :**
```powershell
Get-FileHash .\Mobexler.ova -Algorithm SHA256
```

**Linux / macOS :**
```bash
sha256sum Mobexler.ova
```

> [!TIP]
> La vérification du hash garantit que le fichier n'a pas été corrompu lors du téléchargement et qu'il correspond à l'image officielle attendue. C'est une pratique essentielle en sécurité.

### ✅ Point de contrôle

- [ ] Fichier `.ova` présent sur le disque
- [ ] Taille du fichier cohérente (plusieurs Go)
- [ ] Hash SHA-256 vérifié (si référence disponible)

---

## Étape 2 — Import dans VirtualBox / VMware

### 2.1 Importer l'appliance

#### VirtualBox

1. Ouvrir **VirtualBox** → `File` → `Import Appliance`
2. Sélectionner le fichier `.ova` téléchargé
3. Cliquer sur **Import** (garder les paramètres par défaut)

#### VMware

1. Ouvrir **VMware** → `File` → `Open`
2. Sélectionner le fichier `.ova`
3. Confirmer l'import

### 2.2 Configurer les interfaces réseau (double NIC)

Après l'import, accéder aux paramètres de la VM :

| Adaptateur | Mode | Rôle |
|---|---|---|
| **Adapter 1** | `NAT` | Accès Internet |
| **Adapter 2** | `Host-Only Adapter` | Réseau lab isolé |

**Navigation VirtualBox :**  
`VM` → `Settings` → `Network`

> [!NOTE]
> Si l'option **Host-Only Adapter** n'apparaît pas dans la liste :
> 1. Aller dans `VirtualBox` → `Tools` → `Network Manager`
> 2. Onglet **Host-Only Networks** → cliquer sur **Create**
> 3. Valider la configuration par défaut (généralement `192.168.56.1/24` avec DHCP activé)
> 4. Retourner dans les paramètres réseau de la VM

### ✅ Point de contrôle

- [ ] VM importée visible dans la liste
- [ ] Adapter 1 configuré en **NAT**
- [ ] Adapter 2 configuré en **Host-Only Adapter**

---

## Étape 3 — Premier démarrage et connexion

### 3.1 Démarrer la VM

Sélectionner la VM Mobexler → cliquer sur **Start**.

### 3.2 Se connecter

| Champ | Valeur |
|---|---|
| **Username** | `mobexler` |
| **Password** | `mobexler` |

> [!NOTE]
> Selon la version de Mobexler, le username peut varier. Si l'écran de login affiche un utilisateur différent :
> - Essayer l'utilisateur affiché avec le mot de passe `mobexler`
> - Consulter la [documentation officielle de Mobexler](https://mobexler.com/) pour les identifiants exacts

### ✅ Point de contrôle

- [ ] VM démarre sans erreur
- [ ] Accès au bureau / terminal

---

## Étape 4 — Vérification réseau (tests de santé)

Ouvrir un **terminal** dans Mobexler et exécuter les commandes suivantes :

### 4.1 Vérifier les adresses IP

```bash
ip a
```

**Résultat attendu :**

| Interface | Type | Plage IP |
|---|---|---|
| `eth0` (ou `enp0s3`) | NAT | `10.0.2.x` |
| `eth1` (ou `enp0s8`) | Host-Only | `192.168.56.x` |

### 4.2 Vérifier la route par défaut

```bash
ip route
```

La route par défaut (`default via ...`) doit pointer vers l'interface NAT.

### 4.3 Tester la connectivité Internet

```bash
# Test de connectivité IP
ping -c 2 8.8.8.8

# Test de résolution DNS
ping -c 2 google.com
```

**Résultat attendu :** réponses reçues (0% packet loss) pour les deux commandes.

### 4.4 Exemple de sortie complète

```
mobexler@mobexler:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> ...
    inet 127.0.0.1/8 ...
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 10.0.2.15/24 brd 10.0.2.255 ...
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 192.168.56.101/24 brd 192.168.56.255 ...

mobexler@mobexler:~$ ping -c 2 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=12.3 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=117 time=11.8 ms
--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss

mobexler@mobexler:~$ ping -c 2 google.com
PING google.com (142.250.x.x) 56(84) bytes of data.
64 bytes from ...: icmp_seq=1 ttl=117 time=13.1 ms
64 bytes from ...: icmp_seq=2 ttl=117 time=12.7 ms
--- google.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss
```

### ✅ Point de contrôle

- [ ] 2 interfaces réseau visibles (NAT + Host-Only)
- [ ] Route par défaut configurée
- [ ] Ping `8.8.8.8` → OK
- [ ] Ping `google.com` → OK (DNS fonctionnel)

---

## Étape 5 — Création du snapshot CLEAN

> [!IMPORTANT]
> Le snapshot doit être créé **uniquement après** la validation réussie de toutes les étapes précédentes. Il constituera le **point de restauration de référence** pour tous les futurs labs.

### 5.1 Créer le snapshot

#### VirtualBox

1. `VM` → `Snapshots` (icône appareil photo)
2. Cliquer sur **Take**
3. Remplir :

| Champ | Valeur |
|---|---|
| **Nom** | `CLEAN_BASELINE_TP1` |
| **Description** | `Import OK, NAT+HostOnly OK, boot OK, prêt ADB` |

#### VMware

1. `VM` → `Snapshot` → `Take Snapshot`
2. Même nom et description

### 5.2 Tester la restauration (optionnel mais recommandé)

1. Créer un fichier temporaire : `touch /tmp/test_snapshot`
2. Restaurer le snapshot `CLEAN_BASELINE_TP1`
3. Vérifier que `/tmp/test_snapshot` n'existe plus

> [!TIP]
> **Bonne pratique :** Avant chaque nouveau TP, restaurer ce snapshot pour repartir d'un environnement propre. Cela garantit la reproductibilité et évite les conflits entre les configurations de différents labs.

### ✅ Point de contrôle

- [ ] Snapshot `CLEAN_BASELINE_TP1` visible dans le gestionnaire
- [ ] Restauration testée avec succès (optionnel)

---

## Étape 6 — Préparation de la cible Android

> Choisir **une** des deux options ci-dessous selon votre matériel.

### Option A — Smartphone test via USB (recommandé)

#### A.1 Activer le mode développeur + débogage USB

1. **Paramètres** → **À propos du téléphone**
2. Taper **7 fois** sur « Numéro de build » → message : _« Vous êtes maintenant développeur »_
3. Retourner dans **Paramètres** → **Options pour les développeurs**
4. Activer **Débogage USB** → `ON`

#### A.2 Connecter l'USB à la VM

| Hyperviseur | Navigation |
|---|---|
| **VirtualBox** | `Devices` → `USB` → sélectionner l'appareil Android |
| **VMware** | Connecter le périphérique USB à la VM via la barre d'outils |

#### A.3 Vérifier ADB dans Mobexler

```bash
# Vérifier la version d'ADB
adb version

# Lister les appareils connectés
adb devices
```

**Résultat attendu :**
```
List of devices attached
XXXXXXXXX    device
```

> [!WARNING]
> **Dépannage courant :**
> - **`unauthorized`** → Accepter la popup d'autorisation RSA sur le téléphone
> - **Appareil non listé** → Vérifier le passthrough USB dans les paramètres de la VM, puis :
>   ```bash
>   adb kill-server
>   adb start-server
>   adb devices
>   ```

---

### Option B — Émulateur (Genymotion conseillé)

> [!NOTE]
> Les émulateurs Android Studio sont souvent trop lourds en VM imbriquée. **Genymotion** offre de meilleures performances pour ce cas d'usage.

#### B.1 Démarrer un device Genymotion sur l'hôte

1. Installer Genymotion sur la machine **hôte**
2. Créer et démarrer un device virtuel Android
3. Noter l'**adresse IP** du device (généralement dans le réseau Host-Only `192.168.56.x`)

#### B.2 Connecter via ADB depuis Mobexler

```bash
# Connexion réseau à l'émulateur
adb connect <IP_DEVICE>:5555

# Vérifier la connexion
adb devices
```

**Résultat attendu :**
```
List of devices attached
<IP_DEVICE>:5555    device
```

### ✅ Point de contrôle

- [ ] L'appareil/émulateur apparaît dans `adb devices`
- [ ] État = `device` (pas `unauthorized` ni `offline`)

---

## 🔧 Dépannage

| Problème | Cause probable | Solution |
|---|---|---|
| VM ne démarre pas | Virtualisation désactivée | Activer VT-x/AMD-V dans le BIOS |
| Pas d'interface Host-Only | Réseau non créé | VirtualBox → Tools → Network Manager → Create |
| `ping 8.8.8.8` échoue | NAT mal configuré | Vérifier Adapter 1 = NAT dans les paramètres VM |
| DNS ne résout pas | DNS non configuré | `echo "nameserver 8.8.8.8" > /etc/resolv.conf` |
| ADB `unauthorized` | Clé RSA non acceptée | Accepter la popup sur l'appareil Android |
| ADB ne liste rien | USB passthrough inactif | Devices → USB → activer l'appareil ; relancer `adb kill-server && adb start-server` |
| Écran noir au boot | Mémoire vidéo insuffisante | Augmenter la vidéo RAM à 128 Mo dans les paramètres VM |

---

## ✅ Checklist finale

| # | Vérification | Statut |
|---|---|---|
| 1 | Fichier OVA téléchargé et vérifié | ☐ |
| 2 | VM importée avec 2 NICs (NAT + Host-Only) | ☐ |
| 3 | Login Mobexler fonctionnel | ☐ |
| 4 | Connectivité Internet (ping 8.8.8.8 + google.com) | ☐ |
| 5 | Snapshot `CLEAN_BASELINE_TP1` créé | ☐ |
| 6 | Cible Android connectée via ADB | ☐ |
| 7 | `adb devices` → appareil en statut `device` | ☐ |

---

## 📚 Références

- [Mobexler — Site officiel](https://mobexler.com/)
- [VirtualBox — Documentation](https://www.virtualbox.org/manual/)
- [Android Debug Bridge (ADB) — Documentation officielle](https://developer.android.com/studio/command-line/adb)
- [Genymotion — Site officiel](https://www.genymotion.com/)
- [OWASP Mobile Security Testing Guide (MSTG)](https://owasp.org/www-project-mobile-security-testing-guide/)

---

## 📝 Licence

Ce projet est à usage éducatif dans le cadre du module de sécurité des applications mobiles.

---

<p align="center">
  <i>Lab réalisé dans le cadre du cours de Sécurité Mobile — 2025/2026</i>
</p>
