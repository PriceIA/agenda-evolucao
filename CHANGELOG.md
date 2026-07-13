# Changelog

Todas as mudanças notáveis deste projeto serão documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/).

> Nota: o histórico de commits anterior a este arquivo não tinha mensagens descritivas
> ("Update index.html" repetido), então as entradas abaixo anteriores a 2026-07 foram
> reconstruídas a partir do estado atual do código, não do histórico do Git.

## [Unreleased]

### Adicionado
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

### Alterado
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
