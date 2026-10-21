# StreamFlow — Guia de Instalação e Execução

## Pré-requisitos

- **Docker** ≥ 24.0 e **Docker Compose** ≥ 2.20
- **Node.js** 22 (compatível com os Dockerfiles atuais; use uma linha ainda suportada) (apenas para desenvolvimento local sem Docker)
- **Git** (para clonar o repositório)
- **curl** e **jq** para os comandos deste roteiro; Bash em Linux, macOS ou WSL.
- Acesso ao daemon Docker. Node.js no host não é necessário para executar via Compose.

Verifique sua instalação:
```bash
docker --version
docker compose version
node --version
```

---

## Executar com Docker Compose (recomendado)

### 1. Clonar o repositório
```bash
git clone https://github.com/elderbernardiifsul/StreamFlow-Microsservices-madness.git
cd StreamFlow-Microsservices-madness
```

Se estiver usando a cópia da disciplina, entre em `sw/atividades/analise_critica_microsservicos_netflix/projeto`. Execute os comandos na pasta que contém `docker-compose.yml`. Para entregar, use seu fork.

### 2. Verificar as chaves RSA
O projeto já inclui um par público de chaves RSA exclusivamente para o laboratório. Não reutilize essas chaves ou credenciais em aplicações reais. Se quiser gerar novas:
```bash
mkdir -p keys
openssl genrsa -out keys/private.pem 2048
openssl rsa -in keys/private.pem -pubout -out keys/public.pem
```

### 3. Subir todos os serviços
```bash
docker compose up --build
```

Você verá logs de todos os 8 serviços (gateway + 7 microsserviços).
O gateway estará acessível em `http://localhost:8080`.

Aguarde a inicialização e confira `docker compose ps` e `/health` antes de medir. A ordem de inicialização não garante que os serviços já estejam prontos.

### 4. Testar o sistema

**Login (obter token):**
```bash
curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@streamflow.com","password":"admin123"}' | jq
```

**Salvar o token numa variável (Linux/Mac):**
```bash
TOKEN=$(curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@streamflow.com","password":"admin123"}' | jq -r '.token')
```

**Listar catálogo:**
```bash
curl -s http://localhost:8080/api/catalog \
  -H "Authorization: Bearer $TOKEN" | jq
```

**Iniciar reprodução (fluxo com cadeia síncrona):**
```bash
curl -s -X POST http://localhost:8080/api/streaming/play \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"movieId": 1}' | jq
```

**Ver recomendações:**
```bash
curl -s http://localhost:8080/api/recommendations/user_1 \
  -H "Authorization: Bearer $TOKEN" | jq
```

**Ver faturamento:**
```bash
curl -s http://localhost:8080/api/billing/user_1 \
  -H "Authorization: Bearer $TOKEN" | jq
```

**Gerar relatório de analytics (pipeline sequencial):**
```bash
curl -s http://localhost:8080/api/analytics/report \
  -H "Authorization: Bearer $TOKEN" | jq
```

**Health check geral:**
```bash
curl -s http://localhost:8080/health | jq
```

### 5. Parar os serviços
```bash
docker compose down

# Opcional e destrutivo: apaga todos os dados do laboratório; salve as evidências antes:
docker compose down -v
```

---

## Executar sem Docker (desenvolvimento local)

Cada serviço pode ser executado individualmente:

```bash
# Instalar dependências de um serviço
cd services/auth
npm install

# Executar com hot-reload
npm run dev
```

Crie o diretório de dados antes de iniciar cada serviço (`mkdir -p data`). Exporte explicitamente as variáveis de ambiente: o código não carrega `.env` automaticamente. Configure também os caminhos das chaves RSA e o mesmo `DB_PATH` absoluto para billing e analytics. Inicialize billing antes de consultar analytics.
Para o gateway, ajuste as URLs dos serviços para `http://localhost:<porta>`.

---

## Estrutura de portas

| Serviço | Porta | Acessível externamente? |
|---|---|---|
| **Gateway** | 8080 | Sim — única porta pública |
| auth-service | 3001 | Não (apenas via gateway) |
| catalog-service | 3002 | Não |
| streaming-service | 3003 | Não |
| recommendation-service | 3004 | Não |
| notification-service | 3005 | Não |
| billing-service | 3006 | Não |
| analytics-service | 3007 | Não |

---

## Usuários disponíveis para testes

| Email | Senha | Role |
|---|---|---|
| admin@streamflow.com | admin123 | admin |
| joao@streamflow.com | user123 | user |
| maria@streamflow.com | user123 | user |
