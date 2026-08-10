# Changelog

Todas as mudanças notáveis deste projeto serão documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/).

> Nota: o histórico de commits anterior a este arquivo não tinha mensagens descritivas
> ("Update index.html" repetido), então as entradas abaixo anteriores a 2026-07 foram
> reconstruídas a partir do estado atual do código, não do histórico do Git.

## [Unreleased]

### Adicionado
- **Menu lateral (drawer) substituindo a barra de abas, 2026-08-10.** A navegação passou a ser
  um único drawer que desliza da esquerda, aberto por um chip escuro no canto superior esquerdo
  da topbar. **Feito em duas etapas, com validação do Felipe no celular entre elas:** a etapa 1
  construiu o drawer convivendo com as abas antigas; a etapa 2 removeu as abas.
  - **Motivo:** a `.tb-nav` já estava no limite. Com 7 abas, o mobile precisava quebrar a topbar
    em duas linhas e dar scroll horizontal ao menu — e cada aba nova piorava isso. O drawer tem
    espaço vertical de sobra e não muda de forma conforme o app cresce.
  - **Um componente só, desktop E mobile.** Duas navegações (abas no desktop, drawer no celular)
    seriam duas fontes de verdade, e toda página nova teria que ser cadastrada nos dois lugares.
  - **Três grupos com rótulo visual, sem accordion:** *Agenda* (Agenda, Calendário, Rodízio),
    *Operação* (Treinos, Planejamento) e *Gestão* (Relatórios, Config.). Com 7 itens cabe tudo
    visível ao mesmo tempo; accordion só acrescentaria um clique e esconderia opção.
  - **Grupo sem nenhum item visível para o perfil não é renderizado — o título junto.** Não é
    detalhe teórico: "Gestão" só tem itens de gestor, então some inteiro para professor e
    recepção. Título de grupo sozinho, sem item embaixo, é bug visual.
  - Registry `NAV_GRUPOS` como **fonte única** dos itens, mesmo precedente do registry
    `RELATORIOS`: página nova é uma entrada ali, sem tocar em CSS nem no `goPage()`. O `soGestor`
    de Relatórios e Config. substituiu os dois `style.display` que o `enterApp()` fazia nos
    botões de aba. **Continua sendo conveniência de interface, não segurança** — igual ao
    `podeCadastrarTreino()` e ao `podeEditarPlanejamento()`; quem garante é o RLS.
  - Fecha por backdrop, ESC, botão ✕ e ao navegar. Trava o scroll com `body.drawer-open`, que age
    na `.app-content` — **o `body` já é `overflow:hidden`, quem rola de verdade é ela**. A posição
    do scroll é salva e restaurada em JS, porque alternar `overflow` pode zerá-la.
- **Nome da página na topbar (etapa 2, 2026-08-10),** ocupando o espaço que era da barra de abas.
  É orientação visual: não é clicável, não tem hover e usa o cinza discreto que as abas tinham
  quando **não** estavam ativas — de propósito, para não competir com o item ativo do drawer
  (branco sólido). O texto sai do `rotuloPagina()`, que lê o `NAV_GRUPOS`: **nome de página não
  pode existir em dois lugares.**
- **Login real com usuário + senha via Supabase Auth (Fase 1 de 5), 2026-07-31.** Substitui a
  seleção de perfil + PIN. O usuário digita só o login (ex.: `FelipePolenta`) e nunca vê email:
  o app monta `usuario.trim().toLowerCase() + '@equipe.evolucaofit.app'` e chama
  `signInWithPassword`. O `toLowerCase()` é obrigatório — o Supabase guarda o email em minúsculas.
  Confirmação por email desligada e sem SMTP configurado (ver "Segurança" abaixo).
- Dependência nova: `supabase-js` **fixado em 2.111.0** via jsDelivr, com atributo `integrity`
  (SRI). Versão fixa porque num app sem build uma tag flutuante (`@2`) mudaria o app sozinho, sem
  ninguém tocar em nada; o SRI faz o navegador recusar executar se o CDN entregar conteúdo
  diferente — relevante numa tela onde se digita senha. **Ao trocar a versão, recalcular o hash.**
  Custo assumido: ~210 KB no primeiro carregamento, e uma dependência externa nova. Por isso o
  `sbClient` nulo (CDN fora do ar/bloqueado ou SRI falhando) dispara `showLibError()` com banner
  vermelho e texto próprio — mesmo princípio da correção de 2026-07-03, nada falha em silêncio.
- Sessão persistente: `persistSession` + `autoRefreshToken` (storageKey `agenda-evolucao-auth`).
  No boot o `getSession()` roda **antes** de qualquer chamada de rede — se rodasse depois do
  `sbGet('config')`, quem já está logado veria a tela de login piscar. Botão "Sair" agora faz
  `signOut()` de verdade e limpa os campos.
- Roteamento de perfil pela tabela `equipe`: `select perfil, nome ... where auth_user_id = <uid>`,
  feito **autenticado** via supabase-js (decisão consciente: o atalho seria ler com a anon key,
  mas isso só empurraria o problema pra Fase 2, que exige leitura autenticada). Sem linha vinculada
  ou com `perfil` inválido, o app faz `signOut()` e mostra erro claro — não existe entrar sem
  perfil. O nome de quem está logado passou a aparecer na topbar, ao lado do badge de papel.
- Colunas novas na `equipe`: `auth_user_id uuid unique references auth.users(id) on delete set null`
  e `perfil text check (perfil in ('professor','recepcao','gestor'))`. `perfil` aceita NULL de
  propósito (quem não tem perfil não loga; um `default` daria acesso por acidente). `perfil` é
  acesso e **não se confunde com `cargo`**, que é função no rodízio — o Felipe tem
  `cargo = Recepção` e `perfil = gestor`. Backfill: professores → `professor`, recepção →
  `recepcao`, e a linha do Felipe vinculada ao usuário de teste com `perfil = gestor`.
- **Trocar senha dentro do app** (`updateUser({ password })`), com sucesso e erro visíveis. Um
  modal único com **duas portas de entrada**: botão "🔑 Senha" na topbar, disponível para **todos
  os perfis**, e card "Minha senha" na Config (só Gestor), no lugar exato onde ficava o PIN. As
  duas portas são deliberadas: a Config só é visível pro Gestor, então um acesso único ali
  deixaria professor e recepção sem conseguir trocar a própria senha — que é justamente o ponto.
  O modal segue o padrão **branco** dos demais (Novo Evento / Nova Escala) e não o vidro intenso,
  porque aquilo era característica do overlay de PIN, que deixou de existir. Mínimo de 6
  caracteres, alinhado ao mínimo padrão do Supabase (regra mais rígida só no cliente confundiria
  o usuário quando o servidor discordasse). Erros do Supabase chegam em inglês e passam por
  `traduzErroSenha()`; o que fica fora da lista vira mensagem genérica na tela e erro cru no
  console. Validado pelo Felipe ponta a ponta em 2026-08-01: trocar pela topbar, sair e entrar
  com a senha nova.
- Tela de login redesenhada no padrão liquid glass, **sem inventar valor nenhum**: card de vidro
  com a mesma base do `#page-calendario .cal-wrap` (blur 3px, saturate 180%, raio 18px, linha de
  highlight no topo), inputs **SÓLIDOS** pela mesma regra da Config (onde se digita não leva
  vidro — a mesma decisão que mantinha o keypad do PIN opaco), caixa de erro com os valores do
  `.cfg-danger`, e o `@keyframes shake` do PIN reaproveitado para o card tremer na senha errada.
- `CLAUDE.md` com contexto do projeto para desenvolvimento assistido por IA.
- `README.md`, `.env.example` e `docs/schema.sql` documentando o projeto e o banco de dados.
- `.github/workflows/ping-supabase.yml`: workflow que pinga a tabela `config` via REST API
  2x/semana (segunda e quinta) para evitar a pausa automática do Supabase free tier por
  inatividade. Antes disso, a documentação (CLAUDE.md/CHANGELOG) já citava essa mitigação
  como se existisse, mas o workflow nunca tinha sido criado de fato — corrigido agora.

- **Tela de Treinos e tabela `montagem_treinos`, 2026-08-02.** Fila de montagem de treino:
  recepção/gestor cadastra o pedido, professor monta e confirma pelo botão "Marcar como montado".
  Colunas: `id` (uuid, PK), `aluno_nome` (text), `avaliador_id` (uuid, NOT NULL),
  `professor_id` (uuid, NOT NULL), `data_solicitacao` (date, NOT NULL),
  `data_inicio_desejada` (date), `status` (text: `pendente`/`montado`/`cancelado`),
  `montado_por` (uuid), `data_montagem` (timestamptz), `observacoes` (text),
  `created_at` (timestamptz).
- **Esta tela quebra o padrão das vizinhas de propósito:** usa `sbClient.from()` (autenticado) em
  vez dos helpers `sbGet`/`sbPost`/`sbPatch`, porque a `montagem_treinos` só concedeu grant ao role
  `authenticated` — com a anon key dos helpers a resposta é `401`/`42501`. Não copiar o padrão
  antigo aqui.
- O UPDATE de confirmação encadeia **`.select('id')`** (= `Prefer: return=representation`). Sem
  isso o PostgREST devolve `204` vazio e um UPDATE barrado pelo RLS pareceria sucesso; com isso,
  o bloqueio chega como `200` + zero linhas e vira mensagem na tela. Mesma família da pegadinha
  já registrada para o SELECT.

- **Aba Planejamento (Kanban) — Fase 4, 2026-08-06.** Quadros com listas coloridas e cartões, para
  o gestor organizar trabalho em andamento e liberar a visão dele para quem interessa. **Todos os
  perfis veem a aba**; professor e recepção entram em modo somente leitura (nenhum botão de ação é
  renderizado para eles) e enxergam apenas os cartões que o gestor liberou.
- **Quatro tabelas novas, criadas direto no painel do Supabase antes desta implementação** — o app
  só as consome, não cria nem altera schema:
  - `quadros` — `id` uuid PK, `tema` text, `nome` text, `criado_por` uuid→`equipe.id`,
    `criado_em` timestamptz, `ativo` boolean default true.
  - `colunas` — `id` uuid PK, `quadro_id` uuid→`quadros.id` (on delete cascade), `nome` text,
    `cor` text nullable, `ordem` int.
  - `cards` — `id` uuid PK, `coluna_id` uuid→`colunas.id` (on delete cascade), `titulo` text,
    `descricao` text nullable, `criado_por` uuid→`equipe.id`, `ordem` int, `criado_em`,
    `atualizado_em`.
  - `card_visibilidade` — `id` uuid PK, `card_id` uuid→`cards.id` (on delete cascade),
    `perfil` text nullable, `equipe_id` uuid nullable.
- **Dois CHECKs que o código respeita, sob pena de erro do banco:** `colunas.cor` só aceita
  `#3B82F6` · `#8B5CF6` · `#EAB308` · `#22C55E` · `#EF4444` · `#64748B` ou `NULL` (por isso a
  constante `PLAN_CORES` no JS é a única fonte dessas cores); e `card_visibilidade` exige
  **exatamente um** de `perfil`/`equipe_id` preenchido — nunca os dois, nunca nenhum, o que
  obriga o `sincronizarVisibilidade()` a montar as linhas em dois `map` separados.
- **Modelo de visibilidade:** o gestor marca um ou mais perfis e/ou uma ou mais pessoas.
  **Sem nenhuma seleção o cartão fica visível só para gestores** — isso não é convenção da
  interface, é como a policy `cards_select` se comporta; o formulário diz isso em texto fixo.
- **Consumo 100% autenticado via `sbClient.from()`**, como na aba Treinos e ao contrário das telas
  antigas. As 7 policies são para o role `authenticated`; com a anon key dos helpers
  `sbGet`/`sbPost` a resposta seria zero linhas em silêncio.
- Toda escrita encadeia **`.select('id')`**, pelo mesmo motivo já registrado no `marcarMontado()`:
  RLS que barra devolve `200` com zero linhas, não erro. Vale inclusive para o DELETE de
  `card_visibilidade`, onde "barrado" e "não havia nada" são indistinguíveis — ali o código compara
  com a contagem que já tinha em memória antes de decidir se deu certo.
- **Criar quadro** pede tema, nome e um dos 3 modelos (Simples · Com espera · Em branco). O modelo é
  só uma lista fixa no JS que popula a tabela `colunas` na criação: depois disso o quadro não fica
  preso a ele, e listas podem ser criadas, renomeadas, reordenadas, recoloridas e apagadas. As
  listas entram num segundo passo — se falharem, o quadro já existe e o app **avisa que ele nasceu
  vazio** em vez de dar sucesso completo.
- **Apagar quadro exige digitar o nome exato**, não `confirm()` simples: o DELETE dispara CASCADE em
  listas, cartões e visibilidades, e não existe lixeira. O modal mostra antes quantas listas e
  quantos cartões vão junto.
- **Mover cartão:** drag-and-drop nativo em JS puro no desktop (nenhuma dependência nova) e menu
  `⋮` com "Mover para →" no mobile, onde arrastar brigaria com o scroll da tela. A detecção é por
  `matchMedia('(pointer:coarse)')` + largura, **nunca por user-agent**, e um listener de `resize`
  repinta o quadro quando o modo realmente vira. Os dois caminhos passam pela mesma função
  `moverCardPara()`, que reordena em memória, pinta na hora e relê do banco se a gravação falhar —
  a tela nunca fica mostrando uma posição que o banco recusou.
- **Regra visual — os cartões são SÓLIDOS escuros (`var(--g1)`), nunca vidro e nunca na cor da
  lista;** só levam uma faixa de 3px no topo na cor dela. É a mesma regra já aplicada aos inputs do
  Login e da Config e ao antigo keypad do PIN: sobre o fundo laranja, onde se lê o conteúdo não
  leva vidro. O vidro fica nas colunas, que reutilizam os tokens de `#page-treinos` sem inventar
  valor novo (blur 3px + saturate 180%, borda `1px rgba(255,255,255,0.4)`, raio 16px, linha de
  highlight no topo). O cabeçalho de cada lista é uma faixa **sólida opaca** na cor, com o texto no
  tom escuro da mesma família.
- Cartões na **última coluna do quadro** ficam com `opacity:0.6` (o mesmo valor do
  `.treino-card.st-cancelado`). **Decisão consciente:** a tabela `cards` não tem campo de status e a
  Fase 4 não mexeu em schema, então "concluído" é derivado da posição, não de um dado.
- O **cadeado** de visibilidade no cartão só aparece para o gestor, porque só ele lê
  `card_visibilidade` (a policy `vis_all` é `ALL using eh_gestor()`). Para os outros perfis não faz
  falta: todo cartão que eles enxergam é, por definição, um cartão liberado para eles. O app nem
  consulta essa tabela quando não é gestor — consultar traria zero linhas em silêncio.

- **Planejamento: prazo por quadro (início e fim previsto), 2026-08-07.** Primeira parte da
  atualização do Kanban. O gestor define um período opcional para o quadro e a lista de quadros
  passa a mostrar o andamento. **Etapa 100% de app/UI: nenhuma policy, função `security definer`
  ou grant foi criada ou alterada.** A etapa seguinte (quadro privado) é commit separado e aí sim
  mexe em RLS.
- **Duas colunas novas na `quadros`, criadas direto no painel antes desta implementação** — o app
  só as consome: `data_inicio` (date, nullable) e `data_fim_prevista` (date, nullable). Nullable de
  propósito: quadro sem prazo é o caso mais comum e continua funcionando exatamente como antes,
  sem nenhuma mudança visual.
- **Os campos entraram no modal que já existia, em vez de um "Editar quadro" novo.** O
  `#quadro-modal` da Fase 4 já servia criação e edição (título dinâmico, `#quadro-tpl-wrap` que se
  esconde na edição, UPDATE com `.select('id')`); o que faltava era só a **porta de entrada pela
  lista** — até aqui só dava pra editar entrando no quadro. Criar um segundo modal para a mesma
  tabela produziria duas fontes de verdade, e uma delas envelheceria esquecida.
- Botão `✏️` no card da lista, **só para quem `podeEditarPlanejamento()`** — conveniência de
  interface, não segurança, igual ao resto da aba: quem garante é a policy `quadros_write`
  (`ALL using eh_gestor()`), que não foi tocada. O botão leva `event.stopPropagation()` porque o
  card inteiro já tem `onclick="abrirQuadro(...)"`.
- **`input type="date"` nativo, e não o date picker customizado do app.** Decisão consciente: o
  picker (`dpDates`/`dpCals`) foi feito para data única obrigatória, e plugar duas chaves novas nos
  dois objetos globais seria mais superfície de bug do que ganho. Mesmo precedente do `type="time"`
  no modal de evento. Os dois campos usam a `.frow`, que já existia no CSS **sem nenhum uso** — e
  empilham no mobile via media query, senão ficariam espremidos no modal estreito.
- Validação: campo vazio grava **NULL**, não string vazia; as duas datas invertidas (`fim < inicio`)
  são recusadas com mensagem no modal. **Só uma das datas é permitido** — as colunas são
  independentes, o quadro apenas não ganha indicador. As datas vão no UPDATE mesmo quando nulas,
  que é o que permite limpar um prazo já gravado.
- **Cinco estados de badge**, calculados no cliente por `statusPrazoQuadro()`:
  sem as duas datas → **nenhum badge**; antes do início → "Ainda não começou"; dentro do período →
  "Em andamento" + barra de progresso; depois do fim com cartão fora da última lista → "⚠️
  Atrasado"; depois do fim sem nada pendente → "Encerrado".
- O texto do andamento sai em **semanas quando o período fecha redondo** ("3 de 6 semanas") e em
  dias quando não fecha ("7 de 31 dias"). O período é inclusivo nas duas pontas, então 20/07 a
  30/08 são 42 dias = 6 semanas. As diferenças em dias reusam a `diasEntre()` já escrita para os
  Relatórios, que é imune a fuso e horário de verão; a comparação entre datas é string-compare, que
  em `YYYY-MM-DD` ordena certo.
- **"Concluído" continua derivado da POSIÇÃO do cartão** (última lista), o mesmo conceito do
  `.plan-fim` da Fase 4 — a tabela `cards` segue sem campo de status e esta etapa não mexeu em
  schema.
- **Nenhuma cor nova.** Os três estilos de badge são exatamente os que a aba Treinos já usa, com a
  mesma semântica: branco sólido = pede atenção (`.treino-status.st-pendente`), vidro contornado =
  encerrado (`.treino-status.st-montado`), vermelho translúcido = alerta (os 3 valores do
  `.cfg-danger`). A barra usa `rgba(255,255,255,0.18)` no trilho (o mesmo do `.treino-avatar`) e
  `var(--white)` no preenchimento.
- **O card atrasado leva borda E fundo vermelhos, não só borda.** Verificado em preview: sobre o
  gradiente laranja, a borda vermelha sozinha praticamente não se distingue da borda branca padrão
  — o "destaque duplo" só existe de fato com o fundo junto, que é como o `.cfg-danger` sempre
  funcionou nas telas de vidro.
- **A contagem de pendentes só roda para os quadros JÁ VENCIDOS** (`carregarPendenciasVencidos()`).
  No caso comum não há quadro vencido e a função **não faz consulta nenhuma**; quando há, são no
  máximo duas (`colunas` e `cards`, ambas por `sbClient.from()` autenticado, como o resto da aba).
- **⚠️ A contagem passa pelo RLS, então pode divergir entre perfis.** O `cards_select` entrega a
  cada um só os cartões liberados para ele: um cartão pendente invisível para o professor não entra
  na conta dele, e o mesmo quadro pode aparecer como "Encerrado" para ele e "Atrasado" para o
  gestor. **Não é bug** — a alternativa (contar cartões que a pessoa não pode ver) vazaria a
  existência deles. **Decisão do Felipe em 2026-08-07:** o badge aparece para todos os perfis,
  com essa divergência assumida, porque quem executa o trabalho precisa enxergar o prazo.
- **Falha na conferência nunca vira "Encerrado".** Se a consulta de listas/cartões der erro, ou se
  o PostgREST truncar o resultado (o corte é em 1000 linhas por padrão, e um corte aqui esconderia
  justamente o cartão pendente), o quadro é mostrado como **"Atrasado" sem contagem** e o motivo
  vai pro console. Falso alarme que faz o gestor abrir o quadro é muito menos grave que dizer
  "Encerrado" para algo pendente — mesmo raciocínio do aviso de truncamento nos Relatórios.
- **Testado antes do commit, não só lido:** as funções puras (`statusPrazoQuadro`, `fmtPeriodo`)
  passaram por 15 casos em Node, incluindo primeiro e último dia do período, período de 1 dia,
  período que não fecha em semanas, e fim anterior ao início; a `carregarPendenciasVencidos()`
  passou por 10 casos com um cliente Supabase falso — listas fora de ordem, dois quadros vencidos
  simultâneos, quadro sem lista, quadro sem cartão, e os dois caminhos de erro. **Nenhum teste
  tocou o banco de produção.** O visual foi conferido em preview isolado (desktop e 390px), com o
  CSS e o JS reais extraídos do `index.html`.

- **Planejamento: quadro privado — Fase 4.1, 2026-08-07.** O gestor marca um quadro como privado e
  ele deixa de aparecer para todo mundo que não seja o criador — **inclusive para outros gestores**.
  O RLS desta etapa **já estava em produção e testado com contas reais antes desta implementação**
  (Felipe vê e edita o quadro marcado; Leonardo Neves, outro gestor, vê o quadro sumir da lista
  dele). Esta entrada cobre só a UI: **nenhuma policy, função ou tabela foi criada ou alterada.**
- Já existiam no banco, e o app apenas consome: a coluna
  `quadros.privado boolean not null default false`, a função `security definer`
  `posso_ver_quadro(p_quadro_id uuid)`, e as 7 policies do Kanban já reescritas em cima dela.
- **Checkbox "Privado" no modal, que é um só** — o `#quadro-modal` serve criação e edição desde a
  Fase 4. Nasce **desmarcado** na criação (quadro público segue sendo o comportamento padrão;
  privacidade por acidente seria pior que a falta dela) e reflete `quadros.privado` na edição.
  Reusa `.plan-pessoa` (a linha de checkbox da lista de pessoas do cartão) e `.plan-vis-aviso`
  (o aviso fixo da visibilidade do cartão) — **nenhum CSS novo no modal**.
- `privado` entra no INSERT e no UPDATE, mantendo o `.select('id')` já padrão da aba. No UPDATE o
  campo vai **mesmo quando `false`** — é o que permite desmarcar e voltar o quadro a público.
- **Selo `🔒 PRIVADO` na lista de quadros**, na linha do tema, herdando dela tamanho, peso e
  uppercase. `privado` entrou no SELECT do `loadQuadros()` só para alimentar o selo e o checkbox:
  **não há refiltro por `privado` no JavaScript**, porque o quadro privado de outra pessoa nem
  chega no array — quem filtra é a policy `quadros_select`.
- **O selo leva os TRÊS valores do `.cfg-danger` (fundo + borda + texto), não só a cor do texto.**
  Testadas três variantes em preview sobre o gradiente real: `#ffb4ab` como texto solto lê como
  rosa desbotado, **mais fraco que o próprio tema ao lado** — o oposto do que um marcador de
  privacidade deveria fazer; `#ff5a46` melhora pouco, porque vermelho sobre laranja é contraste
  ruim por natureza. O vermelho do `.cfg-danger` sempre veio do **fundo**, e sem ele o alerta se
  perde. Fica `inline` de propósito, para o selo quebrar junto com o texto quando o tema é longo.

- **Trava do checkbox "Privado" para quem não criou o quadro (mitigação de risco, 2026-08-07).**
  Identificado **antes de qualquer teste em produção**: a policy `quadros_write` é
  `eh_gestor() AND posso_ver_quadro(id)`, então qualquer gestor pode editar um quadro público que
  não é dele — inclusive marcá-lo privado. Como a `posso_ver_quadro()` libera quadro privado só
  para o `criado_por`, o quadro **sumiria da lista de quem acabou de marcar**, sem forma de
  desfazer, porque a pessoa perderia o acesso junto. O banco não erraria; o resultado é que seria
  absurdo.
- Mitigação: `souCriadorDoQuadro(q)` compara `quadros.criado_por` com `currentEquipeId` (para isso
  o `criado_por` passou a ser lido no SELECT, o que antes não acontecia). No modal de edição, quem
  não é o criador vê o checkbox **desabilitado e explicado**, com o nome de quem criou — e não
  escondido: sumir sem dizer nada deixaria o gestor procurando a opção. No save, `privado` fica
  **fora do payload** nesse caso, então o PostgREST nem toca a coluna. A criação nunca trava
  (`!id || souCriadorDoQuadro(q)`), porque ali o criador é sempre quem está criando.
- **Uma função só para as duas pontas**, de propósito: checagem duplicada entre modal e save
  poderia divergir num refactor futuro e o campo voltaria a escapar para o payload.
- **Isto é conveniência de interface, não segurança** — mesma natureza do
  `podeEditarPlanejamento()` e do `podeCadastrarTreino()`. A `quadros_write` continua permitindo,
  pelo DevTools, que um gestor marque privado um quadro alheio. Fechar de verdade exigiria um
  `WITH CHECK` na policy (algo como `criado_por = meu_equipe_id()` quando `privado = true`), o que
  é mudança de RLS: **decisão consciente do Felipe de não fazer agora** e avaliar junto com a
  Fase 2.
- Cobertura: 16 casos em Node sobre `souCriadorDoQuadro`/`nomeCriador` e a montagem do payload,
  incluindo não-criador com o checkbox forçado a `true`, conta sem vínculo na equipe, criador
  desativado (fora do array `equipe`, que só tem gente ativa) e **`criado_por` nulo com
  `currentEquipeId` nulo** — este último é o perigoso: sem o `!!` na função, `null === null` daria
  `true` e liberaria a trava exatamente para quem não deveria.

- **⚠️ RESSALVA CONHECIDA E NÃO TESTADA — Fase 4.1 subiu com ela em aberto (2026-08-07).**
  **Não testado ainda: se a trava do checkbox "Privado" (para não-criador) interfere na edição
  normal de outros campos (tema/nome/datas) do mesmo modal. Testar na próxima sessão antes de
  confiar 100% na Fase 4.1 em uso real por outro gestor que não o criador do quadro.**
  Contexto para quem for testar: a leitura do código sugere que **não** interfere — o `disabled`
  age só no próprio checkbox, os demais campos são lidos separadamente no `saveQuadro()`, e o
  payload do não-criador (`tema`, `nome`, `data_inicio`, `data_fim_prevista`) é aceito pela
  `quadros_write`, já que num quadro público a `posso_ver_quadro()` devolve `true` para qualquer
  gestor. **Mas isso é raciocínio sobre o código, não observação** — e este projeto já tem
  histórico de RLS que barra em silêncio, devolvendo zero linhas em vez de erro. Só vale como
  testado depois de rodar com a conta de um segundo gestor.

### Corrigido
- Chamadas `sbGet`/`sbPost`/`sbPatch`/`sbDelete` que falham por erro de conexão agora exibem
  um banner fixo de aviso no topo da tela, em vez de silenciar o erro e mostrar "0 eventos"
  como se fosse dado real.

### Segurança
- **`montagem_treinos` é a primeira tabela do projeto com RLS de verdade (2026-08-02)** — habilitado,
  com **cinco** policies (o SELECT é dividido em duas, somadas com `OR` pelo Postgres) e escopo
  derivado de `auth.uid()`, em vez do `anon_all` genérico usado em `eventos`/`rodizios`/`config` ou
  do `using(true)` da `equipe`. Texto real extraído de `pg_policies`:

  ```sql
  create policy treinos_select_professor on public.montagem_treinos for SELECT to authenticated
    using ((professor_id IN ( SELECT equipe.id
     FROM equipe
    WHERE (equipe.auth_user_id = auth.uid()))));

  create policy treinos_select_gestor_recepcao on public.montagem_treinos for SELECT to authenticated
    using ((EXISTS ( SELECT 1
     FROM equipe
    WHERE ((equipe.auth_user_id = auth.uid()) AND (equipe.perfil = ANY (ARRAY['gestor'::text, 'recepcao'::text]))))));

  create policy treinos_insert_recepcao_gestor on public.montagem_treinos for INSERT to authenticated
    with check ((EXISTS ( SELECT 1
     FROM equipe
    WHERE ((equipe.auth_user_id = auth.uid()) AND (equipe.perfil = ANY (ARRAY['gestor'::text, 'recepcao'::text]))))));

  create policy treinos_update_montagem on public.montagem_treinos for UPDATE to authenticated
    using (((professor_id IN ( SELECT equipe.id FROM equipe WHERE (equipe.auth_user_id = auth.uid())))
     OR (EXISTS ( SELECT 1 FROM equipe WHERE ((equipe.auth_user_id = auth.uid()) AND (equipe.perfil = ANY (ARRAY['gestor'::text, 'recepcao'::text])))))))
    with check (((professor_id IN ( SELECT equipe.id FROM equipe WHERE (equipe.auth_user_id = auth.uid())))
     OR (EXISTS ( SELECT 1 FROM equipe WHERE ((equipe.auth_user_id = auth.uid()) AND (equipe.perfil = ANY (ARRAY['gestor'::text, 'recepcao'::text])))))));

  create policy treinos_delete_recepcao_gestor on public.montagem_treinos for DELETE to authenticated
    using ((EXISTS ( SELECT 1
     FROM equipe
    WHERE ((equipe.auth_user_id = auth.uid()) AND (equipe.perfil = ANY (ARRAY['gestor'::text, 'recepcao'::text]))))));
  ```

  Professor lê e atualiza só as linhas onde `professor_id` é a linha dele; gestor/recepção fazem
  tudo. O `with check` do UPDATE repete o `using` — é o que impede o professor de reatribuir o
  treino para outro professor.
- **Grants (2026-08-02):** `authenticated` com `SELECT, UPDATE, DELETE, TRUNCATE, REFERENCES,
  TRIGGER` lidos no painel; `INSERT` não apareceu na leitura mas necessariamente existe (policy não
  substitui grant, e o INSERT do teste passou). `anon` **sem nenhum CRUD** — sobraram só `TRUNCATE`
  e `REFERENCES` da concessão padrão do Supabase.
- **Pendência de segurança aberta: revogar `TRUNCATE` do `anon`.** `TRUNCATE` **não passa por RLS** —
  nenhuma policy protege contra ele. Sem caminho de exploração conhecido hoje (a Data API do
  PostgREST não expõe verbo de TRUNCATE), então é risco latente e não buraco aberto, mas é
  privilégio sem uso legítimo: `revoke truncate, references on public.montagem_treinos from anon;`
- **Validação por teste automatizado, não por leitura de policy (2026-08-02).** Script Node
  (`teste-rls-treinos.js`, mantido fora do repositório) logando com **anon key** — nunca a service
  role, que ignora RLS e faria qualquer teste passar. Ele cria um treino descartável para o
  Professor B, loga como Professor A e tenta agir sobre ele. Resultado:
  - **SELECT bloqueado:** o professor A não enxerga o treino do professor B (zero linhas).
  - **UPDATE bloqueado:** `HTTP 200` com **zero linhas afetadas** — o bloqueio silencioso esperado
    da policy. Distinto de `42501`, que seria grant faltando (problema diferente, e o script
    diferencia os dois no veredito).
  - Gestor/recepção seguem com acesso total.
  - O teste criou e removeu um treino descartável e dois professores de teste (`ativo = false`),
    sem tocar em dado real de produção.
  As senhas são digitadas no terminal com máscara — nunca em arquivo, `.env` ou histórico de shell.
- **Este é o padrão-alvo da Fase 2.** Ao fechar `eventos`, `rodizios`, `config` e `equipe`, replicar
  as três decisões que fizeram diferença aqui: (1) grant de tabela **além** da policy — são coisas
  diferentes e ambas precisam existir; (2) policy separada por operação em vez de um `ALL` genérico;
  (3) teste automatizado com a anon key **provando** o bloqueio, em vez de conferir a policy no olho.
- **Fase 4 (2026-08-06): as 4 tabelas do Planejamento nascem com RLS de verdade.** Segunda vez que o
  projeto acerta isso — e desta vez com um desenho melhor que o da `montagem_treinos`. Texto real
  lido de `pg_policies`, **7 policies, todas para o role `authenticated`**:

  | tabela | policy | cmd | using | with check |
  |---|---|---|---|---|
  | `quadros` | `quadros_select` | SELECT | `true` | — |
  | `quadros` | `quadros_write` | ALL | `eh_gestor()` | `eh_gestor()` |
  | `colunas` | `colunas_select` | SELECT | `true` | — |
  | `colunas` | `colunas_write` | ALL | `eh_gestor()` | `eh_gestor()` |
  | `cards` | `cards_select` | SELECT | ver abaixo | — |
  | `cards` | `cards_write` | ALL | `eh_gestor()` | `eh_gestor()` |
  | `card_visibilidade` | `vis_all` | ALL | `eh_gestor()` | `eh_gestor()` |

  ```sql
  -- cards_select (texto ATUAL, depois da correção de 2026-08-06):
  -- gestor vê tudo; os demais, só o que foi liberado pro perfil ou pra pessoa.
  eh_gestor() OR pode_ver_card(cards.id)
  ```

  Quadros e listas são legíveis por qualquer autenticado (`using true`) **de propósito**: a estrutura
  do quadro não é segredo, o conteúdo é. O que filtra de verdade é o `cards_select`.
- **As 4 funções auxiliares — todas `security definer`, `STABLE`, `SET search_path TO 'public'`:**

  ```sql
  eh_gestor()      -> boolean : select exists (select 1 from equipe
                                where auth_user_id = auth.uid() and perfil = 'gestor')
  meu_equipe_id()  -> uuid    : select id from equipe where auth_user_id = auth.uid() limit 1
  meu_perfil()     -> text    : select perfil from equipe where auth_user_id = auth.uid() limit 1

  ```

  E a `pode_ver_card()`, criada na correção de 2026-08-06, com a definição completa:

  ```sql
  create or replace function public.pode_ver_card(p_card_id uuid)
  returns boolean
  language sql
  security definer
  stable
  set search_path = public
  as $$
    select exists (
      select 1 from card_visibilidade v
      where v.card_id = p_card_id
        and (v.equipe_id = meu_equipe_id() or v.perfil = meu_perfil())
    );
  $$;
  ```

  Corpos lidos do banco em 2026-08-06.

  **Olhe o corpo dela e compare com o `EXISTS` quebrado logo abaixo: é a mesma consulta.** Mesmo
  `select exists`, mesma tabela, mesmas comparações. A única coisa que mudou foi *onde* ela roda —
  dentro de uma função `security definer` em vez de inline na policy. **A lógica nunca esteve
  errada.** É por isso que esse bug é tão difícil de enxergar lendo a policy: não há nada de errado
  para ver.

  Dois detalhes do `pode_ver_card()` que parecem descuido e não são: (a) ele não precisa testar se
  `perfil`/`equipe_id` é nulo, porque o CHECK garante que só um dos dois está preenchido e
  `NULL = valor` resulta em `NULL`, nunca em `true`; (b) quem não tem linha na `equipe` recebe
  `NULL` das duas funções, não casa com nada e não vê cartão nenhum — que é o comportamento certo.

  **Por que elas existem, que é o ponto a não esquecer:** nenhuma das 7 policies consulta `equipe`
  ou `card_visibilidade` diretamente — todas passam por essas funções. Como são `security definer`,
  rodam com os privilégios do dono e **não são afetadas pelo RLS das tabelas que consultam**.
  Consequência prática: quando a Fase 2 apertar a `equipe` para cada pessoa ler só a própria linha,
  **nenhuma policy do Planejamento quebra**. É a mesma preocupação registrada para a
  `montagem_treinos` (cujas policies foram escritas com o cuidado de só tocar a linha de quem
  chamou), resolvida aqui de forma estrutural em vez de por disciplina. O
  `SET search_path TO 'public'` **não é enfeite**: sem ele, `security definer` é um vetor clássico
  de escalada de privilégio, porque o chamador poderia apontar o `search_path` para um schema com
  uma tabela `equipe` falsa.
- **⚠️ LIÇÃO APRENDIDA DA FASE 4 (2026-08-06) — subconsulta dentro de policy herda o RLS da tabela
  consultada.** Está aqui no mesmo nível do "policy não substitui grant" da Fase 1: é o tipo de
  erro que custa horas porque **não dá erro nenhum**.

  **O bug.** A `cards_select` original fazia um `EXISTS` direto contra `card_visibilidade`:

  ```sql
  -- ERRADO — não use como referência. Registrado só para reconhecer o padrão.
  eh_gestor() OR (EXISTS ( SELECT 1
     FROM card_visibilidade v
    WHERE ((v.card_id = cards.id) AND ((v.equipe_id = meu_equipe_id()) OR (v.perfil = meu_perfil())))))
  ```

  Isso não funciona. A subconsulta em `card_visibilidade` **herda o RLS daquela tabela**, que tem
  só a `vis_all` (`ALL using eh_gestor()`). Para quem não é gestor, a subconsulta enxerga zero
  linhas — então o `EXISTS` **nunca encontrava nada**, mesmo com o dado 100% correto gravado na
  tabela. Resultado: professor e recepção não viam cartão nenhum, e nada no log acusava.

  É o "RLS barra em silêncio" já conhecido do projeto, só que **aninhado dentro de outra policy** —
  bem mais difícil de enxergar, porque a policy que falha (`cards_select`) não é a policy culpada
  (`vis_all`), e a tabela que você está depurando não é a tabela que está barrando.

  **A correção**, aplicada em produção pelo SQL Editor em 2026-08-06: criar a função
  `pode_ver_card(p_card_id uuid)` como `security definer`, encapsulando a checagem de
  visibilidade, e trocar a policy para `eh_gestor() OR pode_ver_card(cards.id)`. A função ignora o
  RLS de `card_visibilidade`, que é exatamente o que se quer aqui.

  **Confirmado com conta real**, não por leitura de policy: o Leonardo (perfil `recepcao`) passou a
  ver corretamente só o cartão liberado para o perfil dele. Antes da correção, via zero.

  **Cobertura do teste — os dois ramos do `pode_ver_card()` estão validados** com dado e sessão
  reais, em 2026-08-06. Isso importa porque a função tem dois caminhos independentes, e passar num
  não prova nada sobre o outro: são linhas diferentes da `card_visibilidade` (uma com `perfil`
  preenchido e `equipe_id` nulo, outra ao contrário) e comparações diferentes na função.

  | ramo | cartão usado | conta | resultado |
  |---|---|---|---|
  | `v.perfil = meu_perfil()` | "Campanha de Reavaliação" | Leonardo, perfil `recepcao` | vê o cartão (antes da correção, via zero) |
  | `v.equipe_id = meu_equipe_id()` | "Design" (sem perfil marcado, badge "1 PESSOA") | Leonardo Aparecido, por `equipe_id` | vê o cartão |

  Nenhum dos dois foi verificado por leitura de policy — só por login real numa conta que não é
  gestor, que é o único jeito de provar que o `cards_select` filtra como deveria.

  **REGRA GERAL DO PROJETO, a partir daqui:** qualquer policy que precise consultar **outra tabela
  com RLS restritivo** tem que passar por função `security definer` — **nunca `EXISTS` direto**.
  Com `eh_gestor()`, `meu_equipe_id()`, `meu_perfil()` e agora `pode_ver_card()`, são quatro casos:
  isto é **padrão do projeto, não exceção**. Vale integralmente para a Fase 2, que vai escrever
  policies novas em cima da `equipe` — a tabela mais consultada por policy no banco inteiro.

  Corolário útil: **a `montagem_treinos` está a salvo disso** porque suas policies consultam a
  `equipe`, que hoje tem policy permissiva (`using(true)`). No dia em que a Fase 2 apertar a
  `equipe`, aqueles `EXISTS` diretos passam a ter exatamente este bug — silenciosamente. Converter
  as policies da `montagem_treinos` para `security definer` **antes** de mexer na `equipe`, não
  depois.
- **Correção de documentação:** o commit `3adf47a` registrou a `cards_select` com o `EXISTS` direto,
  ou seja, com o bug. O texto acima é o correto. Registrado aqui em vez de reescrito no histórico
  porque a versão errada é justamente o que dá contexto à lição.
- **Grants da Fase 4 (idênticos nas 4 tabelas):** `authenticated` e `postgres` com
  `SELECT, INSERT, UPDATE, DELETE, REFERENCES, TRIGGER, TRUNCATE`; `anon` e `service_role` com
  apenas `REFERENCES, TRIGGER, TRUNCATE` — ou seja, **`anon` sem nenhum CRUD**, como deve ser.
- **Pendência de segurança aberta: revogar `TRUNCATE` do `anon` nas 4 tabelas novas.** Veio do
  `ALTER DEFAULT PRIVILEGES` padrão do Supabase, não de decisão de ninguém. Importa porque
  **`TRUNCATE` não passa por RLS** — policy nenhuma protege contra ele. Sem caminho de exploração
  conhecido hoje (o PostgREST não expõe verbo de TRUNCATE), então é risco latente e não buraco
  aberto; **o Felipe decidiu em 2026-08-06 não revogar agora** e deixar para a Fase 2, junto com o
  mesmo problema já registrado na `montagem_treinos`:
  `revoke truncate on quadros, colunas, cards, card_visibilidade from anon;`
- **`docs/schema.sql` não contém `montagem_treinos` nem as 4 tabelas da Fase 4; conferir no Table
  Editor antes de confiar nele.** O arquivo também declara `equipe.id` como
  `bigint generated by default as identity`, enquanto as chaves em uso são UUID. Atualizá-lo é
  pendência de auditoria — hoje ele descreve um banco que não existe mais.
- **Policy `anon_all_equipe_authenticated` (ALL para `authenticated`, `USING (true) WITH CHECK
  (true)`) criada em 2026-07-31**, sem tocar na `anon_all_equipe` original. Necessária porque
  depois do login o usuário deixa de ser `anon` e vira `authenticated` — sem ela, a leitura do
  perfil falharia e o roteamento quebraria.
- **`GRANT SELECT ON public.equipe TO authenticated` — descoberto durante os testes.** A policy
  sozinha não bastava: sem o grant de tabela o Postgres recusa antes de sequer avaliar o RLS
  (erro `42501 permission denied`). O sintoma era traiçoeiro — o login passava e só a leitura do
  perfil falhava depois. Fica a regra: **policy e grant são coisas diferentes e ambas precisam
  existir.**
- **Ambos são provisórios e a Fase 2 tem que apertá-los com `auth.uid()`.** Hoje qualquer usuário
  autenticado lê a tabela `equipe` inteira.
- **Pegadinha de diagnóstico, para não perder tempo de novo:** RLS que barra não devolve erro,
  devolve **zero linhas**. "Policy faltando" e "linha não vinculada" dão exatamente o mesmo
  sintoma na tela. Grant faltando, ao contrário, dá erro explícito (`42501`).
- **Rate limit do Supabase Auth:** após várias tentativas de login erradas seguidas, o Supabase
  bloqueia temporariamente e recusa **até a senha correta** por alguns minutos. Encontrado durante
  os testes de 2026-07-31. **Não é bug, é proteção** — antes de resetar senha de alguém, pergunte
  quantas vezes a pessoa errou e espere alguns minutos.
- **Não existe "esqueci minha senha" para o usuário final**, porque não há SMTP configurado e a
  confirmação por email está desligada. Reset é sempre feito pelo Felipe, pelo painel
  (Authentication → Users) ou via SQL com pgcrypto:
  `create extension if not exists pgcrypto;` +
  `update auth.users set encrypted_password = crypt('<nova senha>', gen_salt('bf')) where email = '<login>@equipe.evolucaofit.app';`
- **Incidente durante o desenvolvimento (2026-07-31): a senha real do Felipe foi trocada por
  acidente num teste.** Ao validar as mensagens de erro do formulário de trocar senha, os casos
  foram executados direto na função `savePassword()`; o caso "senha válida" rodou com uma sessão
  real ainda ativa no `localStorage` do navegador (sobrevivente do teste de login anterior) e o
  `updateUser` foi pra valer. Corrigido com reset pelo painel. Nenhum outro dado foi tocado — a
  única escrita foi essa. **Regra que fica: não executar função que escreve enquanto houver sessão
  ativa; conferir `getSession()` antes, ou testar só os ramos anteriores à chamada da API.** O
  risco real não é o gestor (que reseta a própria senha em um minuto) e sim repetir isso com a
  conta de um professor ou recepcionista, que ficaria sem acesso no meio do expediente sem saber
  por quê. Registrado também no `CLAUDE.md`.
- **O banco continua ABERTO — a Fase 1 não fechou nada.** `eventos`, `rodizios` e `config` seguem
  legíveis e graváveis por qualquer pessoa com a anon key (que é pública por design). O login
  resolveu *quem é você*; *o que você pode fazer* ainda é decidido por `if` no JavaScript, que é
  cosmético e burlável pelo DevTools. Fechar isso é a Fase 2.
- **Resolvido pela raiz: o PIN do Gestor foi eliminado.** O registro de 2026-07-13 apontava que o
  PIN em produção ainda era o padrão de fábrica (`1234`) e que a troca manual continuava pendente.
  Com o login real, o PIN deixou de existir — não há mais o que trocar. Fica o registro histórico
  do risco: PIN padrão em produção é risco real, mesmo em app interno, e ele sobreviveu ~18 dias
  depois de detectado.
- Registro histórico relacionado: a coluna `pin` da tabela `config` era legível via anon key (o
  app lia o PIN no client para validar o Gestor), então qualquer pessoa com a anon key conseguia
  ler o PIN atual via REST. O login via Supabase Auth resolve isso — a senha nunca trafega nem
  fica legível pelo client. A coluna `pin` virou dado morto e continua no banco; candidata a
  `drop column` na Fase 2.

### Removido
- **Barra de abas `.tb-nav`, em todas as suas partes (2026-08-10, etapa 2 do drawer):** os 7
  botões `.tb-nav-item`, o CSS deles (normal/hover/active), as regras de scroll horizontal do
  mobile e os dois `style.display` que o `enterApp()` fazia em `#nav-relatorios` e `#nav-config`.
  O `goPage()` perdeu o parâmetro `el` (só existia para marcar a aba clicada) e o
  `irParaTreinos()` deixou de procurar o botão **pelo texto**, virando `goPage('treinos')`.
  Removida junto a compensação `min-height:calc(100vh - 100px)` do mobile, que só existia porque
  a barra de abas ocupava uma segunda linha na topbar — voltou ao `calc(100vh - 56px)` geral.
  - **⚠️ Consequência que exigiu decisão: no celular a marca "EVOLUÇÃO" saiu da topbar** e o nome
    da página ocupou o lugar dela. Medido em 390px: chip 34 + marca 92 + página 75 + nome 78 +
    badge 66 + ⋯ 34, com os gaps dá ~409px para 366px úteis — os dois não cabiam. Entre manter o
    que nunca muda (nome da empresa) e o que muda (onde estou), o Felipe escolheu o segundo.
    Com isso a topbar mobile ficou em **51px de altura, uma linha só**, o que é o que torna a
    volta ao `calc(100vh - 56px)` correta. A marca continua no desktop, no splash e no login.
- **PIN do Gestor, em todas as suas partes (2026-07-31):** o overlay com teclado numérico, o CSS
  do teclado e do modal, as funções `pinKey`/`pinDel`/`updateDots`/`openPinOverlay`/
  `closePinOverlay`, os globais `currentPIN`/`pinBuffer`, e também o card "PIN do Gestor" na tela
  de Config junto com `savePIN()`. O card foi removido por decisão consciente e não por descuido:
  deixá-lo permitiria ao gestor configurar uma proteção que não protege mais nada — pior do que
  não ter. O lugar dele foi ocupado pelo card "Minha senha" (ver "Adicionado").
- Seleção de perfil por card na tela de entrada (`selectProfile`, `.profile-card`, `.profile-grid`).
  O perfil agora vem do banco, pela pessoa autenticada — não é mais escolha do usuário.

### Alterado
- **Refino da base de vidro propagado para o app inteiro (2026-08-10).** As três camadas validadas
  no Planejamento (ver o item seguinte) passaram a valer em **todas as 7 telas** — Agenda,
  Calendário, Rodízio, Treinos, Relatórios, Config e Login — **e no drawer**, que era o último
  painel de vidro na base antiga. Aprovado em teste real a cada etapa.
  - **Os três valores subiram do `#page-planejamento` para o `:root`.** Três valores repetidos em
    sete telas seriam sete fontes de verdade — a mesma armadilha que o `PLAN_CORES` evita nas
    cores de lista.
  - Elementos refinados: `.stat`/`.ev-card`/`.tl-card` (Agenda), `.cal-wrap`/`.cal-day-panel`/
    `.ev-card` (Calendário), `.rodo-card` (Rodízio), `.treino-card` (Treinos), `.rel-card`/
    `.rel-totais`/`.rel-tabela` (Relatórios), `.cfg-card` (Config), `.login-card` (Login) e
    `.drawer`.
  - **Não tocados, pela regra dos opacos:** inputs do Login e do Config, cartões sólidos do
    Kanban, itens do drawer, botões brancos, badges e pills. Verificado por script: zero
    elementos sólidos receberam reflexo.
  - **9 regras que trocam `background` de elemento de vidro precisaram repetir o
    `var(--vidro-refl)`** — quase todas dentro do `@supports`, onde cada elemento redefine a
    própria opacidade. Sem isso o reflexo sumiria em silêncio naquele elemento.
  - A `::before` de 1px no topo saiu de todos os elementos refinados: o inset branco de 2px é a
    mesma refração, mais forte e na largura inteira.
- **Destaque de STATUS ficou de fora do refino, por decisão (2026-08-10).**
  `#page-agenda .ev-card.cat-reuniao` e `#page-treinos .treino-card.st-pendente` mantêm o
  `box-shadow` original (borda esquerda branca de 3px + glow interno), sem reflexo de canto e sem
  contorno de 2px. Os dois efeitos disputam a mesma aresta esquerda, e ali o destaque é
  **informação** ("isto é uma reunião", "isto pede ação") enquanto o refino é **acabamento** — com
  o refino aplicado a hierarquia chegava a inverter, deixando o cartão pendente com menos contorno
  que os já montados e cancelados. A exclusão vale nos **dois** lugares (o `box-shadow` da regra
  de destaque e o `background` dentro do `@supports`); tirar só o contorno deixaria o reflexo
  competindo do mesmo jeito.
- **Removido o glow decorativo do `.cal-day-panel` (2026-08-10).** Era um `::after` no canto
  superior-**direito**, a única luz num terceiro canto em todo o app, maior e mais forte que a
  própria fonte de luz do refino. Com ele, aquele painel tinha três luzes enquanto todo o resto
  tem duas. **Não recriar.**
- **Refino da base de vidro — protótipo na aba Planejamento (2026-08-10).** Três camadas
  novas sobre os elementos de vidro, aprovadas no celular físico. Blur, saturate, raio e cor de
  base seguem os **mesmos tokens de antes** — o refino só acrescenta.
  1. **Reflexo de borda nos dois cantos opostos, esticado ao longo da aresta** (não circular):
     dois `radial-gradient` por canto, um achatado na horizontal e outro na vertical.
     Superior-esquerdo em branco forte (a fonte de luz), inferior-direito em tom quente
     (`rgba(255,220,180,…)`) e mais discreto. **Quente e não escuro de propósito:** sobre fundo
     escuro um tom escuro simplesmente some.
  2. **Contorno de 2px nos mesmos dois cantos**, via `box-shadow: inset`, por cima do `border`
     de 1px que continua dando a volta.
  3. **Sombra externa de flutuação em tom quente escuro** (`rgba(20,6,2,0.55)`) — preto puro
     sobre o gradiente laranja parece colado, sem naturalidade.
  - **A posição do brilho é FIXA** (superior-esquerdo / inferior-direito) em todos os elementos,
    mesmo quando o glow de fundo daquela tela está em outro canto. É consistência entre telas,
    não espelho do gradiente de cada uma.
  - Os três valores vivem em **custom properties** no `#page-planejamento` (`--vidro-refl`,
    `--vidro-borda`, `--vidro-sombra`). Dois motivos: é protótipo feito para ajuste fino, então o
    tuning fica num lugar só; e o `.plan-atrasado` **sobrescreve o `background` inteiro** — sem o
    token ele seria o único card sem reflexo, e falharia calado.
  - **Os cartões do quadro (`.plan-card`) continuam SÓLIDOS.** Os reflexos entram por cima do
    `var(--g1)` (o gradiente vem antes da cor no mesmo `background`), sem blur e sem translucidez:
    a exceção registrada segue valendo, o cartão só ganhou a luz na aresta.
  - **O `#quadro-modal` não foi tocado** — é branco e usa a `.modal-box` compartilhada por todos
    os modais do app. Reflexo branco sobre fundo branco não aparece, e mexer ali vazaria para os
    outros modais, muito além de um protótipo.
  - A `::before` de 1px no topo **saiu desses dois elementos**: o novo inset branco de 2px é a
    mesma refração, mais forte e na largura inteira; manter as duas dobrava o destaque justamente
    na aresta que o refino quer mostrar.
  - Na época, divergência temporária e consciente: as outras telas seguiam na base antiga até
    esta ser validada. **Propagado no mesmo dia** — ver o item no topo desta seção.
- Tela de Configurações redesenhada com liquid glass — última tela do redesign (todas as
  telas do app agora seguem o mesmo visual): cards de seção em vidro translúcido, inputs
  mantidos SÓLIDOS (brancos opacos, sem vidro onde se digita), botões de ação em branco
  sólido, pills de cargo no padrão dos filtros da Agenda, e "Limpar todos os dados" em
  vermelho translúcido de alerta (ação destrutiva precisa continuar parecendo perigosa).
  CSS escopado em `#page-config` + classe `cfg-danger` no botão de limpar (cores saíram do
  style inline). Testado funcionalmente contra produção: adicionar/remover funcionário,
  salvar identidade e PIN (regravando valores idênticos), permissão por perfil intacta.
- Telas de entrada (seleção de perfil) e modal de PIN do Gestor redesenhadas com liquid glass:
  fundo gradiente "sol" no login com "Agenda." em destaque, perfil selecionado em branco
  sólido opaco vs não selecionados em vidro translúcido, botão "Entrar →" branco sólido.
  Modal de PIN com vidro intenso (blur 28px + saturate 200%, mais forte que os 3px do resto
  do app), franja de luz nas bordas, glows atrás do painel e bolinhas de progresso com glow —
  mas dígitos 0-9 mantidos SÓLIDOS (sem vidro) por legibilidade, por ser tela de segurança.
  Mudança 100% CSS (escopada em `#screen-login` e `#pin-overlay`) + classe `num-del` no botão
  de apagar; nenhuma lógica de autenticação, validação de PIN ou fluxo de perfil alterada.
  Splash e Config seguem com o visual original.
- Backfill no Supabase (2026-07-13): preenchido o `data_inicio` das 6 escalas de rodízio
  criadas antes da coluna existir (datas deduzidas do texto da semana + ano do `created_at`,
  todas sábados, validadas pelo Felipe antes do UPDATE). Com isso o destaque do próximo fim
  de semana e a ordenação cronológica passaram a funcionar em produção.
- Tela de Rodízio redesenhada com o mesmo visual "liquid glass" das telas de Calendário e
  Agenda: cabeçalho "Finais de semana" + título "Rodízio", um card de vidro por fim de semana
  (a estrutura de dados é por fim de semana, não por dia) com badges PROF. em branco sólido e
  RECEP. em vidro, botão "+ Nova escala" tracejado em vidro sutil (ação secundária, só Gestor)
  e aviso de bloqueio em vidro para não-gestores. O card do próximo fim de semana ganha vidro
  mais opaco — depende do campo `data_inicio`, que está nulo nas 6 escalas criadas antes da
  coluna existir; nesses casos nenhum card é destacado (degrada sem quebrar) e escalas
  novas/editadas ganham o destaque automaticamente. CSS escopado em `#page-rodizio` + classe
  `rodo-next` calculada em `renderRodizio()`; lógica, dados e permissões inalterados. Config
  e login seguem com o visual claro original.
- Tela de Agenda redesenhada com o mesmo visual "liquid glass" do Calendário: fundo em
  gradiente radial laranja→preto com glow no canto superior direito, cabeçalho com data do dia
  e título "Agenda" (antes "Eventos"), stats/cards/timeline em vidro translúcido (mesmo
  fallback `@supports`), hora do evento à esquerda fora do card com avatar circular de
  iniciais, reuniões com destaque (vidro mais opaco + barra lateral branca), filtros em pills
  (ativo branco sólido) e botão "+ Novo" em branco sólido para contraste máximo. Mudança
  visual: CSS escopado em `#page-agenda` + template dos cards em `renderEvents()`; lógica,
  dados, filtros e permissões por perfil inalterados. Rodízio e Config seguem com o visual
  claro original.
- Tela de Calendário redesenhada com visual "liquid glass": fundo em gradiente radial
  laranja→preto, card do dia em destaque com número grande, cards de evento e mini calendário
  em vidro translúcido (backdrop-filter com fallback sólido semi-opaco via `@supports` para
  navegadores sem suporte). Mudança só de CSS + template visual do painel do dia em
  `calClickDay()` — lógica, dados e perfis intactos. Somente esta tela; as demais permanecem
  com o visual claro original.
- Tabela `equipe` no Supabase: RLS estava habilitado sem nenhuma política, bloqueando todo
  INSERT/SELECT via anon key (erro 42501). Criada a política `anon_all_equipe` (ALL para o
  role anon). Isso é uma exceção deliberada ao padrão das demais tabelas (`config`, `eventos`,
  `rodizios`), que usam RLS desabilitado + permissão "anon_all" na Data API. Ver `CLAUDE.md`
  para o motivo da divergência.

## [2026-07] — Estado conhecido

### Adicionado
- App de agenda com três perfis de acesso: Professor, Recepção e Gestor (via PIN).
- Eventos com tipos (reunião, entrevista, rodízio, outro), recorrência configurável,
  controle de visibilidade por perfil e atas de reunião por evento.
- Escala de revezamento de fim de semana (professor + recepcionista), com suporte a recorrência.
- Configurações de Gestor: identidade da empresa, PIN, cadastro de equipe.
- Exportação de eventos em texto (WhatsApp), PDF/impressão e CSV.
- Suporte a instalação como PWA.

### Corrigido / Conhecido
- Identificado incidente em que o Supabase free tier pausa após 7 dias de inatividade, e o app
  engolia o erro de conexão silenciosamente, mostrando "0 eventos" em vez de avisar o usuário.
  Mitigação parcial: ping periódico ao Supabase para evitar a pausa (ver `CLAUDE.md`).
  Tratamento de erro visível ao usuário resolvido em 2026-07-03 (ver `[Unreleased]` acima).
