# Aula 01 – IaC e porquê Terraform

## O que é IaC (Infrastructure as Code)

A IaC não torna a infraestrutura *melhor*. Torna-a **revisável, repetível e comparável (diffable)**.

- Ficheiros de definição legíveis por máquina que descrevem a infraestrutura, substituindo os cliques manuais na consola.
- Ficam guardados em **controlo de versões**.
- O código é a **fonte de verdade**: uma ferramenta lê-o e faz a realidade corresponder ao que lá está.
- Enquadramento da HashiCorp: definir a infraestrutura numa configuração legível por humanos e depois provisioná-la de forma programática.

> **No exame:** se a pergunta for "qual é o objetivo principal da IaC?", a resposta é **a criação e configuração programática de recursos**. Nada de pipelines nem de preço.

---

## Declarativo vs. imperativo

O Terraform é **declarativo**: descreves a infraestrutura que queres ter no fim e o Terraform descobre o que tem de criar, alterar ou destruir. Nunca escreves a ordem das operações; é o **grafo de dependências** que a deduz.

| | Declarativo | Imperativo |
|---|---|---|
| Descreves | **O que** deve existir | **Os passos**, por ordem |
| Sequência | A ferramenta calcula os passos | És tu que tratas da sequência |
| Voltar a executar | Converge sempre para o mesmo resultado | Pode repetir o trabalho |
| Exemplos | Terraform, CloudFormation, ARM | Scripts shell, chamadas diretas à API |

---

## Vantagens da IaC

- **Versionamento:** cada alteração tem autor, data, motivo e ponto de reversão (rollback).
- **Reutilização e partilha:** o mesmo ficheiro serve para várias regiões, ambientes ou equipas.
- **Revisão antes de aplicar:** uma alteração proposta pode ser lida e aprovada antes de tocar em seja o que for.
- **Menos configuration drift:** a abordagem declarativa mantém a realidade implantada próxima da intenção.
- **Automação e self-service.**

> **Versionamento, reutilização e partilha** é a resposta correta mais repetida neste domínio.

### O que a IaC NÃO garante (ratoeiras do exame)

- Não protege as tuas credenciais.
- Não torna os recursos mais baratos.
- Não impede modificações manuais.
- Não garante zero downtime.
- Não faz autoscaling.

---

## Infraestrutura mutável vs. imutável

| Mutável | Imutável |
|---|---|
| Altera-se o recurso **no local** | O recurso é **substituído** por completo |
| Aplicam-se patches e reconfigura-se servidores em produção | Constrói-se uma nova versão e troca-se pela antiga |
| Cada servidor acumula o seu próprio historial | Todas as instâncias são idênticas por construção |
| O drift cresce silenciosamente com o tempo | Não há historial acumulado que possa divergir |
| Depuração: "o que é diferente aqui?" | As atualizações ficam muito menos complexas |

> **Vantagem típica de exame da infraestrutura imutável:** a **redução da complexidade**. Não é a velocidade, nem a automação.

---

## Terraform vs. escrever chamadas à API manualmente

- O Terraform chama as **mesmas APIs dos providers** que chamarias tu. Nesse aspeto não há diferença.
- A diferença está na **configuração**: versionada, repetível e expressa como estado desejado.
- O Terraform **regista o que já construiu** (state), por isso uma segunda execução altera apenas o que for diferente.
- Não substitui a API do provider por um protocolo próprio. Os providers são camadas finas sobre essas APIs.

> Dizer que é "apenas um wrapper, logo não há diferença" é um **distrator**. O wrapper não é o ponto; o que importa são o **state** e o **grafo**.
>
> Os dois extremos estão errados: o Terraform não contorna a API, mas também não é só um wrapper.

---

## Workflow

1. **Write:** descrever a infraestrutura num ficheiro de configuração.
2. **Plan:** mostra exatamente o que seria criado, alterado ou destruído.
3. **Apply:** a alteração proposta é executada através da API do provider.
4. **Review:** a alteração fica em controlo de versões, com autor e motivo.

O passo **plan** é a razão pela qual a IaC permite revisão: transforma uma intenção num artefacto legível e verificável *antes* de acontecer seja o que for.

---

## Multi-cloud, cloud híbrida e agnóstico ao serviço

- **Um só workflow, vários providers:** o mesmo ciclo write → plan → apply em todo o lado.
- Mais de **3000 providers**: clouds, on-premises, SaaS, DNS, monitorização.
- **Cloud híbrida:** on-premises + cloud pública.
- **Agnóstico ao serviço:** basta existir um provider capaz de gerir aquilo de que precisas.
- **Não** torna a configuração portável: recursos AWS continuam a ser recursos AWS.

> A afirmação correta é **um workflow para vários providers**. "Uma configuração que corre em qualquer lado" não é verdade, nem nunca foi.

---

## Terraform ou ferramenta nativa do provider?

**Ferramenta nativa** (CloudFormation, ARM, Deployment Manager):
- Usa-se quando se trabalha com **um único fornecedor** e se pretende ficar lá.
- Tudo vive numa só cloud.
- O fornecedor guarda o state por ti.

**Terraform** (quando o ambiente atravessa fronteiras):
- Mais do que uma cloud, ou on-premises.
- SaaS e infraestrutura no mesmo workflow.
- Uma só linguagem e um só conjunto de competências para tudo.
- **O state file é teu.**

> Uma ferramenta nativa **não consegue provisionar na plataforma de outro fornecedor**. Esta limitação decide a maioria das perguntas deste tipo.

---

## Segredos e controlo de versões

- **Configuração** → no controlo de versões. Sempre a resposta certa.
- **Segredos** (palavras-passe, tokens, chaves) → **nunca** são submetidos junto com a configuração.
- O repositório guarda o historial: apagar um segredo mais tarde **não o remove**.
- Alternativas: gestor de segredos, variáveis de ambiente ou obtenção em tempo de execução.
- No Terraform, a resposta é o **Vault provider**, juntamente com **valores efémeros (ephemeral values)** (módulo 13).

> Duas perguntas quase idênticas, com respostas opostas:
> - Configuração em controlo de versões → **boa prática**.
> - Segredos em controlo de versões → **nunca**.

Visualizado 13min:56s