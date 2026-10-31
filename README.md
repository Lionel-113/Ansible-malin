# Ansible malin

Base Ansible pour administrer des serveurs Debian et des pare-feux OPNsense.
La premiere brique gere les alias OPNsense de type **Host(s)** : les adresses
declarees dans `content` sont ecrites dans le champ **Content** de l'alias.

## Demarrage

1. Copier `inventory/group_vars/opnsense.yml.example` vers
   `inventory/group_vars/opnsense.yml` pour les variables communes non
   sensibles.
2. L'inventaire contient les cinq pare-feux OPNsense et configure leur API HTTPS
   sur le port `1000`. Chaque firewall utilise un coffre distinct dans
   `inventory/host_vars/<nom>/vault.yml` ; voir la section *Coffres Ansible
   Vault*.
3. Declarer les alias communs dans `opnsense_aliases_global` et, si besoin,
   les alias propres a un firewall dans `inventory/host_vars/<nom>/aliases.yml`.
   Puis verifier et appliquer :


Depuis WSL 
Modifier les routes par default qui sont en conflit avec les IP WSL : 

```bash


sudo ip route replace 172.20.20.0/24 via 172.30.0.1 dev eth0
sudo ip route replace 172.20.100.0/24 via 172.30.0.1 dev eth0

ansible-playbook playbooks/opnsense_aliases.yml --check
ansible-playbook playbooks/opnsense_aliases.yml --ask-vault-password
```

Le compte API doit disposer au minimum du droit `Firewall: Alias: Edit`.
Les alias sont crees en type `host`. Le role ne convertit jamais un alias
existant d'un autre type, afin de ne pas modifier silencieusement une
configuration de filtrage.

Le role est idempotent : il ne reconfigure OPNsense qu'apres une creation ou
une modification. Une execution avec la meme liste d'adresses ne produit aucun
changement.

## Exemple

```yaml
# inventory/group_vars/opnsense.yml : applique a tous les firewalls
opnsense_aliases_global:
  - name: SRV_WEB
    description: Serveurs web publies
    content:
      - 10.20.10.11
      - 10.20.10.12
```

Un alias propre a `opnsense-web2lfm` reste dans
`inventory/host_vars/opnsense-web2lfm/aliases.yml` sous la cle
`opnsense_aliases`. Les deux listes sont fusionnees automatiquement. Un meme
nom d'alias ne peut pas etre declare dans les deux listes.

Les adresses IPv4 et IPv6 sont acceptees par OPNsense. Pour des reseaux CIDR,
des plages ou des FQDN, il faudra ajouter un role/type d'alias distinct plutot
que de surcharger ce role `host`.

## Authentification API OPNsense

Sur chaque pare-feu, creer un utilisateur local dedie, par exemple
`ansible-malin`, dans **System > Access > Users**. Dans la section ou l'onglet
**API keys** de cet utilisateur, cliquer sur `+` pour generer une cle. OPNsense
telecharge alors un fichier contenant une paire `key` / `secret` ; le secret
n'est disponible qu'a ce moment.

Attribuer a cet utilisateur le privilege **Firewall: Alias: Edit** (directement
ou via un groupe). La cle API est envoyee comme identifiant HTTP Basic et le
secret comme mot de passe. Elle ne correspond pas au mot de passe du compte
WebGUI.

## Coffres Ansible Vault par firewall

Le projet prevoit une paire de secrets independante pour chaque firewall. Les
fichiers `vault.yml.example` servent uniquement de modele et ne contiennent pas
de secret. Pour chaque equipement, creer directement un fichier chiffre :

```bash
ansible-vault create inventory/host_vars/opnsense-web1/vault.yml
```

Saisir un mot de passe Vault robuste, puis coller dans l'editeur :

```yaml
opnsense_api_key: "key-generee-sur-opnsense-web1"
opnsense_api_secret: "secret-genere-sur-opnsense-web1"
```

Repeter l'operation pour `opnsense-web2`, `opnsense-web2lfm`,
`opnsense-web3` et `opnsense-web3pp`, en utilisant leur paire respective. Le
mot de passe Vault peut etre commun aux cinq coffres ; il ne doit pas etre
stocke dans ce depot. Pour executer le playbook :

```bash
ansible-playbook playbooks/opnsense_aliases.yml --ask-vault-pass
```

Pour confirmer qu'un coffre est bien chiffre sans reveler son contenu :

```bash
head -1 inventory/host_vars/opnsense-web1/vault.yml
# $ANSIBLE_VAULT;...
```
