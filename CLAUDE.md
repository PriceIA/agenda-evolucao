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
- Três perfis de acesso: Professor, Recepção, Gestor (com PIN).
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
- Link CSS quebrado na linha ~12 do index.html (`<link href="./index_files/css2">`, resto do
  "save as" original): a fonte Inter nunca carrega e o app usa a fonte padrão do sistema.
  Corrigir trocando por link real do Google Fonts (ou embutir a fonte). Decidido em 2026-07-13,
  não mexer sem combinar.
- Testar a tela de Calendário (liquid glass, 2026-07-13) em celular de verdade após o deploy,
  pra confirmar que o backdrop-filter não pesa no scroll em aparelhos simples.

## Como trabalhar comigo neste projeto
- Antes de editar o index.html, leia o arquivo inteiro primeiro — é um arquivo único e grande,
  fácil de quebrar algo em outro lugar sem perceber.
- Depois de qualquer mudança, me diga claramente o que mudou e por quê, em português, sem jargão
  desnecessário.
- Sempre sugira o commit assim que uma etapa funcionar — não deixe trabalho sem versionar.
- Se identificar outros riscos parecidos com o da pausa do Supabase (silent failures, dependências
  externas sem fallback, etc.), aponte proativamente mesmo que eu não tenha perguntado.
- Nunca coloque senhas ou chaves sensíveis em arquivos texto dentro do repositório (evitar recriar
  o hábito do SENHAS.txt).
