# CLAUDE.md — Relatório de Caixa Vetor SST

Contexto para qualquer sessão (Claude ou humano) que for mexer neste repo. Leia antes de alterar fórmulas.
Origem: parecer de um "conselho" (CFO de PME, estatístico, tech lead, designer de dashboards) em 27/10/2026, sobre o estado do commit `a589944`.

## Regras de trabalho (obrigatórias)

- **Nunca commitar direto na `main`.** Branch + PR; o Pedro (e o Leonardo, quando mexer em fórmula) aprova.
- **Testar local antes do PR:** `python -m http.server 8000` → http://localhost:8000 (não usar `file://`). O localStorage do localhost é separado do site publicado.
- **Mudou fórmula?** Atualize no mesmo PR: o texto explicativo na página (`projNote`, Observações, tooltips), o `README.md` e este arquivo.
- **Nenhum número de negócio novo hard-coded** (percentuais, fatores). Destino: chaves na ponte `relatorio-api` (ver Fase 1).
- **Dado ausente nunca vira zero silencioso.** Mostrar aviso visível ("sem dado") em vez de projetar com 0.
- **Não engolir erros** (`catch(e){}`): logar e mostrar banner.
- Não colocar no repo dados sensíveis, IDs de planilha, senhas ou hashes novos — o repo é público (ver Segurança).
- Comunicação com o Pedro: português, direto, conciso.

## Arquitetura atual

- `index.html` — relatório ao vivo do mês (arquivo único HTML/CSS/JS, GitHub Pages). Funções-chave: `carregarPlanilha`, `recalc`, `parseBR`, `bizDays`/`bizDaysFrom`, `simulate`/`calcStats` (Monte Carlo), `persistData`/`restoreData`, `downloadHTML`, migrações `ensure*`.
- `fechamento.html` — ata mensal só-leitura (não calcula, só exibe a ponte `fechamento-api`).
- Fluxo: Financeiro 2026 (Google Sheets) → planilha-ponte → "Publicar na web" CSV → `fetch` no navegador → cache em localStorage.
- Estado mora no DOM (contenteditable lido como número; linhas achadas por `data-role`/palavra-chave; `innerHTML` salvo no localStorage `vs_report_v1`). É a principal fonte de fragilidade.

### Ponte `relatorio-api` (chaves hoje)
`caixa`, `receber`, `atrasados`, `saldo`, `sicredi`, `paga`, `apagar`, `saldo_inicial_mes` (ignorada).
**`vendas_avista` NÃO existe na ponte** — o código passou a depender dela em 23/10 (ver bugs).

### Ponte `fechamento-api`
1 linha por mês (`2026-01`…`2026-12`), 22 colunas (ver README).

## Fórmulas atuais (a589944)

- Ritmo diário de vendas = `vendas_avista` ÷ dias úteis decorridos (seg–sex, sem feriados; hoje conta como restante).
- Projeção conservadora (herói) = Saldo + A receber × (1 − 0,15) + ritmo × 0,30 × dias restantes − A pagar.
- "Cenário positivo" = Saldo + A receber × 0,95 + ritmo × 1,00 × restantes − A pagar.
- Posição de hoje (déficit/superávit) = Saldo + A receber × 0,85 + Atrasados − A pagar.
- Monte Carlo: Recursividade = A receber × 0,85/0,90/0,95; Vendas Novas = ritmo × 0,30/0,65/1,00 × restantes; Falta = A pagar − Saldo.

## Fatos de negócio confirmados pelo Pedro (27/10/2026)

- **Vendas à vista geram boleto, Pix e antecipação de cartão de crédito.**
  - Já vendido no mês: Pix/cartão já está no Caixa/Saldo; boleto em aberto já está no A receber → **não projetar de novo**.
  - Venda futura (dias restantes): Pix/cartão vira caixa no mês; boleto pode vencer no mês seguinte.
  - Logo, o fator sobre o ritmo deve significar **"% da venda à vista que vira caixa ainda no mês"** — medir pela forma de pagamento/vencimento na base Comercial, não chutar.
- **Receita de Exames entra no A receber** (pelo menos a parte relevante). Portanto **Exames NÃO deve ser fonte separada no Monte Carlo** (o MC antigo contava em dobro). A parte de Exames fora do A receber fica de fora da projeção (viés conservador aceito).

## Avaliação das mudanças do sócio (23/10, commits 866bae6..a589944)

**Manter (acertos):**
- Ritmo pelas vendas à vista (não pelo Caixa gerado — o Caixa inclui recebimento de títulos e contava em dobro com o A receber; o fator −70% era remendo disso).
- Exames fora do Monte Carlo.
- Falta a cobrir = A pagar − Saldo.

**Reverter/corrigir (bugs):**
1. `vendas_avista` ausente na ponte → vendas projetadas = 0 em todos os cenários; valor digitado à mão fica só no localStorage de quem digitou (cada sócio vê outro número). Criar a chave na ponte; se vazia, aviso visível.
2. `setInterval(carregarPlanilha, 5min)` em `showApp` sobrescreve edições manuais (inclusive Saldo, editável de propósito desde b14183d) sem avisar. Remover ou não sobrescrever campos editados.
3. "Vendas à vista" sempre editável (classe `.live`), até fora do modo ✎ Editar — qualquer clique muda o número principal. O aviso "preencha (✎ Editar)" está incorreto.
4. `parseBR`: regex `/^-?\d{1,3}\.\d{3}$/` torna o parse inconsistente ("0.150"→150, "1234.567"→1234,567). A normalização do CSV já tratava "90.000". Reverter.
5. `ensurePosSection` recria o "Cenário positivo" mesmo se a seção for removida, e move o MC para o fim. O título "Lucratividade" está errado (é saldo de caixa, não lucro).
6. Comparação Base × Pessimista do MC morreu (todos os campos readonly → cenários iguais); cenários salvos antigos (com Exames) são comparados com o modelo novo.
7. Texto-semente das Observações ainda fala em "semana / 7 dias / R$ 20.000" e aparece quando o `recalc` falha.

## Problemas pré-existentes

- **Muitos números-resposta** para a mesma pergunta (conservador, positivo, posição de hoje, probabilidade do MC). Nenhum diz "use este".
- **Parâmetros no feeling:** fator foi −25%→−30%→−50%→−60%→−70%; inadimplência 20%→15%.
- **Atrasados a 100%** na Posição de hoje (deveriam ter o maior desconto, não o menor).
- **Sicredi Invest** carregado mas fora de todas as contas.
- **Dias úteis sem feriados** (ex.: 12/10; em nov/2026, 02/11 e 20/11). Divisor pequeno nos primeiros dias do mês (1 venda grande vira o ritmo do mês).
- **Monte Carlo sem valor informativo:** limites = os dois cenários determinísticos; fontes tratadas como independentes; faixas arbitrárias (±5%) vs. variação real de receita ~27%/mês. Hoje resulta em "100%" por aritmética.
- **Horizonte "fim do mês" esconde o risco:** folha (R$ 48–64k) sai no 5º dia útil do mês seguinte; impostos/FGTS dia 20; A pagar só tem títulos já lançados.
- **Snapshot baixado** recalcula com a data de hoje e pode compartilhar localStorage com outros snapshots abertos via `file://`.
- **Segurança:** repo e Pages públicos; URL do CSV e hash SHA-256 (sem salt) da senha no fonte → dados financeiros legíveis sem senha. A URL/hash atuais ficam no histórico do git: precisam ser **trocados**, não só removidos.

## Dados históricos (fechamento-api, 2026)

- Inadimplência (% carteira, fev–ago): média 7,2%, mediana 3,9%, desvio 6,3 p.p., mín 0,8%, máx 16,7% (ago). Bimodal (abr 15%, ago 17%). Os 15% fixos estão perto do pior caso. Atenção: o histórico mede inadimplência sobre a carteira do mês inteiro; o código aplica sobre o saldo **ainda em aberto** no dia — taxa real tende a ser maior ali.
- Receita média ~R$ 157k/mês (desvio 27%). Assessoria é a linha mais estável; Laudos e Exames as mais voláteis.
- Margem média ~3,3%, dois meses negativos (abr, ago). Saldo de fim de mês entre R$ 1k e R$ 39k (< 1 semana de despesa).
- **Base com furos:** set/2026 incompleto (receita R$ 37k, pessoal vazio); jan/2026 "plugado" (receita = despesa, saldo 0); saldo inicial ≠ saldo final do mês anterior em várias passagens (−R$ 8k cravados em jun→jul e jul→ago) — provável retirada/transferência não lançada.
- Com 8 meses **não dá** para estimar caudas/probabilidades nem fazer backtest (nada foi registrado no meio do mês).

## Plano em fases

| Fase | O quê |
|---|---|
| 0 | Reversão seletiva do diff de 23/10 (itens "Reverter/corrigir" acima), mantendo os acertos. Criar `vendas_avista` na ponte. Trocar URL do CSV (despublicar/republicar a ponte) e a senha. |
| 1 | Parâmetros na ponte (`inad`, `fator_vendas`, `vendas_avista`, …); chave obrigatória ausente = erro visível; feriados nacionais + Porto Alegre; mínimo de dias no divisor do ritmo (ou mistura com média de 3 meses); cálculo em funções puras (`calc.js`) com testes `node --test`, sem build. |
| 2 | Um número só com faixa (conservador–provável–otimista); alerta acionável ("faltam R$ X até dia Y; alavancas: cobrar atrasados, resgatar Sicredi, renegociar A pagar"); remover "Cenário positivo" e card de déficit separado; MC recolhido em "Como calculamos" até ter calibração; vocabulário simples (Entra/Sai/Sobra, Recebíveis); PDF em 1 página. |
| 3 | Snapshot diário da projeção (data, entradas, parâmetros, projeções) para backtest; `fechamento.html` mostra previsto × realizado, tendência 12 meses, Δ vs. mês anterior, "Mês sem dados ainda" em vez de "—". |
| 4 | Visão rolante de 8 semanas com folha/impostos/recorrentes por vencimento; destaque = menor saldo projetado e em que semana; caixa mínimo formal (folha + encargos + impostos dos próximos 15 dias). |
| 5 | Hospedagem com login real (Cloudflare Pages + Access ou Vercel com proteção) + CSV não-público (função server-side busca a planilha); repo privado; `main` protegida, PR com preview e CI. |

## Pendências em aberto

- Medir na base Comercial a fração da venda à vista por forma de pagamento (boleto × Pix × cartão) e o prazo do boleto → define o `fator_vendas`.
- Corrigir set/2026 e as quebras de saldo inicial/final na Financeiro 2026 antes de calibrar qualquer parâmetro.
- Decidir se Sicredi Invest entra como reserva disponível no caixa.
