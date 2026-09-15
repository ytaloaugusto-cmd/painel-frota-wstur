# Painel da Frota — W2S Locação / S2 Turismo — Documentação técnica

> **INSTRUÇÃO PARA A CLAUDE (leia isto primeiro, em qualquer conversa sobre
> este projeto):**
>
> 1. Este arquivo é a **fonte de verdade** do projeto. No início de qualquer
>    conversa nova sobre o painel, leia este arquivo direto do link abaixo
>    antes de fazer qualquer alteração — não confie em resumo de conversa
>    antiga nem em memória própria, que podem perder detalhes ou inventar
>    informação:
>    `https://raw.githubusercontent.com/ytaloaugusto-cmd/painel-frota-wstur/main/NOTES.md`
> 2. **Sempre que você fizer uma mudança relevante no projeto** (nova
>    funcionalidade, mudança de arquitetura, decisão de design, pendência
>    resolvida ou nova pendência, mudança na automação do n8n, etc.),
>    **atualize este mesmo arquivo** (edite as seções correspondentes,
>    atualize a data em "Última atualização") **e faça o commit dele no
>    repositório**, junto com a mudança de código — pelo mesmo fluxo de
>    upload descrito na seção 3. Nunca deixe este arquivo desatualizado em
>    relação ao que está de fato no ar.
> 3. Se o usuário pedir para "salvar"/"anotar" algo sobre o projeto, o lugar
>    certo é aqui — não apenas responder na conversa.

Última atualização: 2026-09-15

Link direto (sempre a versão mais atual):
`https://raw.githubusercontent.com/ytaloaugusto-cmd/painel-frota-wstur/main/NOTES.md`

---

## 1. O que é o projeto

Painel HTML único (`index.html`) de gestão da frota da W2S Locação / S2
Turismo (Maceió/AL): inventário de veículos, manutenção preventiva (KM x
revisão), gráficos, referência técnica dos motores e ocorrências (O.S.).

- Repositório GitHub: `ytaloaugusto-cmd/painel-frota-wstur`, branch `main`.
- Publicado via **GitHub Pages**.
- Domínio próprio: **https://frotaws.com** (DNS apontando pro GitHub Pages).
- Protegido por **Cloudflare Access** (login obrigatório antes de ver o site).
- Todo o app é client-side: um `index.html` só, sem backend próprio.

## 2. Estrutura de arquivos no repositório

- `index.html` — o painel inteiro (HTML + CSS + JS em 2 blocos `<script>`).
- `data/manutencao.json` — pequeno arquivo JSON, atualizado automaticamente
  pelo n8n, com KM atual e próxima revisão de cada veículo (ver seção 4).
- `CNAME` — domínio customizado do GitHub Pages (`frotaws.com`).
- `busca.html`, `menu.html` — páginas auxiliares antigas (não mexidas
  recentemente).

Dentro do `index.html`, dois blocos `<script>`:
1. **Script principal** — contém `const FROTA=[...]` (64 veículos, campos:
   `f,p,m,cat,mot,motorSN,motorObs?,cv,cap,ano,emp,antt,chas,renavam,cil,torq,
   euro,cambio,pbt,oleo,interv`), `const MANUT={...}` (por frota: `plano,ulRev,
   dataRev,kmAtual,proxRev,obs,paradoDesde?`), `const CADASTRO={...}` (por
   frota: `plot,cor,prop,crv,fin`), e todas as funções de render/lógica.
2. **Script da aba Ocorrências (O.S.)** — uma IIFE separada com
   `const fleet=[...]` e `window.ocRenderCharts`. **Nunca deve ser tocado**
   em atualizações relacionadas a KM/manutenção — é um módulo independente.

## 3. Como fazer deploy de uma alteração

Não há CI/CD nem `git push` configurado nesta sessão (o ambiente de trabalho
da Claude não é um clone git, é só uma cópia local dos arquivos). O deploy é
manual, via interface web do GitHub:

1. Editar o arquivo localmente.
2. Rodar `node --check` nos dois blocos `<script>` extraídos, pra garantir
   que não quebrou a sintaxe.
3. Ir em `https://github.com/ytaloaugusto-cmd/painel-frota-wstur/upload/main`
   (ou `/upload/main/data` pra atualizar algo dentro de `data/`).
4. Fazer upload do arquivo (arrastar ou pelo seletor de arquivos).
5. Clicar em "Commit changes" (commit direto na `main`).
6. Verificar o deploy buscando
   `https://raw.githubusercontent.com/ytaloaugusto-cmd/painel-frota-wstur/main/index.html`
   (não passa pelo Cloudflare Access, então dá pra conferir sem login) e
   checando se os trechos esperados estão lá.

**Detalhe de automação via browser (Claude in Chrome):** o botão "Commit
changes" às vezes fica com uma referência de elemento "velha" que aponta pro
lugar errado. Solução que funciona sempre: `scroll_to` no elemento, tirar
`screenshot`, e clicar pelas coordenadas do pixel em vez de pela referência.
Para o upload do arquivo em si, usar o `file_upload` apontando pro input
`type=file` da página (o arquivo precisa estar na pasta de uploads da sessão,
não em outputs — o `file_upload` só aceita a pasta de uploads).

## 4. Automação de KM via n8n (opção escolhida: "opção 3")

O rastreador/planilha de quilometragem é gerenciado pelo n8n. Em vez de
mandar e-mail com planilha pra alguém lançar manualmente, o n8n **escreve
direto no repositório GitHub**, atualizando só o arquivo `data/manutencao.json`.

- Formato do `data/manutencao.json`: array de objetos, um por veículo:
  ```json
  { "placa": "QLA2804", "kmAtual": 413328, "proxRevisao": 430007, "paradoDesde": "2026-09-15" }
  ```
  O campo `paradoDesde` (data ISO `YYYY-MM-DD`) é a fonte **durável** do
  alerta de "veículo parado" — ver seção 5. **Pendência:** o n8n ainda NÃO
  está mandando esse campo hoje (2026-09-15); enquanto isso, o app usa um
  fallback local por navegador. Regra que o n8n deve aplicar para preencher
  `paradoDesde`: no GET que ele já faz do arquivo (pra pegar o `sha`), comparar
  o `kmAtual` anterior de cada placa com o novo — se **mudou** (ou placa nova)
  → `paradoDesde = hoje`; se está **igual** → **manter** o `paradoDesde` que já
  estava no arquivo. Assim a data representa "desde quando o KM está congelado".
- Mecanismo do n8n: GET no arquivo via GitHub Contents API (pra pegar o
  `sha` atual) → monta o novo JSON → PUT com `sha` + conteúdo em base64 +
  mensagem de commit.
- Autenticação: GitHub **fine-grained Personal Access Token** chamado
  `n8n-frota-km`, escopo `Contents: Read and write` (+ `Metadata: Read-only`
  automático), limitado só ao repo `painel-frota-wstur`, expira em
  **2026-09-27** — **precisa ser renovado antes dessa data ou a automação
  para de funcionar**.
- No `index.html`, a função `syncManutencaoData()` faz
  `fetch('data/manutencao.json',{cache:'no-store'})` e mescla os dados no
  objeto `MANUT` em memória (casando por placa com `FROTA`), incluindo o
  campo `paradoDesde` quando presente.
- **Atualização em tela**: não existe polling automático (`setInterval` foi
  removido a pedido do usuário, pra não pesar a aba/rede). A atualização
  acontece só quando a aba volta a ficar visível (`visibilitychange` →
  `refreshManutencaoData()`), cobrindo o caso real de uso (KM é atualizado
  uma vez por dia, de madrugada).
- Status: **automação de KM 100% funcionando e confirmada pelo usuário**. O
  único ponto aberto é o n8n passar a mandar o novo campo `paradoDesde`.

## 5. Alerta de veículo parado (funcionalidade mais recente)

Pedido do usuário: um **único** alerta — se um veículo ficar com o **KM sem
mudar por 7 dias ou mais**, mostrar alerta de **"parado"** (veículo realmente
parado, ou rastreador travado sempre enviando a mesma leitura).

> Histórico: existia também um alerta separado de "sem-comunicação" (placa
> sumindo do feed) e uma fonte `ultimaLeitura` vinda do rastreador. Ambos
> foram **removidos** em 2026-09-15 a pedido do usuário — hoje é só o alerta
> de "parado". Não reintroduzir sem pedir.

Implementado em `index.html` via a função `getKmAlert(f)`, que retorna
`{tipo:'parado', dias:N}` ou `null`. A **data "parado desde"** (dia em que o
KM parou de variar) vem de `kmParadoDesde(f)`, que tem duas fontes, nesta
ordem de preferência:

1. **Campo `paradoDesde` do `data/manutencao.json`** (fonte oficial/durável):
   gravado pelo n8n, vive no arquivo, funciona em qualquer navegador/dispositivo
   e sobrevive a limpar cache. Usado sempre que presente. **Ainda a implementar
   no n8n** (ver seção 4).
2. **Histórico local em `localStorage`** (`KM_TRACK_KEY = 'wsfrota_km_track_v1'`,
   campo `kmSince`) — **fallback**, usado só enquanto o n8n não manda
   `paradoDesde`. **Limitação:** é local ao navegador — some se limpar cache
   ou usar outro dispositivo/navegador, e não tem histórico nos primeiros ~7
   dias após um deploy novo (só passa a alertar depois de acumular leituras).
   Foi justamente essa limitação que motivou criar o campo durável do n8n.

Limiar: **7 dias** (`const DIAS_PARADO_ALERTA = 7`).

Onde o alerta aparece:
- **Inventário** (tabela): selo `⏸ PARADO Xd` (roxo, classe CSS
  `.rev-stopped`) na coluna Status; o `title` mostra "Parado desde DD/MM…".
  Filtro dedicado `activeCat==='parados'` usa `getKmAlert()`.
- **Manutenção Preventiva** (tabela): linha destacada + coluna "Últ.
  atualização KM" mostrando `⏸ parado desde DD/MM (Xd)`; para veículo normal,
  mostra a data da última mudança de KM; sem histórico, `—`.
- **Ficha do veículo (drawer)**: linha "Última atualização de KM" com a data +
  `.obs-box.crit` explicando "Veículo parado: a quilometragem não muda desde
  DD/MM (há X dias)".
- **Indicador no topo** ("Parado 7+ dias", `k-parado`): conta os veículos com
  alerta. Soma também no contador vermelho geral (`alerta-count`), junto com
  as revisões vencidas.
- **`sync-banner`** (topo do Inventário): separado do alerta por veículo —
  aparece só se o **arquivo inteiro** `data/manutencao.json` falhar ao
  carregar (rede fora do ar, JSON quebrado, etc.), via `lastSyncFailed` +
  `updateSyncBanner()`. Mostra a data da última sincronização bem-sucedida
  (guardada em `localStorage['wsfrota_last_sync_ok']`).

Testado (Node.js, mocks de `FROTA`/`MANUT`/histórico local): campo `paradoDesde`
presente com 10 dias → `parado 10` (ignora o fallback local); campo com 3 dias
→ `null`; sem campo, fallback local com 8 dias → `parado 8`; sem campo e sem
histórico → `null`. `fmtDataBR('2026-09-01')` → `01/09/2026`. Todos os casos
bateram como esperado; `node --check` OK nos dois scripts; aba Ocorrências
byte-a-byte idêntica.

Deploy desta funcionalidade: **feito e confirmado no ar** em 2026-09-15
(raw.githubusercontent.com verificado: `DIAS_PARADO_ALERTA = 7`, `paradoDesde`
presente, zero `sem-comunicacao`/`rev-nodata`, KPI/chip "Parado 7+ dias",
`ocRenderCharts` intacto).

## 6. Decisões de design importantes (não reverter sem avisar o usuário)

- **Um único alerta de KM: "parado"** (KM sem variar). O antigo alerta de
  "sem-comunicação" e a fonte `ultimaLeitura` do rastreador foram removidos
  em 2026-09-15 — não reintroduzir sem pedido explícito.
- **Limiar de 7 dias** (`DIAS_PARADO_ALERTA = 7`).
- **Data "parado desde" durável no `data/manutencao.json`** (campo
  `paradoDesde`, gerado pelo n8n): o app prefere ela; sem ela, cai no
  histórico local por navegador (fallback de transição). Motivo: o
  `localStorage` sozinho é por navegador e some com o cache; a data precisa
  ficar registrada "no sistema" (no arquivo) pra ser durável e cross-device.
- **Sem polling/`setInterval`**: atualização só por `visibilitychange`
  (pedido explícito do usuário, pra não pesar a aba/rede).
- **Aba Ocorrências (O.S.) é intocável** em qualquer merge relacionado a
  frota/manutenção/KM — é um módulo separado, sempre confirmar que o script
  dela ficou idêntico depois de qualquer edição.
- **Nunca commitar dados sensíveis** (o token do n8n não deve nunca aparecer
  em nenhum arquivo do repositório — ele fica só configurado dentro do
  próprio n8n; por isso a persistência da data NÃO pode ser feita gravando do
  navegador com token embutido).

## 7. Pendências / próximos passos conhecidos

- [ ] **Renovar o GitHub PAT `n8n-frota-km` antes de 2026-09-27** (expira; se
  não renovar, a automação de KM para).
- [ ] **Implementar no n8n a geração do campo `paradoDesde`** no
  `data/manutencao.json` (regra na seção 4). O `index.html` já lê o campo e
  cai no fallback local enquanto ele não vier — então isso pode ser feito a
  qualquer momento, sem quebrar o painel. Enquanto não for feito, o alerta de
  "parado" depende do histórico local por navegador (menos confiável).
- [ ] Nenhuma pendência bloqueante de código no `index.html` no momento — o
  alerta de "parado" (7 dias, com fonte durável + fallback) está deployado e
  verificado.

## 8. Como retomar o trabalho numa conversa nova

Basta pedir pra Claude ler este arquivo
(`raw.githubusercontent.com/ytaloaugusto-cmd/painel-frota-wstur/main/NOTES.md`)
ou anexar/colar o conteúdo dele no início da conversa. Isso substitui
qualquer necessidade de repetir o histórico completo da conversa anterior.
