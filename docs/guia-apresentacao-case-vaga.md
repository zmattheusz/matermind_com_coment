# Guia de apresentação — Case (Mastermind) × Vaga × Código

Use este arquivo como **colinha na banca**: leia na ordem do **case** (`docs/case.md`) e abra as pastas indicadas. A descrição da vaga está em `docs/descircao-da-vaga.md`.

---

## 0. Fontes

| Fonte | Conteúdo |
|--------|-----------|
| `docs/case.md` | Enunciado oficial do case (requisitos funcionais e não funcionais). |
| `docs/descircao-da-vaga.md` | Expectativas da vaga (Java/Angular, qualidade, segurança, AWS, arquitetura). |
| `docs/guia-codigo-fonte.md` | Mapa rápido arquivo a arquivo (consulta paralela). |
| `README.md` | Como rodar, stack, GIFs/prints da demo. |

---

## 1. Alinhamento rápido: vaga × o que você mostra

- **Java + Angular (case recomendou; vaga pede):** Spring Boot 3 + Angular 18 — diga que seguiu a stack recomendada do case e que casa com o perfil da vaga.
- **Qualidade e testes (vaga + case 4.3):** `mvn test`, `npm test`; cite `AuthServiceTest`, `GameServiceTest`, `AuthControllerTest` e os `.spec.ts` do front.
- **Desenvolvimento seguro / boas práticas (vaga):** senha com bcrypt, JWT, validação de entrada, erros padronizados — aponte `SecurityConfig`, `GlobalExceptionHandler`, DTOs com `@Valid`.
- **Arquitetura (vaga “noções”):** monólito modular REST (não microserviço); camadas claras; opcionalmente diga que event-driven não se aplica ao escopo do jogo síncrono, mas a API é stateless com JWT.

---

## 2. Roteiro de apresentação guiado pelo case (`docs/case.md`)

Siga a numeração do PDF na sua fala: “no item 3.1 do case…”.

### Seção 1–2 — Contexto e stack (case)

**O que falar (30 s):** jogo Mastermind web; objetivo era estrutura limpa, decisões coerentes, solução funcional; escolhi **Java 17 + Spring Boot**, **Angular 18**, **H2 relacional** (case permite qualquer BD relacional ou não).

**Onde mostrar:** raiz do repo `backend/` + `frontend/`; `README.md` (descrição + decisões).

---

### 3.1 Autenticação (case)

| Exigência do case | Onde no projeto | Dica na demo |
|-------------------|-----------------|--------------|
| Login com usuário **ou** e-mail + senha e validação de formulário | `frontend/src/app/features/auth/login/` (`login.component.html`, `.ts`) | Mostrar validação (campos vazios). |
| Redirecionamento ao dashboard após login | `login.component.ts` (após sucesso) + `app.routes.ts` (`dashboard`) | Fluxo ao vivo. |
| Mensagem amigável se credenciais inválidas | Front: tratamento da resposta de erro; back: `BadCredentialsException` / handler | Forçar senha errada. |
| Logout → tela de login | `frontend/src/app/app.component.ts` (botão Sair), `core/services/auth.service.ts` (`logout`) | Clicar Sair. |

**Backend que atende o login:** `backend/src/main/java/com/itau/mastermind/controller/AuthController.java`, `service/AuthService.java`, `security/` (JWT + filtro).

---

### 3.2 Dados (case)

| Exigência do case | Onde no projeto | Dica na demo |
|-------------------|-----------------|--------------|
| Usuário único na base; histórico e melhor pontuação | `domain/User.java`, `repository/UserRepository.java`; partidas em `Game` ligadas ao usuário | H2 console ou explicar modelo. |
| Partida: usuário, pontuação final, **matriz de tentativas**, duração, **resposta esperada** | `domain/Game.java`; persistência em `GameService.java` (JSON da matriz + `expectedCode`) | Abrir `Game` ou trecho de `submitAttempt` / `getGameStatus`. |

---

### 3.3 Visual — frontend (case)

| Exigência do case | Onde no projeto | Dica na demo |
|-------------------|-----------------|--------------|
| Tela de login | `features/auth/login/` | — |
| Dashboard inicial (menu jogo, ranking…) | `features/dashboard/dashboard.component.*` | Cards / nav. |
| Tela do jogo em andamento | `features/game/game.component.*` | Matriz de linhas + feedback. |
| Ranking ordenado por desempenho | `features/ranking/` + API ranking | Abas por dificuldade se quiser destacar extra. |
| Usabilidade / responsividade | CSS dos componentes acima | Mencionar esforço de layout. |

**Rotas:** `frontend/src/app/app.routes.ts` (login, dashboard, game, ranking, etc.).

---

### 3.4 Regras do jogo (case)

| Exigência do case | Onde no projeto | Dica na demo |
|-------------------|-----------------|--------------|
| Backend gera registro: resultado esperado, **código único** da partida, jogador | `GameService.startGame` + `GameRepository.save`; entidade `Game` | `POST` start; mostrar `gameCode` na URL ou HUD. |
| Matriz visual estilo tabuleiro | `game.component.html` / `.ts` — linhas de tentativa | Comparar com figura do case. |
| Cada tentativa: front chama back; back **não entrega a combinação** ao front durante o jogo | `GameController` + `GameService.submitAttempt`; gabarito só após fim em `getGameStatus` | Insistir nisso na banca. |
| Resposta só com **número de acertos** (case: quadrantes = quantidade de células corretas, não posições das “parciais”) | `AttemptResponse.getCorrectCount()` a partir de `countCorrect` em `GameService` | **Frase pronta:** “O case pede feedback numérico por tentativa sem revelar o código; implementei contagem de posições com cor e lugar certos (`countCorrect`); não há pinos de ‘cor certa lugar errado’ nesta versão.” |
| Máximo **10** tentativas | `application.yml` `app.mastermind.max-attempts` + `GameService` | Mostrar constante ou config. |
| Front atualiza feedback a cada tentativa | `game.component.ts` após `submitAttempt` | Jogar uma rodada. |

---

### 4.1 Backend — não funcional (case)

| Exigência | Onde |
|-----------|------|
| Controller → Service → Repository | `controller/*`, `service/*`, `repository/*` |
| Validação de inputs + erros padronizados | DTOs com Bean Validation; `GlobalExceptionHandler.java` + `dto`/exceções |
| Exceções globais com HTTP 400, 401, 404, 500 | `GlobalExceptionHandler.java` |
| Documentação API (Swagger/OpenAPI) | `OpenApiConfig.java`; Swagger UI no `README` |

---

### 4.2 Frontend — não funcional (case)

| Exigência | Onde |
|-----------|------|
| Componentização; lógica fora do template | `features/*` — TS com métodos; templates principalmente bindings |
| Services para API | `core/services/auth.service.ts`, `game.service.ts` |
| Erro da API com feedback visual | Tratamento nos componentes/serviços (login, game, etc.) |
| Validações em formulários | Reactive forms em `login`, `forgot-password`, etc. |

---

### 4.3 Gerais (case)

| Exigência | Onde |
|-----------|------|
| Testes unitários back e front | `backend/src/test/java/...`, `frontend/src/app/**/*.spec.ts` |
| Ambiente configurável + guia | `application.yml`, `environment.ts`, `.env.example`, `README.md` |

---

### 5 — Entrega (case)

| Exigência | Onde |
|-----------|------|
| Repo com `backend` e `frontend` na raiz | Estrutura do projeto |
| README: descrição, decisões, pré-requisitos, passos back/front, `.env.example`, mídia | `README.md`, `docs/gifs`, `docs/prints` |
| Documentação da API | Swagger + README |

---

## 3. O que você entregou **além** do texto mínimo do case (diferenciais)

Use só se sobrar tempo ou perguntarem “o que mais você fez?”.

- **Recuperação de senha** (não está no case): `features/auth/forgot-password/`, `AuthController` `/reset-password`.
- **Três dificuldades** e textos de ajuda via API: `GameService.parametrosDaDificuldade`, `features/difficulty-selector/`, `features/como-jogar/`.
- **Streak diário** e **cronômetro** com desempate por tempo no ranking: `AuthService` / `GameService` / `User` / `ranking.component`.
- **JWT** + interceptor Angular (case fala login/logout; não obriga JWT, mas reforça segurança alinhada à vaga).

---

## 4. Perguntas que o entrevistador pode fazer (e resposta curta + onde olhar)

### Sobre o case e o escopo

- **“Onde o backend garante que o front não vê o código secreto durante a partida?”** — `getGameStatus` / DTO só inclui `answer` quando o jogo não está mais `IN_PROGRESS`; durante o jogo só tentativas e contagem. (`GameService.java`)
- **“Onde decide vitória, derrota e desistência?”** — `submitAttempt` e `forfeitGame` em `GameService.java`.
- **“Como você atende o máximo de 10 tentativas?”** — `maxAttempts` em `application.yml` + checagem em `submitAttempt`.
- **“Onde está a matriz completa para auditoria?”** — Campos JSON da entidade `Game` (`attemptsMatrix`, feedbacks), escritos em `GameService`.

### Sobre arquitetura e qualidade

- **“Por que monólito e não microserviços?”** — Escopo do case; camadas internas claras; microserviço seria overkill e aumentaria operação sem requisito.
- **“Como padroniza erro de API?”** — `GlobalExceptionHandler` + corpo JSON (`ApiError`).
- **“O que você testou?”** — Serviço de auth e jogo, controller de auth com MockMvc; smoke no `AuthService` e `GameComponent` no front (listar arquivos de teste).

### Sobre segurança (vaga menciona prevenção / soluções seguras)

- **“Como armazena senha?”** — Bcrypt em `AuthService.register`; `PasswordEncoder` no Spring.
- **“Como protege endpoints de partida?”** — `SecurityConfig` + `JwtAuthFilter`; rotas `/games/**` autenticadas.
- **“Risco do JWT no browser?”** — Reconhecer; HTTPS em produção, expiração, possível refresh/httpOnly em evolução.

### Sobre frontend

- **“O que é standalone no Angular?”** — `bootstrapApplication`, `app.config.ts`, componentes com `standalone: true` (`main.ts`, `app.component.ts`).
- **“Onde o JWT entra na requisição?”** — `core/interceptors/auth.interceptor.ts`.

### Alinhamento vaga (AWS, dados, IA)

- **“Usou AWS?”** — Neste case não houve requisito; próximo passo seria deploy em ECS/S3+CloudFront, RDS, secrets no Parameter Store.
- **“Onde entra dados/IA?”** — Fora do escopo do Mastermind; na vaga você comenta interesse em pipelines de dados e modelos onde fizer sentido para investigações.

---

## 5. Ordem sugerida de demo (10–15 min)

1. **README** — 1 tela: objetivo + stack + como rodar.  
2. **Login** → **dashboard** (case 3.1 + 3.3).  
3. **Novo jogo** → **partida** → 1–2 tentativas + feedback numérico (case 3.4).  
4. **Ranking** (case 3.3 + regras de desempate que você implementou).  
5. **IDE** — uma volta rápida: `GameController` → `GameService` → `GlobalExceptionHandler` → `SecurityConfig`.  
6. **Testes** — `mvn test` ou mostrar pasta `src/test`.  
7. **Encerramento** — limitações (H2, JWT storage) + próximo passo (Postgres, AWS).

---

## 6. Uma frase de fechamento por bloco do case

- **3.1:** “Login com validação, mensagem de erro, dashboard após sucesso e logout limpando sessão no cliente e token.”  
- **3.2:** “Usuário e partida são entidades JPA; a partida guarda tentativas, tempo, pontuação e código esperado para auditoria.”  
- **3.3:** “Quatro telas pedidas: login, dashboard, jogo e ranking, com services e componentes separados.”  
- **3.4:** “Código secreto só no servidor até o fim; até dez tentativas; feedback numérico por rodada.”  
- **4.x:** “Camadas Spring, validação e handler global, Swagger, componentes Angular com services e testes nos dois lados.”  
- **5:** “README com passos, variáveis de ambiente de exemplo e evidências em GIF/print.”

Boa apresentação.
