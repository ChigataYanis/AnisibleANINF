# Messagerie gouvernementale ANINF — Zimbra 10.1 sur Ubuntu 24.04

Déploiement Ansible par VEONE de la plateforme décrite dans la fiche
*FICHE-synthese-veone-messagerie-zimbra10-v1* (ANINF, 8 octobre 2026).
Les renvois « §n » et « Qn » désignent les sections et questions de cette fiche.

## Périmètre

| Inventaire | Machines | Playbook |
|---|---|---|
| `inventories/commune` | 3 LDAP multi-maîtres, 1 service de licence (LDS), 3 MTA, 3 proxys, 6 serveurs de boîtes | `playbooks/site.yml` |
| `inventories/zmpres` | msg-zmpres, instance tout-en-un de la Présidence (§5) | `playbooks/zmpres.yml` |

Ce projet couvre **uniquement l'installation et la configuration de Zimbra**.
La migration des courriers (msg-imapsync, §7) est un projet séparé
(`../ANINFMigration`). Seule l'exemption de l'adresse de msg-imapsync dans
Zimbra (`imapsync_ips`, §7.5) reste ici, car elle se règle sur la plateforme.

État au 15/10/2026 : les accès aux environnements ANINF ne sont pas encore
remis. Rien n'a été exécuté sur une machine réelle ; seules la syntaxe et le
rendu des modèles ont été vérifiés.

## Ce que l'ANINF livre et ce que le projet fait

L'ANINF livre les VM prêtes (§3.4). Le projet **contrôle** ces prérequis sans
les modifier et s'arrête s'il en manque un :

- Ubuntu 24.04 ;
- AppArmor désactivé, aucun pare-feu local, `nc` présent ;
- horloge synchronisée ;
- `/opt/zimbra` monté en XFS, sur LVM ;
- l'adresse déclarée est bien portée par la VM.

Le projet ne touche ni aux agents de l'ANINF, ni aux disques (aucun
partitionnement), ni au DNS du système.

Il **modifie** :

- le nom d'hôte complet ;
- `/etc/hosts`, si Q2 = `hosts` ;
- le fuseau `Africa/Libreville` ;
- les dépendances de Zimbra, et retire postfix et exim4 s'ils sont présents.

Il installe et configure ensuite Zimbra.

## Ordre d'exécution (§6.2)

`site.yml` enchaîne les étapes suivantes. Chaque étape a son étiquette (*tag*) :

| Tag | Étape |
|---|---|
| `common` | Contrôles préalables, système, dépôt de l'installateur |
| `ldap` | LDAP 1 (maître, active la réplication multi-maîtres), puis LDAP 2 et 3 un par un |
| `mmr` | Contrôle des accords de réplication entre les trois annuaires |
| `lds` | Service de licence, activé **avant** le premier serveur de boîtes |
| `mta`, `proxy`, `mailbox` | Installation, une machine à la fois |
| `finalize` | Clés SSH (`zmupdateauthkeys`), ordre des annuaires différent par serveur (§6.3), zmtrainsa désactivé, syslog |
| `postconfig` | Réglages §6.4 (détail ci-dessous), puis compression des volumes (§6.6) |
| `certs` | Certificat principal et certificats par domaine (§4.4) |
| `restart` | Redémarrage ordonné, une machine à la fois |

Les réglages de l'étape `postconfig` sont les suivants :

- relais vers les FortiMail et réseaux de confiance des MTA ;
- livraison LMTP en résolution native ;
- envoi du webmail par les FortiMail (`zimbraSmtpHostname`) ;
- adresses de confiance des proxys et adresses exemptées du filtrage ;
- memcached déclaré aux serveurs de boîtes ;
- antivirus, antispam, DKIM et listes noires DNS retirés des MTA ;
- POP3 désactivé.

L'installation est **rejouable**. Chaque étape laisse un témoin dans
`/var/lib/zimbra-ansible/`, et une machine déjà installée n'est jamais
réinstallée.

### Comment l'installateur est piloté

Les invites de `install.sh -s` reçoivent leurs réponses par un script
`expect`. Chaque invite « Install zimbra-xxx » reçoit « y » si le paquet est
dans la liste de la machine, « n » sinon. Cela ne dépend donc pas de l'ordre
des questions, contrairement aux fichiers `y/n` du projet CNPS.

Une invite non prévue arrête l'installation au bout du délai défini, sans
répondre au hasard. Le journal se trouve dans `/var/log/zimbra-ansible-install.log`.

La configuration passe ensuite par `zmsetup.pl -c`. Le fichier de réponses
contient les mots de passe : il est supprimé dès la fin de l'étape.

## Mise en route

Ansible est lancé **depuis msg-imapsync** (100.64.35.4), où l'on se connecte
par Teleport. De là, les serveurs Zimbra sont joints directement en SSH par
leur adresse dans le VLAN 2035, avec le compte `aninf` et `sudo`
(`ansible_host` = `zimbra_ip`, voir `group_vars/all/main.yml`).

1. **Préparer msg-imapsync** (une fois) :
   ```bash
   tsh ssh aninf@msg-imapsync
   ```
   ```bash
   sudo apt install -y ansible-core tmux
   ```
   Copier le projet dans le répertoire de `aninf` (par exemple
   `~/ANINFZimbra10`) et lancer toutes les commandes depuis sa racine.
2. **Accès SSH de msg-imapsync vers les serveurs.** Créer une clé pour
   `aninf` (`ssh-keygen -t ed25519`) et faire autoriser sa clé publique par
   l'ANINF sur les 17 serveurs, puis tester :
   ```bash
   ansible all -m ping
   ```
   Si `sudo` demande un mot de passe, ajouter `--ask-become-pass` aux
   commandes `ansible-playbook` ci-dessous. Lancer les installations dans
   `tmux` : une session Teleport coupée n'interrompt alors pas Ansible.
3. **Variables.** Dans `inventories/*/group_vars/all/main.yml`, renseigner :
   - `zimbra_tarball` (Q1) ;
   - `zimbra_name_resolution` (Q2) ;
   - `zimbra_default_domain` ;
   - les adresses des FortiMail.

   Les adresses des serveurs, `zimbra_hosts_domain` (`aninf.ga`) et
   `imapsync_ips` sont déjà reportés depuis le tableau de l'ANINF du 15/10/2026.
4. **Coffre.** Créer et chiffrer le coffre :
   ```bash
   cp inventories/commune/group_vars/all/vault.yml.example inventories/commune/group_vars/all/vault.yml
   ```
   ```bash
   ansible-vault encrypt inventories/commune/group_vars/all/vault.yml
   ```
5. **Archive Zimbra.** Déposer l'archive Network Edition Ubuntu 24.04 dans
   `files/`, depuis le portail client Zimbra.
6. **Contrôles seuls** :
   ```bash
   ansible-playbook playbooks/site.yml --ask-vault-pass --tags common
   ```
7. **Déploiement** :
   ```bash
   ansible-playbook playbooks/site.yml --ask-vault-pass
   ```
8. **Instance de la Présidence** :
   ```bash
   ansible-playbook -i inventories/zmpres/hosts.yml playbooks/zmpres.yml --ask-vault-pass
   ```

L'installation utilise le dépôt de paquets Zimbra : demander à l'ANINF
l'ouverture Internet temporaire (§4.5) avant les étapes d'installation.

msg-imapsync est **temporaire** : elle est détruite à la fin de la migration
(§7.2). Avant, recopier ailleurs le projet, son coffre et `ansible.log`, et
prévoir un autre poste de lancement pour les exploitations ultérieures.

## À valider sur une maquette avant la production

Ces points ne sont pas décrits de façon assez précise par la documentation
pour être figés sans essai. Ils sont isolés dans des variables :

| Point | Où |
|---|---|
| Nom exact du paquet du service de licence (`zimbra-license-daemon` supposé) | `group_vars/zimbra_lds.yml`, `zmpres/.../main.yml` |
| Clé(s) `zmsetup` qui désignent le LDS au serveur de boîtes : à relever dans `/opt/zimbra/config.*` après une installation manuelle | `zimbra_setup_lds` |
| Commande d'activation de la licence (Q12) ; tant qu'elle est vide, le playbook fait une pause pour une activation manuelle | `zimbra_license_activation_command` |
| Accords de réplication entre LDAP 2 et LDAP 3 : le contrôle `mmr` arrête le déploiement s'il en manque un | `roles/zimbra_ldap_mmr` |
| Liste des dépendances Ubuntu 24.04 | `roles/common/defaults/main.yml` |
| Attributs non cités par la documentation (filtre d'échecs de connexion, etc.), à confirmer par VEONE (§6.4) | `postconfig.zmprov.j2` |
| Compression et `zmdedupe` sous la licence (Q14) | `roles/zimbra_postconfig/tasks/compression.yml` |

## Questions encore ouvertes qui changent des variables

- **Q1** : version exacte, qui fixe `zimbra_tarball`.
- **Q2** : `zimbra_name_resolution` (`hosts` ou `dns`) et `zimbra_hosts_domain`.
- **Q3 et Q15** : noms publics et certificats, qui alimentent `zimbra_domain_certs`.
- **Q4** : POP3 est désactivé ; IMAPS et SMTPS sont publiés ou non par le FortiGate.
- **Q10** : VLAN de msg-zmpres.
- **Q12** : mode d'activation de la licence et seconde licence pour msg-zmpres.

## Différences avec le projet CNPS

| | CNPS | ANINF |
|---|---|---|
| OS et version | RHEL 8 (`dnf`), Zimbra 9.0 | Ubuntu 24.04 (`apt`), Zimbra 10.1 |
| Organisation du code | Un fichier de tâches par machine | Des rôles génériques par fonction, avec `group_vars` |
| Disques | Formatage de `/dev/sdb` | Aucune opération sur les disques (`/opt/zimbra` livré par l'ANINF) |
| Accès | Clé root | Compte `aninf` et `sudo`, lancé depuis msg-imapsync (atteinte par Teleport) |
| Secrets | Mots de passe en clair dans `defaults/main.yml` | `ansible-vault` |
| Inclusions | `include:` (retiré d'ansible-core 2.16) | `import_tasks` / `include_role` |
| zmtrainsa | `zmtrainsa_cleanup_host=TRUE` | Désactivé, comme l'exige la fiche (§6.4) |
# AnisibleANINF
# AnisibleANINF
