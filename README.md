# OneBitFlix

EAD inspirado na Netflix

## 1 - Iniciando o Projeto

- Crie uma pasta para o projeto, inicialize um projeto do Node.js e instale as dependências:
  - Obs.: Repare que o @ serve para especificar a versão a ser instalado e o ~ serve para que o npm seja flexível com atualizações menores, então se houver alguma nova versão menor (o último dígito da versão) o pacote será instalado na mais recente.

```javascript
npm init -y
npm i express@~4.17.2 express-formidable@~1.2.0 adminjs@~5.5.1 @adminjs/express@~4.0.1 @adminjs/sequelize@~2.1.0 @adminjs/upload@~2.0.1 pg@~8.7.1 sequelize@~6.13.0
```

- Instale as dependências de desenvolvimento:

```javascript
npm i -D typescript@~4.5.4 ts-node-dev@~1.1.8 sequelize-cli@~6.4.1 @types/express@~4.17.13 @types/node@~17.0.10
```

- Se estiver usando o Git, crie o arquivo .gitignore na raiz do projeto com o seguinte conteúdo:

```javascript
node_modules/
```

- Crie uma pasta src e um arquivo server.ts para ser o ponto de entrada:
  - Obs.: Neste momento vamos apenas fazer um teste para ver que o express está funcionando corretamente.

```javascript
// src/server.ts

import express from "express";

const app = express();

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`Server started successfuly at port ${PORT}`);
});
```

- Inclua o script para executar o projeto com o ts-node-dev ao seu package.json:

```javascript
"scripts": {
	"dev": "ts-node-dev --transpile-only --ignore-watch node_modules src/server.ts"
},
```

- Crie o arquivo tsconfig.json na raiz do projeto (ou utilize o comando “npx tsc --init”):
  - Obs.: Aqui é legal destacar que o “module”: “commonjs” e o “esModuleInterop”: true são importantes para que o typescript funcione sem problemas com o Node.js.

```javascript
{
  "compilerOptions": {
    "target": "es2016",
    "module": "commonjs",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true
  }
}
```

Explicação para cada opção no tsconfig.json: [https://www.typescriptlang.org](https://www.typescriptlang.org)

- Ao executar a aplicação deve estar funcionando:

```javascript
npm run dev
```

## 2 - Configurando o Sequelize

A sequelize-cli será utilizada para gerarmos alguns boilerplates através de comandos do terminal e executar tarefas como criar o banco de dados, rodar migrations, rodar seeders, etc.

- Para configurar as opções da sequelize-cli, crie uma pasta config na raiz do projeto e o arquivo sequelizeCli.js dentro dela:
  - Obs.: A sequelize-cli precisa de acesso ao banco de dados para rodar a parte do nosso app, por isso temos que configurá-la separadamente (depois faremos uma configuração semelhante para nosso próprio app).
  - Obs².: Como estamos utilizando um arquivo js para essas configurações esses valores poderiam ser obtidos através de variáveis de ambiente, mas no momento não faremos isso (para simplificar).

```javascript
// config/sequelizeCli.js

module.exports = {
  development: {
    dialect: "postgres",
    host: "localhost",
    port: "5432",
    database: "onebitflix_development",
    username: "onebitflix",
    password: "onebitflix",
  },
};
```

- Na raiz do projeto crie o arquivo .sequelizerc:
  - Obs.: Esse arquivo é muito importante. Como estamos fazendo nossa própria estrutura precisamos “dizer” para a sequelize-cli que queremos que ela procure/gere arquivos nesses caminhos específicos. Por padrão esses caminhos seriam outros.

```javascript
// .sequelizerc

const path = require("path");

module.exports = {
  config: path.resolve("config", "sequelizeCli.js"),
  "models-path": path.resolve("src", "models"),
  "seeders-path": path.resolve("src", "database", "seeders"),
  "migrations-path": path.resolve("src", "database", "migrations"),
};
```

- Caso ainda não possua um usuário no PostgreSQL, crie um com a permissão CREATEDB:

```shell
sudo -u postgres psql

CREATE USER onebitflix WITH CREATEDB ENCRYPTED PASSWORD 'onebitflix';
```

- Agora já será possível utilizar a sequelize-cli para criar o banco de dados:

```bash
npx sequelize-cli db:create
```

- Crie as pastas necessárias conforme especificamos no arquivo .sequelizerc:

```bash
mkdir src/models src/database src/database/migrations
src/database/seeders
```

- Por fim, crie o arquivo de conexão com o banco de dados em um arquivo index.ts na pasta src/database:
  - Obs.: Agora sim, a conexão que será usada pela nossa aplicação. Mais uma vez, o ideal seria não ter nenhuma senha exposta aqui no código, e sim através de variáveis de ambiente. Mas faremos isso posteriormente (para simplificar).
  - Obs².: Aqui é legal destacar que o underscored é bem importante. Ele serve para que o sequelize converta automaticamente os nomes das colunas de snake_case (convenção no SQL) para camelCase (convenção no Javascript), assim referenciaremos nossas tabelas (no BD) usando snake_case e nossos models (no app) usando camelCase.

```typescript
// src/database/index.ts

import { Sequelize } from "sequelize";

export const database = new Sequelize({
  dialect: "postgres",
  host: "localhost",
  port: 5432,
  database: "onebitflix_development",
  username: "onebitflix",
  password: "onebitflix",
  define: {
    underscored: true,
  },
});
```

- É possível testar a conexão atualizando o arquivo server.ts dessa forma:
  - Obs.: O método authenticate() só serve para executar uma query 1 + 1 de teste e ver se a conexão funciona.

```typescript
// src/server.ts

import express from "express";
import { database } from "./database";

const app = express();

const PORT = process.env.port || 3000;

app.listen(PORT, () => {
  database.authenticate().then(() => {
    console.log("DB connection successfull.");
  });

  console.log(`Server started successfuly at port ${PORT}.`);
});
```

## 3 - Configurando o Painel do AdminJS

Nessa aula vamos colocar o painel do AdminJS para funcionar.

- Crie a pasta “adminjs” dentro de “src” e o arquivo index.ts dentro dela contendo o registro do adapter do sequelize e a criação de um router a partir de uma instância da classe AdminJs:
  - Obs.: Aqui é legal mostrar que o Typescript ajuda no autocomplete das opções, o que é muito útil.

```typescript
// src/adminjs/index.ts

import AdminJs from "adminjs";
import AdminJsExpress from "@adminjs/express";
import AdminJsSequelize from "@adminjs/sequelize";
import { sequelize } from "../database";

AdminJs.registerAdapter(AdminJsSequelize);

export const adminJs = new AdminJs({
  databases: [sequelize],
  rootPath: "/admin",
  branding: {
    companyName: "OneBitFlix",
    logo: "/onebitflix.svg",
    theme: {
      colors: {
        primary100: "#ff0043",
        primary80: "#ff1a57",
        primary60: "#ff3369",
        primary40: "#ff4d7c",
        primary20: "#ff668f",
        grey100: "#151515",
        grey80: "#333333",
        grey60: "#4d4d4d",
        grey40: "#666666",
        grey20: "#dddddd",
        filterBg: "#333333",
        accent: "#151515",
        hoverBg: "#151515",
      },
    },
  },
});

export const adminJsRouter = AdminJsExpress.buildRouter(adminJs);
```

- Crie uma pasta public na raiz do projeto e dentro dela adicione o arquivo do logo onebitflix.svg.
- No arquivo server.ts adicione o middleware de arquivos estáticos do express e o router do AdminJS que criamos:

```typescript
import express from "express";
import { database } from "./database";
import { adminJs, adminJsRouter } from "./config/adminjs";

const app = express();

app.use(express.static("public"));

app.use(adminJs.options.rootPath, adminJsRouter);

const PORT = process.env.port || 3000;

app.listen(PORT, async () => {
  await database.authenticate().then(() => {
    console.log("DB connection successfull.");
  });

  console.log(`Server started successfuly at port ${PORT}.`);
});
```

- Teste acessando o endereço [http://localhost:3000/admin](http://localhost:3000/admin)
