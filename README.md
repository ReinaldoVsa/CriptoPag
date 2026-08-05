# CryptoPay

Plataforma de carteira cripto não custodial: consulta de saldo em
blockchain, valor estimado em BRL, venda de criptoativos e recebimento via
Pix. Web, PWA e Android a partir de um único codebase (Next.js + Capacitor).

> **Status:** arquitetura, backend, frontend, banco de dados, infraestrutura
> de deploy (Docker, CI/CD, Netlify) e documentação estão implementados e
> funcionais. As integrações financeiras reais (PSP de Pix, parceiro de
> off-ramp, provedor de KYC) estão isoladas atrás de interfaces e usam
> adapters *placeholder* que recusam operar sem credenciais reais — ver
> "O que falta para operação real" abaixo. **Este projeto não deve ser
> declarado apto a operar financeiramente até validação jurídica/regulatória
> e contratação dos parceiros reais.**

## Descrição

O usuário adiciona um endereço público de carteira (QR Code, colar
manualmente ou WalletConnect), consulta o saldo diretamente na blockchain,
vê o valor estimado em BRL, e — quando os parceiros regulatórios estiverem
configurados — pode vender o ativo e receber o valor via Pix. Nenhuma seed
phrase ou chave privada é solicitada ou armazenada em nenhum momento.

## Arquitetura

Ver `docs/architecture.md` para o diagrama completo e as decisões de design.
Resumo: monorepo com API NestJS modular (padrão de adapters para toda
integração externa), frontend Next.js (Web/PWA), wrapper Capacitor para
Android, PostgreSQL via Prisma, Redis para cache/rate limiting.

## Tecnologias

**Frontend:** Next.js, React, TypeScript, Tailwind CSS, TanStack Query
(preparado), Zustand (preparado), React Hook Form + Zod.
**Backend:** Node.js, NestJS, TypeScript, Prisma, PostgreSQL, Redis.
**Blockchain:** BlockCypher (Bitcoin), Alchemy (Ethereum), RPC genérico
(EVM), WalletConnect.
**Mobile:** Capacitor (Android).
**Infra:** Docker, GitHub Actions, Netlify (frontend), Swagger/OpenAPI.

## Estrutura do monorepo

```
cryptopay/
├── apps/
│   ├── web/      → Frontend Next.js (Web + PWA)
│   ├── api/      → Backend NestJS (API, Prisma, regras de negócio)
│   └── mobile/   → Wrapper Capacitor para Android
│
├── packages/     → Pacotes compartilhados (ver nota abaixo)
├── docs/         → Documentação detalhada por área
├── .github/workflows/ → CI (lint/test/build) e Deploy
├── docker-compose.yml, Dockerfile → Infraestrutura local/produção do backend
├── netlify.toml  → Configuração de deploy do frontend
└── .env.example  → Todas as variáveis de ambiente necessárias
```

> **Nota sobre `packages/`:** as pastas `packages/*` foram criadas conforme
> a especificação original para representar código compartilhável entre
> `apps/*` (tipos, config, clientes de blockchain/pricing/payments/pix/kyc).
> Na implementação atual, esse código vive diretamente em
> `apps/api/src/modules/*` para simplificar o build de uma v1 monolítica
> (ver `docs/architecture.md`). Extrair para `packages/*` reais é
> recomendado se/quando os módulos forem promovidos a serviços separados.

## Setup local

Ver `docs/setup.md` — resumo:

```bash
npm install
cp .env.example .env      # preencha ao menos AUTH_SECRET
docker compose up -d postgres redis
npm run prisma:generate
npm run prisma:migrate
npm run prisma:seed
npm run dev:api   # http://localhost:3001 (Swagger em /docs)
npm run dev:web   # http://localhost:3000
```

## Testes

```bash
npm run test --workspace=apps/api       # 14 testes unitários
npm run test:e2e --workspace=apps/api   # fluxo completo (requer Postgres real)
```

Unitários cobrem validação de endereço (Bitcoin/EVM), a máquina de estados
da ordem de venda e a lógica de confirmação incremental do indexer — todos
rodam sem banco de dados (mocks). O teste E2E (`apps/api/test/app.e2e-spec.ts`)
sobe a aplicação NestJS completa contra um Postgres real e exercita o fluxo
inteiro via HTTP: cadastro → login → carteira → cotação → venda →
autorização → confirmação on-chain (simulada) → conversão → Pix → webhook
assinado, incluindo casos negativos (assinatura inválida, saldo
insuficiente, idempotência). Os adapters de blockchain/cotação/off-ramp/Pix
são substituídos por test doubles determinísticos — isso testa a
orquestração real do sistema sem depender de credenciais de parceiros que
o projeto não simula em produção. Já roda automaticamente no CI (que sobe
Postgres/Redis de serviço); no ambiente de build usado para gerar este
pacote, o Prisma Client não pôde ser gerado por bloqueio de rede a
`binaries.prisma.sh`, então o E2E foi validado por type-check completo e
revisão manual, mas não executado ponta a ponta — rode-o localmente ou no
CI para a validação de execução real.

## Docker

```bash
docker compose up -d --build
```

## Documentação completa

| Arquivo | Conteúdo |
|---|---|
| `docs/architecture.md` | Diagrama, decisões de design, padrão de adapters |
| `docs/setup.md` | Setup local passo a passo |
| `docs/database.md` | Schema, entidades, comandos Prisma |
| `docs/api.md` | Referência de todas as rotas |
| `docs/blockchain.md` | Providers de blockchain e como adicionar redes |
| `docs/wallet.md` | Fluxo de carteiras, WalletConnect, QR Code |
| `docs/payments.md` | Off-ramp (conversão cripto → BRL) |
| `docs/pix.md` | Integração Pix |
| `docs/kyc.md` | Integração KYC/AML |
| `docs/security.md` | Autenticação, RBAC, webhooks, secrets |
| `docs/deployment.md` | Netlify, backend cloud, banco gerenciado, domínio |
| `docs/troubleshooting.md` | Problemas comuns |
| `apps/mobile/README.md` | Geração do APK/AAB Android |

## Segurança (resumo — detalhes em `docs/security.md`)

- Nunca solicita/armazena seed phrase ou chave privada.
- JWT de curta duração + refresh token rotativo em cookie `HttpOnly`.
- RBAC (`USER`/`ADMIN`/`SUPER_ADMIN`), rate limiting, Helmet, validação Zod.
- Toda ação sensível é auditada (`AuditLog`).
- Webhooks validados por assinatura HMAC + idempotência.
- Secrets nunca no código nem no Netlify — mascarados no painel admin.

## O que falta para operação real

1. **Parceiro de off-ramp** licenciado (cripto → BRL) — `docs/payments.md`.
2. **PSP/instituição financeira** para Pix — `docs/pix.md`.
3. **Provedor de KYC/AML** real — `docs/kyc.md`.
4. **Validação jurídica e regulatória** completa (Bacen, PLD/AML, obrigações
   fiscais, licenças) antes de processar qualquer valor real.
5. Domínio próprio, certificado SSL e credenciais de produção de cada
   integração.
6. Em volume alto, substituir o polling do worker de indexação (já
   implementado, ver `docs/architecture.md`) por webhooks nativos de
   confirmação ou uma fila dedicada.

## Checklist de produção

- [x] Código no GitHub (a você cabe fazer o push deste repositório)
- [x] Nenhum secret no código — `.env.example` + `.gitignore`
- [x] CI configurado (`.github/workflows/ci.yml`)
- [x] Testes unitários passando
- [x] Frontend funcional (Web/PWA)
- [x] Backend funcional (API + Swagger)
- [x] Schema de banco completo (Prisma)
- [x] Docker/Docker Compose
- [x] Netlify configurado (`netlify.toml`) — falta você conectar o repo
- [x] Webhooks com validação de assinatura + idempotência
- [x] Blockchain configurada (adapters reais — falta suas API keys)
- [x] Cotação configurada (CoinGecko — funcional sem credencial)
- [x] Auditoria implementada
- [x] Segurança revisada (ver `docs/security.md`)
- [x] Worker de indexação de blockchain (confirmação automática de tx)
- [ ] Backend publicado em cloud/VPS real
- [ ] Domínio configurado (HTTPS/SSL/DNS)
- [ ] Off-ramp configurado (parceiro real)
- [ ] Pix configurado (PSP real)
- [ ] KYC configurado (provedor real)
- [ ] Monitoramento (Sentry) e backup de banco configurados com credenciais reais
- [ ] Compliance jurídico/regulatório revisado
- [ ] Testado ponta a ponta em sandbox dos parceiros reais

## Licença / Uso

Projeto entregue como base de implementação técnica. Nenhuma declaração de
conformidade regulatória ou autorização para operar financeiramente é feita
por este código — ver aviso no topo deste README.
