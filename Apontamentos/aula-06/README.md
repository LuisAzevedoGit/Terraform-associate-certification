# Terraform Associate: Resumo da Aula 6 (simplificado)

---

## 1. Workspaces: Local vs HCP Terraform

| Componente | Terraform local | HCP Terraform |
|---|---|---|
| **Configuração Terraform** | No **disco** | Num **repositório de controlo de versões** ligado |
| **Valores das variáveis** | Ficheiros `.tfvars` ou argumentos da CLI | No **workspace** (ou em **variable sets** partilhados) |
| **State** | No disco ou num **remote backend** | No **workspace** |
| **Credenciais e segredos** | Em **variáveis de ambiente** da shell ou inseridos em prompts | No workspace, guardados como **variáveis sensíveis** (ou obtidos por **dynamic credentials**, ver Aula 5, secção 17) |

---

## 2. Policy as code: Sentinel e OPA

**Policy as code:** escrever código para **automatizar políticas** regulatórias ou de governação.

O HCP Terraform suporta **dois frameworks** de policy as code:

| Framework | Notas |
|---|---|
| **Sentinel** | Framework da HashiCorp, **embutida**, integrada com o Terraform |
| **OPA** (Open Policy Agent) | Framework open source, com a linguagem **Rego** |

- As policies são agrupadas em **policy sets** e aplicadas a **workspaces ou projects**.
- Correm **depois do `plan` e antes do `apply`**.
- **Policy enforcement** está disponível nos planos **Standard** e **Premium** (ver Aula 5, Parte 4).

### Características do Sentinel

| Característica | Significado |
|---|---|
| **Embedded** | **Rejeita** comportamentos que violam as regras, em vez de apenas os detetar passivamente |
| **Fine-grained, condition-based** | Decide com base nas **condições de outros valores** |
| **Multiple enforcement levels** | **Advisory**, **soft mandatory** e **hard mandatory**: permitem avisar ou rejeitar |
| **External information** | Usa informação externa para decisões mais completas |
| **Multi-cloud** | Compatível com várias clouds |

### Níveis de enforcement (Sentinel)

| Nível | O que acontece quando a policy falha |
|---|---|
| **Advisory** | Só **avisa**; a run continua |
| **Soft mandatory** | **Bloqueia**, mas um utilizador com permissão pode **fazer override** |
| **Hard mandatory** | **Bloqueia** sempre; **sem override** possível |

> Nas policies OPA os níveis são **advisory** e **mandatory**.

### Exemplos de policies na AWS

- Restringir os **tipos de EC2**
- Impor **tags obrigatórias** em recursos AWS que as suportem
- Restringir **availability zones**
- Proibir blocos CIDR **`0.0.0.0/0`** em security groups
- Exigir **S3 privados e encriptados**
- Exigir **DNS hostnames ativos** nas VPCs

### Exemplo de código (Sentinel)

```python
import "tfplan-functions" as plan

allowed_zones = [
  "us-east-1a", "us-east-1b"
]

allEC2Instances = plan.find_resources("aws_instance")

violatingEC2Instances = plan.filter_attribute_not_in_list(
  allEC2Instances, "availability_zone", allowed_zones, true
)

main = rule {
  length(violatingEC2Instances["messages"]) is 0
}
```

Lê-se: *encontra todas as instâncias EC2, filtra as que estão fora das zonas permitidas, e a regra `main` só passa se não houver nenhuma violação.*

### Benefícios de policy as code

| Benefício | Descrição |
|---|---|
| **Sandboxing** | Cria "guardrails" que evitam ações perigosas ou removem processos manuais |
| **Codification** | As políticas ficam **bem documentadas** e representam o que é imposto |
| **Version control** | Histórico encadeado das alterações |
| **Testing** | Testa **sintaxe e comportamento** das policies |
| **Automation** | Integra-se com sistemas para **auto-remediar** ou **notificar** |

**Sentinel e policy as code:**

- **Linguagem:** amigável para programadores, embebível e segura
- **Desenvolvimento:** fornece uma **CLI**
- **Testes:** fornece um **framework de testes**

### Policies com Terraform

```
SCM → HCP Terraform → plan → policies (Sentinel / OPA) → apply → nova infraestrutura
```

> As policies correm **depois do `plan` e antes do `apply`**.

---

## 3. HashiCorp Packer

> **Nota 004:** o Packer não aparece nos objetivos do exame 004. Prioridade baixa.

Cria uma **imagem** (build image) que fica guardada num repositório.

**Vantagens de usar uma imagem antes do deploy:**

- **Infraestrutura imutável**
- As VMs da frota ficam **idênticas em configuração** (1 para 1)
- **Deploys mais rápidos** para vários servidores
- **Deteção antecipada** de alterações em pacotes ou de tecnologia obsoleta

### Fluxo típico

```
git commit → CI/CD → dispara build da imagem → Packer → provisiona a imagem
→ guarda a imagem (pode usar-se com Ansible, por exemplo)
→ referencias a imagem no Terraform → provisiona a infraestrutura (ex.: AWS)
```

### Packer Template

Configura uma **máquina ou container**. Os templates usam **HCL** (ficheiros `.pkr.hcl`).

```hcl
variable "ami_id" {
  type    = string
  default = "ami-01e78c5619c5e68b4"
}

locals {
  app_name = "httpd"
}

source "amazon-ebs" "httpd" {
  ami_name      = "my-server-${local.app_name}"
  instance_type = "t2.micro"
  region        = "us-west-2"
  source_ami    = "${var.ami_id}"
  ssh_username  = "ec2-user"
  tags = {
    Env  = "DEMO"
    Name = "my-server-${local.app_name}"
  }
}

build {
  sources = ["source.amazon-ebs.httpd"]

  provisioner "shell" {
    script = "script/script.sh"
  }

  post-processor "shell-local" {
    inline = ["echo foo"]
  }
}
```

| Bloco | Função |
|---|---|
| **`source`** | Diz **onde** e **que tipo** de imagem construir. Neste caso cria uma **AMI baseada em EBS**, guardada na AWS em EC2 Images |
| **`build`** | Permite fornecer **scripts de configuração** |
| **`provisioner`** | Suporta muitos: **Chef, Puppet, Ansible, PowerShell, Bash, Salt**... |
| **`post-processor`** | Corre **depois** de a imagem estar construída; serve para **fazer upload de artefactos** ou **re-empacotar** |

### Integração Terraform + Packer

1. **Construir a imagem:** o Packer **não é uma ferramenta de desenvolvimento**, por isso tens de o correr **manualmente** ou **automatizar** a construção.
2. **Referenciar a imagem:** depois de construída, referencias a imagem como um **data source**.

---

## 4. Terraform e Consul

> **Nota 004:** o Consul não aparece nos objetivos do exame 004. Prioridade baixa.

**Consul:** plataforma de **service networking** que oferece:

| Funcionalidade | Descrição |
|---|---|
| **Service discovery** | **Regista** serviços na rede; permite **comunicação direta**, sem ponto único de falha (sem load balancers) |
| **Service mesh** | Gere o **tráfego de rede**; é uma camada de comunicação por cima dos containers (pensa em *middleware*) |
| **App configuration** | Capacidades de configuração de aplicações |

> Útil em arquiteturas de **microserviços** ou orientadas a serviços, com **centenas ou milhares** de serviços (containers ou workloads).

---

## 5. HashiCorp Vault

Serve para **aceder a segredos de forma segura**. É instalado num servidor onde:

- **Admins** gerem os segredos diretamente
- **Operators** (developers) acedem aos segredos por **API**

**Interface unificada** para qualquer segredo (AWS secrets, Azure service principals...), com controlo de acesso apertado:

| Princípio | Significado |
|---|---|
| **Just-in-time** | Dar acesso **apenas durante um intervalo de tempo** |
| **Just enough privilege** | Dar **só as permissões necessárias** |
| **Audit log** | Regista **logs de auditoria** detalhados |

### Terraform + Vault

Se as credenciais forem dadas **na hora** e **expirarem rapidamente**, reduz-se a **superfície de ataque** da máquina local.

O Vault injeta **segredos de curta duração** no momento do `terraform apply`, através de **data sources**.

**Pré-requisitos:**

1. Um **servidor Vault**
2. Um **engine** configurado (ex.: **AWS secrets engine**)
3. O Vault cria um **machine user** na AWS
4. O Vault gera **credenciais AWS de curta duração**
5. O Vault gere e aplica a **policy AWS**

```hcl
data "vault_aws_access_credentials" "creds" {
  backend = data.terraform_remote_state.admin.outputs.backend
  role    = data.terraform_remote_state.admin.outputs.role
}

provider "aws" {
  region     = var.region
  access_key = data.vault_aws_access_credentials.creds.access_key
  secret_key = data.vault_aws_access_credentials.creds.secret_key
}
```

- No `terraform apply`, são obtidas credenciais de curta duração, válidas **apenas durante a run atual**.
- **Cada `apply` gera credenciais novas.**

> **Atenção (004):** os valores lidos por **data sources ficam guardados no state**, mesmo que as credenciais sejam de curta duração. A abordagem mais segura usa **ephemeral resources** do provider, que **não ficam no state** (ver Aula 2, secção 12). Consulta o Registry para ver que ephemeral resources o provider Vault oferece.

**Outras formas de evitar segredos no código:**

- **Ephemeral values e write-only arguments** (Aula 2, secção 12).
- **Dynamic provider credentials** no HCP Terraform (Aula 5, secção 17).
- **Variáveis sensíveis** no workspace (continuam a ser guardadas no state se usadas num recurso).

---

## 6. Outras ferramentas

> **Nota 004:** esta secção não aparece nos objetivos do exame 004. Prioridade baixa; mantém como cultura geral.

### Atlantis

Ferramenta **open source** para **automatizar pull requests de Terraform**.

### CDK for Terraform (CDKTF)

| | AWS CDK | CDK for Terraform |
|---|---|---|
| Tipo | IaC **imperativo**, com SDK na tua linguagem (Python, Java, Ruby...) | Igual, mas para Terraform |
| Gera | Templates **CloudFormation** | Configuração **Terraform** |
| Providers | Só AWS | **Qualquer provider** disponível |

### Gruntwork

Empresa que cria ferramentas DevOps que **estendem ou usam** o Terraform:

- **Terragrunt**
- **Terratest**
- Infrastructure as Code Library
- Gruntwork Pipelines

### Terragrunt

**Wrapper fino** do Terraform com ferramentas extra:

- Manter as configurações **DRY**
- Trabalhar com **vários módulos Terraform**
- Gerir o **remote state**

> **DRY (Don't Repeat Yourself):** metodologia que abstrai código repetido em funções, módulos ou bibliotecas, reduzindo a complexidade.

### Terratest

Biblioteca para **testes unitários e de integração** da infraestrutura:

1. **Faz deploy temporário**
2. **Valida** os resultados
3. **Destrói** o ambiente de teste

> Os testes são escritos em **Go (Golang)**.

---

## 7. Testes em Terraform (pirâmide)

| Nível | Quando usar | Ferramentas |
|---|---|---|
| **e2e Tests** (topo) | Tens de montar um **ambiente de teste persistente** | Gruntwork Reference Architecture |
| **Integration Tests** | Testar **vários módulos IaC** a funcionar em conjunto | **Terratest** |
| **Unit Tests** | Testar **um único módulo** (dividir os módulos em unidades muito pequenas) | **Terratest**, Kitchen-terraform, Inspec |
| **Static analysis** (base) | Testar o código **sem fazer deploy** | **Tflint**, `terraform validate`, `terraform plan`, terraform-compliance, **Sentinel** |

---

## Pontos-chave para decorar

1. Workspace no HCP Terraform guarda **config (via VCS), variáveis, state e segredos (variáveis sensíveis)**.
2. **Policy as code** no HCP Terraform: **Sentinel** ou **OPA**, em **policy sets**, a partir do plano **Standard**; corre **entre plan e apply**.
3. Níveis do Sentinel: **advisory, soft mandatory, hard mandatory**; OPA: **advisory, mandatory**.
4. Benefícios de policy as code: sandboxing, codification, version control, testing, automation.
5. **Packer** = cria imagens (imutáveis), usa **HCL** (`.pkr.hcl`), não é ferramenta de desenvolvimento.
6. Packer: `source` (onde/tipo), `build`, **provisioners**, **post-processors**.
7. Terraform referencia a imagem do Packer como **data source**.
8. **Consul** = service discovery + service mesh + app configuration.
9. **Vault** = segredos; **just-in-time** e **just enough privilege**; credenciais **de curta duração**.
10. Valores lidos por **data source ficam no state**; **ephemeral resources** não ficam.
11. **Atlantis** = automatiza pull requests de Terraform (open source).
12. **CDKTF** = CDK que gera **Terraform** (qualquer provider); AWS CDK gera **CloudFormation**.
13. **Terragrunt** = wrapper, mantém o código **DRY**; **Terratest** = testes em **Go**.
14. Pirâmide de testes: **static analysis → unit → integration → e2e**.