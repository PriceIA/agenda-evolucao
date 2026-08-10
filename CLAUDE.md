# Agenda Evolução — Contexto do Projeto

## Sobre mim (Felipe)
- CEO da EvoluçãoFit (academia em São José dos Campos-SP) e dev freelancer sob a org GitHub `PriceIA`.
- Sou mais criativo do que "execution-oriented" — prefiro entender o panorama e decidir, não quero
  que me expliquem cada passo técnico óbvio.
- Sou usuário de desenvolvimento web comum, não sou dev profissional. Explique termos técnicos
  novos, mas sem infantilizar — direto ao ponto.
- Quero feedback honesto e crítico, não validação. Se uma decisão minha for ruim, diga e explique
  o porquê. Não concorde só pra agradar.
- Comunicação: português brasileiro, direto, sem enrolação.
- Regra de ouro: nunca avançar pra próxima fase sem antes commitar a fase anterior no GitHub.
  Eu tendo a pular etapas quando fico empolgado — me pare se eu tentar fazer isso.

## O que é o Agenda Evolução
- App interno de agendamento da EvoluçãoFit. URL: agenda-evolucao.vercel.app.
- Não é um projeto Next.js — é um único arquivo index.html estático (HTML + CSS + JS vanilla
  inline), hospedado na Vercel como site estático. Sem build, sem framework, sem package.json.
- Backend: Supabase (ref indapxewhfobzxnwswhy), acessado via REST API direto do client
  (fetch para /rest/v1/...), usando a anon key embutida no HTML.
- Única dependência externa: `supabase-js` via CDN (jsDelivr), **versão fixa `2.111.0` + atributo
  `integrity` (SRI)**. Versão fixa porque num app sem build uma tag flutuante (`@2`) mudaria o
  app sozinho. Ao atualizar a versão, **recalcular o hash do integrity junto**, senão o script
  para de carregar. Se o CDN falhar, `window.supabase` fica indefinido, `sbClient` vira `null`
  e o app mostra o banner vermelho com texto próprio (`showLibError()`) — nunca falha em silêncio.
- Três perfis de acesso: Professor, Recepção, Gestor.
- Funcionalidades: eventos recorrentes, atas de reunião, escala de revezamento de fim de semana,
  instalável como PWA.
- Tabelas conhecidas: events, rodizios, equipe (confirmar nomes exatos direto no Supabase
  Table Editor antes de assumir).
- RLS (Row Level Security): `config`, `eventos` e `rodizios` usam RLS **desabilitado** +
  permissão "anon_all" (ALL para role anon) na Data API. A tabela `equipe` é uma **exceção
  deliberada** (2026-07-13): RLS fica **habilitado**, com uma política explícita
  `anon_all_equipe` (ALL para anon, `USING (true) WITH CHECK (true)`) em vez de desabilitar o
  RLS como nas demais. Motivo: primeiro passo de um hardening futuro do banco, feito só nessa
  tabela por enquanto — não replicar esse padrão nas outras tabelas sem decisão explícita.

## Autenticação (Fase 1, concluída em 2026-07-31)
- Login é **usuário + senha via Supabase Auth**. O PIN do Gestor foi removido do app.
- O usuário digita **só o login** (ex.: `FelipePolenta`); ele nunca digita nem vê email. O app
  monta o email interno: `usuario.trim().toLowerCase() + '@equipe.evolucaofit.app'`. O
  `toLowerCase()` é obrigatório — o Supabase guarda o email em minúsculas e sem isso o match falha.
- **Confirmação por email está desligada** e não há SMTP configurado. Consequência prática:
  **não existe "esqueci minha senha" para o usuário**. Ver "Operação de contas" abaixo.
- O papel vem da tabela `equipe`: `select perfil, nome from equipe where auth_user_id = <auth.uid()>`,
  feita **autenticada** via supabase-js (não pela anon key). Valores de `perfil`:
  `professor` / `recepcao` / `gestor` — minúsculos, sem acento, batendo com as strings que o
  `currentRole` já usava.
- **`perfil` ≠ `cargo`.** `cargo` (`Professor`/`Recepção`/`Outro`) é função no rodízio e na tela
  de Config; `perfil` é acesso. Ex.: o Felipe tem `cargo = Recepção` e `perfil = gestor`.
- Sessão persistente (`persistSession` + `autoRefreshToken`, storageKey `agenda-evolucao-auth`).
  No boot, `getSession()` roda **antes** de qualquer chamada de rede — se rodasse depois, quem já
  está logado veria a tela de login piscar.
- Sem linha vinculada ou sem `perfil` válido: o app faz `signOut()` e mostra erro claro. Não
  existe "entrar sem perfil".
- **Trocar senha dentro do app** (`updateUser({ password })`): um modal único com **duas portas de
  entrada** — botão "🔑 Senha" na topbar (**todos os perfis**) e card "Minha senha" na Config
  (só Gestor, no lugar onde ficava o PIN). Duas portas porque a Config só é visível pro Gestor:
  se ficasse só lá, professor e recepção nunca trocariam a própria senha. O modal segue o padrão
  **branco** dos demais (Novo Evento / Nova Escala) — o vidro intenso era característica do
  overlay de PIN, que não existe mais; não recriar aquela exceção aqui.
- Mínimo de 6 caracteres, que é o **mínimo padrão do próprio Supabase**. Não inventar regra mais
  rígida só no cliente — validação que não bate com o servidor confunde. Para exigir mais, subir
  primeiro no painel (Authentication → Policies) e só então ajustar o número no app.
- Erros do Supabase chegam em inglês e passam por `traduzErroSenha()` (senha curta, igual à
  anterior, fraca, sessão expirada, rate limit). Qualquer caso fora da lista cai numa mensagem
  genérica **e** vai pro console com o erro cru — nunca silêncio.

### Operação de contas (só o Felipe faz, pelo painel do Supabase)
- **Criar usuário novo:** criar no painel (Authentication → Users) e depois vincular no SQL:
  `update public.equipe set auth_user_id = '<uid>', perfil = '<papel>' where id = '<id da pessoa>';`
- **Cadastrar alguém pela tela de Config NÃO cria login.** O `addEquipe()` grava só
  `nome`/`cargo`/`ativo`. Pessoa nova entra sem `perfil` e sem `auth_user_id` — e não loga até o
  vínculo manual acima.
- **Resetar senha esquecida:** pelo painel (Authentication → Users → Reset/Update password) ou
  via SQL com pgcrypto:
  ```sql
  create extension if not exists pgcrypto;
  update auth.users set encrypted_password = crypt('<nova senha>', gen_salt('bf'))
  where email = '<login>@equipe.evolucaofit.app';
  ```
- **Rate limit do Supabase Auth:** depois de várias tentativas de login erradas seguidas, o
  Supabase bloqueia temporariamente e recusa **até a senha certa** por alguns minutos. **Isso não
  é bug, é proteção.** Se alguém jurar que a senha está certa e não entra, pergunte primeiro
  quantas vezes errou antes — e espere alguns minutos em vez de sair resetando senha.

### Colunas e permissões acrescentadas na `equipe` (2026-07-31, Fase 1)
- `auth_user_id uuid unique references auth.users(id) on delete set null` — vínculo com o login.
  `unique` para um login não apontar pra duas pessoas; `on delete set null` para apagar um usuário
  no painel não travar por chave estrangeira.
- `perfil text check (perfil in ('professor','recepcao','gestor'))` — **aceita NULL de propósito**:
  quem não tem perfil não loga. Um `default` daria acesso por acidente.
- Policy `anon_all_equipe_authenticated` (ALL para `authenticated`, `USING (true) WITH CHECK (true)`),
  criada **sem tocar** na `anon_all_equipe` original. Necessária porque depois do login o usuário
  deixa de ser `anon` e vira `authenticated`.
- `GRANT SELECT ON public.equipe TO authenticated`. **Policy não basta**: sem o grant de tabela o
  Postgres recusa antes de avaliar o RLS (erro `42501 permission denied`). O sintoma era cruel —
  o login passava, mas a leitura do perfil falhava depois.
- **AMBOS SÃO PROVISÓRIOS.** A policy é `using(true)` (qualquer autenticado lê a equipe inteira) e
  o grant é amplo. **A Fase 2 tem que apertar os dois com `auth.uid()`** — ex.: cada pessoa lendo
  só a própria linha, e gestor lendo todas.

### Diagnóstico de RLS — a pegadinha que custa tempo
**O RLS não devolve erro quando barra: devolve ZERO linhas.** Ou seja, "policy/grant faltando" e
"linha realmente não vinculada" produzem exatamente o mesmo sintoma na tela ("sem perfil
vinculado"). Ao investigar, cheque a policy **e** o grant antes de suspeitar do vínculo.
Exceção útil: falta de **grant** dá erro explícito (`42501`); falta de **policy** dá silêncio.

## Tabela `montagem_treinos` (tela Treinos, 2026-08-02)
Fila de montagem de treino: a recepção/gestor cadastra o pedido, o professor monta e confirma.

### Schema
| coluna | tipo | obs |
|---|---|---|
| `id` | uuid | PK |
| `aluno_nome` | text | nome do aluno |
| `avaliador_id` | uuid **NOT NULL** | quem fez a avaliação (`equipe.id`) |
| `professor_id` | uuid **NOT NULL** | professor responsável por montar (`equipe.id`) |
| `data_solicitacao` | date **NOT NULL** | usa `fmtDate()` |
| `data_inicio_desejada` | date | opcional |
| `status` | text | `pendente` / `montado` / `cancelado` |
| `montado_por` | uuid | quem confirmou (`equipe.id`) |
| `data_montagem` | timestamptz | usa `fmtDataHora()`, não `fmtDate()` |
| `observacoes` | text | |
| `created_at` | timestamptz | |

### Acesso — diferente do resto do app, de propósito
- A tela usa **`sbClient.from()` (autenticado)**, e **não** os helpers `sbGet`/`sbPost`/`sbPatch`.
  Os helpers mandam a anon key fixa, e esta tabela **só concedeu grant ao role `authenticated`** —
  com a anon key a resposta é `401` / `42501 permission denied`. **Não copiar o padrão das telas
  vizinhas aqui.**
- O RLS **filtra as linhas**: professor recebe só as dele, gestor/recepção recebem todas. Por isso
  não há refiltro por perfil no JavaScript — e o `updateTreinosBadge()` conta em cima do que o
  banco já entregou. Duplicar a regra no cliente só criaria duas fontes de verdade.
- `podeCadastrarTreino()` (só gestor/recepção) é **conveniência de interface**, não segurança:
  quem garante é a policy de INSERT.
- `marcarMontado()` encadeia **`.select('id')`** no UPDATE (= `Prefer: return=representation`).
  Sem isso o PostgREST responde `204` vazio e um UPDATE recusado pelo RLS pareceria sucesso.
  Com isso, bloqueio vem como `200` + **zero linhas**, e o app avisa na tela.

### Policies de RLS — texto real (extraído de `pg_policies` em 2026-08-02)
São **cinco** policies, não quatro: o SELECT é dividido em duas. No Postgres, policies permissivas
da mesma operação se somam com `OR`, então "professor vê as dele" + "gestor/recepção veem todas" é
exatamente como se escreve isso — duas policies simples em vez de uma condição grande.

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

Leitura rápida do desenho:
- **Professor** lê e atualiza só as linhas onde `professor_id` é a linha dele na `equipe`. Não
  cadastra e não apaga.
- **Gestor/recepção** fazem tudo, via `EXISTS` na `equipe` com `perfil in ('gestor','recepcao')`.
- O `with check` do UPDATE repete o `using`, o que **impede o professor de reatribuir o treino para
  outro professor** (a linha resultante tem que continuar sendo dele). Gestor/recepção podem
  reatribuir, e isso é intencional.

### Grants
`authenticated`: `SELECT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER` (lidos na tela do painel).
`INSERT` não apareceu na leitura, mas **necessariamente existe**: policy não substitui grant, e o
`INSERT` do script de teste passou logado como gestor. Confirmar em definitivo com:
```sql
select grantee, string_agg(privilege_type, ', ' order by privilege_type)
  from information_schema.role_table_grants
 where table_schema='public' and table_name='montagem_treinos' group by grantee;
```
`anon`: **sem nenhum CRUD** — só `TRUNCATE` e `REFERENCES`, sobra da concessão padrão do Supabase
(ver o alerta no backlog: **TRUNCATE ignora RLS**).

### Validação (2026-08-02) — testado, não só lido
As policies foram verificadas por **script automatizado** (`teste-rls-treinos.js`, rodado fora do
repositório), logando de verdade com a **anon key** — nunca a service role, que ignora RLS e faria
qualquer teste passar. O script cria um treino descartável para o Professor B, loga como Professor
A e tenta agir sobre ele:
- **SELECT:** o professor A **não enxerga** o treino do professor B (zero linhas).
- **UPDATE:** o professor A **não consegue** confirmar o treino do professor B — `HTTP 200` com
  **zero linhas afetadas**, que é o bloqueio silencioso esperado da policy (não é erro `42501`,
  que indicaria grant faltando, problema diferente).
- Gestor/recepção seguem com acesso total, como antes.
- O teste criou e removeu um treino descartável e dois professores de teste (desativados com
  `ativo = false`), sem tocar em dado real de produção.

### Esta tabela é o padrão-alvo da Fase 2
`montagem_treinos` é a **primeira tabela do projeto com RLS de verdade** — habilitado, com policies
por operação e escopo derivado de `auth.uid()`, e validado por teste. **É o padrão que a Fase 2 deve
replicar em `eventos`, `rodizios`, `config` e `equipe`**, que hoje ainda estão abertas
(RLS desabilitado + `anon_all`, ou policy `using(true)` no caso da `equipe`). Ao fechar as outras,
copiar daqui as quatro decisões que fizeram diferença: (1) grant de tabela **além** da policy,
(2) policies pequenas por operação (e até duas na mesma operação, somadas com `OR`) em vez de um
`ALL` genérico, (3) `with check` repetindo o `using` no UPDATE, senão a pessoa pode transformar uma
linha sua numa linha de outro, (4) teste automatizado com a anon key provando o bloqueio, em vez de
leitura visual da policy.

**Detalhe que facilita a Fase 2:** todas as cinco policies consultam a `equipe` **apenas pela linha
de quem chamou** (`where auth_user_id = auth.uid()`). Ou seja, quando a Fase 2 apertar a `equipe`
para cada pessoa ler só a própria linha, **nenhuma destas policies quebra**. Isso não foi sorte e
não deve ser desfeito: qualquer policy nova que precise varrer a `equipe` inteira cria uma
dependência que trava justamente o aperto que a gente quer fazer.

## Aba Planejamento — Kanban (Fase 4, 2026-08-06)
Quadros com listas coloridas e cartões. Gestor cria e move; professor e recepção só leem, e só
enxergam os cartões liberados para eles. A aba aparece para **todos** os perfis.

### Schema (criado no painel antes da implementação; o app só consome)
| tabela | colunas |
|---|---|
| `quadros` | `id` uuid PK, `tema` text, `nome` text, `criado_por` uuid→`equipe.id`, `criado_em` timestamptz, `ativo` bool default true, **`data_inicio` date nullable**, **`data_fim_prevista` date nullable** (2026-08-07), **`privado` bool not null default false** (Fase 4.1, 2026-08-07) |
| `colunas` | `id` uuid PK, `quadro_id` uuid→`quadros.id` (cascade), `nome` text, `cor` text nullable, `ordem` int |
| `cards` | `id` uuid PK, `coluna_id` uuid→`colunas.id` (cascade), `titulo` text, `descricao` text nullable, `criado_por` uuid→`equipe.id`, `ordem` int, `criado_em`, `atualizado_em` |
| `card_visibilidade` | `id` uuid PK, `card_id` uuid→`cards.id` (cascade), `perfil` text nullable, `equipe_id` uuid nullable |

**Dois CHECKs que quebram o INSERT se ignorados:**
- `colunas.cor` só aceita `#3B82F6` (azul), `#8B5CF6` (roxo), `#EAB308` (dourado), `#22C55E`
  (verde), `#EF4444` (vermelho), `#64748B` (cinza) ou `NULL`. No JS a constante `PLAN_CORES` é a
  **única** fonte dessas cores — não escrever hex de coluna em nenhum outro lugar.
- `card_visibilidade` exige **exatamente um** de `perfil`/`equipe_id`. Nunca os dois na mesma linha,
  nunca nenhum. Por isso `sincronizarVisibilidade()` monta as linhas em dois `map` separados.

### Acesso — mesmo desvio deliberado da aba Treinos
- Usa **`sbClient.from()` (autenticado)**, nunca `sbGet`/`sbPost`/`sbPatch`/`sbDelete`. As policies
  são para o role `authenticated`; com a anon key dos helpers a resposta é zero linhas em silêncio.
- Toda escrita encadeia **`.select('id')`** — RLS que barra devolve `200` com zero linhas, não erro.
  Inclusive o DELETE de `card_visibilidade`, onde "barrado" e "não havia nada" são indistinguíveis:
  ali o código compara com a contagem que já tinha em memória.
- `podeEditarPlanejamento()` (só gestor) é **conveniência de interface, não segurança** — igual ao
  `podeCadastrarTreino()`. Quem garante é a policy.

### Modelo de visibilidade
O gestor marca um ou mais **perfis** e/ou uma ou mais **pessoas** da `equipe` (ativas).
**Sem nenhuma seleção o cartão fica visível só para gestores** — isso é o comportamento da policy
`cards_select`, não convenção da tela; o formulário diz isso em texto fixo.

O **cadeado** no cartão só aparece para o gestor, porque só ele lê `card_visibilidade` (policy
`vis_all` = `ALL using eh_gestor()`). O app **nem consulta** essa tabela quando não é gestor —
consultar traria zero linhas em silêncio, sem erro. Para professor/recepção o cadeado não faz falta:
todo cartão que eles enxergam é, por definição, um cartão liberado para eles.

### RLS — 7 policies e 4 funções `security definer`
Todas para `authenticated`. `quadros_select` e `colunas_select` são `using (true)` **de propósito**:
a estrutura do quadro não é segredo, o conteúdo é. Quem filtra é o `cards_select`:
```sql
eh_gestor() OR pode_ver_card(cards.id)
```
As demais (`quadros_write`, `colunas_write`, `cards_write`, `vis_all`) são `ALL` com
`using eh_gestor()` e `with check eh_gestor()`.

As 4 funções (`eh_gestor()`, `meu_equipe_id()`, `meu_perfil()`, `pode_ver_card(uuid)`) são
`security definer`, `STABLE` e `SET search_path TO 'public'`. **Não removê-las nem trocá-las por
consulta direta às tabelas:** como são `security definer`, elas ignoram o RLS de `equipe` e
`card_visibilidade`, e é isso que garante que **a Fase 2 possa apertar a `equipe` sem quebrar
nenhuma policy do Planejamento**. O `search_path` fixo também não é enfeite — sem ele,
`security definer` vira vetor de escalada de privilégio.

### ⚠️ REGRA: policy que consulta outra tabela com RLS precisa de `security definer`
**Nunca `EXISTS` direto contra outra tabela protegida por RLS dentro de uma policy.** Aprendido na
marra na Fase 4 (2026-08-06), e está no mesmo nível do "policy não substitui grant" da Fase 1.

**O que aconteceu:** a `cards_select` original fazia
`EXISTS (select 1 from card_visibilidade v where v.card_id = cards.id and (...))`. A subconsulta
**herda o RLS de `card_visibilidade`**, que só tem a `vis_all` (`ALL using eh_gestor()`). Para quem
não é gestor, a subconsulta via zero linhas, o `EXISTS` nunca achava nada, e professor/recepção não
viam cartão algum — **com o dado 100% correto na tabela e sem nenhum erro em lugar nenhum**.

É o "RLS barra em silêncio" de sempre, só que **aninhado dentro de outra policy**: a policy que
falha não é a policy culpada, e a tabela que você está depurando não é a que está barrando. Por isso
é caro de achar.

**Correção:** função `pode_ver_card(p_card_id uuid)` `security definer` encapsulando a checagem, e
policy virou `eh_gestor() OR pode_ver_card(cards.id)`. Confirmado com conta real (Leonardo, perfil
`recepcao`), não por leitura de policy.

**Com `eh_gestor`, `meu_equipe_id`, `meu_perfil` e `pode_ver_card` são quatro casos: isto é padrão
do projeto, não exceção.**

**Consequência direta para a Fase 2 — não ignorar:** as policies da `montagem_treinos` fazem
`EXISTS` direto contra a `equipe`. Hoje funcionam só porque a `equipe` tem policy permissiva
(`using(true)`). **No dia em que a Fase 2 apertar a `equipe`, aquelas policies passam a ter
exatamente este bug, em silêncio, e a aba Treinos para de mostrar linha para todo mundo.**
Converter a `montagem_treinos` para `security definer` **antes** de mexer na `equipe`, nunca depois.

### Prazo do quadro (2026-08-07) — só app/UI, nenhuma policy tocada
`data_inicio` e `data_fim_prevista` são **opcionais**. Sem as **duas** preenchidas o quadro não
mostra badge nenhum e o card fica idêntico ao de antes. Só uma das datas é permitido (as colunas
são independentes); as duas invertidas são recusadas no modal. Campo vazio grava **NULL**, e as
datas vão no UPDATE mesmo nulas — é o que permite limpar um prazo.

**Não existe um modal "Editar quadro" separado.** O `#quadro-modal` serve criação e edição desde a
Fase 4; a edição agora tem duas portas: o `✏️` no card da lista e o "✏️ Editar" dentro do quadro
aberto. Não criar um segundo modal — seriam duas fontes de verdade para a mesma tabela.

Os campos usam `input type="date"` **nativo**, e não o date picker customizado (`dpDates`/`dpCals`):
o picker é para data única obrigatória, e duas chaves novas nos globais dele seria mais risco que
ganho. Mesmo precedente do `type="time"` no modal de evento.

Cinco estados, em `statusPrazoQuadro()`:

| condição | badge |
|---|---|
| falta uma das datas | nenhum |
| hoje < início | "Ainda não começou" (neutro) |
| dentro do período | "Em andamento" + barra ("3 de 6 semanas" ou "7 de 31 dias") |
| passou do fim, sobrou cartão fora da última lista | "⚠️ Atrasado" (vermelho) |
| passou do fim, nada pendente | "Encerrado" (neutro) |

"Concluído" segue **derivado da posição** (última lista), como o `.plan-fim` — `cards` continua sem
campo de status. O período é **inclusivo** nas duas pontas: 20/07 a 30/08 = 42 dias = 6 semanas.
Reusa a `diasEntre()` dos Relatórios (imune a fuso); comparação de datas é string-compare, que em
`YYYY-MM-DD` ordena certo.

**⚠️ A contagem de pendentes passa pelo RLS e pode divergir entre perfis.** A
`carregarPendenciasVencidos()` só roda para quadros já vencidos (no caso comum, **zero consultas**)
e conta em cima do que o `cards_select` entregou **àquele usuário**. Um cartão pendente não
liberado para o professor não entra na conta dele, então o mesmo quadro pode ser "Encerrado" para
ele e "Atrasado" para o gestor. Não é bug: contar cartão invisível vazaria a existência dele.
Decisão do Felipe (2026-08-07): o badge aparece para **todos os perfis** mesmo assim.

**Falha na conferência nunca vira "Encerrado".** Erro na consulta ou truncamento do PostgREST
(corte em 1000 linhas) → o quadro é mostrado como "Atrasado" **sem contagem** e o motivo vai pro
console. Falso alarme é muito menos grave que dizer "encerrado" para algo pendente.

### Quadro privado (Fase 4.1, 2026-08-07)
Quadro marcado como privado **só aparece para quem o criou — inclusive outros gestores deixam de
ver**. O RLS já estava em produção e validado com contas reais antes da UI; a Fase 4.1 no
`index.html` **não criou nem alterou policy, função ou tabela**.

Já existiam no banco: `quadros.privado`, a função `security definer` `posso_ver_quadro(uuid)`, e as
**7 policies do Kanban já reescritas em cima dela**. Não mexer nelas sem decisão explícita.

Na tela: checkbox no `#quadro-modal` (o mesmo que serve criação e edição), selo `🔒 PRIVADO` na
linha do tema, e `privado` nos dois payloads. No UPDATE o campo vai **mesmo quando `false`** — é o
que permite voltar o quadro a público. Não há refiltro por `privado` no JS: quadro privado alheio
nem chega no array, quem filtra é a `quadros_select`.

**⚠️ TRAVA: só o CRIADOR mexe no `privado`.** A `quadros_write` é
`eh_gestor() AND posso_ver_quadro(id)`, então qualquer gestor pode editar quadro público alheio —
e marcá-lo privado faria o quadro **sumir da lista dele no instante do save**, sem desfazer,
porque perderia o acesso junto. Por isso `souCriadorDoQuadro(q)` (compara `criado_por` com
`currentEquipeId`): para não-criador o checkbox fica **desabilitado e explicado** (não escondido) e
`privado` fica **fora do payload**, deixando a coluna intocada. Criação nunca trava.
**Uma função só para o modal e o save** — checagem duplicada divergiria num refactor e o campo
voltaria a escapar. Cuidado ao mexer: o `!!` da função existe para impedir que `criado_por` nulo
case com `currentEquipeId` nulo e libere a trava para quem não deveria.

**É conveniência de interface, não segurança** — igual ao `podeEditarPlanejamento()`. Pelo DevTools
ainda dá para marcar privado um quadro alheio. Fechar exigiria `WITH CHECK` na `quadros_write`
(`criado_por = meu_equipe_id()` quando `privado = true`); **decisão do Felipe: avaliar na Fase 2**,
não agora.

**Pendência de teste (ver backlog):** falta confirmar que a trava não atrapalha a edição de
tema/nome/datas do mesmo modal por um não-criador.

### Regra visual (exceção deliberada — não "corrigir")
- **Os cartões são SÓLIDOS escuros (`var(--g1)`), nunca vidro, nunca na cor da lista.** Só levam uma
  faixa de 3px no topo na cor dela. Mesma regra dos inputs do Login/Config e do antigo keypad do
  PIN: sobre o fundo laranja, onde se lê o conteúdo não leva vidro.
- O **vidro** fica nas colunas, com os tokens de `#page-treinos` (blur 3px + saturate 180%, borda
  `1px rgba(255,255,255,0.4)`, raio 16px, linha de highlight no topo). Nenhum valor novo foi criado.
- O cabeçalho de cada lista é faixa **sólida opaca** na cor, com texto no tom escuro da mesma
  família (vem do `PLAN_CORES`, aplicado inline).
- Cartões na **última coluna** ficam com `opacity:0.6` (mesmo valor do `.treino-card.st-cancelado`).
  **"Concluído" é derivado da posição, não de um campo** — a tabela `cards` não tem status e a
  Fase 4 não mexeu em schema. Se um dia virar campo de verdade, é decisão a tomar, não detalhe.
- **Mobile não tem drag** (brigaria com o scroll): cada cartão usa o menu `⋮` com "Mover para →".
  Detecção por `matchMedia('(pointer:coarse)')` + largura, **nunca por user-agent**.
- **Badges de prazo não usam `PLAN_CORES`** — aquilo é exclusivo de cor de lista, com CHECK no
  banco. Os três estilos vêm da aba Treinos, com a mesma semântica: **branco sólido = pede
  atenção** (`.treino-status.st-pendente`), **vidro contornado = encerrado**
  (`.treino-status.st-montado`), **vermelho translúcido = alerta** (os 3 valores do `.cfg-danger`).
  O card atrasado leva **borda E fundo** vermelhos: sobre o gradiente laranja a borda sozinha não
  se distingue da branca padrão, então "destaque duplo" só existe de fato com o fundo junto.

## Navegação — menu lateral (drawer), 2026-08-10
A barra de abas **não existe mais**. A navegação é um drawer que desliza da esquerda, aberto pelo
chip escuro no canto superior esquerdo da topbar. **O mesmo componente vale no desktop e no
mobile** — duas navegações seriam duas fontes de verdade, e toda página nova teria que ser
cadastrada em dois lugares.

Motivo da troca: com 7 abas a `.tb-nav` já obrigava a topbar a quebrar em duas linhas no celular,
com scroll horizontal no menu. Cada aba nova piorava isso; o drawer não muda de forma conforme o
app cresce.

### `NAV_GRUPOS` é a fonte única dos itens
Página nova = **uma entrada no `NAV_GRUPOS`**, sem tocar em CSS nem no `goPage()`. Mesmo
precedente do registry `RELATORIOS`. Três grupos com rótulo visual, **sem accordion** (com 7 itens
cabe tudo visível; accordion só somaria um clique e esconderia opção):

| grupo | itens |
|---|---|
| Agenda | Agenda, Calendário, Rodízio |
| Operação | Treinos, Planejamento |
| Gestão | Relatórios, Config. — os dois com `soGestor:true` |

**Grupo sem nenhum item visível para o perfil não é renderizado — o título junto.** Não é regra
teórica: "Gestão" só tem itens de gestor, então **some inteiro para professor e recepção**.
Título de grupo sozinho, sem item embaixo, é bug visual.

O `soGestor` substituiu os `style.display` que o `enterApp()` fazia nos botões de aba. Como antes,
**é conveniência de interface, não segurança** — igual ao `podeCadastrarTreino()` e ao
`podeEditarPlanejamento()`. Quem garante é o RLS.

### Detalhes que quebram se mexidos sem cuidado
- **O drawer começa ABAIXO da topbar, e o `top` é medido em JS** (`posicionarDrawer()`), não é
  fixo. Dois motivos: (1) o painel é de vidro, e vidro sobre a topbar **branca** apaga o texto;
  (2) o `.notif-banner` aparece e some, mudando a altura. A medida usa
  `topbar.offsetTop + offsetHeight` (offset **dentro** do `#screen-app`) e **não**
  `getBoundingClientRect()`: a `.screen` tem `will-change:transform`, o que a torna o containing
  block deste `position:fixed`, e durante a transição de entrada o rect de viewport chega a dar
  negativo. Já aconteceu — o drawer abriu a `-889px`.
- **Os ITENS do drawer são sólidos escuros (`var(--g1)`), nunca vidro.** Mesma exceção deliberada
  dos cartões do Kanban e dos inputs do Login/Config: onde se lê conteúdo não leva vidro. Aqui não
  é só estética — com vidro dentro de vidro o texto da página atrás **vaza por dentro** do item.
  O painel continua de vidro; só os itens é que não.
- **A trava de scroll age na `.app-content`, não no `body`** — o `body` já é `overflow:hidden` e
  quem rola de verdade é ela. O `scrollTop` é salvo e restaurado em JS porque alternar `overflow`
  pode zerá-lo.
- `.modal-cls.drawer-cls` usa **seletor duplo de propósito**: a `.modal-cls` é declarada bem
  depois no arquivo e venceria o empate de especificidade, deixando o ✕ cinza sobre o vidro.
  Mesmo motivo do `.btn-cancel.plan-btn-del`.
- O `☰` do menu de ações (Senha/Exportar/Sair) no mobile **virou `⋯`**: o `☰` agora é o chip de
  navegação, e dois ícones iguais com funções diferentes na mesma barra confundem. As ações não
  saíram do lugar.

### Nome da página na topbar
Onde ficava a barra de abas. É **orientação visual**: não é clicável, não tem hover, e usa o cinza
discreto que as abas tinham quando **não** estavam ativas — para não competir com o item ativo do
drawer, que é branco sólido. O texto vem do `rotuloPagina()`, que lê o `NAV_GRUPOS`: **nome de
página não pode existir em dois lugares.**

**No celular a marca "EVOLUÇÃO" sai da topbar** e o nome da página ocupa o lugar dela. Medido em
390px, os dois não cabem (~409px necessários para 366px úteis). Decisão do Felipe (2026-08-10):
entre o que nunca muda (nome da empresa) e o que muda (onde estou), fica o segundo. A marca
continua no desktop, no splash e no login. É isso que mantém a topbar mobile em **uma linha
(51px)** e o que torna correto o `min-height:calc(100vh - 56px)` das telas de vidro — a
compensação de `100px`, que existia por causa da segunda linha, foi removida.

## Incidente conhecido (jul/2026)
- Supabase free tier pausa após 7 dias de inatividade. O app engolia o erro de conexão
  silenciosamente (catch(e){return null}) e mostrava "0 eventos" em vez de avisar — parecia que
  os dados tinham sumido, mas era só o projeto pausado (dados sempre ficaram intactos).
- Mitigação 1: workflow .github/workflows/ping-supabase.yml pinga o Supabase 2x/semana pra evitar
  a pausa.
- Mitigação 2 (concluída em 2026-07-03): banner fixo vermelho no topo (`#conn-error-banner`)
  avisa "Não foi possível conectar ao banco de dados..." sempre que sbGet/sbPost/sbPatch/
  sbDelete falhar por erro de rede ou resposta não-ok. Tem botão de fechar (X), mas reaparece
  se uma nova chamada falhar depois.

## Pendências (backlog)
- **⚠️ Fase 4.1 subiu para produção com uma ressalva ABERTA (2026-08-07) — testar antes de confiar.**
  **Não foi testado se a trava do checkbox "Privado" (para não-criador) interfere na edição normal
  dos outros campos (tema/nome/datas) do mesmo modal.** Testar na próxima sessão, com a conta de um
  segundo gestor, antes de confiar 100% na Fase 4.1 em uso real por quem não criou o quadro.
  A leitura do código sugere que **não** interfere (o `disabled` age só no próprio checkbox; os
  demais campos são lidos à parte no `saveQuadro()`; e num quadro público a `posso_ver_quadro()`
  devolve `true` para qualquer gestor, então a `quadros_write` aceita o UPDATE). **Mas isso é
  raciocínio sobre o código, não observação** — e aqui RLS que barra devolve zero linhas em
  silêncio, não erro. Só conta como testado depois de rodar com conta real.
- **Passo 6 da aba Treinos (PDF) — pausado em 2026-08-03, sem prazo de retomada.** Decisão do
  Felipe: outro projeto tem prioridade, e ele volta a este quando puder. **Não é urgência e não
  bloqueia nada** — a aba Treinos está completa e em uso até o Passo 5 (Relatórios).
  **Atenção ao retomar:** o escopo do Passo 6 nunca foi escrito em lugar nenhum — a numeração dos
  passos só existiu em conversa, e o que ficou registrado é apenas "PDF", presumivelmente exportar
  o relatório de Treinos por Professor (o Passo 5, commit `bf9a431`). **Confirmar com o Felipe o
  que o PDF deve conter antes de começar a implementar**, em vez de deduzir pelo nome.
  Ponto de partida: `bf9a431` é o último commit da sequência.
- **Fase 2 — fechar o banco (não feita).** Hoje `eventos`, `rodizios` e `config` continuam
  abertos: qualquer pessoa com a anon key (que é pública por design) lê e escreve via DevTools.
  A Fase 1 resolveu *quem é você*, não *o que você pode fazer* — o `currentRole` no JavaScript é
  cosmético. Inclui apertar a policy e o grant provisórios da `equipe` (ver acima).
  **Modelo a seguir: as tabelas da Fase 4** (`quadros`/`colunas`/`cards`/`card_visibilidade`) —
  usam funções `security definer` e por isso não quebram quando a `equipe` for apertada. A
  `montagem_treinos` (2026-08-02) foi o primeiro acerto, mas consulta a `equipe` por `EXISTS`
  direto: **ela vai quebrar em silêncio quando a `equipe` for fechada.** Ver a regra
  "policy que consulta outra tabela com RLS precisa de `security definer`" na seção do
  Planejamento — e converter a `montagem_treinos` ANTES de mexer na `equipe`.
- **`anon` ainda tem `TRUNCATE` na `montagem_treinos` — revogar.** Detectado em 2026-08-02 ao ler os
  grants. O CRUD do `anon` já foi tirado, mas `TRUNCATE` e `REFERENCES` ficaram como sobra da
  concessão padrão do Supabase. Importa porque **`TRUNCATE` não passa por RLS**: policy nenhuma
  protege contra ele. Hoje não há caminho de exploração conhecido (o PostgREST não expõe verbo de
  TRUNCATE pela Data API), então é risco latente, não buraco aberto — mas é privilégio sem nenhum
  uso legítimo para o `anon`, e a correção é uma linha:
  `revoke truncate, references on public.montagem_treinos from anon;`
  Conferir o mesmo nas outras tabelas ao fazer a Fase 2.
- **O UPDATE da `montagem_treinos` não é restrito por coluna.** A policy diz *quais linhas* o
  professor pode alterar, não *quais campos*. Na prática ele pode reescrever `aluno_nome`,
  `observacoes`, `avaliador_id` ou datas da própria linha, e pode gravar `montado_por` apontando
  para outra pessoa (a tela mostraria "montado por Fulano" sem ter sido). Não é escalada de
  privilégio — é integridade de dado. Só dá pra fechar com trigger ou grant por coluna; avaliar na
  Fase 2 se vale a complexidade num app de uso interno.
- **`anon` também tem `TRUNCATE` nas 4 tabelas da Fase 4 — revogar junto.** Mesma origem (default do
  Supabase) e mesma gravidade do item acima: **`TRUNCATE` não passa por RLS**. Decisão do Felipe em
  2026-08-06: **não revogar agora**, deixar para a Fase 2. A correção é uma linha:
  `revoke truncate on quadros, colunas, cards, card_visibilidade from anon;`
- **`docs/schema.sql` não contém `montagem_treinos` nem as 4 tabelas da Fase 4; conferir no Table
  Editor antes de confiar nele.** Além disso declara `equipe.id` como
  `bigint generated by default as identity` — mas as chaves em uso hoje são UUID
  (ex.: `ced151d4-9f3e-4930-b830-befaa003d406`), e `montagem_treinos.professor_id` é `uuid`.
  Ou a coluna mudou de tipo em algum momento sem o arquivo acompanhar, ou o arquivo nunca esteve
  certo. Hoje o arquivo descreve um banco que não existe mais — atualizá-lo é pendência de
  auditoria. Detectado em 2026-08-02, ampliado em 2026-08-06.
- Coluna `pin` da tabela `config` virou **dado morto** — nada no app lê ou escreve nela desde
  2026-07-31. Não foi removida (a Fase 1 não mexeu em schema além da `equipe`). Candidata a
  `drop column` na Fase 2, junto com o resto do hardening.
- Link CSS quebrado na linha ~12 do index.html (`<link href="./index_files/css2">`, resto do
  "save as" original): a fonte Inter nunca carrega e o app usa a fonte padrão do sistema.
  É a origem do 404 de CSS que aparece no console. Corrigir trocando por link real do Google
  Fonts (ou embutir a fonte). Decidido em 2026-07-13, não mexer sem combinar.
- ~~Testar a tela de Calendário (liquid glass) em celular de verdade~~ — feito em 2026-07-13,
  scroll OK em aparelho físico; o estilo foi estendido pra tela de Agenda na sequência.

## Como trabalhar comigo neste projeto
- **NUNCA rodar teste de função que ESCREVE (`updateUser`, `signOut`, `sbPost`/`sbPatch`/
  `sbDelete`, `clearAllData`) enquanto houver sessão real ativa no navegador.** Aconteceu de
  verdade em 2026-07-31: ao testar as validações do formulário de trocar senha, o caso "senha
  válida" foi executado com a sessão do Felipe viva no `localStorage` e **trocou a senha real
  dele** — foi preciso resetar pelo painel. A sessão sobrevive a F5 e a `enterApp()` forçado por
  console, então "não fiz login agora" não quer dizer nada.
  Antes de qualquer teste desse tipo: rodar `await sbClient.auth.getSession()` e confirmar que
  voltou `null`; ou testar só os ramos que retornam ANTES da chamada à API; ou usar uma conta
  descartável. **Esse cuidado vale em dobro com a conta de um professor ou recepcionista** — ali
  o estrago não é só um reset, é alguém sem acesso no meio do expediente sem saber por quê.
- Antes de editar o index.html, leia o arquivo inteiro primeiro — é um arquivo único e grande,
  fácil de quebrar algo em outro lugar sem perceber.
- Depois de qualquer mudança, me diga claramente o que mudou e por quê, em português, sem jargão
  desnecessário.
- Sempre sugira o commit assim que uma etapa funcionar — não deixe trabalho sem versionar.
- Se identificar outros riscos parecidos com o da pausa do Supabase (silent failures, dependências
  externas sem fallback, etc.), aponte proativamente mesmo que eu não tenha perguntado.
- Nunca coloque senhas ou chaves sensíveis em arquivos texto dentro do repositório (evitar recriar
  o hábito do SENHAS.txt).
