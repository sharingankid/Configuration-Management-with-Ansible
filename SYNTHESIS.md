# Synthèse Ansible — Configuration Management

## Concepts clés

**Control node vs managed node**
Le control node (ici : conteneur Docker `ansible-control:2.17.7`) exécute Ansible et se connecte en SSH aux hosts managés. Aucun agent n'est requis sur les hosts.

**Inventaire** (`inventory.ini`)
```ini
[web]
sandbox ansible_host=<HOST> ansible_port=<PORT> ansible_user=<USER>

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```
- Groupes `[nom]`, variables par host inline, variables globales dans `[all:vars]`
- `ansible-inventory --graph` / `--host <nom>` pour inspecter la résolution
- **Ne jamais** mettre le mot de passe dedans → `--ask-pass` à l'exécution

**Ad-hoc commands** (inspection ponctuelle, pas d'état à maintenir)
```bash
ansible <group> -m <module> -a "<args>" --ask-pass
ansible web -m ansible.builtin.ping --ask-pass
ansible web -m ansible.builtin.setup --ask-pass   # facts
```

**command vs shell**
- `command` : exécute le binaire directement (`execve`), pas d'interprétation shell → sûr par défaut, insensible à l'injection
- `shell` : passe par `/bin/sh -c`, interprète `|`, `>`, `&&`, variables → nécessaire seulement si on a besoin de ces fonctionnalités

## Playbooks

**Structure de base**
```yaml
- name: ...
  hosts: web
  become: true          # sudo
  vars:
    ma_liste: [a, b, c]
  tasks:
    - name: ...
      ansible.builtin.<module>:
        ...
```

**Modules utilisés**
- `apt` : gestion paquets (`state: present`, `cache_valid_time`, liste dans une variable plutôt que dupliquer les tâches)
- `template` : rend un fichier Jinja2 (`.j2`) avec les variables/facts (`src`, `dest`, `owner`, `group`, `mode`)
- `service` : démarre/active un service (`state: started`, `enabled: true`, `use: sysvinit` si pas de systemd PID 1)
- `uri` : requête HTTP pour valider un résultat (`return_content: true`)

**Idempotence**
Un playbook bien écrit ne "change" rien s'il est rejoué sans modification de l'état cible → `changed=0` au 2e run. C'est le critère de validation principal.

**Variables & group_vars**
`group_vars/<groupe>.yml` charge automatiquement des variables pour tous les hosts du groupe — évite de les répéter dans le playbook.

**Templates Jinja2**
`{{ variable }}` et `{{ ansible_facts... }}` sont interpolés au moment du rendu. Vérifier qu'aucune expression `{{ }}` ne reste littéralement dans le fichier final = preuve que le rendu a fonctionné.

**Handlers**
```yaml
tasks:
  - name: ...
    ansible.builtin.template:
      ...
    notify: Reload nginx

handlers:
  - name: Reload nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded
```
- Ne s'exécutent **qu'une fois**, **à la fin du play**, et **seulement si** une tâche notifiante a réellement `changed`
- Ne notifier que depuis les tâches dont le changement exige réellement l'action du handler (ex: config nginx → reload ; contenu HTML → pas besoin)

**Playbooks composés**
```yaml
- ansible.builtin.import_playbook: install.yml
- ansible.builtin.import_playbook: webserver.yml
```
`site.yml` orchestre plusieurs playbooks en un seul point d'entrée.

## Vérification / preview
```bash
ansible-playbook site.yml --check --diff --ask-pass   # dry-run (limité si dépendances non installées, ex python3-apt)
ansible-playbook site.yml --syntax-check               # validation syntaxique seule
ansible all -m ping --ask-pass -vvv                     # debug connexion, montre la commande SSH réelle
```

## Qualité de code
```bash
yamllint .                              # style YAML
ansible-lint --project-dir . site.yml   # bonnes pratiques Ansible (profil "production" = le plus strict)
```
Corriger la cause du problème plutôt que désactiver une règle.

## Pièges rencontrés pendant l'exercice
- **`--check` + module `apt`** échoue si `python3-apt` n'est pas déjà installé (impossible de simuler son auto-installation)
- Toujours vérifier **dans quel shell** on se trouve (WSL / conteneur Docker / sandbox SSH) — chaque commande n'existe que dans son propre environnement
- Un correcteur automatique peut attendre un chemin de fichier précis (ex: `/etc/nginx/sites-available/default` plutôt qu'un fichier custom) — bien lire l'énoncé structurel attendu
