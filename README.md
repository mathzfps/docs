# Paysure Docs

Documentação pública da Paysure API, publicada via [Mintlify](https://mintlify.com).

## Deploy

1. Criar repositório no GitHub (privado ou público): `paysure/paysure-docs`
2. Push deste diretório pro repo
3. No painel Mintlify (https://dashboard.mintlify.com):
   - Conectar com o GitHub
   - Selecionar o repo `paysure-docs`
   - Mintlify auto-detecta `mint.json` e `openapi.yaml`
4. (Opcional) Custom domain: `docs.paysurebr.com` → CNAME pra `cname.mintlify.app`

## Preview local

```bash
npm i -g mintlify
mintlify dev
```

Abre `http://localhost:3000` com hot reload.

## Estrutura

- `mint.json` — config (navegação, cores, links)
- `openapi.yaml` — spec OpenAPI 3.0 dos endpoints
- `*.mdx` — páginas de conteúdo (markdown + JSX)
- `api-reference/` — páginas geradas a partir do openapi.yaml com contexto extra

## O que NÃO deve estar aqui

- Rotas internas (`/admin/*`, `/receive-wh/*`, rotas com path-secret)
- Chaves PIX da Paysure (`92277bd7-...`)
- Detalhes de adquirentes específicas (OnlyU/Horizon/Pluggou/etc) — sempre falar genericamente "provedor PIX"
- Schemas internos do DB
- IPs internos / Tailscale / hostnames

Se algo sensível for adicionado por engano, remover do histórico git antes do push público.
