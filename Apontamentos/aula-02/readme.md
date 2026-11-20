# Terraform Associate: Resumo da Aula 2 (simplificado)

---

## 1. Provisioners

Servem para **instalar software, editar ficheiros e configurar máquinas** criadas com Terraform.

> **Para o exame:** são considerados **má prática**, porque **não ficam registados no state** do Terraform e quebram a imutabilidade. As alternativas preferidas são **cloud-init** e **Packer**.

> **Nota 004:** os provisioners **não aparecem na lista de objetivos do exame 004**. Prioridade baixa.

### local-exec

Executa comandos **na tua máquina local** depois de o recurso ser criado.

```hcl
resource "aws_instance" "web" {
  # ...
  provisioner "local-exec" {
    command = "echo ${self.private_ip} >> private_ips.txt"
  }
}
```

| Argumento | Função |
|---|---|
| `command` | **Obrigatório**: o comando a executar |
| `working_dir` | Diretório onde corre o comando |
| `interpreter` | Programa que executa o comando (bash, PowerShell...). Ex.: `["PowerShell", "-Command"]` |
| `environment` | Variáveis de ambiente (chaves, segredos...) |

### remote-exec

Executa comandos **no recurso criado** (na máquina remota). Tem 3 modos:

- **`inline`**: lista de comandos em texto
- **`script`**: um único script
- **`scripts`**: vários scripts

```hcl
provisioner "remote-exec" {
  inline = ["puppet apply", "consul join ${aws_instance.web.private_ip}"]
}

provisioner "remote-exec" {
  scripts = ["./setup_user.sh", "/home/lfoa/Desktop/bootstrap"]
}
```

### file

**Copia ficheiros ou diretórios** da máquina local para o novo recurso.

```hcl
provisioner "file" {
  source      = "conf/myapp.conf"
  destination = "/etc/myapp.conf"
}
```

### connection

Diz ao provisioner **como ligar-se** ao recurso (SSH, WinRM...).

```hcl
connection {
  type     = "ssh"
  user     = "root"
  password = var.root_password
  host     = var.host
}
```

### null_resource e terraform_data

- **null_resource:** recurso "vazio", sem ligação a nenhum provider. Serve para correr provisioners com `connection` e `triggers`.
  - `triggers`: valores que, ao mudarem, **fazem os provisioners correrem de novo**.
- **terraform_data:** semelhante ao null_resource, mas **não precisa de nenhum provider**.

---

## 2. Providers

**Plugins** que permitem ao Terraform comunicar com:

- Clouds (AWS, Azure...)
- SaaS (GitHub...)
- Outros (Kubernetes, PostgreSQL...)

| Tier | Significado |
|---|---|
| **Official** | Pertence e é mantido pela **HashiCorp** (namespace `hashicorp`) |
| **Partner** | Escrito, mantido, validado e publicado por **empresas terceiras** sobre as suas próprias APIs (tem badge de Partner). **Antigo nome: "Verified"** |
| **Community** | Publicado pela comunidade, **sem garantias** de manutenção ou compatibilidade |
| **Archived** | Providers Official ou Partner que **já não são mantidos** (ex.: API descontinuada) |

- `terraform init` **descarrega** os plugins de providers necessários.
- `terraform providers` **lista** os providers que estás a usar.

### Dependency lock file

- O `terraform init` cria/atualiza o ficheiro **`.terraform.lock.hcl`**, que regista as **versões exatas** dos providers escolhidas (e checksums).
- Deve ser **commitado no VCS**, para que toda a equipa (e o HCP Terraform) use as mesmas versões.
- Para subir de versão: `terraform init -upgrade`.
- `required_providers` + `version` dizem **o que é permitido**; o lock file guarda **o que foi escolhido**.

### Provider alias (providers alternativos)

Para usar **mais de uma configuração do mesmo provider** (ex.: duas regiões):

```hcl
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_instance" "foo" {
  provider = aws.west   # referencia o alias
}
```

Num **módulo filho**, declaras os aliases esperados com `configuration_aliases`:

```hcl
terraform {
  required_providers {
    mycloud = {
      source                = "mycorp/mycloud"
      version               = "~> 1.0"
      configuration_aliases = [mycloud.default]
    }
  }
}
```

---

## 3. Terraform Registry

Website para **procurar, descarregar ou publicar** providers e módulos.

- **Provider:** plugin que faz o mapeamento para a API de um serviço cloud.
- **Módulo:** grupo de ficheiros de configuração reutilizáveis. Impõe boas práticas, reduz código e poupa tempo.
- **Private registry:** no **HCP Terraform** podes publicar módulos privados.
- Docs do AWS: https://registry.terraform.io/providers/hashicorp/aws/latest/docs

> **Fine line:** perceber a divisão de responsabilidades entre o **Terraform** (provisionar infraestrutura) e as ferramentas de **gestão de configuração** de terceiros (configurar o software dentro das máquinas).

---

## 4. Linguagem Terraform

- Ficheiros com extensão **`.tf`** ou **`.tf.json`**.
- Estrutura:

```hcl
<tipo de bloco> "<label>" "<label>" {
  <identificador> = <expressão>   # argumento
}

resource "aws_vpc" "main" {
  cidr_block = var.base_cidr_block
}
```

| Elemento | O que é |
|---|---|
| **Blocks** | Contentores de conteúdo que representam um objeto (têm tipo e labels) |
| **Arguments** | Atribuem um valor a um nome |
| **Expressions** | Representam um valor, literal ou por referência |

**HCL** (HashiCorp Configuration Language) é usada também noutras ferramentas, como o Packer (`.pkr.hcl`).

### Terraform settings

Bloco `terraform { ... }`, onde se define:

- `required_version`: versão do Terraform
- `required_providers`: providers a descarregar
- `experiments`: funcionalidades experimentais da linguagem
- `provider_meta`: informação específica do módulo para providers

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 2.7.0"
    }
  }
}
```

---

## 5. Variáveis de entrada (Input Variables)

São **parâmetros** de módulos Terraform, definidos com blocos `variable`.

| Argumento | Função |
|---|---|
| `default` | Valor por defeito |
| `type` | Tipo do valor |
| `description` | Descrição |
| `validation` | Regras de validação (ver secção 11) |
| `sensitive` | Esconde o valor no output do Terraform (mas **fica no state**) |
| `ephemeral` | O valor **não é guardado no state nem no plan** (ver secção 12) |

```hcl
variable "image_id" {
  type = string
}

variable "AZ" {
  type    = list(string)
  default = ["us-west-1a"]
}

variable "docker_ports" {
  type = list(object({
    internal = number
    external = number
    protocol = string
  }))
  default = [{
    internal = 8300
    external = 8300
    protocol = "tcp"
  }]
}
```

### Formas de definir valores

- **Ficheiros `.tfvars`:** definem vários valores de uma vez.
- **Variáveis de ambiente:** com prefixo `TF_VAR_`. Ex.: `export TF_VAR_image_id=ami-abc123`
- **Linha de comandos:**
  - `-var-file=dev.tfvars` (carrega um ficheiro)
  - `-var 'ec2_type=t2.medium'` (um valor)

### Precedência (do mais fraco para o mais forte)

1. Variáveis de ambiente
2. `terraform.tfvars`
3. `terraform.tfvars.json`
4. `*.auto.tfvars`
5. `-var` e `-var-file` (**vencem sempre**)

---

## 6. Output Values

Valores calculados **depois do `terraform apply`** (ex.: IP de uma máquina). Servem para integração programática e para referenciar entre stacks via state.

```hcl
output "db_password" {
  value       = aws_db_instance.db.password
  description = "db password"
  sensitive   = true
}
```

- `sensitive = true` esconde o valor **no terminal**, mas ele **continua visível no state file**.
- `ephemeral = true` (só em outputs de **módulos filhos**, **não** no root module) evita que o valor seja guardado no state.
- Ver outputs: `terraform output` (todos) ou `terraform output <nome>` (um).
- `terraform output -json` (formato JSON) e `terraform output -raw <nome>` (sem aspas, útil em scripts).

---

## 7. Local Values

Atribuem um **nome a uma expressão**, para a reutilizar várias vezes dentro de um módulo.

```hcl
locals {
  service_name = "forum"
  owner        = "community team"
}

locals {
  common_tags = {
    service = local.service_name
    owner   = local.owner
  }
}
```

- Declaras com **`locals`** (plural) e referencias com **`local.nome`** (singular).
- Podes ter vários blocos `locals`, com valores estáticos ou calculados.

---

## 8. Data Sources

Obtêm **informação definida fora do Terraform**.

```hcl
data "aws_ami" "web" {
  filter {
    name   = "state"
    values = ["available"]
  }
  filter {
    name   = "tag:Component"
    values = ["web"]
  }
  most_recent = true
}

resource "aws_instance" "web" {
  ami = data.aws_ami.web.id
}
```

- Indicas o **tipo** de recurso externo (`aws_ami`), usas `filter` para restringir a seleção e referencias com **`data.`**.

---

## 9. Referências a valores

| O que | Sintaxe |
|---|---|
| Recurso | `<tipo>.<nome>` |
| Variável de entrada | `var.nome` |
| Local value | `local.nome` |
| Módulo filho | `module.nome` |
| Data source | `data.<tipo>.<nome>` |
| Caminhos | `path.module`, `path.root`, `path.cwd` |
| Dentro de blocos | `count.index`, `each.key` / `each.value`, `self.atributo` |

---

## 10. Resource Meta-Arguments

Funcionam em **qualquer tipo de recurso** e alteram o seu comportamento:

| Meta-argumento | Função |
|---|---|
| `depends_on` | Define dependências explícitas |
| `count` | Cria várias instâncias segundo um número |
| `for_each` | Cria várias instâncias a partir de um map ou set de strings |
| `provider` | Escolhe um provider não default |
| `lifecycle` | Personaliza o ciclo de vida |
| `provisioner` / `connection` | Ações extra após a criação |

### depends_on

Define **a ordem em que os recursos são criados**.

| Tipo | Como surge | Quando usar |
|---|---|---|
| **Implícita** | Um recurso **referencia** outro (ex.: `aws_instance.x.id`) | Sempre que possível (forma preferida) |
| **Explícita** | `depends_on = [...]` | Só quando a dependência **não é visível** numa referência |

- O Terraform constrói um **grafo de dependências** e cria recursos independentes **em paralelo**.

Exemplo: a instância EC2 depende explicitamente da `aws_iam_role_policy`, para garantir que a policy é criada **antes** da instância:

```hcl
resource "aws_instance" "example" {
  ami           = "ami-a1b2c3d4"
  instance_type = "t2.micro"

  depends_on = [
    aws_iam_role_policy.example,
  ]
}
```

### count

Gere um **conjunto de objetos idênticos** (ex.: frota de VMs).

```hcl
resource "aws_instance" "server" {
  count         = 4          # cria 4 EC2
  ami           = "some ami"
  instance_type = "t2.micro"

  tags = {
    Name = "server ${count.index}"
  }
}
```

- `count.index` dá a posição atual e **começa em 0**.
- O valor tem de ser um **número inteiro** e **conhecido antes** da execução.

### for_each

Gere **objetos relacionados mas diferentes**, iterando sobre um **map** ou **set de strings**.

- `each.key`: chave atual
- `each.value`: valor atual

### lifecycle

Altera o que acontece ao recurso:

| Opção | Efeito |
|---|---|
| `create_before_destroy` | Cria o **novo** primeiro e só depois destrói o antigo (evita downtime) |
| `prevent_destroy` | O plan **dá erro** se o recurso fosse destruído |
| `ignore_changes` | Ignora alterações a certos atributos |

```hcl
resource "aws_instance" "web" {
  # ...
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [tags]
  }
}
```

- **Ordem de substituição:** por defeito é destruir e depois criar (`-/+`). Com `create_before_destroy` é criar e depois destruir (`+/-`).
- Atenção: com `create_before_destroy`, os dois recursos existem em simultâneo, por isso nomes únicos podem colidir.
- Se removeres o bloco do código, o `prevent_destroy` deixa de existir.
- O `lifecycle` também aceita `precondition` e `postcondition` (ver secção 11).

### Símbolos no plano de execução

| Símbolo | Significado |
|---|---|
| `+` | Criar |
| `-` | Destruir |
| `~` | Alterar (in-place) |
| `-/+` | Destruir e recriar |

---

## 11. Custom conditions (validar a configuração)

Quatro mecanismos:

| Mecanismo | Onde | Quando corre | Se falhar |
|---|---|---|---|
| **`validation`** | Dentro de `variable` | Ao avaliar o input | **Erro** (bloqueia) |
| **`precondition`** | `lifecycle` de resource, data ou output | **Antes** de criar/avaliar o objeto | **Erro** (bloqueia) |
| **`postcondition`** | `lifecycle` de resource ou data | **Depois** de criar/ler o objeto (usa `self`) | **Erro** (bloqueia) |
| **`check`** | Bloco `check` independente | No fim do plan/apply | **Só aviso** (não bloqueia) |

```hcl
variable "image_id" {
  type = string

  validation {
    condition     = length(var.image_id) > 4 && substr(var.image_id, 0, 4) == "ami-"
    error_message = "O image_id tem de começar por \"ami-\"."
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.web.id
  instance_type = "t2.micro"

  lifecycle {
    precondition {
      condition     = data.aws_ami.web.architecture == "x86_64"
      error_message = "A AMI tem de ser x86_64."
    }
    postcondition {
      condition     = self.public_dns != ""
      error_message = "A instância tem de ter DNS público."
    }
  }
}

check "app_health" {
  data "http" "app" {
    url = "https://example.com/health"
  }

  assert {
    condition     = data.http.app.status_code == 200
    error_message = "A aplicação não respondeu com 200."
  }
}
```

- Podes ter **vários blocos `validation`** na mesma variável.
- `condition` tem de dar `true` ou `false`; `error_message` é o texto mostrado quando falha.

---

## 12. Ephemeral values e write-only arguments

**Problema:** os segredos acabam no **state** e no **plan**. O `sensitive = true` só os **esconde no terminal**, mas **não impede que fiquem guardados**.

**Solução:** valores **efémeros**, que existem **só durante a run** e **não são guardados** no state nem no plan.

| Construção | O que é | Versão |
|---|---|---|
| **`ephemeral = true`** em variable / output de módulo filho | Variável ou output efémero | Terraform **1.10+** |
| **Bloco `ephemeral`** (ephemeral resource) | Recurso temporário que só existe durante a operação | Terraform **1.10+** |
| **Write-only argument** | Atributo de um **managed resource** enviado ao provider mas **não guardado** | Terraform **1.11+** |

```hcl
ephemeral "random_password" "db_password" {
  length = 16
}

resource "aws_db_instance" "db" {
  # ...
  password_wo         = ephemeral.random_password.db_password.result  # write-only
  password_wo_version = 1                                             # este fica no state
}
```

- Os argumentos write-only terminam em **`_wo`** e têm um **`_wo_version`** associado.
- Como o Terraform não guarda o valor, **não deteta mudanças**. Para o atualizar, **incrementas o `_wo_version`**.
- **São os providers** que definem quais argumentos são write-only e quais ephemeral resources existem (ver o Registry).
- Só podes usar valores efémeros em **contextos efémeros**: `locals`, variáveis e outputs de módulos filhos com `ephemeral`, write-only arguments, bloco `provider`, `provisioner` e `connection`.
- Como não ficam guardados, se quiseres preservar um valor gerado tens de o capturar noutro recurso.

| | `sensitive` | `ephemeral` / write-only |
|---|---|---|
| Esconde no CLI e na UI | Sim | Sim (se combinado com `sensitive`) |
| Fica no state e no plan | **Sim** | **Não** |

---

## Pontos-chave para decorar

1. **Provisioners** = má prática; não afetam o state. Preferir cloud-init/Packer.
2. `local-exec` corre **local**; `remote-exec` corre **no recurso**; `file` **copia ficheiros**.
3. `connection` define SSH/WinRM; `triggers` do null_resource fazem **re-executar**.
4. `terraform_data` = null_resource **sem provider**.
5. Providers: **Official** (HashiCorp), **Partner** (empresas terceiras, antes "Verified"), **Community** (sem garantias), **Archived** (sem manutenção).
6. `terraform init` descarrega providers e cria o **`.terraform.lock.hcl`** (commitar no VCS); `terraform providers` lista-os.
7. **Alias** permite várias configurações do mesmo provider (`provider = aws.west`).
8. Precedência de variáveis: **`-var` / `-var-file` vence tudo**.
9. Variáveis de ambiente: prefixo **`TF_VAR_`**.
10. Output `sensitive` esconde no terminal, **mas aparece no state**.
11. `locals` (declarar) vs `local.` (referenciar).
12. `count.index` começa em **0**; `for_each` usa **map ou set**.
13. `depends_on` = dependência **explícita** (só quando a implícita não chega).
14. `lifecycle`: `create_before_destroy`, `prevent_destroy`, `ignore_changes`.
15. Custom conditions: `validation` (inputs), `precondition` (antes), `postcondition` (depois), `check` (**só avisa**).
16. **Ephemeral** e **write-only** = não ficam no state; **sensitive** = fica no state, só esconde.
17. `ephemeral` em outputs: só em **módulos filhos**; write-only: Terraform **1.11+**.