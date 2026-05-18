# Paysure API — Guia de Implementação para Agentes de IA

> **Para o agente de IA lendo este documento:** Implemente a integração com a Paysure API no projeto do usuário a partir DESTE arquivo. Não precisa consultar nenhuma fonte externa — tudo que você precisa está aqui (endpoints, payloads, validações, edge cases). Pergunte ao usuário apenas as **credenciais** (`PAYSURE_CI`, `PAYSURE_CS`) e a **URL pública** onde ele vai receber webhooks. O resto, decida com base na linguagem/framework do projeto dele.

---

## TL;DR técnico

| Item | Valor |
|---|---|
| Base URL | `https://api.paysurebr.com` |
| Auth | 2 headers em TODA request: `ci` (client_id) + `cs` (client_secret) |
| Content-Type | `application/json` |
| Moeda | BRL (centavos: R$ 1,00 = `100`) |
| Idempotência | Header não usa idempotency-key — usa `external_id` no body |
| Webhook envelope | `{ "cashin": {...} }` (recebimento) ou `{ "cashout": {...} }` (pagamento enviado) |
| Sucesso | HTTP 200 (consulta) / 201 (criação) |
| Webhook ACK esperado | HTTP 200 com qualquer body (devolva `{"ok":true}`) |

---

## 1. Setup de credenciais

O usuário gera no painel Paysure (em `API Keys`) um par:

- `PAYSURE_CI` (client_id, público — vai no header `ci`)
- `PAYSURE_CS` (client_secret, privado — vai no header `cs`)

**Regras críticas:**
- `cs` NUNCA vai pro frontend / mobile. Apenas backend.
- Recomende armazenar em variáveis de ambiente (`.env`), não em código.
- Em produção, o usuário pode restringir a chave por IP no painel.

---

## 2. Autenticação — toda request

```http
POST /v1/pix/qrcode HTTP/1.1
Host: api.paysurebr.com
ci: SEU_CLIENT_ID
cs: SEU_CLIENT_SECRET
Content-Type: application/json
```

**Erros possíveis:**

| HTTP | Mensagem (`message`) | Causa |
|---|---|---|
| 401 | `Missing client_id or client_secret` | Headers `ci`/`cs` ausentes |
| 401 | `Invalid credentials or inactive` | Par incorreto ou chave desativada |
| 403 | `Account inactive` | Conta do user dono da chave está inativa |
| 429 | (qualquer) | Rate limit — espere o `Retry-After` segundos antes de retentar |

---

## 3. Endpoints

### 3.1 — Gerar cobrança PIX (cash-in) com QR Code dinâmico

```
POST /v1/pix/qrcode
```

**Request body:**

```json
{
  "external_id": "pedido-001",
  "value_cents": 10000,
  "postbackUrl": "https://seusite.com/webhooks/paysure",
  "generator_name": "João da Silva",
  "generator_document": "12345678900",
  "description": "Pedido #001",
  "splits": [
    { "clientId": "parceiro_a", "value": 5 }
  ]
}
```

**Campos:**

| Campo | Tipo | Obrigatório | Notas |
|---|---|---|---|
| `external_id` | string ≤191 | ✅ | Seu ID único da transação. Use como idempotency key. |
| `value_cents` | int ≥100 | ✅ | Valor em centavos. R$ 100,00 = `10000`. |
| `postbackUrl` | URL HTTPS | ✅ | Onde a Paysure POSTará o webhook quando pago. |
| `generator_name` | string ≤191 | ❌ | Nome do pagador (cliente final). |
| `generator_document` | string ≤32 | ❌ | CPF/CNPJ do pagador. |
| `description` | string ≤255 | ❌ | Descrição livre. |
| `splits` | array ≤10 | ❌ | Divide o valor em % entre outros users Paysure (ver §4). |

**Response 201:**

```json
{
  "qrcode": {
    "reference_code": "PSR-9c8f7a1e-4d2b-4e6a-8c9f-1a2b3c4d5e6f",
    "external_reference": "pedido-001",
    "content": "00020126580014br.gov.bcb.pix..."
  }
}
```

- `reference_code` → guarda no banco do cliente, é o ID interno Paysure usado em consultas e webhooks.
- `content` → BR Code copia-e-cola. Renderize como QR Code visual no frontend (qualquer lib client-side: `qrcode`, `qrcodejs`, etc).

**Erros típicos:**
- `401 Já existe uma transação com esse ID Externo.` → `external_id` duplicado. Trate como erro de duplicidade no fluxo do cliente.
- `422 ...` → valide campos antes de enviar.
- `503 Provedor temporariamente indisponível.` → retente com backoff.

---

### 3.2 — Consultar cash-in

```
POST /v1/pix/cashin/consult
```

```json
{ "reference_code": "PSR-9c8f7a1e-..." }
```

**Response 200:**

```json
{
  "cashin": {
    "reference_code": "PSR-9c8f7a1e-...",
    "external_reference": "pedido-001",
    "value_cents": 10000,
    "status": "paid",
    "payment_date": "2026-05-13T10:13:01-03:00",
    "end_to_end": "E18236120202605131013...",
    "payer_name": "João da Silva",
    "payer_document": "12345678900"
  }
}
```

`status` possíveis: `pending` | `paid` | `expired` | `refunded` | `failed`.

**NÃO use polling agressivo.** O fluxo correto é receber via webhook (§5). Use essa rota só pra reconciliação manual.

---

### 3.3 — Pagar PIX via chave (cash-out)

```
POST /v1/pix/payment
```

**Request body:**

```json
{
  "external_id": "withdraw-001",
  "value_cents": 5000,
  "pix_key_type": "cpf",
  "pix_key": "12345678900",
  "receiver_name": "Maria Souza",
  "receiver_document": "12345678900",
  "postbackUrl": "https://seusite.com/webhooks/paysure",
  "description": "Saque pedido #001",
  "splits": []
}
```

**Campos:**

| Campo | Tipo | Obrigatório | Notas |
|---|---|---|---|
| `external_id` | string ≤191 | ✅ | Único por transação. |
| `value_cents` | int ≥100 | ✅ | |
| `pix_key_type` | enum | ✅ | `cpf` \| `cnpj` \| `email` \| `phone` \| `evp` |
| `pix_key` | string ≤255 | ✅ | A chave em si (formato deve bater com `pix_key_type`). |
| `receiver_name` | string ≤191 | ✅ | Nome do destinatário. |
| `receiver_document` | string ≤32 | ✅ | CPF/CNPJ do destinatário. |
| `postbackUrl` | URL HTTPS | ✅ | |
| `description` | string ≤255 | ❌ | |
| `splits` | array ≤20 | ❌ | Idêntico ao cash-in. |

**Validações locais antes de enviar pro provedor:**
- CPF/CNPJ: dígitos verificadores (módulo 11)
- email: formato válido
- phone: padrão E.164 (`+55XXXXXXXXXXX`)
- evp: UUID v4

Se a chave for malformada, recebemos `HTTP 422` imediatamente (sem custo).

**Response 201:**

```json
{
  "cashout": {
    "reference_code": "PSR-1234abcd-...",
    "external_reference": "withdraw-001",
    "status": "processing"
  }
}
```

A wallet do user é **debitada imediatamente** quando o cash-out é aceito. Se a adquirente recusar depois, o valor é **estornado automaticamente** (status final `refunded` via webhook) — o cliente integrador não precisa fazer nada do lado dele do ponto de vista de saldo Paysure.

**Erros típicos:**
- `401 Saldo insuficiente.` → saldo da conta menor que `value_cents + taxas + splits`.
- `401 Essa retirada excede o valor máximo por transação.` → acima do limite por transação do user.
- `401 Valor máximo diário atingido.` → soma de cash-outs do dia ultrapassou limite.
- `422 CPF inválido #PV02` → validação local de chave falhou.
- `503` → provedor PIX indisponível, retente.

---

### 3.4 — Consultar cash-out

```
POST /v1/pix/cashout/consult
```

```json
{ "reference_code": "PSR-1234abcd-..." }
```

**Response 200:**

```json
{
  "cashout": {
    "reference_code": "PSR-1234abcd-...",
    "external_reference": "withdraw-001",
    "status": "paid",
    "end_to_end": "E29477089202605131013...",
    "occurred_at": "2026-05-13T10:13:01-03:00"
  }
}
```

`status` possíveis: `processing` | `paid` | `refunded` | `failed`.

---

### 3.5 — Chave PIX estática (copia-e-cola permanente)

```
GET /v1/me/pix-static
```

Sem body. Retorna o copia-e-cola fixo da conta — útil quando o pagador escolhe o valor (doação, gorjeta, link na bio).

**Response 200:**

```json
{
  "pix_static": {
    "pix_key": "92277bd7-26f6-46dd-...",
    "pix_key_type": "evp",
    "merchant_name": "PAYSURE",
    "merchant_city": "SAO PAULO",
    "copia_cola": "00020126580014br.gov.bcb.pix..."
  }
}
```

Pagamentos feitos via este copia-e-cola caem como **cash-in normais** no `postbackUrl` cadastrado na conta — mesma estrutura de webhook de §5.

---

## 4. Splits

Em qualquer cash-in ou cash-out, o array `splits[]` distribui % do bruto pra outros users Paysure (que ficam com o crédito imediatamente quando a transação principal completa).

```json
{
  "value_cents": 10000,
  "splits": [
    { "clientId": "parceiro_a", "value": 10 },
    { "clientId": "parceiro_b", "value": 5 }
  ]
}
```

No exemplo (R$ 100,00 cobrados):
- R$ 10,00 (10%) → `parceiro_a`
- R$ 5,00 (5%) → `parceiro_b`
- R$ 85,00 menos taxas → quem criou a cobrança

**Regras:**
- `value` é PERCENTUAL (1 = 1%, 5 = 5%) — NÃO centavos.
- `clientId` é o **username Paysure** do destinatário (não o `external_id` da sua aplicação).
- Soma dos splits ≤ 100% (senão `422`).
- Cash-in: até 10 splits. Cash-out: até 20.
- Se `clientId` não existe como user, o valor fica **retido** até alocação manual. Valide com a Paysure antes de habilitar splits dinâmicos.
- Em `refunded`, os splits são estornados automaticamente.

---

## 5. Webhooks

A Paysure POSTará na URL `postbackUrl` que você forneceu na criação da transação. O cliente integrador precisa expor um endpoint HTTPS pra receber.

### 5.1 Envelope cash-in (recebimento confirmado)

```json
{
  "cashin": {
    "reference_code": "PSR-9c8f7a1e-...",
    "external_reference": "pedido-001",
    "value_cents": 5000,
    "status": "paid",
    "payer_name": "João da Silva",
    "payer_document": "12345678900",
    "payment_date": "2026-05-13T10:13:01-03:00",
    "registration_date": "2026-05-13T10:12:15-03:00",
    "end_to_end": "E18236120202605131013...",
    "content": "00020126580014br.gov.bcb.pix..."
  }
}
```

Status possíveis em webhook cash-in: `paid`, `refunded`, `expired`.

### 5.2 Envelope cash-out (pagamento enviado)

```json
{
  "cashout": {
    "reference_code": "PSR-1234abcd-...",
    "external_reference": "withdraw-001",
    "status": "paid",
    "end_to_end": "E29477089202605131013...",
    "occurred_at": "2026-05-13T10:13:01-03:00"
  }
}
```

Status possíveis em webhook cash-out: `paid`, `refunded`.

### 5.3 Regras de implementação do handler

| Regra | Detalhe |
|---|---|
| **Responder HTTP 200** | Sempre. Mesmo se não reconhece o evento, devolva `{"ok":true}` em 200. |
| **Timeout** | Tem até 10 segundos pra responder. Se demorar mais, é retentado. |
| **Idempotência** | Webhooks podem chegar mais de uma vez. Deduplique por `cashin.reference_code` / `cashout.reference_code`. |
| **Retry policy** | Em falha: 1ª retry imediata, 2ª em ~5 min, 3ª em ~30 min. Após 3 falhas, vai pra fila manual no painel. |
| **Origem** | Vem de IPs Cloudflare. Validação por IP é opcional; HMAC assinada está no roadmap. |

### 5.4 Pseudo-código do handler

```
POST /webhooks/paysure
  payload = json
  ref_code = payload.cashin?.reference_code OR payload.cashout?.reference_code
  type = payload.cashin ? 'cashin' : 'cashout'
  status = payload[type].status

  IF webhookAlreadyProcessed(ref_code, status):
    return 200 { ok: true }

  IF type == 'cashin' AND status == 'paid':
    creditar_produto_ao_cliente(external_reference)
  ELSE IF type == 'cashin' AND status == 'refunded':
    estornar_produto(external_reference)
  ELSE IF type == 'cashout' AND status == 'paid':
    marcar_saque_concluido(external_reference)
  ELSE IF type == 'cashout' AND status == 'refunded':
    desbloquear_saldo_user_origem(external_reference)

  markWebhookProcessed(ref_code, status)
  return 200 { ok: true }
```

---

## 6. Tabela de erros HTTP (resumo)

| HTTP | Quando | Ação sugerida |
|---|---|---|
| 200 | Sucesso de consulta | — |
| 201 | Recurso criado | — |
| 401 | Auth falhou / credenciais inativas / regras de negócio (saldo, duplicidade) | Trate como business error. |
| 403 | Conta inativa | Contate suporte. |
| 404 | Recurso não encontrado | Confirme `reference_code`. |
| 409 | Conflito (duplicidade) | Use outro `external_id`. |
| 422 | Validação | Confira campos. |
| 429 | Rate limit | Aguarde `Retry-After`. |
| 500 | Erro interno | Retente. |
| 501 | Operação indisponível pra esse recurso | — |
| 503 | Provedor PIX indisponível | Retente em alguns minutos. |

**Rate limits padrão:**
- `POST /v1/pix/qrcode` — definido por chave/conta
- `POST /v1/pix/payment` — 60/minuto por user
- preview/consult — 120/minuto por user

---

## 7. Snippets prontos por linguagem

### 7.1 — Node.js (fetch nativo)

```js
const PAYSURE_BASE = 'https://api.paysurebr.com';

async function paysure(path, body) {
  const r = await fetch(`${PAYSURE_BASE}${path}`, {
    method: 'POST',
    headers: {
      ci: process.env.PAYSURE_CI,
      cs: process.env.PAYSURE_CS,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(body),
  });
  const data = await r.json();
  if (!r.ok) throw new Error(`Paysure ${r.status}: ${data.message || r.statusText}`);
  return data;
}

// Cash-in
const { qrcode } = await paysure('/v1/pix/qrcode', {
  external_id: 'pedido-001',
  value_cents: 5000,
  postbackUrl: 'https://seusite.com/webhooks/paysure',
});
console.log(qrcode.content); // BR Code

// Cash-out
const { cashout } = await paysure('/v1/pix/payment', {
  external_id: 'withdraw-001',
  value_cents: 5000,
  pix_key_type: 'cpf',
  pix_key: '12345678900',
  receiver_name: 'Maria',
  receiver_document: '12345678900',
  postbackUrl: 'https://seusite.com/webhooks/paysure',
});
```

### 7.2 — Node.js webhook handler (Express)

```js
import express from 'express';
const app = express();
app.use(express.json());

// Set<string> em memória pra dedupe; em produção use Redis ou DB.
const processed = new Set();

app.post('/webhooks/paysure', async (req, res) => {
  const { cashin, cashout } = req.body;
  const event = cashin || cashout;
  if (!event?.reference_code) return res.status(200).json({ ok: true });

  const dedupeKey = `${event.reference_code}:${event.status}`;
  if (processed.has(dedupeKey)) return res.status(200).json({ ok: true });
  processed.add(dedupeKey);

  try {
    if (cashin?.status === 'paid')      await onCashinPaid(cashin);
    if (cashin?.status === 'refunded')  await onCashinRefunded(cashin);
    if (cashout?.status === 'paid')     await onCashoutPaid(cashout);
    if (cashout?.status === 'refunded') await onCashoutRefunded(cashout);
  } catch (e) {
    console.error('paysure webhook err', e);
    // Mesmo em erro, devolva 200 — Paysure retenta automaticamente baseado no dedupe da SUA tabela.
  }
  res.status(200).json({ ok: true });
});

async function onCashinPaid(ev)      { /* creditar produto */ }
async function onCashinRefunded(ev)  { /* estornar produto */ }
async function onCashoutPaid(ev)     { /* marcar saque ok */ }
async function onCashoutRefunded(ev) { /* avisar user que falhou */ }

app.listen(3000);
```

### 7.3 — PHP (cURL)

```php
<?php
function paysure(string $path, array $body): array {
    $ch = curl_init('https://api.paysurebr.com' . $path);
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST           => true,
        CURLOPT_HTTPHEADER     => [
            'ci: ' . getenv('PAYSURE_CI'),
            'cs: ' . getenv('PAYSURE_CS'),
            'Content-Type: application/json',
        ],
        CURLOPT_POSTFIELDS     => json_encode($body),
    ]);
    $resp = curl_exec($ch);
    $code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    $data = json_decode($resp, true) ?: [];
    if ($code >= 400) throw new RuntimeException("Paysure $code: " . ($data['message'] ?? ''));
    return $data;
}

// Cash-in
$r = paysure('/v1/pix/qrcode', [
    'external_id' => 'pedido-001',
    'value_cents' => 5000,
    'postbackUrl' => 'https://seusite.com/webhooks/paysure',
]);
echo $r['qrcode']['content']; // BR Code
```

### 7.4 — PHP webhook handler (Laravel)

```php
Route::post('/webhooks/paysure', function (Request $request) {
    $body  = $request->json()->all();
    $event = $body['cashin'] ?? $body['cashout'] ?? null;
    if (!$event || empty($event['reference_code'])) {
        return response()->json(['ok' => true]);
    }
    $kind  = isset($body['cashin']) ? 'cashin' : 'cashout';
    $key   = "{$event['reference_code']}:{$event['status']}";

    if (DB::table('paysure_webhook_log')->where('dedup', $key)->exists()) {
        return response()->json(['ok' => true]);
    }
    DB::table('paysure_webhook_log')->insert([
        'dedup' => $key, 'created_at' => now()
    ]);

    match (true) {
        $kind === 'cashin'  && $event['status'] === 'paid'     => onCashinPaid($event),
        $kind === 'cashin'  && $event['status'] === 'refunded' => onCashinRefunded($event),
        $kind === 'cashout' && $event['status'] === 'paid'     => onCashoutPaid($event),
        $kind === 'cashout' && $event['status'] === 'refunded' => onCashoutRefunded($event),
        default => null,
    };

    return response()->json(['ok' => true]);
});
```

### 7.5 — Python (requests)

```python
import os, requests

PAYSURE = 'https://api.paysurebr.com'
HEADERS = {
    'ci': os.environ['PAYSURE_CI'],
    'cs': os.environ['PAYSURE_CS'],
    'Content-Type': 'application/json',
}

def paysure(path: str, body: dict) -> dict:
    r = requests.post(f'{PAYSURE}{path}', headers=HEADERS, json=body, timeout=15)
    data = r.json() if r.content else {}
    if not r.ok:
        raise RuntimeError(f"Paysure {r.status_code}: {data.get('message')}")
    return data

# Cash-in
r = paysure('/v1/pix/qrcode', {
    'external_id': 'pedido-001',
    'value_cents': 5000,
    'postbackUrl': 'https://seusite.com/webhooks/paysure',
})
print(r['qrcode']['content'])
```

### 7.6 — Python webhook handler (Flask)

```python
from flask import Flask, request, jsonify

app = Flask(__name__)
processed = set()  # use Redis em prod

@app.post('/webhooks/paysure')
def paysure_webhook():
    body = request.get_json(silent=True) or {}
    event = body.get('cashin') or body.get('cashout')
    if not event or not event.get('reference_code'):
        return jsonify(ok=True), 200

    kind = 'cashin' if 'cashin' in body else 'cashout'
    key = f"{event['reference_code']}:{event['status']}"
    if key in processed:
        return jsonify(ok=True), 200
    processed.add(key)

    if kind == 'cashin' and event['status'] == 'paid':
        on_cashin_paid(event)
    elif kind == 'cashin' and event['status'] == 'refunded':
        on_cashin_refunded(event)
    elif kind == 'cashout' and event['status'] == 'paid':
        on_cashout_paid(event)
    elif kind == 'cashout' and event['status'] == 'refunded':
        on_cashout_refunded(event)

    return jsonify(ok=True), 200
```

### 7.7 — Go (net/http)

```go
package paysure

import (
  "bytes"
  "encoding/json"
  "fmt"
  "net/http"
  "os"
)

const Base = "https://api.paysurebr.com"

func Call(path string, body any) (map[string]any, error) {
  buf, _ := json.Marshal(body)
  req, _ := http.NewRequest("POST", Base+path, bytes.NewReader(buf))
  req.Header.Set("ci", os.Getenv("PAYSURE_CI"))
  req.Header.Set("cs", os.Getenv("PAYSURE_CS"))
  req.Header.Set("Content-Type", "application/json")

  resp, err := http.DefaultClient.Do(req)
  if err != nil { return nil, err }
  defer resp.Body.Close()

  var out map[string]any
  json.NewDecoder(resp.Body).Decode(&out)
  if resp.StatusCode >= 400 {
    return out, fmt.Errorf("paysure %d: %v", resp.StatusCode, out["message"])
  }
  return out, nil
}
```

---

## 8. Checklist final (use isso pra validar que terminou a integração)

- [ ] Variáveis `PAYSURE_CI` e `PAYSURE_CS` no `.env` (não commitadas).
- [ ] Wrapper/cliente HTTP que injeta os headers `ci`/`cs` automaticamente em toda request.
- [ ] Função `createPixCashin(externalId, valueCents, postbackUrl, extras?)` que retorna `{ reference_code, content }`.
- [ ] Função `payPixCashout(externalId, valueCents, pixKey, pixKeyType, receiverName, receiverDoc, postbackUrl)` que retorna `{ reference_code }`.
- [ ] Endpoint público HTTPS `POST /webhooks/paysure` que:
  - aceita JSON
  - detecta `cashin` vs `cashout`
  - **deduplica** por `reference_code + status`
  - dispara handlers de negócio (creditar produto, estornar etc)
  - **sempre devolve HTTP 200**
- [ ] Persistência de transações no DB do cliente (no mínimo: `external_id`, `reference_code`, `status`, `value_cents`, `created_at`).
- [ ] Tratamento de erro pra HTTP 401/422/503 (logar + sinalizar usuário ou retentar).
- [ ] Geração visual do QR Code a partir de `qrcode.content` no frontend (lib client-side).
- [ ] (Opcional) Tela de admin que lista transações + permite reenviar webhook manualmente caso o cliente perca um evento.

---

## 9. Glossário

| Termo | Significado |
|---|---|
| `external_id` | ID único da transação no sistema DO CLIENTE integrador (idempotency key). |
| `reference_code` | ID interno Paysure (formato `PSR-...`). Use em consultas e webhooks. |
| `end_to_end` | ID BACEN da transação PIX. |
| Cash-in | Recebimento (cliente paga, sua conta credita). |
| Cash-out | Pagamento (sua conta debita, alguém recebe). |
| BR Code | String do PIX copia-e-cola (formato EMV/TLV). |
| Split | Divisão automática em % entre múltiplos users Paysure. |
| Webhook | POST que a Paysure envia pro `postbackUrl` em eventos. |

---

## 10. Perguntas que o agente DEVE fazer pro usuário antes de implementar

1. **Stack do projeto?** (linguagem + framework + ORM) — pra escolher o snippet certo de §7.
2. **Onde guardar as credenciais?** (`.env` é o padrão — mas se já tem secret manager, use ele).
3. **URL pública pro webhook?** (precisa ser HTTPS, exposta à internet — em dev pode usar ngrok/cloudflared).
4. **Já tem tabela de pedidos/transações no DB?** — pra integrar `external_id` + `reference_code`.
5. **Quer split em alguma operação?** — se sim, peça os usernames Paysure dos destinatários.

Não pergunte mais nada além disso. Tudo o resto está nesta doc.

---

## 11. Suporte humano

Se você travar em algo não coberto aqui, oriente o usuário a abrir ticket em `suporte@paysurebr.com` com:
- `reference_code` da transação afetada
- HTTP code recebido
- payload enviado (sem `cs`!)
- payload de resposta

---

**Fim.** Boa implementação. 🚀
