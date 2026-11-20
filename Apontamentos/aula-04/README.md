# Terraform Associate: Resumo da Aula 4 (simplificado)

---

## 1. Módulos

### Encontrar módulos

- Procuram-se no **Terraform Registry**.
- Nas pesquisas só aparecem **módulos verificados** (*verified*).

### Origens de um módulo (`source`)

| Origem | Exemplo de `source` | Aceita `version`? |
|---|---|---|
| Registry público | `"hashicorp/consul/aws"` | **Sim** |
| Registry privado | `"app.terraform.io/org/vpc/aws"` | **Sim** |
| Caminho local | `"./modules/vpc"` | **Não** |
| GitHub / Git | `"github.com/org/repo"` ou `"git::https://...?ref=v1.2.0"` | **Não** (usa `?ref=`) |
| HTTP URL, S3, GCS | `"https://..."`, `"s3::..."` | **Não** |

### Usar módulos

**Módulos públicos** (integrados diretamente no Terraform):

```hcl
module "consul" {
  source  = "hashicorp/consul/aws"
  version = "0.1.0"
}
```

**Módulos privados** (private registry):

```hcl
module "vpc" {
  source  = "app.terraform.io/example_corp/vpc/aws"
  version = "0.9.3"
}
```

- `terraform init` **descarrega e guarda em cache** os módulos.
- Para aceder a módulos privados, tens de te **autenticar no HCP Terraform** com `terraform login`.

### Versões de módulos

- O argumento **`version` só existe para módulos de registry** (público ou privado) e aceita as mesmas constraints (`~>`, `>=`...).
- Para módulos Git, fixas a versão com **`?ref=<tag|branch|commit>`**.
- Se mudares a versão, corre `terraform init -upgrade`.

### Variáveis e outputs de um módulo (scope)

- Cada módulo tem **o seu próprio âmbito**: **não vê** as variáveis do módulo pai.
- O pai passa valores através de **argumentos** do bloco `module` (que correspondem às `variable` do módulo).
- O módulo devolve dados através de **outputs**, acedidos no pai por **`module.<nome>.<output>`**.

```hcl
# No módulo pai
module "vpc" {
  source = "./modules/vpc"
  cidr   = var.vpc_cidr          # argumento = variável do módulo
}

output "vpc_id" {
  value = module.vpc.vpc_id      # output do módulo filho
}
```

### Providers dentro de módulos

- Um módulo filho **herda** a configuração default do provider do pai.
- Para passar uma configuração com alias, usa **`providers`** no bloco `module`:

```hcl
module "vpc" {
  source    = "./modules/vpc"
  providers = {
    aws = aws.west
  }
}
```

### Publicar módulos

- **Qualquer pessoa** pode publicar. Os módulos públicos são geridos através de **repositórios Git públicos no GitHub**.
- O registry dá suporte a: versionamento, geração de documentação, navegação de versões, exemplos...
- O repositório tem de seguir o formato de nome: **`terraform-<provider>-<name>`**

### Módulos verificados (Verified)

- **Revistos pela HashiCorp** e mantidos ativamente.
- Têm um **badge** de verificado.

### Estrutura standard de um módulo

```
.
├── README.md
├── main.tf
├── variables.tf
├── outputs.tf
├── LICENSE
├── modules/
│   ├── nestedA/
│   │   ├── README.md
│   │   ├── variables.tf
│   │   ├── main.tf
│   │   └── outputs.tf
│   └── nestedB/
└── examples/
    ├── exampleA/
    │   └── main.tf
    └── exampleB/
```

**Root Module** (ponto de entrada principal). Ficheiros no diretório raiz:

| Ficheiro | Função |
|---|---|
| `main.tf` | Ficheiro de entrada do módulo |
| `variables.tf` | Variáveis que podem ser passadas ao módulo |
| `outputs.tf` | Valores de saída |
| `README` | Descreve como o módulo funciona |
| `LICENSE` | Licença sob a qual o módulo está disponível |
| `/examples` | exemplos de uso do modulo |

**Nested modules** (opcionais, ficam na pasta `modules/`):

- Submódulo **com README** → considerado **utilizável por utilizadores externos**.
- Submódulo **sem README** → considerado de **uso interno apenas**.
- A slide também refere para evitar **caminhos relativos** ao referenciar blocos `module` (o texto está parcialmente tapado pela legenda).

---

## 2. Team Workflows

Fluxo base: **write → plan → apply**

À medida que a equipa cresce, o fluxo evolui:

1. Praticante individual
2. Equipas com **OSS** (open source)
3. Equipas com **HCP Terraform**

### Praticante individual

| Fase | O que acontece |
|---|---|
| **Write** | Escreve a configuração no editor; guarda o código num VCS (ex.: GitHub); usa `plan` e `validate` para achar erros de sintaxe; ciclo de feedback rápido |
| **Plan** | Faz commit no repositório local; talvez só uma branch (ex.: `main`) |
| **Apply** | Corre `terraform apply` e revê o plano; aprova e espera o provisionamento; depois faz push dos commits para o remoto |

### Equipa (OSS)

| Fase | O que acontece |
|---|---|
| **Write** | Cada membro escreve a configuração numa **branch** própria (evita conflitos e permite resolvê-los no merge para `main`); `plan` serve de feedback rápido em equipas pequenas; em equipas grandes, as **credenciais sensíveis** tornam-se um problema, por isso usa-se **CI/CD** para as abstrair |
| **Plan** | Quando a branch está pronta, abre-se um **pull request** e o plano de execução é gerado e mostrado no PR para revisão |
| **Apply** | O PR é **aprovado e feito merge**, o que aciona um **build server** que corre `terraform apply` |

**Desafios para a equipa DevOps:**

- Montar e manter o CI/CD
- Decidir onde guardar o **state file**
- Controlos de acesso limitados
- Guardar e injetar **segredos** de forma segura
- Gerir **múltiplos ambientes**

### Equipa com HCP Terraform

| Fase | O que acontece |
|---|---|
| **Write** | Usa o HCP Terraform como **remote backend**; as variáveis ficam guardadas no HCP Terraform; integra com o VCS para montar CI/CD rapidamente; escreve-se código numa branch como habitual |
| **Plan** | Ao criar um PR, o HCP Terraform gera um **speculative plan** para revisão no VCS; o membro pode também rever e comentar o plano no HCP Terraform |
| **Apply** | Após o merge do PR, o ambiente de execução do HCP Terraform corre `terraform apply`; um membro da equipa pode **confirmar** e aplicar |

> O HCP Terraform simplifica muito o CI/CD, guarda e protege credenciais sensíveis e facilita **auditar o histórico** de várias execuções.

---

## 3. Backends

Definem **onde e como** as operações são executadas e **onde os snapshots do state são guardados**.

Para o exame, foca-te em **três opções**:

| Opção | Onde fica o state | Notas |
|---|---|---|
| **Local backend** | Disco local (`terraform.tfstate`) | O default; backup em `terraform.tfstate.backup` |
| **Remote backend** (bloco `backend`) | Serviço remoto (ex.: S3) | State **partilhado** e, se o backend suportar, com **locking** |
| **HCP Terraform** (bloco `cloud`) | No workspace do HCP Terraform | Também executa as operações remotamente |

> **Classificação do curso (histórica):** *standard* = só guarda o state (ex.: S3); *enhanced* = guarda o state e executa operações (local, remote). A documentação atual já não usa esta divisão. Mantém só como contexto.

### Backend local

```hcl
terraform {
  backend "local" {
    path = "relative/path/to/terraform.tfstate"
  }
}
```

Especificas o backend `local` e podes mudar o `path`.

### Referenciar o state de outra stack

```hcl
data "terraform_remote_state" "networking" {
  backend = "local"
  config = {
    path = "${path.module}/networking/terraform.tfstate"
  }
}
```

Permite **ler os outputs** de outro state file (cross-reference entre stacks).

### Backend remoto do HCP Terraform (`remote`, legado)

É o **HCP Terraform** ou o **Terraform Enterprise**.

- Quando fazes `terraform apply` via CLI, o **ambiente de execução do HCP Terraform** é que executa a operação.
- As **credenciais do provider** têm de estar configuradas como **variáveis de ambiente no HCP Terraform**.
- Tens de definir **workspaces** no HCP Terraform:
  - **Um só:** por **name**
  - **Vários:** por **prefix**

### Cloud backend (bloco `cloud`)

```hcl
terraform {
  cloud {
    organization = "company"
    hostname     = "app.terraform.io"
    workspaces {
      tags = ["app"]
    }
  }
}
```

> Ainda podes usar `backend "remote"` com o HCP Terraform, mas é **recomendado usar o bloco `cloud`**.

- Para autenticar na CLI: `terraform login`.
- Ao mudar de backend (ex.: de local para HCP Terraform): `terraform init -migrate-state` copia o state existente; `terraform init -reconfigure` ignora a configuração anterior **sem migrar**.

### Inicialização do backend (`-backend-config`)

Separas a configuração do backend num ficheiro à parte.

**main.tf:**
```hcl
terraform {
  backend "remote" {}
}
```

**backend.hcl:**
```hcl
workspaces { name = "workspace" }
organization = "company"
hostname     = "app.terraform.io"
```

**Comando:**
```bash
terraform init -backend-config=backend.hcl
```

---

## 4. Terraform Remote State

Data source `terraform_remote_state`: obtém os **output values do root module** de outra configuração Terraform (via remote backend ou local backend).

```hcl
data "terraform_remote_state" "network" {
  backend = "remote"
  config = {
    organization = "company"
    workspaces = {
      name = "network"
    }
  }
}

# uso: data.terraform_remote_state.network.outputs.vpc_id
```

- Só ficam expostos os **outputs de nível root**.
- **Dados de recursos** e outputs de módulos aninhados **não** são acessíveis.
- Para expor o output de um módulo aninhado, tens de configurar explicitamente um **passthrough** no root module.
- No HCP Terraform, o workspace de origem tem de **permitir partilhar o seu state** com o workspace que o lê.

**Alternativa:** usar um **data source** e referenciar os recursos diretamente.

---

## 5. State Locking

**Bloqueia** o state para que operações que possam escrever nele **não o corrompam**.

| Opção | Notas |
|---|---|
| `-lock=false` | **Desativa** o lock (**não recomendado**) |
| `terraform force-unlock <LOCK_ID>` | **Força** o desbloqueio. Se outra pessoa tiver o lock, pode causar **vários escritores**. Requer o **lock ID** único |

---

## 6. Proteger dados sensíveis

O state pode conter **dados sensíveis** (ex.: credenciais AWS de longa duração) e é um possível **vetor de ataque**.

| Onde está o state | Boas práticas |
|---|---|
| **Local** | **Não partilhar**, **não fazer commit** |
| **Remote (HCP Terraform)** | Fica **em memória** (não persiste em disco); **encriptado** em trânsito e em repouso; tem **audit logging** |
| **Remote (terceiros)** | Rever as capacidades do backend; alguns não são seguros por defeito. Ex.: no **S3** tens de garantir que **encriptação** e **versionamento** estão ativos |

**Evitar que o segredo chegue ao state:**

- `sensitive = true` **não chega**: só esconde no terminal, o valor **continua no state**.
- Usa **valores efémeros** e **write-only arguments**, que **não são guardados** no state nem no plan (ver Aula 2, secção 12).
- Usa um gestor de segredos (ex.: **Vault**) em vez de colocar segredos no código.

### `.terraformignore`

Ficheiro na **raiz** do diretório de configuração que define **caminhos a ignorar no upload**.

- Funciona como o `.gitignore`.
- Diferença: **não podes ter vários** em subdiretórios. Só é lido o da **raiz**.

---

## 7. Recursos e tipos complexos

### Resources

Representam **objetos de infraestrutura**: VMs, bases de dados, componentes de rede, storage...

- O **resource type** determina o tipo de objeto (ex.: `aws_instance`). Consulta os nomes no Terraform Registry.

### Tipos complexos

Agrupam **vários valores num só**.

| Categoria | Tipos | Descrição |
|---|---|---|
| **Collection types** | `list`, `map`, `set` | Agrupam vários valores **do mesmo tipo** |
| **Structural types** | `object`, `tuple` | Agrupam vários valores de **tipos diferentes** |

Os structural types precisam de um **schema** que indique os tipos permitidos:

```hcl
variable "structural_types" {
  type = object({
    a = string
    b = number
  })
}
```

---

## Pontos-chave para decorar

1. Módulos: procurados no **Registry**; só **verified** aparecem na pesquisa.
2. `terraform init` **descarrega** os módulos; módulos privados exigem `terraform login`.
3. Nome do repo de módulo: **`terraform-<provider>-<name>`**.
4. Verified = **revisto pela HashiCorp**, com badge.
5. Estrutura: `main.tf`, `variables.tf`, `outputs.tf`, README, LICENSE; nested em `modules/`.
6. Submódulo **com README** = uso externo; **sem README** = interno.
7. `version` só em módulos de **registry**; em Git usa `?ref=`.
8. Um módulo **não vê** as variáveis do pai: recebe **argumentos** e devolve **outputs** (`module.nome.output`).
9. Fluxo: **write → plan → apply**. PR gera **speculative plan**; merge aciona o apply.
10. Backends para o exame: **local**, **remote** (bloco `backend`) e **HCP Terraform** (bloco `cloud`, o recomendado).
11. `backend "remote"` é o legado; **bloco `cloud`** usa **tags** (vários workspaces) ou **name** (um).
12. `terraform init -backend-config=ficheiro.hcl`; mudar de backend: `-migrate-state` (copia) ou `-reconfigure` (não copia).
13. `terraform_remote_state` só expõe **outputs do root module**.
14. `-lock=false` desativa lock (mau); `force-unlock` precisa de **lock ID**.
15. State local: **nunca partilhar nem commitar**; S3: ativar **encriptação e versionamento**.
16. `sensitive` fica no state; **ephemeral / write-only** não ficam.
17. `.terraformignore`: só o da **raiz** é lido.
18. Collection: **list, map, set**; Structural: **object, tuple**.