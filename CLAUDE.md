# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este repositório

Dois dashboards de produção do setor de Acabamento (Pangeia 96), cada um em **um único arquivo HTML autocontido** (HTML + CSS + JS inline, ES5, sem framework):

- `index.html` (~4.500 linhas) — dashboard analítico completo, 5 abas, usado no desktop.
- `dashTV_acabamento.html` — versão "modo TV": ranking gamificado em tela cheia, tema escuro, auto-refresh e auto-scroll, sem interação.
- `imagem_pegada_urso.png` — usada só pelo dashTV (caminho relativo).

Não há build, bundler, dependências npm, testes ou linter. A única dependência externa é **Chart.js 4.4.1 via CDN** (apenas no `index.html`; o dashTV não usa gráficos).

## Como rodar

```powershell
# Abrir direto no navegador (suficiente para index.html)
start index.html

# Servir via HTTP (recomendado para o dashTV, que carrega o PNG local)
python -m http.server 8000   # depois: http://localhost:8000/dashTV_acabamento.html
```

Toda alteração é validada abrindo a página no navegador e conferindo o console — não existe suíte de testes. Deploy = commit + push na `main` (remote: `thiagopangeia96/pangeia-dash-acabamento`).

## Fonte de dados: Google Apps Script Web App

Ambos os arquivos apontam para a **mesma** constante `SCRIPT_URL` (um Web App `/exec` do Apps Script que lê as planilhas). Ela está **duplicada** nos dois HTMLs — ao trocar a URL do deployment, atualize `index.html:686` **e** `dashTV_acabamento.html:271`.

Endpoints usados:

| Chamada | Retorno | Onde |
|---|---|---|
| `?t=<ts>` (sem action) | `{colaboradoras[], colabsInfo[{nome,contratacao,areaPrincipal}], complexidadeModelos{}}` | `carregarListas()` |
| `?action=registros&t=<ts>` | `{registros: [][]}` — apontamentos de acabamento | `carregarDados()` (ambos os arquivos) |
| `?action=recebimento&t=<ts>` | `{recebimentos: [][], cabecalho: []}` — planilha de recebimento de facção | `carregarEfic()` (só aba Eficiência) |

`registros` vem como **array de arrays posicional** — esse mapeamento é o contrato mais importante do projeto (`processarRegistros()`):

```
0 data   1 col (colaboradora)  2 tipo   3 os   4 fac (facção)  5 mod (modelo)
6 cor    7 tam                 8 tot    9 ok   10 baz (bazar)  11 con (conserto)  12 suj (sujo)
```

Strings de identidade (`col`, `os`, `fac`, `mod`, `cor`) são normalizadas para MAIÚSCULAS + trim na entrada; comparações posteriores assumem isso. Registros com data inválida são descartados. O dashTV usa só os campos 0, 1, 2 e 8.

`recebimentos` é posicional também: `1` data, `2` facção, `3` OS, `4` produto, `5` cor, `6` tipo, `7..16` quantidades por tamanho (nomes vindos de `cabecalho`).

### Categorias de trabalho (constantes de negócio)

`TIPO_REV = 'Revisão'` é sempre tratado separado da produção. `TIPOS_ACAB = ['Linha / Embalagem','Trocas e Consertos']` é o que conta como "Acabamento"; `TODOS_PROD` acrescenta `Etiquetagem` e `Prensa Térmica`. Metas, cores e ícones por área ficam em `METAS_IND`, `METAS_AREA_P`, `COR_AREA`, `ICONE_AREA` (`index.html:2097-2100`). No dashTV a lista de tipos de acabamento é redefinida localmente (`Linha / Embalagem` + `Trocas e Consertos`) — ele não conhece Etiquetagem/Prensa.

## Arquitetura do `index.html`

### Fluxo de dados (global mutável, por design)

```
carregarDados() ──> todos[]            (todos os registros processados)
        └─ cache: sessionStorage 'pangeia_dash_cache_v4', TTL 15 min
filtrar() ────────> window._base       (todos[] filtrado pelo período dtI/dtF)
                    filt               (_base + filtroTipoAtivo; nunca filtra Revisão)
baseFiltrada() ───> _base/filt + filtro global "Só FIXOS" (_soFixos, via _tiposColab)
render*() ────────> innerHTML
```

- `aplicar()` (botão Aplicar e os presets `.pbtn`) **não vai ao servidor** — só refiltra em memória. Rede só em `carregarDados()`.
- `carregarDados(false)` prefere o cache; `carregarDados(true)` força rede. Auto-refresh forçado a cada 10 min; em erro de rede cai para o cache mesmo expirado.
- `renderKPIs()` recalcula sua própria base a partir de `todos` e **ignora** `_soFixos` de propósito (KPIs refletem a operação real, FIXO+TESTE). `renderProd()` também usa `window._base` no gráfico diário pelo mesmo motivo. Ao mexer em filtros, respeite essa assimetria.
- KPIs por tipo mostram sempre o **último dia com registro** no período (`dr`), comparado contra a média diária do período.

### Abas (`goTab(id, btn)`)

`prod` · `qual` · `falhas` · `perf` · `efic`. `prod`/`qual` são renderizadas em `renderTudo()`; as outras renderizam sob demanda no `goTab`.

- **Produtividade** — gráfico diário por tipo, ranking de colaboradoras com seleção múltipla (`colabsSelecionadas`) que alimenta o painel comparativo, tabela Modelo/Cor/Tamanho, popup de produtividade média.
- **Qualidade** — só registros de Revisão; defeitos = `baz + con + suj`, agregados por facção e por dia, com popups de drill-down (`abrirCol/abrirFac/abrirDia` → sub-popups de modelo/colaboradora).
- **Controle de Falhas** — rastreamento reverso: filtra por facção/OS/modelo/cor/tamanho e lista quais colaboradoras tocaram o item. Exige ao menos um filtro preenchido.
- **Performance Individual** — visão por área, pódio, comparativo (`_perfCompSet`, máx. 4 cores), ficha individual (`abrirFichaColab`) e detalhe (`abrirPerfDetalhe`). Depende de `_tiposColab` (FIXO/TESTE), que chega em `carregarListas()` — por isso essa função re-renderiza a aba se ela já estiver ativa.
- **Eficiência Produtiva** — única aba com carregamento lazy (`carregarEfic()`, tela de boas-vindas com botão até o primeiro load; `_eficCarregado`). Cruza recebimento × apontamento para medir lead time até a 1ª revisão e até a 1ª linha/embalagem, com sub-abas `rev` / `lin` / `pend` (`setEficSubTab`).

### Regras da aba Eficiência (funções prefixadas `efic*`)

- `EFIC_DATA_INICIO = 2026-05-14` — corta tudo antes do novo modelo de revisão, nos dois lados do cruzamento.
- `eficClassificar(prod, tipo)` → `'excluir'` (brindes e calças, por tipo ou por lista de modelos), `'somente_linha'` (tricot — vem pronto, não passa por revisão) ou `'normal'`.
- Cruzamento por **chave composta** `OS numérica + produto normalizado` (`eficIndexReceb` / `eficIndexAcab`), com fallback só pela OS. `eficNormProduto` colapsa `BRA 001/002` (TOP vs REGATA) numa chave só; `eficNorm` remove acentos/caixa/espaços. Do lado do apontamento, o modelo é cortado antes do `—` porque o recebimento só tem o nome do produto.

### Padrões de UI a respeitar

- Renderização é **concatenação de string em `innerHTML`**; handlers em markup gerado usam `onclick="fn(...)"` global ou atributos `data-*` capturados por **um único listener delegado** registrado no `load` (`.ficha-btn`, `data-efickpi`, `data-eficmod`, `data-eficos`, `data-eficsort`) — payloads compostos vêm `encodeURIComponent`ados e separados por `|||`. Prefira a delegação por `data-*` em código novo.
- Toda função chamada de markup precisa continuar **global** (script único, sem módulos).
- Instâncias de Chart.js ficam em `charts{}` / `_eficCharts{}` e precisam de `.destroy()` antes de recriar o mesmo canvas.
- Cores, espaçamentos e semântica visual vêm de CSS custom properties no `:root` (roxo/lavanda/amarelo Pangeia; `--bad/--warn/--ok`; `--c-rev/--c-lin/--c-troc/--c-etiq/--c-pren`). Use as variáveis, não hex solto.

## Arquitetura do `dashTV_acabamento.html`

Reimplementação enxuta e independente (não compartilha código com o `index.html`; utilitários como `fd`, `n`, `processarRegistros` são duplicados de propósito):

- Dois rankings lado a lado (Linha/Acab e Revisão) com medalhas ouro/prata/bronze, e um rodapé de **Record** = maior produção de uma colaboradora em um único dia, calculado sobre **todo o histórico**, não sobre o período filtrado.
- Sem cache: `carregarDados()` sempre vai à rede; auto-refresh a cada **5 min**.
- `render()` dispara `passagemUrso()` — animação central das pegadas em toda atualização de tela.
- **Auto-scroll TV** (`autoScrollTV`, a cada 5 s): posição, direção e pausa de cada ranking são mantidas em `_scrollPos/_scrollDir/_scrollHold` porque ler `el.scrollTop` durante o `scroll-behavior:smooth` devolve valor defasado. Não substitua por leitura direta de `scrollTop`. Redesenhar a lista reseta o estado para o topo.
- Virada de dia: um timer de 1 min detecta que o filtro está em "dia único" (`dtI === dtF`) e move para a data nova.
- Animações são desligadas em `prefers-reduced-motion` — mantenha esse cuidado ao adicionar efeitos.

## Convenções

- Código, comentários, identificadores e UI em **português (pt-BR)**; datas/números formatados com `toLocaleString('pt-BR')`.
- Estilo ES5: `var`, `function`, callbacks em vez de `async/await`, `fetch().then()`. Há uso pontual de `Set`, spread, `Object.assign` e `closest` — aceitável, mas não introduza sintaxe que exija transpilação.
- Mensagens de commit em português **sem acentos**, no imperativo, prefixadas com o alvo quando aplicável (`dashTV: corrige subida do ranking...`).
- Mantenha os arquivos autocontidos: sem novos arquivos JS/CSS, sem passo de build.

## Armadilhas conhecidas

- `SCRIPT_URL`, `processarRegistros`, `fd`, `n` e as listas de tipos existem em duplicidade nos dois HTMLs. Mudança de contrato de dados exige editar os dois.
- `fd(d)` usa `toISOString()` (UTC), enquanto os filtros montam `new Date(s+'T00:00:00')` (local). Perto da meia-noite / em fuso negativo a "data do registro" pode cair no dia anterior. Não mexa nisso sem verificar as duas pontas.
- `index.html` define `iniciarPerfArea()` duas vezes: a versão real (`:2107`, que monta as tabs em `#perfAreaTabs`) é **sobrescrita** pelo noop de compatibilidade (`:3403`) — na prática as tabs de área são criadas por `renderPerfMainContent()`. `_perfCompSet` também é declarado duas vezes (`:690` e `:2065`). Ao editar essa região, confirme qual definição está em vigor.
- Cache do dashboard é versionado no nome da chave (`pangeia_dash_cache_v4`): se o formato de `processarRegistros` mudar, incremente a versão para invalidar caches antigos dos usuários.
