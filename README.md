# Vaulta

Marketplace de itens digitais. Frontend estático com rotas reais (History API) + API em Vercel Functions + Postgres + Supabase Storage. Sem build, sem bundler.

## Estrutura
- `public/` — site (`index.html`, `assets/css`, `assets/js` em módulos ES, `assets/js/views/*`)
- `api/[...path].js` — função única da API
- `lib/` — `db.js` (Postgres/SQLite), `auth.js` (scrypt, sessão em cookie HttpOnly assinado, rate limit), `storage.js` (Supabase Storage), `users.js`, `listings.js`
- `scripts/` — `dev-server.js` (imita a Vercel), `e2e.js` e `ui-smoke.mjs` (testes)
- `vercel.json` — fallback de rotas, cabeçalhos de segurança e CSP

## Rotas
`/` · `/anuncios` · `/anuncios/<jogo>/<id>` · `/login` · `/cadastro` · `/anunciar` · `/painel` · `/painel/anuncios` · `/painel/anuncios/novo` · `/painel/anuncios/<id>/editar`

API: `GET/POST /api/anuncios`, `GET/PUT/DELETE /api/anuncios/:id`, `PATCH /api/anuncios/:id/status`, `PUT/DELETE /api/anuncios/:id/imagem`, `GET /api/painel/anuncios[/:id]`, `POST /api/auth/{register,login,logout}`, `GET /api/auth/me`, `GET /api/meta`.

## Rodar localmente (Node 22+, sem instalar nada)
```
npm run dev     # http://localhost:3000  (SQLite + uploads em .data/)
npm test        # testes ponta a ponta
```

## Deploy na Vercel
1. Crie um projeto no **Supabase**. Em Storage, crie o bucket **vaulta** marcado como *público*.
2. Na Vercel, importe o repositório e defina as variáveis de `.env.example` (`DATABASE_URL` com a string do *pooler*, `AUTH_SECRET`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_BUCKET`).
3. Deploy. As tabelas são criadas sozinhas na primeira requisição.

Em produção o app recusa iniciar sem `DATABASE_URL`/`AUTH_SECRET`/credenciais do storage (nunca usa disco ou segredo padrão).


## Administração

- **Login:** `/admin/login` (sem link no site público). Sessão própria em cookie separado (`HttpOnly`, `SameSite=Strict`, `__Host-` em produção): expira em 2 h ou 30 min sem uso. O papel `admin` é conferido no banco a cada requisição, então rebaixar ou suspender derruba o acesso na hora. Uma sessão de usuário comum nunca abre a área admin.
- **Dono (primeira conta):** não existe cadastro de admin pela web. Crie o dono pelo terminal, com o banco configurado em `.env.local`:
  `npm run admin:create -- voce@dominio.com "Seu Nome"` (senha pedida sem eco, mínimo 12 caracteres). Rodar de novo para o mesmo e-mail troca a senha e encerra as sessões. Com `-- ... --admin` cria um administrador comum direto pelo terminal.
- **Mais administradores:** o dono entra em Admin > Usuários, digita a própria senha e clica em **Tornar admin** (ou **Remover admin**). Administradores moderam, mas não criam outros admins. O dono não pode ser rebaixado pela web.
- **Selos** (aparecem ao lado do nome do vendedor nos anúncios): branco = loja verificada (qualquer admin concede), vermelho = equipe e parceiros (admins ganham automático; parceiro só o dono concede), verde = conta oficial (dono, automático; outros só o dono concede). Em Admin > Usuários use o seletor de selo de cada linha.
- **Painel:** visão geral, usuários (buscar, suspender/reativar), anúncios (buscar, bloquear/desbloquear, excluir) e auditoria (quem fez o quê, quando e de qual IP).
- **Moderação:** anúncio *bloqueado* sai do catálogo e o vendedor não consegue reativá-lo. Usuário *suspenso* perde as sessões, não entra e seus anúncios somem do catálogo. Contas admin não podem ser suspensas pela web.
- **Proteções:** limite de tentativas de login (por IP+e-mail, por IP e por e-mail), mesma resposta para senha errada/conta inexistente/sem permissão, requisições de escrita exigem `X-Vaulta-Admin: 1` (força preflight de CORS) além da checagem de Origin, e `X-Robots-Tag: noindex` + `Cache-Control: no-store` em `/admin`.
- **Supabase:** as tabelas novas (`admin_sessions`, `admin_audit`) e as demais ganham RLS ligado automaticamente em Postgres, para a API REST pública do Supabase não expor senhas e sessões.


## Deploy na Vercel (passo a passo)
1. **Supabase**: crie o projeto. Em *Storage*, crie o bucket **vaulta** marcado como *Public*. Em *Project Settings > Database*, copie a string do **Connection Pooler** (porta 6543). Em *Project Settings > API*, copie a **URL** e a chave **service_role** (secreta, só no servidor).
2. **Vercel**: *Add New > Project* e importe este repositório. *Framework Preset*: **Other**. Deixe *Build Command* vazio e *Output Directory* = `public`.
3. **Environment Variables** (Production): `DATABASE_URL`, `AUTH_SECRET` (`openssl rand -hex 32`), `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_BUCKET=vaulta`. Recomendado: `DATA_ENCRYPTION_KEY` (`openssl rand -hex 32`). Nunca troque `AUTH_SECRET`/`DATA_ENCRYPTION_KEY` depois de ter dados.
4. **Deploy**. As tabelas são criadas na primeira requisição, já com RLS ligado.
5. **Dono**: no seu computador, preencha `.env.local` com o mesmo `DATABASE_URL` e rode `npm install && npm run admin:create -- voce@dominio.com "Seu Nome"`. Entre em `/admin/login`: no primeiro acesso você cadastra o app autenticador e recebe 8 códigos de recuperação.

## Segurança (resumo)
- **Admin com 2FA obrigatório** (TOTP, anti-replay, 8 códigos de recuperação de uso único, bloqueio de força bruta). Perdeu o aparelho e os códigos? `npm run admin:create -- email --reset-2fa` (precisa de acesso ao banco).
- **Dados de entrega cifrados** no banco (AES-256-GCM, amarrados ao id do anúncio). Dados antigos em texto puro continuam legíveis.
- **Senhas**: scrypt N=32768; hashes antigos são atualizados no login; bloqueia senhas comuns ou que contêm e-mail/nome; mesma resposta e mesmo custo para conta inexistente.
- **Sessões**: cookie `__Host-`, HttpOnly, Secure e **SameSite=Strict**; trocar a senha ou "sair de todos os dispositivos" (Painel > Segurança) derruba as outras sessões.
- **CSRF/entrada**: checagem de Origin e `Sec-Fetch-Site`, JSON obrigatório, limites de tentativas no banco; o IP só vem de cabeçalho de proxy na Vercel (ou com `TRUST_PROXY=1`), então não dá para burlar o limite forjando `X-Forwarded-For`.
- **Cabeçalhos**: CSP sem scripts inline, COOP/CORP, Permissions-Policy, nosniff, `object-src 'none'`; `/admin` sem cache e fora de buscadores.
- **Supabase**: RLS ligado em todas as tabelas (a API pública não enxerga nada); o storage só apaga arquivos no padrão `listings/<id>/<arquivo>`.
- Testes: `npm test` (e2e, interface, admin e segurança).
