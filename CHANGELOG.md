# Changelog

Todas as mudanças notáveis deste projeto serão documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/).

> Nota: o histórico de commits anterior a este arquivo não tinha mensagens descritivas
> ("Update index.html" repetido), então as entradas abaixo anteriores a 2026-07 foram
> reconstruídas a partir do estado atual do código, não do histórico do Git.

## [Unreleased]

### Adicionado
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

### Corrigido
- Chamadas `sbGet`/`sbPost`/`sbPatch`/`sbDelete` que falham por erro de conexão agora exibem
  um banner fixo de aviso no topo da tela, em vez de silenciar o erro e mostrar "0 eventos"
  como se fosse dado real.

### Segurança
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
- **PIN do Gestor, em todas as suas partes (2026-07-31):** o overlay com teclado numérico, o CSS
  do teclado e do modal, as funções `pinKey`/`pinDel`/`updateDots`/`openPinOverlay`/
  `closePinOverlay`, os globais `currentPIN`/`pinBuffer`, e também o card "PIN do Gestor" na tela
  de Config junto com `savePIN()`. O card foi removido por decisão consciente e não por descuido:
  deixá-lo permitiria ao gestor configurar uma proteção que não protege mais nada — pior do que
  não ter. O lugar dele foi ocupado pelo card "Minha senha" (ver "Adicionado").
- Seleção de perfil por card na tela de entrada (`selectProfile`, `.profile-card`, `.profile-grid`).
  O perfil agora vem do banco, pela pessoa autenticada — não é mais escolha do usuário.

### Alterado
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
