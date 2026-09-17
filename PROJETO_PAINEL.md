# Painel da Frota — W2S Locação / S2 Turismo — Documentação técnica

> **INSTRUÇÃO PARA A CLAUDE (leia isto primeiro, em qualquer conversa sobre
> este projeto):**
>
> 1. Este arquivo é a **fonte de verdade** do projeto. No início de qualquer
>    conversa nova sobre o painel, leia este arquivo direto do link abaixo
>    antes de fazer qualquer alteração — não confie em resumo de conversa
>    antiga nem em memória própria, que podem perder detalhes ou inventar
>    informação:
>    `https://raw.githubusercontent.com/ytaloaugusto-cmd/painel-frota-wstur/main/PROJETO_PAINEL.md`
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
> 4. **Só existe UM arquivo de documentação: este (`PROJETO_PAINEL.md`).** O
>    antigo `NOTES.md` foi removido do repositório em 2026-09-17 por estar
>    desatualizado e causar confusão. Não recriar.

Última atualização: 2026-09-17

Link direto (sempre a versão mais atual):
`https://raw.githubusercontent.com/ytaloaugusto-cmd/painel-frota-wstur/main/PROJETO_PAINEL.md`

---

## 1. O que é o projeto

Painel HTML único (`index.html`) de gestão da frota da W2S Locação / S2
Turismo (Maceió/AL): inventário de veículos, manutenção preventiva (KM x
revisão), oficina (corretiva), gráficos, referência técnica dos motores e
ocorrências (O.S.).

- Repositório GitHub: `ytaloaugusto-cmd/painel-frota-wstur`, branch `main`.
- Publicado via **GitHub Pages**.
- Domínio próprio: **https://frotaws.com** (DNS apontando pro GitHub Pages).
- Protegido por **Cloudflare Access** (login obrigatório antes de ver o site).
- Todo o app é client-side: um `index.html` só, sem backend próprio.

## 2. Estrutura de arquivos no repositório

- `index.html` — o painel inteiro (HTML + CSS + JS em 2 blocos `<script>`).
- `data/manutencao.json` — pequeno arquivo JSON, atualizado automaticamente
  pelo n8n, com KM atual, próxima revisão e a data "parado desde" de cada
  veículo (ver seção 4).
- `data/oficina.json` — pequeno arquivo JSON (mesmo padrão do anterior) com
  os veículos inoperantes na oficina/corretiva (ver seção 7). Hoje só tem o
  snapshot inicial (seed) — ainda não é atualizado por automação nenhuma.
- `CNAME` — domínio customizado do GitHub Pages (`frotaws.com`).
- `busca.html`, `menu.html` — páginas auxiliares antigas (não mexidas
  recentemente).

Dentro do `index.html`, dois blocos `<script>`:
1. **Script principal** — contém `const FROTA=[...]` (64 veículos, campos:
   `f,p,m,cat,mot,motorSN,motorObs?,cv,cap,ano,emp,antt,chas,renavam,cil,torq,
   euro,cambio,pbt,oleo,interv`), `const MANUT={...}` (por frota: `plano,ulRev,
   dataRev,kmAtual,proxRev,obs,paradoDesde?`), `const CADASTRO={...}` (por
   frota: `plot,cor,prop,crv,fin`), `let MANUT_CORRETIVA={...}` (dados da aba
   Oficina — ver seção 7), e todas as funções de render/lógica.
2. **Script da aba Ocorrências (O.S.)** — uma IIFE separada com
   `const fleet=[...]` e `window.ocRenderCharts`. **Nunca deve ser tocado**
   em atualizações relacionadas a KM/manutenção — é um módulo independente.
   (Não confundir com a aba "Oficina", que é outra coisa — ver seção 7.)

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

**Detalhes de automação via browser (Claude in Chrome):**
- Upload: usar `file_upload` apontando pro input `type=file` da página. O
  arquivo precisa estar na pasta de **uploads** da sessão (`file_upload` não
  aceita a pasta de outputs).
- Botão "Commit changes": às vezes fica com referência de elemento "velha".
  Solução que funciona sempre: `scroll_to`/`screenshot` e clicar pelas
  **coordenadas do pixel** em vez de pela referência.
- Campo de mensagem de commit: clicar bem no centro do campo (borda fica azul)
  antes de digitar. Se digitar sem foco, o texto vaza pra busca global do
  GitHub / abre o painel do Copilot — nesse caso apertar `Escape`, reclicar no
  campo e digitar de novo.

## 4. Automação de KM via n8n (opção escolhida: "opção 3")

O rastreador/planilha de quilometragem é gerenciado pelo n8n. Em vez de
mandar e-mail com planilha pra alguém lançar manualmente, o n8n **escreve
direto no repositório GitHub**, atualizando só o arquivo `data/manutencao.json`.

- Formato do `data/manutencao.json`: array de objetos, um por veículo:
  ```json
  { "placa": "QLA2804", "kmAtual": 413328, "proxRevisao": 430007, "paradoDesde": "2026-09-17" }
  ```
  O campo `paradoDesde` (data ISO `YYYY-MM-DD`) é a fonte **durável** do
  alerta de "veículo parado" — ver seção 5. **Pendência:** o n8n ainda NÃO
  está mandando esse campo (2026-09-17); enquanto isso, o app usa um fallback
  local por navegador. Regra que o n8n deve aplicar para preencher
  `paradoDesde`: no GET que ele já faz do arquivo (pra pegar o `sha`), comparar
  o `kmAtual` anterior de cada placa com o novo — se **mudou** (ou placa nova)
  → `paradoDesde = hoje`; se está **igual** → **manter** o `paradoDesde` que já
  estava no arquivo. Assim a data representa "desde quando o KM está congelado".
  (Data no fuso America/Fortaleza: `new Date().toLocaleDateString('en-CA',{timeZone:'America/Fortaleza'})`.)
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

## 5. Alerta de veículo parado

Um **único** alerta: se um veículo ficar com o **KM sem mudar por 7 dias ou
mais**, mostra alerta de **"parado"** (veículo realmente parado, ou rastreador
travado sempre enviando a mesma leitura).

> Histórico: antes existia também um alerta de "sem-comunicação" (placa sumindo
> do feed) e uma fonte `ultimaLeitura` do rastreador. Ambos foram **removidos**
> em 2026-09-15 a pedido do usuário — hoje é só o alerta de "parado". Não
> reintroduzir sem pedir. (Obs.: em 2026-09-15 houve uma colisão de edições
> paralelas — o upload que adicionou a aba Oficina foi montado sobre uma cópia
> antiga e reverteu esta mudança; ela foi **reaplicada sobre a versão com
> Oficina em 2026-09-17**.)

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
   dias após um deploy novo. Foi essa limitação que motivou o campo durável.

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
  `updateSyncBanner()`.

Testado (Node.js, mocks): `paradoDesde` com 10 dias → `parado 10` (ignora
fallback local); com 3 dias → `null`; sem campo, fallback local com 8 dias →
`parado 8`; sem campo e sem histórico → `null`. `node --check` OK nos dois
scripts; aba Ocorrências byte-a-byte idêntica. Deploy confirmado no ar em
2026-09-17.

## 6. Decisões de design importantes (não reverter sem avisar o usuário)

- **Um único alerta de KM: "parado"** (KM sem variar). O antigo alerta de
  "sem-comunicação" e a fonte `ultimaLeitura` do rastreador foram removidos —
  não reintroduzir sem pedido explícito.
- **Limiar de 7 dias** (`DIAS_PARADO_ALERTA = 7`).
- **Data "parado desde" durável no `data/manutencao.json`** (campo
  `paradoDesde`, gerado pelo n8n): o app prefere ela; sem ela, cai no
  histórico local por navegador (fallback de transição). Motivo: o
  `localStorage` sozinho é por navegador e some com o cache; a data precisa
  ficar registrada "no sistema" (no arquivo) pra ser durável e cross-device.
- **Sem polling/`setInterval`**: atualização só por `visibilitychange`.
- **Ícones/selos em SVG inline, não emoji**, para renderizar igual em qualquer
  tela (ex.: o selo "oficina" no Inventário usa um SVG pequeno + legenda, não
  o emoji 🔧, que variava de tamanho por dispositivo).
- **Aba Ocorrências (O.S.) é intocável** em qualquer merge relacionado a
  frota/manutenção/KM — é um módulo separado, sempre confirmar que o script
  dela ficou idêntico depois de qualquer edição.
- **Nunca commitar dados sensíveis** (o token do n8n não deve nunca aparecer
  em nenhum arquivo do repositório — ele fica só configurado dentro do
  próprio n8n; por isso a persistência da data NÃO pode ser feita gravando do
  navegador com token embutido).

## 7. Aba "Oficina" (manutenção corretiva) — adicionada em 2026-09-15

Nova aba (`tab-oficina`, botão "Oficina" no menu, entre "Manutenção
Preventiva" e "Gráficos") pra acompanhar veículos **inoperantes na
oficina** (corretiva) — diferente da aba "Manutenção Preventiva" (que é sobre
revisão por KM) e diferente da aba "Ocorrências (O.S.)" (que é outro módulo,
independente, vindo de antes).

- Fonte de origem: o "Relatório de manutenção WS 2026" da oficina.
- Dados guardados em `let MANUT_CORRETIVA={atualizadoEm, fonte, historico[],
  veiculos[]}` no `index.html`, com um **snapshot embutido** (14/09/2026, 10
  veículos) usado como fallback.
- Sincronização: `syncCorretiva()` busca `data/oficina.json` no mesmo padrão
  do `syncManutencaoData()` (fetch same-origin, `cache:'no-store'`), chamada
  automaticamente junto com o sync de KM (`loadManutencaoData()` /
  `refreshManutencaoData()`) e também pelo botão "↻ Sincronizar agora" na
  aba. Se o arquivo não existir ainda (404) ou falhar, cai no snapshot
  embutido — mostra isso na barra de status da aba (bolinha
  cinza/verde/amarela/vermelha: verificando / sincronizado / modo local /
  erro).
- **Ainda não tem automação real**: hoje `data/oficina.json` só tem o mesmo
  snapshot inicial, escrito manualmente (seed). **Se quiser automatizar via
  n8n**, é só reaproveitar exatamente o mesmo mecanismo já usado pro KM (GET
  sha → monta JSON → PUT), mas escrevendo em `data/oficina.json` em vez de
  `data/manutencao.json`. Schema documentado em comentário no próprio
  `index.html`, logo antes de `MANUT_CORRETIVA`.
- Cada veículo tem: `frota` (casa com `FROTA[].f` via `ofFrotaKey()`, que
  tenta o código puro e depois com sufixo `A`/`B`), `apelido`, `entrada`
  (data ISO), `cat` (Motor | Elétrica | Ar-condicionado | Pneus/Alinhamento |
  Lanternagem/Pintura | Freios | Outros), `status`, `problema`.
- Aparece em mais lugares além da própria aba:
  - **Inventário**: chip de filtro "🔧 Na oficina" + selo "oficina" na coluna
    Status da tabela. O selo usa um **SVG inline pequeno (10px) + legenda
    "oficina"** (classe `cat-badge c-motor`), não o emoji 🔧 — mudança de
    2026-09-17 pra ficar legível/consistente em qualquer tela.
  - **Ficha do veículo (drawer)**: seção "Oficina — corretiva" com situação,
    entrada, categoria e problema relatado, quando o veículo está na lista.
- KPIs da aba: total inoperante, crônicos (+30 dias), entradas nos últimos 7
  dias, maior categoria de problema, disponibilidade da frota (%). Gráfico
  de barras simples com o histórico diário (`historico[]`).
- CSS novo: bloco `/* ABA OFICINA / CORRETIVA */` no `<style>`, com as
  classes `sync-bar`, `sync-dot` (+ `.live`/`.local`/`.err`), `trend-*`,
  `cat-badge`/`c-*`, `dias-tag`/`dias-*`, `status-pill`, `problema-cell`.

## 8. Pendências / próximos passos conhecidos

- [ ] **Renovar o GitHub PAT `n8n-frota-km` antes de 2026-09-27** (expira; se
  não renovar, a automação de KM para).
- [ ] **Implementar no n8n a geração do campo `paradoDesde`** no
  `data/manutencao.json` (regra na seção 4). O `index.html` já lê o campo e
  cai no fallback local enquanto ele não vier — então isso pode ser feito a
  qualquer momento, sem quebrar o painel. Enquanto não for feito, o alerta de
  "parado" depende do histórico local por navegador (menos confiável).
- [ ] Se quiser a aba Oficina realmente automática, configurar o n8n pra
  também escrever `data/oficina.json` (mesmo mecanismo do KM — ver seção 7).
  Até lá, ela funciona com o snapshot embutido/seed, e pode ser atualizada
  manualmente editando `data/oficina.json` e fazendo upload pelo GitHub.
- [ ] Nenhuma outra pendência bloqueante de código no momento.

## 9. Como retomar o trabalho numa conversa nova

Basta pedir pra Claude ler este arquivo
(`raw.githubusercontent.com/ytaloaugusto-cmd/painel-frota-wstur/main/PROJETO_PAINEL.md`)
ou anexar/colar o conteúdo dele no início da conversa. Isso substitui
qualquer necessidade de repetir o histórico completo da conversa anterior.
