# Relatório Mensal de Caixa — Vetor SST

Duas páginas em arquivo único cada, hospedadas no GitHub Pages e alimentadas por planilhas-ponte do Google Sheets publicadas em CSV:

- **`index.html`** — relatório ao vivo do mês corrente (projeções, Monte Carlo), fonte: ponte **relatorio-api**.
- **`fechamento.html`** — ata de fechamento mensal, navegável mês a mês, fonte: ponte **fechamento-api**.

As páginas compartilham a mesma senha (sessão vale para as duas) e têm links cruzados na barra superior.

## Relatório ao vivo (`index.html`)

Os 7 números dos cards **Receita** e **Despesa** são carregados automaticamente de uma planilha Google Sheets publicada em CSV; todo o resto (projeções, Monte Carlo, observações) é calculado no próprio navegador.

## Fonte de dados: a ponte "relatorio-api"

O relatório **não** lê a planilha Financeiro 2026 diretamente. Ele lê uma planilha-ponte separada (**relatorio-api**), publicada na web como CSV, com 9 linhas:

| chave (col. A) | valor (col. B) | descrição (col. C — ignorada pelo relatório) |
|---|---|---|
| `chave` | `valor` | cabeçalho |
| `caixa` | ex.: `201322,12` | Caixa gerado no mês |
| `receber` | | A receber no mês |
| `atrasados` | | Atrasados |
| `saldo` | | Saldo em conta |
| `sicredi` | | Sicredi Invest |
| `paga` | | Despesa paga no mês |
| `apagar` | | Despesa a pagar no mês |
| `saldo_inicial_mes` | | usada por outras integrações — **ignorada** pelo relatório |

As fórmulas da ponte replicam os filtros do Looker sobre as abas **Fluxo de Caixa**, **Contas a Receber** e **Contas a Pagar** da Financeiro 2026. O recorte de mês é dinâmico: no dia 1º a ponte passa a refletir o mês novo sozinha, sem intervenção.

Formato dos valores: decimal com vírgula, sem separador de milhar (ex.: `201322,12`). Valores negativos (`-1234,56`) são suportados. O carregador também tolera `1234.56`, `R$ 1.234,56` e variantes.

## Como conectar (setup único)

1. Na planilha **relatorio-api**: *Arquivo → Compartilhar → Publicar na web* → escolher **somente a aba da ponte** (nunca "Documento inteiro") → formato **Valores separados por vírgula (.csv)** → Publicar.
2. Copiar a URL gerada — formato `https://docs.google.com/spreadsheets/d/e/2PACX-…/pub?gid=NNN&single=true&output=csv`. Ela precisa conter `output=csv` (não usar a URL `pubhtml` nem a de edição).
3. Colar a URL na constante `SHEET_CSV_URL`, no início do `<script>` do `index.html`.
4. Commit + push → GitHub Pages publica.

Enquanto a URL não for configurada, o relatório funciona como sempre (valores locais) e mostra "Planilha não configurada" na barra superior.

## Comportamento

- **Ao abrir**: a página carrega instantânea com os últimos valores salvos e, assim que o CSV responde, os 7 números são atualizados, tudo recalcula e o status mostra "Atualizado da planilha em DD/MM HH:MM".
- **Botão 🔄 Atualizar**: re-busca a planilha sob demanda, sem recarregar a página.
- **Sem conexão / planilha fora do ar**: o relatório abre normalmente com os últimos valores bons (localStorage) e avisa "usando valores locais". Nunca quebra.
- **Edição manual**: continua funcionando como sempre (botão ✎ Editar). Vale até o próximo carregamento da planilha — a ponte é a fonte de verdade dos 7 números.
- **↓ Baixar HTML**: o snapshot baixado é um registro histórico **congelado** — ele não re-busca a planilha ao ser aberto (atributo `data-snapshot` no HTML).
- **Célula vazia ou chave ausente no CSV**: o valor atual do relatório é mantido (fallback).

## Fechamento mensal (`fechamento.html`)

Página de consulta do fechamento de cada mês — **é ata, não rascunho**: nada é editável. Navegação ‹ › entre meses; o padrão ao abrir é o mês anterior ao atual. O estado é definido pela **data** (não pelo valor): meses passados aparecem como "Fechado", o mês corrente como "Em andamento (parcial)" e meses futuros ficam desabilitados na navegação. Células vazias na ponte aparecem como "—".

### Fonte: a ponte "fechamento-api"

Planilha-ponte publicada na web como CSV, com a **mesma fonte dos BIs** (não o FCO). Formato: 1 linha de cabeçalho + 12 linhas (`2026-01` … `2026-12`), 22 colunas:

| Grupo | Colunas |
|---|---|
| Identificação | `mes` (AAAA-MM) |
| Resumo | `receita`, `despesa`, `resultado`, `saldo_inicial`, `saldo_final`, `margem_pct` |
| Carteira | `carteira`, `recebido`, `inadimplentes`, `inadim_pct` |
| Receita por serviço | `receita_exames`, `receita_laudos`, `receita_assessoria`, `receita_treinamentos`, `receita_certificados`, `receita_outras` |
| Despesa por blocos | `desp_exames` (repasse), `desp_pessoal`, `desp_marketing`, `desp_demais` |
| Linha de Exames | `lucro_exames` (= `receita_exames` − `desp_exames`; há meses legitimamente negativos) |

Números no CSV vêm com vírgula decimal entre aspas (`"241639,97"`) ou sem aspas quando inteiros (`135890`). Célula vazia = sem dado.

### Comportamentos

- **Cache offline**: o último CSV bom fica no `localStorage` (`vs_fech_v1`); sem conexão a página abre com ele e avisa "usando dados locais de DD/MM HH:MM".
- **↓ Baixar HTML**: gera snapshot congelado (`data-snapshot`) — não re-busca a planilha e trava o seletor no mês do snapshot.
- **Inadimplência histórica**: reflete o **status ATUAL** dos títulos daquele mês — um título de março pago em julho sai da inadimplência de março. Comportamento esperado (a conciliação é viva), não é bug.

## Limitações conhecidas (aceitas)

- O CSV publicado tem **cache de ~5 min no Google**: edições na ponte demoram até 5 minutos para aparecer, mesmo clicando em Atualizar.
- "Publicar na web" torna a aba da ponte **pública** para quem tiver a URL (que fica visível no fonte do `index.html`). Os 7 números já são públicos hoje (repo/Pages públicos) — por isso a ponte deve conter **apenas** essas linhas, nunca outras abas ou dados sensíveis.
- A senha do relatório é proteção de fachada (hash no cliente); não tratar o relatório como confidencial.

## Desenvolvimento local

```
python -m http.server 8000
# abrir http://localhost:8000  (não usar file:// — o login exige contexto seguro)
```
