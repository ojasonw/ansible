# ansible

Playbooks para provisionar hosts Linux e bootstrapar clusters k3s + ArgoCD isolados por VM.

## Estrutura

```
ansible/
├── inventory/hosts.yml       # hosts do homelab
├── playbooks/
│   ├── setup_linux.yml       # Docker, users, node-exporter, vmagent
│   ├── setup-k3s.yml         # k3s standalone + ArgoCD + bootstrap GitOps
│   └── get_argocd_pass.yml   # exibe senha inicial do ArgoCD
└── roles/                    # roles usadas pelos playbooks
```

## Pré-requisitos

- Ansible 2.15+
- Acesso SSH com `become` nos hosts
- Para conteúdo cifrado: `--ask-vault-pass` ou `.vault_system_pass`

## Inventário

Edite `inventory/hosts.yml` adicionando o IP da VM após criação via Terraform:

```yaml
all:
  vars:
    k3s_master: CORE-K3S
  hosts:
    CORE-K3S:
      ansible_host: 192.168.15.92
      ansible_user: homelab
    homelab-monitoring:
      ansible_host: 192.168.15.XXX  # atualizar após terraform apply
      ansible_user: homelab
```

## Uso

### 1. Verificar conectividade SSH
```bash
ansible -i inventory/hosts.yml all -m ping
```

### 2. Provisionar host Linux (Docker, users, exporters)
```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup_linux.yml \
  -l <host> --ask-become-pass
```

Tags disponíveis: `users`, `utils`, `docker`, `node_exporter`, `vmagent`, `golang`, `blackbox_exporter`

### 3. Bootstrapar k3s + ArgoCD

Cada VM recebe seu próprio cluster k3s isolado. O ArgoCD é apontado para `nodes/<vm>/apps/` no `homelab-gitops`.

**CORE-K3S** (cluster principal, inclui ingress-nginx e Infisical):
```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup-k3s.yml --ask-vault-pass
```

**Nova VM** (cluster standalone simples):
```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup-k3s.yml -e target=homelab-monitoring
ansible-playbook -i inventory/hosts.yml playbooks/setup-k3s.yml -e target=homelab-dev
```

Após o bootstrap, o ArgoCD da VM lê `nodes/<vm>/apps/` do `homelab-gitops` e sincroniza os serviços automaticamente.

### 4. Obter senha do ArgoCD
```bash
ansible-playbook -i inventory/hosts.yml playbooks/get_argocd_pass.yml -l <host>
```

## Variáveis principais (`setup-k3s.yml`)

| Variável | Padrão | Descrição |
|----------|--------|-----------|
| `k3s_version` | `v1.30.0+k3s1` | Versão do k3s |
| `argocd_version` | `v3.3.0` | Versão do ArgoCD |
| `homelab_gitops_repo` | `ojasonw/homelab-gitops` | Repo GitOps fonte de verdade |
| `target` | — | VM alvo para novo cluster (Play 2) |
