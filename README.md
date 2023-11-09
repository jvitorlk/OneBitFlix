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

## 4 - Criando o Primeiro Resource

Nessa aula vamos criar nosso primeiro recurso. Para isso vamos seguir um passo a passo que será repetido muitas vezes durante o desenvolvimento, criar a migration, rodar a migration, criar o model, depois incluir o model no arquivo de associações e então incluir suas opções no AdminJs

- Em primeiro lugar, crie a migration responsável por criar/excluir a tabela “categories”:

```typescript
npx sequelize-cli migration:generate --name create-categories-table
```

- Isso criará um arquivo JS na pasta migrations com uma timestamp de criação. Adicione o código para criar e excluir a tabela nos métodos up e down respectivamente:
  - Obs.: Repare que na criação da tabela precisamos especificar todas as colunas no padrão snake_case, incluindo as timestamps.

```typescript
// src/database/migrations/XXXXXXXXXXXXXX-create-categories-table

"use strict";

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable("categories", {
      id: {
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
        type: Sequelize.DataTypes.INTEGER,
      },
      name: {
        allowNull: false,
        type: Sequelize.DataTypes.STRING,
      },
      position: {
        allowNull: false,
        unique: true,
        type: Sequelize.DataTypes.INTEGER,
      },
      created_at: {
        allowNull: false,
        type: Sequelize.DataTypes.DATE,
      },
      updated_at: {
        allowNull: false,
        type: Sequelize.DataTypes.DATE,
      },
    });
  },

  async down(queryInterface, Sequelize) {
    await queryInterface.dropTable("categories");
  },
};
```

- Execute as migrations através da sequelize-cli:

```typescript
npx sequelize-cli db:migrate
```

- Opcionalmente, acesse o banco de dados para verificar que as tabelas foram criadas:

```typescript
sudo -u postgres psql -d onebitflix_development

\dt
```

- Crie um arquivo chamado Category.ts na pasta models. Adicione as interfaces necessárias nele e a parte de criação do model:
  - Obs.: O sequelize permite declarar models usando classes ou a função “define()”, optaremos pela abordagem funcional.
  - Obs².: Nesse caso, a forma como o sequelize trabalha com typescript exige a criação de algumas interfaces específicas. Seus nomes são arbitrários, mas a documentação segue o seguinte formato:
    - ModelAttributes - para todos os atributos do model (timestamps são inclusas automaticamente)
    - ModelCreationAttributes - para os atributos usados na instanciação do model, ou seja, o id é especificado aqui como opcional porque ele não existe inicialmente. Também é possível especificar outras propriedades opcionais utilizando | (union type).
    - ModelInstance - uma interface para a instancia do model. Nessa interface também é possível definir dentro das chaves atributos e métodos extras manualmente para o typescript reconheça na instância. Por enquanto as chaves podem ficar vazias, mas faremos isso mais a frente.
  - Obs³.: No define() utilizamos os _generics_ usando essas interfaces que criamos para indicar quais serão as propriedades do model.

```typescript
// src/models/Category.ts

import { sequelize } from "../database";
import { DataTypes, Model, Optional } from "sequelize";

export interface Category {
  id: number;
  name: string;
  position: number;
}

export interface CategoryCreationAttributes extends Optional<Category, "id"> {}

export interface CategoryInstance
  extends Model<Category, CategoryCreationAttributes>,
    Category {}

export const Category = sequelize.define<CategoryInstance, Category>(
  "Category",
  {
    id: {
      allowNull: false,
      autoIncrement: true,
      primaryKey: true,
      type: DataTypes.INTEGER,
    },
    name: {
      allowNull: false,
      type: DataTypes.STRING,
    },
    position: {
      allowNull: false,
      unique: true,
      type: DataTypes.INTEGER,
    },
  }
);
```

- Crie um arquivo index.ts na pasta models. Por enquanto ele não terá muito conteúdo, mas iremos tratar as associações nesse arquivo e sempre importaremos os models a partir dele:

```typescript
// src/models/index.ts

import { Category } from "./Category";

export { Category };
```

- Crie na pasta adminjs uma pasta resources. Dentro de resources crie um arquivo category.ts:
  - Obs.: Nesse objeto resourceOptions vamos definir as configurações do model dentro do AdminJS, como quais campos estão presentes no formulário, na tabela de visualização, em qual grupo na barra lateral ele vai estar, etc.

```typescript
// src/adminjs/resources/category.ts

import { ResourceOptions } from "adminjs";

export const categoryResourceOptions: ResourceOptions = {
  navigation: "Catálogo",
  editProperties: ["name", "position"],
  filterProperties: ["name", "position", "createdAt", "updatedAt"],
  listProperties: ["id", "name", "position"],
  showProperties: ["id", "name", "position", "createdAt", "updatedAt"],
};
```

- Ainda na pasta resources, crie um arquivo index.ts:
  - Obs.: Esse arquivo é um array de todas as opções de models que vamos definir. Ele serve apenas para deixar mais organizado.

```typescript
// src/adminjs/resources/index.ts

import { ResourceWithOptions } from "adminjs";
import { Category } from "../../models";
import { categoryResourceOptions } from "./category";

export const adminJsResources: ResourceWithOptions[] = [
  {
    resource: Category,
    options: categoryResourceOptions,
  },
];
```

- Inclua os resources no arquivo de configuração em config/adminjs.ts:

```typescript
// src/adminjs/index.ts

// ...
import { database } from '../database'
import { adminJsResources } from './resources'

// ...
	databases: [database],
  resources: adminJsResources,
  rootPath: '/admin',
//...
```

- Teste verificando que o recurso já pode ser gerenciado pelo painel de administração:

```bash
npm run dev
```

## 5 - Relacionamentos entre Modelos

Nessa aula vamos fazer um procedimento parecido com o da última aula, mas vamos criar também a primeira associação entre tabelas/modelos.

- Crie a migration para a tabela courses:

```javascript
// No terminal
npx sequelize-cli migration:generate --name create-courses-table
```

- Adicione o conteúdo da migration:
  - Obs.: Destaque para a definição da chave estrangeira.

```typescript
// src/database/migrations/XXXXXXXXXXXXXX-create-titles-table.js

"use strict";

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable("courses", {
      id: {
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
        type: Sequelize.DataTypes.INTEGER,
      },
      name: {
        allowNull: false,
        type: Sequelize.DataTypes.STRING,
      },
      synopsis: {
        allowNull: false,
        type: Sequelize.DataTypes.TEXT,
      },
      thumbnail_url: {
        type: Sequelize.DataTypes.STRING,
      },
      featured: {
        defaultValue: false,
        type: Sequelize.DataTypes.BOOLEAN,
      },
      category_id: {
        allowNull: false,
        type: Sequelize.DataTypes.INTEGER,
        references: { model: "categories", key: "id" },
        onUpdate: "CASCADE",
        onDelete: "RESTRICT",
      },
      created_at: {
        allowNull: false,
        type: Sequelize.DataTypes.DATE,
      },
      updated_at: {
        allowNull: false,
        type: Sequelize.DataTypes.DATE,
      },
    });
  },

  async down(queryInterface, Sequelize) {
    await queryInterface.dropTable("courses");
  },
};
```

- Crie o model Course.ts:
  - Obs.: Aqui já dá pra ver que no model nós usamos camelCase ao invés de snake_case como no banco de dados.

```typescript
// src/models/Course.ts

import { sequelize } from "../database";
import { DataTypes, Model, Optional } from "sequelize";

export interface Course {
  id: number;
  name: string;
  synopsis: string;
  thumbnailUrl: string;
  featured: boolean;
  categoryId: number;
}

export interface CourseCreationAttributes
  extends Optional<Course, "id" | "thumbnailUrl" | "featured"> {}

export interface CourseInstance
  extends Model<Course, CourseCreationAttributes>,
    Course {}

export const Course = sequelize.define<CourseInstance, Course>("Course", {
  id: {
    allowNull: false,
    autoIncrement: true,
    primaryKey: true,
    type: DataTypes.INTEGER,
  },
  name: {
    allowNull: false,
    type: DataTypes.STRING,
  },
  synopsis: {
    allowNull: false,
    type: DataTypes.TEXT,
  },
  thumbnailUrl: {
    type: DataTypes.STRING,
  },
  featured: {
    defaultValue: false,
    type: DataTypes.BOOLEAN,
  },
  categoryId: {
    allowNull: false,
    type: DataTypes.INTEGER,
    references: { model: "categories", key: "id" },
    onUpdate: "CASCADE",
    onDelete: "RESTRICT",
  },
});
```

- Adicione o model Course ao index.ts da pasta models e os relacionamentos entre Category e Course:

```typescript
// src/models/index.ts

import { Category } from "./Category";
import { Course } from "./Course";

Category.hasMany(Course);

Course.belongsTo(Category);

export { Course, Category };
```

- Crie o arquivo de opções do resource course.ts:
  - Obs.: A propriedade “uploadThumbnail” ainda não existe, mas ela servirá para representar o input de upload no formulário. Incluiremos ela no futuro.

```typescript
// src/adminjs/resources/course.ts

import { ResourceOptions } from "adminjs";

export const courseResourceOptions: ResourceOptions = {
  navigation: "Catálogo",
  editProperties: [
    "name",
    "synopsis",
    "uploadThumbnail",
    "featured",
    "categoryId",
  ],
  filterProperties: [
    "name",
    "synopsis",
    "featured",
    "categoryId",
    "createdAt",
    "updatedAt",
  ],
  listProperties: ["id", "name", "featured", "categoryId"],
  showProperties: [
    "id",
    "name",
    "synopsis",
    "featured",
    "thumbnailUrl",
    "categoryId",
    "createdAt",
    "updatedAt",
  ],
};
```

- E o inclua em resources.ts:

```typescript
// src/adminjs/resources.ts

import { ResourceWithOptions } from "adminjs";
import { Category, Course } from "../../models";
import { categoryResourceOptions } from "./category";
import { courseResourceOptions } from "./course";

export const adminJsResources: ResourceWithOptions[] = [
  {
    resource: Course,
    options: courseResourceOptions,
  },
  {
    resource: Category,
    options: categoryResourceOptions,
  },
];
```

- Faça o teste pelo painel de administração e veja que é possível cadastrar categorias e cadastrar cursos ligados a uma determinada categoria.
