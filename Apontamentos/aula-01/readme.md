# Terraform Associate: Resumo da Aula 1 (simplificado)

> **Exame:** 57 questões, 70% para passar, 1 hora. Formatos: escolha múltipla, várias respostas e preencher espaços.

---

## 1. Infrastructure as Code (IaC)

**Configuração manual:** é fácil começar, mas é fácil errar, difícil gerir o estado e difícil de transferir para outros.

**IaC:** escreves código (scripts) que cria, atualiza ou destrói infraestrutura cloud automaticamente.

- Funciona como o "plano" da tua infraestrutura.
- É fácil de partilhar, versionar e inventariar.

### Declarativo vs Imperativo

| | **Declarativo** | **Imperativo** |
|---|---|---|
| Ideia | Descreves o resultado final | Descreves os passos / o sistema preenche o resto |
| Verbosidade | Mais verboso | Menos verboso |
| Risco de má configuração | Muito baixo | Maior |
| Linguagens | JSON, YAML, XML, HCL | Python, Ruby, JavaScript |
| Exemplos | Terraform, CloudFormation, Azure Blueprints | AWS CDK, Pulumi |

> **Para o exame:** o Terraform é **declarativo**, mas tem funcionalidades "imperativas" (loops, dynamic blocks, locals, estruturas complexas como maps e coleções).

> **Nota:** a definição do curso sobre "explícito vs implícito" é um pouco invulgar. Fora do curso, "declarativo" costuma significar "defines o estado desejado" e "imperativo" significa "defines os passos". Para o exame, o que interessa é: Terraform = declarativo.

---

## 2. Ciclo de vida da infraestrutura

Fases bem definidas: **planear → desenhar → construir → testar → entregar → manter → retirar**.

**Benefícios do IaC:**

- **Fiabilidade:** resultados idempotentes, consistentes, repetíveis e previsíveis.
- **Gestão:** alterações feitas via código, revistas e com mudanças mínimas.
- **Prudência:** evita perdas financeiras e de reputação.

### Idempotente vs não idempotente

Idempotente significa que **executar várias vezes dá sempre o mesmo resultado**.

- **Não idempotente:** deploy de 2 VMs. Precisas de mais CPU, fazes outro deploy de 2 VMs, e ficas com **4 VMs**.
- **Idempotente:** deploy de 2 VMs. Precisas de mais CPU, o sistema modifica (ou apaga e recria) as VMs, e ficas com **2 VMs**.

---

## 3. Provisioning vs Deployment vs Orchestration

- **Provisioning:** preparar servidores (sistema, dados, software) para ficarem operacionais. Exemplo: lançar e configurar um serviço cloud.
- **Deployment:** entregar a versão da tua aplicação num servidor já provisionado.
- **Orchestration:** coordenar vários sistemas ou serviços (ex.: containers e Kubernetes).

---

## 4. Configuration Drift

**O que é:** a infraestrutura provisionada sofre alterações inesperadas.

**Causas:**

- Alterações manuais
- Atacantes maliciosos
- Efeitos secundários de API, SDK ou CLI

**Risco:** perda ou violação de dados e serviços, interrupções e downtime.

| | Como |
|---|---|
| **Detetar** | Ferramentas de compliance (AWS Config, Azure Policy); no Terraform, o **state file** guarda o estado esperado |
| **Corrigir** | Ferramentas de compliance (remediação); comandos `terraform refresh` e `terraform plan`; correção manual (não recomendado) |
| **Prevenir** | Infraestrutura imutável; GitOps (controlo de versões do IaC) |

### Mutável vs Imutável

- **Mutável:** desenvolver → deploy → configurar (alteras o que já existe).
- **Imutável:** desenvolver → configurar → deploy (nunca reutilizas, **crias e destróis sempre**).

### Garantias da infraestrutura imutável

Ajuda em falhas de recursos cloud (ex.: EC2 que falha o status check), falhas de aplicação, tempo de deploy e piores cenários (eliminação, ataque, mudança de região).

### GitOps

Usas um **repositório Git** com um processo formal de revisão e aprovação das alterações ao código de infraestrutura. Quando o código é aceite, é **automaticamente** executado e implementado.

---

## 5. HashiCorp e Terraform

**HashiCorp** (ferramentas open source): Boundary, Consul, Nomad, Packer, **Terraform**, **Vault** (gestão de segredos).

**Terraform:**

- IaC open source e declarativo.
- Ficheiros escritos em **HCL** (HashiCorp Configuration Language).

**Terraform Cloud:**

- Portal web unificado.
- O plano gratuito permite até **5 utilizadores** por organização.

---

## 6. Ciclo de vida do Terraform

| Comando / Fase | Função |
|---|---|
| **code** | Escrever ou atualizar os ficheiros de configuração |
| `terraform init` | Inicializar o projeto |
| `terraform validate` | Verificar se tipos e valores são válidos |
| `terraform plan` | Prever o que vai mudar / gerar um plano de execução |
| `terraform apply` | Executar o plano |
| `terraform destroy` | Destruir a infraestrutura remota |

> **Dica:** a ordem habitual de uso é `init → validate → plan → apply`.

---

## 7. Change Automation

- **Change management:** abordagem para aplicar mudanças e resolver conflitos causados por elas.
- **Change automation:** gerir pedidos de mudança de forma automática, consistente e previsível, com controlos e políticas.

O Terraform faz isto através de **execution plans** e **resource graphs**.

- **Changeset:** conjunto de commits que representa as mudanças feitas num repositório.
- **Vantagem:** sabes exatamente **o que** o Terraform vai alterar e **em que ordem**, evitando erros humanos.

### Execution Plan

Revisão manual do que vai ser **adicionado, alterado ou destruído**.

### Visualizar o plano

`terraform graph` (precisa do **GraphViz** instalado).

### Resource Graph

Representação visual do grafo de dependências:

- **Resource node:** um único recurso.
- **Resource meta node:** grupo de recursos.
- **Provider configuration node:** representa o tempo necessário para configurar totalmente um provider.

---

## 8. Casos de uso do Terraform

- Suporta **milhares de providers**; como é open source, qualquer API pode ser usada para criar infraestrutura.
- **Aplicações multi-tier:** dividir apps complexas em módulos isolados.
- **Ambientes descartáveis.**
- **Resource schedulers.**
- **Multi-cloud:** é cloud agnostic, com um só ficheiro de configuração para vários providers.

---

## 9. Terraform Core vs Plugins

- **Core:** escrito em **Go** e comunica com os plugins via **RPC** (Remote Procedure Call).
- **Plugins:** expõem a implementação de um serviço ou provisioner específico (ex.: providers).

---


## Pontos-chave para decorar

1. Terraform = **declarativo**, open source, usa **HCL**.
2. **Idempotente** = mesmo resultado, não importa quantas vezes executes.
3. **Drift** = alterações inesperadas; deteta-se com o state file e `plan`.
4. **Imutável** = nunca alterar, sempre recriar.
5. Ciclo: `init → validate → plan → apply → destroy`.
6. Core (Go) fala com plugins via **RPC**.
7. `terraform graph` precisa de **GraphViz**.
8. Terraform Cloud grátis: **5 utilizadores**.
9. **Vault** = gestão de segredos.