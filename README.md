# InstaOrg SaaS — Gestão Multi-Contas Instagram (Multi-Tenant)

Aplicação web em português para gestão de conteúdo e operação em múltiplas contas no Instagram, com autenticação própria, isolamento completo por organização (Multi-Tenant), fila em segundo plano (BullMQ + Redis), lock distribuído por conta (`account-lease`), armazenamento privado compatível com S3 (MinIO) e sessão segura com cookie `HttpOnly`.

## Arquitetura Implementada

- **Web Experience (`src/components` & `src/app`)**:
  - `auth-form.tsx`: Login e Cadastro com criação automática de Organização no primeiro acesso.
  - `dashboard.tsx`: Painel com dados de demonstração explícitos (`metrics.ts`), gestão de perfis (`@username` apenas ou com credenciais criptografadas), grupos de contas e biblioteca de mídia privada.
  - `auth.ts`: Sessão segura via cookie `HttpOnly` + validação estrita de identidade e organização (`requireIdentity`).
- **Organization Content (`src/app/api` & `src/lib/domain.ts`)**:
  - `domain.ts`: Regras de domínio, normalização de `@username`, validação de tipos MIME e chaves isoladas no S3 (`orgs/{organizationId}/media/...`).
  - `/api/profiles`: Leitura e cadastro de perfis do Instagram isolados por organização.
  - `/api/groups`: Criação e listagem de grupos de contas com proteção anti-IDOR.
  - `/api/media/prepare`: Registro de mídia `PENDING_UPLOAD` e assinatura de URL de upload (`PUT` presigned URL).
  - `/api/media/verify`: Verificação do objeto no S3 via `HeadObject` e ativação para `READY`.
  - `/api/media`: Listagem de mídias prontas e assinatura de URL temporária de download.
- **Background Runtime (`src/worker`)**:
  - `index.ts`: Worker BullMQ consumindo filas de publicação e métricas.
  - `account-lease.ts`: Lock distribuído no Redis (`SET NX PX` + script Lua) para impedir operações simultâneas na mesma conta do Instagram.
  - `metrics.ts`: Contratos e snapshot de métricas de demonstração.
  - `queues.ts`: Filas `instagram-publications` e `instagram-metrics`.
  - `session-service/index.ts`: Persistência criptografada (AES-256-GCM) de sessão e credenciais do Instagram.
  - `instagram-adapter.ts`: Adaptador de login privado via `username` e `password` e publicação de mídias.
- **Persistence & Storage**:
  - PostgreSQL + Prisma ORM (`prisma/schema.prisma`) preparado para papéis (`OWNER`, `ADMIN`, `EDITOR`, `VIEWER`) e convites (`OrganizationInvite`).
  - Redis 7 para filas BullMQ e leases de conta.
  - MinIO (S3-compatible) configurado automaticamente com bucket privado.

---

## Como Rodar Localmente com Docker

1. Copie o arquivo de variáveis de ambiente:
   ```bash
   cp .env.example .env
   ```

2. Suba os containers do PostgreSQL, Redis e MinIO:
   ```bash
   docker compose up -d
   ```

3. Instale as dependências e sincronize o banco de dados:
   ```bash
   npm install
   npx prisma db push
   ```

4. Inicie o servidor web e o worker em segundo plano:
   ```bash
   # Terminal 1: Aplicação Next.js
   npm run dev

   # Terminal 2: Background Worker (BullMQ)
   npm run worker
   ```

5. Acesse `http://localhost:3000/registro` para criar sua conta e organização.

---

## Como Subir para o GitHub

```bash
git init
git add .
git commit -m "feat: estrutura completa saas multi-tenant instagram"
git branch -M main
git remote add origin https://github.com/SEU_USUARIO/SEU_REPOSITORIO.git
git push -u origin main
```
