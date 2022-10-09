---
title: "Deploy do Wordpress automatizado na AWS usando ansible e terraform"
date: 2022-09-27T21:29:31-03:00
draft: false
tags: ['terraform', 'wordpress', 'portifolio', 'ansible', 'aws']

cover:
  image: "/banner/deploy-wordpress.png" # image path/url
  alt: "<alt text>" # alt text
  caption: "<text>" # display caption under cover
  relative: false # when using page bundles set this to true
  hidden: false # only hide on current single page
---

## O objetivo desse projeto é realizar o deploy do wordpress de forma automatizada
Confira o código completo aqui no [github](https://github.com/juam-sv/wordpress-aws-deploy)

> ### Para isso estou usando as seguintes tecnologias
>
> - AWS
>   - RDS
>   - EC2
> - Terraform
> - Ansible
> - Docker
> - Wordpress
> - NGINX
>
> 

### Estrutura básica dos diretórios
```bash
╭─    ~/Code/deploy-wordpress   juam-veras !16 ?5 ▓▒░······················░▒▓ ✔  16:29:06 ─╮
╰─ tree -L 3
.
├── ansible # Contendo as roles, vars e playbooks pro deploy da arquitetura.
│   ├── bkp-hosts
│   ├── hosts
│   ├── main.yml
│   ├── roles
│   │   ├── common # Role pra pacotes basicos
│   │   ├── docker-cluster  # Inicia o cluster do swarm
│   │   ├── docker-deploy # Faz o deploy dos docker-composes.
│   │   ├── docker-install # Instala o docker, pip requeriments e add user aos grupos
│   │   ├── wordpress # Instalação e configuração do wordpress-apache
│   │   └── wp-ports
│   └── vars
│       ├── main.yml
│       └── tf_ansible_vars_file.yml # Variaveis geradas pelo terraform
├── README.md
├── terraform # Arquivos do terraform pra deploy da ec2, rds, networks, security groups etc
│   ├── data.tf
│   ├── init.sh
│   ├── instances.tf
│   ├── locals.tf
│   ├── main.tf
│   ├── network.tf
│   ├── outputs.tf
│   ├── provider.tf
│   ├── rds.tf
│   ├── security-group.tf
│   ├── terraform.tfstate
│   ├── terraform.tfstate.backup
│   └── variables.tf # Arquivo pra definir as variaveis do terraform
├── terraform.tfstate
└── Vagrantfile

10 directories, 21 files

```
## Terraform
Neste projeto o terraform está sendo naturalmente responsavel por gerar toda a parte de Rede como VPC, Subnet, Gateway, EC2, RDS, Security group como pode ser visto abaixo.

![Diagrama](/imgs/resized.svg)

Já o processo de execução é o classico terraform init, plan e apply
```bash

╭─    ~/Code/wordpress-aws-deploy/terraform   main !1 ?1 ▓▒░······················░▒▓ ✔   us-east-1  19:35:57 ─╮
╰─ terraform init                                                                                                                                                                                                                  ─╯

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 4.16"...
- Finding latest version of hashicorp/local...

.....

╭─    ~/Code/wordpress-aws-deploy/terraform   main !1 ?1 ▓▒░······················░▒▓ ✔   7s   us-east-1  19:36:07 ─╮
╰─ terraform plan                                                                                                                                                                                                                  ─╯
data.aws_ami.ubuntu_ami: Reading...
data.aws_ami.ubuntu_ami: Read complete after 0s [id=ami-0c1704bac156af62c]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_db_instance.mysql_db will be created
  + resource "aws_db_instance" "mysql_db" {
      + address                               = (known after apply)
      + allocated_storage                     = 10
      + apply_immediately                     = (known after apply)
      + arn                                   = (known after apply)
      + auto_minor_version_upgrade            = true
    }

.......

Plan: 14 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + private_ip   = (known after apply)
  + public_dns   = (known after apply)
  + public_ip    = (known after apply)
  + rds_endpoint = (known after apply)

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.

.......

╭─    ~/Code/wordpress-aws-deploy/terraform   main !1 ?1 ▓▒░······················░▒▓ ✔   5s   us-east-1  19:41:09 ─╮
╰─ terraform apply                                                                                                                                                                                                                 ─╯
data.aws_ami.ubuntu_ami: Reading...
data.aws_ami.ubuntu_ami: Read complete after 1s [id=ami-0c1704bac156af62c]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_db_instance.mysql_db will be created
  + resource "aws_db_instance" "mysql_db" {
      + address                               = (known after apply)
      + allocated_storage                     = 10
      + apply_immediately                     = (known after apply)
  
.......

aws_db_instance.mysql_db: Still creating... [4m30s elapsed]
aws_db_instance.mysql_db: Creation complete after 4m39s [id=terraform-database]
local_file.tf_ansible_vars_file_new: Creating...
local_file.tf_ansible_vars_file_new: Creation complete after 0s [id=fab95d8ef388fdb452006e1da8f2fc295df0644c]

Apply complete! Resources: 14 added, 0 changed, 0 destroyed.

Outputs:

private_ip = "10.0.1.155"
public_dns = "ec2-54-227-217-109.compute-1.amazonaws.com"
public_ip = "54.227.217.109"
rds_endpoint = "terraform-database.c2vyhwdzispb.us-east-1.rds.amazonaws.com:3306"
```
A partir dai a parte de infraestrutura em si está pronta, o terraform gerará o arquivo hosts e tf_ansible_vars_file.yml os valores prontos para serem executados no ansible.

```bash
╭─    ~/Code/wordpress-aws-deploy/ansible   main !1 ?1 ▓▒░······················░▒▓ ✔   us-east-1  19:48:21 ─╮
╰─ cat hosts                                                                                                                                                                                                                       ─╯
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: hosts
       │ Size: 398 B
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # Ansible hosts containing inventory values from Terraform.
   2   │ # Generated by Terraform mgmt configuration.
   3   │ 
   4   │ [all:vars]
   5   │ ansible_user='ubuntu'
   6   │ ansible_become=yes
   7   │ ansible_ssh_private_key_file=~/.ssh/madra.pem
  11   │ ansible_python_interpreter=/usr/bin/python3
  12   │ 
  13   │ [aws]
  14   │ 54.227.217.109
  15   │ 
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

╭─    ~/Code/wordpress-aws-deploy/ansible   main !1 ?1 ▓▒░······················░▒▓ ✔   us-east-1  19:48:24 ─╮
╰─ cat vars/tf_ansible_vars_file.yml                                                                                                                                                                                               ─╯
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: vars/tf_ansible_vars_file.yml
       │ Size: 330 B
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # Ansible vars_file containing variable values from Terraform.
   2   │ # Generated by Terraform mgmt configuration.
   3   │ 
   4   │ WORDPRESS_DB_HOST: terraform-database.c2vyhwdzispb.us-east-1.rds.amazonaws.com:3306
   5   │ WORDPRESS_DB_USER: wordpress
   6   │ WORDPRESS_DB_PASSWORD: supersenha123
   7   │ WORDPRESS_DB_NAME: wordpress
   8   │ 
   9   │ mgmt_net: 10.0.1.0/24
  10   │ 
  11   │ user: ubuntu
  12   │     

```
Para executar o ansible o comando é o seguinte:


```bash
╭─    ~/Code/wordpress-aws-deploy/ansible   main !1 ?1 ▓▒░···································································································································░▒▓ ✔   us-east-1  19:51:23 ─╮
╰─ cd ansible && ansible-playbook -i hosts main.yml -vvv                                                                                                                                                                                         ─╯
ansible-playbook [core 2.13.4]
  config file = /home/juamsv/.ansible.cfg
  configured module search path = ['/home/juamsv/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.10/site-packages/ansible
  ansible collection location = /home/juamsv/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible-playbook
  python version = 3.10.7 (main, Sep  6 2022, 21:22:27) [GCC 12.2.0]
  jinja version = 3.1.2

.....

```

Agora é só esperar alguns minutos e acessar pelo endpoint fornecido no output do terraform :D.

### Proximos ajustes:

- [ ] Validar se existe ssh key, caso não, gerar uma.
- [ ] Corrigir o redirecionamento do NGINX
- [ ] Adicionar o variables.tfvars.exemple
- [ ] Rodar o ansible automaticamente com o ansible_local pelo terraform
- [ ] arrumar algumas coisas do ansible ex: flush dos handlers ao final de algumas roles.