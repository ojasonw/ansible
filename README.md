# Automação Ansible
Playbooks para provisionar hosts Linux, instalar Docker, levantar um cluster K3s com Argo CD e habilitar componentes básicos de observabilidade entre outras automações.

## Estruturas Exemplos
- `inventory/hosts.yml` – inventário em YAML.
- `playbooks/setup_linux.yml` – prepara servidores Linux (Docker, usuários, utilitários e exporters).
- `playbooks/setup-k3s.yml` – instala K3s, Argo CD, ingress-nginx e secrets do Infisical.
- `playbooks/get_argocd_pass.yml` – exibe a senha inicial do Argo CD.
- `roles/` – coleção de roles usadas pelos playbooks.

## Pré‑requisitos
- Ansible 2.15+ na máquina de controle.
- Acesso SSH e permissão de `become` nos hosts.
- Python disponível nos alvos Linux; acesso à internet para pacotes e manifests.
- Para conteúdo cifrado: senha do Vault (`--ask-vault-pass` ou `ANSIBLE_VAULT_PASSWORD_FILE`).

## Inventário (modelo)
Adapte o inventário ao seu ambiente. Exemplo genérico:

```yaml
all:
  children:
    k3s_cluster:
      hosts:
        k3s-master-1:
          ansible_host: 10.0.0.10
          ansible_user: admin
    linux_nodes:
      hosts:
        app-node-1:
          ansible_host: 10.0.0.20
          ansible_user: admin
```

## Uso rápido
1) Verificar acesso SSH:
```bash
ansible -i inventory/hosts.yml all -m ping
```

2) Preparar hosts Linux:
```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup_linux.yml \
  -l linux_nodes --ask-become-pass [--ask-vault-pass]
```
Variáveis úteis:
- `monitoring_packages`: lista adicional de pacotes de monitoramento.
- `blackbox_service`: `true` para incluir blackbox-exporter e Go.

3) Instalar K3s + Argo CD:
```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup-k3s.yml \
  -l k3s_cluster -e k3s_version=<versao_k3s> --ask-vault-pass
```
Variáveis ajustáveis:
- `k3s_version`: versão do K3s a instalar (ex.: `v1.29.4+k3s1`).
- `argocd_version`: versão do Argo CD.
- `platform_gitops_repo`: repositório Git usado para bootstrap da root-app.

4) Obter senha inicial do Argo CD:
```bash
ansible-playbook -i inventory/hosts.yml playbooks/get_argocd_pass.yml -l k3s_cluster
```

## Tags
Principais tags disponíveis: `users`, `utils`, `docker`, `node_exporter`, `vmagent`, `golang`, `blackbox_exporter`, `monitoring_deps`.  
Use `--tags` ou `--skip-tags` para execuções parciais.

## Licença
MIT
