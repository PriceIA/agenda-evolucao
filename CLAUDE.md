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
- **Fase 2 — fechar o banco (não feita).** Hoje `eventos`, `rodizios` e `config` continuam
  abertos: qualquer pessoa com a anon key (que é pública por design) lê e escreve via DevTools.
  A Fase 1 resolveu *quem é você*, não *o que você pode fazer* — o `currentRole` no JavaScript é
  cosmético. Inclui apertar a policy e o grant provisórios da `equipe` (ver acima).
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
