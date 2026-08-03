---
name: agenda-evolucao
description: >
  Convenções obrigatórias do app Agenda Evolução (EvoluçãoFit). Use SEMPRE que
  for criar, editar ou revisar qualquer tela, componente ou estilo do index.html,
  e sempre que for criar tabelas, policies de RLS ou mudanças no Supabase deste
  projeto. Cobre o design system "liquid glass", a arquitetura sem-build e as
  regras de banco. Se a tarefa toca a UI ou o Supabase do Agenda, esta skill se aplica.
---

# Agenda Evolução — design system e convenções

App interno de agendamento da EvoluçãoFit. Um único `index.html` estático, sem
framework e sem build, que conversa direto com o Supabase (ref `indapxewhfobzxnwswhy`)
via REST. Perfis: Professor, Recepção, Gestor.

## Regra de ouro

**Nunca "derive" o visual. Preserve o que já existe.**

Este design tem decisões deliberadas que parecem inconsistências mas NÃO são —
elas estão listadas abaixo como exceções conscientes. Ao criar uma tela nova,
copie os padrões das telas existentes em vez de reinventar. Se uma escolha sua
diverge do padrão, ela precisa ser uma decisão consciente registrada no
`CHANGELOG.md`, não um acidente.

## A fonte da verdade dos valores é o CSS, não esta skill

Esta skill descreve **como** o visual funciona, não os números exatos.

Os valores canônicos (cores, opacidades, raios, intensidade de blur) vivem no
`<style>` do `index.html`. **Antes de escrever qualquer CSS novo, leia os tokens
do arquivo atual e reutilize-os.** Nunca invente um `rgba()`, um raio de borda ou
um valor de blur "de cabeça" — se o valor não estiver claro no CSS existente,
pare e pergunte ao Felipe em vez de chutar.

Se o projeto ainda não tiver os tokens centralizados num `:root` (custom
properties como `--glass-bg`, `--sun-core`, `--blur-card`), proponha centralizar
antes de seguir — isso torna todo o resto desta skill confiável. Não faça essa
refatoração sem confirmar.

## O design "liquid glass"

A metáfora é luz do sol atravessando vidro fosco. Três camadas:

**Fundo — o sol.** Gradiente radial partindo de um núcleo alaranjado, escurecendo
para quase-preto e terminando em `#000` nas bordas. É a única fonte de "luz" da
tela; tudo mais reage a ela.

**Cards de vidro.** Fundo semitransparente + `backdrop-filter: blur(3px)
saturate(180%)`. No topo de cada card há uma linha de destaque (highlight) clara
simulando a refração da luz na borda superior do vidro. Os cards deixam o gradiente
do fundo vazar por trás — essa transparência é o efeito, não a esconda.

**Conteúdo.** Fica sobre o vidro, com contraste suficiente para ler sobre o fundo
translúcido.

## Exceções deliberadas (NÃO "corrija" nenhuma delas)

- **Modais usam blur mais forte que os cards** e bordas com destaque multicolorido.
  Isso é intencional: distingue o modal (camada de cima, foco) do conteúdo normal.
  Não iguale o blur do modal ao dos cards "por consistência".
- **Os botões do teclado numérico (keypad) são sólidos/opacos**, não de vidro.
  Legibilidade acima de estética: número tem que ler de primeira. Todo keypad,
  em qualquer tela, segue essa regra.

## Arquitetura — o que não pode quebrar

- **Sem build, sem framework.** Dependências entram só por tag `<script>`/`<link>`
  via CDN. Não introduza npm, bundler ou etapa de build.
- **`supabase-js` via CDN** é a biblioteca oficial de acesso (auth + REST). Use os
  métodos dela; não reescreva chamadas de auth na mão com `fetch` puro.
- **Autenticação:** login é por usuário + senha via Supabase Auth. O identificador
  interno vira email de um domínio próprio da equipe montado pelo app — o usuário
  nunca digita nem vê email, só o login dele. Confirmação por email fica desligada.
- O papel (Professor/Recepção/Gestor) é resolvido pela pessoa autenticada
  (`auth.uid()`) via tabela `equipe`, não por PIN no cliente.

## Banco de dados e RLS

- **Toda tabela nova declara sua abordagem de RLS explicitamente** no `CLAUDE.md` e
  no `CHANGELOG.md`. O projeto tem histórico de padrão misto (algumas tabelas com
  RLS desligado + `anon_all`, outras com RLS ligado + policy explícita); por isso a
  abordagem de cada tabela precisa estar escrita, nunca implícita.
- **Segurança de verdade mora em policy de RLS no servidor, não em `if` no
  JavaScript.** Se uma regra ("professor só vê o que é dele") importa, ela vira
  policy usando `auth.uid()`. Regra só no cliente é cosmética e pode ser burlada
  pelo DevTools.
- **Audita antes de executar.** Antes de rodar `UPDATE`/`DELETE`/`ALTER` que toca
  muitas linhas, primeiro faça o `SELECT` que mostra o que será afetado, mostre ao
  Felipe, e só então rode a alteração. Tenha o SQL de rollback escrito antes.
- Ao trocar policies numa tabela que o app usa em produção, faça **uma tabela por
  vez, testando entre cada uma**, fora do horário de pico.

## Fluxo de trabalho com o Felipe

- Uma etapa por vez. **Não avance para a próxima sem ele confirmar que a anterior
  está commitada e testada no celular.** Ele tende a pular etapas quando empolga —
  segure o ritmo.
- Antes de commitar mudança visual, gere um screenshot pra ele aprovar.
- Feedback honesto. Se um pedido dele for problemático (segurança, perda de dado,
  descaracterização do design), diga e explique o porquê antes de executar.

## Checklist antes de commitar uma tela

1. Reutilizei os tokens do CSS existente em vez de inventar valores?
2. Card tem a linha de highlight no topo e deixa o fundo vazar?
3. Modal manteve o blur mais forte e as bordas multicoloridas?
4. Keypad (se houver) ficou sólido/opaco?
5. Tabela nova teve a abordagem de RLS documentada no CLAUDE.md e CHANGELOG.md?
6. Regra de acesso importante virou policy no banco, não `if` no cliente?
7. Screenshot gerado pra aprovação antes do commit?
