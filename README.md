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

Hosts ativos em `inventory/hosts.yml`:

```yaml
all:
  hosts:
    core-k3s:
      ansible_host: 192.168.15.92
      ansible_user: homelab
    monitoring:
      ansible_host: 192.168.15.201
      ansible_user: ubuntu
    dev:
      ansible_host: 192.168.15.202
      ansible_user: ubuntu
    homolog:
      ansible_host: 192.168.15.203
      ansible_user: ubuntu
```

> `monitoring`, `dev` e `homolog` são VMs provisionadas via Terraform. O `ansible_user` é `ubuntu` (criado pelo cloud-init).

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

`-e target` é **obrigatório** — não há default. Especifique sempre o host alvo:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup-k3s.yml \
  -e target=monitoring --ask-vault-pass

ansible-playbook -i inventory/hosts.yml playbooks/setup-k3s.yml \
  -e target=dev --ask-vault-pass

ansible-playbook -i inventory/hosts.yml playbooks/setup-k3s.yml \
  -e target=homolog --ask-vault-pass
```

Após o bootstrap, o ArgoCD da VM lê `nodes/<vm>/apps/` do `homelab-gitops` e sincroniza os serviços automaticamente.

### 4. Obter senha do ArgoCD
```bash
ansible-playbook -i inventory/hosts.yml playbooks/get_argocd_pass.yml -l <host>
```

## Variáveis principais (`setup-k3s.yml`)

| Variável | Descrição |
|----------|-----------|
| `target` | **Obrigatório.** Nome do host no inventário (ex: `monitoring`, `dev`, `homolog`) |
| `k3s_version` | Versão do k3s (padrão: `v1.30.0+k3s1`) |
| `argocd_version` | Versão do ArgoCD (padrão: `v3.3.0`) |
| `homelab_gitops_repo` | Repo GitOps fonte de verdade |
