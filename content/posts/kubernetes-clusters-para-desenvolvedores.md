---
title: "Kubernetes Clusters Para Desenvolvedores"
date: 2022-05-24T16:14:43-03:00
draft: false
tags: ["kubernetes", "docker", "iac", "terraform", "ansible", "linode", "kind", "ingress", "linux"]
categories: ["tech", "devops"]
---

# KIND - Kubernetes Clusters para Desenvolvedores
## Infrastructure as Code
Para este workshop iremos provisionar uma máquina na nuvem, conforme abaixo:

| Item | Descrição |
|------|-----------|
| Nuvem | [Linode](../fornecedores/linode.md) |
| IaC | [Terraform](../fornecedores/terraform.md) |
| IT Automation Tool |  [Ansible](../fornecedores/ansible.md) |
| Linux |  [Ubuntu](../fornecedores/ubuntu.md) |

## Criar um Access Token no Linode
* [Linode Guides - Get an API Access Token](https://www.linode.com/docs/products/tools/linode-api/guides/get-access-token/)

## Instalar Terraform CLI
* [Install Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)

## Terraform Linode Provider
* [Linode Provider](https://registry.terraform.io/providers/linode/linode/latest/docs)


## Estrutura de diretórios do workshop
```bash
$ tree
.
├── IaC
│   ├── ansible
│   │   ├── hosts
│   │   ├── hosts.sample
│   │   └── playbook.yml
│   └── terraform
│       ├── main.tf
│       ├── terraform.auto.tfvars
│       ├── terraform.auto.tfvars.sample
```

---

## TERRAFORM - Criação da máquina virtual

A máquina virtual foi provisionada utilizando a ferramenta de [Infrastructure as Code](https://en.wikipedia.org/wiki/Infrastructure_as_code) [Terraform](https://terraform.io)

Para esta etapa, entraremos no diretório **IaC/terraform**. Estrutura abaixo listada:

```bash
$ tree
.
├── IaC
│   └── terraform
│       ├── main.tf
│       ├── terraform.auto.tfvars
│       ├── terraform.auto.tfvars.sample
```

Entre no diretório do terraform.

```bash
cd IaC/terraform
```

### Arquivo: **main.tf**

```terraform
# Autor: Fábio Sartori
# Copyright: 20220524

terraform {
  required_providers {
    linode = {
      source = "linode/linode"
      version = "1.26.1"
    }
  }
}

provider "linode" {
    token = var.shared_token
}

resource "linode_instance" "ubuntupi" {
    label = var.ubuntu_label
    image = var.ubuntu_image
    type = var.ubuntu_type
    region = var.shared_region
    root_pass = var.shared_root_pass
    authorized_keys = var.authorized_keys
    private_ip = var.shared_private_ip
    tags = var.ubuntu_tags
}

variable "shared_token" {}
    variable "region" {
    default = "us-central"
}
variable "shared_root_pass" {}
variable "shared_private_ip" {}
variable "shared_region" {}
variable "authorized_keys" {}
variable "ubuntu_tags" {}
variable "ubuntu_label" {}
variable "ubuntu_image" {}
variable "ubuntu_type" {}
```

### Arquivo **terraform.auto.tfvars**
Para criar este acruivo de variáveis, use como base o arquivo de exemplo **terraform.auto.tfvars.sample**

```properties
# Token de usuário do linode (ESTE TOKEN É SOMENTE PARA EXEMPLIFICAR, POIS NÃO É VÁLIDO)
shared_token = "00e3261a6e0d79c329445acd540fb2b07187a0dcf6017065c8814010283ac67f"
# Senha de root dos servidores
shared_root_pass = "_SENHA_DO_ROOT_"
# Região do Datacenter Linode
shared_region = "us-central"
# true se a máquina tiver ip privado
shared_private_ip = true
# Array com as chaves públicas autorizadas a logar no servidor (AS CHAVES ABAIXO SÃO SOMENTE PARA EXEMPLIFICAR, POIS NÃO SÃO VÁLIDAS)
authorized_keys = ["ssh-rsa 724c70d82400c99516c0b9513125c14bfd995fecfb7cc9ab757943459c0397fff92bbc8e1b9facbbaec8f27e0426cb4f7def7c6c83df3f3955373729a366303b= juvenal@bla"]
# Tags para identificar o servidor
ubuntu_tags = ["workshop","kind"]
# Label para identificar o servidor
ubuntu_label = "ubuntu_pi"
# Imagem de Linux para a criação do servidor
ubuntu_image = "linode/ubuntu20.04"
# Tipo do linode com 4GB de RAM (verifique os custo $$ antes)
ubuntu_type = "g6-standard-4"

```

### Inicializando o terraform

```bash
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of linode/linode from the dependency lock file
- Using previously-installed linode/linode v1.26.1

Terraform has been successfully initialized!
```

### Validando o plano de execução do terraform

```bash
$ terraform plan

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
  + create
  .
  .
  .
  Plan: 1 to add, 0 to change, 0 to destroy.
```

### Aplicando o plano de execução

Para criar a máquina virtual, aplique o plano de execução, conforme abaixo.

```bash
$ terraform apply

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # linode_instance.ubuntupi will be created
  + resource "linode_instance" "ubuntupi" {
  .
  .
  .
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:  
```

Confirme, digitando **yes** e pressionando **\<ENTER\>**


> **Ao final da execução, você deverá receber uma mensagem conforme abaixo:**
> .
> .
> linode_instance.ubuntupi: Creation complete after 55s [id=36484787]
> Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

### Listando os dados da Máquina Virtual

```bash

```

### Listando IP público da Máquina Virtual

```bash
terraform show -json | jq .values.root_module.resources[0].values.ip_address
```

---

# Setup de um [Kind](../fornecedores/kind.md) Cluster pronto para [Ingress](../fornecedores/ingress.md)

## Criação de um script contendo a configuração do cluster e pronto para o ingress com mapeamentos nas portas 80 e 443

Crie um arquivo chamado **cluster-config.yaml** com o conteúdo abaixo:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: juvenal-ingress
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

## Criação do cluster

Aplique o script do passo anterior e crie o cluster

```bash
kind create cluster --config=cluster-config.yaml
```

A partir deste momento, o cluster está pronto para a instalação de um ingress controller.

## Instalação

```bash
kubectl apply --filename https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml
```

Executando Aguardando a conclusão 

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

## Listando a arquitetura do cluster

Neste exemplo o **juvenal-ingress**, é o nome do cluster criado no Kind

### Kind Cluster

```bash
$ kind get clusters
juvenal-ingress
```

### Container [Docker](../fornecedores/docker.md) referente ao Kind Cluster

```bash
$ kubectl get nodes | grep "juvenal-ingress"                                                            
NAME                                   STATUS   ROLES                  AGE   VERSION
juvenal-ingress-control-plane   Ready    control-plane,master   50m   v1.21.1
```

### Listar o IP do cluster para acesso externo

```bash
docker container inspect juvenal-ingress-control-plane --format '{{ .NetworkSettings.Networks.kind.IPAddress }}'
```

## Exemplo de Container para Testes
Seguem testes para validação da instalação do ingress

### Criar um container simples para testes
```bash
kubectl run juvenal-hello --expose --image nginxdemos/hello:plain-text --port 80
```

### Criar regras de tráfego para Teste

* Crie o script abaixo, com as regras de tráfego com o nome de **juvenal-ingress.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: juvenal
spec:
  rules:
    - host: juvenal.example.com
      http:
        paths:
          - pathType: ImplementationSpecific
            backend:
              service:
                name: hello
                port:
                  number: 80
```

Aplique o script com o comando abaixo:

```bash
kubectl create -f juvenal-ingress.yaml
```

### Testando a regra do ingress

#### Cadastro no /etc/hosts (se não tiver DNS)

Adicione a linha abaixo no arquivo /etc/hosts

```bash
172.18.0.2	juvenal.example.com
```

#### Testando a configuração

```bash
$ curl -X GET http://juvenal.example.com/hello
Server address: 10.244.0.8:80
Server name: juvenal-hello
Date: 11/May/2022:02:21:07 +0000
URI: /hello
Request ID: ea75707dcde6e5ec6530bd5205f87153
```

---

## Operações com contextos e múltiplos clusters

### Listar contexto atual

```bash
$ kubectl config current-context
kind-k8s-teste
```

### Listar todos contextos

```bash
$ kubectl config get-contexts   
CURRENT   NAME              CLUSTER           AUTHINFO          NAMESPACE
*         kind-k8s-teste    kind-k8s-teste    kind-k8s-teste    
          kind-wso2am-poc   kind-wso2am-poc   kind-wso2am-poc   
          minikube          minikube          minikube          default
```

### Trocar de contexto

```bash
$ kubectl config set-context kind-wso2am-poc                                            1 ⨯
Context "kind-wso2am-poc" modified.
```

---

## Referências do kubectl

[https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)