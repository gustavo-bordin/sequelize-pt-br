# Iniciando

Neste tutorial você aprenderá a fazer uma configuração simples do Sequelize.

## Instalando

O Sequelize está disponível via npm [npm](https://www.npmjs.com/package/sequelize) (ou [yarn](https://yarnpkg.com/package/sequelize)).

```sh
npm install --save sequelize
```
Você tabém precisará fazer manualmente a instalação do driver para o banco de dados de sua escolha:

```sh
# Uma das opções abaixo:
$ npm install --save pg pg-hstore # Postgres
$ npm install --save mysql2
$ npm install --save mariadb
$ npm install --save sqlite3
$ npm install --save tedious # Microsoft SQL Server
```

## Conectando-se a uma base de dados

Para conexão a uma base de dados, você precisa instanciar o Sequelize. Isso pode ser feito passando os parâmetros de conexão separadamente para o constructor do Sequelize ou passando uma URI única de conexão:

```js
const { Sequelize } = require('sequelize');

// Opção 1: Passando uma URI de conexão
const sequelize = new Sequelize('sqlite::memory:') // Examplo para o sqlite
const sequelize = new Sequelize('postgres://user:pass@example.com:5432/dbname') // Exemplo para o postgres

// Opção 2: Passando os parâmetros separadamente (sqlite)
const sequelize = new Sequelize({
  dialect: 'sqlite',
  storage: 'path/to/database.sqlite'
});

// Opção 2: Passando os parãmetros separadamente (other dialetos)
const sequelize = new Sequelize('database', 'username', 'password', {
  host: 'localhost',
  dialect: /* um destes: 'mysql' | 'mariadb' | 'postgres' | 'mssql' */
});
```

O constructor do Sequelize aceita várias opções. Elas estão documentadas na [API Reference](../class/lib/sequelize.js~Sequelize.html#instance-constructor-constructor).

### Testando a conexão

Você pode usar a função `.authenticate()` para testar se a conexão está funcionando corretamente:

```js
try {
  await sequelize.authenticate();
  console.log('Connection has been established successfully.');
} catch (error) {
  console.error('Unable to connect to the database:', error);
}
```

### Fechando a conexão

O Sequelize mantém a conexão aberta por padrão e a usa para todas as consultas. Se você precisar fechar a conexão, chame a função `sequelize.close()` (que é assíncrona e retorna uma Promise).

## Convenções de terminologia

Note que, nos exemplos acima, `Sequelize` refere-se à biblioteca em si, enquanto `sequelize` se refere a uma instância do Sequelize, que representa uma conexão com uma base de dados. Essa é a convenção de terminologia recomendada e é a que será usada em toda a documentação.

## Dica para leitura da documentação

Encorajamos você a executar os códigos de exemplo de forma local enquanto lê a documentação do Sequelize. Isso ajudará a agilizar seu aprendizado. A maneira mais fácil de se fazer isso é usando o dialeto do SQLite:

```js
const { Sequelize, Op, Model, DataTypes } = require("sequelize");
const sequelize = new Sequelize("sqlite::memory:");

// Coloque seu código aqui! Ele funciona!
```

Para experimentar outros dialetos que são mais complexos de se configurar localmeente, você pode usar o repositório do GitHub para o [Sequelize SSCCE](https://github.com/papb/sequelize-sscce), que permite que você execute códigos em todos os dialetos suportados diretamente no GitHub, de forma gratuita e sem nenhuma necessidade de configuração!

## Novas bases de dados versus bases de dados existentes

Se você está começando um projeto do zero e sua base de dados ainda não foi criada, o Sequelize pode ser usado desde o princípio para automatizar a criação de cada tabela no seu banco.

E ainda, se você quer usar o Sequelize para conexão a uma base de dados já populada com tabelas e dados, isso também é possível. O Sequelize vai te ajudar em ambos os casos.

## Logging

Por padrão, o Sequelize vai fazer o logging para o console de toda operação SQL performada. A opção `options.logging` pode ser usada para customizar esse comportamento, definindo qual função deve ser executada toda vez que o Sequelize precisar fazer o log de algo. O valor padrão é `console.log` e quando usado, somente o primeiro valor de log da chamada à função de log é exibido. Por exemplo: para o log de consultas, o primeiro valor exibido é a raw query e o segundo (que é ocultado por padrão) é o objeto Sequelize.

Alguns valores úteis para `options.logging`:

```js
const sequelize = new Sequelize('sqlite::memory:', {
  // Escolha uma das opções de logging
  logging: console.log,                  // Padrão, exibe o primeiro parâmetro da chamada à função de logging
  logging: (...msg) => console.log(msg), // Exibe todos os parâmetros da chamada à função de logging
  logging: false,                        // Desativa o logging
  logging: msg => logger.debug(msg),     // Utiliza um logger customizado  (ex. Winston ou Bunyan), exibindo o primeiro parâmetro
  logging: logger.debug.bind(logger)     // Modo alternativo para uso de um logger customizado, exibe todos as mensagens
});
```

## Promises e async/await

A maioria dos métodos fornecidos pelo Sequelize é assíncrona e, por causa disso, retorna Promises. Elas são todas [Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) , então você pode usar a Promise API (por exemplo, usando `then`, `catch`, `finally`) sem necessidade de qualquer outra configuração.

E é claro que, ao usar `async` e `await`, isso também funciona da maneira esperada.
