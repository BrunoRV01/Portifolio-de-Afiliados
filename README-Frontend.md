# Frontend — Painel de Afiliados (React + Vite)

Dashboard visual conectado ao backend FastAPI: login, cadastro de produtos
afiliados, métricas de tráfego/lucratividade e tendência de vendas.

## Direção visual

Conceito "mesa de operações": fundo azul-marinho profundo, números em fonte
monoespaçada (como um painel de cotações), com um **ticker rolante no topo**
mostrando cada produto e sua tendência (alta/queda) — o elemento de
assinatura da interface, que conecta visualmente a ideia de "acompanhar
produtos" com a de "acompanhar ativos".

- Cores: `#0F1420` fundo · `#3ECF8E` alta · `#F2637B` declínio · `#F5B841` destaque
- Tipografia: Space Grotesk (títulos) · Inter (corpo) · IBM Plex Mono (números)

## Estrutura

```
frontend/
├── src/
│   ├── api/
│   │   ├── client.js       # axios com JWT + refresh automático
│   │   ├── products.js     # produtos, paginação, upload de imagem, import CSV
│   │   ├── integrations.js # busca automática de produto + sincronização
│   │   ├── webhooks.js     # CRUD de webhooks
│   │   ├── assistant.js    # chat com o assistente de IA
│   │   └── errors.js       # leitura padronizada de erro da API
│   ├── context/
│   │   └── AuthContext.jsx # login, registro, logout, usuário atual
│   ├── components/
│   │   ├── Ticker.jsx       # ticker rolante (assinatura visual)
│   │   ├── Sidebar.jsx      # navegação lateral
│   │   ├── KpiCard.jsx      # cards de indicadores
│   │   ├── TrendPieChart.jsx  # gráfico de pizza (alta/estável/declínio)
│   │   ├── TrafficChart.jsx   # gráfico de barras (cliques x conversões)
│   │   ├── ProductsTable.jsx  # tabela de produtos, com upload de foto por linha
│   │   ├── ProductFormModal.jsx # formulário de novo produto (+ busca automática)
│   │   ├── WebhookFormModal.jsx # formulário de novo webhook
│   │   ├── ChatWidget.jsx     # widget de chat flutuante com a IA
│   │   └── ProtectedRoute.jsx  # rota de layout: bloqueia acesso sem login
│   ├── pages/
│   │   ├── Login.jsx
│   │   ├── Register.jsx
│   │   ├── Dashboard.jsx    # painel com KPIs + gráficos
│   │   ├── Products.jsx     # gestão de produtos (paginada, sync, import CSV)
│   │   ├── OrgChart.jsx     # organograma: Portfólio → Marketplace → Tendência
│   │   └── Webhooks.jsx     # gestão de webhooks
│   ├── App.jsx               # rotas
│   └── main.jsx
├── index.html
├── package.json
├── vite.config.js
├── Dockerfile
└── nginx.conf
```

## Como rodar

**Pré-requisito:** o backend precisa estar rodando em `http://localhost:8000`
(veja o README da pasta `backend/`).

```bash
cd frontend
npm install
npm run dev
```

Abre em `http://localhost:5173`. As chamadas para `/api/*` são
automaticamente redirecionadas para o backend (configurado em
`vite.config.js`), então não precisa mudar nada para desenvolver localmente.

## Fluxo

1. Crie uma conta em `/registro` (chama `POST /api/v1/auth/register` e já
   faz login).
2. No Painel (`/`), veja o ticker de tendências, os KPIs (produtos,
   cliques, conversões, receita estimada, taxa de conversão) e os gráficos.
3. Em Produtos (`/produtos`), cadastre produtos com link original + link
   de afiliado, e navegue entre páginas (10 por página). O link de
   afiliado real deve ser gerado nos programas Amazon Associates / Shopee
   Affiliate — ainda não integrado automaticamente, veja `backend/README.md`.
4. Ao sair (botão "Sair" na barra lateral), o refresh token é revogado no
   servidor (`POST /api/v1/auth/logout`), não só apagado localmente.

## Atualizado para o backend v2

- `baseURL` do axios agora aponta para `/api/v1` (rotas versionadas).
- `fetchProducts()` lida com a resposta paginada (`{ items, total, limit,
  offset }`) em vez de um array simples.
- `Products.jsx` ganhou paginação real (10 itens por página, com botões
  Anterior/Próxima) em vez de listar tudo de uma vez.
- O Painel usa `stable_products` e `conversion_rate`, já calculados pelo
  backend, em vez de derivar esses números no frontend.
- `logout()` chama `POST /auth/logout` para revogar o refresh token no
  servidor (best-effort — o logout local acontece de qualquer forma).
- **Leitura de erro corrigida**: o backend v2 responde erros como
  `{ error: { code, message, details } }` (não mais `{ detail: "..." }`
  do FastAPI default). Centralizado em `src/api/errors.js`
  (`extractErrorMessage`) e usado em Login, Registro e no formulário de
  produto — se o formato mudar de novo, só precisa mexer em um lugar.
- A validação de senha no formulário de registro agora espelha a
  política real do backend (8+ caracteres, maiúscula, minúscula, número),
  com feedback antes de tentar enviar.

## Busca automática de produto (integração Amazon/Shopee)

O formulário de "novo produto" (`ProductFormModal.jsx`) tem um botão
**"Buscar dados"** ao lado do campo de link original. Ele chama
`POST /integrations/lookup` (`src/api/integrations.js`) e preenche
automaticamente nome, imagem, preço e link de afiliado — o usuário só
confere e ajusta antes de salvar, em vez de preencher tudo à mão.

Sem credenciais reais configuradas no backend (Amazon/Shopee), a busca
retorna dados de sandbox (determinísticos, mas não reais) — o frontend
não precisa saber a diferença, o formato da resposta é o mesmo.

Qualquer edição manual num campo depois de uma busca bem-sucedida
desarma o aviso "✓ preenchido automaticamente", já que os dados exibidos
podem não refletir mais o que veio da busca.

## Assistente de IA (widget de chat)

Botão flutuante no canto inferior direito (`ChatWidget.jsx`), disponível
em todas as páginas autenticadas. Chama `POST /assistant/chat`
(`src/api/assistant.js`).

- **Stateless, como o backend**: o histórico da conversa vive só no
  estado do componente React — a cada mensagem, envia as últimas 20
  trocas junto (o backend não guarda nada entre requisições).
- **Persiste entre páginas, não entre recarregamentos**: `ProtectedRoute`
  virou uma *rota de layout* do React Router (`<Outlet />`) em vez de
  envolver cada página individualmente — isso mantém o `ChatWidget`
  montado uma única vez, então trocar entre Painel e Produtos não reseta
  a conversa. Um F5 na página, sim, reseta (não há persistência em
  localStorage por enquanto).
- Mostra um indicador discreto ("↳ consultou seus produtos") quando o
  assistente usou a ferramenta `search_products` do backend para
  responder — transparência sobre quando ele foi buscar dado de verdade
  em vez de responder só do contexto inicial.
- Sem `OPENAI_API_KEY` configurada no backend, as respostas vêm em modo
  sandbox (prefixadas com `[Modo sandbox...]`) — o widget não trata esse
  caso de forma especial, simplesmente exibe o que a API retornar.

## Notas de build para produção

```bash
npm run build
```

Gera a pasta `dist/` pronta para servir via Nginx, Vercel, Netlify ou
qualquer storage estático.

## Docker

```bash
docker build -t afiliados-frontend .
docker run -p 3000:80 afiliados-frontend
```

O `Dockerfile` faz build multi-stage (Node para compilar, Nginx para
servir os arquivos estáticos). O `nginx.conf` já faz proxy de `/api`,
`/r` (redirect rastreado) e `/uploads` (imagens em modo sandbox) para um
serviço chamado `api` — isso bate com o nome do serviço no
`docker-compose.yml` do backend, que já inclui este frontend como
serviço (assumindo que as pastas `backend/` e `frontend/` estão lado a
lado). Rodando via `docker compose` a partir da pasta `backend/`, não
precisa configurar CORS nem apontar URL nenhuma manualmente — o Nginx e
o backend conversam na rede interna do Docker.

## Organograma

Página em `/organograma` (`OrgChart.jsx`). Agrupa os produtos numa
árvore de três níveis — **Portfólio → Marketplace (Amazon/Shopee) →
Tendência (alta/estável/declínio)** — com contagem e receita estimada em
cada nível, e a lista de produtos aparece ao expandir uma tendência.

A árvore é renderizada com indentação + borda esquerda colorida por
nível (CSS puro, sem biblioteca de diagrama) — optei por esse estilo
vertical em vez do clássico "organograma horizontal" porque é mais
robusto de acertar visualmente e funciona melhor em telas estreitas. A
receita por marketplace é recalculada no próprio frontend a partir da
lista de produtos (mesma fórmula do backend: preço × comissão% ×
conversões) — não existe um endpoint dedicado pra isso ainda.

## Webhooks, upload de imagem, sincronização e importação de CSV

As quatro telas que faltavam agora existem:

- **`/webhooks`** (`Webhooks.jsx`) — cadastra, lista e remove webhooks.
  O `secret` só é mostrado uma vez, logo após a criação, num banner com
  botão de copiar — igual ao padrão usado por provedores de API em
  geral. Depois disso, o backend nunca mais devolve ele (nem na
  listagem), então perder o secret significa recriar o webhook.
- **Upload de foto por produto** — clique na miniatura (ou no "+") na
  primeira coluna da tabela de Produtos para trocar a imagem. Aceita
  JPEG/PNG/WebP até 5MB (mesmo limite validado no backend).
- **"Sincronizar agora"** (`Products.jsx`) — chama
  `POST /integrations/products/sync-all` e recarrega a página atual da
  tabela com os números atualizados.
- **"Importar conversões (CSV)"** — abre um seletor de arquivo, envia
  pra `POST /products/import-conversions`, e mostra quantas linhas foram
  atualizadas/puladas. Linhas puladas (ambíguas ou sem correspondência)
  ficam num `<details>` expansível, cada uma com o motivo.

**Correção que veio junto**: o proxy do Vite (`vite.config.js`) só
encaminhava `/api` para o backend — imagens salvas em modo sandbox
(`/uploads/...`, disco local sem S3 configurado) e o redirect rastreado
(`/r/...`) quebrariam em desenvolvimento sem o proxy também cobrir essas
rotas. Adicionei as duas.

## Ainda não implementado

- Filtros na tabela de produtos (por marketplace, tendência, etc.) — a
  paginação já existe, os filtros ainda não.
- Persistência da conversa do assistente entre recarregamentos de página
  (hoje reseta no F5 — daria pra guardar em sessionStorage).
- Edição de webhook existente (hoje só dá pra criar e remover — pra
  mudar a URL ou os eventos, precisa recriar).
