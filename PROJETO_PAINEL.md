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
- `manifest.webmanifest` + `icon-192.png` / `icon-512.png` /
  `apple-touch-icon.png` — arquivos do PWA (atalho na tela inicial do celular),
  na raiz do repo. Ver seção 9. Os ícones são a logo WS Receptivo em fundo azul.
- `busca.html`, `menu.html` — páginas auxiliares antigas (não mexidas
  recentemente).

Dentro do `index.html`, dois blocos `<script>`:
1. **Script principal** — contém `const FROTA=[...]` (64 veículos, campos:
   `f,p,m,cat,mot,motorSN,motorObs?,cv,cap,ano,emp,antt,chas,renavam,cil,torq,
   euro,cambio,pbt,oleo,interv`), `const MANUT={...}` (por frota: `plano,ulRev,
   dataRev,kmAtual,proxRev,obs,atualizado?,atualizacao?`), `const CADASTRO={...}` (por
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
  { "placa": "QLA2804", "kmAtual": 414992, "proxRevisao": 430007, "atualizado": "nao", "atualizacao": "2026-09-17" }
  ```
  Além de `kmAtual`/`proxRevisao`, o n8n manda dois campos vindos direto da
  planilha da telemetria (ver seção 5):
  - `atualizado`: `"sim"` / `"nao"` — se o KM do veículo foi atualizado na
    telemetria (coluna "Atualizado" da planilha). O app também aceita
    `true`/`false`. `nao` → dispara o alerta "sem atualização".
  - `atualizacao`: data ISO `YYYY-MM-DD` da última leitura (coluna
    "Atualizacao" da planilha).
  **Pendência:** o n8n ainda precisa passar a mandar esses dois campos
  (2026-09-17). Enquanto não vier, nenhum veículo dispara alerta (seguro).
- Origem: a planilha "WS 2024" da telemetria já tem as colunas PLACA,
  KM/HR ATUAL, Atualizacao e Atualizado — o n8n lê essa planilha, casa por
  placa e escreve no JSON. Obs.: antes da sincronização diária da telemetria,
  a coluna "Atualizado" pode vir toda "NÃO"; depois que ela roda, quem ficar
  "NÃO" é veículo com problema real de atualização.
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
  objeto `MANUT` em memória (casando por placa com `FROTA`), incluindo os
  campos `atualizado` e `atualizacao` quando presentes.
- **Atualização em tela**: não existe polling automático (`setInterval` foi
  removido a pedido do usuário, pra não pesar a aba/rede). A atualização
  acontece só quando a aba volta a ficar visível (`visibilitychange` →
  `refreshManutencaoData()`), cobrindo o caso real de uso (KM é atualizado
  uma vez por dia, de madrugada).
- Status: **automação de KM 100% funcionando e confirmada pelo usuário**. O
  ponto aberto é o n8n passar a mandar os campos `atualizado`/`atualizacao`.

## 5. Alerta de KM "sem atualização" (telemetria)

Um **único** alerta: se o KM de um veículo **não está sendo atualizado na
telemetria**, o painel mostra **"sem atualização"**. A fonte é **direta**: o
flag `atualizado` (sim/nao) que o n8n manda por veículo no
`data/manutencao.json` (coluna "Atualizado" da planilha da telemetria).

> Histórico da evolução (não reverter sem pedir):
> - Antes tinha "sem-comunicação" + "parado" (2 tipos) e a fonte `ultimaLeitura`
>   — removidos em 2026-09-15 (ficou só "parado").
> - "parado" era calculado por **7 dias** de KM congelado, com data durável
>   `paradoDesde` (n8n) + fallback local em `localStorage` — 2026-09-17.
> - Em 2026-09-17 isso foi **substituído** pelo flag direto `atualizado` da
>   telemetria (este documento). Sumiram: `DIAS_PARADO_ALERTA`, `paradoDesde`,
>   `kmParadoDesde`, o histórico local (`kmSince`/`KM_TRACK_KEY`).

Implementado em `index.html`:
- `kmDesatualizado(f)` → `true` quando `MANUT[f].atualizado` é "nao"/"não"/"n"/
  `false`/`0` (tolerante). "sim"/`true`/ausente → `false`.
- `getKmAlert(f)` → `{tipo:'sem-atualizacao'}` quando desatualizado, senão
  `null`. **Se o campo não vier, não dispara alerta** (seguro).
- `ultAtualizacao(f)` → data ISO da coluna "Atualizacao" (exibida como DD/MM).

Visual: ícone estilo **"sem sinal de celular"** (SVG inline `ICON_SEMSINAL`,
mesmo padrão de traço dos outros ícones) + legenda **"sem atualização"**, em
vermelho (`.rev-nodata`, cor `--critical`).

Onde o alerta aparece:
- **Inventário** (tabela): selo `[ícone] SEM ATUALIZAÇÃO` na coluna Status; o
  `title` mostra "última em DD/MM". Filtro dedicado `activeCat==='parados'`
  (chave interna mantida) usa `getKmAlert()`.
- **Manutenção Preventiva** (tabela): linha destacada (fundo vermelho suave) +
  coluna "Últ. atualização KM" mostrando `[ícone] sem atualização (últ. DD/MM)`;
  para veículo ok, mostra a data de atualização; sem data, `—`.
- **Ficha do veículo (drawer)**: linha "Atualização de KM" + `.obs-box.crit`
  explicando "Sem atualização de KM: a telemetria não está atualizando o KM…".
- **Indicador no topo** ("Sem atualização", `k-parado`, cor `--critical`):
  conta os veículos com alerta; soma no contador vermelho geral
  (`alerta-count`) junto com as revisões vencidas.
- **`sync-banner`** (topo do Inventário): separado — só aparece se o arquivo
  inteiro `data/manutencao.json` falhar ao carregar.

Testado (Node.js, mocks): `atualizado` = nao/não/NÃO/false/0 → alerta;
sim/SIM/true/1/ausente → sem alerta. `node --check` OK nos dois scripts; abas
Ocorrências e Oficina byte-a-byte intactas. Deploy confirmado no ar em
2026-09-17.

## 6. Decisões de design importantes (não reverter sem avisar o usuário)

- **Um único alerta de KM: "sem atualização"**, acionado pelo flag
  `atualizado` (sim/nao) que a telemetria manda via n8n. Não há mais cálculo
  por dias, `paradoDesde` nem histórico local — foram removidos. O antigo
  "sem-comunicação" e a fonte `ultimaLeitura` também não existem mais. Não
  reintroduzir nada disso sem pedido explícito.
- **A verdade do alerta vem da telemetria** (arquivo), não de conta no
  navegador. Se o flag não vier para um veículo, ele não dispara alerta.
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
- [ ] **Fazer o n8n mandar os campos `atualizado` (sim/nao) e `atualizacao`
  (data)** no `data/manutencao.json`, lidos da planilha da telemetria (colunas
  "Atualizado" e "Atualizacao"), casando por placa. O `index.html` já lê esses
  campos; enquanto não vierem, nenhum veículo dispara o alerta "sem
  atualização" (comportamento seguro). Ver seções 4 e 5.
- [ ] Se quiser a aba Oficina realmente automática, configurar o n8n pra
  também escrever `data/oficina.json` (mesmo mecanismo do KM — ver seção 7).
  Até lá, ela funciona com o snapshot embutido/seed, e pode ser atualizada
  manualmente editando `data/oficina.json` e fazendo upload pelo GitHub.
- [ ] Nenhuma outra pendência bloqueante de código no momento.

## 9. Atalho no celular (PWA / tela inicial)

O painel é instalável como PWA — vira um ícone na tela inicial e abre em tela
cheia (sem barra do navegador). Implementação:

- `manifest.webmanifest` (raiz): `name`/`short_name` = "Frota WS",
  `display: standalone`, `start_url`/`scope` = "./", `theme_color` `#08203F`
  (tema do painel), `background_color` `#284789` (azul da logo), e os ícones
  `icon-192.png` + `icon-512.png` (`purpose: any maskable`).
- Ícones: a **logo WS Receptivo em fundo azul** (`#284789`), gerada nos
  tamanhos 512/192 e `apple-touch-icon.png` (180). Ficam na raiz do repo.
- No `<head>` do `index.html`: `<link rel="manifest">`, `theme-color`,
  `apple-touch-icon`, e as metas `apple-mobile-web-app-*` (título "Frota WS").
- Para instalar: abrir `https://frotaws.com` no celular (logado) →
  iPhone/Safari: Compartilhar → "Adicionar à Tela de Início"; Android/Chrome:
  menu → "Adicionar à tela inicial". Sem service worker (não é offline; o
  painel precisa de rede pra sincronizar os JSON mesmo).

## 10. Como retomar o trabalho numa conversa nova

Basta pedir pra Claude ler este arquivo
(`raw.githubusercontent.com/ytaloaugusto-cmd/painel-frota-wstur/main/PROJETO_PAINEL.md`)
ou anexar/colar o conteúdo dele no início da conversa. Isso substitui
qualquer necessidade de repetir o histórico completo da conversa anterior.
