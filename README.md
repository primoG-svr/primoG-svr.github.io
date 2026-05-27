# 🚀 NexHub Backend — Node.js + Mercado Pago

Backend completo para a plataforma NexHub com sistema de Escrow e integração total com o Mercado Pago.

---

## 📁 Estrutura

```
nexhub-backend/
├── src/
│   ├── server.js                 ← Entrada principal
│   ├── config/
│   │   ├── mercadopago.js        ← Configuração do SDK MP
│   │   └── database.js           ← Banco de dados (memória / troque por Postgres)
│   └── routes/
│       ├── payments.js           ← PIX, Cartão, Checkout Pro
│       ├── escrow.js             ← Sistema de Escrow completo
│       ├── webhooks.js           ← Notificações em tempo real do MP
│       ├── products.js           ← CRUD de produtos
│       └── users.js              ← Usuários e carteiras
├── public/
│   ├── nexhub-api.js             ← Script de integração para o frontend HTML
│   └── [coloque o comunidade.html aqui]
├── .env.example                  ← Template de variáveis de ambiente
├── package.json
└── README.md
```

---

## ⚡ Instalação

```bash
# 1. Entre na pasta
cd nexhub-backend

# 2. Instale as dependências
npm install

# 3. Configure o .env
cp .env.example .env
# Edite o .env com seu Access Token do Mercado Pago

# 4. Inicie o servidor
npm run dev       # desenvolvimento (hot reload)
npm start         # produção
```

---

## 🔑 Configuração do Mercado Pago

### 1. Obter credenciais

1. Acesse: https://www.mercadopago.com.br/developers/panel
2. Crie uma aplicação ou use uma existente
3. Em **Credenciais**, copie:
   - `Access Token` → `MP_ACCESS_TOKEN` no .env
   - `Public Key` → `MP_PUBLIC_KEY` no .env

> Use as credenciais de **Sandbox** para testes e **Produção** para go-live.

### 2. Configurar Webhook

O Mercado Pago precisa de uma URL pública para notificar pagamentos.

**Em desenvolvimento (use ngrok):**
```bash
# Instale o ngrok: https://ngrok.com
ngrok http 3000

# Copie a URL gerada, ex: https://abcd1234.ngrok.io
# Configure no .env:
MP_WEBHOOK_URL=https://abcd1234.ngrok.io/webhooks/mercadopago
```

**No painel do MP:**
1. Vá em: Aplicações → Sua App → Webhooks
2. Adicione a URL: `https://seudominio.com/webhooks/mercadopago`
3. Selecione eventos: `payment`, `merchant_order`
4. Copie o Secret e coloque em `MP_WEBHOOK_SECRET`

---

## 📡 Endpoints da API

### Pagamentos

| Método | Rota | Descrição |
|--------|------|-----------|
| `POST` | `/api/payments/pix` | Gerar QR Code + Copia e Cola PIX |
| `POST` | `/api/payments/card` | Pagar com cartão (token do Brick) |
| `POST` | `/api/payments/checkout` | Checkout Pro (redirect MP) |
| `GET`  | `/api/payments/:id` | Consultar status do pagamento |
| `GET`  | `/api/payments/` | Listar transações |

### Escrow

| Método | Rota | Descrição |
|--------|------|-----------|
| `POST` | `/api/escrow/create` | Criar escrow + gerar link de pagamento |
| `GET`  | `/api/escrow/:id` | Consultar escrow |
| `GET`  | `/api/escrow/` | Listar escrows (filtrar por buyerId/sellerId) |
| `POST` | `/api/escrow/:id/deliver` | Vendedor marca como entregue |
| `POST` | `/api/escrow/:id/release` | Comprador confirma e libera pagamento |
| `POST` | `/api/escrow/:id/dispute` | Abrir disputa |
| `POST` | `/api/escrow/:id/resolve` | Admin resolve disputa |
| `POST` | `/api/escrow/:id/cancel` | Cancelar (antes do pagamento) |

### Outros

| Método | Rota | Descrição |
|--------|------|-----------|
| `GET`  | `/api/products` | Listar produtos (filtros: category, minPrice, maxPrice) |
| `POST` | `/api/products` | Criar produto |
| `GET`  | `/api/users/:id/wallet` | Carteira + extrato do usuário |
| `POST` | `/webhooks/mercadopago` | Webhook do MP (interno) |
| `GET`  | `/health` | Health check |

---

## 🧪 Testando

### Teste PIX (sandbox)
```bash
curl -X POST http://localhost:3000/api/payments/pix \
  -H "Content-Type: application/json" \
  -d '{
    "productId": "p1",
    "buyerEmail": "comprador@teste.com",
    "buyerName": "João Teste",
    "buyerCpf": "12345678900"
  }'
```

### Criar Escrow
```bash
curl -X POST http://localhost:3000/api/escrow/create \
  -H "Content-Type: application/json" \
  -d '{
    "productId": "p1",
    "buyerId": "u3",
    "sellerId": "u1",
    "buyerEmail": "joao@nexhub.com",
    "buyerName": "João Vitor",
    "amount": 197.00,
    "description": "UI/UX Masterclass 2025"
  }'
```

### Simular pagamento aprovado (dev)
```bash
# Após criar o escrow, use o ID retornado:
curl http://localhost:3000/webhooks/test-approve/{escrowId}
```

### Liberar pagamento
```bash
curl -X POST http://localhost:3000/api/escrow/{escrowId}/release \
  -H "Content-Type: application/json" \
  -d '{ "buyerId": "u3", "rating": 5, "reviewText": "Excelente produto!" }'
```

---

## 🔗 Integração com o Frontend HTML

1. Copie `comunidade.html` para a pasta `public/`
2. Adicione antes do `</body>` do HTML:
```html
<script src="/nexhub-api.js"></script>
```
3. Substitua a função `confirmPayment()` no HTML:
```javascript
// Troque a chamada no botão "Pagar com Segurança" por:
onclick="confirmPaymentWithAPI(currentProductId, currentPrice, 'email@user.com', 'Nome User', '00000000000')"
```

---

## 🏗️ Banco de Dados (produção)

Para produção, substitua o `src/config/database.js` por PostgreSQL:

```bash
npm install pg
```

```javascript
// Exemplo com pg:
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function getEscrow(id) {
  const { rows } = await pool.query('SELECT * FROM escrows WHERE id=$1', [id]);
  return rows[0];
}
```

---

## 🔒 Fluxo de Escrow

```
Comprador paga → MP aprova → Webhook recebido
      ↓
  Dinheiro RETIDO (held) no sistema
      ↓
  Vendedor entrega produto/serviço
      ↓
  Comprador confirma recebimento
      ↓
  Dinheiro LIBERADO ao vendedor (líquido - taxa)
      ↓
  Plataforma retém taxa (3% padrão)
```

Em caso de problema → **Disputa** → Admin decide reembolso ou liberação.

---

## 🚢 Deploy (produção)

```bash
# Render / Railway / Heroku
# Adicione as variáveis de ambiente no painel
# Defina NODE_ENV=production
# Start command: npm start
```

---

## 📞 Suporte Mercado Pago

- Documentação: https://www.mercadopago.com.br/developers/pt/docs
- Sandbox: https://www.mercadopago.com.br/developers/panel/test-credentials
- Status: https://status.mercadopago.com
