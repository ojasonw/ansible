# Automação Ansible para Homelab
Playbooks para preparar hosts Linux, subir um cluster K3s com Argo CD e instalar componentes básicos de observabilidade.

## Estrutura
- `inventory/hosts.yml` – inventário padrão em YAML.
- `playbooks/setup_linux.yml` – instala Docker, usuários, utilitários de dev e exporters.
- `playbooks/setup-k3s.yml` – provisiona K3s + Argo CD + ingress-nginx + segredos do Infisical.
- `playbooks/get_argocd_pass.yml` – recupera a senha inicial do Argo CD.
- `roles/` – collection de roles (users, dev.utils, geerlingguy.docker, ansible-node-exporter, vmagent, golang, blackbox-exporter).

## Pré‑requisitos
- Ansible 2.15+ instalado na máquina de controle.
- Acesso SSH aos hosts (chave pré-configurada e `become` habilitado).
- Python presente nos alvos Linux; internet liberada para baixar pacotes e manifests do K3s/Ingress.
- Para secrets do Infisical: definir senha do Vault (`--ask-vault-pass` ou `ANSIBLE_VAULT_PASSWORD_FILE`).

## Inventário
Edite `inventory/hosts.yml` conforme seu ambiente. Exemplo com agrupamento para o cluster:

```yaml
all:
  children:
    k3s_cluster:
      hosts:
        CORE-K3S:
          ansible_host: 192.168.15.92
          ansible_user: homelab
    linux_nodes:
      hosts:
        ubuntu-server:
          ansible_host: 192.168.15.64
          ansible_user: ubuntu-server
```

## Como usar (comandos rápidos)
1) Testar conectividade:
```bash
ansible -i inventory/hosts.yml all -m ping
```

2) Preparar hosts Linux (Docker, usuários, exporters):
```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup_linux.yml \
  -l linux_nodes --ask-become-pass [--ask-vault-pass]
```
Variables úteis:
- `monitoring_packages`: lista de pacotes extras para monitoramento (opcional).
- `blackbox_service`: `true` para instalar blackbox-exporter e Go.

3) Instalar K3s + Argo CD no nó master:
```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup-k3s.yml \
  -l k3s_cluster -e k3s_version=v1.29.4+k3s1 --ask-vault-pass
```
Notas:
- `argocd_version` e `platform_gitops_repo` podem ser sobrescritos via `-e` se necessário.
- O playbook já aplica patch `--insecure`, cria NodePort 30080 e faz bootstrap da root-app (GitOps).

4) Obter senha inicial do Argo CD:
```bash
ansible-playbook -i inventory/hosts.yml playbooks/get_argocd_pass.yml -l k3s_cluster
```

## Tags principais
- `users`, `utils`, `docker`, `node_exporter`, `vmagent`, `golang`, `blackbox_exporter`, `monitoring_deps`.
Use `--tags` ou `--skip-tags` para ajustar a execução parcial.

## Dicas e troubleshooting
- Certifique-se de que o host K3s tenha pelo menos 2 vCPUs e 2–4 GB de RAM.
- O kubeconfig remoto padrão é `/etc/rancher/k3s/k3s.yaml`; após a instalação copie para sua máquina se precisar.
- Se usar senha de vault, configure `ANSIBLE_VAULT_PASSWORD_FILE=~/.ansible/vault.pass` para não digitar sempre.
- Reaplicar `setup-k3s.yml` é seguro (idempotente); use `--limit` para atingir apenas o nó master.

## Licença
MIT
