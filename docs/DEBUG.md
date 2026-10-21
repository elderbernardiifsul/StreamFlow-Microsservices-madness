# StreamFlow — Guia de Debugging

Este guia ajuda a diagnosticar problemas e entender o comportamento do sistema.

---

Antes dos cenários, execute o login de `SETUP.md` e defina `TOKEN` na mesma sessão de shell. Confirme o estado saudável inicial e restaure os serviços depois de cada experimento. A consulta ao analytics pressupõe que billing já inicializou seu banco.

## Logs

### Ver logs de todos os serviços
```bash
docker compose logs -f
```

### Ver logs de um serviço específico
```bash
docker compose logs -f streaming-service
docker compose logs -f gateway
```

### Filtrar logs por nível
Os serviços usam logs JSON do Fastify/Pino; o gateway usa pino-pretty e não segue esse formato na saída. Para filtrar erros dos serviços:
```bash
docker compose logs -f | grep '"level":50'   # erros
docker compose logs -f | grep '"level":40'   # warnings
```

---

## Inspecionar o banco de dados

Para acessar o SQLite dentro de um container:
```bash
# Entrar no container
docker compose exec catalog-service sh

# Dentro do container, usar sqlite3 (instalar se necessário)
apt-get update && apt-get install -y sqlite3
sqlite3 /app/data/catalog.db

# Listar tabelas
.tables

# Ver dados
SELECT * FROM movies;

# Sair
.quit
exit
```

---

## Cenários de teste para entender os antipatterns

### Cenário 1: Cadeia síncrona — impacto de latência
Observe o tempo de resposta do endpoint de play:
```bash
time curl -s -X POST http://localhost:8080/api/streaming/play \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"movieId": 2}'
```
Compare com o tempo de uma chamada direta ao catálogo (rota simples):
```bash
time curl -s http://localhost:8080/api/catalog \
  -H "Authorization: Bearer $TOKEN"
```
A diferença inclui chamadas, processamento, acesso a banco e atrasos artificiais. Ela não isola latência de rede. Faça aquecimento e pelo menos cinco medições de cada rota; registre status HTTP junto ao tempo com `curl -w "\nhttp=%{http_code} tempo=%{time_total}s\n"`.

### Cenário 2: Notification como ponto de falha
Pare o notification-service e observe o que acontece com o play:
```bash
# Parar apenas o notification-service
docker compose stop notification-service

# Tentar reproduzir
curl -s -X POST http://localhost:8080/api/streaming/play \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"movieId": 3}' | jq
```
O play ainda funciona (fail silencioso), mas observe nos logs do streaming-service
a mensagem de warning. Parar um contêiner pode provocar recusa imediata de conexão; isso não demonstra espera até o timeout. O código define até 5 segundos para cada chamada interna. Para demonstrar timeout seria necessário um experimento controlado adicional com uma dependência que aceite a conexão e não responda no prazo.

```bash
# Reiniciar o serviço
docker compose start notification-service
```

### Cenário 3: Banco compartilhado — acoplamento visível
Pare o billing-service e tente gerar um relatório de analytics:
```bash
docker compose stop billing-service

# Analytics ainda funciona! Porque lê o banco diretamente.
curl -s http://localhost:8080/api/analytics/report \
  -H "Authorization: Bearer $TOKEN" | jq
```
Isso demonstra o acoplamento: o analytics não depende da **API** do billing,
mas depende do **banco** do billing. Se o billing alterar o schema da tabela,
o analytics quebra sem aviso.

```bash
docker compose start billing-service
```

### Cenário 4: Health check — visão agregada
```bash
curl -s http://localhost:8080/health | jq
```
Experimente parar um serviço e observar o status mudar para "degraded".

---

## Problemas comuns

| Problema | Causa provável | Solução |
|---|---|---|
| `ECONNREFUSED` nos logs do gateway | Serviço ainda não iniciou | Aguarde ou reinicie: `docker compose restart gateway` |
| `Token inválido` | Chaves RSA não montadas corretamente | Verifique se `keys/` contém `private.pem` e `public.pem` |
| Banco de dados vazio | Volume ou inicialização incorretos | Inspecione `docker compose logs` e os volumes antes de alterar dados; recriar volumes apaga o estado e não é o primeiro passo |
| Módulo nativo falha no build | Binário pré-compilado indisponível ou ferramentas de compilação ausentes | Registre o erro de instalação; os Dockerfiles usam `node:22-bookworm-slim` (Debian), não Alpine |
