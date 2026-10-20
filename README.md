# Atividade: Análise Crítica de Arquitetura de Microsserviços — Cenário StreamFlow

**Disciplina:** Serviços Web  
**Professor:** Élder F. F. Bernardi  
**Prazo de entrega do report:** 03/11/2026 (quarta)  
**Modalidade:** Dupla ou individual

---

## Contexto

Você é **Arquiteto(a) de Sistemas** na **StreamFlow**, uma plataforma de streaming de vídeo que opera com 7 microsserviços em produção. A equipe de engenharia tem 15 pessoas.

Nas últimas semanas, uma série de eventos colocou a arquitetura sob escrutínio:

- Os custos de infraestrutura em nuvem aumentaram 40% nos últimos 3 meses, sem crescimento proporcional na base de usuários.
- Dois tech leads da equipe de plataforma deixaram a empresa. O conhecimento sobre o service mesh e o sistema de tracing distribuído saiu com eles.
- Nas últimas 2 semanas, ocorreram 3 incidentes graves em que a equipe não conseguiu identificar a origem da falha, nem o impacto real (tempo de indisponibilidade, custo financeiro, número de usuários afetados).

O **Gerente de Produto**, Marcos Oliveira, assistiu a uma apresentação sobre os problemas de microsserviços em escala e enviou o seguinte e-mail à equipe.

---

### O e-mail do Gerente de Produto

> **De:** Marcos Oliveira (VP de Produto)  
> **Para:** Equipe de Engenharia  
> **Assunto:** Precisamos conversar sobre a nossa arquitetura — urgente
>
> Pessoal,
>
> Vocês sabem que eu não sou de ficar alarmista, mas os últimos 3 meses me deixaram preocupado. Vou ser direto:
>
> 1. **Custos:** A fatura da AWS subiu 40% desde janeiro. Estou tendo que justificar isso para a diretoria e não consigo. Temos 7 microsserviços rodando — cada um com seu container, seu banco, seu deploy pipeline, seu monitoramento. Será que tudo isso é necessário?
> 2. **Equipe:** O Rafael e a Camila saíram. Eles eram os únicos que entendiam o tracing distribuído e o service mesh. Agora, quando algo quebra, a equipe leva horas para descobrir onde. Na última sexta, um cliente Premium ficou sem conseguir assistir nada por 45 minutos e ninguém sabia se o problema era no streaming, no catálogo ou na autenticação.
> 3. **Observabilidade:** Falando nisso — tivemos 3 incidentes nas últimas 2 semanas. Em nenhum deles conseguimos responder com confiança: "o problema foi AQUI, durou TANTO, custou TANTO e afetou TANTOS usuários." Isso é inaceitável para uma plataforma de streaming.
> 4. **O caso do concorrente:** Eu assisti a uma apresentação que mostra como a equipe do **Prime Video** consolidou um pipeline de microsserviços e reduziu custos em 90%. Noventa por cento. Eles tinham exatamente os nossos problemas — pipeline sequencial, custos de infraestrutura, complexidade de operação — e resolveram consolidando.
>
> Preciso que vocês analisem a nossa arquitetura com olho crítico e me entreguem um **report completo até sexta-feira**. Quero saber:
>
> - O que está certo e deve ser mantido
> - O que está errado e precisa ser corrigido
> - O que precisa mudar de rumo
>
> Sem jargão técnico gratuito — me convençam com fatos, dados e argumentos claros.
>
> O vídeo que mencionei está disponível como material de apoio para a análise.
>
> Marcos

---

## O que você deve fazer

### Material de apoio

1. **Projeto StreamFlow:** Repositório com o código-fonte da arquitetura de microsserviços (instruções de setup em `docs/SETUP.md`)
2. **Especificação arquitetural:** `arquitetura_streamflow_reference.md` — documentação técnica dos serviços, fluxos e bounded contexts
3. **Vídeo de referência:** _"Microservices at Scale"_ — análise crítica da adoção de microsserviços ([Microservices at Scale: Engineering Debt and System Complexity](https://www.youtube.com/watch?v=ILXcsNxbas4)) -
4. **Guia de debugging:** `docs/DEBUG.md` — cenários de teste para observar os problemas na prática

### Atividade prática obrigatória

**Antes de redigir o report**, execute o projeto localmente com Docker Compose e realize os cenários de teste descritos em `docs/DEBUG.md`. Registre suas observações — elas servirão de evidência para o report.

### Estrutura do Report

O report deve ser estruturado em **6 seções**, cada uma respondendo a uma objeção do Gerente de Produto. Para cada seção, você deve:

1. **Reconhecer** o problema levantado pelo gerente (ele está certo? parcialmente? errado?)
2. **Diagnosticar** a causa raiz na arquitetura do StreamFlow
3. **Propor** uma ação concreta (manter, corrigir ou mudar)

---

#### Seção 1 — Custo Operacional Oculto

> _"Temos mais gente mantendo infraestrutura do que escrevendo features."_

- Quantos serviços o StreamFlow possui? Todos são necessários como microsserviços independentes?
- Identifique serviços que poderiam ser consolidados ou eliminados, justificando com base na arquitetura.
- Calcule (de forma estimada) o overhead operacional: quantos Dockerfiles, pipelines de CI/CD, bancos de dados e configs de rede a equipe precisa manter?

---

#### Seção 2 — Latência de Rede

> _"Um clique no Play toca 12 serviços antes do vídeo iniciar."_

- Analise o fluxo de `POST /streaming/play` no diagrama arquitetural e no código. Quantos hops síncronos ocorrem?
- Estime a latência adicionada pela cadeia síncrona (inclua a latência do notification-service).
- Proponha alternativas para reduzir a cadeia, indicando quais chamadas poderiam ser assíncronas.

---

#### Seção 3 — Observabilidade Frágil

> _"Tivemos 3 incidentes e não sabemos nem onde foi."_

- Analise o sistema de logging do StreamFlow. Ele permite rastrear uma requisição do gateway até o último serviço da cadeia?
- Identifique o que falta para que a equipe consiga responder: "o problema foi AQUI, durou TANTO, afetou TANTOS."
- Proponha uma estratégia de observabilidade (logging, tracing, métricas) com ferramentas concretas.

---

#### Seção 4 — Consistência Eventual

> _"Recomendações mostrando conteúdo que já foi removido."_

- Onde no sistema um dado pode ficar "desatualizado" entre serviços?
- Analise o bounded context do Catalog vs. Recommendation: o que acontece se um filme for removido do catálogo mas ainda constar no histórico de recomendações?
- Defina qual nível de consistência é aceitável para cada domínio (forte vs. eventual).

---

#### Seção 5 — Lei de Conway

> _"Mesma equipe de 15 pessoas operando 7 serviços."_

- A Lei de Conway afirma que a arquitetura de um sistema reflete a estrutura da organização que o produz. 15 pessoas para 7 serviços — essa proporção faz sentido?
- Após a saída dos 2 tech leads, como ficou a distribuição de conhecimento?
- Proponha uma reorganização realista (equipe vs. serviços).

---

#### Seção 6 — O Caso Prime Video

> _"O concorrente fez melhor com menos."_

- Qual serviço do StreamFlow tem perfil mais similar ao pipeline que o Prime Video consolidou?
- Analise o analytics-service: ele se beneficia de ser um microsserviço independente? Justifique.
- Identifique se há outros serviços candidatos à consolidação e explique por quê.

---

## Tarefa Bônus (+15%)

Após receber seu report, o gerente responde:

> _"Bom trabalho. Agora quero entender se existe um framework que resolva parte desses problemas de infraestrutura automaticamente. Alguém mencionou o Moleculer.js — pesquise e me diga: o que ele resolve de graça, o que ele esconde, e se vale a pena migrar."_

### Entrega bônus

Em uma seção adicional do report (1-2 páginas), analise o **Moleculer.js** como framework de microsserviços para Node.js:

1. **O que o Moleculer resolve nativamente:** service discovery, circuit breaker, retry, timeout, fallback, load balancing, tracing, métricas
2. **O que o Moleculer esconde/abstrai:** e por que isso pode ser perigoso — conecte com o argumento do vídeo sobre complexidade oculta
3. **Recomendação:** migrar integralmente, adotar parcialmente, ou não migrar — justifique
4. **(Opcional)** Reescreva 1 serviço do StreamFlow usando Moleculer como prova de conceito

Documentação do Moleculer: https://moleculer.services/docs/0.14/

---

## Critérios de Avaliação

| Critério                | Peso     | Descrição                                                                                   |
| ----------------------- | -------- | ------------------------------------------------------------------------------------------- |
| Compreensão do vídeo    | 20%      | Demonstra ter assistido e compreendido os argumentos centrais da apresentação               |
| Análise da arquitetura  | 25%      | Identifica os problemas arquiteturais no StreamFlow com base no código e na especificação   |
| Qualidade das respostas | 25%      | Respostas fundamentadas, com referência ao material de apoio e evidências dos testes locais |
| Propostas de melhoria   | 20%      | Soluções concretas, viáveis e bem justificadas para os problemas identificados              |
| Apresentação oral       | 10%      | Clareza, objetividade e capacidade de defesa técnica durante a apresentação                 |
| **Bônus — Moleculer**   | **+15%** | Análise comparativa com framework nativo de microsserviços                                  |

---

## Formato e entrega

- **Report:** documento em Markdown (.md) ou PDF
- **Evidências dos testes:** capturas de tela dos logs, tempos de resposta medidos, ou trechos relevantes
- **Entrega:** via repositório Git (incluir o report na raiz do projeto forkado) e postar no classroom.
- **Apresentação:** 10-15 minutos de defesa + 5 minutos de perguntas

---

## Dicas

- **Comece fazendo um fork deste repositório.** Vai ter tudo que precisa e liberdade para alterar e escrever o que for necessário.
- **Execute o projeto antes de escrever.** Os cenários de teste do `docs/DEBUG.md` revelam evidências concretas que fortalecem seu report.
- **Não há gabarito fechado.** A atividade avalia sua capacidade de análise crítica, não a memorização de respostas certas.
- **Contexto importa.** Considere o tamanho da equipe (15 pessoas), o nível de maturidade operacional (tracing perdido com a saída dos tech leads) e o momento da empresa (custos em alta).
- **O gerente não é técnico.** Escreva de forma que um gestor consiga entender suas conclusões e tomar decisões com base nelas.
