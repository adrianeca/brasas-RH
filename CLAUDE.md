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

### Autenticação (Google SSO via Hub)

- Login via conta Google Workspace (`brasas.com`) — sem formulário de email/senha.
- `doGet()` chama `getGoogleSession_()` que usa `Session.getActiveUser().getEmail()` para identificar o usuário.
- O email é consultado na aba `USUARIOS` da planilha principal (colunas: email, ~~senha~~, nome, role, unidade — a coluna senha não é mais usada).
- Se a sessão for criada no servidor, o token é injetado no HTML via template scriptlet `<?!= initialSession ?>`.
- Se `Session.getActiveUser()` retornar vazio (configuração de deploy), o frontend chama `loginWithGoogle()` como fallback via `google.script.run`.
- Sessão cria UUID no `CacheService` com `SESSION_TTL = 8h`. Todas as funções de dados exigem token válido.
- **Configuração de deploy necessária:** o webapp deve estar publicado com acesso restrito a usuários do domínio `brasas.com` (não anônimo), para que `Session.getActiveUser()` retorne o email correto.

### Roles

- **`admin`** — vê todos os dados de todas as unidades.
- **`diretor`** — vê Visão Geral, People, Professores, Brindes, Horas, Faltas e VR — filtrado pela própria unidade no backend.
- **`marketing`** — acesso às abas Visão Geral e Brindes.
- **`dp`** — acesso às abas Visão Geral, People, Desligamentos, Horas, Faltas, VR e Coparticipação.

### Funções principais

| Função | O que faz |
|---|---|
| `doGet()` | Ponto de entrada — detecta email Google, cria sessão e injeta token no template |
| `getGoogleSession_()` | Helper — obtém email via `Session.getActiveUser()`, consulta USUARIOS, cria token no cache |
| `loginWithGoogle()` | Fallback chamado pelo frontend se `doGet()` não conseguiu criar sessão |
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
| `getDropoutsData(token)` | Carrega linhas de dropout por turma/mês da planilha CONSOLIDADO GERAL |
| `getNotasData(token)` | Carrega notas de observação de coordenadores da planilha NOVO Teste class observation |
| `getNotasMediasData(token)` | Carrega médias de notas da aba `Notas Médias` — retorna `photoUrl: null` (fotos buscadas separadamente via `getDropoutPodiumFotos`) |
| `getDropoutPodiumFotos(token, entries)` | Busca fotos base64 no Drive para uma lista de até 3 professores — usado por AMBOS os pódios (notas e dropout) |
| `diagFotos(token)` | Diagnóstico: lista arquivos da pasta de fotos e testa match com linhas da planilha |

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

### Navbar

- Logo **BRASAS Analytics** à esquerda (sem emoji).
- Botão **🏠 Hub** logo após o logo — link para o Hub central (`target="_top"` para sair do iframe). Estilo `.hub-btn`.
- Abas de navegação na sequência.
- Lado direito (`.cockpit-nav-right`): nome do usuário + botão **Sair**.
- `.cockpit-nav-inner` usa `width:100%` (sem `max-width`) para aproveitar a largura total da tela.

### Sistema de design (variáveis CSS)

Paleta navy escuro como base, com tokens semânticos:
- `--navy-*` (900→25) — cor principal
- `--success-*`, `--danger-*`, `--warning-*`, `--accent-*`, `--teal-*` — estados
- `--font-main` (DM Sans) e `--font-mono` (JetBrains Mono)
- `--shadow-sm/md/lg`, `--radius/radius-sm`

## Bugs conhecidos e corrigidos

### Acesso à aba Desligamentos para roles customizados (ex: `g&g`)

**Sintoma:** usuários com roles definidos via aba `ROLES` (ex: `g&g`) que tinham `rh_desligamentos = true` conseguiam ver a aba no frontend, mas ao carregar os dados recebiam erro "Sem permissão".

**Causa raiz:** `getDesligamentosData` tinha um check hardcoded `if (role !== 'admin' && role !== 'dp')` que ignorava completamente o `allowedPages` armazenado na sessão (gerado pela aba `ROLES`).

**Solução:** substituir o check hardcoded por verificação dinâmica que prioriza `session.allowedPages`:

```javascript
const allowed = session.allowedPages || null;
const canAccess = allowed
  ? allowed.includes('desligamentos')
  : (role === 'admin' || role === 'dp');
if (!canAccess) return { ok: false, error: 'Sem permissão.' };
```

> **Regra geral:** nunca usar check hardcoded de role em funções de dados. Sempre verificar `session.allowedPages` primeiro — é o que reflete a configuração real da aba `ROLES`. O fallback por role só deve existir para compatibilidade com sessões antigas que não carregaram `allowedPages`.

---

### Cálculo de Turnover na aba People Analytics (`computePeriod`)

**Sintoma:** o Turnover exibido na tabela de Rotatividade era aproximadamente o dobro do valor esperado pelo mercado.

**Causa raiz:** a fórmula usada era `(Entradas + Saídas) / Efetivo Médio`, que equivale à soma das taxas admissional e demissional. A fórmula padrão ABRH divide por 2 antes de dividir pelo efetivo médio.

**Solução:** dividir a rotatividade bruta por 2 antes de dividir pelo efetivo médio.

```javascript
// Antes (errado):
var turnover = rot / media;

// Depois (fórmula ABRH):
var turnover = (rot / 2) / media;
// onde rot = contrat + demiss, media = (saldoInicio + saldoFim) / 2
```

**Tooltip adicionado:** ao passar o mouse sobre "Turnover | Rotatividade ⓘ" na tabela, aparece a explicação da fórmula. Implementado com `<span class="th-tip" data-tip="...">` (mesmo padrão CSS dos outros tooltips do projeto). Aplicado em ambas as views (anual e mensal).

---

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

### Brindes — filtros e lista

**Filtros disponíveis na barra:**

| ID | Label | Lógica |
|---|---|---|
| `f-brinde-tipo-un` | Tipo de Unidade | `isFranquia(e.tipo)` — valores: `"franquia"` / `"propria"` |
| `f-brinde-anos` | Anos de Casa | `g.anos === parseInt(fAnos)` — opções: 5, 10, 15 … 50 |
| `f-brinde-setor` | Setor | `"pedagogico"` = funcao inclui PROFESSOR, COORDENADOR ou é GERENTE DE UNIDADE; `"administrativo"` = demais |

O campo `e.tipo` vem da coluna `TIPO UNIDADE` da aba `CONSOLIDADO_new`. A função `isFranquia(tipo)` faz o match case-insensitive. Todos os filtros são resetados em `clearBrindesFilters()`.

**Lista de Brindes — colunas:** Nome · Unidade · **Função** · Tipo · Data Admissão · Anos de Casa · Semestre · Brinde

A coluna `Função` exibe o campo `funcao` retornado por `getBrindesConsolidado` (coluna `FUNÇÃO` da aba `CONSOLIDADO_new`). Também incluída no CSV exportado por `downloadBrindesCSV()`.

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

Acesso restrito por permissão — verifica `session.allowedPages` (aba `ROLES`) antes de cair no fallback hardcoded (`admin` ou `dp`). Lê a planilha de entrevistas de desligamento.

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
- **Status padrão = Todos** — o select `#f-status` abre sem filtro de status. O botão "Limpar" também restaura para `""` (Todos).
- Todas as colunas novas estão incluídas no CSV exportado por `downloadCSV()`.

---

---

### Professores — Dropouts & Observações

Seção na aba Professores com análise cruzada entre dropouts de alunos e notas de observação de coordenadores.

**Planilhas:**
- `DROPOUTS_SPREADSHEET_ID = '1d8votmt6HlwPmHcup2iQLqJa9dzpcHorPGGC3LO27hw'` → planilha "CONSOLIDADO GERAL", aba `Total Unidades`
- `NOTAS_SPREADSHEET_ID = '1h6-an2pv-ei-ZaXXYaA1y5UCUq2Q7Za8OCrZ4pQLuNQ'` → planilha "NOVO Teste class observation", aba `notas forms`

**Join key — número de matrícula puro (após extrair o `|`):**

| Lado | Coluna na planilha | Valor bruto de exemplo | `chaveMatricula` final |
|---|---|---|---|
| Dropouts | Col M = `MATRÍCULA BI` | `"AISHA \| 793"` | `"793"` |
| Notas | Col CC (índice 80) = `Chave Matrícula` | `"ZOEY \| 525"` | `"525"` |

Ambos fazem `split('|')` e pegam a parte após o `|`. Isso evita qualquer divergência de APELIDO entre as duas planilhas.

**Leitura de colunas em `getNotasData`:**
- Nota: coluna CA = índice 78
- Chave: coluna CC = índice 80 (por índice fixo, não por cabeçalho)
- **Nível: coluna CD = índice 81** (por índice fixo, como primária; fallback cabeçalhos `Level` / `Nivel`) — resultado recebe `.replace(/,/g,'.')` para normalizar "1,1" → "1.1" (mesma lógica de `getData()`)
- Data: tenta cabeçalhos `Observation Date` → `Data da Observacao` → `Data` → fallback `row[0]` (Timestamp do Google Forms)

**Detecção da coluna de matrícula em `getDropoutsData`:**
Tenta os cabeçalhos na ordem: `MATRÍCULA BI` → `MATRICULA BI` → `MATRÍCULA` → `MATRICULA` → fallback índice 12 (col M). Usa `normalizeH_()` para tolerância de acento/case.

**Ambas as funções retornam `debug`** com amostra de `chaveMatricula` para diagnóstico no console do browser.

**Backend:**
- `getDropoutsData(token)` → `{unidade, teacher, nomeTeacher, book, alunos, dropouts, mes, ano, mesAno, chaveMatricula, ativo, debug}`
- `getNotasData(token)` → `{teacher, coordenador, unidade, observationDate, mes, ano, mesAno, nivel, nota, chaveMatricula, debug}` — `nivel` lido de col CD (índice 81)
- `getData(token)` inclui campo `chaveMatriculaDP` = número extraído da col AM (índice 38, formato pipe igual ao dropouts) — usado para join com `ALL_DROPOUTS` no gráfico Dropouts por Nível
- `role=diretor` filtra por unidade da sessão em ambas

**Frontend — carregamento:**
- `loadDropoutsData()` e `loadNotasData()` chamados com `setTimeout 300ms` após o load inicial de professores
- Ambas logam no console (`[DROPOUTS]` e `[NOTAS]`) para diagnóstico — verificar F12 se algo não aparecer

**Globais:** `ALL_DROPOUTS`, `ALL_NOTAS`, `DO_CHARTS` (instâncias Chart.js)

**Filtros globais da seção (afetam os 2 gráficos globais):**
- `#do-unidade` — unidade; ao mudar, chama `onDOUnidadeChange()` que recascateia o dropdown de professor
- `#do-ano` — ano
- `#do-prof` — professor; populado dinamicamente conforme unidade selecionada

**3 gráficos globais** (a linha temporal global foi removida por ser redundante com a timeline individual):
- `chartTopDropouts` — barras horizontais top 20 professores por dropouts; âmbar = tem observação registrada, navy = sem
- `chartNotasDist` — distribuição de notas em 5 faixas: < 6, 6–7, 7–8, 8–9, 9–10
- `chartDONivel` — barras verticais full-width: total de dropouts por nível do professor

**Gráfico Dropouts por Nível (`chartDONivel`):**
- Join: `ALL_DROPOUTS[i].chaveMatricula` ↔ `ALL_EMP[j].chaveMatriculaDP` (col AM da planilha principal)
- Nível vem de `ALL_EMP[j].nivel` (col AH)
- Ordenação por `NIVEL_ORDER` (A, B, 1.1 … 3.4); "Não informado" cai no final (índice 99)
- Professores sem correspondência listados em `#do-nivel-unmatched` abaixo do título e logados em `[DO-NIVEL]` no console

**Timeline individual por professor:**
- Filtros próprios: `#do-tl-unidade` (unidade) → `onDOTlUnidadeChange()` → recascateia `#do-prof-timeline`
- Gráfico dual-axis: eixo esquerdo = dropouts (azul), eixo direito = nota 0–10 (laranja)
- Pontos estrela laranja no dataset de dropouts marcam meses com observação; segundo dataset (`showLine:false`) plota a nota no eixo direito
- Tooltip mostra nota e coordenador nos meses com observação

**KPIs da timeline (abaixo do gráfico):**
- **Total Dropouts** — soma histórica completa
- **Observações no Ciclo** — conta só as observações do ciclo atual (nov–out); subtítulo mostra o ciclo ex: "Ciclo nov/25–out/26"
- **Nota Média** — média de todas as notas do professor (histórico completo)

**Lógica do ciclo (nov–out):**
```javascript
var cycleY = nowD.getMonth() >= 10 ? nowD.getFullYear() : nowD.getFullYear()-1;
// Se mês < novembro (getMonth < 10): ciclo iniciou no ano passado
// Ex: maio/2026 → cycleY = 2025 → ciclo = nov/2025 a out/2026
function inCyc_(mes, ano, sy) {
  return (ano > sy || (ano === sy && mes >= 11)) && (ano < sy+1 || (ano === sy+1 && mes <= 10));
}
```

---

### Professores — Scatter Nota de Observação × Dropouts no Ciclo

Gráfico de dispersão abaixo dos gráficos globais da seção Dropouts & Observações.

**Canvas:** `#chartDOScatter` | **Detail panel:** `#do-sc-detail`

**Filtros próprios** (independentes dos filtros globais da seção):
- `#do-sc-unidade` — unidade (populado em `initDOFilters()` a partir de `ALL_DROPOUTS`)
- `#do-sc-nivel` — nível; **populado dinamicamente** de `ALL_NOTAS`, ordenado por `NIVEL_ORDER = ['A','B','1.1','1.2','1.3','1.4','2.1','2.2','2.3','2.4','3.1','3.2','3.3','3.4']`
- `#do-sc-min-obs` — mínimo de observações (1/2/3)

**Dados:**
- Eixo X = nota média de `ALL_NOTAS` **filtrado para o ciclo atual (nov–out)** por `chaveMatricula`; **min:4.5, max:10.5**
- Eixo Y = soma de dropouts de `ALL_DROPOUTS` filtrados para o ciclo atual (nov–out) por `chaveMatricula`; **min:-4** (espaço abaixo de 0 para bolinhas não serem cortadas — aumentado de -1.5 para -4)
- Ambos usam a mesma função `inCycSC_` para consistência ciclo × ciclo
- Apenas professores com ≥ `fMin` observações entram no gráfico
- **Apenas professores ativos**: filtro por `Set` de `chaveMatriculaDP` de `ALL_EMP` onde `ativo=true`
- Nível no filtro `#do-sc-nivel`: populado apenas com valores presentes em `NIVEL_ORDER` (`filter(Boolean)` não é suficiente — valores inválidos como "J2B" são excluídos)

**`chaveMatriculaDP` em `getData()`:** `String(row[38]||'').trim().split('|').pop().trim()` — col AM (índice 38) da planilha principal; mesmo formato de matrícula que `chaveMatricula` em dropouts e notas.

**Quadrantes** (divisor = mediana dos dados filtrados):
- `SC_MED_NOTA` e `SC_MED_DROP` calculados a cada render — armazenados globalmente
- Labels: "Nota abaixo da média · Dropout alto/baixo" / "Nota acima da média · Dropout baixo"
- Cores: vermelho (TL), âmbar (TR), cinza (BL), verde (BR) — definidas em `SC_Q_COLORS`
- `SC_Q_LABELS` define o texto dos quadrantes

**Datasets e labels:**
- Datasets de pontos: `label:'q_tl'`, `'q_tr'`, `'q_bl'`, `'q_br'` — **sem prefixo `__`**
- Datasets decorativos (linhas de mediana): `label:'__medNota'`, `'__medDrop'` — prefixo `__`
- `pointHitRadius:18` nos datasets de scatter; `pointHitRadius:0` nos datasets `__med*`
- O plugin `SC_SIGLA_PLUGIN` pula datasets onde `!ds.label || ds.label.startsWith('__med')` (só processa pontos de scatter)

**Plugin de siglas (`SC_SIGLA_PLUGIN`):**
- Plugin **local** (não global): passado via `plugins: [SC_SIGLA_PLUGIN]` no config do chart
- `scSigla_(unidade)` gera sigla dinâmica: 1 palavra → 3 primeiras letras; múltiplas → inicial de cada (ex: "CAMPO GRANDE" → "CG")
- **Nunca usar `Chart.register()` para esse plugin** — afetaria todos os outros gráficos da página

**Tooltip (hover nativo do Chart.js):**
- `mode:'nearest', intersect:true`
- `filter`: `function(item){ return !!(item.raw && item.raw._p); }` — exclui os `__med*` do tooltip
- Exibe: nome do professor (title), unidade, nível, nota média, dropouts no ciclo
- **`options.onClick` / `options.onHover` não são confiáveis** em iframes do Apps Script

**Click detection (painel de detalhe):**
- `canvas.onclick` com `getElementsAtEventForMode(evt,'nearest',{intersect:true},false)`
- Verifica `pt._p` antes de chamar `showSCDetail(pt._p)`
- `canvas.onclick` atribuído por propriedade (não addEventListener) para não acumular listeners ao re-renderizar

**Globals:** `DO_SCATTER_CHART`, `SC_MED_NOTA`, `SC_MED_DROP`, `SC_Q_COLORS`, `SC_Q_LABELS`, `SC_SIGLA_PLUGIN`

---

### Professores — Dropouts por Nível (`chartDONivel`)

Gráfico de barras full-width abaixo dos gráficos globais de top dropouts e distribuição de notas.

- Join entre `ALL_DROPOUTS` (campo `chaveMatricula`) e `ALL_EMP` (campo `chaveMatriculaDP`) para obter o nível do professor
- Nível de `ALL_EMP` vem da coluna AH (col 33) da planilha principal — mesmo campo `nivel` já presente em `getData()`
- Professores **sem nível informado são excluídos** do gráfico (não aparecem como "Não informado")
- `#do-nivel-unmatched` sempre limpo (nenhuma mensagem exibida sobre não-matches)
- Barras ordenadas por `NIVEL_ORDER`; cores navy/âmbar alternadas

---

### People — Tabela Gênero × Função

- Ordenada por headcount decrescente
- Funções com percentual arredondado para 0% são agrupadas em **"Outros (N funções)"** — mesmo padrão da pirâmide de funções na aba Professores

---

---

### Professores — Filtros unificados no topo da página

Todos os filtros da aba Professores ficam em uma única barra no topo, afetando toda a página:

| ID | Label | Afeta |
|---|---|---|
| `#f-prof-status` | Status | Lista de professores |
| `#f-prof-unidade` | Unidade | Lista, gráfico de turmas, pódio, dropouts, scatter, timeline |
| `#f-prof-ano` | Ano | Dropouts e notas |
| `#f-prof-prof` | Professor | Dropouts, notas, timeline individual |

- **`onProfFilterChange()`** — orquestrador central. Ordem: `refreshProfProf()` PRIMEIRO (reseta o dropdown de professor para a nova unidade), depois `renderProfessores()`, `renderTurmasChart()`, `renderPodium()`, `renderDropoutPodium()`, `renderDropoutsNotas()`.
- **`refreshProfProf()`** — popula `#f-prof-prof` filtrando `ALL_DROPOUTS` pela unidade selecionada. Deve ser chamado ANTES dos renders para evitar filtro por professor da unidade anterior zerando os dados.
- **`onProfProfChange()`** — chamado só quando o professor muda; renderiza `renderDropoutsNotas()` + `renderDOTimeline()`.
- **`getDOFiltered()`** — lê `f-prof-unidade`, `f-prof-ano`, `f-prof-prof` e filtra `ALL_DROPOUTS` e `ALL_NOTAS`.
- **Filtro de Nível removido** da barra — não existe mais `#f-prof-nivel` no HTML.

**Seções de dropouts sem filtros próprios** — todos os filtros locais (do-unidade, do-prof, do-sc-unidade, f-podium-unidade) foram removidos; tudo lê os selects globais.

---

### Professores — Pódio Top 3 (ranking por nota média)

Seção no topo da aba Professores (antes dos KPIs de turmas), logo abaixo dos filtros.

**Backend — `getNotasMediasData(token)`:**
- Lê aba `Notas Médias` da planilha `NOTAS_SPREADSHEET_ID`
- Colunas da aba: `Chave Matrícula`, `Nota Média`, `Apelido`, `Unidade 1°`, `Unidade 2°`
- **Normalização de cabeçalhos (`normH_`)**: remove acentos E indicadores ordinais (° U+00B0, º U+00BA, ª U+00AA) — "Unidade 1º" e "Unidade 1°" ambos mapeiam para "UNIDADE 1"
- Professor em duas unidades gera **duas entradas** — aparece no ranking de cada unidade
- Exclui professores cujas unidades são MÉTODOS, EDITORA ou EC NEW (`PODIUM_EXCL_UNITS`)
- **Retorna `photoUrl: null` para todas as linhas** — fotos NÃO são buscadas aqui (era lento: lia Drive para ~297 professores). Fotos buscadas separadamente via `getDropoutPodiumFotos` só para o top 3.

**Backend — `getDropoutPodiumFotos(token, entries)`:**
- Recebe lista de até 3 `{apelido, unidade, chaveMatricula}` — usado por AMBOS os pódios
- Itera a pasta `FOTOS_FOLDER_ID` uma vez; busca em dois passos: 1) match exato `"apelido - sigla"`, 2) fallback por apelido só
- `normP_` = minúsculo + sem acento (NFD + remove combining marks)
- Retorna `{ ok: true, fotos: { [chaveMatricula]: 'data:image/jpeg;base64,...' } }`
- `FOTOS_FOLDER_ID = '1D6fq0r-OPxiK_Pa56rZ4t4BfDkrkDvrs'` — pasta com fotos `APELIDO - SIGLA.jpg`
- **NÃO usar URL direta**: bloqueado por CSP do HtmlService iframe; base64 é o único método confiável

**Frontend:**
- Global `ALL_NOTAS_MEDIAS` preenchido por `loadNotasMediasData()` (chamado 300ms após load de professores)
- Avatar size: **96px** (aumentado de 80px)
- `renderPodium()` — **renderização em duas fases**: (1) renderiza imediatamente com placeholder via `renderPodiumHTML(top, stg)`, (2) chama `getDropoutPodiumFotos` async para o top 3 e re-renderiza com fotos
- `renderPodiumHTML(top, stg)` — helper que monta o HTML do pódio (reutilizado nas duas fases)
- Layout visual: 2º lugar (esquerda) | 1º lugar (centro, elevado) | 3º lugar (direita)
- **Os dois pódios ficam lado a lado** dentro de um `div.podium-row` (`display:flex; gap:16px`); cada `.podium-section` recebe `flex:1; min-width:0; margin-bottom:0`. O `margin-bottom:24px` fica no `.podium-row`.
- `PODIUM_PH` = SVG placeholder base64 para professores sem foto
- Foto no `<img>` com `onerror="this.src=PODIUM_PH;this.onerror=null"` — fallback seguro
- Subtítulo: `"Baseado nas notas médias do ciclo"` (sem "· aba Notas Médias")
- Botão "🔍 Diagnóstico Fotos" e painel `#podium-diag` **removidos** do frontend

**Bug resolvido — performance do ranking de notas:**
- Antes: `getNotasMediasData` lia o Drive listando ~297 arquivos de foto na carga inicial — lento demais
- Depois: `getNotasMediasData` retorna instantaneamente (sem Drive); fotos buscadas só para o top 3 via chamada separada assíncrona

**Bug resolvido — URL direta vs base64:**
- `drive.google.com/uc?id=...&export=view` é bloqueado pela CSP do HtmlService iframe
- `getThumbnailLink()` falha por SameSite=Lax cookie (cross-origin iframe)
- **Solução correta: base64 embutido** — único que funciona de forma confiável no iframe

**Bug resolvido — indicador ordinal º vs grau °:**
- Planilhas usam `º` (U+00BA, indicador ordinal) em "Unidade 1º"
- Código anterior buscava `°` (U+00B0, grau) — caracteres visualmente idênticos mas diferentes em Unicode
- Solução: `normH_` remove AMBOS antes de comparar; busca usa `ci('UNIDADE 1')` sem o caractere especial

---

### Professores — Pódio Top 3 · Menores % de Dropouts

Segundo pódio na aba Professores, imediatamente abaixo do pódio de notas médias.

**Fórmula:** `% dropout = (dropouts / (alunos + dropouts)) * 100`

**Filtro de elegibilidade:** só entram professores com **carga horária média ≥ 42h/mês no ciclo atual (nov–out) em pelo menos uma unidade**. A média é calculada individualmente por unidade — não é somada entre unidades diferentes. O ciclo vai de novembro do ano anterior até o mês atual (ex: em maio/2026 = nov/2025 a mai/2026).

**Fonte da carga horária:** planilha `Horas_2024+` (`HORAS_SPREADSHEET_ID`), já carregada em `ALL_HORAS`. Soma apenas **col H (`HORAS TURMAS`) + col I (`HORAS TURMAS SÁBADO`)** — extras, substituições e faltas não entram.

**Frontend — `renderDropoutPodium()`:**
- Aguarda `ALL_HORAS` estar disponível (guarda similar ao `ALL_DROPOUTS`)
- Constrói `ciclo`: array de `{m, a}` de novembro do ano anterior até o mês atual (UTC para evitar bugs de fuso)
- Chama `mediaHorasCiclo_(chaveMatricula, unidade, ciclo)` para cada registro de dropout
  - Join por matrícula: `ALL_HORAS.matricula` normalizado via `.split('|').pop().trim()` ↔ `ALL_DROPOUTS.chaveMatricula`
  - Retorna `null` se não há registros no ciclo (professor fica fora do ranking)
  - Retorna média mensal de `hTurmas + hTurmasSab` nos meses do ciclo
- Eligibilidade: `mediaH !== null && mediaH >= 42`
- `map` separado por `"chaveMatricula|UNIDADE"` — dropouts/alunos agregados por professor+unidade
- O % de dropout é calculado individualmente para cada unidade onde o professor atua
- Filtra por unidade (`f-prof-unidade`) e por elegibilidade de carga horária
- Ordena pelo % crescente (menor % = melhor)
- Pega top 3 e renderiza via `renderDropoutPodiumHTML(ranked)`
- **Renderização em duas fases**: placeholder imediato → fotos async via `getDropoutPodiumFotos`
- **`renderDropoutPodium()` é chamada também em `loadHorasData` success** para garantir render após horas chegarem

**Frontend — `renderDropoutPodiumHTML(ranked)`:**
- Mesmo layout visual do pódio de notas (2º esquerda | 1º centro | 3º direita)
- Exibe `p.pct.toFixed(1).replace('.',',') + '%'` no lugar da nota
- Canvas: `#dropout-podium-stage`

**Quando é chamado:**
- `loadTurmasData` success handler → `renderDropoutPodium()` (garante que turmaCount seja populado)
- `loadDropoutsData` success handler → `renderDropoutPodium()` (atualiza após dropouts chegarem)
- `onProfFilterChange()` → `renderDropoutPodium()` (reage ao filtro de unidade)

**Bug resolvido — timing ALL_TURMAS vazio:**
- `loadDropoutsData` completava antes de `loadTurmasData`, então `turmaCount` ficava vazio e nenhum professor passava no filtro de 4 turmas
- Solução: chamar `renderDropoutPodium()` também de `loadTurmasData`, garantindo que quando turmas chegam o pódio seja re-renderizado

**Bug resolvido — chaveMatricula mismatch:**
- `ALL_TURMAS.chaveMatricula` formato: `"AISHA | 793"` (string bruta)
- `ALL_DROPOUTS.chaveMatricula` formato: `"793"` (número extraído)
- Solução: `turmaCount` indexado pelo número após `|`, não pelo string completo; mais fallback por apelido

---

### Brindes — filtros Setor e Anos de Casa (adicionados nesta sessão)

Além dos filtros já existentes:
- **`f-brinde-setor`** — `"pedagogico"` = funcao inclui PROFESSOR, COORDENADOR ou é GERENTE DE UNIDADE; `"administrativo"` = demais
- **`f-brinde-anos`** — filtra por anos de casa (5, 10, 15 … 50)

A coluna **Função** foi adicionada à lista de brindes (campo `funcao` da aba `CONSOLIDADO_new`) e ao CSV exportado.

---

### Professores — bug `onProfFilterChange` e dados de dropouts zerados

**Sintoma:** ao trocar de unidade no filtro, dropouts mostravam zero.

**Causa:** `onProfFilterChange()` chamava `renderDropoutsNotas()` ANTES de `refreshProfProf()`. O dropdown `f-prof-prof` ainda tinha o professor da unidade anterior selecionado — `getDOFiltered()` filtrava por ele e não encontrava nada na nova unidade.

**Correção:** `refreshProfProf()` é chamado PRIMEIRO em `onProfFilterChange()`, zerando a seleção de professor antes de renderizar.

---

### Desligamentos — Nuvem de Palavras (`renderDlWordCloud`)

Nuvem de sentimentos na aba Desligamentos, construída a partir dos campos `comentario` e `motivoOutros`.

**Análise dupla — palavras + bigramas:**
- Tokeniza cada comentário e filtra stop words
- Conta palavras individuais significativas (mín. 4 letras, mín. 2 menções, top 35)
- Extrai **bigramas**: pares de tokens significativos consecutivos (mín. 2 menções, top 18)
- Bigramas aparecem primeiro no cloud e são exibidos com **borda tracejada** para diferenciá-los de palavras soltas
- Badge mostra `"N frases · M palavras"`

**Stop words:**
- Lista extensa (~150 entradas) cobrindo artigos, preposições, pronomes, verbos auxiliares, conjunções, advérbios, tempo, numerais e palavras institucionais genéricas
- Palavras institucionais bloqueadas explicitamente: `brasas`, `empresa`, `escola`, `sala`, `unidade`, `professor`, `coordenador`, `gestor`, `colega`, `equipe`, `gente`
- Verbos comuns bloqueados: `tive`, `tinha`, `havia`, `seria`, `quero`, `posso`, `deixo`, `saindo`, `levo`, `fui`, `foi`, `foram`, etc.
- Conectivos e advérbios: `nao`, `sao`, `tambem`, `porque`, `porem`, `entao`, `assim`, `apenas`, `ainda`, `ja`, `so`, `algo`, `longo`, `desses`, etc.
- Para adicionar novas stop words, editar o array `STOPS` em `renderDlWordCloud`

**Sentimento:**
- `POSITIVE` e `NEGATIVE` são Sets de palavras-chave
- Bigramas: classificados se qualquer uma das duas palavras bater com o Set (mas não ambas simultaneamente no sentido contrário)
- Verde = positivo, Vermelho = negativo, Azul-navy = neutro

**Legenda visual** (HTML acima do `#dl-wordcloud`):
- Bolinhas coloridas para Positiva/Negativa/Neutra
- Ícone de borda tracejada para "Frase (2 palavras)"

---

### DP — Filtro por Responsável (Jessica / Priscila)

Dropdown **"Responsável"** adicionado à barra de filtros de todas as 5 abas DP: Horas, Faltas, VR Administrativo, VR Docente e Coparticipação.

**Mapeamento de siglas** (definido no frontend em `DP_JESSICA_UNITS` / `DP_PRISCILA_UNITS`):
- **Jessica:** ME, BOD, BR, GR, NT, VP, VO, NL, MRI, MR, TJ, CG, CX, DT, ED, EC NEW, RC
- **Priscila:** IT, BF, IG, NI, FG, PC, LJ, IP, TQ, CP, VQ, NS, CH, BG, PN, PO
- `MRI` e `MR` são variantes da mesma empresa (Meier) — ambas mapeadas para Jessica
- `NS` e `CH` são variantes da mesma empresa (Cachambi) — ambas mapeadas para Priscila

**Função helper:** `getDPResp(u)` — converte sigla em `'jessica'` ou `'priscila'` (string `.toUpperCase()` + `indexOf`)

**IDs dos filtros:** `h-resp` (horas), `f2-resp` (faltas), `vra-resp` (VR adm), `vrd-resp` (VR doc), `cp-resp` (copa)

**Funções afetadas:** `filteredHoras`, `filteredFaltas`, `renderVRAdm`, `renderVRDoc`, `filteredCopa` — cada uma lê seu respectivo `resp` e aplica `getDPResp(r.unidade) !== resp` como condição de exclusão.

**Botão Limpar:** cada `clearXFilters()` já inclui o campo `resp` correspondente.

> O filtro é **manual** (visível na UI, qualquer usuário pode usar). Não há filtragem automática por sessão/login.

---

### Abrir no Sheets — botões nas abas DP

Botão "↗ Abrir no Sheets" adicionado ao lado de "⬇ Exportar CSV" nas tabelas das 4 abas do DP:
**Horas** (`#sheets-horas`), **VR Administrativo** (`#sheets-vra`), **VR Docente** (`#sheets-vrd`), **Coparticipação** (`#sheets-copa`).

**Backend — `createTempSheet(token, title, headers, rows)`:**
- Cria uma nova planilha Google com os dados filtrados
- Cabeçalho formatado: fundo `#162d4a`, texto branco, negrito
- Linha 1 congelada, colunas auto-redimensionadas
- Retorna `{ ok: true, url }` — frontend abre a URL em nova aba

**Frontend — helper `_openSheets_(btn, title, headers, rows)`:**
- Desabilita o botão e mostra "⏳ Criando…" durante a chamada
- Chama `google.script.run.createTempSheet(TOKEN, title, headers, rows)`
- Em sucesso: `window.open(r.url, '_blank')`

**Funções individuais:** `openHorasSheets`, `openVRAdmSheets`, `openVRDocSheets`, `openCopaSheets` — usam os mesmos dados e filtros ativos das funções de CSV correspondentes.

**CSS:** `.btn-sheets` — verde `#1a7340` / fundo `#e6f4ea`

---

### Aba Wellhub (`getWellhubData`)

**Acesso:** roles `dp` e `admin`. Verificação via `session.allowedPages` → fallback hardcoded.

**Backend — `getWellhubData(token)`:**
- `WELLHUB_SPREADSHEET_ID = '1bFjtYHLn1R4_ZgkSnNyxrb3a6JhYJ72bHmwtoBg1Wzs'`, aba `Wellhub`
- Colunas lidas via `normalizeH_`: `Mês de Referência`, `Ajuste Empresa`, `Colaborador`, `Email do colaborador`, `Dependente`, `Tipo`, `Plano`, `Preço após desconto`, `Data de início do plano`, `Status`
- `precoFinal` converte vírgula → ponto antes do `parseFloat`

**Frontend:**
- Tab `💪 Wellhub` na navbar após Brindes
- `ROLE_TABS`: `dp` e `admin` incluem `'wellhub'`; `ROLE_TABS['diretor']` e outros **não** incluem
- Aba ROLES: coluna `rh_wellhub` controla acesso por role customizado
- Filtros: `wh-mes` (mês mais recente pré-selecionado), `wh-empresa`, `wh-plano`, `wh-status`
- 4 KPIs: Assinantes Ativos (Employee·Active) · Dependentes Ativos (FamilyMember·Active) · Custo Total (Active) · Plano Mais Popular
- 3 Gráficos: donut planos · barras horizontais status · linha histórico mensal (ALL_WELLHUB, ignora filtro de mês)
- Tabela full com CSV + Abrir no Sheets
- Globais: `ALL_WELLHUB`, `WH_CHARTS`, `WH_PLANO_ORDER`
- Ordem dos planos: `DIGITAL → BASIC → BASIC+ → SILVER → SILVER+ → GOLD → GOLD+`

**⚠ Ação necessária na planilha:** adicionar coluna `rh_wellhub` na aba `ROLES` da planilha principal para os roles que devem ter acesso.

---

## Fluxo de trabalho no Git

Esse projeto não tem CI nem deploy automático. As alterações são:
1. Feitas e testadas diretamente no editor do Apps Script.
2. Copiadas para os arquivos `.txt` locais.
3. Commitadas e enviadas ao GitHub **manualmente** (`git push`) quando solicitado.

**Sempre fazer `git pull` antes de começar a editar** para garantir que a cópia local está atualizada.
