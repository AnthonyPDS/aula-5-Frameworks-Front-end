# Questões para Refletir — Aula 05 (Criando APIs para o Front-end)

---

### 1. Quais são os riscos de segurança desse projeto e como poderiam ser eliminados ou mitigados?

- **CORS totalmente aberto (`Access-Control-Allow-Origin: *`)**: Qualquer site na internet pode fazer requisições para a API.
  - *Mitigação*: Restringir a origem no middleware `cors({ origin: 'https://seu-frontend.vercel.app' })` apenas aos domínios autorizados.
- **Ausência de Autenticação e Autorização**: Qualquer pessoa com a URL da API pode criar, editar ou apagar todas as notas.
  - *Mitigação*: Implementar autenticação via **JWT (JSON Web Tokens)** ou OAuth 2.0 e verificar permissões antes de alterar/excluir registros.
- **Falta de Validação e Sanitização de Entrada (`req.body`)**: O servidor aceita qualquer dado enviado em `titulo` e `texto`, abrindo brecha para payloads malformados, objetos gigantes ou injeção de scripts (XSS armazenado caso o front-end renderize HTML puro).
  - *Mitigação*: Validar tipos e tamanho máximo dos campos com bibliotecas como **Zod**, **Joi** ou **express-validator**, além de limitar o tamanho do body (`express.json({ limit: '10kb' })`).
- **Ausência de Rate Limiting**: Um atacante pode disparar milhares de requisições `POST` por segundo, esgotando o disco e a memória do servidor (ataque DoS).
  - *Mitigação*: Utilizar `express-rate-limit` e cabeçalhos de segurança com `helmet`.

---

### 2. Usar um arquivo JSON (`data.json`) para armazenar notas é uma boa prática em produção? Quais são as vantagens e desvantagens dessa abordagem?

**Não é uma boa prática em produção.** É excelente para aprendizado e prototipagem rápida, mas inadequado para ambientes reais.

- **Vantagens**:
  - Zero configuração de infraestrutura (não exige instalar nem configurar SGBD).
  - Fácil de inspecionar, editar manualmente e versionar em pequenos testes.
  - Ideal para fins didáticos e compreensão do fluxo HTTP/CRUD.
- **Desvantagens**:
  - **Condições de corrida (*Race Conditions*)**: Se duas requisições simultâneas lerem e gravarem o arquivo ao mesmo tempo, uma sobrescreverá os dados da outra.
  - **Armazenamento efêmero em nuvem (como no plano gratuito do Render)**: Sempre que o contêiner reinicia ou ocorre novo deploy, as alterações feitas no `data.json` em tempo de execução são perdidas.
  - **Falta de propriedades ACID**: Se o processo cair no meio de um `writeFileSync`, o arquivo JSON pode ficar corrompido.

---

### 3. Quais limitações um servidor baseado em arquivo JSON teria se o número de notas crescesse para 10.000 registros?

- **Bloqueio da Event Loop do Node.js**: As funções `fs.readFileSync` e `fs.writeFileSync` são síncronas. Ler, fazer `JSON.parse()` e `JSON.stringify()` de 10.000 registros a cada requisição travaria a *thread* principal do Node.js, impedindo o servidor de responder outros usuários enquanto o arquivo é lido ou gravado.
- **Ausência de Índices (Busca Linear `O(n)`)**: Operações como `findIndex` e `filter` percorrem todos os 10.000 itens em memória a cada consulta, edição ou exclusão, ao contrário de um banco de dados com índices B-Tree ou Hash (`O(log n)` ou `O(1)`).
- **Alto Consumo de Memória e I/O de Disco**: Mesmo para alterar uma única nota, o servidor carrega todo o arquivo na memória RAM e reescreve os 10.000 registros inteiros no disco.

---

### 4. O código atual está todo em `server.js`. Por que isso é problemático e como você melhoraria a organização da lógica?

- **Por que é problemático**:
  - Viola o princípio de responsabilidade única (SRP): configuração do servidor, middlewares, definição de rotas, regras de negócio e acesso a dados estão misturados em um único arquivo.
  - Dificulta a manutenção, a escalabilidade (adicionar novas entidades como usuários ou categorias tornaria o arquivo gigante) e a criação de testes unitários isolados.
- **Como melhorar a organização (Arquitetura em Camadas / MVC)**:
  - `src/routes/notes.routes.js`: Define apenas os endpoints Express (`GET`, `POST`, `PUT`, `DELETE`).
  - `src/controllers/notes.controller.js`: Recebe `req` e `res`, valida parâmetros e retorna os status HTTP adequados.
  - `src/services/notes.service.js`: Contém as regras de negócio da aplicação.
  - `src/repositories/notes.repository.js` (ou `models/`): Abstrai a leitura/escrita de dados (permitindo trocar o `data.json` por PostgreSQL, MongoDB ou SQLite sem alterar as rotas ou controllers).
  - `src/app.js` e `server.js`: Apenas configuram middlewares globais e inicializam a porta HTTP.
