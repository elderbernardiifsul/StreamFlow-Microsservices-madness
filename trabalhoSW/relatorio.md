# Relatório — Análise Crítica da Arquitetura de Microsserviços da StreamFlow

**Disciplina:** Serviços Web · Prof. Élder F. F. Bernardi · IFSul — Campus Passo Fundo
**Autoria:** helenzn
**Commit analisado:** `ea0c63bc097304d87e7fcb8fa690d7d13d6c8ffa`
**Ambiente:** Windows, Windows PowerShell, Docker 29.9.2, Docker Compose v5.6.1
**Evidências completas:** [`evidencias.md`](evidencias.md) (registros E01–E06, A01–A03, B01)

> **Como ler este relatório.** Cada conclusão indica a origem da informação:
> **[medido]** = resultado de teste executado · **[código]** = observado no código-fonte · **[premissa]** = dado do cenário fictício, não verificável no repositório · **[proposta]** = recomendação.

---

## Resumo executivo

A StreamFlow tem problemas reais, mas não exatamente os que o e-mail descreve. Executei o sistema localmente e li o código dos 8 componentes (gateway + 7 serviços).

**O que está certo e deve ser mantido:** o gateway como porta única de entrada, a autenticação centralizada e a verificação de licença antes de cada reprodução, que protege uma regra de negócio.

**O que está errado e precisa ser corrigido:**
1. **O alarme de saúde está quebrado:** responde "degradado" mesmo com tudo funcionando. Um alarme que sempre toca acaba ignorado.
2. **Não há como seguir uma requisição entre os serviços.** Por isso a equipe não consegue dizer onde foi o problema.
3. **O Play espera a notificação terminar.** Quando o serviço de notificação fica lento, cada Play passa de 0,24 s para 5,07 s, cerca de **21 vezes mais**, sem que nenhum erro apareça.

**O que precisa mudar de rumo:**
4. O Analytics lê diretamente o banco do Billing: uma mudança no Billing quebra os relatórios sem aviso. Proponho incorporar o Analytics ao Billing.
5. A notificação deve sair do caminho do Play e virar processamento em fila.

**Sobre as alegações do e-mail:** o Play passa por 4 serviços além do gateway (5 chamadas), não por 12. O repositório não contém pipelines de deploy, service mesh nem tracing; essas afirmações e os números de custo são premissas do cenário. O caso Prime Video não se transfere diretamente: lá a economia veio de uma orquestração cobrada por etapa e da troca de quadros de vídeo por armazenamento externo, mecanismos que a StreamFlow não usa.

As três correções prioritárias têm custo baixo e podem ser validadas repetindo os mesmos testes deste relatório.

---

## O vídeo "Microservices at Scale" diante das evidências

| Trecho | Argumento do vídeo | O que a StreamFlow mostra | Avaliação |
|---|---|---|---|
| 0:04–1:30 | Microsserviços surgiram para dar escala e autonomia a equipes | Há 1 instância de cada serviço e nenhum dado sobre autonomia dos times | Os benefícios prometidos não aparecem no laboratório |
| 1:30–5:30 | A distribuição exige Kubernetes, service mesh e equipes de infraestrutura | Nada disso existe no repositório (A02) | Alerta válido, mas não descreve a StreamFlow atual |
| 5:30–8:50 | Latência, depuração difícil e consistência de dados | Confirmado: E05 (5,07 s), A03 (sem rastreamento), A03 (IDs sem reconciliação) | **Confirmado pelas evidências** |
| 8:50–11:12 | Conway: sem mudar a organização, o gargalo só muda de lugar | O cenário não traz os dados necessários | Plausível, não verificável aqui |
| 11:12–14:48 | Prime Video voltou ao "monólito" e economizou 90% | Fonte primária: um único serviço, mecanismos de custo ausentes na StreamFlow | **Generalização exagerada** |
| 14:48–16:09 | A melhor arquitetura é a que a organização consegue operar | Coerente com a recomendação deste relatório | Concordo |

**Síntese:** o vídeo acerta no diagnóstico dos custos da distribuição, e três deles foram medidos aqui. Ele erra quando trata a consolidação como solução geral. Na StreamFlow, os problemas mais graves vêm de decisões de desenho dentro dos microsserviços (chamada síncrona dispensável, health check quebrado, banco compartilhado), e se corrigem sem voltar a um monólito.

---

## Seção 1 — Custo Operacional Oculto

> _"Temos mais gente mantendo infraestrutura do que escrevendo features."_

### O gerente está certo?
**Parcialmente.** Existe custo operacional que não gera funcionalidade, mas parte do que o e-mail descreve não existe no repositório.

### Diagnóstico

**Inventário [código] (A02):**

| Artefato | Quantidade |
|---|---|
| Unidades implantáveis (contêineres) | 8 (gateway + 7 serviços) |
| Dockerfiles | 8 |
| Bancos SQLite distintos | 5 (um deles compartilhado por billing e analytics) |
| Volumes nomeados | 5 |
| Redes | 1 |
| Pipelines de CI/CD | **não implementados** |
| Service mesh, tracing distribuído, monitoramento | **não implementados** |
| `healthcheck` no Compose | **não implementado** |

- **Configuração sem função [código] (A03):** 5 serviços (catalog, streaming, recommendation, billing e analytics) recebem a chave pública JWT e montam a pasta `keys/`, mas só o gateway valida tokens. É configuração mantida, revisada e distribuída sem uso.
- **Alegação "cada um com seu deploy pipeline, seu monitoramento" [premissa]:** não há pipeline nem monitoramento no repositório. O custo de nuvem de +40% não pode ser verificado aqui.
- **Guia, §2.5 "Quando NÃO adotar microsserviços" e §15 "Erros comuns e antipadrões":** a decomposição tem custo próprio e só se justifica quando traz benefício. **§3.3 "Banco de Dados por Serviço":** billing e analytics violam esse princípio ao dividir `shared_billing.db` (E04). **§4.3 "Critérios práticos para traçar fronteiras":** base para decidir quais serviços consolidar (tabela abaixo).
- **Vídeo, 1:30–5:30 ("economia oculta"):** o vídeo afirma que dividir em serviços cria um sistema distribuído que exige Kubernetes e um service mesh (Istio), e equipes inteiras só para mantê-los. **Confronto:** na StreamFlow, nada disso existe no repositório. O custo oculto observado é menor e mais concreto: 8 unidades implantáveis, configuração sem uso e um banco compartilhado. O argumento do vídeo vale como alerta, mas a StreamFlow ainda não pagou esse preço; a recomendação é justamente não começar a pagá-lo serviço por serviço.

### Candidatos à consolidação

| Serviço | Decisão | Justificativa |
|---|---|---|
| **analytics** | **Consolidar no billing** | Não tem dados próprios: lê o banco do billing diretamente (E04). Já são acoplados; separá-los só esconde o acoplamento. |
| **notification** | **Transformar em consumidor de fila** | Não tem banco, não guarda estado e é dispensável para o Play (E03, E05). Pode virar um processo leve que consome eventos, ou um módulo de outro serviço. |
| auth | Manter | Fronteira de segurança: é o único com a chave privada. |
| catalog | Manter | Dono das licenças; regra de negócio crítica no Play. |
| streaming | Manter | Núcleo do produto. |
| recommendation | Manter | Domínio próprio (personalização), com dados próprios. |

### Estimativa de esforço (com premissas explícitas)
Não há dados reais de esforço no repositório, então a estimativa é **ilustrativa**:

- **Fórmula:** esforço operacional ≈ (nº de unidades implantáveis) × (horas de manutenção por unidade por mês).
- **Premissa ilustrativa:** 4 horas-pessoa/mês por unidade (atualizar dependências, imagem base, configuração, investigar falhas).
- **Hoje:** 8 unidades × 4 h = **32 h-pessoa/mês**.
- **Após consolidar analytics no billing:** 7 unidades × 4 h = **28 h-pessoa/mês** (−12,5%). Se a notification virar módulo, 6 × 4 = 24 h (−25%).
- O valor absoluto depende da premissa; o que se sustenta é a **proporção**: cada unidade a menos reduz o esforço fixo.

### Proposta [proposta]
1. Incorporar o analytics ao billing como módulo de relatórios.
2. Tirar a notification do caminho síncrono (ver Seção 2).
3. Decidir sobre a chave pública nos 5 serviços que não a usam: **ou** validar o token também neles (defesa em profundidade, conforme o guia §7.1 "Onde validar o token?" e §7.2 "Por que RS256 é essencial"), **ou** remover a configuração sem uso. O que não deve continuar é a configuração montada e ignorada, que dá a falsa impressão de que os serviços estão protegidos.
4. **Não** criar pipeline, monitoramento e mesh separados para cada serviço: começar com um pipeline e um painel compartilhados.

---

## Seção 2 — Latência de Rede

> _"Um clique no Play toca 12 serviços antes do vídeo iniciar."_

### O gerente está certo?
**Não na quantidade, sim no risco.** O Play não toca 12 serviços, mas a cadeia síncrona é real e degrada muito quando uma dependência fica lenta.

### Diagnóstico

**Contagem das chamadas [código] (A01, A03):**

| # | Origem → Destino | Tipo |
|---|---|---|
| 1 | Cliente → gateway | externa |
| 2 | Gateway → streaming | interna |
| 3 | Streaming → catalog (licença) | interna, síncrona |
| 4 | Streaming → recommendation (registrar visualização) | interna, síncrona |
| 5 | Streaming → notification | interna, síncrona |

- **5 chamadas HTTP** (1 externa + 4 internas), passando por **4 serviços únicos** além do gateway. O gateway valida o token localmente; o auth-service **não** é chamado (A03). A gravação da sessão no banco do streaming é uma etapa local, não uma chamada.
- **Alegação dos "12 serviços": não confirmada.**
- As 3 chamadas internas são **sequenciais**, cada uma com timeout de 5 s.
- **Atraso artificial [código]:** o notification espera aleatoriamente 100–300 ms a cada chamada.

**Medições [medido] (aquecimento + 5 medições, mediana e mín–máx):**

| Condição | Mediana | Mín–Máx | Registro |
|---|---|---|---|
| GET catálogo | 0,025 s | 0,023–0,047 s | E02 |
| POST play, tudo ligado | 0,238 s | 0,194–0,359 s | E02 |
| POST play, notification **parado** | 1,727 s | 1,716–1,808 s | E03 |
| POST play, notification **congelado** | 5,070 s | 5,050–5,097 s | E05 |

- **A diferença catálogo × play (~0,21 s) não é toda rede.** O play faz 3 chamadas internas, grava no banco e inclui 100–300 ms de atraso artificial do notification. A maior parte da diferença é compatível com esse atraso.
- **Três modos de falha diferentes [medido]:**
  - **Parado:** o Docker remove o nome do serviço da rede, e o erro é `ENOTFOUND` (falha de DNS), que levou ~1,7 s. Não é conexão recusada.
  - **Congelado:** o serviço aceita a conexão e não responde, e o erro é `TimeoutError` após 5 s.
  - **Conexão recusada (`ECONNREFUSED`):** não foi observada em nenhum experimento.
  - **Resposta HTTP de erro (ex.: 500): não testada.** Pelo código [código], o streaming não verifica `response.ok` nas chamadas ao recommendation e ao notification (`streaming/index.js`, passos 2 e 3): uma resposta 500 desses serviços passaria sem nenhum registro no log. Na licença, qualquer resposta de erro do catalog vira **404 "Título não encontrado"**, mesmo que o catalog esteja com erro interno. O mesmo padrão aparece no health check do gateway, que também não verifica `response.ok` (E01).
- **Em todos os casos o Play responde 201**: a falha é silenciosa. Uma dependência **lenta** causou mais dano que uma dependência **fora do ar**.
- **Projeção [hipótese, não medida]:** com as 3 dependências lentas ao mesmo tempo, a espera passaria de 15 s.
- **Guia, §5.1 "Comunicação Síncrona" e §5.2 "Comunicação Assíncrona":** o Play usa só comunicação síncrona, inclusive para uma tarefa que não precisa de resposta imediata (a notificação). **§5.4 "Abordagem Híbrida":** fundamenta a proposta abaixo (licença síncrona, o resto em fila). **§3.5 "Tolerância a Falhas (Design for Failure)" e §14 "Padrões de Resiliência" (14.1 Circuit Breaker, 14.3 Timeout):** o único mecanismo presente é o timeout de 5 s, e o E05 mostra que, sozinho, ele não protege o usuário. **§10.4 "DNS Automático":** explica o erro `ENOTFOUND` do E03 (o nome do serviço sai do DNS do Compose quando o contêiner para).
- **Vídeo, 5:30–8:50 ("imposto da complexidade"):** o vídeo aponta a latência de rede como um dos custos da distribuição. **Confronto:** as medições confirmam o risco, mas mostram que o problema principal não é a latência da rede em si. Com tudo funcionando, o Play leva só 0,24 s. O dano grande vem do **acoplamento síncrono a uma dependência dispensável**: quando o notification fica lento, o Play vai a 5,07 s (E05). A causa é uma decisão de desenho, que se corrige sem desfazer os microsserviços.

### Proposta [proposta]

| Chamada | Decisão | Motivo |
|---|---|---|
| Licença (catalog) | **Manter síncrona**, com timeout curto (ex.: 1 s) e circuit breaker | Regra de negócio: não se pode reproduzir sem licença |
| Registrar visualização (recommendation) | **Tornar assíncrona** (evento em fila) | Alimenta recomendações; pode chegar alguns segundos depois |
| Notificação | **Tornar assíncrona** (evento em fila) | O usuário não precisa esperar a notificação para assistir |

- Com essa mudança, o caminho síncrono do Play cai de 3 para **1 chamada interna**.
- Eventos em fila exigem **fila durável** e **consumidor idempotente**, porque a entrega costuma ser "pelo menos uma vez".
- **Validação:** repetir o E05 (notification congelado); o Play deve ficar na faixa do E02 (~0,24 s).

---

## Seção 3 — Observabilidade Frágil

> _"Tivemos 3 incidentes e não sabemos nem onde foi."_

### O gerente está certo?
**Sim.** As evidências mostram que, com as ferramentas atuais, é impossível dizer onde foi, quanto durou e quantos foram afetados.

### Diagnóstico

1. **O health check mente [medido + código] (E01, E06).** O gateway responde **503 / "degraded" sempre**, com 0, 1 ou 2 serviços fora do ar. Causa: na linha 131 do `gateway/index.js`, o status `'up'` é sobrescrito pelo `'ok'` que vem do serviço, e a linha 137 compara com `'up'`.
2. **O health check fica lento durante incidentes [medido] (E06).** Ele consulta os serviços um por vez; cada serviço fora do ar acrescenta ~3 s (2 serviços fora → ~7 s).
3. **Não existe ID de rastreamento [código] (A03).** Nenhuma ocorrência de `request-id`, `correlation` ou `traceparent`. O `reqId` dos logs é um contador local de cada serviço ("req-1", "req-2"...).
4. **As linhas de erro não identificam a requisição [medido] (E03).** "Falha ao enviar notificação" não traz `reqId`. Com usuários simultâneos, não dá para ligar a falha ao Play de origem.
5. **Falhas silenciosas [medido] (E03, E05).** O Play responde 201 mesmo esperando 5 s. Um monitor que olha só erros nunca dispararia.
6. **Erro do cliente vira erro do servidor [medido] (B01).** Um cabeçalho não suportado gerou HTTP 500, o que inflaria a contagem de falhas do servidor.

- **Alegação "tracing e mesh saíram com os tech leads" [premissa]:** não existem no repositório, então não há o que recuperar; o sistema analisado nunca teve essas ferramentas.

### O que falta para responder "foi AQUI, durou TANTO, afetou TANTOS"

| Pergunta | O que falta |
|---|---|
| **Onde?** | ID de rastreamento propagado em todas as chamadas + logs com esse ID |
| **Quanto durou?** | Métricas de latência e erro ao longo do tempo, com horário de início e fim |
| **Quantos afetados?** | Contagem de usuários distintos com requisições lentas ou com erro na janela do incidente |

### Estratégia proposta [proposta]

| Camada | Ação | Ferramenta concreta |
|---|---|---|
| Health | Corrigir o bug; consultar em paralelo; registrar o motivo da falha | Correção no `gateway/index.js` (`Promise.allSettled`) |
| Tracing | Gerar e propagar o cabeçalho padrão `traceparent` | OpenTelemetry (SDK Node.js) → Jaeger ou Grafana Tempo |
| Logs | Incluir `traceId` e `userId` em toda linha de log | Pino (já usado) + Grafana Loki |
| Métricas | Taxa, erros e duração (p50/p95) por rota | Prometheus + Grafana |
| Alertas | Alertar por **latência** do Play (p95), não só por erro | Alertmanager / Grafana Alerting |

- **Ordem sugerida:** health → ID de correlação nos logs → métricas → tracing completo. Cada passo já ajuda sozinho.
- **Risco:** é infraestrutura nova para operar. Como a equipe perdeu conhecimento de plataforma [premissa], vale considerar um serviço gerenciado e documentar o uso em runbooks.
- **Validação:** repetir o E05 e verificar se (a) o alerta de latência dispara e (b) o trace mostra que o tempo foi gasto na chamada ao notification.
- **Guia, §13.1 "Os Três Pilares da Observabilidade":** a StreamFlow tem só um dos três pilares (logs), e incompleto; não tem métricas nem rastreamento. **§13.2 "Correlation ID":** é exatamente o que falta (A03); o primeiro passo da estratégia acima. **§13.3 "Health Checks":** o health check existe, mas está quebrado (E01) e é sequencial (E06).
- **Vídeo, 5:30–8:50:** o vídeo diz que depurar um sistema distribuído exige distributed tracing. **Confronto:** a StreamFlow confirma o argumento na prática. Sem nenhum ID de rastreamento (A03), não foi possível ligar a falha no notification ao Play que a causou (E03). Mas o primeiro problema encontrado é ainda mais básico que a falta de tracing: o próprio health check está errado (E01).

---

## Seção 4 — Consistência Eventual

> _"Recomendações mostrando conteúdo que já foi removido."_

### O gerente está certo?
**O risco existe no desenho, mas não foi demonstrado.** O repositório não tem rota de remoção de títulos, então o cenário é uma **hipótese de projeto**.

### Diagnóstico [código] (A03)

**Onde um dado pode ficar desatualizado:**

| Serviço | O que guarda | Consulta o catálogo? |
|---|---|---|
| recommendation | `view_history (user_id, movie_id)` | **Não** |
| streaming | sessões `(userId, movieId)` | Só no momento do Play (licença) |
| analytics | lê o banco do billing | Não se aplica: lê o dado original (acoplamento, não cópia) |

- O recommendation grava o ID do filme sem validar no catálogo e nunca mais o confere.
- **Não existe** evento, fila ou rotina de reconciliação entre catálogo e recomendações.
- **Consequência [hipótese]:** um título removido continuaria no histórico **indefinidamente**. Isso **não é consistência eventual**, porque não há mecanismo que faça os dados convergirem.
- **O que está certo [código]:** o Play consulta a licença no catálogo de forma síncrona. Mesmo que uma recomendação desatualizada apareça, o usuário **não consegue reproduzir** um título sem licença.

### Regra de negócio e atraso tolerável [proposta]

| Domínio | Regra | Atraso tolerável |
|---|---|---|
| Reprodução | Nunca reproduzir título sem licença ativa | **Zero** (consistência forte, já garantida pela verificação síncrona) |
| Recomendação | Não recomendar título removido | **Minutos** (ex.: até 15 min, valor a validar com o negócio) |
| Histórico | O usuário assistiu: é um fato, pode continuar registrado | Não se aplica (não deve ser apagado, só excluído das recomendações) |

### Mecanismo de convergência [proposta]
1. **Evento:** o catalog publica "título removido" numa fila durável quando uma licença acaba. O recommendation consome e exclui o título das recomendações.
2. **Publicação confiável:** gravar o evento na mesma transação da remoção (padrão *outbox*), para não perder eventos.
3. **Reconciliação periódica:** um job compara os IDs recomendados com o catálogo e corrige divergências (rede de segurança se um evento se perder).
4. **Filtro na leitura:** como defesa adicional, filtrar recomendações por títulos ativos.

- **Validação:** implementar uma rota de remoção, remover um título e medir o tempo até ele sumir das recomendações; deve ficar dentro do atraso tolerável.
- **Guia, §5.3 "Consistência Eventual vs. Imediata":** a reprodução precisa de consistência imediata (licença) e a recomendação tolera consistência eventual, desde que exista um mecanismo de convergência. **§3.3 "Banco de Dados por Serviço":** cada serviço guarda seus próprios dados, por isso as cópias de IDs entre catálogo e recomendações precisam ser reconciliadas. **§4.1 "Decomposição por Domínio de Negócio (DDD)":** catálogo e recomendação são contextos diferentes, ligados só pelo ID do filme.

---

## Seção 5 — Lei de Conway

> _"Mesma equipe de 15 pessoas operando 7 serviços."_

### O gerente está certo?
**A proporção sozinha não diz nada.** 15 pessoas para 7 serviços pode ser adequado ou não, dependendo de como a equipe está organizada.

### Diagnóstico
A Lei de Conway (Conway, 1968) diz que um sistema tende a refletir a estrutura de comunicação da organização que o constrói. Para julgar a divisão atual, seriam necessárias informações que o cenário **não fornece**:

- **Responsabilidade:** quem é dono de cada serviço? Quem atende quando ele cai?
- **Autonomia:** cada grupo consegue alterar e publicar seu serviço sem coordenar com os outros?
- **Comunicação:** quais grupos precisam conversar para entregar uma funcionalidade (ex.: o Play envolve streaming, catalog, recommendation e notification)?
- **Carga cognitiva:** quantos serviços cada pessoa precisa entender para trabalhar?

**Distribuição de conhecimento após a saída dos tech leads [premissa]:** o conhecimento de plataforma estava concentrado em duas pessoas (risco de "fator ônibus"). O repositório não contém mesh nem tracing, então o que se perdeu não pode ser verificado aqui; o problema central é o conhecimento concentrado, não a ferramenta.

- **Vídeo, 8:50–11:12 ("problema organizacional"):** o vídeo argumenta que, se a estrutura de comunicação da empresa não mudar, os microsserviços só transferem o gargalo do código para a coordenação entre equipes. **Confronto:** o argumento se aplica, mas o cenário não traz os dados para verificá-lo (quem é dono de cada serviço, quem fala com quem). Por isso a proposta abaixo começa pela organização, e não pela contagem de pessoas por serviço.
- **Guia, §3.2 "Autonomia e Independência de Deploy" e §3.6 "Descentralização":** os benefícios dos microsserviços dependem de times que publiquem seus serviços sem coordenar com os outros, e é isso que precisa ser verificado na equipe. **§4.1 "Decomposição por Domínio de Negócio (DDD)":** base para alinhar os times aos domínios na proposta abaixo.

### Reorganização proposta [proposta]
Times alinhados a domínios de negócio (Skelton & Pais, *Team Topologies*), com a arquitetura consolidada da Seção 1:

| Time | Pessoas | Serviços |
|---|---|---|
| **Reprodução** | 5 | streaming, catalog, notification (consumidor de fila) |
| **Descoberta** | 4 | recommendation |
| **Conta e Receita** | 4 | auth, billing (com relatórios do analytics incorporados) |
| **Plataforma (apoio)** | 2 | gateway, observabilidade, pipeline compartilhado |

- **Contra a concentração de conhecimento:** rodízio de plantão entre os times, runbooks escritos para cada serviço e pareamento nas mudanças de infraestrutura.
- **Validação:** simulação de incidente (ex.: repetir o E05) atendida por uma pessoa que não seja da plataforma, medindo o tempo até localizar a causa.

---

## Seção 6 — O Caso Prime Video

> _"O concorrente fez melhor com menos."_

### O gerente está certo?
**A comparação não se sustenta diretamente.** A redução de 90% foi em **um serviço específico**, por mecanismos de custo que a StreamFlow não tem.

### O que aconteceu no Prime Video (Kolny, 2023)
- **Escopo:** apenas a ferramenta de **monitoramento de qualidade de áudio/vídeo** (equipe de Video Quality Analysis), não a plataforma inteira. Segundo Adrian Cockcroft, foi a refatoração de **um** dos muitos microsserviços do Prime Video, não uma volta ao monólito.
- **Arquitetura original:** componentes serverless (AWS Step Functions orquestrando AWS Lambda), com os quadros de vídeo guardados no S3 entre as etapas.
- **Mecanismos de custo:**
  1. A orquestração cobrava **por transição de estado**, e havia várias transições **para cada segundo** de vídeo analisado.
  2. As etapas trocavam **quadros de vídeo** pelo S3, gerando um volume alto de leituras cobradas.
  3. A solução atingiu um limite de escala em cerca de **5% da carga esperada**.
- **Solução:** todos os componentes num **único processo** (ECS sobre EC2), com os dados trafegando **em memória**. O custo de infraestrutura caiu mais de 90%.

### Comparação com a StreamFlow

| Mecanismo | Prime Video (antes) | StreamFlow |
|---|---|---|
| Orquestração cobrada por etapa | Sim (Step Functions) | **Não** |
| Dados volumosos entre etapas | Sim (quadros de vídeo via S3) | **Não** (JSON pequeno via HTTP) |
| Execução | Serverless (cobrança por invocação) | Contêineres de longa duração |
| Processamento contínuo por segundo de vídeo | Sim | **Não** (a aplicação simula sessões, não transmite vídeo) |

**Conclusão:** os dois mecanismos que geraram a economia não existem na StreamFlow, então não há base para esperar 90% de redução. O que se transfere é a **lição**: avaliar caso a caso se a separação traz benefício real.

### O analytics-service [código + medido] (E04, A03)
- O pipeline do analytics (coleta → agregação → formatação) **já roda dentro de um único processo**, com os dados em memória. Ou seja, internamente ele já tem o formato que o Prime Video adotou.
- **Sequência de etapas não é antipadrão:** cada etapa depende da anterior, e rodar no mesmo processo é o correto.
- **O problema real é o banco compartilhado:** com o billing parado, o analytics continuou gerando o relatório financeiro completo (E04), porque lê `shared_billing.db` diretamente. Uma mudança no schema do billing quebraria o analytics sem aviso.
- **Necessidade de escala própria:** não há evidência. É um relatório de baixa frequência (~0,27 s, quase todo atraso artificial).

**Decisão [proposta]:** **consolidar o analytics no billing** como módulo de relatórios. Ele não tem dados próprios, não precisa escalar separadamente e já está acoplado ao billing pelo banco. A consolidação torna esse acoplamento explícito e remove uma unidade implantável.

### Outros candidatos à consolidação
- **notification:** sem banco, sem estado e dispensável para o Play. Candidato a virar consumidor de fila ou módulo (Seção 1).
- **Não candidatos:** auth (fronteira de segurança), catalog (dono das licenças), streaming (núcleo do produto) e recommendation (domínio próprio).

- **Vídeo, 11:12–14:48 (caso Prime Video):** o vídeo apresenta o caso como a consolidação de um pipeline "de volta para um monólito" e o usa como prova de que nem todo fluxo se beneficia da distribuição. **Confronto com a fonte primária:** a segunda parte é correta, mas a palavra "monólito" exagera. O relato de Kolny trata de **um** serviço de monitoramento, e Adrian Cockcroft observa que o resultado foi a refatoração de um microsserviço, não o abandono da arquitetura. Além disso, o vídeo destaca o número (90%) mais do que os mecanismos que o produziram, e são justamente os mecanismos que decidem se a lição se transfere para a StreamFlow.

---

## Tabela de decisões priorizadas

| Prioridade | Ação | Evidência | Benefício esperado | Custo / risco | Como validar |
|---|---|---|---|---|---|
| **P1** | Corrigir o health check (status, `response.ok`, consulta em paralelo, motivo da falha) | E01, E06 | Alarme volta a ter significado; health rápido durante incidentes | Baixo: poucas linhas no gateway | Repetir E01 (esperar 200) e E06 (esperar 503 só com serviço parado, em < 3 s) |
| **P1** | Tirar notificação e registro de visualização do caminho síncrono do Play | A01, E03, E05 | Play estável mesmo com dependências lentas | Médio: fila nova para operar; exige consumidor idempotente | Repetir E05: Play na faixa de ~0,24 s |
| **P1** | Propagar ID de rastreamento (`traceparent`) e incluí-lo nos logs | A03, E03 | Localizar a origem de uma falha | Médio: instrumentação OpenTelemetry + servidor de tracing | Um Play gera um trace com 5 trechos ligados |
| **P2** | Timeout curto e circuit breaker na verificação de licença | E05 | Falha rápida em vez de espera de 5 s | Baixo; risco de bloquear Plays se o limite for mal calibrado | Congelar o catalog e medir o tempo de resposta |
| **P2** | Métricas por rota e alerta de latência p95 do Play | E03, E05 | Responder "durou TANTO, afetou TANTOS" | Médio: Prometheus + Grafana | Repetir E05 e verificar se o alerta dispara |
| **P2** | Consolidar o analytics no billing | E04, A02 | Fim do acoplamento escondido; −1 unidade implantável | Médio: migração de código; relatórios passam a depender do billing | Alterar o schema de `invoices` e confirmar que os relatórios acompanham |
| **P3** | Evento de remoção de título + reconciliação periódica | A03 | Recomendações convergem para o catálogo | Médio: fila, outbox e job | Remover um título e medir o tempo até sumir das recomendações |
| **P3** | Chave JWT nos 5 serviços: validar o token neles ou remover a configuração sem uso | A02, A03 | Fim da falsa sensação de proteção; ou defesa em profundidade, ou menos configuração | Baixo | Chamar um serviço interno sem token pela rede do Compose: deve ser recusado (se validar) |
| **P3** | Reorganizar times por domínio, com rodízio de plantão e runbooks | Seção 5 | Conhecimento distribuído | Organizacional; período de adaptação | Simulação de incidente atendida fora da plataforma |

---

## Bônus — Moleculer.js

**Versão consultada:** documentação 0.15 (https://moleculer.services/docs/0.15/). Página de tolerância a falhas atualizada em 06/09/2026; versão mais recente no changelog: 0.15.2 (29/08/2026).
**Data de acesso:** 03/11/2026.
**Compatibilidade:** a 0.15 exige Node ≥ 22; os Dockerfiles do StreamFlow usam `node:22-bookworm-slim`, portanto compatíveis.

### 1. O que o Moleculer resolve nativamente

| Recurso | Disponível | Ligado por padrão? | Problema do StreamFlow que atacaria | Ressalva |
|---|---|---|---|---|
| Service discovery / registry | Sim (dinâmico, com heartbeat) | Sim, quando há transporter | Endereços fixos no Compose; falha de DNS ao parar um serviço (E03) | Um nó caído só sai do registro depois do timeout de heartbeat |
| Load balancing | Sim (RoundRobin, Random, CPU, Latency, Shard) | Sim (RoundRobin) | — | Só faz diferença com várias instâncias; hoje há 1 de cada |
| Timeout | Sim, inclusive **distribuído** (o prazo diminui a cada chamada aninhada) | **Não** (`requestTimeout: 0`) | Um timeout de 5 s medido no Play (E05); a soma dos três é projeção | Precisa ser configurado |
| Retry | Sim (backoff exponencial) | **Não** | — | Ver idempotência abaixo |
| Circuit breaker | Sim (por endpoint, janela de tempo) | **Não** | Espera de 5 s por um notification travado (E05) | Só abre após 20 requisições na janela (padrão) |
| Bulkhead | Sim (limita concorrência por ação) | **Não** | Um serviço lento prendendo recursos dos outros | Precisa ser dimensionado |
| Fallback | Sim (resposta alternativa na chamada) | Definido por chamada | Notificação e recomendação são dispensáveis no Play | Exige decidir o que devolver |
| Tracing | Sim (Console, Jaeger, Zipkin, Datadog, NewRelic, Event) | **Não** | Nenhum ID de rastreamento entre serviços (A03) | Exige operar um servidor de tracing |
| Métricas | Sim (Prometheus, Datadog, StatsD, CSV, Console) | **Não** | Sem métricas para "durou TANTO, afetou TANTOS" | Exige operar Prometheus/Grafana |
| Eventos | Sim, mas *fire-and-forget* | Sim | Notificação síncrona no Play (A01, E03) | Entrega garantida exige `@moleculer/channels` com fila externa |

**Idempotência ao usar retry.** O Play **cria** uma sessão (HTTP 201, `INSERT` em `streaming/index.js:108`) e grava a visualização (`INSERT` em `recommendation/index.js:76`). Com retry ligado, um timeout seguido de nova tentativa pode duplicar sessões e visualizações, porque a primeira tentativa pode ter sido gravada mesmo sem resposta. Retry só é seguro na consulta de licença (leitura); nas gravações, exige uma chave de idempotência ou deve ficar desligado. Além disso, como o timeout é tratado como erro repetível, o pior caso de uma chamada fica perto de (tentativas + 1) × timeout.

**Garantia de entrega ao usar eventos.** Os eventos nativos não são persistidos: se o notification estiver fora do ar no momento do evento, a mensagem se perde. Para entrega garantida, é preciso o `@moleculer/channels` com fila externa (Redis Streams, RabbitMQ, NATS JetStream ou Kafka), que oferece confirmação, nova tentativa e fila de mensagens mortas. Essa entrega é "pelo menos uma vez", então o consumidor também precisa ser idempotente.

### 2. O que o Moleculer esconde e por que isso é perigoso

- **Chamada de rede com cara de chamada local.** `broker.call("catalog.license")` tem a mesma sintaxe, esteja o serviço no mesmo processo ou em outra máquina. Latência, falha parcial e timeout ficam invisíveis no código: exatamente os comportamentos medidos no E02, E03 e E05. Isso se liga ao argumento central do vídeo (1:30–5:30 e conclusão, 14:48–16:09): a complexidade de um sistema distribuído não desaparece quando é escondida, só fica menos visível para quem precisa operá-lo.
- **Uma peça nova e crítica de infraestrutura.** Toda a comunicação passa pelo transporter (NATS, Redis, Kafka etc.). Se ele cair, todos os serviços ficam incomunicáveis. Ele também precisa de correções de segurança: a 0.15.2 corrigiu uma negação de serviço no transporter TCP.
- **Conhecimento concentrado de novo.** Um framework com configurações próprias de discovery, retry e circuit breaker cria outro conhecimento especializado que pode ficar com poucas pessoas (Seção 5).
- **Configuração que parece proteção.** Como os recursos vêm desligados, é possível "migrar para o Moleculer" e continuar sem proteção nenhuma. E a combinação errada (retry + timeout) pode **multiplicar** a espera do usuário.
- **Dependência do framework.** Antes da 1.0, mudanças incompatíveis saem a cada versão secundária (a 0.15 trocou o protocolo de comunicação e removeu serializadores).
- **O que ele não resolve.** O banco compartilhado (E04), a falta de reconciliação de dados (A03) e a decisão de chamar a notificação de forma síncrona (A01) são problemas de **desenho**.

### 3. Recomendação: não migrar agora

**Decisão:** não migrar integralmente nem parcialmente neste momento; reavaliar depois de corrigir os problemas de desenho.

1. Os problemas mais graves são de desenho e se resolvem com mudanças pequenas e locais (tabela de decisões).
2. A migração adicionaria um transporter para operar e um framework para aprender, justamente quando a equipe perdeu conhecimento de plataforma e os custos estão em alta [premissa].
3. Os mesmos padrões podem ser aplicados sem trocar o framework: OpenTelemetry para tracing, uma biblioteca de circuit breaker nas chamadas síncronas e uma fila para a notificação.

**Quando reconsiderar:** se a StreamFlow precisar de várias instâncias por serviço (onde discovery e balanceamento passam a valer a pena) **e** tiver uma equipe de plataforma estável para operar o transporter.

**Como validar a alternativa:** repetir o E05 depois de tornar a notificação assíncrona. O resultado esperado é que o Play volte à faixa do E02 (~0,24 s) mesmo com o notification congelado.

**Argumento contrário mais forte:** o Moleculer traz tracing, métricas e circuit breaker integrados, o que poupa a equipe de montar essas peças uma a uma. A resposta é que, além de configurar cada recurso, a equipe teria de operar o transporter, e esse custo pesa mais no momento atual.

---

## Referências

**Material da disciplina**
- Bernardi, É. F. F. *Atividade: Análise Crítica de Arquitetura de Microsserviços — Cenário StreamFlow*. IFSul, rev. 20/10/2026.
- *Guia Completo de Arquitetura de Microsserviços e Containerização com Docker* (material da disciplina), `docs/guia_completo_microsservicos.md`. Seções citadas: 2.5, 3.2, 3.3, 3.5, 3.6, 4.1, 4.3, 5.1–5.4, 7.1, 7.2, 10.4, 13.1–13.3, 14.1, 14.3, 15.
- CONHEÇA A TECNOLOGIA. *Microservices at Scale: Engineering Debt and System Complexity*. YouTube, vídeo (~16 min). https://www.youtube.com/watch?v=ILXcsNxbas4. Trechos usados: 0:04–1:30, 1:30–5:30, 5:30–8:50, 8:50–11:12, 11:12–14:48, 14:48–16:09.
- Repositório StreamFlow (fork), commit `ea0c63bc097304d87e7fcb8fa690d7d13d6c8ffa`: `README.md`, `docs/SETUP.md`, `docs/DEBUG.md`, `arquitetura_streamflow_reference.md`, `docker-compose.yml` e código em `gateway/` e `services/`.

**Caso Prime Video**
- Kolny, M. *Scaling up the Prime Video audio/video monitoring service and reducing costs by 90%*. Prime Video Tech, 22/03/2023. https://www.primevideotech.com/video-streaming/scaling-up-the-prime-video-audio-video-monitoring-service-and-reducing-costs-by-90 (cópia em PDF indicada no README da atividade).
- Cockcroft, A., comentário sobre o caso (maio/2023), citado em: https://mjtsai.com/blog/?p=39291

**Arquitetura e organização**
- Conway, M. E. *How Do Committees Invent?* Datamation, 1968.
- Skelton, M.; Pais, M. *Team Topologies*. IT Revolution, 2019.
- Fowler, M. *CircuitBreaker*. https://martinfowler.com/bliki/CircuitBreaker.html
- W3C. *Trace Context* (cabeçalho `traceparent`). https://www.w3.org/TR/trace-context/
- OpenTelemetry. Documentação. https://opentelemetry.io/docs/

**Moleculer (bônus)**: todos acessados em 03/11/2026
- https://moleculer.services/docs/0.15/
- https://moleculer.services/docs/0.15/fault-tolerance
- https://moleculer.services/docs/0.15/registry
- https://moleculer.services/docs/0.15/balancing
- https://moleculer.services/docs/0.15/tracing
- https://moleculer.services/docs/0.15/metrics
- Changelog: https://github.com/moleculerjs/moleculer/blob/master/CHANGELOG.md
- `@moleculer/channels`: https://www.npmjs.com/package/@moleculer/channels
