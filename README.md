# Agenda Evolução

App interno de agendamento da EvoluçãoFit (academia em São José dos Campos-SP).

Produção: [agenda-evolucao.vercel.app](https://agenda-evolucao.vercel.app)

## O que é

Uma agenda para a equipe controlar reuniões, entrevistas, escala de revezamento de fim de
semana e atas de reunião, com três perfis de acesso: **Professor**, **Recepção** e **Gestor**
(protegido por PIN). Instalável como PWA no celular.

## Stack

Este projeto **não** usa Next.js, nem qualquer framework ou build step. É um único arquivo
[`index.html`](./index.html) com HTML, CSS e JavaScript vanilla inline.

- **Frontend**: `index.html` puro, sem dependências de bundler.
- **Backend**: [Supabase](https://supabase.com) (projeto `indapxewhfobzxnwswhy`), acessado
  direto do navegador via REST API (`/rest/v1/...`) usando a *anon key* pública embutida no HTML.
- **Hospedagem**: Vercel, como site estático (sem `vercel.json`, sem comando de build).

## Rodando localmente

Não há `npm install` nem servidor de dev necessário — é só abrir o arquivo:

```bash
# qualquer servidor estático simples serve, por exemplo:
npx serve .
```

Ou simplesmente abra `index.html` direto no navegador. Como o backend é o Supabase de produção,
qualquer alteração feita localmente já lê/grava nos dados reais — não existe ambiente de
"desenvolvimento" separado hoje.

## Banco de dados

Ver [`docs/schema.sql`](./docs/schema.sql) para a estrutura das tabelas (`eventos`, `rodizios`,
`equipe`, `config`).

## Aviso: pausa do Supabase (free tier)

O projeto Supabase usado aqui é do plano gratuito, que **pausa automaticamente após 7 dias sem
uso**. Se o app parecer estar com "0 eventos" do nada, é provável que o projeto esteja pausado —
os dados continuam intactos, só precisam ser reativados no painel do Supabase. Veja mais detalhes
e mitigações em [`CLAUDE.md`](./CLAUDE.md).

## Contexto para IA / desenvolvimento

Este repositório tem um [`CLAUDE.md`](./CLAUDE.md) com contexto de projeto, preferências de
trabalho e incidentes conhecidos, usado pelo Claude Code ao assistir no desenvolvimento.
