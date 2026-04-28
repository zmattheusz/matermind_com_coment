# Projeto Mastermind — texto-fonte para estudo assistido (entrevista técnica)

**Uso:** suba este arquivo inteiro ao Notebook LM (ou gere áudio vídeo com base nele só). É complementar ao `docs/guia-codigo-fonte.md`: o guia é referência rápida por arquivo; **este texto** conta a história do sistema em ordem lógica, com vocabulário de entrevista e respostas a perguntas frequentes.

**Stack resumida:** Java 17, Spring Boot 3, Spring MVC (REST), Spring Security, Spring Data JPA, H2 arquivo, JWT stateless no header HTTP, Angular 18 SPA com componentes standalone, HttpClient + interceptor, guard de rota, RxJS onde necessário.

---

## 1. Propósito do sistema

É um **Mastermind** jogável no navegador. O backend gera uma combinação secreta de símbolos (cores representadas por letras: R, G, B…). O jogador envia tentativas; a API informa apenas **quantas posições** estão completamente corretas (cor certa na coluna certa). Não informa sem posição nem “bola branca/negra” ao estilo clássico estendido — a regra de negócio implementada é `countCorrect` por posição alinhando `expected` com `guess`. Quando todas as posições batem ou acabam as tentativas ou o jogador desiste, o status da partida muda e o gabarito pode ser revelado pelo GET de status.

Há **três níveis de dificuldade** definidos em `GameService`: Fácil (4 posições, 6 cores, sem repetir nas posições da tentativa onde a regra proíbe), Médio (4 posições, repetição permitida), Difícil (5 posições, mais cores disponíveis, repetição permitida).

---

## 2. Arquitetura geral à prova de entrevista

- **Cliente:** SPA Angular servida pelo dev server ou estática em produção. Não há renderização pelo servidor dos HTML do jogo para o navegador; tudo é API JSON.
- **Servidor:** aplicação Spring Boot expõe contexto servlet em `/api` (ver property `server.servlet.context-path`). Porta típica 8080 local.
- **Autenticação:** **stateless** com JWT. Não há `HttpSession` com cookie de sessão para a proteção principal das APIs de jogo. O navegador armazena o token (no projeto: `localStorage` em chaves definidas em `AuthService`).
- **Persistência:** H2 persistente em arquivo em desenvolvimento (`jdbc:h2:file`). Em produção você trocaria SGBD; o modelo JPA permite isso desde que configure datasource e Hibernate dialect.

Fluxo físico típico: Angular chama `http://localhost:4200`, `environment.ts` define `apiUrl` para `http://localhost:8080/api`. Requests autenticados levam cabeçalho `Authorization: Bearer <jwt>` aplicado pelo `authInterceptor`.

---

## 3. Por que JWT e onde ele nasce

**Por que usar JWT aqui:** em SPA + API REST é comum evitar estado de sessão no servidor por usuário; isso ajuda escalar em cluster e reduz estado no servidor (a identidade vai no Bearer). **Trade-off:** revogar token antes de expirar exige lista de invalidação ou TTL curto.

**Onde o token é criado:** após login bem-sucedido, `AuthService.login` (Java) usa `JwtUtil.generateToken(username)` configurado com secret e expiração em `application.yml` (`app.jwt.*`).

**Onde o servidor confia:** `JwtAuthFilter` (filtro Spring Security) intercepta antes do fluxo típico, lê Bearer, extrai usuário (`JwtUtil.getUsernameFromToken`), carrega detalhes com `UserDetailsServiceImpl.loadUserByUsername` (permite username **ou** e-mail), valida correspondência (`validateToken`), e instala `UsernamePasswordAuthenticationToken` no `SecurityContextHolder`.

**Por que permissões em `SecurityConfig`:** apenas rotas públicas explícitas passam sem autenticação: login, register, reset-password, Swagger, console H2. **Todo `/games/**` está protegido**, inclusive listar instruções e dificuldades; o navegador precisa estar logado com JWT válido para esses endpoints. CSRF está desabilitado porque a API espera Bearer no header em vez de autenticação por cookie de sessão.

---

## 4. Cadastro e login no detalhe (backend)

**`AuthController`** mapeia `POST /auth/register`, `/auth/login`, `/auth/reset-password`, `GET /auth/me`.

**register:** `AuthService.register` verifica unicidade username e email, normaliza e-mail lowercase, faz **encode** bcrypt de senha e da resposta da pergunta de segurança. O hash da “resposta secreta” nunca volta ao cliente em texto plano nas respostas.

**login:** recebe JSON com campo username-or-email combinado mais senha. `AuthenticationManager` autentica; no sucesso recupera usuário do banco, eventualmente reaplica política de streak se dias sem jogar zeram sequência (`ajustarSequenciaSeUsuarioFicouInativo`), monta `UserResponse`, gera token e retorna `LoginResponse`.

**reset-password:** localiza usuário pelo e-mail, compara bcrypt da resposta de segurança (normalização minúscula/trim igual ao cadastro), troca apenas o hash da senha.

**`/auth/me`:** permite ao front atualizar streak e melhor score após jogos sem pedir novo login quando o JWT ainda é válido.

---

## 5. Modelo relacional pensado pelo JPA

**User:** guarda dados de conta, hashes, melhor pontuação global e melhores marcas por modo fácil/médio/difícil, incluindo **durações em segundos** quando existem registros atualizados, mais campos para streak (`streakDays`, `lastStreakDate` interpretados via fuso São Paulo onde aplicável).

**Game:** cada linha é uma sessão jogável única pelo `gameCode` string (curto tipo slug). Estado `GameStatus`: IN_PROGRESS, WON ou LOST. Campos grandes armazenam JSON serializado pela `ObjectMapper` em string: lista de tentativas (matriz), lista paralela com “quantidade de corretos por tentativa”. `startedAt` e `finishedAt` em `Instant`, `durationSeconds` calculados na finalização quando aplicável para vitória ou derrota por limite/desistência.

---

## 6. Motor da partida: `GameService`

**startGame(difficulty)** cria `Game` novo, código secreto pelo `generateSecretCode` coerente com `parametrosDaDificuldade`, marca `startedAt` agora, status IN_PROGRESS.

**submitAttempt(gameCode):** garante mesmo usuário do token; recusa partida já finalizada; valida comprimento igual ao `codeLength` do modo; valida cada cor contra a lista permitida do modo (`parametrosDaDificuldade`); no modo sem repetição impõe cores distintas na tentativa atual; conta `correctCount` com `countCorrect`: posição a posição, cor certa na coluna certa; registra tentativas e feedbacks em JSON textual no `Game`; incrementa número de tentativas; vitória quando `correctCount == codeLength`; derrota por limite quando `attemptCount` atinge `maxAttempts` configurado antes de ganhar nessa jogada; em vitória grava timestamps, atualiza bests com `novoRecorde`, chama `applyStreak`; persiste usuário quando alterado e o `game`. Devolve `AttemptResponse` com contagem de pins corretos, flags de vitória e fim de jogo.

**forfeitGame:** valida mesmo usuário e partida ainda jogável; força LOSS, fecha tempos para duração, persiste Game, monta dto de status com `getGameStatus` já podendo liberar código esperado.

**getRanking:** delega ordenação aos métodos JPQL dos repositórios e numera linhas só no serviço.

---

## 7. Ordenação de ranking por dificuldade em `UserRepository`

Há vários métodos (`findAllForRanking` para placar global, `findAllForRankingEasy` / Medium / Hard para cada modo) com ordenação JPQL no mesmo molde:

1. usuários **sem pontuação registrada** vêm depois (`CASE WHEN bestScore… IS NULL OR = 0 THEN 1 ELSE 0`).
2. **Menor número de tentativas** no campo correspondente (`bestScore` ou `bestScoreEasy` etc.).
3. **Menor duração** em segundos no mesmo modo; registros sem tempo (`bestDurationSecs* NULL`) perdem esse desempate (segundo CASE).
4. desempate final por **`id`** do usuário.

O método `novoRecorde` em `GameService` espelha a mesma filosofia entre tentativa e tempo.

---

## 8. Frontend Angular sob o olhar do entrevistador

**Roteamento:** `app.routes` define guarded paths com `authGuard` verificação simples pelo token atual.

**interceptors:** garante Bearer em todas requisições HttpClient onde token não nulo reduz erro 401 repetitivo.

**auth.service.ts:** estado reativo usando signals opcionalmente combinado ao localStorage pra persistência de refresh inicial da SPA.

**login.component:** reactive forms diferenciando modo login/register com validações distintas.

**difficulty-selector:** busca textual metadados e em `POST` start redireciona via router para rota parametrizada de jogo.

**game.component:** ao montar, se já tem código na rota faz GET estado; inicializa cronômetro se partida não finalizada usando `startedAtEpochMs`; ao finalização reflete `durationSeconds` do backend; comandos usuario enviam POST attempt e opcional POST forfeit.

---

## 9. Tratamento de erros REST

Classe **`GlobalExceptionHandler`** consolida **`MethodArgumentNotValidException`** (Bean Validation falhou), **`BadRequestException`**, **`UnauthorizedException`**, **`BadCredentialsException`**, **`NotFoundException`**, **`Exception`** genérica 500 sanitizada. Frontend pode mostrar estas mensagens se expuser `message` dentro do objeto.

---

## 10. Testes automatizados existentes no repo

Existem **`GameServiceTest`**, **`AuthServiceTest`**, **`AuthControllerTest`** (MockMvc `@WebMvcTest`). Servem regressão rápida, não garantem aceitação E2E completa no browser.

---

## 11. Perguntas que costumam cair na entrevista (com âncoras no código)

**Onde a partida termina com vitória e atualiza pontuações?**

No método `submitAttempt` de `GameService` quando há match completo antes de falha limite tentativas. Caminho completo arquivo Java serviço de jogo.

**Como garante que apenas o dono mexa na partida?**

Compara ids de usuários extraídos de `Principal` segurança atual com relacionamento usuário ligado ao `Game`.

**Por que JSON em texto no banco e não só tabela de tentativa separada?**

Simplicidade e poucas tabelas; trade-off típico de entrevista: consultas relacionais ficam menos ricas até normalizar depois.

**Por que bcrypt e não plaintext?**

Se vazar backup do banco, senhas continuam trabalhosas sem o salt interno ao hash.

**O que você mudaria em produção próximo passo.**

Trocaria H2 file por Postgres, externalizaria secreto JWT apenas variável ambiente, possivelmente adicionaria testes contrato e healthcheck.

---

## 12. Limitações honestas aceitável citar na entrevista

- Existe placar por modo (`findAllForRankingEasy`/`Medium`/`Hard`) e um placar global (`findAllForRanking`): o global usa apenas os campos cumulativos na entidade (`bestScore`, `bestDurationSecs`), não reprocessa o histórico completo de partidas a cada query.
- Estado de jogo novo depende apenas de JWT válido para rotas sob `/games` (escala horizontal só precisa mesmo secret/compartilhamento de datasource; não há afinitidade de sessão).
- Tempo atualizado no front faz GET de novo status; não há WebSocket nem SSE para servidor empurrar o relógio.

---

Este documento fecha o ciclo para consumo multimídia com o `guia-codigo-fonte.md`: **narrativa primeiro**, **mapa arquivo a arquivo segundo**. Boa revisão combinada vídeo e texto no Notebook LM.
