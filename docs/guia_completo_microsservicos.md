> Revisão de ambiente em 20/10/2026: exemplos genéricos usam Node.js 22. O laboratório StreamFlow usa `node:22-bookworm-slim` (Debian); comandos `apk` dos exemplos Alpine não se aplicam aos seus contêineres. Consulte `SETUP.md` e `DEBUG.md` para executar a atividade.

# Guia Completo: Arquitetura de Microsserviços e Containerização com Docker

**Disciplina:** Serviços Web (PF_CC.44)  
**Curso:** Ciência da Computação — IFSul, Campus Passo Fundo  
**Professor:** Élder F. F. Bernardi  
**Unidade:** 2 — Tecnologias de Serviços Web  
**Contexto:** Material de apoio às Aulas 8 e 9 (Microsserviços + Docker) e ao Trabalho Final

---

## Sumário

1. [Introdução](#1-introdução)
2. [De Monolítico a Microsserviços — A Evolução Arquitetural](#2-de-monolítico-a-microsserviços--a-evolução-arquitetural)
3. [Princípios Fundamentais de Microsserviços](#3-princípios-fundamentais-de-microsserviços)
4. [Padrões de Decomposição — Onde Cortar?](#4-padrões-de-decomposição--onde-cortar)
5. [Comunicação entre Serviços](#5-comunicação-entre-serviços)
6. [API Gateway — A Porta de Entrada Unificada](#6-api-gateway--a-porta-de-entrada-unificada)
7. [Autenticação em Microsserviços — A Ponte com JWT](#7-autenticação-em-microsserviços--a-ponte-com-jwt)
8. [Docker — Fundamentos de Containerização](#8-docker--fundamentos-de-containerização)
9. [Dockerfile — Construindo Imagens](#9-dockerfile--construindo-imagens)
10. [Docker Compose — Orquestrando Múltiplos Serviços](#10-docker-compose--orquestrando-múltiplos-serviços)
11. [Implementação Prática — Decompondo uma API Monolítica (Express)](#11-implementação-prática--decompondo-uma-api-monolítica-express)
12. [Implementação com Fastify — Microsserviços com Persistência](#12-implementação-com-fastify--microsserviços-com-persistência)
13. [Observabilidade e Monitoramento](#13-observabilidade-e-monitoramento)
14. [Padrões de Resiliência](#14-padrões-de-resiliência)
15. [Erros Comuns e Antipadrões](#15-erros-comuns-e-antipadrões)
16. [Referências](#16-referências)

---

## 1. Introdução

A arquitetura de **microsserviços** representa uma das transformações mais significativas na forma como aplicações são projetadas, desenvolvidas e operadas desde a adoção de sistemas de nuvem. Em vez de construir um sistema como um bloco único e monolítico, a abordagem de microsserviços propõe a decomposição da aplicação em **serviços pequenos, independentes e especializados**, cada um responsável por uma capacidade de negócio específica.

Essa mudança não é apenas técnica — ela reflete a necessidade de organizações modernas de entregar software com **velocidade, escalabilidade e resiliência** em ambientes de nuvem e operações distribuídas.

### Onde este conteúdo se encaixa na disciplina

Na disciplina de Serviços Web, você já estudou os fundamentos da arquitetura REST e implementou APIs com Node.js e Express.js. Também aprendeu a proteger essas APIs com autenticação JWT. A arquitetura de microsserviços representa o **próximo passo natural** nessa jornada: quando uma API monolítica cresce em complexidade e em requistos de escalabilidade, quais estratégias arquiteturais permitem que ela continue evoluindo de forma sustentável e quais os principais desafios para que esta evolução permaneça estável?

Um dos maiores risco que temos ao adotar uma arquitetura de microsserviços, é que ela pode se tornar tão complexa que se torna mais difícil de gerenciar e manter do que um monolito. Este "sweet spot" entre realmente escalar de um monolito para microsserviços, sem degradar custos, manutenção e escalabilidade é o grande desafio que teremos que aprender a dominar. O objetivo deste estudo e trabalho seguinte é escavarmos os termos, ambientes, conceitos e arquiteturas que orbitam microsserviços e nos tornarmos aptos a tomar decisões conscientes e inteligentes para realizar a leitura de cenários e fazer as escolhas mais racioanais.

### Objetivos deste guia

Ao final da leitura, você será capaz de:

- Explicar as diferenças entre arquitetura monolítica e microsserviços
- Identificar quando a adoção de microsserviços é (e quando não é) apropriada
- Aplicar padrões de decomposição para dividir uma aplicação em serviços independentes
- Compreender os modelos de comunicação entre serviços (síncrono e assíncrono)
- Construir imagens Docker e orquestrar múltiplos containers com Docker Compose
- Decompor uma API REST monolítica (Node.js + Express) em microsserviços containerizados
- Ter um vocabulário mais rico para poder analisar sistemas existentes e diagnosticar problemas e soluções.

## 2. De Monolítico a Microsserviços — A Evolução Arquitetural

### 2.1 O que é uma aplicação monolítica?

Uma aplicação monolítica é aquela em que **todo o código** — desde a interface de autenticação até as regras de negócio e o acesso ao banco de dados — reside em uma **única base de código** e é implantado como **uma única unidade**.

A API que você construiu no Trabalho I é um exemplo clássico de monolito: rotas de autenticação, rotas de tarefas, middlewares, serviços e DAOs — tudo vive no mesmo projeto, compartilhando o mesmo processo Node.js e o mesmo banco de dados.

```
┌─────────────────────────────────────────────────┐
│              APLICAÇÃO MONOLÍTICA                │
│                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │  Auth    │ │  Tarefas │ │    Usuários      │ │
│  │  Routes  │ │  Routes  │ │    Routes        │ │
│  └────┬─────┘ └────┬─────┘ └───────┬──────────┘ │
│       │             │               │            │
│  ┌────┴─────────────┴───────────────┴──────────┐ │
│  │          Controllers + Services             │ │
│  └─────────────────────┬───────────────────────┘ │
│                        │                         │
│  ┌─────────────────────┴───────────────────────┐ │
│  │              Banco de Dados                 │ │
│  └─────────────────────────────────────────────┘ │
│                                                  │
│          Um único processo · Uma porta           │
└─────────────────────────────────────────────────┘
```

**Vantagens do monolito** (sim, elas existem e são importantes):

| Vantagem                | Detalhamento                                                     |
| ----------------------- | ---------------------------------------------------------------- |
| Simplicidade inicial    | Um único projeto, uma implantação, fácil de entender             |
| Desenvolvimento rápido  | Sem complexidade de rede entre componentes                       |
| Debugging direto        | Stack trace completa em um único processo                        |
| Transações ACID simples | Tudo no mesmo banco, sem coordenação distribuída                 |
| Implantação trivial     | Um único artefato para fazer deploy                              |
| Logging natural         | Sistemas de logging comuns costumas ser suficientes para debugar |

### 2.2 Quando o monolito se torna um problema

À medida que a aplicação cresce, surgem dificuldades concretas:

**Acoplamento involuntário:** Uma alteração no módulo de pagamentos pode quebrar o módulo de relatórios, porque ambos compartilham modelos de dados e dependências internas. Essa interdependência silenciosa torna cada mudança um risco.

**Escalabilidade limitada:** Se o módulo de busca recebe 100x mais requisições que o módulo de administração, não é possível escalar apenas a busca — todo o monolito precisa ser replicado, desperdiçando recursos computacionais. Temos um problema de escalabilidade vertical, que acaba custando mais, pois também obriga a escala vertical.

**Deploy com risco total:** Qualquer mudança, por menor que seja, exige reimplantar a aplicação inteira. Um bug introduzido em um módulo periférico pode derrubar o sistema completo.

**Ciclos de entrega longos:** Em equipes maiores, múltiplos desenvolvedores editam a mesma base de código. Conflitos de merge, testes que quebram em cascata e coordenação excessiva atrasam entregas.

### 2.3 O que são microsserviços?

A arquitetura de microsserviços é um estilo arquitetural que estrutura uma aplicação como uma **coleção de serviços independentes**, onde cada serviço:

- Implementa uma **capacidade de negócio** específica e bem delimitada
- É **desenvolvido, implantado e escalado independentemente**
- Possui seu **próprio armazenamento de dados** (banco isolado)
- Comunica-se com outros serviços por meio de **interfaces bem definidas** (APIs REST, mensagens, eventos)
- Pode ser escrito em **linguagens e tecnologias diferentes** (poliglotismo)

```
┌────────────┐     ┌────────────┐     ┌────────────┐
│  Serviço   │     │  Serviço   │     │  Serviço   │
│    Auth    │     │  Tarefas   │     │  Usuários  │
│            │     │            │     │            │
│ ┌────────┐ │     │ ┌────────┐ │     │ ┌────────┐ │
│ │ BD own │ │     │ │ BD own │ │     │ │ BD own │ │
│ └────────┘ │     │ └────────┘ │     │ └────────┘ │
│  :3001     │     │  :3002     │     │  :3003     │
└─────┬──────┘     └─────┬──────┘     └──────┬─────┘
      │                  │                    │
      └──────────────────┼────────────────────┘
                         │
                  ┌──────┴──────┐
                  │ API Gateway │
                  │   :8080     │
                  └──────┬──────┘
                         │
                    ┌────┴────┐
                    │ Cliente │
                    └─────────┘
```

### 2.4 Comparação direta

| Aspecto                      | Monolítico                               | Microsserviços                                       |
| ---------------------------- | ---------------------------------------- | ---------------------------------------------------- |
| **Estrutura**                | Uma base de código unificada             | Múltiplos serviços independentes                     |
| **Implantação**              | Tudo junto, uma unidade                  | Cada serviço separadamente                           |
| **Escalabilidade**           | Vertical (máquina maior)                 | Horizontal (mais instâncias por serviço)             |
| **Banco de dados**           | Compartilhado                            | Um por serviço (isolamento)                          |
| **Tecnologia**               | Uma stack para tudo                      | Cada serviço pode usar a sua                         |
| **Tolerância a falhas**      | Falha derruba tudo                       | Falha isolada em um serviço                          |
| **Complexidade operacional** | Baixa                                    | Alta (rede, observabilidade, deploy)                 |
| **Indicado para**            | MVPs, equipes pequenas, domínios simples | Sistemas grandes, equipes múltiplas, escala variável |

### 2.5 Quando NÃO adotar microsserviços

É fundamental ter consciência de que microsserviços **não são a resposta para todos os problemas**. A adoção prematura dessa arquitetura é um dos erros mais frequentes na indústria.

Microsserviços **provavelmente não são a melhor escolha** quando:

- A equipe é pequena (menos de 5–8 desenvolvedores)
- O domínio de negócio ainda não é bem compreendido
- O sistema tem menos de 10 módulos com fronteiras claras
- A organização não tem maturidade em DevOps, CI/CD e monitoramento
- O projeto está em fase de MVP (Produto Mínimo Viável)

> **Martin Fowler**, uma das principais referências em arquitetura de software, cunhou o termo **"Monolith First"** (monolito primeiro): comece com um monolito bem estruturado e só decomponha em microsserviços quando a complexidade justificar o custo operacional adicional.

---

## 3. Princípios Fundamentais de Microsserviços

A arquitetura de microsserviços não é definida apenas pela separação física de código em repositórios distintos. Ela é sustentada por um conjunto de **princípios de design** que, quando aplicados de forma coerente, resultam em sistemas genuinamente modulares e resilientes.

### 3.1 Responsabilidade Única de Negócio

Cada microsserviço deve encapsular uma **capacidade de negócio** completa e coesa. Não se trata de dividir por camada técnica (um serviço para controllers, outro para DAOs), mas sim por **domínio funcional**.

```
  ✗ Divisão ERRADA (por camada técnica)    ✓ Divisão CORRETA (por domínio)

  ┌──────────────┐                         ┌──────────────┐ ┌──────────────┐
  │  Serviço de  │                         │  Serviço de  │ │  Serviço de  │
  │  Controllers │                         │  Pedidos     │ │  Pagamentos  │
  ├──────────────┤                         │              │ │              │
  │  Serviço de  │                         │ Controller   │ │ Controller   │
  │  Services    │                         │ Service      │ │ Service      │
  ├──────────────┤                         │ DAO + BD     │ │ DAO + BD     │
  │  Serviço de  │                         └──────────────┘ └──────────────┘
  │  DAOs        │
  └──────────────┘
```

### 3.2 Autonomia e Independência de Deploy

Um microsserviço deve poder ser **compilado, testado e implantado** sem depender de outros serviços. Isso significa que:

- Cada serviço tem seu próprio **repositório** (ou diretório independente em um monorepo)
- Cada serviço tem seu próprio **pipeline de CI/CD**
- A alteração em um serviço **não exige reimplantação** de outros

### 3.3 Banco de Dados por Serviço (Database per Service)

Este é um dos princípios mais importantes — e mais difíceis de aceitar para quem vem do mundo monolítico. Cada microsserviço **deve possuir e gerenciar seu próprio armazenamento de dados**.

| Princípio               | Implicação                                                            |
| ----------------------- | --------------------------------------------------------------------- |
| Cada serviço tem seu BD | Sem joins entre serviços — dados são obtidos via API                  |
| Tecnologia livre        | `user-service` pode usar PostgreSQL, `logs-service` pode usar MongoDB |
| Encapsulamento          | Nenhum serviço acessa diretamente o banco de outro                    |

```
  ✗ ERRADO: Banco compartilhado           ✓ CORRETO: Banco por serviço

  ┌───────┐  ┌───────┐  ┌───────┐        ┌───────┐  ┌───────┐  ┌───────┐
  │Svc A  │  │Svc B  │  │Svc C  │        │Svc A  │  │Svc B  │  │Svc C  │
  └───┬───┘  └───┬───┘  └───┬───┘        └───┬───┘  └───┬───┘  └───┬───┘
      │          │          │                 │          │          │
      └──────────┼──────────┘            ┌────┴───┐ ┌───┴────┐ ┌──┴─────┐
           ┌─────┴─────┐                 │ BD-A   │ │ BD-B   │ │ BD-C   │
           │    BD     │                 │(Postgres)│(MongoDB)│(SQLite) │
           │(Compartilhado)│             └────────┘ └────────┘ └────────┘
           └───────────┘
```

**Consequência direta:** como não há joins entre bancos distintos, quando o `order-service` precisa do nome do cliente, ele faz uma chamada HTTP ao `user-service`. Isso gera latência de rede, mas garante o desacoplamento. Existem estratégias para mitigar esse custo, como cache local e replicação de dados via eventos — temas que abordaremos nas seções de comunicação e resiliência.

### 3.4 Contratos de Interface Bem Definidos

Serviços se comunicam exclusivamente por meio de **interfaces públicas** — tipicamente APIs REST, gRPC ou mensagens assíncronas. A implementação interna de cada serviço é uma caixa-preta para os demais.

Isso significa que o `order-service` não deve saber (e não pode depender de) como o `user-service` armazena dados internamente. Ele conhece apenas o contrato: "se eu fizer `GET /users/42`, recebo um JSON com `id`, `name` e `email`."

### 3.5 Tolerância a Falhas (Design for Failure)

Em um sistema distribuído, falhas **não são exceção — são a regra**. A rede falha, serviços ficam indisponíveis, latências aumentam. Um microsserviço bem projetado deve:

- **Não depender** da disponibilidade de outro serviço para funcionar minimamente
- Implementar **timeouts** em todas as chamadas de rede
- Usar **circuit breakers** para evitar cascatas de falha (detalhado na Seção 13)
- Ter **fallbacks** para situações de degradação

### 3.6 Descentralização

O modelo de microsserviços favorece a **descentralização** em múltiplas dimensões:

- **Governança descentralizada:** cada equipe escolhe a tecnologia mais adequada para seu serviço
- **Dados descentralizados:** cada serviço gerencia seus próprios dados (princípio 3.3)
- **Decisões descentralizadas:** equipes têm autonomia para projetar e implantar seus serviços

### Resumo dos princípios

| #   | Princípio                         | Pergunta de Validação                                      |
| --- | --------------------------------- | ---------------------------------------------------------- |
| 1   | Responsabilidade única de negócio | "Este serviço faz uma coisa e faz bem?"                    |
| 2   | Autonomia de deploy               | "Posso implantar este serviço sem tocar em outros?"        |
| 3   | Banco de dados isolado            | "Nenhum outro serviço acessa meu banco diretamente?"       |
| 4   | Contratos de interface            | "Minha API pública está documentada e versionada?"         |
| 5   | Tolerância a falhas               | "O que acontece se um serviço vizinho estiver fora do ar?" |
| 6   | Descentralização                  | "Cada equipe pode evoluir seu serviço de forma autônoma?"  |

---

## 4. Padrões de Decomposição — Onde Cortar?

A pergunta mais difícil ao adotar microsserviços não é "como implementar?", mas sim **"onde traçar as fronteiras entre os serviços?"**. Uma decomposição ruim resulta no pior dos mundos: a complexidade de um sistema distribuído combinada com o acoplamento de um monolito. A literatura de engenharia de software consolidou dois padrões principais para guiar esse processo: a **Decomposição por Domínio**, derivada do _Domain-Driven Design_ de **Eric Evans** (2003), e o **Strangler Fig Pattern**, proposto por **Martin Fowler** (2004) para migrações incrementais de monolitos legados.

### 4.1 Decomposição por Domínio de Negócio (Domain-Driven Design)

O **Domain-Driven Design (DDD)**, proposto por Eric Evans, oferece o vocabulário mais preciso para identificar fronteiras de serviço. O conceito central é o de **Bounded Context** (Contexto Delimitado): uma fronteira linguística e funcional dentro da qual um modelo de domínio específico tem significado consistente.

**Exemplo concreto — Sistema de e-commerce:**

```
┌─────────────────────────────────────────────────────────┐
│                    E-COMMERCE                           │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │
│  │  Catálogo   │  │   Pedidos   │  │   Pagamentos    │ │
│  │             │  │             │  │                 │ │
│  │ - Produto   │  │ - Pedido    │  │ - Transação     │ │
│  │ - Categoria │  │ - ItemPedido│  │ - Fatura        │ │
│  │ - Estoque   │  │ - Status    │  │ - Reembolso     │ │
│  │             │  │             │  │                 │ │
│  │ Bounded     │  │ Bounded     │  │ Bounded         │ │
│  │ Context 1   │  │ Context 2   │  │ Context 3       │ │
│  └─────────────┘  └─────────────┘  └─────────────────┘ │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐                       │
│  │   Entregas  │  │  Clientes   │                       │
│  │             │  │             │                       │
│  │ - Remessa   │  │ - Perfil    │                       │
│  │ - Rastreio  │  │ - Endereço  │                       │
│  │ - Frete     │  │ - Preferências                      │
│  │             │  │             │                       │
│  │ Bounded     │  │ Bounded     │                       │
│  │ Context 4   │  │ Context 5   │                       │
│  └─────────────┘  └─────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

Observe que "Produto" aparece no contexto de Catálogo e pode ser referenciado em Pedidos, mas cada contexto tem sua **própria representação**. No Catálogo, um produto tem descrição detalhada, fotos e SEO. No Pedido, o "produto" é apenas um ID, um nome e um preço unitário — uma versão simplificada.

### 4.2 O Padrão Strangler Fig (Figueira Estranguladora)

Quando a decomposição parte de um monolito existente (o cenário mais comum na vida real e no contexto da Aula 9), o padrão **Strangler Fig** oferece uma estratégia de migração **incremental**, sem reescrita total. O nome foi cunhado por **Martin Fowler** em seu [artigo de 2004](https://martinfowler.com/bliki/StranglerFigApplication.html), em referência às figueiras estranguladores australianas — trepadeiras que crescem ao redor de uma árvore hospedeira até substituí-la por completo. A metáfora descreve exatamente o processo: o novo sistema cresce em volta do monolito, assumindo funcionalidades uma a uma, até que o monolito original possa ser removido.

**Fases da migração:**

```
Fase 1: Monolito intacto         Fase 2: Extração parcial
┌───────────────────┐            ┌───────────────────┐
│    Monolito       │            │    Monolito       │
│                   │            │  (sem Auth)       │
│ Auth + Tasks +    │            │  Tasks + Users    │
│ Users             │            └────────┬──────────┘
└────────┬──────────┘                     │
         │                          ┌─────┴─────┐
    ┌────┴────┐                     │  Gateway   │
    │ Cliente │                     └─────┬─────┘
    └─────────┘                           │
                                    ┌─────┴─────┐
                                    │ Auth Svc   │  ← Novo
                                    │ (isolado)  │
                                    └────────────┘

Fase 3: Extração completa
┌────────────┐  ┌────────────┐  ┌────────────┐
│ Auth Svc   │  │ Tasks Svc  │  │ Users Svc  │
└─────┬──────┘  └─────┬──────┘  └─────┬──────┘
      └───────────────┼───────────────┘
                ┌─────┴─────┐
                │  Gateway   │
                └─────┬─────┘
                ┌─────┴─────┐
                │  Cliente   │
                └────────────┘
```

**Passos práticos do Strangler Fig:**

1. **Identificar** o módulo com fronteira mais clara (baixo acoplamento com o resto)
2. **Extrair** o código para um serviço separado com sua própria API
3. **Redirecionar** o tráfego do monolito para o novo serviço (via API Gateway ou proxy reverso)
4. **Remover** o código antigo do monolito
5. **Repetir** para o próximo módulo

### 4.3 Critérios Práticos para Traçar Fronteiras

| Critério                  | Boa fronteira                               | Fronteira problemática                                        |
| ------------------------- | ------------------------------------------- | ------------------------------------------------------------- |
| **Coesão funcional**      | Módulo faz uma coisa completa               | Módulo precisa chamar 5 outros para completar uma operação    |
| **Frequência de mudança** | Módulos que mudam juntos ficam juntos       | Módulos que mudam independentemente estão separados           |
| **Escalabilidade**        | Módulos com demandas de carga distintas     | Módulos com carga similar não precisam ser separados          |
| **Dados**                 | Cada serviço tem suas tabelas exclusivas    | Duas tabelas sempre usadas em JOIN pertencem ao mesmo serviço |
| **Equipe**                | Um serviço cabe em uma equipe (6–8 pessoas) | Se precisa de coordenação entre equipes, está grande demais   |

> **Para a Aula 9:** Na atividade prática de decomposição, a sugestão é começar extraindo o módulo de **autenticação** (`auth-service`) do monolito. Esse é tipicamente o módulo com fronteira mais nítida: recebe credenciais, emite tokens, e não depende da lógica de negócio dos demais módulos.

---

## 5. Comunicação entre Serviços

Em um monolito, módulos se comunicam por chamadas de função em memória — rápidas, confiáveis e transacionais. Em microsserviços, essa comunicação atravessa a **rede**, o que introduz latência, possibilidade de falha e complexidade de serialização. A escolha do padrão de comunicação é uma das decisões arquiteturais mais impactantes.

### 5.1 Comunicação Síncrona (Request/Response)

Na comunicação síncrona, o serviço chamador envia uma requisição e **aguarda a resposta** antes de continuar o processamento. É o modelo que você já conhece das APIs REST.

```
┌──────────────┐    HTTP GET /users/42    ┌──────────────┐
│ order-service │ ──────────────────────→ │ user-service  │
│               │ ←────────────────────── │               │
│  (aguarda)    │    200 { name: "Ana" }  │               │
└──────────────┘                          └──────────────┘
```

**Protocolos comuns:**

| Protocolo     | Formato                    | Desempenho | Quando usar                                                      |
| ------------- | -------------------------- | ---------- | ---------------------------------------------------------------- |
| **REST/HTTP** | JSON (texto)               | Bom        | Comunicação simples, equipes diversas, APIs públicas             |
| **gRPC**      | Protocol Buffers (binário) | Alto       | Comunicação interna de alta frequência, baixa latência           |
| **GraphQL**   | JSON (texto)               | Variável   | Quando o cliente precisa controlar exatamente quais dados recebe |

**Riscos da comunicação síncrona em cadeia:**

```
  Cliente → Gateway → Svc A → Svc B → Svc C → Svc D
                                                 ↑
                                          Se Svc D cair,
                                       TUDO para de funcionar
                                     (cascata de falhas)
```

Quando serviços dependem sincronamente uns dos outros em cadeia, a **disponibilidade do sistema** é o produto das disponibilidades individuais. Se cada serviço tem 99,5% de uptime, uma cadeia de 4 serviços resulta em: `0,995⁴ = 98,01%` — uma degradação significativa.

### 5.2 Comunicação Assíncrona (Event-Driven)

Na comunicação assíncrona, o serviço emissor publica uma mensagem ou evento e **continua processando imediatamente**, sem aguardar resposta. Um componente intermediário (message broker) cuida da entrega.

```
┌──────────────┐   Evento: "PedidoCriado"   ┌─────────────┐
│ order-service │ ────────────────────────→  │   Broker    │
│ (continua)   │                             │ (RabbitMQ / │
└──────────────┘                             │  Redis)     │
                                             └──────┬──────┘
                                                    │
                                    ┌───────────────┼───────────────┐
                                    ↓               ↓               ↓
                             ┌────────────┐  ┌────────────┐  ┌────────────┐
                             │ payment-   │  │ stock-     │  │ email-     │
                             │ service    │  │ service    │  │ service    │
                             └────────────┘  └────────────┘  └────────────┘
```

**Modelos de mensageria:**

| Modelo                    | Comportamento                                          | Tecnologias                 |
| ------------------------- | ------------------------------------------------------ | --------------------------- |
| **Point-to-Point (Fila)** | Uma mensagem é consumida por **um** consumidor         | RabbitMQ, AWS SQS           |
| **Publish-Subscribe**     | Uma mensagem é distribuída para **todos** os inscritos | Redis Pub/Sub, Apache Kafka |
| **Event Streaming**       | Eventos são persistidos e podem ser **relidos**        | Apache Kafka, AWS Kinesis   |

### 5.3 Consistência Eventual vs. Imediata

A escolha entre comunicação síncrona e assíncrona impacta diretamente o modelo de consistência dos dados:

| Aspecto             | Consistência Imediata (síncrona)        | Consistência Eventual (assíncrona)                     |
| ------------------- | --------------------------------------- | ------------------------------------------------------ |
| **Garantia**        | Dado atualizado em tempo real           | Dado pode levar milissegundos a segundos para propagar |
| **Complexidade**    | Menor, mas com acoplamento forte        | Maior, mas com desacoplamento                          |
| **Disponibilidade** | Menor (depende de todos os serviços)    | Maior (serviços funcionam independente)                |
| **Exemplo**         | Consultar saldo antes de aprovar compra | Enviar email de confirmação após pedido                |

### 5.4 Na Prática: Abordagem Híbrida

A maioria dos sistemas de produção combina ambos os padrões:

- **Síncrono** para operações que exigem resposta imediata ao usuário (consultas, validações)
- **Assíncrono** para efeitos colaterais e processamentos em segundo plano (notificações, relatórios, sincronização)

```
Exemplo: Fluxo de criação de pedido

  1. Cliente → POST /orders (síncrono, resposta imediata: "pedido criado")
  2. order-service → Publica evento "PedidoCriado" (assíncrono)
  3. payment-service → Consome evento, processa pagamento
  4. stock-service → Consome evento, reserva estoque
  5. notification-service → Consome evento, envia email
```

> **Para o escopo desta disciplina:** A implementação prática (Seção 11) utilizará comunicação **síncrona via REST** entre os microsserviços, por ser a abordagem que vocês já dominam. A comunicação assíncrona é apresentada aqui para completude conceitual e preparação para cenários reais de produção.

---

## 6. API Gateway — A Porta de Entrada Unificada

### 6.1 O Problema: Múltiplos Endpoints

Sem um ponto de entrada centralizado, o cliente precisaria conhecer o endereço de cada microsserviço individualmente. Em um sistema com 10 serviços, o frontend teria que gerenciar 10 URLs diferentes, lidar com autenticação em cada um e enfrentar problemas de CORS.

```
  ✗ SEM Gateway (cliente lida com tudo)

  ┌─────────┐
  │ Cliente │──→ http://auth:3001/login
  │         │──→ http://tasks:3002/tasks
  │         │──→ http://users:3003/users
  │         │──→ http://reports:3004/reports
  └─────────┘
  Problema: acoplamento direto, CORS, sem ponto central de segurança
```

### 6.2 A Solução: API Gateway

O API Gateway é um componente que atua como **ponto de entrada único** para todas as requisições externas. Ele recebe as requisições do cliente e as encaminha ao serviço interno adequado.

```
  ✓ COM Gateway (ponto único de entrada)

  ┌─────────┐     ┌───────────────┐     ┌──────────┐
  │ Cliente │────→│  API Gateway  │────→│ auth-svc │
  │         │     │   :8080       │────→│ task-svc │
  └─────────┘     │               │────→│ user-svc │
                  └───────────────┘     └──────────┘

  Cliente conhece apenas: http://gateway:8080
```

### 6.3 Responsabilidades do API Gateway

| Responsabilidade              | Descrição                                                                   |
| ----------------------------- | --------------------------------------------------------------------------- |
| **Roteamento**                | Direciona `/api/tasks/*` para `task-service`, `/auth/*` para `auth-service` |
| **Autenticação centralizada** | Valida JWT uma vez, no gateway, antes de encaminhar                         |
| **Rate Limiting**             | Limita número de requisições por cliente/IP                                 |
| **CORS**                      | Configura headers de Cross-Origin em um único ponto                         |
| **Agregação**                 | Combina respostas de múltiplos serviços em uma única resposta               |
| **Load Balancing**            | Distribui carga entre múltiplas instâncias de um serviço                    |

### 6.4 Implementação Simplificada com Express

Para fins didáticos, um API Gateway pode ser implementado como um serviço Express que utiliza `http-proxy-middleware` para redirecionar requisições:

```javascript
// gateway/index.js
const express = require("express");
const { createProxyMiddleware } = require("http-proxy-middleware");

const app = express();

// Roteamento para serviços internos
app.use(
  "/auth",
  createProxyMiddleware({
    target: "http://auth-service:3001",
    changeOrigin: true,
  }),
);

app.use(
  "/api/tasks",
  createProxyMiddleware({
    target: "http://task-service:3002",
    changeOrigin: true,
    pathRewrite: { "^/api/tasks": "/tasks" },
  }),
);

app.use(
  "/api/users",
  createProxyMiddleware({
    target: "http://user-service:3003",
    changeOrigin: true,
    pathRewrite: { "^/api/users": "/users" },
  }),
);

app.listen(8080, () => {
  console.log("API Gateway rodando na porta 8080");
});
```

> Em sistemas de produção, ferramentas como **NGINX**, **Kong**, **Traefik** ou **AWS API Gateway** são utilizadas em vez de implementações customizadas.

### 6.5 Por que Fastify? Expandindo a Pilha da Disciplina

Até este ponto do curso, todas as implementações utilizaram **Express.js** — e por boas razões. O Express é o framework HTTP mais estabelecido do ecossistema Node.js, com uma comunidade massiva e uma curva de aprendizado mínima. Ele serviu perfeitamente para fixar os fundamentos de APIs REST, rotas, middlewares e JWT.

Contudo, ao entrar no território de microsserviços, precisamos de funcionalidades que o Express não oferece nativamente e que fazem diferença em ambientes de produção:

| Funcionalidade          | Express                              | Fastify                                     |
| ----------------------- | ------------------------------------ | ------------------------------------------- |
| **Logging estruturado** | Manual (Winston, Morgan)             | Nativo (Pino — JSON, request-id automático) |
| **Validação de schema** | Manual (Joi, Zod, express-validator) | Nativo (JSON Schema compilado)              |
| **Performance**         | Referência base                      | 2-4x mais rápido em benchmarks              |
| **TypeScript**          | Requer `@types`                      | Suporte nativo (_first-class_)              |
| **Sistema de plugins**  | Middleware chain (linear)            | Plugin system com encapsulamento de escopo  |
| **JSON parsing**        | Requer `app.use(express.json())`     | Automático                                  |

**A justificativa pedagógica é dupla:**

1. **Transferência de conceitos.** Ao implementar o mesmo Gateway em Fastify, o aluno percebe que os fundamentos HTTP que aprendeu com Express são universais — rotas, verbos, headers, JSON, status codes. A troca de framework é uma mudança de sintaxe, não de paradigma.

2. **Práticas de produção.** Logging estruturado (JSON com níveis, request-id), validação de contratos e hooks de ciclo de vida não são _luxos_ opcionais em microsserviços — são requisitos operacionais. Fastify os oferece integrados, sem bibliotecas adicionais.

**Mapeamento direto Express → Fastify:**

```javascript
// ── EXPRESS ──                           // ── FASTIFY ──
const app = express();
const app = Fastify({ logger: true });
app.use(express.json()); // (automático)

app.get("/rota", (req, res) => {
  app.get("/rota", async (request, reply) => {
    res.json({ ok: true });
    return { ok: true }; // retorno direto
  });
});

res.status(201).json(data);
reply.code(201).send(data);
app.listen(3000);
app.listen({ port: 3000 });
```

### 6.6 API Gateway com Fastify e `@fastify/http-proxy`

O Fastify possui um plugin oficial para proxy reverso (`@fastify/http-proxy`) que simplifica a criação de gateways. A principal vantagem sobre o `http-proxy-middleware` do Express é o sistema de **hooks** — funções que executam em pontos específicos do ciclo de vida da requisição.

```javascript
// gateway/index.js (versão Fastify)
const Fastify = require("fastify");
const proxy = require("@fastify/http-proxy");
const jwt = require("jsonwebtoken");
const fs = require("fs");

const app = Fastify({ logger: true }); // Pino logger ativado

const PUBLIC_KEY = fs.readFileSync(
  process.env.JWT_PUBLIC_KEY_PATH || "./keys/public.pem",
  "utf8",
);

// ── Hook de autenticação JWT (reutilizável) ─────────────────
async function authenticateJWT(request, reply) {
  const authHeader = request.headers.authorization;
  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return reply.code(401).send({ error: "Token não fornecido." });
  }

  const token = authHeader.split(" ")[1];
  try {
    const decoded = jwt.verify(token, PUBLIC_KEY, { algorithms: ["RS256"] });
    // Propaga identidade para os serviços internos via headers
    request.headers["x-user-id"] = decoded.sub;
    request.headers["x-user-role"] = decoded.role;
  } catch (err) {
    return reply.code(401).send({ error: "Token inválido ou expirado." });
  }
}

// ── Rotas de proxy ──────────────────────────────────────────

// Auth — público (login não exige token)
app.register(proxy, {
  upstream: "http://auth-service:3001",
  prefix: "/auth",
  rewritePrefix: "",
});

// Tasks — protegido (preHandler valida JWT antes de proxiar)
app.register(proxy, {
  upstream: "http://task-service:3002",
  prefix: "/api/tasks",
  rewritePrefix: "/tasks",
  preHandler: authenticateJWT, // ← autenticação antes do proxy
});

// Users — protegido
app.register(proxy, {
  upstream: "http://user-service:3003",
  prefix: "/api/users",
  rewritePrefix: "/users",
  preHandler: authenticateJWT,
});

// Health check agregado
app.get("/health", async () => {
  const services = [
    "auth-service:3001",
    "task-service:3002",
    "user-service:3003",
  ];
  const results = {};
  for (const svc of services) {
    try {
      const res = await fetch(`http://${svc}/health`, {
        signal: AbortSignal.timeout(3000),
      });
      results[svc.split(":")[0]] = { status: "up" };
    } catch {
      results[svc.split(":")[0]] = { status: "down" };
    }
  }
  return { service: "gateway", downstream: results };
});

app.listen({ port: 8080, host: "0.0.0.0" });
```

**O que muda em relação à versão Express (seção 6.4):**

| Aspecto      | Express + http-proxy-middleware                  | Fastify + @fastify/http-proxy                   |
| ------------ | ------------------------------------------------ | ----------------------------------------------- |
| Roteamento   | `app.use('/path', createProxyMiddleware({...}))` | `app.register(proxy, { prefix: '/path', ... })` |
| Autenticação | Middleware separado na chain                     | `preHandler` hook integrado ao proxy            |
| Logging      | Manual (Morgan, console.log)                     | Automático (Pino, JSON estruturado)             |
| Configuração | `target`, `changeOrigin`, `pathRewrite`          | `upstream`, `prefix`, `rewritePrefix`           |

> **Nota para os alunos:** Ambas as implementações produzem o mesmo resultado funcional. A diferença está na ergonomia do código e nas funcionalidades operacionais (logging, hooks) que o Fastify oferece sem dependências adicionais.

---

## 7. Autenticação em Microsserviços — A Ponte com JWT

Nos nossos estudos sobre autenticação, foi mencionado que a camada de autenticação seria uma boa ideia para decompor em um microsserviço. Pois bem, aqui vamos entender o porque precisa o JWT se encaixa bem principalmente por ser _stateless_.

### 7.1 O Desafio: Onde Validar o Token?

Em um monolito, a validação do JWT acontece em um único middleware. Em microsserviços, surge a pergunta: **cada serviço deve validar o token, ou apenas o gateway?**

Existem duas abordagens principais:

**Abordagem 1 — Validação centralizada no Gateway:**

```
Cliente → Gateway (valida JWT) → Serviço interno (confia no Gateway)
                                  req.headers["x-user-id"] = "42"
                                  req.headers["x-user-role"] = "admin"
```

O Gateway valida o token e propaga as informações do usuário via **headers internos**. Os serviços internos confiam no Gateway e não revalidam.

**Abordagem 2 — Validação distribuída (cada serviço valida):**

```
Cliente → Gateway (repassa token) → Serviço interno (valida JWT localmente)
                                     jwt.verify(token, PUBLIC_KEY)
```

Cada serviço valida o token de forma independente. Isso é mais seguro (zero trust), mas exige que todos os serviços tenham acesso à chave de verificação.

### 7.2 Por que RS256 é Essencial em Microsserviços

No guia JWT, seção 3.1, apresentamos dois algoritmos:

| Algoritmo | Tipo        | Chave para assinar    | Chave para verificar |
| --------- | ----------- | --------------------- | -------------------- |
| **HS256** | Simétrico   | Segredo compartilhado | **Mesmo** segredo    |
| **RS256** | Assimétrico | Chave **privada**     | Chave **pública**    |

Com **HS256**, o mesmo segredo que assina o token também o verifica. Em microsserviços, isso significa distribuir o segredo para **todos** os serviços — se qualquer serviço for comprometido, o atacante pode **forjar tokens**.

Com **RS256**, apenas o `auth-service` possui a chave privada (assina tokens). Os demais serviços possuem apenas a chave pública (verificam tokens). Um serviço comprometido **não pode forjar tokens** — pode apenas verificar os existentes.

```
┌──────────────────┐
│   auth-service   │  Possui: CHAVE PRIVADA (assina)
│                  │  Expõe:  GET /.well-known/jwks.json
└────────┬─────────┘
         │ publica chave pública
         ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  task-service    │  │  user-service    │  │  report-service  │
│                  │  │                  │  │                  │
│ Possui: CHAVE    │  │ Possui: CHAVE    │  │ Possui: CHAVE    │
│ PÚBLICA (verifica)│  │ PÚBLICA (verifica)│  │ PÚBLICA (verifica)│
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

### 7.3 Implementação Prática com RS256

**Geração do par de chaves (terminal):**

```bash
# Gerar chave privada (2048 bits)
openssl genrsa -out private.pem 2048

# Extrair chave pública
openssl rsa -in private.pem -pubout -out public.pem
```

**No `auth-service` (assina com chave privada):**

```javascript
const jwt = require("jsonwebtoken");
const fs = require("fs");

const PRIVATE_KEY = fs.readFileSync("./keys/private.pem", "utf8");

function generateToken(user) {
  const payload = { sub: user.id, role: user.role };
  return jwt.sign(payload, PRIVATE_KEY, {
    algorithm: "RS256",
    expiresIn: "1h",
    issuer: "auth-service",
  });
}
```

**Nos demais serviços (verificam com chave pública):**

```javascript
const jwt = require("jsonwebtoken");
const fs = require("fs");

const PUBLIC_KEY = fs.readFileSync("./keys/public.pem", "utf8");

function authenticateToken(req, res, next) {
  const authHeader = req.headers["authorization"];
  const token = authHeader && authHeader.split(" ")[1];

  if (!token) {
    return res.status(401).json({ error: "Token não fornecido." });
  }

  jwt.verify(token, PUBLIC_KEY, { algorithms: ["RS256"] }, (err, decoded) => {
    if (err) {
      return res.status(403).json({ error: "Token inválido." });
    }
    req.user = decoded;
    next();
  });
}
```

> Observe o parâmetro `algorithms: ["RS256"]` no `jwt.verify()`. Isso é uma medida de segurança que força a biblioteca a aceitar **apenas** tokens assinados com RS256, rejeitando tokens com `alg: "none"` ou `alg: "HS256"` que um atacante poderia tentar injetar.

---

## 8. Docker — Fundamentos de Containerização

### 8.1 Por que Docker é Fundamental para Microsserviços?

Docker resolve um problema clássico do desenvolvimento de software: **"na minha máquina funciona."** Um container Docker empacota a aplicação com todas as suas dependências (runtime, bibliotecas, configurações) em uma unidade isolada e portável.

Para microsserviços, Docker é a tecnologia que viabiliza na prática a independência entre serviços. Cada serviço roda em seu próprio container, com suas próprias dependências, sem conflito com os demais.

### 8.2 Conceitos Essenciais

| Conceito       | Analogia          | Descrição                                                      |
| -------------- | ----------------- | -------------------------------------------------------------- |
| **Imagem**     | Receita de bolo   | Template imutável e somente-leitura que define o ambiente      |
| **Container**  | Bolo pronto       | Instância em execução de uma imagem                            |
| **Dockerfile** | Livro de receitas | Arquivo texto com instruções para construir uma imagem         |
| **Registry**   | Padaria central   | Repositório de imagens (Docker Hub, GitHub Container Registry) |
| **Volume**     | Vasilha separada  | Mecanismo de persistência de dados fora do container           |

```
  ┌───────────────────────────────────────────────────┐
  │  Dockerfile → (docker build) → Imagem            │
  │                                   │               │
  │                          (docker run)             │
  │                                   │               │
  │                              Container            │
  │                          (instância viva)         │
  └───────────────────────────────────────────────────┘
```

### 8.3 Instalação

- **Windows/Mac:** Instale o [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- **Linux:** Siga o [guia oficial](https://docs.docker.com/engine/install/) para sua distribuição

Verifique a instalação:

```bash
docker --version
# Docker version 27.x.x

docker compose version
# Docker Compose version v2.x.x
```

### 8.4 Comandos Essenciais

```bash
# Imagens
docker images                    # Lista imagens locais
docker pull node:22-alpine       # Baixa imagem do registry
docker build -t meu-app .        # Constrói imagem a partir do Dockerfile
docker rmi <image_id>            # Remove imagem

# Containers
docker ps                        # Lista containers em execução
docker ps -a                     # Lista todos (incluindo parados)
docker run -p 3000:3000 meu-app  # Cria e executa container
docker stop <container_id>       # Para container
docker rm <container_id>         # Remove container

# Logs e debug
docker logs <container_id>       # Exibe logs do container
docker exec -it <id> sh          # Abre shell dentro do container
```

### 8.5 Container vs. Máquina Virtual

| Aspecto           | Container                                      | Máquina Virtual                    |
| ----------------- | ---------------------------------------------- | ---------------------------------- |
| **Isolamento**    | Nível de processo (compartilha kernel do host) | Nível de hardware (kernel próprio) |
| **Tamanho**       | Megabytes (5–200 MB)                           | Gigabytes (1–10 GB)                |
| **Inicialização** | Segundos                                       | Minutos                            |
| **Overhead**      | Mínimo                                         | Significativo (hypervisor)         |
| **Portabilidade** | Alta (roda onde Docker roda)                   | Média (depende do hypervisor)      |

---

## 9. Dockerfile — Construindo Imagens

O Dockerfile é um arquivo texto que contém as **instruções sequenciais** para construir uma imagem Docker. Cada instrução cria uma **camada** (layer) na imagem.

### 9.1 Dockerfile para uma API Node.js

```dockerfile
# ---- Imagem base ----
# Usa Node.js 20 com Alpine Linux (imagem leve, ~50MB)
FROM node:22-alpine

# ---- Diretório de trabalho ----
# Cria e define /app como diretório de trabalho
WORKDIR /app

# ---- Dependências ----
# Copia APENAS os arquivos de dependência primeiro (otimiza cache)
COPY package*.json ./

# Instala dependências de produção
RUN npm ci --only=production

# ---- Código da aplicação ----
# Copia o restante do código
COPY . .

# ---- Porta ----
# Declara a porta que o container expõe (documentação)
EXPOSE 3000

# ---- Comando de inicialização ----
# Define o comando que será executado ao iniciar o container
CMD ["node", "index.js"]
```

### 9.2 Instruções Mais Utilizadas

| Instrução | Propósito                              | Exemplo                    |
| --------- | -------------------------------------- | -------------------------- |
| `FROM`    | Define imagem base                     | `FROM node:22-alpine`      |
| `WORKDIR` | Define diretório de trabalho           | `WORKDIR /app`             |
| `COPY`    | Copia arquivos do host para a imagem   | `COPY package.json ./`     |
| `RUN`     | Executa comando durante o build        | `RUN npm ci`               |
| `EXPOSE`  | Documenta porta do container           | `EXPOSE 3000`              |
| `CMD`     | Comando executado ao iniciar container | `CMD ["node", "index.js"]` |
| `ENV`     | Define variável de ambiente            | `ENV NODE_ENV=production`  |

### 9.3 O Arquivo `.dockerignore`

Assim como o `.gitignore`, o `.dockerignore` evita que arquivos desnecessários sejam copiados para a imagem:

```
node_modules
npm-debug.log
.git
.gitignore
.env
*.md
tests/
```

### 9.4 Otimização: Por que Copiar `package.json` Antes?

A ordem das instruções no Dockerfile importa para o **cache de camadas**:

```dockerfile
# ✓ CORRETO: Dependências são cacheadas
COPY package*.json ./     # Camada 1: muda raramente → cache reutilizado
RUN npm ci                # Camada 2: recria só se package.json mudou
COPY . .                  # Camada 3: muda frequentemente

# ✗ INEFICIENTE: Cache desperdiçado a cada mudança de código
COPY . .                  # Qualquer mudança no código invalida TUDO
RUN npm ci                # npm ci roda toda vez, mesmo sem mudanças
```

### 9.5 Construindo e Executando

```bash
# Construir a imagem
docker build -t task-service:1.0 .

# Executar o container
docker run -d \
  --name task-svc \
  -p 3002:3000 \
  -e DB_PATH=/app/data/tasks.db \
  task-service:1.0

# Verificar se está rodando
docker ps

# Ver logs
docker logs task-svc
```

---

## 10. Docker Compose — Orquestrando Múltiplos Serviços

Executar cada container manualmente com `docker run` é viável para um ou dois serviços, mas em uma arquitetura de microsserviços com 3, 5 ou 10 serviços, torna-se impraticável. O **Docker Compose** resolve esse problema permitindo definir e orquestrar **todos os serviços** em um único arquivo YAML.

### 10.1 Estrutura do `docker-compose.yml`

```yaml
# docker-compose.yml
services:
  # --- API Gateway ---
  gateway:
    build: ./gateway
    ports:
      - "8080:8080"
    depends_on:
      - auth-service
      - task-service
    environment:
      - AUTH_SERVICE_URL=http://auth-service:3001
      - TASK_SERVICE_URL=http://task-service:3002
    networks:
      - app-network

  # --- Serviço de Autenticação ---
  auth-service:
    build: ./services/auth
    expose:
      - "3001"
    environment:
      - PORT=3001
      - DB_PATH=/app/data/users.db
      - JWT_PRIVATE_KEY_PATH=/app/keys/private.pem
    volumes:
      - auth-data:/app/data
      - ./keys:/app/keys:ro
    networks:
      - app-network

  # --- Serviço de Tarefas ---
  task-service:
    build: ./services/tasks
    expose:
      - "3002"
    environment:
      - PORT=3002
      - DB_PATH=/app/data/tasks.db
      - JWT_PUBLIC_KEY_PATH=/app/keys/public.pem
    volumes:
      - task-data:/app/data
      - ./keys:/app/keys:ro
    networks:
      - app-network

# --- Volumes persistentes ---
volumes:
  auth-data:
  task-data:

# --- Rede interna ---
networks:
  app-network:
    driver: bridge
```

### 10.2 Conceitos-Chave do Compose

| Conceito      | Descrição                                                                      |
| ------------- | ------------------------------------------------------------------------------ |
| `build`       | Caminho para o Dockerfile do serviço                                           |
| `ports`       | Mapeia porta do host → container (`"host:container"`) — expõe ao mundo externo |
| `expose`      | Expõe porta **apenas** na rede interna (entre containers)                      |
| `depends_on`  | Define ordem de inicialização (não garante que o serviço está pronto)          |
| `environment` | Variáveis de ambiente do container                                             |
| `volumes`     | Persistência de dados e montagem de arquivos                                   |
| `networks`    | Rede virtual compartilhada entre serviços                                      |

### 10.3 A Diferença entre `ports` e `expose`

```yaml
# PORTS: Acessível do host (seu computador) e entre containers
ports:
  - "8080:8080" # localhost:8080 → container:8080

# EXPOSE: Acessível APENAS entre containers na mesma rede
expose:
  - "3001" # Outros containers acessam via auth-service:3001
    # Seu computador NÃO acessa diretamente
```

No contexto de microsserviços, apenas o **Gateway** utiliza `ports` (é a porta de entrada pública). Os serviços internos utilizam `expose`, pois não devem ser acessíveis diretamente de fora. Este é um bom exemplo de como podemos inserir segurança em todas as camadas da aplicação, e não somente através de middlewares de aplicação.

### 10.4 DNS Automático

O Docker Compose cria automaticamente um **DNS interno** baseado nos nomes dos serviços. Dentro da rede `app-network`, o `gateway` pode acessar o `auth-service` pelo hostname `auth-service` (não pelo IP):

```javascript
// Dentro do gateway, ao fazer proxy para o auth-service:
const target = "http://auth-service:3001"; // Resolvido automaticamente pelo Docker DNS
```

### 10.5 Comandos do Compose

```bash
# Construir e iniciar todos os serviços
docker compose up --build

# Iniciar em segundo plano (detached)
docker compose up -d --build

# Ver logs de todos os serviços
docker compose logs -f

# Ver logs de um serviço específico
docker compose logs -f auth-service

# Parar todos os serviços
docker compose down

# Parar e remover volumes (cuidado: apaga dados persistidos)
docker compose down -v

# Reconstruir um serviço específico
docker compose build task-service
docker compose up -d task-service
```

---

## 11. Implementação Prática — Decompondo uma API Monolítica (Express)

Esta seção apresenta o exercício central: transformar a API monolítica do Trabalho I em uma arquitetura de microsserviços containerizados com Docker Compose.

### 11.1 Estrutura de Diretórios do Projeto

```
microservices-tasks/
├── docker-compose.yml
├── keys/
│   ├── private.pem          # Chave privada (só auth-service usa)
│   └── public.pem           # Chave pública (task-service usa)
├── gateway/
│   ├── Dockerfile
│   ├── package.json
│   └── index.js
├── services/
│   ├── auth/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   ├── index.js
│   │   ├── controllers/
│   │   ├── services/
│   │   └── data/
│   └── tasks/
│       ├── Dockerfile
│       ├── package.json
│       ├── index.js
│       ├── controllers/
│       ├── middleware/
│       ├── services/
│       └── data/
```

### 11.2 Serviço de Autenticação (`auth-service`)

```javascript
// services/auth/index.js
const express = require("express");
const jwt = require("jsonwebtoken");
const bcrypt = require("bcrypt");
const fs = require("fs");

const app = express();
app.use(express.json());

const PORT = process.env.PORT || 3001;
const PRIVATE_KEY = fs.readFileSync(
  process.env.JWT_PRIVATE_KEY_PATH || "./keys/private.pem",
  "utf8",
);

// Simulação de banco de dados em memória
const users = [
  {
    id: "user_1",
    email: "aluno@ifsul.edu.br",
    // hash de "senha123" com bcrypt (saltRounds=10)
    password: "$2b$10$abcdefghijklmnopqrstuuABCDEFGHIJKLMNOPQRS",
    role: "user",
  },
];

// POST /login
app.post("/login", async (req, res) => {
  const { email, password } = req.body;

  if (!email || !password) {
    return res.status(400).json({ error: "Email e senha são obrigatórios." });
  }

  const user = users.find((u) => u.email === email);
  if (!user) {
    return res.status(401).json({ error: "Credenciais inválidas." });
  }

  const valid = await bcrypt.compare(password, user.password);
  if (!valid) {
    return res.status(401).json({ error: "Credenciais inválidas." });
  }

  // Assina com RS256 (chave privada)
  const token = jwt.sign({ sub: user.id, role: user.role }, PRIVATE_KEY, {
    algorithm: "RS256",
    expiresIn: "1h",
    issuer: "auth-service",
  });

  return res.json({ token });
});

// Health check
app.get("/health", (req, res) => {
  res.json({ status: "ok", service: "auth-service" });
});

app.listen(PORT, () => {
  console.log(`auth-service rodando na porta ${PORT}`);
});
```

### 11.3 Serviço de Tarefas (`task-service`)

```javascript
// services/tasks/index.js
const express = require("express");
const jwt = require("jsonwebtoken");
const fs = require("fs");

const app = express();
app.use(express.json());

const PORT = process.env.PORT || 3002;
const PUBLIC_KEY = fs.readFileSync(
  process.env.JWT_PUBLIC_KEY_PATH || "./keys/public.pem",
  "utf8",
);

// Middleware de autenticação (verifica com chave pública)
function authenticateToken(req, res, next) {
  const authHeader = req.headers["authorization"];
  const token = authHeader && authHeader.split(" ")[1];

  if (!token) {
    return res.status(401).json({ error: "Token não fornecido." });
  }

  jwt.verify(token, PUBLIC_KEY, { algorithms: ["RS256"] }, (err, decoded) => {
    if (err) {
      if (err.name === "TokenExpiredError") {
        return res.status(401).json({ error: "Token expirado." });
      }
      return res.status(403).json({ error: "Token inválido." });
    }
    req.user = decoded;
    next();
  });
}

// Base de dados em memória
let tasks = [
  { id: "1", title: "Estudar microsserviços", userId: "user_1", done: false },
];
let nextId = 2;

// GET /tasks — público
app.get("/tasks", (req, res) => {
  res.json(tasks);
});

// POST /tasks — protegido
app.post("/tasks", authenticateToken, (req, res) => {
  const { title } = req.body;
  if (!title) {
    return res.status(400).json({ error: "Título é obrigatório." });
  }

  const task = {
    id: String(nextId++),
    title,
    userId: req.user.sub,
    done: false,
  };
  tasks.push(task);
  res.status(201).json(task);
});

// DELETE /tasks/:id — protegido
app.delete("/tasks/:id", authenticateToken, (req, res) => {
  const index = tasks.findIndex((t) => t.id === req.params.id);
  if (index === -1) {
    return res.status(404).json({ error: "Tarefa não encontrada." });
  }
  tasks.splice(index, 1);
  res.status(204).send();
});

// Health check
app.get("/health", (req, res) => {
  res.json({ status: "ok", service: "task-service" });
});

app.listen(PORT, () => {
  console.log(`task-service rodando na porta ${PORT}`);
});
```

### 11.4 Dockerfile (idêntico para ambos os serviços)

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

### 11.5 Executando o Projeto

```bash
# 1. Gerar par de chaves RSA
mkdir -p keys
openssl genrsa -out keys/private.pem 2048
openssl rsa -in keys/private.pem -pubout -out keys/public.pem

# 2. Subir todos os serviços
docker compose up --build

# 3. Testar login (via gateway)
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"aluno@ifsul.edu.br","password":"senha123"}'

# Resposta: { "token": "eyJhbGciOiJSUzI1NiIs..." }

# 4. Criar tarefa (usando o token recebido)
curl -X POST http://localhost:8080/api/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token_aqui>" \
  -d '{"title":"Minha primeira tarefa em microsserviço"}'

# 5. Listar tarefas
curl http://localhost:8080/api/tasks
```

### 11.6 O que Observar na Prática

Ao rodar `docker compose up --build`, observe no terminal:

1. **Build paralelo:** Docker constrói as imagens de cada serviço
2. **Inicialização ordenada:** Gateway aguarda auth e tasks (via `depends_on`)
3. **Logs separados:** Cada serviço exibe seus logs com prefixo colorido
4. **Isolamento de rede:** Serviços internos não são acessíveis de fora (apenas via gateway)
5. **Persistência:** Volumes mantêm dados entre reinícios

---

## 12. Implementação com Fastify — Microsserviços com Persistência

A seção anterior demonstrou a decomposição de um monolito usando Express e dados em memória. Agora, apresentamos a mesma arquitetura reimplementada com **Fastify** e **better-sqlite3**, adotando práticas mais próximas de ambientes de produção: logging estruturado nativo, persistência real via SQLite e um API Gateway com autenticação JWT centralizada via hooks.

> **Por que duas implementações?** A seção 11 fixa os conceitos fundamentais (rotas, JWT, Docker Compose) com a sintaxe Express que o aluno já domina. Esta seção mostra como esses mesmos conceitos se traduzem em um framework moderno, sem alterar a essência — rotas HTTP, JSON, status codes e containers são os mesmos. A diferença está na ergonomia e nas funcionalidades integradas.

Para uma comparação detalhada entre Express e Fastify, consulte a seção 6.5.

### 12.1 Estrutura de Diretórios do Projeto

```
microservices-fastify/
├── docker-compose.yml
├── keys/
│   ├── private.pem          # Chave privada RSA (só auth-service usa)
│   └── public.pem           # Chave pública (gateway e task-service usam)
├── gateway/
│   ├── Dockerfile
│   ├── package.json
│   └── index.js             # Fastify + @fastify/http-proxy
└── services/
    ├── auth/
    │   ├── Dockerfile
    │   ├── package.json
    │   └── index.js          # Fastify + bcrypt + better-sqlite3
    └── tasks/
        ├── Dockerfile
        ├── package.json
        └── index.js          # Fastify + better-sqlite3
```

### 12.2 Serviço de Autenticação (`auth-service`)

```javascript
// services/auth/index.js
const Fastify = require("fastify");
const jwt = require("jsonwebtoken");
const bcrypt = require("bcrypt");
const Database = require("better-sqlite3");
const fs = require("fs");

const app = Fastify({ logger: true }); // Pino logger ativado

const PORT = process.env.PORT || 3001;
const DB_PATH = process.env.DB_PATH || "./data/auth.db";
const PRIVATE_KEY = fs.readFileSync(
  process.env.JWT_PRIVATE_KEY_PATH || "./keys/private.pem",
  "utf8",
);

// ── Banco de dados com persistência real ────────────────────
const db = new Database(DB_PATH);

db.exec(`
  CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL,
    role TEXT DEFAULT 'user',
    name TEXT NOT NULL
  )
`);

// Seed automático (apenas se tabela vazia)
const count = db.prepare("SELECT COUNT(*) as total FROM users").get();
if (count.total === 0) {
  const salt = bcrypt.genSaltSync(10);
  const insert = db.prepare(
    "INSERT INTO users (id, email, password, role, name) VALUES (?, ?, ?, ?, ?)",
  );
  insert.run(
    "user_1",
    "aluno@ifsul.edu.br",
    bcrypt.hashSync("senha123", salt),
    "user",
    "Aluno IFSul",
  );
  insert.run(
    "user_2",
    "admin@ifsul.edu.br",
    bcrypt.hashSync("admin123", salt),
    "admin",
    "Admin IFSul",
  );
  console.log("Seed: 2 usuários criados");
}

// POST /login
app.post("/login", async (request, reply) => {
  const { email, password } = request.body || {};

  if (!email || !password) {
    return reply.code(400).send({ error: "Email e senha são obrigatórios." });
  }

  const user = db.prepare("SELECT * FROM users WHERE email = ?").get(email);
  if (!user) {
    return reply.code(401).send({ error: "Credenciais inválidas." });
  }

  const valid = await bcrypt.compare(password, user.password);
  if (!valid) {
    return reply.code(401).send({ error: "Credenciais inválidas." });
  }

  const token = jwt.sign(
    { sub: user.id, role: user.role, name: user.name },
    PRIVATE_KEY,
    { algorithm: "RS256", expiresIn: "1h", issuer: "auth-service" },
  );

  return { token, user: { id: user.id, name: user.name, role: user.role } };
});

// GET /health
app.get("/health", async () => ({
  status: "ok",
  service: "auth-service",
  uptime: process.uptime(),
  timestamp: new Date().toISOString(),
}));

app.listen({ port: PORT, host: "0.0.0.0" }, (err) => {
  if (err) {
    app.log.error(err);
    process.exit(1);
  }
  app.log.info(`auth-service rodando na porta ${PORT}`);
});
```

**O que mudou em relação à versão Express (seção 11.2):**

| Aspecto      | Express (seção 11)                 | Fastify (seção 12)                  |
| ------------ | ---------------------------------- | ----------------------------------- |
| Dados        | Array em memória (`users = [...]`) | SQLite com `better-sqlite3`         |
| Parsing JSON | `app.use(express.json())`          | Automático                          |
| Logging      | `console.log`                      | Pino (JSON, request-id automático)  |
| Resposta     | `res.json({ token })`              | `return { token }` (retorno direto) |
| Seed         | Hash hardcoded no código           | `bcrypt.hashSync` com seed dinâmico |

### 12.3 Serviço de Tarefas (`task-service`)

```javascript
// services/tasks/index.js
const Fastify = require("fastify");
const Database = require("better-sqlite3");

const app = Fastify({ logger: true });

const PORT = process.env.PORT || 3002;
const DB_PATH = process.env.DB_PATH || "./data/tasks.db";

// ── Banco de dados ──────────────────────────────────────────
const db = new Database(DB_PATH);

db.exec(`
  CREATE TABLE IF NOT EXISTS tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    user_id TEXT NOT NULL,
    done INTEGER DEFAULT 0,
    created_at TEXT DEFAULT (datetime('now'))
  )
`);

// Seed
const taskCount = db.prepare("SELECT COUNT(*) as total FROM tasks").get();
if (taskCount.total === 0) {
  db.prepare("INSERT INTO tasks (title, user_id) VALUES (?, ?)").run(
    "Estudar microsserviços",
    "user_1",
  );
  console.log("Seed: 1 tarefa criada");
}

// GET /tasks — lista tarefas (o gateway já validou o JWT)
app.get("/tasks", async (request) => {
  const userId = request.headers["x-user-id"];
  // Se tiver x-user-id, filtra por usuário; senão, retorna todas
  if (userId) {
    return db.prepare("SELECT * FROM tasks WHERE user_id = ?").all(userId);
  }
  return db.prepare("SELECT * FROM tasks").all();
});

// POST /tasks — criar tarefa (identidade vem via header do gateway)
app.post("/tasks", async (request, reply) => {
  const { title } = request.body || {};
  const userId = request.headers["x-user-id"] || "anonymous";

  if (!title) {
    return reply.code(400).send({ error: "Título é obrigatório." });
  }

  const info = db
    .prepare("INSERT INTO tasks (title, user_id) VALUES (?, ?)")
    .run(title, userId);

  const task = db
    .prepare("SELECT * FROM tasks WHERE id = ?")
    .get(info.lastInsertRowid);
  return reply.code(201).send(task);
});

// DELETE /tasks/:id — remover tarefa
app.delete("/tasks/:id", async (request, reply) => {
  const result = db
    .prepare("DELETE FROM tasks WHERE id = ?")
    .run(request.params.id);
  if (result.changes === 0) {
    return reply.code(404).send({ error: "Tarefa não encontrada." });
  }
  return reply.code(204).send();
});

// GET /health
app.get("/health", async () => ({
  status: "ok",
  service: "task-service",
  uptime: process.uptime(),
  timestamp: new Date().toISOString(),
}));

app.listen({ port: PORT, host: "0.0.0.0" }, (err) => {
  if (err) {
    app.log.error(err);
    process.exit(1);
  }
  app.log.info(`task-service rodando na porta ${PORT}`);
});
```

**Diferença fundamental:** Na versão Express (seção 11.3), o task-service valida JWT localmente com a chave pública. Na versão Fastify, a autenticação é centralizada no Gateway (seção 6.6) — o task-service recebe apenas o `x-user-id` via header, já validado. Isso simplifica o serviço e elimina a duplicação da lógica de autenticação.

### 12.4 API Gateway (`gateway`)

Idêntico ao apresentado na seção 6.6. A implementação completa com `@fastify/http-proxy` e hook `preHandler` de JWT está documentada lá.

### 12.5 Docker Compose

```yaml
# docker-compose.yml
services:
  gateway:
    build: ./gateway
    ports:
      - "8080:8080"
    depends_on:
      - auth-service
      - task-service
    volumes:
      - ./keys/public.pem:/app/keys/public.pem:ro
    environment:
      - AUTH_SERVICE_URL=http://auth-service:3001
      - TASK_SERVICE_URL=http://task-service:3002
      - JWT_PUBLIC_KEY_PATH=/app/keys/public.pem
    networks:
      - app-net

  auth-service:
    build: ./services/auth
    volumes:
      - ./keys/private.pem:/app/keys/private.pem:ro
      - auth-data:/app/data
    environment:
      - PORT=3001
      - JWT_PRIVATE_KEY_PATH=/app/keys/private.pem
      - DB_PATH=/app/data/auth.db
    networks:
      - app-net

  task-service:
    build: ./services/tasks
    volumes:
      - tasks-data:/app/data
    environment:
      - PORT=3002
      - DB_PATH=/app/data/tasks.db
    networks:
      - app-net

networks:
  app-net:
    driver: bridge

volumes:
  auth-data:
  tasks-data:
```

### 12.6 Executando o Projeto

```bash
# 1. Gerar par de chaves RSA
mkdir -p keys
openssl genrsa -out keys/private.pem 2048
openssl rsa -in keys/private.pem -pubout -out keys/public.pem

# 2. Subir todos os serviços
docker compose up --build

# 3. Login
curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"aluno@ifsul.edu.br","password":"senha123"}' | jq

# Salvar token numa variável
TOKEN=$(curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"aluno@ifsul.edu.br","password":"senha123"}' | jq -r '.token')

# 4. Criar tarefa (autenticado)
curl -s -X POST http://localhost:8080/api/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"Minha primeira tarefa com Fastify"}' | jq

# 5. Listar tarefas
curl -s http://localhost:8080/api/tasks \
  -H "Authorization: Bearer $TOKEN" | jq

# 6. Health check do gateway
curl -s http://localhost:8080/health | jq
```

### 12.7 Comparação Lado a Lado

| Aspecto      | Seção 11 (Express)                        | Seção 12 (Fastify)                                  |
| ------------ | ----------------------------------------- | --------------------------------------------------- |
| Framework    | Express 4.x                               | Fastify 5.x                                         |
| Persistência | Dados em memória (perdem-se ao reiniciar) | SQLite via `better-sqlite3` (persistem via volumes) |
| Logging      | `console.log` manual                      | Pino nativo (JSON estruturado, níveis, request-id)  |
| Gateway      | `http-proxy-middleware`                   | `@fastify/http-proxy` com hook `preHandler`         |
| Autenticação | Duplicada em cada serviço                 | Centralizada no gateway (via JWT + headers)         |
| JSON parsing | `app.use(express.json())` explícito       | Automático                                          |
| Respostas    | `res.status(201).json(data)`              | `reply.code(201).send(data)` ou `return data`       |
| Health check | JSON simples                              | JSON com uptime e timestamp                         |

> **Para o aluno:** Execute ambas as versões. Na seção 11, reinicie os containers e observe que os dados desaparecem (memória). Na seção 12, reinicie e observe que os dados persistem (SQLite + volumes Docker). Essa diferença é fundamental para entender por que microsserviços em produção precisam de persistência real.

---

## 13. Observabilidade e Monitoramento

Em um monolito, um erro gera um stack trace completo em um único log. Em microsserviços, uma requisição pode percorrer 3, 5 ou 10 serviços diferentes. Sem ferramentas adequadas, depurar esse caminho é inviável.

### 13.1 Os Três Pilares da Observabilidade

| Pilar                   | O que responde                         | Ferramentas              |
| ----------------------- | -------------------------------------- | ------------------------ |
| **Logs**                | "O que aconteceu?"                     | Winston, Pino, ELK Stack |
| **Métricas**            | "Qual o estado do sistema?"            | Prometheus, Grafana      |
| **Tracing distribuído** | "Qual caminho a requisição percorreu?" | OpenTelemetry, Jaeger    |

### 13.2 Correlation ID — Rastreando Requisições

A técnica mais simples e de maior impacto é gerar um **identificador único** no Gateway e propagá-lo para todos os serviços downstream:

```javascript
// No Gateway: gerar e propagar o correlation ID
const { v4: uuidv4 } = require("uuid");

app.use((req, res, next) => {
  // Usa o ID do cliente se existir, ou gera um novo
  req.correlationId = req.headers["x-correlation-id"] || uuidv4();
  res.setHeader("x-correlation-id", req.correlationId);
  next();
});
```

```javascript
// Nos serviços internos: incluir o correlation ID nos logs
app.use((req, res, next) => {
  const correlationId = req.headers["x-correlation-id"] || "unknown";
  console.log(`[${correlationId}] ${req.method} ${req.url}`);
  next();
});
```

### 13.3 Health Checks

Cada serviço deve expor um endpoint de verificação de saúde:

```javascript
app.get("/health", (req, res) => {
  res.json({
    status: "ok",
    service: "task-service",
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
  });
});
```

---

## 14. Padrões de Resiliência

### 14.1 Circuit Breaker (Disjuntor)

O padrão Circuit Breaker previne **cascatas de falha**. Funciona como um disjuntor elétrico: quando detecta que um serviço downstream está falhando repetidamente, "abre o circuito" e retorna uma resposta de fallback imediata, evitando que requisições continuem sendo enviadas ao serviço degradado.

```
  Estado FECHADO (normal)        Estado ABERTO (proteção)
  ┌─────┐     ┌─────┐           ┌─────┐     ┌─────┐
  │Svc A│────→│Svc B│           │Svc A│──X  │Svc B│ (fora do ar)
  └─────┘     └─────┘           └──┬──┘     └─────┘
  Requisições passam              │
  normalmente                 Retorna fallback
                              imediatamente
```

**Implementação com a biblioteca `opossum`:**

```javascript
const CircuitBreaker = require("opossum");

// Função que chama o serviço externo
async function callUserService(userId) {
  const response = await fetch(`http://user-service:3003/users/${userId}`);
  if (!response.ok) throw new Error(`Status ${response.status}`);
  return response.json();
}

// Configuração do circuit breaker
const breaker = new CircuitBreaker(callUserService, {
  timeout: 3000, // Timeout de 3 segundos
  errorThresholdPercentage: 50, // Abre se 50% das chamadas falham
  resetTimeout: 10000, // Tenta fechar após 10 segundos
});

// Fallback quando o circuito está aberto
breaker.fallback((userId) => ({
  id: userId,
  name: "Usuário indisponível",
  cached: true,
}));

// Uso
app.get("/orders/:id", async (req, res) => {
  const user = await breaker.fire(req.params.userId);
  res.json({ order: orderData, user });
});
```

### 14.2 Retry com Backoff Exponencial

Para falhas transitórias (timeout de rede, serviço reiniciando), retentativas automáticas com intervalos crescentes:

```javascript
async function fetchWithRetry(url, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, { signal: AbortSignal.timeout(3000) });
      if (response.ok) return response.json();
      throw new Error(`Status ${response.status}`);
    } catch (err) {
      if (attempt === maxRetries) throw err;
      const delay = Math.pow(2, attempt) * 100; // 200ms, 400ms, 800ms
      console.log(`Tentativa ${attempt} falhou. Retentando em ${delay}ms...`);
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
}
```

### 14.3 Timeout

Toda chamada de rede **deve** ter um timeout configurado. Uma chamada sem timeout pode bloquear recursos indefinidamente:

```javascript
// fetch com timeout usando AbortSignal
const response = await fetch("http://user-service:3003/users/42", {
  signal: AbortSignal.timeout(5000), // 5 segundos
});
```

---

## 15. Erros Comuns e Antipadrões

| Antipadrão                              | Problema                                                    | Solução                                        |
| --------------------------------------- | ----------------------------------------------------------- | ---------------------------------------------- |
| **Monolito distribuído**                | Serviços "separados" mas que precisam ser deployados juntos | Garantir independência real de deploy e dados  |
| **Banco compartilhado**                 | Múltiplos serviços acessam as mesmas tabelas                | Um banco por serviço, dados via API            |
| **Comunicação síncrona em cadeia**      | A→B→C→D: latência acumulada, fragilidade                    | Usar eventos assíncronos onde possível         |
| **Nano-serviços**                       | Serviços pequenos demais (uma função cada)                  | Agrupar por domínio de negócio, não por função |
| **Sem observabilidade**                 | Impossível depurar problemas em produção                    | Logs + métricas + tracing desde o início       |
| **Sem health checks**                   | Orquestrador não sabe se o serviço está saudável            | Expor `/health` em todo serviço                |
| **Compartilhar bibliotecas de domínio** | Acoplamento via código compartilhado                        | Cada serviço tem suas próprias entidades       |
| **Ignorar idempotência**                | Retentativas duplicam efeitos colaterais                    | Usar IDs de requisição e verificar duplicatas  |
| **Adoção prematura**                    | Microsserviços para 2 desenvolvedores e 3 telas             | Comece com monolito bem estruturado            |

> **Ponto de atenção:** O antipadrão mais perigoso é o **monolito distribuído** — quando os serviços estão fisicamente separados mas logicamente acoplados. Os sintomas incluem: "preciso alterar 3 serviços para adicionar um campo" ou "não consigo fazer deploy do serviço A sem atualizar o serviço B ao mesmo tempo."

---

## 16. Referências

1. **FOWLER, Martin. Microservices.** martinfowler.com, 2014. Disponível em: https://martinfowler.com/articles/microservices.html
2. **FOWLER, Martin. MonolithFirst.** martinfowler.com, 2015. Disponível em: https://martinfowler.com/bliki/MonolithFirst.html
3. **NEWMAN, Sam. Building Microservices: Designing Fine-Grained Systems.** 2ª ed. O'Reilly Media, 2021.
4. **RICHARDSON, Chris. Microservices Patterns.** Manning Publications, 2018. Disponível em: https://microservices.io/patterns/
5. **EVANS, Eric. Domain-Driven Design: Tackling Complexity in the Heart of Software.** Addison-Wesley, 2003.
6. **Docker Documentation — Get Started.** Disponível em: https://docs.docker.com/get-started/
7. **Docker Documentation — Compose file reference.** Disponível em: https://docs.docker.com/compose/compose-file/
8. **RFC 7519 — JSON Web Token (JWT).** IETF, 2015. Disponível em: https://datatracker.ietf.org/doc/html/rfc7519
9. **OPOSSUM — Circuit Breaker for Node.js.** Disponível em: https://nodeshift.dev/opossum/
10. **OpenTelemetry Documentation.** Disponível em: https://opentelemetry.io/docs/
11. **ByteByteGo — System Design Concepts.** Disponível em: https://bytebytego.com/
12. **COULOURIS, George et al. Sistemas Distribuídos: Conceitos e Projeto.** 5ª ed. Porto Alegre: Bookman, 2013.
13. **Fastify — Fast and low overhead web framework for Node.js.** Documentação oficial. Disponível em: https://fastify.dev/docs/latest/
14. **@fastify/http-proxy — Proxy your HTTP requests to another server.** Disponível em: https://github.com/fastify/fastify-http-proxy
15. **better-sqlite3 — The fastest and simplest library for SQLite3 in Node.js.** Disponível em: https://github.com/WiseLibs/better-sqlite3

---

> **Nota:** Este guia foi elaborado como material de apoio para a disciplina de Serviços Web. O conteúdo é baseado nas fontes técnicas citadas e de material de apoio e notas de aula legadas do professo. Para uso em produção, consulte sempre a documentação oficial e atualizada das tecnologias utilizadas. A criação do material foi apoiada por uso de IA, tendo esta sido utilizada como ferramenta de apoio ao processo de escrita e enriquecimento de exemplos, não como autora do conteúdo. Todo o artefato gerado foi revisado e ampliado pelo professor para atender às necessidades pedagógicas. O uso da IA se deu para proporcionar a criação material mais rico e didático.
