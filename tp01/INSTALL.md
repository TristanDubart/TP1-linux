# Installation de rhel9-lab et ubuntu24-lab

## 1. Périmètre
Machines concernées et versions exactes relevées sur les systèmes (`/etc/os-release`, `hostnamectl` et `lsblk -f`) :
* **rhel9-lab** : Red Hat Enterprise Linux 9.8 (Plow) — Image ISO `RHEL-9-8-0-BaseOS-x86_64` (Architecture `x86_64`).
* **ubuntu24-lab** : Ubuntu Server 24.04.3 LTS (Noble Numbat) — Image ISO `ubuntu-24.04.3-live-server-amd64.iso` (Architecture `amd64` / `x86_64`).

## 2. Ressources des machines virtuelles
Hyperviseur utilisé : QEMU/KVM piloté via `virt-manager` (`libvirt`) sur un poste hôte Fedora Linux (Intel Core i7, 16 Gio de RAM). Le mode réseau « accès par pont » est proscrit conformément aux consignes de sécurité.

| Paramètre | rhel9-lab | ubuntu24-lab |
|---|---|---|
| Processeurs | 2 vCPU | 2 vCPU |
| Mémoire vive | 2048 Mio | 2048 Mio |
| Disque système | 30 Gio (`qcow2`, allocation dynamique) | 25 Gio (`qcow2`, allocation dynamique) |
| Réseau 1 (NAT `default`) | `enp1s0` (`192.168.122.109/24`) | `enp1s0` (`192.168.122.18/24`) |
| Réseau 2 (`lab-interne` `10.42.0.0/24`) | `enp2s0` (`10.42.0.141/24`) | `enp2s0` (`10.42.0.119/24`) |
| Interface graphique | Aucune (mode texte / minimal) | Aucune (installation minimale) |
| Instantané initial | `avant-installation` | `avant-installation` |

## 3. Partitionnement appliqué
*Note de conformité (Annexe B.1) : Les disques virtuels attribués en Partie A mesurant 30 Gio (`rhel9-lab`) et 25 Gio (`ubuntu24-lab`), les tailles cibles du schéma théorique (42 Gio) ont été réduites proportionnellement afin de respecter la répartition par usage tout en préservant une réserve explicite d'espace non alloué dans le groupe de volumes `vg_sys` pour le TP 2.*

| Volume / Périphérique | Taille obtenue (RHEL 9) | Taille obtenue (Ubuntu 24.04) | Point de montage | Système de fichiers |
|---|---|---|---|---|
| `/dev/vda1` (Ubuntu uniquement)* | N/A | 1 Mio | — (BIOS boot GRUB) | Non formaté |
| `/dev/vda1` (RHEL) / `/dev/vda2` (Ubuntu)* | 960 Mio (1 Gio) | 974 Mio (1 Gio) | `/boot` | `xfs` (RHEL) / `ext4` (Ubuntu) |
| `/dev/mapper/vg_sys-lv_root` | 10 Gio | 7,8 Gio (8 Gio) | `/` | `xfs` (RHEL) / `ext4` (Ubuntu) |
| `/dev/mapper/vg_sys-lv_var` | 5,0 Gio | 3,9 Gio (4 Gio) | `/var` | `xfs` (RHEL) / `ext4` (Ubuntu) |
| `/dev/mapper/vg_sys-lv_log` | 2,0 Gio | 2,0 Gio | `/var/log` | `xfs` (RHEL) / `ext4` (Ubuntu) |
| `/dev/mapper/vg_sys-lv_home` | 2,0 Gio | 2,0 Gio | `/home` | `xfs` (RHEL) / `ext4` (Ubuntu) |
| `/dev/mapper/vg_sys-lv_swap` | 1024 Mio (1 Gio) | 1024 Mio (1 Gio) | `[SWAP]` | `swap` |
| `tmpfs` | N/A | 512 Mio | `/tmp` | `tmpfs` (Ubuntu uniquement) |
| **Espace libre (`vg_sys`)** | **~9 Gio non alloués** | **~7 Gio non alloués** | — | Réservé (extension TP 2) |

*\*Écart documenté : Sur Ubuntu 24.04 (table GPT en amorçage BIOS), l'installateur Subiquity crée automatiquement une partition `/dev/vda1` de 1 Mio pour GRUB (`bios_grub`), décalant `/boot` sur `/dev/vda2` et le volume physique LVM (`LVM2_member`) sur `/dev/vda3`.*

**Justification des choix :**
1. **Séparation de `/var` et `/var/log` du volume racine `/`** : `/var/log` concentre les écritures des journaux système et applicatifs. En production, si un démon en boucle ou une attaque par déni de service génère un volume massif de logs et que `/var/log` partage la partition `/`, la saturation bloque l'intégralité du serveur (impossibilité de créer des fichiers temporaires/sockets ou d'ouvrir une session d'administration SSH). L'isoler sur `lv_log` garantit la survie du système.
2. **Maintien de `/boot` en partition standard hors LVM** : Le chargeur d'amorçage (GRUB) doit pouvoir lire l'image du noyau (`vmlinuz`) et le disque RAM initial (`initramfs`) dès les premières étapes du démarrage, avant que les pilotes avancés du gestionnaire de volumes logiques (`dm-mod` / LVM) ne soient chargés en mémoire.
3. **Dimensionnement du Swap (1 à 2 Gio pour 2 Gio de RAM)** : Aligner la mémoire d'échange sur la RAM permet d'absorber des pics ponctuels d'utilisation mémoire et de décharger les pages inactives sans déclencher brutalement l'OOM Killer, tout en évitant l'écroulement des performances par pagination excessive (*thrashing*) qu'entraînerait un swap surdimensionné de 8 Gio.
4. **Conservation d'espace non alloué dans `vg_sys`** : Conserver plusieurs gibioctets libres dans le groupe de volumes permet d'effectuer des agrandissements de volumes logiques à chaud et de créer des instantanés (snapshots) LVM lors du TP 2.
5. **Volume appelé à saturer en premier et méthode d'extension sans coupure** : C'est le volume `/var/log` (`lv_log`) qui saturera en premier. Grâce à LVM, il pourra être étendu en ligne sans coupure de service via la commande `sudo lvextend -r -L +XG /dev/vg_sys/lv_log` (l'option `-r` redimensionnant à chaud le système de fichiers `xfs` ou `ext4`).

## 4. Installation
* **Profil retenu** : Installation minimale en mode texte sur les deux distributions (aucun serveur X ni environnement de bureau installé).
* **Identité des machines** :
  * `rhel9-lab` (`Machine ID: 1ea7124589c141bcaa9394125c2849d3`)
  * `ubuntu24-lab` (`Machine ID: f58f0aba514b418f9379e0f71606aa78`)
* **Locale et Clavier** : `fr_FR.UTF-8` vérifiée avec `localectl status` et `locale -a | grep -i fr_FR`.
* **Fuseau horaire et horloge** : Fuseau configuré sur `Europe/Paris` (`sudo timedatectl set-timezone Europe/Paris`) et synchronisation NTP active (`chronyd` sur RHEL 9, `systemd-timesyncd` sur Ubuntu 24.04) vérifiée via `timedatectl status`.

## 5. Mises à jour
* **Sur `rhel9-lab`** :
  * Commande exécutée : `sudo dnf -y update`
  * Trace relevée (`sudo dnf history | head -5`) :
    ```text
    Mise à jour des référentiels de gestion des abonnements.
    Ce système est enregistré auprès d'un serveur de droits, mais ne reçoit pas de mises à jour. Vous pouvez utiliser subscription-manager pour assigner des abonnements.
    ```
  * *Explication* : En environnement de laboratoire sans abonnement Red Hat CDN rattaché, aucun paquet distant n'est téléchargé (comportement attendu et documenté en section 9).
* **Sur `ubuntu24-lab`** :
  * Commandes exécutées : `sudo apt update && sudo apt -y full-upgrade`
  * Extrait horodaté relevé dans `/var/log/apt/history.log` :
    ```text
    Start-Date: 2026-10-02  10:09:17
    Commandline: apt -y full-upgrade
    Requested-By: tristan (1000)
    Upgrade: initramfs-tools-core:amd64 (0.142ubuntu25.5, 0.142ubuntu25.8), netplan-generator:amd64 (1.1.2-2~ubuntu24.04.2, 1.1.2-8ubuntu1~24.04.3), console-setup:amd64 (1.226ubuntu1, 1.226ubuntu1.1), apparmor:amd64 (4.0.1really4.0.1-0ubuntu0.24.04.4, 4.0.1really4.0.1-0ubuntu0.24.04.8), cloud-init:amd64 (25.1.4-0ubuntu0~24.04.1, 26.1-0ubuntu1~24.04.1), systemd-hwe-hwdb:amd64 (255.1.4, 255.1.7), netplan.io:amd64 (1.1.2-2~ubuntu24.04.2, 1.1.2-8ubuntu1~24.04.3), iproute2:amd64 (6.1.0-1ubuntu6.2, 6.1.0-1ubuntu6.4) [...]
    End-Date: 2026-10-02  10:11:01
    ```

## 6. Comptes et accès distant
* **Compte d'administration nominatif** : Création du compte `tristan` (UID `1000`).
* **Mécanisme de délégation** : Délégation via `sudo` par appartenance au groupe `wheel` sur RHEL 9 (`sudo usermod -aG wheel tristan`) et au groupe `sudo` sur Ubuntu 24.04 (`sudo usermod -aG sudo tristan`). La prise d'effet de l'ajout au groupe nécessite une reconnexion de la session utilisateur.
* **Verrouillage du compte `root`** : Exécution de `sudo passwd -l root` sur les deux serveurs (vérifié par `sudo passwd -S root` affichant le statut verrouillé `LK` / `L`).
* **Authentification SSH par clé** :
  * Génération d'une paire de clés `ed25519` sur le poste d'administration Fedora : `ssh-keygen -t ed25519 -a 100 -C "${USER}@lab"`
  * Déploiement vers les deux serveurs via le réseau interne : `ssh-copy-id tristan@10.42.0.141` et `ssh-copy-id tristan@10.42.0.119`.
* **Configuration `sshd` retenue** :
  * Ajout de la directive explicite `PermitRootLogin no` dans `/etc/ssh/sshd_config.d/99-root-deny.conf`, validée par `sudo sshd -t` avant rechargement.
  * Vérification effective via `sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'` :
    ```text
    permitrootlogin no
    passwordauthentication yes
    ```
  * *Note* : `PasswordAuthentication` est maintenu à `yes` jusqu'au TP 6 conformément aux consignes de la Partie D.4.

## 7. Pare-feu
* **RHEL 9 (`rhel9-lab`)** :
  * Outil : `firewalld`.
  * Règles appliquées : Autorisation exclusive du service `ssh` et suppression des services ouverts par défaut (`cockpit` et `dhcpv6-client`) sur la zone `public` active (`interfaces: enp1s0 enp2s0`).
  * Vérification : `sudo firewall-cmd --list-all` confirme `services: ssh` uniquement.
* **Ubuntu 24.04 (`ubuntu24-lab`)** :
  * Outil : `ufw`.
  * Règles appliquées : Politique par défaut `deny (incoming)`, `allow (outgoing)`, et autorisation explicite de `22/tcp (OpenSSH)` en IPv4 et IPv6.
  * Vérification : `sudo ufw status verbose` confirme le statut `active` et la règle `22/tcp (OpenSSH) ALLOW IN Anywhere`.

## 8. Écarts entre les deux distributions
| Opération | RHEL 9 (`rhel9-lab`) | Ubuntu 24.04 LTS (`ubuntu24-lab`) | Raison de la différence |
|---|---|---|---|
| Installateur système | Anaconda | Subiquity | Installateurs propres à Red Hat et Canonical. |
| Partitionnement disque | `vda1` (`/boot`) + `vda2` (`LVM`) | `vda1` (`bios_grub`) + `vda2` (`/boot`) + `vda3` (`LVM`) | Subiquity crée une partition BIOS Boot de 1 Mio sur table GPT. |
| Système de fichiers par défaut | `xfs` | `ext4` | Choix techniques par défaut historiques de chaque éditeur. |
| Point de montage `/tmp` | Géré par le système | `tmpfs` (512 Mio) configuré dans `/etc/fstab` | Exigence du cahier des charges pour Ubuntu. |
| Gestionnaire de paquets | `dnf` (format RPM) | `apt` (format DEB) | Écosystèmes de paquetage différents (Red Hat vs Debian). |
| Preuve des mises à jour | `sudo dnf history` | `/var/log/apt/history.log` | Base transactionnelle DNF vs fichier journal texte APT. |
| Création d'utilisateur | `useradd -m` puis `passwd` | `adduser` (interactif) ou `useradd` | Sur RHEL, `useradd` crée le compte sans mot de passe initial. |
| Groupe de délégation `sudo` | `wheel` | `sudo` | Convention d'administration propre à chaque famille Linux. |
| Nom du service SSH | `sshd.service` | `ssh.service` | Nommage de l'unité systemd du démon OpenSSH. |
| Pare-feu local | `firewalld` (`firewall-cmd`) | `ufw` | Surcouches de filtrage Netfilter/nftables par défaut distinctes. |

## 9. Incidents rencontrés et corrections
1. **Démarrage de l'installation RHEL 9 avant le partitionnement manuel LVM** :
   * *Symptôme* : L'installation a été lancée avec le schéma de partitionnement automatique par défaut.
   * *Diagnostic et correction* : Arrêt immédiat de la VM et restauration de l'instantané `avant-installation` pris à l'étape A.4, permettant de reprendre proprement le partitionnement personnalisé dans Anaconda.
2. **Option de création du groupe de volumes `vg_sys` grisée sous Ubuntu** :
   * *Symptôme* : Impossible de sélectionner `Create volume group (LVM)` dans Subiquity après avoir créé une première partition montée par erreur sur `/home`.
   * *Diagnostic et correction* : Subiquity requiert une partition explicitement laissée non formatée (`Leave unformatted`) pour constituer un Physical Volume (PV) LVM. La première partition de 1 Gio a été corrigée en `/boot` (`ext4`), puis le reste du disque a été créé en `Leave unformatted`, débloquant la création de `vg_sys`.
3. **Échec de `ssh-copy-id` (`Permission denied, please try again`) vers `rhel9-lab`** :
   * *Symptôme* : Refus d'authentification lors de la copie de la clé publique vers `tristan@10.42.0.141`.
   * *Diagnostic et correction* : La commande `useradd` sous RHEL ne définit aucun mot de passe par défaut, ce qui verrouille l'authentification initiale par mot de passe. Un mot de passe a été défini depuis la console `virt-manager` (`sudo passwd tristan`), permettant le transfert de la clé.
4. **Unité `dnf-makecache.service` en échec et absence de mises à jour sur RHEL 9** :
   * *Symptôme* : `systemctl --failed` affichait `dnf-makecache.service` en échec et `dnf history` indiquait l'absence d'abonnement actif.
   * *Diagnostic et correction* : Machine de laboratoire isolée sans souscription Red Hat CDN enregistrée. L'état d'échec systemd a été nettoyé via `sudo systemctl reset-failed`.
5. **Présence de `cockpit`, `dhcpv6-client` et `PermitRootLogin without-password` lors du contrôle initial** :
   * *Symptôme* : La première passe de vérification montrait que `root` était encore autorisé par clé SSH et que `firewalld` exposait `cockpit`.
   * *Correction* : Création de `/etc/ssh/sshd_config.d/99-root-deny.conf` avec `PermitRootLogin no`, rechargement du démon SSH, et retrait permanent des services superflus dans `firewalld`.

## 10. Reconstruction rapide
Séquence minimale de commandes post-installation permettant de reproduire à l'identique la configuration et le durcissement des deux serveurs :

### A. Sur `rhel9-lab`
```bash
# 1. Identité, fuseau horaire et nettoyage systemd
sudo hostnamectl set-hostname rhel9-lab
sudo timedatectl set-timezone Europe/Paris
sudo timedatectl set-ntp true
sudo dnf -y update || sudo systemctl reset-failed

# 2. Compte nominatif, délégation et verrouillage root
sudo useradd -m -c "Tristan" tristan
sudo passwd tristan
sudo usermod -aG wheel tristan
sudo passwd -l root

# 3. Durcissement SSH
echo "PermitRootLogin no" | sudo tee /etc/ssh/sshd_config.d/99-root-deny.conf
sudo sshd -t && sudo systemctl reload sshd

# 4. Pare-feu (SSH uniquement)
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --remove-service=cockpit
sudo firewall-cmd --permanent --remove-service=dhcpv6-client
sudo firewall-cmd --reload
```

### B. Sur `ubuntu24-lab`
```bash
# 1. Identité, fuseau horaire, /tmp en tmpfs et mises à jour
sudo hostnamectl set-hostname ubuntu24-lab
sudo timedatectl set-timezone Europe/Paris
sudo timedatectl set-ntp true
grep -q " /tmp " /etc/fstab || echo "tmpfs /tmp tmpfs defaults,size=512M,mode=1777 0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload && sudo mount /tmp
sudo apt update && sudo apt -y full-upgrade
sudo apt -y install openssh-server ufw

# 2. Compte nominatif, délégation et verrouillage root
 id tristan &>/dev/null || sudo adduser tristan
sudo usermod -aG sudo tristan
sudo passwd -l root

# 3. Durcissement SSH
echo "PermitRootLogin no" | sudo tee /etc/ssh/sshd_config.d/99-root-deny.conf
sudo sshd -t && sudo systemctl reload ssh

# 4. Pare-feu (SSH uniquement)
sudo ufw allow OpenSSH
sudo ufw --force enable
```

### C. Depuis le poste d'administration (Fedora)
```bash
ssh-keygen -t ed25519 -a 100 -C "${USER}@lab"
ssh-copy-id tristan@10.42.0.141
ssh-copy-id tristan@10.42.0.119
ssh tristan@10.42.0.141 'id ; sudo -n true && echo delegation-ok'
ssh tristan@10.42.0.119 'id ; sudo -n true && echo delegation-ok'
```
