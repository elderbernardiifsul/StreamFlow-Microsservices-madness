# Atividade: Análise Crítica de Arquitetura de Microsserviços — Cenário StreamFlow

**Disciplina:** Serviços Web  
**Professor:** Élder F. F. Bernardi  
**Prazo de entrega do relatório:** 04/11/2026 (quarta-feira)

**Modalidade:** Individual ou dupla
**Instituição:** IFSul — Campus Passo Fundo
**Revisão:** 20/10/2026

## Em resumo: o que você precisa fazer

Sua tarefa é investigar a arquitetura da StreamFlow e explicar **quais problemas existem, suas causas e consequências**, propondo decisões justificadas. A entrega obrigatória é um relatório crítico com evidências e defesa oral; não é necessário reescrever o sistema.

1. **Estude:** leia e compreenda o guia obrigatório ([HTML no navegador](https://elderbernardiifsul.github.io/StreamFlow-Microsservices-madness/guia_completo_microsservicos.html) ou [Markdown na IDE](docs/guia_completo_microsservicos.md)); consulte a arquitetura e o vídeo indicados abaixo.
2. **Prepare:** faça o fork, siga o [SETUP](docs/SETUP.md), execute o laboratório e registre a versão analisada.
3. **Investigue:** percorra o código e formule hipóteses sobre os comportamentos que pretende observar.
4. **Experimente:** realize os cenários do [DEBUG](docs/DEBUG.md), registrando comandos, respostas HTTP, tempos e logs; restaure o ambiente entre os testes.
5. **Analise:** relacione as evidências aos conceitos do guia e diferencie observações, hipóteses e premissas fictícias. Discuta causas, consequências e alternativas.
6. **Entregue e defenda:** organize o relatório nas seis seções exigidas, inclua evidências e referências, publique no seu fork e envie o link no Classroom. Prepare a defesa oral das decisões.

Os requisitos detalhados, o protocolo de medições e a rubrica estão nas seções seguintes.

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
> 1. **Custos:** A fatura da AWS subiu 40% nos últimos três meses. Estou tendo que justificar isso para a diretoria e não consigo. Temos 7 microsserviços rodando — cada um com seu container, seu banco, seu deploy pipeline, seu monitoramento. Será que tudo isso é necessário?
> 2. **Equipe:** O Rafael e a Camila saíram. Eles eram os únicos que entendiam o tracing distribuído e o service mesh. Agora, quando algo quebra, a equipe leva horas para descobrir onde. Na última sexta, um cliente Premium ficou sem conseguir assistir nada por 45 minutos e ninguém sabia se o problema era no streaming, no catálogo ou na autenticação.
> 3. **Observabilidade:** Falando nisso — tivemos 3 incidentes nas últimas 2 semanas. Em nenhum deles conseguimos responder com confiança: "o problema foi AQUI, durou TANTO, custou TANTO e afetou TANTOS usuários." Isso é inaceitável para uma plataforma de streaming.
> 4. **O caso do concorrente:** Eu assisti a uma apresentação que mostra como a equipe do **Prime Video** consolidou um pipeline de microsserviços e reduziu custos em 90%. Noventa por cento. Eles resolveram o problema consolidando componentes. Quero verificar se as condições desse caso são comparáveis às nossas: fluxo de processamento, custos de infraestrutura e complexidade de operação.
>
> Preciso que vocês analisem a nossa arquitetura com olho crítico e me entreguem um **relatório completo no prazo indicado no início da atividade**. Quero saber:
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

## Objetivos e limites da análise

Ao concluir, você deverá conseguir mapear chamadas HTTP entre serviços, avaliar acoplamento e falhas parciais, interpretar evidências de execução e justificar decisões arquiteturais.

A StreamFlow é uma **simulação didática com antipadrões intencionais**. Custos, incidentes, equipe e saída dos profissionais pertencem ao cenário fictício. O repositório não contém uma fatura de nuvem, service mesh ou pipelines de CI/CD que comprovem as falas do gerente. Trate essas falas como alegações a investigar.

Diferencie, em cada conclusão: **observação no código**, **resultado medido**, **premissa do cenário** e **proposta**. Não invente medições, custos ou componentes ausentes. A aplicação simula sessões de reprodução; não transmite vídeo real. A entrega obrigatória é a análise: não é necessário reescrever os serviços ou eliminar os antipadrões.

## O que você deve fazer

### Leitura obrigatória: guia de microsserviços e Docker

A **leitura e a compreensão do Guia Completo de Arquitetura de Microsserviços e Containerização com Docker fazem parte obrigatória desta atividade**. O guia fornece os conceitos que devem fundamentar o diagnóstico, as propostas e a defesa oral; não basta executar os comandos do laboratório.

Escolha o formato de leitura — ambos contêm o mesmo material:

- **[Ler no navegador — versão HTML](https://elderbernardiifsul.github.io/StreamFlow-Microsservices-madness/guia_completo_microsservicos.html):** página responsiva, com sumário navegável, tabelas e exemplos de código formatados, para computador ou celular.
- **[Ler na IDE — versão Markdown (.md)](docs/guia_completo_microsservicos.md):** abra o arquivo no editor e use a pré-visualização Markdown. Também pode ser lido diretamente no GitHub.
- **[Arquivo HTML para leitura local](docs/guia_completo_microsservicos.html):** baixe o arquivo e abra-o no navegador; o conteúdo e a estilização funcionam sem conexão.

Leia o guia antes de concluir o relatório e retome as seções relevantes durante os experimentos. Ao justificar uma decisão, relacione o conceito estudado à evidência obtida na StreamFlow e indique a seção consultada. Na defesa oral, esteja preparado para explicar essa relação com suas próprias palavras. Essa compreensão será observada nos critérios de análise da arquitetura e qualidade das respostas já previstos na rubrica.

### Demais materiais e roteiro prático

1. **Projeto StreamFlow:** Repositório com o código-fonte da arquitetura de microsserviços (instruções de setup em `docs/SETUP.md`)
2. **Especificação arquitetural:** `arquitetura_streamflow_reference.md` — documentação técnica dos serviços, fluxos e bounded contexts
3. **Vídeo de referência:** _"Microservices at Scale"_ — análise crítica da adoção de microsserviços ([Microservices at Scale: Engineering Debt and System Complexity](https://www.youtube.com/watch?v=ILXcsNxbas4)). Registre os trechos utilizados com marcação de tempo; confronte argumentos com o código e com fontes primárias.
4. **Guia de debugging:** `docs/DEBUG.md` — cenários de teste para observar os problemas na prática

### Atividade prática obrigatória

**Antes de redigir o relatório**, execute o projeto localmente com Docker Compose e realize os cenários de teste descritos em `docs/DEBUG.md`. Registre suas observações — elas servirão de evidência para o relatório. Siga o roteiro atualizado em `docs/DEBUG.md` e restaure cada serviço antes do cenário seguinte.

Para cada cenário entregue: comando, condição inicial, código HTTP, resposta, tempo observado, trecho de log e interpretação. Identifique o commit analisado e as versões de Docker/Compose. Faça uma chamada de aquecimento e pelo menos cinco medições por condição, informando mediana e intervalo mínimo–máximo. Essa amostra é exploratória, não um teste de capacidade de produção. Remova tokens das evidências.

Se houver falha de preparação, registre o erro e os passos tentados. Distingua esse bloqueio de um comportamento arquitetural observado; não apresente como executado um cenário apenas previsto.

### Estrutura do Relatório

O relatório deve ser estruturado em **6 seções**, cada uma respondendo a uma objeção do Gerente de Produto. Para cada seção, você deve:

1. **Reconhecer** o problema levantado pelo gerente (ele está certo? parcialmente? errado?)
2. **Diagnosticar** a causa raiz na arquitetura do StreamFlow
3. **Propor** uma ação concreta (manter, corrigir ou mudar)

---

#### Seção 1 — Custo Operacional Oculto

> _"Temos mais gente mantendo infraestrutura do que escrevendo features."_

- Quantos serviços o StreamFlow possui? Todos são necessários como microsserviços independentes?
- Identifique serviços que poderiam ser consolidados ou eliminados, justificando com base na arquitetura.
- Faça um inventário de Dockerfiles, processos, arquivos SQLite, volumes e configurações de rede. Conte apenas os artefatos existentes; marque pipelines ausentes como “não implementados”. Para estimar esforço ou custo, explicite premissas e unidades, sem atribuir valores financeiros ao laboratório como se fossem medições.

---

#### Seção 2 — Latência de Rede

> _"Um clique no Play toca 12 serviços antes do vídeo iniciar."_

- Analise o fluxo de `POST /streaming/play` no diagrama arquitetural e no código. Quantas chamadas HTTP ocorrem no caminho de uma reprodução bem-sucedida? Declare se está contando cliente → gateway, gateway → serviço e chamadas internas; diferencie chamadas, serviços únicos e etapas locais. Verifique a alegação dos “12 serviços”.
- Meça a latência total e identifique os atrasos simulados do notification-service. A diferença entre catálogo e play inclui processamento e operações distintas: não a atribua integralmente à rede. Diferencie conexão recusada, resposta HTTP de erro e timeout.
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
- Declare a regra de negócio que precisa ser preservada e o atraso tolerável por domínio. O histórico contém IDs de filmes e não há um fluxo completo de remoção e reconciliação: trate a remoção como hipótese de projeto, salvo se implementar e demonstrar um experimento adicional. Consistência eventual exige um mecanismo que faça os dados convergirem; dados permanentemente desatualizados não bastam.

---

#### Seção 5 — Lei de Conway

> _"Mesma equipe de 15 pessoas operando 7 serviços."_

- A Lei de Conway afirma que a arquitetura de um sistema reflete a estrutura da organização que o produz. Quais informações sobre responsabilidades, autonomia e comunicação seriam necessárias para julgar essa divisão? A proporção pessoas/serviços, isoladamente, não determina uma arquitetura adequada.
- Após a saída dos 2 tech leads, como ficou a distribuição de conhecimento?
- Proponha uma reorganização realista (equipe vs. serviços).

---

#### Seção 6 — O Caso Prime Video

> _"O concorrente fez melhor com menos."_

- Consulte o relato de Marcin Kolny (2023), sobre um serviço específico de monitoramento de qualidade de áudio/vídeo. A redução relatada de 90% não representa uma migração de toda a plataforma Prime Video. Compare os mecanismos de custo e comunicação antes de transferir conclusões para a StreamFlow.
- Analise o analytics-service: suas etapas já executam dentro de um único processo. Sequência de operações não é, por si só, um antipadrão. Avalie separação por responsabilidade, necessidade de escala, compartilhamento de banco e custo operacional; justifique manter ou consolidar.
- Identifique se há outros serviços candidatos à consolidação e explique por quê.

---

## Tarefa Bônus (+15%)

Após receber seu relatório, o gerente responde:

> _"Bom trabalho. Agora quero entender se existe um framework que resolva parte desses problemas de infraestrutura automaticamente. Alguém mencionou o Moleculer.js — pesquise e me diga: quais recursos oferece e quais custos de configuração e operação permanecem, o que ele esconde, e se vale a pena migrar."_

### Entrega bônus

Em uma seção adicional do relatório (1-2 páginas), analise o **Moleculer.js** como framework de microsserviços para Node.js:

1. **O que o Moleculer resolve nativamente:** service discovery, circuit breaker, retry, timeout, fallback, load balancing, tracing, métricas
2. **O que o Moleculer esconde/abstrai:** e por que isso pode ser perigoso — conecte com o argumento do vídeo sobre complexidade oculta
3. **Recomendação:** migrar integralmente, adotar parcialmente, ou não migrar — justifique
4. **(Opcional)** Reescreva 1 serviço do StreamFlow usando Moleculer como prova de conceito

Documentação de referência atualizada: https://moleculer.services/docs/0.15/ . A documentação 0.14 está marcada como encerrada. Informe a versão consultada e a data de acesso; diferencie recurso disponível de recurso habilitado/configurado. Discuta idempotência ao propor retries e garantias de entrega ao propor eventos.

O bônus continua sendo de até 15% da pontuação-base. O teto da nota e o peso desta atividade na etapa serão os informados pelo professor no Classroom.

---

## Critérios de Avaliação

| Critério                | Peso     | Descrição                                                                                   |
| ----------------------- | -------- | ------------------------------------------------------------------------------------------- |
| Compreensão do vídeo    | 20%      | Examina argumentos do vídeo, cita trechos e confronta generalizações com fontes e evidências               |
| Análise da arquitetura  | 25%      | Identifica os problemas arquiteturais no StreamFlow com base no código e na especificação   |
| Qualidade das respostas | 25%      | Respostas fundamentadas, com referência ao material de apoio e evidências dos testes locais |
| Propostas de melhoria   | 20%      | Soluções concretas, viáveis e bem justificadas para os problemas identificados              |
| Apresentação oral       | 10%      | Clareza, objetividade e capacidade de defesa técnica durante a apresentação                 |
| **Bônus — Moleculer**   | **+15%** | Análise comparativa com framework nativo de microsserviços                                  |

---

## Formato e entrega

- **Relatório:** documento em Markdown (.md) ou PDF, com resumo executivo de até 300 palavras, as seis seções, tabela de decisões priorizadas e referências. Na tabela, indique ação, evidência, benefício esperado, custo/risco e como validar o resultado.
- **Evidências dos testes:** capturas de tela dos logs, tempos de resposta medidos, ou trechos relevantes
- **Entrega:** via repositório Git (incluir o relatório na raiz do projeto forkado) e postar no classroom.
- **Apresentação:** 10-15 minutos de defesa + 5 minutos de perguntas

---

## Dicas

- **Comece fazendo um fork deste repositório.** Vai ter tudo que precisa e liberdade para alterar e escrever o que for necessário.
- **Execute o projeto antes de escrever.** Os cenários de teste do `docs/DEBUG.md` revelam evidências concretas que fortalecem seu relatório.
- **Não há gabarito fechado.** A atividade avalia sua capacidade de análise crítica, não a memorização de respostas certas.
- **Contexto importa.** Considere o tamanho da equipe (15 pessoas), o nível de maturidade operacional (tracing perdido com a saída dos tech leads) e o momento da empresa (custos em alta).
- **O gerente não é técnico.** Escreva de forma que um gestor consiga entender suas conclusões e tomar decisões com base nelas.

## Referências complementares

- [Prime Video — relato original preservado em PDF](https://www.wudsn.com/productions/www/site/news/2023/2023-05-08-microservices-01.pdf): Marcin Kolny, 22/03/2023; cópia do artigo de engenharia, hospedada por terceiro.
- [Calendário oficial de versões do Node.js](https://github.com/nodejs/Release#release-schedule).
- [Moleculer 0.15](https://moleculer.services/docs/0.15/).

Documentação e comentários do projeto também devem ser avaliados criticamente. Uma divergência demonstrada entre documentação e execução é uma evidência válida.

## Apoio de agentes de IA

Este repositório inclui um [AGENTS.md](AGENTS.md) que orienta o agente a atuar como tutor de observabilidade e microsserviços. Use esse apoio para formular hipóteses, planejar experimentos e discutir suas evidências. A interpretação dos resultados, as decisões e a defesa do relatório continuam sendo responsabilidade do aluno.
