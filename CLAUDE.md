# BRASAS Analytics — Dashboard RH

## O que é esse projeto

Dashboard interno de RH da rede BRASAS RJ, construído em **Google Apps Script**. Roda 100% dentro do Google Workspace — não há servidor externo, banco de dados local nem deploy. O backend lê dados de planilhas Google e serve o frontend via `HtmlService`.

## Arquivos

| Arquivo | Papel |
|---|---|
| `codigo rh.txt` | Backend — `Code.gs` no Apps Script |
| `index rh.txt` | Frontend — `Index.html` no Apps Script |

> No Apps Script esses arquivos se chamam `Code.gs` e `Index.html`. Os `.txt` são cópias locais para versionamento no Git.

## Backend (`codigo rh.txt`)

### Planilhas conectadas

| Constante | ID da planilha | Aba principal |
|---|---|---|
| `SPREADSHEET_ID` | `1BDiPjv0FqRJp5EwcvLdYXVvEAWesvwdEgbhYdnTlqPY` | `RJ - UNIDADES` |
| `ENGAGEMENT_SHEET_ID` | mesmo ID acima | `Questionário` |
| `HORAS_SPREADSHEET_ID` | `1VNeT_tN6dJItg-bZti1zqdQbmz-0silTVH--nJhHZQg` | `Horas_2024+` |
| `VR_SPREADSHEET_ID` | `1PgcBeGF9GKmt_jwNBaXijzF8T5jhBmGV2LssXGrsoRg` | `db_administrativo`, `db_docente` |
| `COPA_SPREADSHEET_ID` | `1MGlfK7DXbnLpl55zc7AnzJvxDnsEA1umCeAbWRL5HhA` | `oficial_coparticipação` |
| `TURMAS_SPREADSHEET_ID` | `1O5mYkfiFKpSd0aFxWVWjXJFxlbw6jYnDZX0SbiDda5M` | `db_max` |

### Autenticação

- Usuários ficam na aba `USUARIOS` da planilha principal (colunas: email, senha, nome, role, unidade).
- Login gera um UUID que fica no `CacheService` por `SESSION_TTL = 8h`.
- Toda função de dados exige token válido — sem sessão retorna `{ ok: false, error: '...' }`.

### Roles

- **`admin`** — vê todos os dados de todas as unidades.
- **`diretor`** — vê apenas os dados da própria unidade (filtro aplicado no backend).

### Funções principais

| Função | O que faz |
|---|---|
| `doGet()` | Ponto de entrada — serve o `Index.html` |
| `login(email, password)` | Autentica e cria sessão |
| `logout(token)` | Remove sessão do cache |
| `checkSession(token)` | Valida token e retorna dados da sessão |
| `getData(token)` | Carrega colaboradores da aba `RJ - UNIDADES` |
| `getEngagementData(token)` | Carrega respostas do questionário de engajamento |
| `getBrindesConsolidado(token)` | Carrega brindes da aba `CONSOLIDADO_new` |
| `getHorasData(token)` | Carrega horas de professores |
| `getVRData(token)` | Carrega dados de VR (administrativo e docente) |
| `getCopaData(token)` | Carrega coparticipação do plano de saúde |
| `getTurmasData(token)` | Carrega turmas ativas por professor |

### Tabela de brindes por anos de casa

```
5 anos → CANECA
10 → QUADRO MENSAGENS
15 → PLANNER
20 → MOCHILA
25 → KINDLE
30 → CANETA METÁLICA + CERTIFICADO
35 → MALA DE RODINHA
40 → PLACA HOMENAGEM + CERTIFICADO
45 → RELÓGIO DE LUXO
50 → INDEFINIDO
```

Anos calculados: 2025, 2026, 2027 (constante `GIFT_YEARS`).

### Helpers

- `normalizeH_(h)` — normaliza cabeçalhos de planilha (minúsculo, sem acento, sem espaços duplos). Usado para tolerar variações nos nomes de coluna.
- `parseDate_(val)` — converte string ou Date para Date (suporta `DD/MM/YYYY` e `YYYY-MM-DD`).
- `fmtDate_(d)` — formata Date para `DD/MM/YYYY`.
- `diffYears_(from, to)` — calcula diferença em anos completos.

## Frontend (`index rh.txt`)

### Stack

- HTML + CSS + JavaScript puros (sem framework).
- **Chart.js 4.4.0** (CDN) para gráficos.
- Fontes: DM Sans + JetBrains Mono (Google Fonts).

### Abas do dashboard

| Tab ID | Nome | O que mostra |
|---|---|---|
| `geral` | Visão Geral | KPIs, lista de colaboradores, distribuição por função/unidade |
| `people` | People Analytics | Gráficos de gênero, idade, tempo de casa, plano de saúde, admissões por ano, rotatividade |
| `brindes` | Brindes | Lista e resumo de brindes por ano/semestre/unidade |
| `professores` | Professores | KPIs, distribuição por nível/unidade, turmas ativas |
| `horas` | Horas | Horas de aula dos professores |
| `faltas` | Faltas | Faltas descontadas/abonadas |
| `vr` | VR | Vale-refeição — administrativo e docente |
| `cohort` | Cohort | Análise de cohort de colaboradores |
| `engajamento` | Engajamento | Respostas do questionário de engajamento |
| `copa` | Coparticipação | Coparticipação do plano de saúde |

### Sistema de design (variáveis CSS)

Paleta navy escuro como base, com tokens semânticos:
- `--navy-*` (900→25) — cor principal
- `--success-*`, `--danger-*`, `--warning-*`, `--accent-*`, `--teal-*` — estados
- `--font-main` (DM Sans) e `--font-mono` (JetBrains Mono)
- `--shadow-sm/md/lg`, `--radius/radius-sm`

## Fluxo de trabalho no Git

Esse projeto não tem CI nem deploy automático. As alterações são:
1. Feitas e testadas diretamente no editor do Apps Script.
2. Copiadas para os arquivos `.txt` locais.
3. Commitadas e enviadas ao GitHub **manualmente** (`git push`) quando solicitado.

**Sempre fazer `git pull` antes de começar a editar** para garantir que a cópia local está atualizada.
