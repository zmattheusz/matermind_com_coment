# Guia do código — Mastermind (consulta rápida na banca)

Texto pensado pra você abrir arquivo por arquivo no IDE e saber **o que falar**. Caminhos a partir da pasta raiz do projeto (`Projeto Itaú/`).

Para demo só na UI (sem abrir código), use `docs/roteiro-apresentacao.md` como roteiro curto.

---

## Login e senha na tela — onde fica no projeto

| O quê | Caminho |
|-------|---------|
| Markup do formulário (inputs de usuário e senha, botão Entrar, modo cadastro) | `frontend/src/app/features/auth/login/login.component.html` |
| Campos login | `input#username` (`formControlName="username"`) — no login aceita usuário **ou** e-mail no mesmo campo; `input#password` (`formControlName="password"`) |
| Lógica do formulário (envia login/register, troca modo) | `frontend/src/app/features/auth/login/login.component.ts` |
| Estilo da tela | `frontend/src/app/features/auth/login/login.component.css` |
| Chamada HTTP para a API | `frontend/src/app/core/services/auth.service.ts` (`login`, `register`) |
| Endpoint REST de login | `POST /api/auth/login` — tratado em `backend/.../controller/AuthController.java` método `login` |

Recuperação de senha (outra tela): formulário em `frontend/src/app/features/auth/forgot-password/forgot-password.component.html` + `forgot-password.component.ts`.

---

## Fim da partida — onde isso é decidido (pergunta clássica)

**Regra:** a partida para quando **acerta tudo**, **esgota as tentativas** ou **desiste**.

| Situação | Onde no backend | O que acontece |
|----------|-----------------|----------------|
| Vitória (acertou todas as posições) | `GameService.submitAttempt` — quando `correctCount == codeLength` | Marca `Game` como `WON`, grava pontuação, tempo, atualiza ranking/streak do `User`, salva. |
| Derrota (10ª tentativa errada) | Mesmo método — quando `attemptCount >= maxAttempts` sem ter ganho antes | Marca `LOST`, grava `finishedAt` e duração. |
| Desistência | `GameService.forfeitGame` | Força `LOST`, encerra, devolve status com gabarito. |

Front-end chama isso aqui:

- Tentativa: `GameService.submitAttempt` no front (`game.service.ts`) a partir de `game.component.ts` → `submitAttempt()`.
- Desistir: `forfeit()` em `game.component.ts` → `gameService.forfeitGame`.

Depois que acaba, o front costuma chamar `getGameStatus` pra buscar o **gabarito** (`answer`) — só vem quando o jogo não está mais `IN_PROGRESS` (ver `getGameStatus` no Java).

---

## Backend (`backend/src/main/java/com/itau/mastermind/`)

### `MastermindApplication.java`

Ponto de entrada Spring Boot — sobe o contexto, JPA, Security, etc.

---

### Pacote `config`

**`SecurityConfig.java`** — Monta a cadeia de segurança: **sem sessão server-side** (JWT), CSRF off (API REST), libera só login/registro/reset sem token, Swagger e H2 console públicos, todo o resto **autenticado**. Registra **CORS** pra `http://localhost:4200`. Encaixa o `JwtAuthFilter` antes do filtro de usuário/senha padrão.

**`AuthConfig.java`** — Expõe o bean `AuthenticationManager`, que o login usa pra validar senha.

**`OpenApiConfig.java`** — Configura o Swagger/OpenAPI e o esquema **Bearer JWT** pra testar endpoints autenticados na documentação.

**`DbStartupMigration.java`** — Na subida, olha se a coluna `PASSWORD` da tabela `USERS` é curta demais no H2 e, se precisar, aumenta pra caber hash bcrypt (senão migrações antigas quebravam login).

---

### Pacote `controller`

**`AuthController.java`** — REST em `/auth`:

- `POST /login` → `AuthService.login`
- `POST /register` → registro
- `POST /reset-password` → recuperação com resposta de segurança
- `GET /me` → perfil do usuário logado (precisa Bearer)

**`GameController.java`** — REST em `/games` (tudo com JWT):

- `POST /start` e `POST /start/{difficulty}` → nova partida
- `POST /{gameCode}/attempt` → tentativa
- `GET /{gameCode}` → estado da partida (matriz de palpites, cronômetro no DTO, gabarito se já terminou)
- `GET /ranking` → ranking por query `difficulty`
- `GET /difficulties` e `GET /instrucoes` → textos/dicas pro front
- `POST /{gameCode}/forfeit` → desistir

---

### Pacote `service`

**`AuthService.java`** — Negócio de conta:

- `register` — valida duplicidade de usuário/e-mail, **hash** de senha e da resposta de segurança, persiste `User`.
- `login` — delega autenticação ao Spring, gera **JWT**, devolve usuário; antes pode **zerar streak** se a pessoa sumiu mais de um dia (ver método privado `ajustarSequenciaSeUsuarioFicouInativo`).
- `getProfile` — monta `UserResponse` com streak/datas.
- `resetPassword` — acha por e-mail, confere hash da resposta de segurança, troca senha.
- `normalizeSecurityAnswer` — trim + minúsculas pra comparar resposta estável.
- `toUserResponse` — monta DTO com melhor pontuação e dados de streak.

**`GameService.java`** — Coração do jogo:

- `novoRecorde` — decide se uma vitória melhora recorde global ou por dificuldade (**menos tentativas**, empate por **menos tempo**).
- `parametrosDaDificuldade` — tamanho do código, lista de cores, se permite repetir na tentativa.
- `generateSecretCode` — sorteia o código esperado conforme as regras do modo.
- `startGame` — cria `Game`, `startedAt`, status `IN_PROGRESS`.
- **`submitAttempt`** — valida dono da partida, tamanho do palpite, cores permitidas, repetição no modo fácil; conta acertos (`countCorrect`), grava JSON de tentativas e feedbacks; **se ganhou** atualiza usuário/ranking/streak; **se perdeu por limite** marca `LOST`; devolve quantos pinos certos e se acabou o jogo.
- `applyStreak` — atualiza dias seguidos com vitória (fuso Brasil).
- `parseMatrix` / `writeMatrix` / `parseFeedbackCounts` / `writeFeedbackCounts` — serializa listas em JSON no banco.
- `getGameStatus` — monta DTO pro front (inclui `startedAtEpochMs` pro cronômetro).
- `getRanking` — lista usuários já ordenados pelo repositório e numera posição.
- `obterInstrucoesJogo` / `getDifficulties` — textos das telas de ajuda e escolha de modo.
- **`forfeitGame`** — encerra como derrota e devolve status com resposta.

---

### Pacote `domain`

**`User.java`** — Entidade JPA: credenciais, melhores pontuações/tempos por modo e globais, streak, lista de `Game`.

**`Game.java`** — Uma partida: código esperado em string CSV, tentativas em JSON, status (`IN_PROGRESS` / `WON` / `LOST`), dificuldade, timestamps, duração em segundos.

---

### Pacote `repository`

**`UserRepository.java`** — Spring Data JPA + queries customizadas de **ranking** (ordenam tentativas, depois tempo, depois id).

**`GameRepository.java`** — `findByGameCode`, etc.

---

### Pacote `dto`

Só transporte de dados JSON — sem lógica:

- `LoginRequest` / `LoginResponse` — token + usuário.
- `RegisterRequest`, `ResetPasswordRequest`.
- `UserResponse` — perfil + streak + melhor score.
- `GameStartResponse` — `gameCode`.
- `AttemptRequest` / `AttemptResponse` — palpite e resultado da jogada.
- `GameStatusResponse` — snapshot da partida pro Angular.
- `RankingEntry` — linha do ranking.
- `DifficultyInfoResponse`, `InstrucoesJogoResponse` — textos da UI.

---

### Pacote `security`

**`JwtUtil.java`** — Assina e lê JWT (subject = username, expira conforme `application.yml`).

**`JwtAuthFilter.java`** — A cada request: lê header `Authorization: Bearer ...`, valida token, carrega `UserDetails` e joga no `SecurityContext` pra quem estiver logado.

**`UserDetailsServiceImpl.java`** — `loadUserByUsername` busca usuário por **username** ou **e-mail** (login aceita os dois).

**`UserDetailsImpl.java`** — Adapta sua entidade `User` pro contrato do Spring Security (senha hash, `ROLE_USER`, id exposto pra controllers usarem).

---

### Pacote `exception`

**`GlobalExceptionHandler.java`** — Traduz exceções da aplicação e validação Bean Validation em respostas JSON padronizadas (`ApiError`).

Demais classes aí: `BadRequestException`, `NotFoundException`, `UnauthorizedException`, `ApiError` — tipos simples de erro.

---

### Pacote `util`

**`FusoHorarioPadrao.java`** — Referência de timezone Brasil pra streak e datas.

---

## Frontend (`frontend/src/app/`)

### Raiz da app

**`app.component.ts`** — Layout geral: se estiver logado mostra **nav** (Início, Novo jogo, Ranking), fundo, **footer** condicional (some em login/recuperação). Botão “Sair” chama `AuthService.logout` e manda pra `/login`.

**`app.config.ts`** — Registra rotas, **HttpClient** com interceptor JWT, animações.

**`app.routes.ts`** — Mapa de URLs → componente e `authGuard` nas rotas fechadas.

---

### `core/services`

**`auth.service.ts`** — Guarda token e usuário no **localStorage** (`mm_token`, `mm_user`); `login` grava tudo após `POST /auth/login`; `refreshProfile` bate em `/auth/me` depois de vitória; `logout` limpa; expõe `currentUser` como signal computado.

**`game.service.ts`** — Wrappers HTTP: start game, tentativa, status, ranking, desistir, instruções, dificuldades.

---

### `core/guards`

**`auth.guard.ts`** — Se não tem token, redireciona pra `/login`.

---

### `core/interceptors`

**`auth.interceptor.ts`** — Em toda request HTTP, se existir token, adiciona `Authorization: Bearer ...`.

---

### `features/auth/login`

**`login.component.ts`** — Formulário reativo; `toggleMode` alterna login/cadastro e muda validadores (`applyValidators`); `onSubmit` chama `auth.login` ou `auth.register`.

**`login.component.html`** — Onde estão visualmente usuário/senha (ver tabela no topo deste guia).

---

### `features/auth/forgot-password`

**`forgot-password.component.ts` + `.html`** — Form de e-mail, resposta de segurança, nova senha; submete em `resetPassword` do serviço.

---

### `features/dashboard`

**`dashboard.component.ts`** — Cards pra Como jogar / Novo jogo / Ranking; texto do **streak** embaixo; botão Sair duplicado (há também no header global).

---

### `features/como-jogar`

**`como-jogar.component.ts`** — Busca instruções na API e mostra passos (template inline no arquivo).

---

### `features/difficulty-selector`

**`difficulty-selector.component.ts`** — Lista dificuldades da API e ao clicar dá `POST` start com o modo escolhido e navega pra `/game/:gameCode`.

---

### `features/game`

**`game.component.ts`** — Carrega estado da partida; **cronômetro** com `startedAtEpochMs` ou `durationSeconds`; `submitAttempt`, `forfeit`, monta linhas do tabuleiro; paleta de cores.

**`game.component.html`** — HUD com ID da partida e relógio; linhas de tentativa; gabarito quando termina.

**`game.component.css`** — Visual do tabuleiro e cronômetro.

---

### `features/ranking`

**`ranking.component.ts`** — Abas EASY/MEDIUM/HARD, tabela com tentativas/tempo/registro, estatísticas rápidas.

---

### `components/app-footer`

**`app-footer.component.ts`** — Rodapé da aplicação.

---

## Configuração útil

| Arquivo | Pra quê |
|---------|---------|
| `backend/src/main/resources/application.yml` | Porta, JWT, H2, `max-attempts`, secret default |
| `frontend/src/environments/environment.ts` | URL base da API (`http://localhost:8080/api`) |
| `frontend/src/environments/environment.prod.ts` | Mesmo pra build de produção (ajustar deploy) |

---

## Testes (`backend/src/test/`, `frontend/src/app/**/*.spec.ts`)

- **`GameServiceTest`** — Partida feliz e erros básicos no serviço de jogo.
- **`AuthServiceTest`**, **`AuthControllerTest`** — Fluxos de autenticação.

Servem mais pra regressão do que pra banca oral, mas você pode dizer que existem testes automatizados nos serviços principais.

---

## Ordem sugerida se te mandarem “abre o fluxo de login”

1. `login.component.html` (visual)
2. `login.component.ts` (`onSubmit`)
3. `auth.service.ts` (`login`)
4. `AuthController.java` → `AuthService.java` (`login`)
5. `UserDetailsServiceImpl` + `JwtUtil` se falarem de token

Boa sorte na apresentação.
