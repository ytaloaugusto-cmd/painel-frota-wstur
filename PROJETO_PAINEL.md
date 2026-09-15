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

Última atualização: 2026-09-15

Link direto (sempre a versão mais atual):
`https://raw.githubusercontent.com/ytaloaugusto-cmd/painel-frota-wstur/main/PROJETO_PAINEL.md`

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
   dataRev,kmAtual,proxRev,obs,ultimaLeitura`), `const CADASTRO={...}` (por
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

**Detalhe de automação via browser (Claude in Chrome):** o botão "Commit
changes" às vezes fica com uma referência de elemento "velha" que aponta pro
lugar errado. Solução que funciona sempre: `scroll_to` no elemento, tirar
`screenshot`, e clicar pelas coordenadas do pixel em vez de pela referência.

## 4. Automação de KM via n8n (opção escolhida: "opção 3")

O rastreador/planilha de quilometragem é gerenciado pelo n8n. Em vez de
mandar e-mail com planilha pra alguém lançar manualmente, o n8n **escreve
direto no repositório GitHub**, atualizando só o arquivo `data/manutencao.json`.

- Formato do `data/manutencao.json`: array de objetos, um por veículo:
  ```json
  { "placa": "QLA2804", "kmAtual": 413328, "proxRevisao": 430007 }
  ```
  (campo opcional `ultimaLeitura` também é suportado — ver seção 5 — mas o
  n8n **ainda não está mandando esse campo** hoje, 2026-09-14).
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
  objeto `MANUT` em memória (casando por placa com `FROTA`).
- **Atualização em tela**: não existe mais polling automático
  (`setInterval` foi removido a pedido do usuário, pra não pesar a aba/rede).
  A atualização acontece só quando a aba volta a ficar visível
  (`visibilitychange` → `refreshManutencaoData()`), cobrindo o caso real de
  uso (KM é atualizado uma vez por dia, de madrugada).
- Status: **automação 100% funcionando e confirmada pelo usuário** desde
  antes desta atualização.

## 5. Alertas de veículo parado / sem comunicação (funcionalidade mais recente)

Pedido do usuário: se um veículo ficar **5 dias ou mais** sem atualização de
KM, mostrar alerta — mas distinguindo dois motivos diferentes:

1. **"sem-comunicacao"** — o veículo simplesmente **para de aparecer** no
   `data/manutencao.json` (falha do rastreador ou da automação n8n).
2. **"parado"** — o veículo continua aparecendo normalmente, mas o **valor
   de KM não muda** por 5+ dias (veículo realmente parado, ou rastreador
   travado sempre mandando a mesma leitura).

Implementado em `index.html` via a função `getKmAlert(f)`, que retorna
`{tipo:'sem-comunicacao'|'parado', dias:N}` ou `null`:

- Se `MANUT[f].ultimaLeitura` existir (fonte mais confiável, viria do n8n/
  rastreador, funciona entre navegadores/dispositivos), usa ela — só detecta
  o tipo `"parado"` nesse caminho.
- Se não existir (caso atual, já que o n8n ainda não manda esse campo), usa
  um **histórico guardado no `localStorage` do navegador**
  (`KM_TRACK_KEY = 'wsfrota_km_track_v1'`), atualizado a cada sync bem
  sucedido: `lastSeen` (última vez que a placa apareceu no feed — se sumir,
  fica congelado, detectando "sem-comunicacao") e `kmSince` (data em que o
  KM mudou pela última vez — se o valor não mudar, fica congelado,
  detectando "parado"). **Limitação conhecida:** esse histórico é local ao
  navegador — some se limpar cache ou usar outro dispositivo/navegador, e
  não tem histórico nenhum nos primeiros dias após um deploy novo (não vai
  mostrar alerta até acumular ~5 dias de leituras).

Onde os alertas aparecem:
- **Inventário** (tabela): selo `📡 SEM COMUNICAÇÃO Xd` (vermelho, classe CSS
  `.rev-nodata`) ou `⏸ PARADO Xd` (roxo, classe `.rev-stopped`), coluna
  Status. Filtro dedicado `activeCat==='parados'` usa `getKmAlert()`.
- **Manutenção Preventiva** (tabela): linha destacada + coluna "Últ.
  atualização KM" com o mesmo selo/estilo.
- **Ficha do veículo (drawer)**: linha com data/motivo detalhado +
  `.obs-box.crit` explicando qual dos dois problemas é.
- **Indicador no topo** ("Parado 5+ dias", `k-parado`): conta os dois tipos
  juntos. Soma também no contador vermelho geral (`alerta-count`), junto
  com as revisões vencidas.
- **`sync-banner`** (topo do Inventário): separado dos alertas por veículo —
  aparece só se o **arquivo inteiro** `data/manutencao.json` falhar ao
  carregar (rede fora do ar, JSON quebrado, etc.), via `lastSyncFailed` +
  `updateSyncBanner()`. Mostra a data da última sincronização bem-sucedida
  (guardada em `localStorage['wsfrota_last_sync_ok']`).

**Quando o n8n futuramente passar a mandar o campo `ultimaLeitura`**, o
sistema automaticamente vai preferir essa fonte (mais confiável, funciona
em qualquer dispositivo) em vez do histórico local — não precisa mexer em
mais nada no `index.html` pra isso, só o n8n começar a mandar o campo.

Testado (Node.js, simulação com `FROTA`/`MANUT`/histórico local mockados):
leitura antiga via `ultimaLeitura` → `parado`; leitura recente → `null`;
veículo sumindo do feed → `sem-comunicacao`; KM congelado com veículo
presente → `parado`; tudo normal ou sem histórico ainda → `null`. Todos os
casos bateram como esperado.

Deploy desta funcionalidade: **feito e confirmado no ar** em 2026-09-14
(raw.githubusercontent.com verificado, sem sobras de código antigo
`diasParado`/`isInoperante`, aba de Ocorrências intacta).

## 6. Decisões de design importantes (não reverter sem avisar o usuário)

- **Sem polling/`setInterval`**: atualização só por `visibilitychange`
  (pedido explícito do usuário, pra não pesar a aba/rede — KM só muda 1x/dia
  de madrugada mesmo).
- **Alertas de KM parado usam 5 dias** como limiar (`DIAS_PARADO_ALERTA`).
- **Aba Ocorrências (O.S.) é intocável** em qualquer merge relacionado a
  frota/manutenção/KM — é um módulo separado, sempre confirmar que o script
  dela ficou idêntico depois de qualquer edição.
- **Nunca commitar dados sensíveis** (o token do n8n não deve nunca aparecer
  em nenhum arquivo do repositório — ele fica só configurado dentro do
  próprio n8n).

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
  - **Inventário**: chip de filtro "🔧 Na oficina" + selo `🔧 oficina` na
    coluna Status da tabela.
  - **Ficha do veículo (drawer)**: seção "Oficina — corretiva" com situação,
    entrada, categoria e problema relatado, quando o veículo está na lista.
- KPIs da aba: total inoperante, crônicos (+30 dias), entradas nos últimos 7
  dias, maior categoria de problema, disponibilidade da frota (%). Gráfico
  de barras simples com o histórico diário (`historico[]`).
- CSS novo: bloco `/* ABA OFICINA / CORRETIVA */` no `<style>`, com as
  classes `sync-bar`, `sync-dot` (+ `.live`/`.local`/`.err`), `trend-*`,
  `cat-badge`/`c-*`, `dias-tag`/`dias-*`, `status-pill`, `problema-cell`.

## 8. Pendências / próximos passos conhecidos

- [ ] Renovar o GitHub PAT `n8n-frota-km` antes de **2026-09-27** (expira).
- [ ] Se possível, pedir pro n8n passar a mandar o campo `ultimaLeitura` no
  `data/manutencao.json` — melhora a confiabilidade do alerta de "parado"
  (deixa de depender de histórico local por navegador).
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
