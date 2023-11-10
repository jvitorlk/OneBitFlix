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

## 6 - Criando Seeders

- Para criarmos seeders para preencher nossas tabelas automaticamente podemos fazer como nas migrations. Gere o seeder com o comando abaixo:

```javascript
npx sequelize-cli seed:generate --name seed-categories-table
```

- Agora vamos preencher o seeder da tabela categories:
  - Obs.: O seeder utiliza a mesma estrutura da migration, um objeto com um método up para quando for executar e um método down para quando for reverter os seeds.

```javascript
// src/database/seeders/XXXXXXXXXXXXXX-seed-categories-table.js

"use strict";

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.bulkInsert(
      "categories",
      [
        {
          name: "Tecnologias Back-end",
          position: 1,
          created_at: new Date(),
          updated_at: new Date(),
        },
        {
          name: "Tecnologias Front-end",
          position: 2,
          created_at: new Date(),
          updated_at: new Date(),
        },
        {
          name: "Ferramentas de Desenvolvimento",
          position: 3,
          created_at: new Date(),
          updated_at: new Date(),
        },
        {
          name: "Soft-skills",
          position: 4,
          created_at: new Date(),
          updated_at: new Date(),
        },
        {
          name: "Carreira",
          position: 5,
          created_at: new Date(),
          updated_at: new Date(),
        },
      ],
      {}
    );
  },

  async down(queryInterface, Sequelize) {
    await queryInterface.bulkDelete("categories", null, {});
  },
};
```

- Depois disso vamos criar um seeder para a tabela courses:

```javascript
npx sequelize-cli seed:generate --name seed-courses-table
```

- Depois disso vamos criar um seeder para a tabela courses:

```javascript
npx sequelize-cli seed:generate --name seed-courses-table
```

- E então vamos adicionar o seu conteúdo, primeiro obtendo as categorias existentes e então criando cursos para cada uma delas:

```javascript
// src/database/seeders/XXXXXXXXXXXXXX-seed-courses-table.js

"use strict";

module.exports = {
  async up(queryInterface, Sequelize) {
    const [categories] = await queryInterface.sequelize.query(
      "SELECT id FROM categories;"
    );

    await queryInterface.bulkInsert("courses", [
      {
        name: "Programador Full-stack Javascript",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        featured: true,
        category_id: categories[0].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
      {
        name: "Dominando a Linguagem Ruby",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        category_id: categories[0].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
      {
        name: "Micro-serviços com Node.js",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        featured: true,
        category_id: categories[0].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
      {
        name: "Criando APIs Profissionais com Ruby on Rails",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        featured: true,
        category_id: categories[0].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
      {
        name: "TDD na Prática: Testando APIs Node.js",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        featured: true,
        category_id: categories[0].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
      {
        name: "TDD na Prática: Testando Aplicações React",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        featured: true,
        category_id: categories[1].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
      {
        name: "Especialista Front-end: Vue.js",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        category_id: categories[1].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
      {
        name: "Criando Sites e Apps 3D com Three.js",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        category_id: categories[1].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
      {
        name: "Dominando o Bootstrap 5",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        category_id: categories[1].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
      {
        name: "Visual Studio Code para Programadores Javascript",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        category_id: categories[2].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
      {
        name: "Comandos do Terminal Linux: Um Guia Completo",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        category_id: categories[2].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
      {
        name: "Comunicação e Trabalho em Equipe",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        category_id: categories[3].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
      {
        name: "Programador Nômade",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        featured: true,
        category_id: categories[4].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
      {
        name: "O Guia do Programador Freelancer",
        synopsis:
          "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
        category_id: categories[4].id,
        created_at: new Date(),
        updated_at: new Date(),
      },
    ]);
  },

  async down(queryInterface, Sequelize) {
    await queryInterface.bulkDelete("courses", null, {});
  },
};
```

- Por fim, só precisamos rodar os seeder com o comando:

```javascript
npx sequelize-cli db:seed:all
```

- Esses seeds não irão incluir as capas dos cursos, mas você pode fazer isso através do painel do AdminJS.

## 7 - Tabela de Episódios

- Comece gerando a migration para a tabela episodes através da sequelize-cli:

```javascript
npx sequelize-cli migration:generate --name create-episodes-table
```

- Adicione o conteúdo da migration:

```javascript
// src/database/migrations/XXXXXXXXXXXXXX-create-episodes-table.js

"use strict";

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable("episodes", {
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
      order: {
        allowNull: false,
        type: Sequelize.DataTypes.INTEGER,
      },
      video_url: {
        type: Sequelize.DataTypes.STRING,
      },
      seconds_long: {
        type: Sequelize.DataTypes.INTEGER,
      },
      course_id: {
        allowNull: false,
        type: Sequelize.DataTypes.INTEGER,
        references: { model: "courses", key: "id" },
        onUpdate: "CASCADE",
        onDelete: "RESTRICT",
      },
      created_at: {
        allowNull: false,
        type: Sequelize.DATE,
      },
      updated_at: {
        allowNull: false,
        type: Sequelize.DATE,
      },
    });
  },

  async down(queryInterface, Sequelize) {
    await queryInterface.dropTable("episodes");
  },
};
```

- Não se esqueça de executar a migration:

```typescript
npx sequelize-cli db:migrate
```

- Crie o model Episode.ts e adicione o seu conteúdo:

```typescript
// src/models/Episode.ts

import { sequelize } from "../database";
import { DataTypes, Model, Optional } from "sequelize";

export interface Episode {
  id: number;
  name: string;
  synopsis: string;
  order: number;
  videoUrl: string;
  secondsLong: number;
  courseId: number;
}

export interface EpisodeCreationAttributes
  extends Optional<Episode, "id" | "videoUrl" | "secondsLong"> {}

export interface EpisodeInstance
  extends Model<Episode, EpisodeCreationAttributes>,
    Episode {}

export const Episode = sequelize.define<EpisodeInstance, Episode>("Episode", {
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
  order: {
    allowNull: false,
    type: DataTypes.STRING,
  },
  videoUrl: {
    type: DataTypes.STRING,
  },
  secondsLong: {
    type: DataTypes.INTEGER,
  },
  courseId: {
    allowNull: false,
    type: DataTypes.INTEGER,
    references: { model: "courses", key: "id" },
    onUpdate: "CASCADE",
    onDelete: "RESTRICT",
  },
});
```

- E agora no arquivo index.ts da pasta model inclua as associações de Episode e Course:

```typescript
// src/models/index.ts

import { Category } from "./Category";
import { Course } from "./Course";
import { Episode } from "./Episode";

Category.hasMany(Course);

Course.belongsTo(Category);
Course.hasMany(Episode);

Episode.belongsTo(Course);

export { Category, Course, Episode };
```

- Na pasta resources crie o arquivo episode.ts para as opções do recurso episodes:

```typescript
// src/adminjs/resources/episode.ts

import { ResourceOptions } from "adminjs";

export const episodeResourceOptions: ResourceOptions = {
  navigation: "Catálogo",
  editProperties: [
    "name",
    "synopsis",
    "courseId",
    "order",
    "uploadVideo",
    "secondsLong",
  ],
  filterProperties: [
    "name",
    "synopsis",
    "courseId",
    "secondsLong",
    "createdAt",
    "updatedAt",
  ],
  listProperties: ["id", "name", "courseId", "order", "secondsLong"],
  showProperties: [
    "id",
    "name",
    "synopsis",
    "courseId",
    "order",
    "videoUrl",
    "secondsLong",
    "createdAt",
    "updatedAt",
  ],
};
```

- E no arquivo resources.ts inclua o recurso e suas opções:

```typescript
// src/adminjs/resources.ts

import { ResourceWithOptions } from "adminjs";
import { Category, Course, Episode } from "../../models";
import { categoryResourceOptions } from "./category";
import { courseResourceOptions } from "./course";
import { episodeResourceOptions } from "./episode";

export const adminJsResources: ResourceWithOptions[] = [
  {
    resource: Course,
    options: courseResourceOptions,
  },
  {
    resource: Episode,
    options: episodeResourceOptions,
  },
  {
    resource: Category,
    options: categoryResourceOptions,
  },
];
```

- Já é possível testar a aplicação cadastrando alguns episódios pelo painel administrativo. Não se preocupe com questões como o upload dos vídeos no momento, as implementares nas próximas aulas. Por enquanto tente cadastrar episódios com alguns valores fictícios.

## 8 - Upload de Arquivos no AdminJS

Nesta aula iremos adicionar a funcionalidade de upload. Como já temos o @adminjs/upload instalado em nosso projeto passaremos diretamente para sua configuração e inclusão.

- No arquivo episode.ts da pasta resources adicionaremos as configuração da feature de upload. Adicione no arquivo o código mostrado abaixo:
  Vale destacar que:
  - uploadFileFeature é uma função do @adminjs/upload que gera para nós uma configuração válida para o funcionamento do módulo de upload. Ela recebe um objeto de configuração como parâmetro.
  - A propriedade “provider” define onde iremos armazenar nossos arquivos. Armazenaremos localmente e a única opção que precisamos especificar é o bucket, ou a pasta onde serão salvos os arquivos.
  - A propriedade “properties” permite configurar as propriedades geradas pelo próprio AdminJS a partir de dados do upload, como caminho relativo do arquivo, caminho absoluto, nome, tamanho, etc. Usaremos apenas “key” para o caminho relativo e “file” para a propriedade virtual que será usada no frontend para armazenar o arquivo.
  - A propriedade “uploadPath” contém a função que gera o caminho do arquivo dentro da pasta de uploads. Podemos personalizar esse caminho para termos uma estrutura de uploads mais organizada.
- Também é necessário verificar o campo de upload nas propriedades dos formulário dentro do objeto de opções do recurso. Onde colocaríamos a propriedade “videoUrl” agora deve ser a propriedade “uploadVideo” como especificamos nas opções da feature. Essa propriedade virtual permitirá utilizar o componente DropZone para upload de arquivos.
  - Obs.: Este nome da propriedade virtual é arbitrário, poderia ser qualquer nome que preferíssemos desde que fosse referenciado corretamente.

```typescript
// src/adminjs/resources/episode.ts

import path from "path";
import uploadFileFeature from "@adminjs/upload";
import { FeatureType, ResourceOptions } from "adminjs";

export const episodeResourceOptions: ResourceOptions = {
  navigation: "Catálogo",
  editProperties: [
    "name",
    "synopsis",
    "courseId",
    "order",
    "uploadVideo",
    "secondsLong",
  ],
  filterProperties: [
    "name",
    "synopsis",
    "courseId",
    "secondsLong",
    "createdAt",
    "updatedAt",
  ],
  listProperties: ["id", "name", "courseId", "order", "secondsLong"],
  showProperties: [
    "id",
    "name",
    "synopsis",
    "courseId",
    "order",
    "videoUrl",
    "secondsLong",
    "createdAt",
    "updatedAt",
  ],
};

export const episodeResourceFeatures: FeatureType[] = [
  uploadFileFeature({
    provider: {
      local: {
        bucket: path.join(__dirname, "../../../uploads"),
      },
    },
    properties: {
      key: "videoUrl",
      file: "uploadVideo",
    },
    uploadPath: (record, filename) =>
      `videos/course-${record.get("courseId")}/${filename}`,
  }),
];
```

- Após adicionar as opções da feature de upload basta inclui-las em index.ts junto com as outras opções do recurso:

```typescript
// src/adminjs/resources/index.ts

// ...
import { episodeResourceFeatures, episodeResourceOptions } from './episode'

// ...
	},
	{
    resource: Episode,
    options: episodeResourceOptions,
    features: episodeResourceFeatures
  },
	{
//...
```

- Com a feature incluída no AdminJS já será possível fazer os uploads. Crie uma pasta “uploads” na raiz do projeto ao lado de “public” e “src” para armazenarmos os arquivos da nossa aplicação e tente criar um episódio no painel administrativo.
- Por último, se estiver usando Git em seu projeto, não se esqueça de adicionar a pasta uplolads/videos em seu arquivo .gitignore:

````text
node_modules
uploads/videos```
````

- Além disso, é possível garantir que a pasta upload seja mantida no repositório mesmo estando vazia ao criar um arquivo .gitkeep dentro dela.

## 9 - Upload das Capas dos Cursos

- Primeiro, adicione no arquivo de resource course.ts um array de features que irá conter a feature de upload e então exporte-a:
  - Obs.: As capas dos cursos poderão ser públicas para facilitar o acesso sem nenhum problema.

```typescript
// src/adminjs/resources/course.ts

import { FeatureType, ResourceOptions } from "adminjs";
import uploadFileFeature from "@adminjs/upload";
import path from "path";

// ...

export const courseResourceFeatures: FeatureType[] = [
  uploadFileFeature({
    provider: {
      local: {
        bucket: path.join(__dirname, "../../../public"),
      },
    },
    properties: {
      key: "thumbnailUrl",
      file: "uploadThumbnail",
    },
    uploadPath: (record, filename) =>
      `thumbnails/course-${record.get("id")}/${filename}`,
  }),
];
```

- Agora vamos nos certificar de que as propriedades editáveis incluem a propriedade que armazenará o arquivo, ou seja, “uploadThumbnail”:

```typescript
// src/adminjs/resources/course.ts

import { FeatureType, ResourceOptions } from "adminjs";
import uploadFileFeature from "@adminjs/upload";
import path from "path";

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

// ...
```

- E então adicionamos as features exportadas ao arquivo resources.ts:

````typescript
// src/adminjs/resources/index.ts

// ...

import { courseResourceFeatures, courseResourceOptions } from './course'
import { episodeResourceFeatures, episodeResourceOptions } from './episode'

const adminJsResources: ResourceWithOptions[] = [
  {
    resource: Course,
    options: courseResourceOptions,
    features: courseResourceFeatures
  },
  {
    resource: Episode,
    options: episodeResourceOptions,
    features: episodeResourceFeatures
  },```
````

- Por fim, se estiver utilizando Git em seu projeto, adicione a pasta uploads/thumbnails ao seu arquivo .gitignore:

```text
node_modules
uploads/videos
public/thumbnails
```

- Tente atualizar algum curso ou criar um novo e subir sua imagem de capa para testar o funcionamento da feature.

## 10 - Tabela de Usuários

- Comece criando a migration para a tabela de usuários:

````typescript
npx sequelize-cli migration:generate --name create-users-table```
````

- Então, adicione o conteúdo da migration:

```typescript
// src/database/migrations/XXXXXXXXXXXXXX-create-users-table

"use strict";

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable("users", {
      id: {
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
        type: Sequelize.DataTypes.INTEGER,
      },
      first_name: {
        allowNull: false,
        type: Sequelize.DataTypes.STRING,
      },
      last_name: {
        allowNull: false,
        type: Sequelize.DataTypes.STRING,
      },
      phone: {
        allowNull: false,
        type: Sequelize.DataTypes.STRING,
      },
      birth: {
        allowNull: false,
        type: Sequelize.DataTypes.DATE,
      },
      email: {
        allowNull: false,
        unique: true,
        type: Sequelize.DataTypes.STRING,
      },
      password: {
        allowNull: false,
        type: Sequelize.DataTypes.STRING,
      },
      role: {
        allowNull: false,
        type: Sequelize.DataTypes.STRING,
      },
      created_at: {
        allowNull: false,
        type: Sequelize.DATE,
      },
      updated_at: {
        allowNull: false,
        type: Sequelize.DATE,
      },
    });
  },

  async down(queryInterface, Sequelize) {
    await queryInterface.dropTable("users");
  },
};
```

- E execute a migration, criando a tabela no banco de dados:

```typsescript
npx sequelize-cli db:migrate
```

- Agora, crie o model User.ts na pasta models e adicione o seu conteúdo:

```typescript
// src/models/User.ts

import { sequelize } from "../database";
import { DataTypes, Model, Optional } from "sequelize";

export interface User {
  id: number;
  firstName: string;
  lastName: string;
  phone: string;
  birth: Date;
  email: string;
  password: string;
  role: "admin" | "user";
}

export interface UserCreationAttributes extends Optional<User, "id"> {}

export interface UserInstance
  extends Model<User, UserCreationAttributes>,
    User {}

export const User = sequelize.define<UserInstance, User>("User", {
  id: {
    allowNull: false,
    autoIncrement: true,
    primaryKey: true,
    type: DataTypes.INTEGER,
  },
  firstName: {
    allowNull: false,
    type: DataTypes.STRING,
  },
  lastName: {
    allowNull: false,
    type: DataTypes.STRING,
  },
  phone: {
    allowNull: false,
    type: DataTypes.STRING,
  },
  birth: {
    allowNull: false,
    type: DataTypes.DATE,
  },
  email: {
    allowNull: false,
    unique: true,
    type: DataTypes.STRING,
    validate: {
      isEmail: true,
    },
  },
  password: {
    allowNull: false,
    type: DataTypes.STRING,
  },
  role: {
    allowNull: false,
    type: DataTypes.STRING,
    validate: {
      isIn: [["admin", "user"]],
    },
  },
});
```

- Adicione o model ao arquivo index.ts da pasta models, pois futuramente o utilizaremos para registrar as associações da tabela users:

```typescript
// src/models/index.ts

import { User } from "./User";

// ...

export { Category, Course, Episode, User };
```

- Com o model criado podemos adicionar o resource ao painel do AdminJS. Crie o arquivo user.ts em resources e adicione o seguinte código:
  - Obs.: Repare que nesse resource estamos utilizando algumas opções extras. Através da propriedade “properties” podemos personalizar o comportamento de cada campo individualmente, alterando manualmente seu comportamento no painel.

```typescript
// src/adminjs/resources/user.ts

import { ResourceOptions } from "adminjs";

const userResourceOptions: ResourceOptions = {
  navigation: "Administração",
  properties: {
    birth: {
      type: "date",
    },
    password: {
      type: "password",
    },
    role: {
      availableValues: [
        { value: "admin", label: "Administrador" },
        { value: "user", label: "Usuário Padrão" },
      ],
    },
  },
  editProperties: [
    "firstName",
    "lastName",
    "phone",
    "birth",
    "email",
    "password",
    "role",
  ],
  filterProperties: [
    "firstName",
    "lastName",
    "phone",
    "birth",
    "email",
    "role",
    "createdAt",
    "updatedAt",
  ],
  listProperties: ["id", "firstName", "email", "role"],
  showProperties: [
    "id",
    "firstName",
    "lastName",
    "phone",
    "birth",
    "email",
    "role",
    "createdAt",
    "updatedAt",
  ],
};

export { userResourceOptions };
```

- Por fim, inclua o resource no arquivo resources.ts:

```typescript
// src/adminjs/resources/index.ts

// ...

import { userResourceOptions } from "./user";

// ..

  {
    resource: Category,
    options: categoryResourceOptions
  },
  {
    resource: User,
    options: userResourceOptions
  }
]
```

- Já podemos testar o cadastro de usuários e ver que está funcionando. Crie um usuário com a role ’admin’ para utilizarmos na aula sobre autenticação.
- No entanto, se olharmos o usuário cadastrado no banco de dados veremos que sua senha está exposta, o que não é uma boa prática de segurança. Vamos cuidar para que ela seja encriptada antes de o usuário ser salvo no banco de dados. Para isso precisamos instalar a biblioteca “bcrypt”. Instale-a com o comando abaixo:

```typescript
npm install bcrypt@~5.0.1
```

- Além disso, o bcrypt não possui tipagens, portanto precisamos instalar seus tipos separadamente através do @types:

```typescript
npm install --save-dev @types/bcrypt@~5.0.0
```

- Com o bcrypt instalado podemos incluir um hook em nosso model user para executar uma ação sempre antes de um novo registro ser salvo. Adicione o hook beforeSave nas opções do segundo parâmetro do método define:

```typescript
// src/models/user.ts

import bcrypt from 'bcrypt'

// ...

		role: {
	    allowNull: false,
	    type: DataTypes.STRING
	  }
	}, {
  hooks: {
    beforeSave: async (user) => {
      if (user.isNewRecord || user.changed('password')) {
        user.password = await bcrypt.hash(user.password.toString(), 10);
      }
    }
  }
})

export { User }
```

- Agora, ao tentar novamente cadastrar um usuário veremos no banco de dados que sua senha não está mais exposta, salvamos apenas o hash. Com isso concluímos o model user.
- Nós também podemos criar um seeder responsável por criar um usuário administrador, caso ele não exista:

```typescript
npx sequelize-cli seed:generate --name create-admin-user
```

- Dentro do seeder poderemos adicionar o seguinte código:
  Obs.: Repare que precisamos utilizar o bcrypt manualmente dentro do seeder, pois não estamos utilizando um model, estamos realizando uma query um pouco mais “manual”.
  Obs².: Repare também que podemos especificar o bulkDelete do método down para funcionar especificamente no usuário criado pelo método up.

```typescript
// src/database/seeders/XXXXXXXXXXXXXX-create-admin-user.js

const bcrypt = require("bcrypt");

("use strict");

module.exports = {
  async up(queryInterface, Sequelize) {
    const hashedPassword = await bcrypt.hash("123456", 10);

    await queryInterface.bulkInsert("users", [
      {
        first_name: "Admin",
        last_name: "User",
        phone: "555-5555",
        birth: "1990-01-01",
        email: "admin@email.com",
        password: hashedPassword,
        role: "admin",
        created_at: new Date(),
        updated_at: new Date(),
      },
    ]);
  },

  async down(queryInterface, Sequelize) {
    await queryInterface.bulkDelete("users", null, {
      where: { email: "admin@email.com" },
    });
  },
};
```

- Agora podemos executar o seeder:
  Obs.: Repare que podemos executar os seeders individualmente se quisermos através da flag --seed. Lembre-se de utilizar a timestamp correta do nome do seu seeder.

```typescript
npx sequelize-cli db:seed --seed src/database/seeders/XXXXXXXXXXXXXX-create-admin-user.js
```

## 11 - Autenticação e Tela de Login

- Para adicionarmos autenticação ao painel do AdminJs só precisamos trocar o método que constrói as rotas. Substituímos o “buildRouter” para “buildAuthenticatedRouter” no arquivo de configuração do AdminJs:
  - Obs.: Repare que o método “buildAuthenticatedRouter” recebe um argumento a mais, que é um objeto contendo a função de autenticação e uma chave para garantir a segurança do cookie de autenticação. Também é possível passar um nome opcionalmente.
    Usaremos variáveis de ambiente no futuro para não deixar nossa senha do cookie exposta.

```typescript
// src/adminjs/index.ts

// ...

export const adminJsRouter = AdminJsExpress.buildAuthenticatedRouter(adminJs, {
  authenticate: () => {},
  cookiePassword: "",
});
```

- Dentro do método de autenticação adicione uma função recebe como parâmetros o email e a senha digitados na tela de login que será incluída automaticamente. Essa função deve procurar por um usuário e verificar se suas credenciais estão corretas.
- Também vamos incluir uma string qualquer (que será atualizada posteriormente) para servir de senha para o cookie:

```typescript
// src/adminjs/index.ts

import { User } from "../models";
import bcrypt from "bcrypt";

// ...

export const adminJsRouter = AdminJsExpress.buildAuthenticatedRouter(
  adminJs,
  {
    authenticate: async (email, password) => {
      const user = await User.findOne({ where: { email } });

      if (user && user.role === "admin") {
        const matched = await bcrypt.compare(password, user.password);

        if (matched) {
          return user;
        }
      }

      return false;
    },
    cookiePassword: "senha-do-cookie",
  },
  null,
  {
    resave: false,
    saveUninitialized: false,
  }
);
```

- E isso é tudo que precisamos para incluir autenticação ao nosso painel. Navegue até o painel e veja que agora é necessário inserir seu email e senha para entrar. Certifique-se de que você possui um usuário devidamente cadastrado.

## 12 - Adicionando Traduções

- Para incluir o recurso de localização vamos criar um arquivo na pasta adminjs chamado locale.ts. Nele vamos declarar todas as traduções para o nosso painel, tanto de textos do próprio AdminJs quanto das nossas tabelas e colunas:
  - Obs.: Muitos desses textos são apenas os textos padrões do AdminJS, porém traduzidos para o português. Não entraremos tanto em detalhes nessa parte para poupar tempo, mas você poderia utilizar os textos que quisesse.

```typescript
// src/adminjs/locale.ts

import { Locale } from "adminjs";

export const locale: Locale = {
  language: "pt-BR",
  translations: {
    actions: {
      new: "Criar novo(a)",
      edit: "Editar",
      show: "Exibir",
      delete: "Excluir",
      bulkDelete: "Excluir tudo",
      list: "Listar",
    },
    buttons: {
      save: "Salvar",
      addNewItem: "Adicionar Novo Item",
      filter: "Filtrar",
      applyChanges: "Aplicar mudanças",
      resetFilter: "Redefinir",
      confirmRemovalMany: "Confirma a remoção de {{count}} registro",
      confirmRemovalMany_plural: "Confirma a remoção de {{count}} registros",
      logout: "Sair",
      login: "Entrar",
      seeTheDocumentation: "Ver: <1>a documentação</1>",
      createFirstRecord: "Criar Primeiro Registro",
    },
    labels: {
      navigation: "Navegação",
      pages: "Páginas",
      selectedRecords: "Selecionado ({{selected}})",
      filters: "Filtros",
      adminVersion: "Admin: {{version}}",
      appVersion: "App: {{version}}",
      loginWelcome: "Bem-vindo",
      courses: "Cursos",
      episodes: "Episódios",
      categories: "Categorias",
      users: "Usuários",
    },
    properties: {
      firstName: "Primeiro Nome",
      lastName: "Sobrenome",
      phone: "Telefone",
      birth: "Data de Nascimento",
      email: "E-mail",
      password: "Senha",
      role: "Perfil",
      name: "Nome",
      synopsis: "Sinopse",
      featured: "Em Destaque",
      order: "Ordem",
      videoUrl: "URL do Vídeo",
      secondsLong: "Segundos de Duração",
      courseId: "Curso",
      uploadVideo: "Enviar um vídeo",
      thumbnailUrl: "URL da Capa",
      uploadThumbnail: "Upload da Capa",
      categoryId: "Categoria",
      position: "Posição na Tela",
      createdAt: "Criado em",
      updatedAt: "Atualizado em",
      from: "De",
      to: "Até",
    },
    messages: {
      successfullyBulkDeleted: "{{count}} registro removido com sucesso",
      successfullyBulkDeleted_plural:
        "{{count}} registros removidos com sucesso",
      successfullyDeleted: "Registro excluído com sucesso",
      successfullyUpdated: "Registro atualizado com sucesso",
      thereWereValidationErrors:
        "Existem erros de validação - confira-os abaixo",
      forbiddenError:
        "Você não pode executar a ação {{actionName}} em {{resourceId}}",
      anyForbiddenError: "Você não pode executar a ação solicitada",
      successfullyCreated: "Novo registro criado com sucesso",
      bulkDeleteError:
        "Houve um erro ao excluir os registros. Verifique os logs para mais informações",
      errorFetchingRecords:
        "Houve um erro ao obter os registros. Verifique os logs para mais informações",
      errorFetchingRecord:
        "Houve um erro ao obter o registro, Verifique os logs para mais informações",
      noRecordsSelected: "Você não selecionou nenhum registro",
      theseRecordsWillBeRemoved: "O registro a seguir será excluído",
      theseRecordsWillBeRemoved_plural: "Os registros a seguir serão excluídos",
      pickSomeFirstToRemove:
        "Para remover registros você precisa selecioná-los primeiro",
      error404Resource:
        "Recurso indentificado pelo id: {{resourceId}} não pôde ser encontrado",
      error404Action:
        "Recurso indentificado pelo id: {{resourceId}} não possui uma ação com nome: {{actionName}} ou você não está autorizado a usá-la!",
      error404Record:
        "Recurso indentificado pelo id: {{resourceId}} não possui um registro com id: {{recordId}} ou você não está autorizado a acessá-lo!",
      seeConsoleForMore:
        "Veja o console de desenvolvimento para mais detalhes...",
      noActionComponent:
        "Você precisa implementar componente de ação para a sua Ação",
      noRecordsInResource: "Não existem registros neste recurso",
      noRecords: "Nenhum registro",
      confirmDelete:
        "Você tem certeza que deseja remover este item? Essa ação é irreversível",
      welcomeOnBoard_title: "Bem-vindo à bordo!",
      welcomeOnBoard_subtitle:
        "Agora você é um de nós! Preparamos algumas dicas para você começar:",
      loginWelcome:
        "Ao AdminJS - o melhor framework admin para aplicações Node.js, baseado em React.",
      addingResources_title: "Adicionando Recursos",
      addingResources_subtitle: "Como adicionar novos recursos à barra lateral",
      customizeResources_title: "Personalizar Recursos",
      customizeResources_subtitle:
        "Definindo comportamento, adicionando propriedades e mais...",
      customizeActions_title: "Personalizar Ações",
      customizeActions_subtitle: "Modificar ações existentes e adicionar novas",
      writeOwnComponents_title: "Escrever Componentes",
      writeOwnComponents_subtitle: "Como modificar o visual do AdminJS",
      customDashboard_title: "Dashboard Personalizado",
      customDashboard_subtitle:
        "Como modificar esta página e adicionar novas páginas à barra lateral",
      roleBasedAccess_title: "Controle de Acesso Baseado em Perfil",
      roleBasedAccess_subtitle:
        "Criar perfis de usuário e permissões no AdminJS",
      community_title: "Junte-se à comunidade slack",
      community_subtitle:
        "Fale com os criadores do AdminJS e outros usuários do AdminJS",
      foundBug_title: "Encontrou um Bug? Precisa de alguma melhoria?",
      foundBug_subtitle: "Levante um issue em nosso repositório no GitHub",
      needMoreSolutions_title: "Precisa de mais soluções avançadas?",
      needMoreSolutions_subtitle:
        "Estamos aqui para te entregar um belo desenho de UX/UI e software feito sob medida baseado (não apenas) no AdminJS",
      invalidCredentials: "Nome de usuário e/ou senha incorretos",
    },
  },
};
```

- Uma vez que tenhamos o arquivo locale.ts basta incluir ele nas opções do AdminJs através da propriedade locale:

```typescript
// src/adminjs/index.ts

// ...
import { locale } from './locale'

AdminJs.registerAdapter(AdminJsSequelize)

const adminJs = new AdminJs({
  databases: [database],
  resources: adminJsResources,
  rootPath: '/admin',
  locale: locale,
  branding: {

// ...
```

- E isso é tudo! Agora basta acessar o painel e conferir as traduções.

## 13 - Personalizando o Dashboard

- Para começar a personalizar a tela inicial do AdminJs precisamos atualizar o arquivo de configuração para incluir a propriedade “dashboard”. Essa propriedade deve receber o componente React que iremos criar para ser exibido, e ele precisa ser incluído através do método bundle do próprio AdminJs como no código abaixo:

```typescript
// src/adminjs/index.ts

// ...

export const adminJs = new AdminJs({
  databases: [database],
  resources: adminJsResources,
  rootPath: '/admin',
  dashboard: {
    component: AdminJs.bundle('./components/Dashboard')
  },
  locale: locale,

// ...
```

- Agora crie uma pasta components dentro de adminjs e nela crie o arquivo Dashboard.tsx, que será o nosso componente. Dentro dele adicione o seguinte código:

```typescript
// src/adminjs/components/Dashboard.tsx

import React from "react";
import { H1 } from "@adminjs/design-system";

export default function Dashboard() {
  return (
    <section style={{ padding: "1.5rem" }}>
      <H1>Seja bem-vindo!</H1>
    </section>
  );
}
```

- Antes de avançar também precisamos atualizar nosso arquivo de configuração do compilador do typescript para incluir suporte ao JSX do React:

```typescript
// tsconfig.json

{
  "compilerOptions": {
    "target": "es2016",
    "module": "commonjs",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "jsx": "react"
  }
}
```

- Agora já podemos testar e ver que nossa página inicial não é mais a padrão do AdminJs, mas o componente que criamos contendo apenas um título.
- Vamos então incluir alguns dados dinâmicos ao dashboard para deixá-lo mais interessante. Para isso, adicione a propriedade handler às opções de configuração do dashboard com o seguinte conteúdo:
  - Obs.: Veja que a propriedade handler é do tipo PageHandler do próprio AdminJS. Ela recebe como parâmetros a requisição, a resposta e um contexto que contém informações sobre a instância do AdminJS, o usuário atual e alguns helpers para as views.
  - Nesse handler estamos obtendo a quantidade de registros de algumas tabelas e então enviando-os na resposta em um objeto.

```typescript
// src/adminjs/index.ts

// ...
import { Category, Course, Episode, User } from '../models'
// ...

export const adminJs = new AdminJs({
  databases: [database],
  resources: adminJsResources,
  rootPath: '/admin',
  dashboard: {
    component: AdminJs.bundle('./components/Dashboard'),
		handler: async (req, res, context) => {
      const courses = await Course.count()
      const episodes = await Episode.count()
      const category = await Category.count()
      const standardUsers = await User.count({ where: { role: 'user' } })

      res.json({
        'Cursos': courses,
        'Episódios': episodes,
        'Categorias': category,
        'Usuários': standardUsers
      })
    },
  },
  locale: locale,

// ...
```

- No componente react, inclua a chamada ao handler criado utilizando a classe ApiClient do AdminJS. Por baixo dos panos o AdminJS utilizará o axios para fazer as requisições, portanto as respostas virão no formato do axios. Teste a requisição da seguinte forma:

```typescript
// src/adminjs/components/Dashboard.tsx

import React, { useEffect } from "react";
import { H1 } from "@adminjs/design-system";
import { ApiClient } from "adminjs";

export default function Dashboard() {
  const api = new ApiClient();

  useEffect(() => {
    fetchDashboardData();
  }, []);

  async function fetchDashboardData() {
    const res = await api.getDashboard();
    console.log(res.data);
  }

  return (
    <section style={{ padding: "1.5rem" }}>
      <H1>Seja bem-vindo!</H1>
    </section>
  );
}
```

- Ao olharmos o console do navegador vemos a resposta da função handler contendo os nomes dos recursos como chave e suas quantidades como valor em um objeto.
- Vamos agora fazer um pouco mais. Adicione um state para armazenar os recursos e então exiba-os em uma tabela:
  - Obs.: No corpo da tabela estamos transformando o objeto que recebemos em um array de pares [chave, valor] para criarmos as linhas automaticamente através de um loop.

```typescript
// src/adminjs/components/Dashboard.tsx

import React, { useEffect, useState } from "react";
import {
  H1,
  H2,
  Table,
  TableHead,
  TableBody,
  TableRow,
  TableCell,
} from "@adminjs/design-system";
import { ApiClient } from "adminjs";

export default function Dashboard() {
  const [resources, setResources] = useState<{ [key: string]: number }>();
  const api = new ApiClient();

  useEffect(() => {
    fetchDashboardData();
  }, []);

  async function fetchDashboardData() {
    const res = await api.getDashboard();
    console.log(res.data);

    setResources(res.data);
  }

  return (
    <section style={{ padding: "1.5rem" }}>
      <H1>Seja bem-vindo!</H1>

      <section style={{ backgroundColor: "#FFF", padding: "1.5rem" }}>
        <H2>Resumo</H2>
        <Table>
          <TableHead>
            <TableRow style={{ backgroundColor: "#FF0043" }}>
              <TableCell style={{ color: "#FFF" }}>Recurso</TableCell>
              <TableCell style={{ color: "#FFF" }}>Registros</TableCell>
            </TableRow>
          </TableHead>
          <TableBody>
            {resources ? (
              Object.entries(resources).map(([resource, count]) => (
                <TableRow key={resource}>
                  <TableCell>{resource}</TableCell>
                  <TableCell>{count}</TableCell>
                </TableRow>
              ))
            ) : (
              <></>
            )}
          </TableBody>
        </Table>
      </section>
    </section>
  );
}
```

- Agora teste e veja os recursos e suas quantidades de registros aparecem em uma tabela.
- Para concluir, vamos utilizar um hook do próprio AdminJS para obter o usuário logado atual, o hook useCurrentAdmin. Com ele poderemos mostrar na tela de boas-vindas o primeiro nome do usuário

```typescript
// src/adminjs/components/Dashboard.tsx

// ...
import { ApiClient, useCurrentAdmin } from 'adminjs'

export default function Dashboard() {
  const [currentAdmin] = useCurrentAdmin()
  const [resources, setResources] = useState<{ [key: string]: number }>()

// ...

return (
    <section style={{ padding: '1.5rem' }}>
      <H1>Seja bem-vindo, {currentAdmin?.firstName}!</H1>

      <section style={{ backgroundColor: '#FFF', padding: '1.5rem' }}>
        <H2>Resumo</H2>

// ...
```

## 14 - Refatorando as Configurações do AdminJS

- O AdminJS já está pronto para usarmos em nossa aplicação, mas o arquivo de configuração da forma que está pode ser melhorado. Vamos separar algumas configurações específicas movendo-as para a pasta adminjs.
- Vamos começar pela propriedade dashboard do arquivo de configuração. Mova o seu conteúdo para um arquivo dashboard.ts na pasta adminjs e exporte-o:

```typescript
// src/adminjs/dashboard.ts

import AdminJs, { PageHandler } from "adminjs";
import { Category, Course, Episode, User } from "../models";

export const dashboardOptions: {
  handler?: PageHandler;
  component?: string;
} = {
  component: AdminJs.bundle("../adminjs/components/Dashboard"),
  handler: async (req, res, context) => {
    const courses = await Course.count();
    const episodes = await Episode.count();
    const category = await Category.count();
    const standardUsers = await User.count({ where: { role: "user" } });

    res.json({
      Cursos: courses,
      Episódios: episodes,
      Categorias: category,
      "Usuários Padrão": standardUsers,
    });
  },
};
```

- Vamos fazer o mesmo com a propriedade branding. Mova o seu conteúdo para um arquivo branding.ts na pasta adminjs e exporte-o:

```typescript
// src/adminjs/branding.ts

import { BrandingOptions } from "adminjs";

export const brandingOptions: BrandingOptions = {
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
};
```

- Por fim, faça o mesmo com as opções de autenticação utilizadas na construção do router. Crie um arquivo authentication.ts na pasta adminjs, adicione o objeto com as opções e exporte-o:

```typescript
// src/adminjs/authentication.ts

import { AuthenticationOptions } from "@adminjs/express";
import { User } from "../models";
import bcrypt from "bcrypt";

export const authtenticationOptions: AuthenticationOptions = {
  authenticate: async (email, password) => {
    const user = await User.findOne({ where: { email } });

    if (user && user.role === "admin") {
      const matched = await bcrypt.compare(password, user.password);

      if (matched) {
        return user;
      }
    }

    return false;
  },
  cookiePassword: "senha-do-cookie",
};
```

- Agora, basta incluir os objetos exportados no arquivo de configuração:

```typescript
// src/adminjs/index.ts

// ...
import { dashboardOptions } from "./dashboard";
import { brandingOptions } from "./branding";
import { authtenticationOptions } from "./authentication";

AdminJs.registerAdapter(AdminJsSequelize);

export const adminJs = new AdminJs({
  databases: [database],
  resources: adminJsResources,
  rootPath: "/admin",
  dashboard: dashboardOptions,
  locale: locale,
  branding: brandingOptions,
});

export const adminJsRouter = AdminJsExpress.buildAuthenticatedRouter(
  adminJs,
  authtenticationOptions,
  null,
  { resave: false, saveUninitialized: false }
);
```

- Teste a aplicação e veja que seu funcionamento continua inalterado.
