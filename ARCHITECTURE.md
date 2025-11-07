# Documento de Arquitetura — Plataforma de Gestão de Networking

**Autor:** Joclelson Rodrigues
**Objetivo:** Desenhar arquitetura completa e especificar parte da API para a plataforma de gestão de grupos de networking.

## 1. Visão Geral

Plataforma para gerenciar membros, reuniões e gerar/reportar indicações de negócios. MVP focado no fluxo de admissão (intenção → aprovação → cadastro via token).

**Principais Stacks:**

- Frontend: Next.js (App Router) + React + Tailwind
- Backend: Node.js (NestJS) + TypeScript
- Banco de Dados Relacional (Postgres)
- Redis (cache, filas) - Opcional
- Gateway de pagamento (ex: Stripe/Pagar.me) - Opcional
- Armazenamento de arquivos (S3 compatível) - Opcional
- Serviço de entrega de e-mail (SendGrid, Amazon SES) - Opcional
- Workers/Jobs (processamento assíncrono: geração de relatórios, envio de e-mails, reconciliador de pagamentos) - Opcional

---

## 2. Diagrama da Arquitetura

```mermaid
flowchart LR
  subgraph CLIENT
    Browser["Next.js (SSR/SSG) / SPA"]
  end

  subgraph CDN
    CDNNode[CDN (assets)]
  end

  CDNNode --> Browser
  Browser -->|HTTPS| API[API Gateway / Backend (NestJS)]
  API --> Auth[Auth Service (JWT / NextAuth)]
  API --> DB[(Postgres)]
  API --> Redis[(Redis) - cache]
  API --> Storage[S3 (file uploads)]
  API --> Payments[Payments Gateway]
  API --> Mail[Mail Service]

  subgraph WORKERS
    WorkerJobs[Background Workers (BullMQ/Sidekiq-like)]
  end

  Redis --> WorkerJobs
  WorkerJobs --> Mail
  WorkerJobs --> DB

  AdminUI[Admin Panel (Next.js - /admin)] --> API

  note right of DB
    Optionally: read-replica for scaling reads
  end
```

> Diagrama simples em Mermaid.

---

### 2. Modelo de Dados

#### Justificativa da Escolha (PostgreSQL)

> A escolha pelo PostgreSQL se deve à modelo de domínio com muitas relações (membros, reuniões, indicações, pagamentos) se encaixa bem em esquema relacional. Bons recursos para consultas analíticas, úteis para dashboards e relatórios. O PostgreSQL oferece robustez, integra bem com ORMs populares, garantias ACID para transações (essencial para pagamentos e indicações) e grande escalabilidade.

#### Esquema (Tabelas Principais)

```sql
-- Tabela de usuários e autenticação
CREATE TABLE "users" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "email" VARCHAR(255) UNIQUE NOT NULL,
  "name" VARCHAR(255),
  "password_hash" VARCHAR(255) NOT NULL,
  "role" VARCHAR(20) NOT NULL DEFAULT 'member', -- 'member' ou 'admin'
  "created_at" TIMESTAMPTZ DEFAULT now(),
  "updated_at" TIMESTAMPTZ DEFAULT
);

-- Tabela para intenções públicas (Formulário 1)
CREATE TABLE "applications" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "name" VARCHAR(255) NOT NULL,
  "email" VARCHAR(255) NOT NULL,
  "company" VARCHAR(255),
  "reason" TEXT,
  "reviewer_id" UUID REFERENCES "users"("id"),
  "status" VARCHAR(20) NOT NULL DEFAULT 'pending', -- 'pending', 'approved', 'rejected'
  "created_at" TIMESTAMPTZ DEFAULT now(),
  "updated_at" TIMESTAMPTZ DEFAULT
);

-- Tabela para convites (gerados na aprovação)
CREATE TABLE "invitations" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "application_id" UUID REFERENCES "applications"("id"),
  "token" VARCHAR(255) UNIQUE NOT NULL,
  "is_used" BOOLEAN DEFAULT false,
  "expires_at" TIMESTAMPTZ NOT NULL,
  "created_at" TIMESTAMPTZ DEFAULT now(),
  "updated_at" TIMESTAMPTZ DEFAULT
);

-- Tabela de membros (Cadastro Completo)
CREATE TABLE "members" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "user_id" UUID UNIQUE NOT NULL REFERENCES "users"("id"),
  "invitation_id" UUID UNIQUE REFERENCES "invitations"("id"),
  "full_name" VARCHAR(255) NOT NULL,
  "phone" VARCHAR(50),
  "active" BOOLEAN NOT NULL DEFAULT true,
  "created_at" TIMESTAMPTZ DEFAULT now(),
  "updated_at" TIMESTAMPTZ DEFAULT
);

-- Tabela para o Sistema de Indicações (opcional)
CREATE TABLE "referrals" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "from_member_id" UUID NOT NULL REFERENCES "members"("id"), -- Quem indicou
  "to_member_id" UUID NOT NULL REFERENCES "members"("id"), -- Quem foi indicado
  "contact_name" VARCHAR(255) NOT NULL,
  "contact_company" VARCHAR(255),
  "description" TEXT NOT NULL,
  "status" VARCHAR(50) NOT NULL DEFAULT 'sent', -- 'sent', 'negotiating', 'closed', 'rejected'
  "created_at" TIMESTAMPTZ DEFAULT now(),
  "updated_at" TIMESTAMPTZ DEFAULT
);

-- Tabela para agradecimentos (opcional)
CREATE TABLE "thank_yous" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "sender_member_id" UUID NOT NULL REFERENCES "members"("id"),
  "receiver_member_id" UUID NOT NULL REFERENCES "members"("id"),
  "referral_id" UUID REFERENCES "referrals"("id"), -- Opcional, se veio de uma indicação
  "amount" DECIMAL(10, 2),
  "message" TEXT,
  "created_at" TIMESTAMPTZ DEFAULT now()
);

-- Tabela para controle de mensalidades  (opcional)
CREATE TABLE "payments" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "member_id" UUID NOT NULL REFERENCES "members"("id"),
  "amount" DECIMAL(10, 2) NOT NULL,
  "due_date" DATE NOT NULL,
  "status" VARCHAR(20) NOT NULL DEFAULT 'pending', -- 'pending', 'paid', 'overdue'
  "created_at" TIMESTAMPTZ DEFAULT now()
);
```

### 3. Estrutura de Componentes (Frontend)

O frontend será estruturado usando a **App Router** do Next.js, com uma organização focada na separação de responsabilidades:

- `src/app/`

  - Contém as rotas da aplicação (ex: `(public)/apply/page.tsx`, `(admin)/dashboard/page.tsx`). Usamos _Route Groups_ para separar layouts públicos, de membros e de admin.

- `src/components/ui/`

  - Componentes "burros" (Dumb Components) e reutilizáveis, sem lógica de negócio.
  - Ex: `Button.tsx`, `Input.tsx`, `Card.tsx`, `Select.tsx`, `AlertDialog.tsx`.

- `src/components/features/`

  - Componentes "inteligentes" (Smart Components) que contêm lógica de negócio e estado para uma funcionalidade específica.
  - Ex: `ApplicationForm.tsx` (que usa `Input` e `Button` do `ui/`), `AdminApplicationList.tsx`, `ReferralForm.tsx`.

- `src/hooks/`

  - hooks customizados

- `src/lib/`

  - Utilitários e clientes de API.
  - Ex: `api.ts` (uma instância do Axios ou wrapper do `fetch`), `utils.ts` (formatação de datas, etc.), `hooks/useAuth.ts`.

- `src/contexts/`
  - Para gerenciamento de estado global. O estado de autenticação (`AuthContext.tsx`) é o candidato principal, gerenciando os dados do usuário logado.

### 4. Definição da API (RESTful)

A API será construída com NestJS, seguindo os padrões RESTful.

---

**Funcionalidade 1: Fluxo de Admissão de Membros**

- **`POST /applications`** (Pública)

  - **Descrição:** Envia um novo formulário de intenção.
  - **Request Body:** `{ "name": string, "email": string, "company": string, "reason": string }`
  - **Response (201):** `{ "id": "uuid", "name": string, "email": string, "status": "pending" }`

- **`GET /admin/applications`** (Privada: Admin)

  - **Descrição:** Lista todas as intenções submetidas.
  - **Response (200):** `[{ "id": "uuid", "name": string, "email": string, "status": string, "created_at": "date" }]`

- **`PATCH /admin/applications/:id/approve`** (Privada: Admin)

  - **Descrição:** Aprova uma intenção, cria um `invitation` e retorna o link (simulado).
  - **Response (200):** `{ "invitation_token": "uuid", "message": "Convite gerado." }`
  - _(No console do backend, logamos: "Link de convite: /register?token=uuid")_

- **`GET /invitations/:token`** (Pública)

  - **Descrição:** Busca um token de convite para validar se ele é real e não foi usado.
  - **Response (200):** `{ "email": "email_da_application", "valid": true }`
  - **Response (404):** `{ "message": "Convite inválido ou expirado", "valid": false }`

- **`POST /members/register`** (Pública)
  - **Descrição:** Cria o cadastro completo do membro (cria `user` e `member`).
  - **Request Body:** `{ "token": string, "fullName": string, "phone": string, "password": string }`
  - **Response (201):** `{ "id": "uuid", "email": string, "fullName": string }`

---

**Funcionalidade 2: Sistema de Indicações (Opção A)**

- **`POST /referrals`** (Privada: Membro)

  - **Descrição:** Um membro logado cria uma nova indicação.
  - **Request Body:** `{ "referredMemberId": "uuid", "contactName": string, "contactCompany": string, "description": string }`
  - **Response (201):** `{ "id": "uuid", "status": "sent", ... }`

- **`GET /referrals/me`** (Privada: Membro)

  - **Descrição:** Lista todas as indicações feitas e recebidas pelo membro logado.
  - **Response (200):** `{ "made": [...], "received": [...] }`

- **`PATCH /referrals/:id/status`** (Privada: Membro)
  - **Descrição:** Atualiza o status de uma indicação recebida (ex: 'negociando').
  - **Request Body:** `{ "status": "negotiating" | "closed" | "rejected" }`
  - **Response (200):** `{ "id": "uuid", "status": "negotiating" }`

---

<!-- **Funcionalidade 3: Gestão de Avisos**

- **`GET /announcements`** (Privada: Membro)

  - **Descrição:** Lista todos os avisos e comunicados para membros.
  - **Response (200):** `[{ "id": "uuid", "title": string, "content": string, "created_at": "date" }]`

- **`POST /admin/announcements`** (Privada: Admin)
  - **Descrição:** Admin cria um novo aviso.
  - **Request Body:** `{ "title": string, "content": string }`
  - **Response (201):** `{ "id": "uuid", "title": string, ... }` -->
