Installer un snapshot

- [ ] Installer Ansible et pipx
  * pacman -S ansible python-pipx
- [ ] Installer le plugin community.general Ansible pour avoir accès à pipx
  * ansible-galaxy collection install community.general
- [ ] Exécuter le playbook manjaro
  * ansible-playbook -i inventory/hosts.ini manjaro.yml
