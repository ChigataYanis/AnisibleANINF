# Désinstallation de Zimbra — plateforme ANINF

Projet séparé du projet d'installation `ANINFZimbra10`. Il sert à **casser le
cluster** : retirer Zimbra des serveurs, par exemple avant de réinstaller.

> **Destructif et irréversible.** Le playbook efface les données de Zimbra
> (annuaire, boîtes, messages) sur les serveurs visés. Il ne fait **aucune
> sauvegarde**. Une fois les usagers en service, sauvegarder d'abord.

## Ce que fait le playbook

Sur chaque serveur visé, dans l'ordre inverse de l'installation (boîtes,
proxys, MTA, licence, puis annuaires) :

1. Arrête Zimbra (`zmcontrol stop`), puis les processus restants du compte
   `zimbra`, et supprime ses tâches planifiées.
2. Retire tous les paquets `zimbra-*` (`apt purge`, avec retrait forcé si les
   scripts des paquets échouent), ainsi que le dépôt de paquets Zimbra.
3. Vide le **contenu** de `/opt/zimbra`. Le volume livré par l'ANINF n'est
   jamais démonté ni formaté ; le playbook vérifie à la fin qu'il est
   toujours monté.
4. Supprime les comptes et groupes `zimbra`, `postfix` et `postdrop`, les
   fichiers système de Zimbra et les témoins de l'installation Ansible. Une
   réinstallation repart donc de zéro.
5. Contrôle qu'il ne reste aucun paquet Zimbra et que les ports de Zimbra
   (25, 80, 143, 389, 443, 993, 7071, 8081) sont libres.

Il ne touche **pas** :

- aux agents de l'ANINF ;
- au nom d'hôte ;
- à l'installateur décompressé (`/var/tmp/zcs`, gardé pour réinstaller) ;
- au bloc `/etc/hosts` ;
- à la configuration rsyslog : les fichiers qui mentionnent Zimbra sont
  seulement signalés dans le bilan, pour contrôle à la main.

Ces choix se règlent dans `roles/zimbra_uninstall/defaults/main.yml`.

## Garde-fous

- **Cible obligatoire** avec `-e cible=…`. Sans cible, rien ne se passe.
  `--limit` n'est pas accepté, car il sauterait l'étape de confirmation.
- **Confirmation écrite** : le playbook affiche la liste des serveurs visés,
  puis demande de taper `DETRUIRE <nombre de serveurs>`, par exemple
  `DETRUIRE 3`. Toute autre réponse annule tout, sans rien modifier.

## Commandes

À lancer depuis msg-imapsync, à la racine de ce projet, comme l'installation.
Le compte `aninf` et la clé `~/.ssh/key_ansible_aninf.pem` sont les mêmes.

| Cible | Commande |
|---|---|
| Toute la plateforme commune | `ansible-playbook playbooks/desinstallation.yml -e cible=all` |
| Un groupe (ici, les MTA) | `ansible-playbook playbooks/desinstallation.yml -e cible=zimbra_mta` |
| Un seul serveur | `ansible-playbook playbooks/desinstallation.yml -e cible=msg-mta-2` |
| Instance de la Présidence | `ansible-playbook -i inventories/zmpres/hosts.yml playbooks/desinstallation.yml -e cible=all` |

Les groupes disponibles sont `zimbra_ldap`, `zimbra_lds`, `zimbra_mta`,
`zimbra_proxy` et `zimbra_mailbox`. Si `sudo` demande un mot de passe, ajouter
`--ask-become-pass`.

Retirer un seul rôle redondé (un MTA, un proxy) ne libère pas sa place dans
l'annuaire : la plateforme restante continue de le déclarer. Pour réinstaller
ce seul serveur, le relancer ensuite avec le projet d'installation. Pour le
retirer définitivement, le supprimer de l'annuaire (`zmprov deleteServer`).

## Inventaires

`inventories/*/hosts.yml` sont des **copies** de ceux d'`ANINFZimbra10`. Toute
modification des serveurs (adresses, ajouts) doit être reportée dans les deux
projets.

## État

Écrit le 15/10/2026, sans accès aux serveurs. Seuls la syntaxe et le
garde-fou (cible absente, mauvaise confirmation) ont été testés. La première
utilisation réelle est à faire sur un seul serveur
(`-e cible=msg-mta-3`, par exemple), avec le bilan relu avant d'aller plus loin.
