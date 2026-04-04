# 🏗️ Infraestrutura — duopen-2026

Documentação técnica completa da infraestrutura do projeto **duopen-2026**, cobrindo pipeline de dados, estratégia de compressão, banco de dados, agendamento de jobs e CI/CD.

---

## ✅ Status da Configuração

Configurações já realizadas no Supabase:

- ✅ Migração do banco `duopen-backend` → Supabase (concluída)
- ✅ Extensão **pgvector** ativada (embeddings RAG)
- ✅ Extensão **PostGIS** ativada (dados geoespaciais)
- ✅ Extensão **pg_cron** ativada (agendamento de jobs)
- ⏳ **RLS** (Row Level Security) — pendente de configuração
- ⏳ **GitHub Secrets** — pendente de configuração
- ⏳ **Supabase CLI** — pendente de vinculação

---

## 📁 Estrutura do Diretório

```
duopen-2026/
├── infra/
│   └── supabase/            # Migrations SQL, schemas, pg_cron
└── .github/
    └── workflows/
        ├── coleta.yml       # Scrapers agendados (TCE-RJ, IBGE, Portal Transparência)
        ├── backend.yml      # Deploy contínuo no Laravel Cloud
        └── frontend.yml     # Deploy do React
```

---

## 🔄 Pipeline de Dados

O projeto segue uma arquitetura de dados em 7 camadas, desde a coleta nas fontes públicas até a exibição nos dashboards.

```
┌─────────────────────────────────────────────────┐
│  1. Coleta de dados                             │
│     Scrapers: Portal de Transparência,          │
│     TCE-RJ, IBGE                                │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  2. Armazenamento bruto (Raw)                   │
│     PostgreSQL — dados originais para auditoria │
│     sem nenhuma modificação                     │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  3. Tratamento (ETL)                            │
│     Pandas + NumPy: limpeza, normalização,      │
│     enriquecimento dos dados                    │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  4. Armazenamento estruturado                   │
│     PostgreSQL — dados limpos, validados        │
│     e prontos para análise                      │
└──────────────────────┬──────────────────────────┘
                       │
          ┌────────────┼─────────────┐
          │            │             │
    ┌─────▼──────┐ ┌───▼──────┐ ┌───▼──────────┐
    │  ML        │ │ Geoesp.  │ │ Agente IA    │
    │  XGBoost   │ │ PostGIS  │ │ (RAG)        │
    │  previsão  │ │ + Folium │ │ LangChain    │
    │  atrasos/  │ │ mapas de │ │ + pgvector   │
    │  custos    │ │ obras    │ │ linguagem    │
    └─────┬──────┘ └───┬──────┘ └───┬──────────┘
          │            │             │
┌─────────▼────────────▼─────────────▼───────────┐
│  6. Armazenamento analítico                     │
│     PostgreSQL com Materialized Views           │
│     para alta performance                       │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  7. Exibição (Frontend)                         │
│     React / Streamlit + API REST                │
│     dashboards e mapas interativos              │
└─────────────────────────────────────────────────┘
```

### Módulos da camada de análise (etapa 5)

| Módulo | Stack | Função |
|---|---|---|
| **ML** | XGBoost | Previsão de atrasos e custos de obras |
| **Geoespacial** | PostGIS + Folium | Mapeamento geográfico de obras |
| **Agente IA (RAG)** | LangChain + pgvector | Consultas em linguagem natural sobre os dados |

---

## 🗜️ Estratégia de Compressão

A compressão é aplicada em camadas distintas conforme o tipo de dado e o momento do fluxo. O objetivo é reduzir o volume no banco sem impactar a velocidade de análise.

| Etapa | Estratégia | Tecnologia | Motivo |
|---|---|---|---|
| **1. Coleta** | Sem compressão | — | Dados recebidos em formato nativo das fontes |
| **2. Raw (bruto)** | PAGE Compression | PostgreSQL nativo | Redução de espaço com zero código extra |
| **3. ETL** | Huffman via zlib | Python (`zlib`) | Campos de texto com alta repetição |
| **4. Estruturado** | ROW + PAGE Compression | PostgreSQL nativo | Dados limpos comprimidos em nível de linha e página |
| **5. Análise** | Sem compressão | — | Modelos precisam dos dados descomprimidos em memória |
| **6. Analítico (Views)** | PAGE Compression | PostgreSQL nativo | Materialized Views comprimidas para consultas rápidas |
| **7. Exibição (API)** | Gzip HTTP | Laravel (middleware) | Compressão na transferência entre API REST e frontend |

### Huffman / zlib (ETL)

```python
import zlib

# Comprimindo campo texto antes de persistir
valor_comprimido = zlib.compress(texto.encode("utf-8"))

# Descomprimindo ao ler
texto_original = zlib.decompress(valor_comprimido).decode("utf-8")
```

### PAGE / ROW Compression (PostgreSQL)

```sql
-- PAGE Compression (tabela raw)
ALTER TABLE licitacoes_raw SET (toast_tuple_target = 128);

-- ROW Compression (tabela estruturada)
ALTER TABLE licitacoes SET STORAGE EXTENDED;
```

### Gzip HTTP (Laravel)

```php
// app/Http/Kernel.php
\Illuminate\Http\Middleware\GzipEncoding::class,
```

---

## 🗄️ Banco de Dados — Supabase (PostgreSQL)

O projeto utiliza **Supabase** como plataforma principal de banco de dados, aproveitando PostgreSQL com extensões nativas.

### Localização

```
infra/supabase/
├── migrations/   # Arquivos SQL versionados (ex: 20260101_create_contratos.sql)
└── schemas/      # Definições de tabelas, views e policies RLS
```

### Extensões utilizadas

| Extensão | Versão mínima | Função | Status |
|---|---|---|---|
| `pgvector` | >= 0.5 | Embeddings vetoriais para o agente RAG | ✅ Ativada |
| `PostGIS` | >= 3.0 | Dados e consultas geoespaciais | ✅ Ativada |
| `pg_cron` | latest | Agendamento de jobs direto no banco | ✅ Ativada |

> Todas podem ser habilitadas no painel do Supabase em **Database → Extensions**.

### Validar versões instaladas

Execute no **SQL Editor** do Supabase:

```sql
SELECT name, default_version, installed_version
FROM pg_available_extensions
WHERE name IN ('vector', 'postgis')
ORDER BY name;
```

### Convenções de migration

- Arquivos nomeados com timestamp: `YYYYMMDD_descricao.sql`
- Sempre incluir `-- migrate:up` e `-- migrate:down`
- Executar via Supabase CLI:

```bash
supabase db push
```

### Agendamento com `pg_cron`

```sql
-- Atualização das Materialized Views analíticas (todo dia às 04:00 UTC)
SELECT cron.schedule(
  'refresh-views-analiticas',
  '0 4 * * *',
  $$REFRESH MATERIALIZED VIEW CONCURRENTLY mv_licitacoes_resumo$$
);

-- Limpeza de cache de coleta (todo dia às 03:00 UTC)
SELECT cron.schedule(
  'limpar-cache-coleta',
  '0 3 * * *',
  $$DELETE FROM coleta_cache WHERE criado_em < NOW() - INTERVAL '7 days'$$
);
```

### Referência de expressões cron

| Expressão | Significado |
|---|---|
| `* * * * *` | A cada minuto |
| `*/15 * * * *` | A cada 15 minutos |
| `0 * * * *` | A cada hora |
| `0 4 * * *` | Todo dia às 04:00 UTC |
| `0 0 * * 0` | Todo domingo à meia-noite |

### Row Level Security (RLS)

⏳ **PENDENTE** — Todas as tabelas com dados sensíveis devem ter RLS habilitado. As policies ficam versionadas em `infra/supabase/schemas/`.

```sql
-- Habilitar RLS em uma tabela
ALTER TABLE nome_da_tabela ENABLE ROW LEVEL SECURITY;

-- Exemplo de policy de leitura pública
CREATE POLICY "acesso_publico" ON licitacoes
  FOR SELECT USING (true);
```

---

## 🛠️ Supabase CLI

⏳ **PENDENTE** — Instalar e vincular o Supabase CLI ao projeto.

```bash
npm install -g supabase
supabase login
supabase link --project-ref sxckuxuwfwrhjmwprucr
supabase db push
```

---

## ⚙️ CI/CD — GitHub Actions

Os workflows ficam em `.github/workflows/` e cobrem três domínios: coleta de dados, backend e frontend.

---

### 🕷️ `coleta.yml` — Scrapers agendados

Responsável por disparar os scrapers periodicamente (Portal de Transparência, TCE-RJ, IBGE).

**Gatilhos:**
- `schedule` (cron) — execução automática periódica
- `workflow_dispatch` — execução manual via GitHub UI

**Fluxo:**
```
[cron / manual]
      ↓
Checkout do repositório
      ↓
Setup Python + dependências (coleta/requirements.txt)
      ↓
Execução dos scrapers (coleta/scrapers/)
      ↓
ETL: limpeza, normalização, compressão Huffman/zlib (coleta/etl/)
      ↓
Persistência no Supabase (Raw → Estruturado)
```

**Segredos necessários (`Settings → Secrets`):**

| Secret | Descrição | Status |
|---|---|---|
| `SUPABASE_URL` | URL do projeto Supabase | ⏳ Pendente |
| `SUPABASE_SERVICE_KEY` | Chave de serviço (service role) | ⏳ Pendente |
| `TCE_RJ_TOKEN` | Token de acesso à API do TCE-RJ | ⏳ Pendente |

---

### 🚀 `backend.yml` — Deploy no Laravel Cloud

Responsável pelo deploy contínuo da API REST + agentes de IA em Laravel.

**Gatilhos:**
- Push na branch `main` com mudanças em `backend/**`
- `workflow_dispatch`

**Fluxo:**
```
[push main / manual]
      ↓
Checkout do repositório
      ↓
Setup PHP + Composer
      ↓
Testes automatizados (PHPUnit)
      ↓
Deploy no Laravel Cloud via CLI
      ↓
Health check do endpoint /api/health
```

**Segredos necessários:**

| Secret | Descrição | Status |
|---|---|---|
| `LARAVEL_CLOUD_TOKEN` | Token de autenticação do Laravel Cloud | ⏳ Pendente |
| `SUPABASE_URL` | URL do banco de dados | ⏳ Pendente |
| `SUPABASE_SERVICE_KEY` | Chave de serviço Supabase | ⏳ Pendente |
| `APP_KEY` | Chave da aplicação Laravel | ⏳ Pendente |

---

### 🌐 `frontend.yml` — Deploy do React

Responsável pelo build e deploy do dashboard React.

**Gatilhos:**
- Push na branch `main` com mudanças em `frontend/react/**`
- `workflow_dispatch`

**Fluxo:**
```
[push main / manual]
      ↓
Checkout do repositório
      ↓
Setup Node.js + npm install
      ↓
Build de produção (npm run build)
      ↓
Deploy do artefato (Vercel / Netlify / S3)
```

**Segredos necessários:**

| Secret | Descrição | Status |
|---|---|---|
| `VERCEL_TOKEN` | Token da Vercel | ⏳ Pendente |
| `VITE_SUPABASE_URL` | URL pública do Supabase | ⏳ Pendente |
| `VITE_SUPABASE_ANON_KEY` | Chave anônima do Supabase | ⏳ Pendente |

> **Nota sobre Streamlit:** as visualizações ML em `frontend/streamlit/` rodam de forma independente. Considerar deploy separado via Streamlit Cloud ou Docker.

### Como adicionar os GitHub Secrets

1. Acesse seu repositório no GitHub
2. Vá em **Settings → Secrets and variables → Actions**
3. Clique em **New repository secret**
4. As chaves do Supabase ficam em: **Supabase Dashboard → Settings → API**

---

## 🔐 Gerenciamento de Segredos

Todos os segredos são gerenciados via **GitHub Secrets** (`Settings → Secrets and variables → Actions`).

> ⚠️ **NUNCA** versionar tokens, senhas ou chaves de API no repositório.

| Módulo | Variáveis principais |
|---|---|
| `coleta/` | `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, tokens das fontes |
| `backend/` | `APP_KEY`, `SUPABASE_URL`, `SUPABASE_SERVICE_KEY` |
| `frontend/react/` | `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY` |
| `ml/` | `SUPABASE_URL`, `SUPABASE_SERVICE_KEY` |

---

## 🛠️ Pré-requisitos para desenvolvimento local

| Ferramenta | Versão recomendada |
|---|---|
| Python | 3.11+ |
| PHP | 8.2+ |
| Node.js | 20+ |
| Supabase CLI | latest |
| Git | 2.40+ |
| PostgreSQL | 18 (local) |

---

## 📎 Referências

- [Supabase Docs](https://supabase.com/docs)
- [pg_cron no Supabase](https://supabase.com/docs/guides/database/extensions/pg_cron)
- [PostGIS no Supabase](https://supabase.com/docs/guides/database/extensions/postgis)
- [pgvector no Supabase](https://supabase.com/docs/guides/database/extensions/pgvector)
- [GitHub Actions — Secrets](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions)
- [Laravel Cloud](https://cloud.laravel.com)
- [LangChain Docs](https://python.langchain.com/docs)