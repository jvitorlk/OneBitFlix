# 1 - Iniciando o Projeto

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
