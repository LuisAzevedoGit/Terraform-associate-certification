# Terraform Associate: Resumo da Aula 3 (simplificado)

---

## 1. Expressões Terraform

Servem para **referenciar ou calcular valores** dentro da configuração.

### Tipos e valores

O resultado de uma expressão é um **valor**, e todos os valores têm um **tipo**.

| Categoria | Tipos |
|---|---|
| **Primitivos** | `string`, `number`, `bool` |
| **Sem tipo** | `null` (valor por defeito de um atributo de um provider) |
| **Complexos / coleções** | `list`, `map`, `object`, `set`, `tuple` |

```hcl
regions = ["us-east-1a", "us-east-1b"]       # list
config  = { env = "prod", priority = 3 }     # map / object
```

### String templates

- **Interpolação:** `${ }` → `"hello ${var.name}"`
- **Diretivas:** `%{ }` → permitem `if` e `for` dentro de strings

```hcl
"Hello, %{ if var.name != "" }${var.name}%{ else }unnamed%{ endif }!"
```

### Operadores

Os mesmos das linguagens de programação:

| Tipo | Operadores |
|---|---|
| Aritméticos | `*` `/` `%` `+` `-` `-x` |
| Igualdade | `==` `!=` |
| Comparação | `<` `>` `<=` `>=` |
| Lógicos | `&&` `\|\|` `!` |

### Expressões condicionais

Formato: `condição ? valor_se_verdadeiro : valor_se_falso`

```hcl
var.a != "" ? var.a : "default-a"
```

Lê-se: *se `var.a` for diferente de vazio, usa `var.a`; senão usa `"default-a"`.*

### Expressões `for`

Aceitam como entrada: **list, set, tuple, map ou object**.

```hcl
[for s in var.list : upper(s)]
[for k, v in var.map : length(k) + length(v)]   # recebe chave e valor
```

O tipo de parênteses define o resultado:

| Parênteses | Devolve |
|---|---|
| `[ ]` | **tuple** |
| `{ }` | **object** |

**Ordenação implícita** na conversão:

- **Maps e objects:** ordenados pela chave (A-Z)
- **Sets de strings:** ordenados pela string (A-Z)
- **Todo o resto:** ordem arbitrária

### Splat expressions (`*`)

Forma **mais curta** de escrever uma expressão `for`.

```hcl
[for o in var.list : o.id]    # versão longa
var.list[*].id                # versão splat
```

### Dynamic blocks

Permitem **construir dinamicamente blocos aninhados repetíveis**, em vez de os escrever um a um à mão.

```hcl
variable "ingress_ports" {
  type    = list(number)
  default = [80, 443, 8080]
}

resource "aws_security_group" "example" {
  name = "example"

  dynamic "ingress" {                      # nome do bloco a gerar
    for_each = var.ingress_ports           # o que vai ser iterado
    content {                              # corpo de cada bloco gerado
      from_port   = ingress.value          # <nome_do_bloco>.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

Isto equivale a escrever **3 blocos `ingress`** (um por porta). Pontos a lembrar:

- `dynamic "<nome>"` define **que tipo de bloco** é gerado.
- `for_each` define **sobre o quê** se itera.
- `content` define **o conteúdo** de cada bloco.
- Dentro do `content`, o elemento atual acede-se por `<nome>.value` (e `<nome>.key`).

---

## 2. Version Constraints

O Terraform usa **versionamento semântico** (standard aberto): `MAJOR.MINOR.PATCH`

| Operador | Significado |
|---|---|
| `= 1.4.0` ou `1.4.0` | Exatamente esta versão |
| `!= 1.4.0` | Qualquer uma, exceto esta |
| `>= 1.2.0` | Esta ou mais recente |
| `<= 1.2.0` | Esta ou mais antiga |
| `> 1.2.0, < 2.0.0` | Intervalo |
| `~> 1.4.0` | **Pessimista:** só deixa aumentar o dígito mais à direita |

**Exemplos do `~>`:**

| Constraint | Aceita | Rejeita |
|---|---|---|
| `~> 1.4.0` | 1.4.1, 1.4.2 ... 1.4.9 | 1.5.0 |
| `~> 1.4` | 1.5, 1.6 ... 1.9 | 2.0 |

**Onde se usam as constraints:**

| Onde | Exemplo | Limita a versão de... |
|---|---|---|
| `required_version` (bloco `terraform`) | `required_version = ">= 1.12.0"` | O **Terraform CLI** |
| `required_providers` | `version = "~> 5.0"` | Um **provider** |
| Bloco `module` (só registry) | `version = "~> 1.0"` | Um **módulo** |

---

## 3. Terraform State

**State** = a condição dos recursos cloud **num determinado momento**.

- Ao provisionar, o Terraform cria o ficheiro **`terraform.tfstate`** (estrutura **JSON**).

### Comandos `terraform state`

| Comando | Função |
|---|---|
| `terraform state list` | Lista os recursos no state |
| `terraform state show` | Mostra os detalhes de um recurso |
| `terraform state mv` | Move um item dentro do state |
| `terraform state rm` | Remove um item do state |
| `terraform state pull` | Obtém o state remoto atual e escreve no stdout |
| `terraform state push` | Envia um state local para o remoto |
| `terraform state replace-provider` | Troca o provider de recursos no state |

- `terraform show` (sem `state`) mostra o **state completo** ou um **plano guardado** (`terraform show plano.tfplan`). Com `-json` dá formato máquina.

### `terraform state mv`

Serve para:

- **Renomear** recursos existentes
- Mover um recurso **para dentro de um módulo**
- Mover um módulo **para dentro de outro módulo**

### Refactoring por configuração: blocos `moved` e `removed`

A forma **recomendada** de refatorar é por código, porque fica **documentada e revista em PR**:

```hcl
# Renomear / mover (equivale a terraform state mv)
moved {
  from = aws_instance.web
  to   = aws_instance.app
}

# Remover do state SEM destruir o recurso real (equivale a terraform state rm)
removed {
  from = aws_instance.legacy

  lifecycle {
    destroy = false
  }
}
```

| Comando | Equivalente em configuração |
|---|---|
| `terraform state mv` | Bloco **`moved`** |
| `terraform state rm` | Bloco **`removed`** |

### Backups do state

- O Terraform guarda o state anterior em **`terraform.tfstate.backup`**.
- Os backups **não podem ser desativados**.

---

## 4. `terraform init`

Inicializa o projeto:

- **Descarrega** as dependências (providers e módulos)
- **Cria** o diretório `.terraform`
- **Cria** o ficheiro de lock de dependências (`.terraform.lock.hcl`)
- **Inicializa o backend**

| Opção | Função |
|---|---|
| `-upgrade` | Atualiza providers e módulos para a versão mais recente permitida |
| `-plugin-dir=PATH` | Força a ler plugins **apenas** desse diretório |
| `-lockfile=MODE` | Define o modo do lock file (ex.: `readonly`, que não pode combinar com `-upgrade`) |
| `-backend=false` | Salta a configuração do backend |
| `-backend-config=...` | Configuração parcial do backend |
| `-get=false` | Salta a instalação de módulos filhos |
| `-reconfigure` | Ignora a configuração anterior do backend, **sem migrar** o state |
| `-migrate-state` | **Migra** o state existente para o novo backend |

### `terraform get`

**Descarrega e atualiza módulos** do root module.

---

## 5. Comandos de apoio ao código

| Comando | Função |
|---|---|
| `terraform fmt` | Reescreve os ficheiros num **formato e estilo standard** |
| `terraform validate` | Valida **sintaxe** e consistência interna |
| `terraform console` | **Shell interativa** para testar expressões |

- **`fmt`:** corrige indentação. Com `-diff` mostra o que vai mudar; com `-check` só verifica (dá erro se houver ficheiros por formatar, útil em CI); com `-recursive` inclui subdiretórios.
- **`validate`:** não depende de variáveis nem do state. É executado **automaticamente** no `plan` e no `apply`.
- **`console`:** avalia expressões interativamente (ex.: `upper("abc")`).

---

## 6. `terraform plan`

Cria um **plano de execução**:

1. Lê o state atual e garante que está atualizado
2. Compara a configuração com o state anterior
3. Propõe um conjunto de ações que, se aplicadas, fazem os objetos remotos corresponder à configuração

> O `terraform plan` **não executa** as alterações.

| Speculative plan | Saved plan |
|---|---|
| `terraform plan` | `terraform plan -out=FILE` |
| Mostra o efeito do plano, **sem intenção de o aplicar** | Gera um **ficheiro de plano** que podes passar ao apply |
| | `terraform apply FILE` age como auto-approve |

### `-target`

- `terraform plan -target=aws_instance.web` (e o mesmo no `apply`) limita a operação a certos recursos (e às suas dependências).
- Usa-se **só em situações excecionais** (ex.: recuperar de um erro). Pode deixar o state inconsistente com a configuração.

---

## 7. `terraform apply`

**Executa** as ações propostas no plano.

| Automatic plan mode | Saved plan mode |
|---|---|
| `terraform apply` | `terraform apply FILE` |
| Faz plan + validate + apply | Aplica o plano guardado |
| Pede **aprovação manual**, ou usa `-auto-approve` | **Não** pede aprovação |

---

## 8. Managed Resource Drift

**Drift:** recursos num estado diferente do esperado.

| Situação | Solução |
|---|---|
| Recurso degradado/danificado | **Substituir** com a flag `-replace` |
| Recurso novo, fora do Terraform | **Importar** (bloco `import` ou comando `terraform import`) |
| Recurso alterado ou apagado fora do Terraform | **Atualizar o state** com `-refresh-only` |

### Replacing resources

```bash
terraform apply -replace="aws_instance.example[0]"
```

- Usa-se quando um recurso cloud está **degradado ou danificado** e queres voltar ao estado saudável.
- Só funciona **um recurso de cada vez**.
- Antes usava-se `terraform taint`, que está **deprecated**.

### Resource Addressing

String que identifica **zero ou mais instâncias** de recursos. Tem 2 partes:

```
[module path] [resource spec]

module.module_name[module index]
resource_type.resource_name[instance index]
```

| Caso | Endereço |
|---|---|
| Recurso simples | `aws_instance.web` |
| Com `count = 4` (3.ª instância) | `aws_instance.web[2]` |
| Com `for_each` | `aws_instance.web["prod"]` |
| Dentro de um módulo | `module.vpc.aws_subnet.main` |

Com `count = 4`, as instâncias vão de `[0]` a `[3]`.

### Importar recursos existentes

Há **duas formas**:

**1. Bloco `import` (recomendado, por configuração)**

```hcl
import {
  to = aws_instance.web
  id = "i-abc1234"
}
```

```bash
terraform plan -generate-config-out=generated.tf   # gera a configuração do recurso
terraform apply
```

**2. Comando `terraform import` (CLI)**

1. Cria um **bloco vazio** (placeholder) na configuração:
   ```hcl
   resource "aws_instance" "web" {
     # corpo vazio, preenches tu à mão
   }
   ```
2. Executa:
   ```bash
   terraform import <RESOURCE_ADDRESS> <ID>
   terraform import aws_instance.example i-abc1234
   ```

| | `terraform import` (CLI) | Bloco `import` |
|---|---|---|
| Quando corre | Imediatamente | Durante `plan` / `apply` |
| Recursos por execução | **Um** | **Vários** |
| Gera a configuração | Não (escreves tu) | **Sim**, com `-generate-config-out` |
| Revisão em PR | Não | **Sim** |

- **Nem todos os recursos** são importáveis.

### Refresh e Refresh-Only

Lê as definições atuais de todos os objetos remotos e **atualiza o state** para corresponder.

- **Não** altera os objetos reais, **só o state**.
- `terraform refresh` está **deprecated**. Usa:

```bash
terraform apply -refresh-only -auto-approve
```

- Para **só ver** o que mudou, sem alterar o state: `terraform plan -refresh-only`.

### Cenário importante

Criaste uma VM na AWS com Terraform. Um colega termina-a **pela consola AWS**.

| | `terraform apply` | `terraform apply -refresh-only` |
|---|---|---|
| O que deteta | A VM desapareceu | A VM desapareceu |
| Interpretação | É um problema: a config diz que deve existir | A remoção é **intencional** |
| Ação | **Cria uma nova VM** | Propõe **apagar a VM do state** |
| Resultado | State correto | State "errado" (reflete a infra) |
| Direção | Altera a **infra** para igualar o state | Altera o **state** para igualar a infra |

---

## 9. Troubleshooting

**4 tipos de erros:**

| Tipo | Causa | Como resolver |
|---|---|---|
| **Language** | Erro de sintaxe nos ficheiros | `terraform fmt`, `terraform validate`, `terraform version` |
| **State** | O state dos recursos mudou do esperado | `terraform apply -refresh-only`, `terraform apply`, flag `-replace` |
| **Core** | Bug na biblioteca core | `TF_LOG` + abrir issue no GitHub |
| **Provider** | A API do provider mudou ou falha em casos limite | `TF_LOG` + abrir issue no GitHub |

### Debugging com logs

Ativa-se com a variável de ambiente **`TF_LOG`**, com os níveis:

`TRACE` → `DEBUG` → `INFO` → `WARN` → `ERROR` (e `JSON` para formato JSON)

| Variável | Função |
|---|---|
| `TF_LOG` | Ativa os logs (todos) |
| `TF_LOG_CORE` | Logs só do **core** |
| `TF_LOG_PROVIDER` | Logs só dos **providers** |
| `TF_LOG_PATH` | **Ficheiro** onde guardar os logs |

---

## Pontos-chave para decorar

1. Tipos primitivos: **string, number, bool**. `null` = sem valor.
2. Interpolação `${}`; diretivas `%{}`.
3. Condicional: `cond ? verdadeiro : falso`.
4. `for` com `[]` devolve **tuple**; com `{}` devolve **object**.
5. Splat `[*]` = atalho para `for`.
6. **Dynamic block** = gera blocos repetidos (`dynamic` + `for_each` + `content`).
7. `~> 1.4.0` aceita 1.4.x; `~> 1.4` aceita 1.x (≥ 1.4). `required_version` limita o **Terraform**.
8. State = ficheiro **JSON** `terraform.tfstate`; backup em `.tfstate.backup` (não desativável).
9. `terraform init` cria `.terraform` e o **lock file**.
10. `fmt` formata (`-check` só verifica), `validate` valida sintaxe, `console` testa expressões.
11. `plan` **não aplica nada**; `plan -out=FILE` gera plano guardado.
12. `apply FILE` não pede aprovação; `apply` normal sim (ou `-auto-approve`).
13. Drift: `-replace` (substituir), `import` (adicionar), `-refresh-only` (atualizar state).
14. `terraform taint` e `terraform refresh` estão **deprecated**.
15. `-refresh-only` altera **só o state**; `apply` normal altera a **infra**.
16. Logs: `TF_LOG`, `TF_LOG_CORE`, `TF_LOG_PROVIDER`, `TF_LOG_PATH`.
17. Refactoring por código: **`moved`** (renomear/mover), **`removed`** (tirar do state sem destruir), **`import`** (importar, vários de uma vez).
18. `-target` só em casos excecionais.