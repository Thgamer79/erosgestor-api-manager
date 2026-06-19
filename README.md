# ErosGestor API Manager — Backend (Fase 1)

Plataforma SaaS de gerenciamento de instâncias WhatsApp (Baileys) e bots Telegram,
com API REST exclusiva por instância para consumo externo (bots, CRMs, automações).

> **Status:** Fase 1 concluída — Backend completo (auth, instâncias, mensagens, webhooks,
> Telegram, admin, Socket.IO, Swagger, Docker). Frontend React, Mercado Pago/PIX e
> Evolution API como provider alternativo entram na Fase 2.

## Estrutura

```
erosgestor-api-manager/
├── backend/
│   ├── src/
│   │   ├── config/        # env, database (Sequelize), swagger
│   │   ├── models/        # User, Plan, Instance, ApiKey, Webhook, TelegramBot, Log, Subscription
│   │   ├── controllers/   # auth, instance, message, webhook, telegram, admin
│   │   ├── routes/        # mapeamento HTTP -> controllers
│   │   ├── middlewares/   # JWT auth, API Key auth, rate limit, admin guard, error handler
│   │   ├── services/
│   │   │   ├── whatsapp/baileysService.js   # núcleo Baileys (multi-instância)
│   │   │   ├── telegram/telegramService.js  # núcleo Telegram (webhook nativo)
│   │   │   └── webhookService.js            # dispara eventos para apps externos
│   │   ├── sockets/        # Socket.IO (QR Code, status, mensagens em tempo real)
│   │   ├── utils/          # jwt, crypto (AES-256-GCM), apiKeyGenerator, logger, seed, migrate
│   │   ├── app.js
│   │   └── server.js
│   ├── storage/sessions/   # sessões do Baileys (1 pasta por instância)
│   ├── .env.example
│   └── Dockerfile
├── database/schema.sql
└── docker-compose.yml
```

## 1. Instalação local (sem Docker)

### Pré-requisitos
- Node.js 20+
- MySQL 8+

### Passos

```bash
cd backend
cp .env.example .env
# Edite o .env: DB_*, JWT_SECRET, JWT_REFRESH_SECRET, ENCRYPTION_KEY

npm install
```

Gere os segredos antes de editar o `.env`:

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"   # JWT_SECRET
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"   # JWT_REFRESH_SECRET
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"   # ENCRYPTION_KEY (32 bytes exatos)
```

Crie o banco e as tabelas (duas opções):

```bash
# Opção A: importar o schema.sql diretamente
mysql -u root -p < ../database/schema.sql

# Opção B: deixar o Sequelize criar/sincronizar (modo dev)
npm run migrate
```

Crie o usuário admin inicial e garanta os planos padrão:

```bash
SEED_ADMIN_EMAIL=admin@seudominio.com SEED_ADMIN_PASSWORD='SenhaForte123!' npm run seed
```

Inicie o servidor:

```bash
npm run dev     # desenvolvimento (nodemon)
npm start       # produção
```

A API estará em `http://localhost:3000`, e a documentação Swagger em `http://localhost:3000/docs`.

## 2. Instalação com Docker

```bash
cd backend
cp .env.example .env
# Edite o .env (DB_HOST deve continuar "mysql" — é o nome do serviço no compose)

cd ..
docker compose up -d --build
docker compose exec backend npm run seed
docker compose logs -f backend
```

## 3. Testando a criação de instância + API Key exclusiva

```bash
# 1. Login
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@seudominio.com","password":"SenhaForte123!"}'
# -> copie o accessToken retornado

# 2. Criar instância (gera QR Code + API Key exclusiva automaticamente)
curl -X POST http://localhost:3000/api/instances \
  -H "Authorization: Bearer SEU_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Atendimento Comercial","type":"whatsapp","provider":"baileys"}'
# -> resposta contém "apiKey": "eros_xxxxxxxx..." — GUARDE, só aparece uma vez

# 3. Buscar QR Code para escanear no WhatsApp
curl http://localhost:3000/api/instances/1/qrcode \
  -H "Authorization: Bearer SEU_ACCESS_TOKEN"

# 4. Após conectar, enviar mensagem usando a API Key da instância (consumo externo)
curl -X POST http://localhost:3000/api/sendText \
  -H "Authorization: Bearer eros_xxxxxxxx..." \
  -H "Content-Type: application/json" \
  -d '{"to":"5511999999999","text":"Olá, vim da API exclusiva da minha instância!"}'
```

## 4. Deploy em produção (resumo)

1. **Servidor**: VPS com Docker + Docker Compose (Ubuntu 22.04+ recomendado).
2. **Domínio + HTTPS**: aponte um subdomínio (ex: `api.seudominio.com`) para o servidor
   e use Nginx ou Caddy como reverse proxy com certificado Let's Encrypt na frente do
   container `backend` (porta 3000).
3. **Variáveis de ambiente**: nunca reaproveite os segredos do `.env.example` —
   gere novos `JWT_SECRET`, `JWT_REFRESH_SECRET` e `ENCRYPTION_KEY` para produção.
4. **Backups**: agende dump diário do volume `mysql_data` (`mysqldump`) e do volume
   `backend_sessions` (sessões do Baileys — perdê-las força reconexão via QR Code).
5. **Processo resiliente**: o `docker-compose.yml` já usa `restart: unless-stopped`;
   para múltiplas réplicas do backend, é necessário externalizar o registro de sockets
   Baileys (hoje em memória) para Redis ou afinidade de instância por worker.
6. **Logs**: os logs do `pino` vão para stdout (capturados pelo Docker); os logs de
   negócio (login, criação de instância, etc.) ficam na tabela `logs` e aparecem no
   painel admin (`GET /api/admin/logs`).

## 5. Próximas fases

- **Fase 2**: Frontend React (Vite + Tailwind), tema escuro estilo Evolution API.
- **Fase 3**: Integração com Evolution API como provider alternativo ao Baileys.
- **Fase 4**: Cobrança via Mercado Pago (assinaturas + PIX) e enforcement automático
  de limites de plano/cobrança recorrente.
- **Fase 5**: Documentação Swagger expandida (JSDoc por rota) + testes automatizados.
