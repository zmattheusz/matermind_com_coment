Case Eng full-stack JR 1. Contexto e Objetivo
Você foi contratado(a) para desenvolver um jogo digital baseado no clássico Mastermind. Sua
missão é desenvolver uma versão web funcional do jogo, com base nos requisitos descritos
abaixo.
Objetivo do case: Avaliar sua capacidade de estruturar um projeto do zero, escrever código
limpo e organizado, tomar decisões técnicas coerentes e entregar uma solução funcional
dentro do prazo.
Não existe resposta única correta — valorizamos raciocínio, clareza e consistência.

2. Stack Tecnológica
Você tem liberdade para escolher as tecnologias a serem utilizadas. A recomendação abaixo
se baseia nos desafios atuais da posição disponível no time, por isso é uma Stack
Recomendada, mas não mandatória/eliminatória:
Camada Tecnologias
Backend Java (Spring Boot) ou Python
Frontend Angular (15 ou superior)
Banco de dados Qualquer relacional ou não relacional

Gerenciador Maven (Java) / pip (Python) / npm (Angular)
3. Requisitos Funcionais
3.1 Autenticação
• Tela de login com campos de usuário (e-mail ou nome de usuário) e senha, com
validação de formulário.
• Redirecionamento para o dashboard após login bem-sucedido.
Mensagem de erro amigável em caso de credenciais inválidas.
• Opção de logout que redireciona o usuário para a tela de login.

3.2 Dados
• Cada usuário deve possuir um registro único na base, para guardar seu histórico de
partidas e melhor pontuação.
• Cada partida deve ser salva com identificação do usuário, pontuação final, matriz
completa com todas as tentativas para auditar a partida, tempo de duração do jogo e
a resposta esperada para aquela partida.

3.3 Visual (Frontend)
• O frontend deve conter minimamente:
o Tela de Login
o Dashboard inicial (menu para iniciar jogo, visualizar ranking etc.)
o Tela do jogo em andamento
o Tela de ranking ordenado por desempenho dos jogadores
• Recomendável priorizar usabilidade e responsividade nas interfaces.

3.4 Regras do Jogo
• Ao iniciar uma partida, o backend deve gerar um novo registro contendo o resultado
esperado, um código único da partida e o identificador do jogador.
• O frontend deve montar uma matriz visual para o usuário no mesmo formato do
tabuleiro do Mastermind (veja exemplo abaixo).
• A cada tentativa submetida pelo usuário, o frontend aciona o backend para validar o
resultado; o backend responde apenas com o número de acertos (em nenhum
momento o frontend sabe a combinação correta).
• O usuário terá no máximo 10 tentativas para adivinhar a combinação.
• O frontend deve atualizar visualmente o feedback para cada tentativa realizada.
Exemplo Visual:

#PraTodosVerem Imagem de um tabuleiro Mastermind, contendo no topo um retângulo com 4 bolinhas com as letras de A a D, representando as 4 cores resultado
da partida atual. Abaixo a imagem possui 10 linhas que seriam as tentativas de cada turno. Em cada linha tem um exemplo semelhante ao do resultado, onde o
usuário irá selecionar a cor de cada letra, e ao final da linha possui um quadrado dividido em 4 quadrantes, onde será apresentado quantas cores a pessoa acertou
no turno atual.
Cada linha representa uma tentativa; os quadrantes à direita indicam apenas a quantidade de
células corretas, não suas posições. Este é apenas um modelo de exemplo de um tabuleiro
Mastermind, mas você tem liberdade para definir a melhor forma visual de representar o
tabuleiro, ou se basear em outros modelos existentes.

4. Requisitos Não Funcionais
4.1 Backend
• Estrutura em camadas: Controller → Service → Repository (ou padrões de arquitetura
equivalentes).
• Validação de todos os inputs com mensagens de erro padronizadas.
• Tratamento global de exceções com respostas HTTP adequadas (400, 401, 404, 500).
• Documentação mínima da API (Swagger/OpenAPI ou similar) é um diferencial.

4.2 Frontend
• Componentização adequada — evite lógica de negócio nos templates.
• Uso de Services para comunicação com a API.
• Tratamento de erros vindos da API com feedback visual ao usuário.
• Validações robustas nos formulários e inputs dos usuários.

4.3 Gerais
• Testes unitários, funcionando para o back e frontend
• Ambiente configurável para execução local, com guia para execução na documentação
do projeto.

5. Instruções de Entrega
5.1 Repositório
• Crie um repositório público no GitHub.
• Organize o projeto, minimamente, em duas pastas na raiz: /backend e /frontend .
• Envie o link do repositório no formulário disponibilizado até o prazo estabelecido.

5.2 Documentação
• README com:
o Breve descrição da solução e das decisões técnicas relevantes.
o Pré-requisitos para rodar o projeto (versões de Java/Python, Node etc.).
o Passo a passo para rodar o backend localmente.
o Passo a passo para rodar o frontend localmente.
o Relação das variáveis de ambiente necessárias (não inclua valores reais —
utilize um `.env.example`).
o Sugestão: Inclua prints ou GIFs demonstrando o funcionamento da aplicação.
• Documentação da API
5.3 O que será avaliado
• Código completo (back e front).

• Documentação clara
• Aplicação funcionando local para testes
• Regras do jogo atendidas (O jogo deve ser funcional, permitindo partidas completas, e
testes do ranking das partidas anteriores durante a execução).