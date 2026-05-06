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
- **`marketing`** — acesso às abas Visão Geral e Brindes (antes chamado `brindes`, que dava acesso só a Brindes).
- **`dp`** — acesso às abas Visão Geral, People, Horas, Faltas, VR e Coparticipação.

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

## Bugs conhecidos e corrigidos

### Filtro de Mês/Ano na aba Coparticipação (`getCopaData`)

**Sintoma:** o filtro não mostrava todos os meses — fevereiro e abril de 2026, por exemplo, sumiam do dropdown ou apareciam com o mês errado (fev → jan, abr → mar).

**Causa raiz:** bug de fuso horário no Apps Script. A coluna `Mês/Ano` da planilha `oficial_coparticipação` armazena o 1º dia de cada mês como data (ex: `01/04/2026`). O Apps Script converte células de data em objetos `Date` com **meia-noite UTC**. O servidor do Apps Script roda em fuso negativo (tipo UTC-5), então `getMonth()` interpreta essa meia-noite UTC como o dia anterior — jogando o mês uma posição para trás.

**Tentativas fracassadas:**
- `getMonth()` + `getFullYear()` → errado porque o JVM do servidor tem fuso negativo
- `Utilities.formatDate(..., Session.getScriptTimeZone())` → fuso do script pode divergir da planilha
- `Utilities.formatDate(..., ss.getSpreadsheetTimeZone())` → UTC-3 deslocava meia-noite UTC para 21h do dia anterior, quebrando todos os dados (0 registros)

**Solução correta:** usar `getUTCMonth()` e `getUTCFullYear()`. O Apps Script cria os objetos `Date` com a data da planilha representada como meia-noite **UTC**, então os métodos UTC sempre retornam o mês e ano corretos, independente do fuso do servidor.

```javascript
// Correto — não usar getMonth()/getFullYear() para datas de planilha no Apps Script
mesStr = String(mesAnoRaw.getUTCMonth()+1).padStart(2,'0') + '/' + mesAnoRaw.getUTCFullYear();
```

> **Regra geral:** ao trabalhar com datas vindas de `sheet.getValues()` no Apps Script, sempre usar `getUTCMonth()` / `getUTCFullYear()` para extrair mês e ano. Nunca usar `getMonth()` / `getFullYear()` pois dependem do fuso do servidor, que é imprevisível.

### Formatação de valores na aba Coparticipação

O KPI "Total Gratuidades" e a coluna de valores da tabela de gratuidades devem usar `fmtBRL()` (formato `R$ X.XXX,XX`). Não usar `.toFixed(2)` direto.

---

### Aba DP — menu dropdown na navbar

As abas **Horas, Faltas, VR e Coparticipação** foram removidas da barra de navegação principal e agrupadas em um botão **👤 DP** com menu dropdown flutuante.

**Estrutura HTML (nav):**
- `<div class="dp-nav-wrap" id="dp-nav-wrap">` envolve o botão e o menu
- O botão `#dp-nav-btn` chama `toggleDpDropdown(event)`
- O menu `#dp-nav-menu` contém 4 itens `.dp-nav-item` que chamam `selectDpSub('id', this)`

**CSS relevante:** `.dp-nav-wrap`, `.dp-nav-menu`, `.dp-nav-item`, `.dp-chevron`, `.dp-nav-btn-open`

**Funções JS:**
- `toggleDpDropdown(e)` — abre/fecha o menu, rotaciona a seta
- `selectDpSub(id, menuItem)` — fecha o menu, mostra o tab-content correspondente, marca o botão DP e o item como ativos
- `document.addEventListener('click', ...)` — fecha o menu ao clicar fora

**`applyRoleAccess`:** o wrapper `#dp-nav-wrap` é mostrado/ocultado com base em se qualquer uma das 4 sub-abas está no `ROLE_TABS` do usuário.

**ROLE_TABS relevantes:**
- `dp` → `['geral','people','horas','faltas','vr','copa']`
- `diretor` → `['geral','people','horas','faltas','vr']` (sem copa)
- `admin`/`viewer` → acesso completo incluindo as 4

---

### Engajamento — tooltip em respostas cortadas

`.eng-bar-option` tem `text-overflow:ellipsis`. Para exibir o texto completo ao hover, foi adicionado um tooltip via CSS (`::after` com `content:attr(title)`), mais confiável que `title` nativo em iframes do Apps Script. Ambas as funções de renderização (`engBarBlock` e `engBarFull`) já geram `title="..."` em cada opção.

### Engajamento — layout "Expectativa profissional"

"O que mais te motiva a permanecer na empresa?" foi movido para ficar ao lado de "Você está na área que sempre quis?" em um grid de 2 colunas. O `id="eng-grid-motiva"` permanece o mesmo — só a posição no HTML mudou.

### Professores — paginação da lista (40/página)

Variáveis globais: `PROF_PAGE`, `PROF_PAGE_SIZE=40`, `PROF_ROWS`. A função `renderProfessores()` preenche `PROF_ROWS` e chama `renderProfPage()`. Controles de navegação: `pagProf(dir)` e elementos `#prof-pag-prev`, `#prof-pag-next`, `#prof-pag-info`.

### Professores — gráficos de Turmas por Professor (dois gráficos)

O gráfico único foi substituído por dois gráficos de barras **verticais**.

**Campos retornados pelo backend (`getTurmasData`):**
- `chaveMatricula` — coluna Z (`APELIDO | MATRÍCULA`): chave única para agrupar professores com mesmo apelido
- `chave` — coluna Y (`UNIDADE | APELIDO`): label com unidade visível, usado no gráfico de baixo
- `nivel` — coluna AC (`NÍVEL`): nível do professor, exibido como segunda linha do label no gráfico de baixo
- Filtro de teste: turmas cujo nome (coluna B) bate com `/testes?\s+inc/i` são ignoradas

**Chart 1 — `chartTurmasTop35` (Top 35 com mais turmas):**
- Barras verticais azul-navy (`#2a4d76`), labels = apelido, rotação 45–90°
- Linha de referência tracejada laranja (`#e05c2a`) mostrando a **média BRASAS Geral** (total turmas ÷ total professores únicos, sobre todos os dados antes do filtro de unidade)
- Gráfico misto (`type:'bar'` + `type:'line'` no mesmo dataset) criado diretamente com `new Chart(...)` (não usa `mkChart`)
- Canvas `#chartTurmasTop35`, wrapper `#wrapTurmasTop35`, altura fixa `440px`

**Chart 2 — `chartTurmasBot20` (Top 20 com menos turmas):**
- Barras verticais âmbar (`#c9a227`), rotação 45–90°
- Label em 2 linhas: `['UNIDADE | APELIDO', '(NÍVEL)']` — Chart.js renderiza arrays como multi-linha
- Canvas `#chartTurmasBot20`, wrapper `#wrapTurmasBot20`, altura fixa `460px`
- Criado diretamente com `new Chart(...)` para controle de rotação dos ticks

---

## Fluxo de trabalho no Git

Esse projeto não tem CI nem deploy automático. As alterações são:
1. Feitas e testadas diretamente no editor do Apps Script.
2. Copiadas para os arquivos `.txt` locais.
3. Commitadas e enviadas ao GitHub **manualmente** (`git push`) quando solicitado.

**Sempre fazer `git pull` antes de começar a editar** para garantir que a cópia local está atualizada.
