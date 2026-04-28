# Materiais para Notebook LM — Mentor técnico + entrevista simulada (Mastermind Itaú)

**Instrução para os apresentadores de IA do Notebook LM (cole também no campo da ferramenta):**

> Apresente o conteúdo como um entrevistador técnico avaliando um candidato experiente. Destaque os motivos das escolhas de stack e as limitações do projeto. Misture tom de mentor pedagógico com perguntas diretas como numa banca. Ao final das seções, faça pausas para “reflexão” e sugira como o candidato pode reformular a resposta para um coordenador que não é especialista em todos os detalhes.

**Como usar:** suba este arquivo como fonte principal. Complementos opcionais: `docs/guia-codigo-fonte.md` (mapa rápido de arquivos) e `docs/notebook-lm-fonte-entrevista.md` (narrativa técnica densa).

---

## Orientação inicial do mentor (som para estudo ou roteiro de vídeo)

Você implementou um desafio clássico de lógica com **persistência**, **usuários**, **critérios de ranqueamento** e **UX clara**. Na apresentação para o coordenador, você não ganha pontos listando 50 classes; ganha quando consegue dizer **em três frases**: (1) o que o sistema faz para o usuário final, (2) como o backend e o front se conversam, (3) qual decisão técnica você defenderia se alguém questionasse. O resto é detalhe que você abre **só se pedirem**, apontando arquivo e método.

Ordem sugerida na banca:

1. **Problema e escopo:** jogo Mastermind no navegador, conta com login/cadastro, recuperação de senha, ranking e streak por dias com vitória.
2. **Arquitetura em uma frase:** SPA Angular falando REST com Spring Boot; autenticação JWT stateless; dados em H2 arquivo no dev com modelo JPA.
3. **Um fluxo ponta a ponta:** exemplo “login até ver o tabuleiro” ou “mandar uma tentativa até receber feedback”. Nomes de pasta podem ficar só no final se pedirem aprofundar.
4. **Qualidade:** menção a testes de serviço/controller no backend, tratamento global de erro, Swagger para documentar API.
5. **Limitações com maturidade:** H2 para protótipo, sem refresh token, sem fila de eventos, etc., e o que faria no próximo sprint.

---

## Por que essa stack (para defender na entrevista)

**Spring Boot 3:** acelera configuração de servidor REST, segurança, injeção de dependências e observabilidade padrão; é o ecossistema mais comum em corporações Java e permite separar bem camadas Controller → Service → Repository.

**Spring Security + JWT:** API REST atrás de SPA não precisa de sessão servidor cheia para cada usuário — o cliente manda credenciais uma vez no login e depois apenas o Bearer. Trade-off declarado na banca: revogar token antes de expirar exigiria lista de revogação ou token de curta duração + refresh pattern (este projeto faz o essencial primeiro).

**JPA/Hibernate:** o domínio `User` e `Game` mapeiam tabelas com validação de negócio nos serviços; troca de SGBD em produção é principalmente config + dialect, não reescrever SQL manual em todo lugar.

**H2 em arquivo:** zero instalação externa para revisar localmente e demonstrar rápido; na entrevista você reconhece que **produção** usaria Postgres ou outro SGBD operacional.

**Angular 18 (standalone components):** front organizável por funcionalidades (`features/auth`, `features/game`), `HttpClient` com interceptor JWT, rotas guardadas para não expor telas privadas só removendo o token do storage.

---

## Como falar para o coordenador (versus para o desenvolvedor sênior)

| Situação | O que você diz |
|---------|----------------|
| Coordenador pergunta “como garante que só eu vejo minha partida?” | “O servidor identifica quem eu sou pelo JWT e relaciona cada jogo ao meu usuário no banco; se outra pessoa tentar o código da partida mas com outro token, a API recusa.” |
| Quer entender ranking | “Quem tem menos tentativas na melhor vitória naquele modo vem primeiro; se empatar, quem foi mais rápido naquele registro; isso está no repositório de usuários e espelhado na regra de atualizar recorde no serviço do jogo.” |
| Pergunta de negócio: “o que é streak?” | “Contagem de dias seguidos em que o jogador venceu pelo menos uma partida; o backend ajusta em login e ao registrar vitória, com fuso de referência para o dia.” |

Evite jargão sem ancorar: em vez de “eu injeto o `AuthenticationManager`”, prefira “o Spring valida usuário e senha no login e só então gera o token”.

---

## Simulação de entrevista técnica — rodada 1: autenticação e segurança

**P:** Onde o token JWT é criado?

**R:** Depois da autenticação bem-sucedida no fluxo de login no `AuthService` (Java em `backend/.../service/AuthService.java`), usando `JwtUtil` com segredo e tempo de vida lidos da configuração.

**P:** Como o servidor sabe que requests seguintes são do mesmo usuário?

**R:** O filtro `JwtAuthFilter` lê `Authorization: Bearer ...`, valida com `JwtUtil`, carrega detalhes com `UserDetailsServiceImpl` (aceita login por username ou e-mail) e monta o contexto de segurança antes de chegar aos controllers protegidos.

**P:** Por que CSRF desligado?

**R:** Porque não estamos usando cookie de sessão para autenticação bearer em chamadas API tratadas assim; CSRF típico ataca estado em cookie quando o navegador envia sozinho. O trade-off já foi assumir que outras superfícies (ex.: form legado mesmo origin) não estão misturadas nesse escopo.

**Pergunta de reflexão:** Se o avaliador disser “e se alguém roubar o token do storage?”, resposta madura: “localStorage é convenção de SPA; mitigações reais são HTTPS, expiração curta, eventual refresh com rotação, e não logar dados sensíveis no front. Eu documentaria esse risco e o aceite para o caso de uso.”

---

## Rodada 2: modelo de dados e persistência

**P:** Como está modelado uma partida?

**R:** Entidade `Game` com código interno esperado como string tipo CSV de cores, estado (em andamento / ganhou / perdeu), tentativas e feedback guardados como JSON textual em campo string para evitar várias linhas relacionais só para histórico (trade-off citável).

**P:** Como o usuário mantém seus melhores resultados?

**R:** Na entidade `User` há campos de melhor desempenho global e por dificuldade (tentativas e duração em segundos), atualizados na vitória via `GameService`, alinhados à ordenação do ranking nos métodos JPQL em `UserRepository`.

---

## Rodada 3: regra de negócio do jogo

**P:** Como o backend calcula feedback de cada tentativa?

**R:** Método `countCorrect`: compara posição por posição o código esperado e o palpite; só conta correto na casa certa. Não há pinos “existem cores certas mas fora de lugar” nesta implementação — é uma decisão de escopo simplificada.

**P:** Quando a partida termina?

**R:** Ganhou quando acertos iguaem o tamanho do código no modo; perde quando atinge o máximo de tentativas sem vitória antes; pode encerrar por desistência (`forfeit`) com derrota e revelação posterior do código no estado.

**P:** Ranking — critérios?

**R:** Queries `findAllForRanking*` ordenam primeiro quem já tem marca válida à frente dos sem marca; menor número de tentativas; menor tempo quando tentativas empatadas; desempate final por identificador de usuário. A mesma lógica qualitativa está em `novoRecorde` na camada de serviço.

---

## Rodada 4: frontend Angular

**P:** Como o front evita chamar jogo sem login?

**R:** `authGuard` nas rotas privadas e token persistido; sem token, redireciona para login.

**P:** Onde o JWT entra em cada request?

**R:** `authInterceptor` adiciona o header Authorization quando o serviço de auth tem token no storage.

**P:** Onde fica a tela do tabuleiro?

**R:** `features/game` — componente carrega estado da partida, envia tentativas, mostra cronômetro com base no instante de início devolvido pela API, e chama desistência se o usuário desistir.

---

## Rodada 5: qualidade, API e próximos passos

**P:** Como documenta a API?

**R:** OpenAPI/springdoc configurado (`OpenApiConfig`); Swagger público conforme segurança do projeto permite testes com Bearer.

**P:** Tem testes?

**R:** Testes automatizados no backend para serviço de auth, serviço de jogo e controller de auth (`mvn test`). Nível ideal futuro incluiria mais integração ponta a ponta ou testes no front por component críticos.

**P:** O que mudaria primeiro em produção?

**R:** Trocar H2 arquivo por Postgres com migration versionada; segredo JWT e credenciais via variável de ambiente ou cofre; healthcheck e readiness; opcionalmente refresh token e auditoria em login.

---

## Guia rápido: roteiro de demonstração falada (aproximadamente 8 a 12 minutos)

**Minutos 1–2:** Mostrar README ou tela inicial; dizer problema e stack em uma vez.

**Minutos 3–5:** Demo ao vivo: cadastro rápido ou login, escolha de modo, jogar até uma jogada bem-sucedida ou derrota, mostrar cronômetro e ranking.

**Minutos 6–8:** Abrir dois arquivos no IDE como prova técnica: `SecurityConfig.java` (o que está aberto público versus protegido) e um trecho curto de `GameService.submitAttempt` (onde decide vitória ou continuação).

**Minutos finais:** Uma vulnerabilidade ou limitação conscientemente assumida — por exemplo JWT em storage — e uma melhoria concreta. Fecha mostrando `GlobalExceptionHandler` ou teste automatizado como sinal de cuidado com regressão.

---

## Lista fechada: limitações aceitável na banca com resposta ganha pontos de maturidade

- **JWT em localStorage:** convenção de SPA demo; refinamento seria política httpOnly cookie + backend BFF ou refresh curtíssimo.
- **Histórico de tentativas em JSON em coluna:** simples para o case; relatórios analíticos exigiriam normalização.
- **H2 arquivo:** suficiente para desenvolvimento; produção migra JDBC e pooling.
- **Sem WebSocket:** tempo atualizado cliente com estado devolvido pela API suficiente para o caso; escala realtime seria canal separado.
- **Streak usa regra temporal em fuso São Paulo definido no projeto:** assumir comunicação dessa decisão quando negócio falar “dia” como calendário local.

---

## Fechamento do mentor ao candidato

Você tem três ferramentas quando o coordenador fizer perguntas difíceis:

1. **Demonstrar comportamento:** rodar o projeto e narrar ao mesmo tempo.
2. **Apontar arquivo e método** sem ler linha por linha: “no serviço de jogo eu centralizo todas as decisões sobre fim de partida”.
3. **Converter limitação em backlog:** sempre acoplar “o que não fiz” com “qual seria o próximo incremental realista”. Isso soa menos desculpa e mais roadmap.

Fim do documento-fonte para geração de áudio ou vídeo com foco mentor + entrevistador técnico + orientação ao coordenador.
