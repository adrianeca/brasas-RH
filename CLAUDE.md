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
- **`marketing`** — acesso às abas Visão Geral e Brindes.
- **`dp`** — acesso às abas Visão Geral, People, Desligamentos, Horas, Faltas, VR e Coparticipação.

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
| `getDesligamentosData(token)` | Carrega respostas da entrevista de desligamento |

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
| `desligamentos` | Desligamentos | Análise das entrevistas de desligamento |
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

### Professores — gráfico combinado de Turmas (`chartTurmasComb`)

Os dois gráficos separados (Top35 + Bottom20) foram substituídos por **um único gráfico combinado**: Top 15 com mais turmas (azul) + Top 15 com menos turmas (âmbar), com linha de média BRASAS Geral tracejada.

**Campos retornados pelo backend (`getTurmasData`):**
- `chaveMatricula` — (`APELIDO | MATRÍCULA`): chave única para agrupar professores com mesmo apelido
- `chave` — (`UNIDADE | APELIDO`): label com unidade
- `nivel` — (`NÍVEL`): nível do professor
- `unidade` — (`UNIDADE AJUSTADA`): unidade do professor
- `alVigente`, `alMaximo` — alunos vigentes e máximos da turma
- Filtro de teste: turmas cujo nome (col B) bate com `/testes?\s+inc/i` são ignoradas
- Filtro de ativas: `TERMINO` ausente ou >= hoje

**`renderTurmasChart()`:**
- Reage ao filtro `f-prof-unidade` — barras mostram professores da unidade selecionada
- **Média BRASAS sempre calculada sobre ALL_TURMAS inteiro** (ignora filtro de unidade) via `globalCounts`
- Top 15 (desc) → dataset azul `#2a4d76`; Bottom 15 (asc, sem overlap) → dataset âmbar `#c9a227`
- Linha tracejada laranja `#e05c2a` para a média
- Labels em 2 linhas: `[apelido, 'UNIDADE · NÍVEL']` via array (Chart.js multi-linha)
- Canvas `#chartTurmasComb`, wrapper `#wrapTurmasComb`, altura `440px`

**Unidades excluídas** de filtro e tabela de Professores por Unidade/Nível:
- `EXCL_PROF_UNITS = ['EDITORA', 'MÉTODOS', 'EC NEW']`

**Tabela Professores por Unidade e Nível (`renderProfNivelTable`):**
- Header com `background:var(--navy-700); color:#fff`
- Linhas ímpares com `background:var(--gray-50)` (zebra striping)
- **Sempre filtra `ativo=true` + funcao inclui PROFESSOR** internamente, ignorando filtro de status da UI
- Três linhas de rodapé:
  - **Total** (navy-600): soma bruta de aparições (professor em 2 unidades conta 2×)
  - **Em +1 unidade** (cinza): professores com 2 unidades operacionais — EDITORA/MÉTODOS/EC NEW NÃO contam como segunda unidade válida
  - **Total único** (âmbar): `rows.length − totalDups + profsNotInRows` = headcount real
- `profsNotInRows` = professores cujas AMBAS as unidades (primária e secundária) são excluídas — entram no KPI mas não em nenhuma linha da tabela; são somados de volta ao Total único para fechar a conta com o KPI
- Função `isExcl(u)`: `u.normalize('NFD').replace(combining-marks,'').toUpperCase()` → compara com `'EDITORA'`, `'METODOS'`, `'EC NEW'` — cobre acentos NFC, NFD e maiúsculas/minúsculas
- `nivel` lido de `getData()` com fallback: `'NÍVEL PROFESSORES'` → `'NÍVEL'` → `'NIVEL'` (col AH)

### Brindes — acesso do role `diretor`
- `diretor` agora tem acesso à aba Brindes (`ROLE_TABS['diretor']` inclui `'brindes'`)
- `getBrindesConsolidado` filtra pelo campo `unidade` do session quando `role === 'diretor'`
- **Brindes é o último item da navbar**, depois do dropdown DP

---

### Professores — Histórico de Turmas e Alunos (`getHistoricoTurmas`)

Seção abaixo do gráfico combinado com evolução mensal de turmas e alunos por professor.

**Backend — `getHistoricoTurmas(token)`:**
- Lê a aba `db` da mesma planilha `TURMAS_SPREADSHEET_ID`
- Coluna V = data de atualização (`DATA DE ATUALIZAÇÃO` / `DATA ATUALIZACAO`)
- Coluna F = `AL_VIGENTE` (alunos)
- Pega apenas a **última data de cada mês** (último snapshot — não é cumulativo)
- Exclui testes incompletos: `/testes?\s+inc/i` no nome da turma (col B)
- Datas parseadas com `getUTCMonth()`/`getUTCDate()` (mesmo padrão do bug de fuso corrigido)
- 12 meses para trás a partir de hoje
- Role `diretor` vê só a própria unidade
- Retorna: `{ ok: true, meses: [{mes: 'MM/YYYY', rows: [{apelido, chaveMatricula, unidade, alunos}]}] }`

**Frontend — filtros e gráfico:**
- Global `ALL_HISTORICO = []` (preenchido por `loadHistoricoData()`, chamado após `loadTurmasData()`)
- Filtro cascata: Unidade → Professor
- **Professores ativos** vêm de `ALL_EMP` (funcao inclui 'PROFESSOR', ativo = true), usando colunas `unidadeAcesso` (col AJ) e `unidadeSecAcesso` (col AK) da planilha principal — professores podem ter duas unidades simultaneamente
- Mesmas unidades excluídas (`EXCL_PROF_UNITS`)
- **Loading indicator** `#hist-loading` aparece durante o carregamento (a requisição é lenta)
- Gráfico dual-axis: turmas no eixo esquerdo (azul/azul-claro), alunos no eixo direito (teal)
- 4 linhas: turmas do professor, alunos do professor, média BRASAS turmas, média BRASAS alunos
- Médias calculadas sobre TODOS os professores de TODOS os meses (sem filtro de unidade)
- 3 KPI cards: Turmas (mês atual) + delta, Alunos (mês atual) + delta, vs Média BRASAS
- KPIs renderizados com `innerHTML` diretamente (não usar `set()` — essa função usa `textContent` e escapa HTML)

**Campos `getData()` adicionados:**
- `unidadeAcesso` — col AJ (`UNIDADE ACESSO`): unidade primária do professor
- `unidadeSecAcesso` — col AK (`UNIDADE SEC ACESSO`): unidade secundária

---

### Professores — Distribuição por Função (cap em 15)

`renderFuncPyramid()` agrupa funções excedentes: as 15 primeiras aparecem individualmente; as demais viram **"Outros (N funções)"** em cinza `#94a3b8`.

---

---

### Aba Desligamentos (`getDesligamentosData`)

Acesso restrito a `admin` e `dp`. Lê a planilha de entrevistas de desligamento.

**Constantes no backend:**
- `DESLIG_SPREADSHEET_ID = '1ZDvSlEsZOHUdPO7y2YexSoA4TqO_nltCF9TlqpMFRN8'`
- Aba localizada pelo GID `121305848` (mais confiável que nome) com fallback para `'Respostas Estrevista de Desligamento'`

**Campo `motivoSaida` em `getData()`:**
- Lê a coluna `INATIVOS` (coluna AG) da aba `RJ - UNIDADES`
- Usado no frontend para cruzar pedidos de demissão com entrevistas realizadas
- `getData()` usa match exato de coluna (não `normalizeH_`) — o cabeçalho deve ser exatamente `INATIVOS`

**Frontend — estrutura da aba:**
- Filtros: Ano (multi-seleção via checkboxes), Unidade, Setor
- Filtro de ano limitado a 2023+; piso de 2023 aplicado em `getDlFiltered()`
- KPIs: Total Desligamentos, Taxa de Cobertura, Foram p/ Outra Empresa, Principal Motivo, Imagem Positiva BRASAS
- **Taxa de Cobertura** = entrevistas realizadas ÷ pedidos de demissão no período (de `ALL_EMP`)

**Gráfico "Entrevistas de Desligamento por Período"** (full-width, linha 1):
- Duas barras agrupadas: "Pedidos de Demissão" (navy `#2a4d76`) vs "Entrevistas Realizadas" (âmbar `#f59e0b`)
- Toggle Mensal/Trimestral/Anual (`DL_PERIODO_VIEW`)
- Pedidos vêm de `ALL_EMP` filtrado por `motivoSaida.includes('pedido de demiss')` + `!ativo` + ano ≥ 2023
- Entrevistas vêm de `ALL_DESLIG` (já filtrado por `getDlFiltered()`)

**Layout dos gráficos:**
- Linha 1 (full-width): Período/crossover
- Linha 2: Motivos de Desligamento + Perfil por Cargo
- Linha 3: Tempo de Empresa + Mobilidade Salarial
- Linha 4: Percepção Salarial

**Avaliações de Saída (`renderDlAvaliacoes`):**
- 4 grupos: Benefícios, Comunicação Interna, Relacionamento, Empresa
- Cards de nota média (1–4) — card com menor nota recebe borda âmbar e badge `⚠ ponto crítico`
- Constante `DL_AVAL_GROUPS` define os grupos, `DL_AVAL_COLORS` define as cores por rating

**Análise dos Comentários (`renderDlInsights`):**
- Análise de frequência de temas nos comentários livres
- Constante `DL_COMMENT_THEMES` define 7 temas com palavras-chave (normaliza acentos para match)
- Temas: Liderança/Gestão, Salário/Benefícios, Clima/Equipe, Crescimento/Carreira, Carga/Pressão, Proposta Externa, Comunicação

**Filtro de ano multi-seleção:**
- Global `DL_ANOS_SEL = []` (vazio = todos os anos)
- Dropdown customizado com checkboxes (`.multisel-btn`, `.multisel-menu`, `.multisel-item`)
- Funções: `toggleAnoMenu(e)`, `toggleDlAno(ano, cb)`
- Fecha ao clicar fora — tratado no `document.addEventListener('click', ...)` existente

---

### Visão Geral — Detalhamento por Unidade

- **Unidades MD e CT excluídas** da tabela (`EXCL_UNIT_TABLE = ['MD','CT']` em `renderUnitTable`)
- **Coluna "Distribuição"** tem tooltip explicativo via CSS `::after` com `content:attr(data-tip)` (classe `.th-tip`). Padrão igual ao tooltip da aba Engajamento — mais confiável que `title` nativo em iframes.

---

### Visão Geral — Lista de Colaboradores

**Colunas presentes (em ordem):** Nome · Apelido · Unidade · Função · Sexo · Status · Admissão · Dt. Nasc. · Anos Casa · Dt. Deslig. · E-mail BRASAS

- **Coluna Apelido** (`apelido` de `getData()`) adicionada após Nome, com suporte a ordenação.
- **Coluna Dt. Nasc.** (`dataNasc`, formato `DD/MM/YYYY`) adicionada após Admissão, com suporte a ordenação (usa `toISO` como `dataAdm` e `dataDeslig`).
- **Filtro "Mês de Nasc."** (`#f-nasc-mes`) — dropdown com os 12 meses. Filtra pelo campo `dataNasc`: extrai o mês via `dataNasc.split('/')[1]` e compara com o valor `MM` selecionado. Colaboradores sem `dataNasc` são excluídos quando o filtro está ativo.
- **Status padrão = Ativos** — o select `#f-status` tem `selected` na opção `Ativo`, então a lista já abre filtrada por ativos. O botão "Limpar" também restaura para `Ativo` (não para "Todos").
- Todas as colunas novas estão incluídas no CSV exportado por `downloadCSV()`.

---

---

### Professores — Dropouts & Observações

Nova seção na aba Professores com análise cruzada entre dropouts de alunos e notas de observação de coordenadores.

**Planilhas:**
- `DROPOUTS_SPREADSHEET_ID = '1d8votmt6HlwPmHcup2iQLqJa9dzpcHorPGGC3LO27hw'` → aba `Total Unidades`
- `NOTAS_SPREADSHEET_ID = '1h6-an2pv-ei-ZaXXYaA1y5UCUq2Q7Za8OCrZ4pQLuNQ'` → aba `notas forms`

**Join key:** `APELIDO | MATRÍCULA` — campo `chaveMatricula` em ambas as fontes
- Dropouts: coluna `CHAVE NOME | MATRÍCULA` (normalizeH_ → `chave nome | matricula`)
- Notas: coluna `Chave Matrícula` (normalizeH_ → `chave matricula`)

**Backend:**
- `getDropoutsData(token)` — retorna linhas por turma/mês: `{unidade, teacher, nomeTeacher, book, alunos, dropouts, mes, ano, mesAno, chaveMatricula, ativo}`
- `getNotasData(token)` — retorna linhas por observação: `{teacher, coordenador, unidade, observationDate, mes, ano, mesAno, nivel, nota, chaveMatricula}`
- `role=diretor` filtra por unidade da sessão em ambas

**Frontend — carregamento:** `loadDropoutsData()` e `loadNotasData()` chamados com `setTimeout 300ms` após o load inicial de professores

**Globais:** `ALL_DROPOUTS`, `ALL_NOTAS`, `DO_CHARTS` (instâncias Chart.js)

**Filtros da seção:** `#do-unidade` (unidade) e `#do-ano` (ano)

**4 gráficos globais:**
- `chartDropoutsMes` — linha temporal de dropouts totais; estrelas laranjas marcam meses com observação
- `chartTopDropouts` — barras horizontais top 20 por dropouts; âmbar = professor com observação, navy = sem
- `chartNotasDist` — distribuição de notas em 5 faixas (< 6, 6–7, 7–8, 8–9, 9–10)
- `chartImpacto` — barras agrupadas Antes vs Depois da última observação (janela de 3 meses); tooltip mostra nota e Δ; só aparece se há dados suficientes (≥ 1 mês antes E ≥ 1 depois)

**Timeline individual (`#do-prof-timeline`):**
- Seletor populado com todos os professores de `ALL_DROPOUTS`
- Gráfico de linha: dropouts por mês; pontos laranja estrelados = mês com observação
- Tooltip mostra nota e coordenador nos pontos de observação
- KPIs: total dropouts, nº observações, nota média, tendência (↓ queda / ↑ alta / → estável)

---

## Fluxo de trabalho no Git

Esse projeto não tem CI nem deploy automático. As alterações são:
1. Feitas e testadas diretamente no editor do Apps Script.
2. Copiadas para os arquivos `.txt` locais.
3. Commitadas e enviadas ao GitHub **manualmente** (`git push`) quando solicitado.

**Sempre fazer `git pull` antes de começar a editar** para garantir que a cópia local está atualizada.
