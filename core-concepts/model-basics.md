# Noções básicas de Model

Nesse tutorial você vai aprender o que são models no Sequelize e como usar eles.

## Conceito

Models são a essência do Sequelize. Um model é uma abstração que representa uma tabela em seu banco de dados. No Sequelize, é uma classe que extende [Model](../class/lib/model.js~Model.html).

O model fala ao Sequelize várias coisas sobre a entidade que ele representa, tal como o nome da tabela no banco de dados e quais colunas ele tem (e seus tipos de dados).

Um model no Sequelize tem um nome. Esse nome não tem de ser o mesmo da tabela que ele representa no banco de dados. Geralmente, models tem nomes no singular (tal como `User`) enquanto tabelas tem nomes no plural (tal como `Users`), embora isso seja completamente configurável.

## Definição de uma Model

Models podem ser definidas em duas maneiras equivalentes em Sequelize:

* Chamando [`sequelize.define(modelName, attributes, options)`](../class/lib/sequelize.js~Sequelize.html#instance-method-define)
* Extendendo [Model](../class/lib/model.js~Model.html) and calling [`init(attributes, options)`](../class/lib/model.js~Model.html#static-method-init)

Depois de um model estar definido, ele está disponível dentro de `sequelize.models` pelo seu nome.

Para aprender com um exemplo, vamos considerar que queremos criar um model para representar users, no qual tem um `firstName` e um `lastName`. Queremos que nosso model seja chamado `User`, e a tabela que ele representa é chamada `Users` no banco de dados.

Ambas maneiras para definir esse model são mostradas abaixo. Depois de ser definido, podemos acessar nosso model com `sequelize.models.User`.

### Usando [`sequelize.define`](../class/lib/sequelize.js~Sequelize.html#instance-method-define):

```js
const { Sequelize, DataTypes } = require('sequelize');
const sequelize = new Sequelize('sqlite::memory:');

const User = sequelize.define('User', {
  // Atributos do model são definidos aqui
  firstName: {
    type: DataTypes.STRING,
    allowNull: false
  },
  lastName: {
    type: DataTypes.STRING
    // allowNull true por padrão
  }
}, {
  // Outras opções do model vão aqui
});

// `sequelize.define` também returna o model
console.log(User === sequelize.models.User); // true
```

### Extendendo [Model](../class/lib/model.js~Model.html)

```js
const { Sequelize, DataTypes, Model } = require('sequelize');
const sequelize = new Sequelize('sqlite::memory');

class User extends Model {}

User.init({
  // Atributos do model são definidos aqui
  firstName: {
    type: DataTypes.STRING,
    allowNull: false
  },
  lastName: {
    type: DataTypes.STRING
    // allowNull true por padrão
  }
}, {
  // Outras opções do model vão aqui
  sequelize, // Precisamos passar a instância da conexão
  modelName: 'User' // Precisamos escolher o nome do model
});

// O model definido é a classe em si
console.log(User === sequelize.models.User); // true
```

Internamente, `sequelize.define` chama `Model.init`, então ambas abordagens são essencialmente equivalentes.

## Inferência do nome da tabela

Observe que, em ambos os métodos acima, o nome da tabela (`Users`) nunca foi explicitamente definido. No entanto, o nome dado ao model foi (`User`).

Por padrão, quando o nome da tabela não é dado, Sequelize automaticamente pluraliza o nome do model e usa como o nome da tabela. Essa pluralização é feita embaixo do capô por uma library chamada [inflection](https://www.npmjs.com/package/inflection), assim plurais irregulares (tal como `person -> people`) são computados corretamente.

Claro, esse comportamente é facilmente configurável.

### Forçando o nome da tabela para ser igual ao nome do model

Você pode pausar a auto-pluralização performada pelo Sequelize usando a opção `freezeTableName: true`. Dessa maneira, Sequelize vai inferir o nome da tabela para ser equal ao nome do model, sem qualquer modificações:

```js
sequelize.define('User', {
  // ... (atributos)
}, {
  freezeTableName: true
});
```

O exemplo acima vai criar um model chamado `User` apontando para uma tabela também chamada `User`:

Esse comportamente pode também ser definido globalmente para a instância do sequelize, quando ele é criado:

```js
const sequelize = new Sequelize('sqlite::memory:', {
  define: {
    freezeTableName: true
  }
});
```

Dessa maneira, todas as tabelas vão usar o mesmo nome como o nome do model.

### Providenciando o nome da tabela diretamente

Você pode simplesmente dizer ao Sequelize o nome da tabela diretamente também:

```js
sequelize.define('User', {
  // ... (atributos)
}, {
  tableName: 'Employees'
});
```

## Sincronização do model

Quando você define um model, você está dizendo ao Sequelize algumas coisas sobre sua tabela no banco de dados. No entanto, e se a tabela na verdade nem existir no banco de dados? E se existe, mas tem diferentes colunas, menos colunas, ou qualquer outra diferença?

Isso é onde a sincronização do model entra. Um model pode ser sincronizado com o banco de dados chamando [`model.sync(options)`](https://sequelize.org/master/class/lib/model.js~Model.html#static-method-sync), uma função assíncrona (que retorna uma Promise). Com essa chamada, Sequelize vai automaticamente performar uma SQL query ao banco de dados. Note que isso altera somente a tabela no banco de dados, não o model no lado do Javascript.

* `User.sync()` - Cria a tabela se ela não existe (e não faz nada se já existe)
* `User.sync({ force: true })` - Cria a tabela, dropando ela primeiro se já existir
* `User.sync({ alter: true })` - Verifica qual é o atual estado da tabela no banco de dados (quais colunas ela tem, quais são seus tipos de dados, etc), e então performa as mudanças necessárias no tabela para fazer corresponder o model.

Exemplo:

```js
await User.sync({ force: true });
console.log("A tabela para o model User foi (re)criada!");
```

### Sincronizando todos os models de uma vez

Você pode usar [`sequelize.sync()`](../class/lib/sequelize.js~Sequelize.html#instance-method-sync) para automaticamente sincronizar todos os models. Exemplo:

```js
await sequelize.sync({ force: true });
console.log("Todos os models foram sincronizados com sucesso.");
```

### Dropando tabelas

Para dropar a tabela relacionada ao model:

```js
await User.drop();
console.log("Tabela User foi dropada!");
```

Para dropar todas as tabelas:

```js
await sequelize.drop();
console.log("Todas tabelas dropadas!");
```

### Verificação de segurança do banco de dados

Como mostrado acima, as operações `sync` e `drop` são destrutivas. Sequelize aceita uma opção `match` como um verificação de segurança adicional, no qual recebe um RegExp:

```js
// Isso vai executar .sync() somente se o nome do banco de dados termina com '_test'
sequelize.sync({ force: true, match: /_test$/ });
```

### Sincronização em produção

Como mostrado acima, `sync({ force: true })` e `sync({ alter: true })` podem ser operações destrutivas. Sendo assim, eles são não recomendados para software em nível de produção. Ao invés, sincronização deve ser feito com o conceito avançado de [Migrations](migrations.html), com a ajuda do [Sequelize CLI](https://github.com/sequelize/cli).

## Timestamps

Por padrão, Sequelize automaticamente adiciona os campos `createdAt` e `updatedAt` à todo model, usando o tipo de dado `DataTypes.DATE`. Esses campos são automaticamente gerenciados também - sempre que você usar Sequelize para criar ou atualizar alguma coisa, esses campos vão ser definidos corretamente. O campo `createdAt` vai conter a representação do timestamp no momento da criação, e o `updatedAt` vai conter o timestamp da última atualização.

**Nota:** Isso é feito no Sequelize (ex: não é feito com *SQL triggers*).Isso significa que queries SQL direta (por exemplo queries performadas sem Sequelize, por qualquer outro meio), não vai atualizar esses campos automaticamente.

Esse comportamente pode ser desativado por um model com a opção `timestamp: false`:

```js
sequelize.define('User', {
  // ... (atributos)
}, {
  timestamps: false
});
```

É também possível ativar somente um dos `createdAt`/`updatedAt`, e providenciar um nome personalizado para essas colunas:

```js
class Foo extends Model {}
Foo.init({ /* atributos */ }, {
  sequelize,

  // Não esqueça de ativar o timestamps
  timestamps: true,

  // Não quero createdAt
  createdAt: false,

  // Quero que updatedAt na verdade seja chamado updateTimestamp
  updatedAt: 'updateTimestamp'
});
```

## Sintaxe abreviada de declaração da coluna

Se a única coisa sendo especificada sobre uma coluna é o seu tipo de dado, a sintaxe pode ser abreviada:

```js
// Isso:
sequelize.define('User', {
  name: {
    type: DataTypes.STRING
  }
});

// Pode ser simplificado para:
sequelize.define('User', { name: DataTypes.STRING });
```

## Valores padrões

Por padrão, Sequelize assume que o valor padrão de uma coluna é `NULL`. Esse comportamente pode ser alterado ao passar um `defaultValue` específico para a definição da coluna:

```js
sequelize.define('User', {
  name: {
    type: DataTypes.STRING,
    defaultValue: "John Doe"
  }
});
```

Alguns valores especiais, tal como `Sequelize.NOW`, são também aceitos:

```js
sequelize.define('Foo', {
  bar: {
    type: DataTypes.DATETIME,
    defaultValue: Sequelize.NOW
    // Dessa maneira, a atual data/hora vai ser usado para preencher essa coluna (no momento da inserção)
  }
});
```

## Tipos de dados

Toda coluna que você define em seu model deve ter um tipo de dado. Sequelize providencia [bastante tipos de dados built-in](https://github.com/sequelize/sequelize/blob/master/lib/data-types.js). Para acessar um tipo de dado built-in, você deve importar `DataTypes`:

```js
const { DataTypes } = require("sequelize"); // Importa os tipos de dados built-in
```

### Strings

```js
DataTypes.STRING             // VARCHAR(255)
DataTypes.STRING(1234)       // VARCHAR(1234)
DataTypes.STRING.BINARY      // VARCHAR BINARY
DataTypes.TEXT               // TEXT
DataTypes.TEXT('tiny')       // TINYTEXT
DataTypes.CITEXT             // CITEXT          PostgreSQL e SQLite somente.
```

### Boolean

```js
DataTypes.BOOLEAN            // TINYINT(1)
```

### Numbers

```js
DataTypes.INTEGER            // INTEGER
DataTypes.BIGINT             // BIGINT
DataTypes.BIGINT(11)         // BIGINT(11)

DataTypes.FLOAT              // FLOAT
DataTypes.FLOAT(11)          // FLOAT(11)
DataTypes.FLOAT(11, 10)      // FLOAT(11,10)

DataTypes.REAL               // REAL            PostgreSQL somente.
DataTypes.REAL(11)           // REAL(11)        PostgreSQL somente.
DataTypes.REAL(11, 12)       // REAL(11,12)     PostgreSQL somente.

DataTypes.DOUBLE             // DOUBLE
DataTypes.DOUBLE(11)         // DOUBLE(11)
DataTypes.DOUBLE(11, 10)     // DOUBLE(11,10)

DataTypes.DECIMAL            // DECIMAL
DataTypes.DECIMAL(10, 2)     // DECIMAL(10,2)
```

#### Inteiros não assinados & inteiros Zerofill - MySQL/MariaDB somente

Em MySQL e MariaDB, os tipos de dados `INTEGER`, `BIGINT`, `FLOAT` e `DOUBLE` podem ser definidos como não assinados ou zerofill (ou ambos), como à seguir:

```js
DataTypes.INTEGER.UNSIGNED
DataTypes.INTEGER.ZEROFILL
DataTypes.INTEGER.UNSIGNED.ZEROFILL
// Você pode também especificar o tamanho, ex: INTEGER(10) ao invés de simplesmente INTEGER
// O mesmo para BIGINT, FLOAT e DOUBLE
```

### Datas

```js
DataTypes.DATE       // DATETIME para mysql / sqlite, TIMESTAMP WITH TIME ZONE para postgres
DataTypes.DATE(6)    // DATETIME(6) para mysql 5.6.4+. Segundos fracionários suportam até 6 dígitos de precisão
DataTypes.DATEONLY   // DATE sem hora
```

### UUIDs

Para UUIDs, use `DataTypes.UUID`. Ele torna o tipo de dado `UUID` para PostgreSQL e SQLite, e `CHAR(36)` para MySQL. Sequelize pode gerar UUIDs automaticamente para esses campos, simplesmente use `Sequelize.UUIDV1` or `Sequelize.UUIDV4` como valor padrão:

```js
{
  type: DataTypes.UUID,
  defaultValue: Sequelize.UUIDV4 // Ou Sequelize.UUIDV1
}
```

### Outros

Há outros tipos de dados, cobridos em um [guia separado](other-data-types.html).

## Opções das colunas

Quando definindo uma coluna, além de especificar o `type` da coluna, e as opções `allowNull` e `defaultValue` mencionadas acima, há muito mais opções que podem ser usadas. Alguns exemplos estão abaixo.

```js
const { Model, DataTypes, Deferrable } = require("sequelize");

class Foo extends Model {}
Foo.init({
  // Instanciamento vai automaticamente atribuir a flag para true se não definido
  flag: { type: DataTypes.BOOLEAN, allowNull: false, defaultValue: true },

  // Valores padrões para datas => hora atual
  myDate: { type: DataTypes.DATE, defaultValue: DataTypes.NOW },

  // Definindo allowNull para false vai adicionar NOT NULL na coluna, no qual significa um erro vai ser
  // lançado do DB quando a query é executada se a coluna é null. Se você quer checar que um valor
  // não é null antes de consultar o DB, olhe na seção de validações abaixo.
  title: { type: DataTypes.STRING, allowNull: false },


  // Criando dois objetos com o mesmo valor vai lançar um erro. A propriedade unica pode ser ou um boleano, ou uma string. Se você providenciar a mesma string para múltiplas colunas, eles vão formar uma chave única composta.
  // 
  uniqueOne: { type: DataTypes.STRING,  unique: 'compositeIndex' },
  uniqueTwo: { type: DataTypes.INTEGER, unique: 'compositeIndex' },

  // A propriedade unique é simplesmente uma abreviação para criar uma restrição exclusiva.
  someUnique: { type: DataTypes.STRING, unique: true },

  // Continue lendo para mais informações sobre chaves primárias
  identifier: { type: DataTypes.STRING, primaryKey: true },

  // autoIncrement pode ser usado para criar colunas do tipo inteiro para auto_incrementing
  incrementMe: { type: DataTypes.INTEGER, autoIncrement: true },

  // Você pode especificar um nome de coluna personalizado pelo atributo 'field':
  fieldWithUnderscores: { type: DataTypes.STRING, field: 'field_with_underscores' },

  // É possível criar chaves estrangeiras:
  bar_id: {
    type: DataTypes.INTEGER,

    references: {
      // Essa é uma referência à um outro model
      model: Bar,

      // Esse é o nome da coluna do model referenciado
      key: 'id',

      // Com PostgreSQL, é opcionalmente possível declarar quando verificar a restrição de chave estrangeira, passando o tipo Deferrable.
      deferrable: Deferrable.INITIALLY_IMMEDIATE
      
      // Opções:
      // - `Deferrable.INITIALLY_IMMEDIATE` - Imediatamente verifica a restrição de chave estrangeira
      // - `Deferrable.INITIALLY_DEFERRED` - Retarda toda verificação de restrição de chave estrangeira para o final de uma transação
      // - `Deferrable.NOT` - Não retarda a verificação (padrão) - Isso não vai permitir você dinamicamente mudar a regra em uma transação

    }
  },

  // Comentários podem somente ser adicionados às colunas em MySQL, MariaDB, PostgreSQL e MSSQL
  commentMe: {
    type: DataTypes.INTEGER,
    comment: 'Esse é um nome de coluna que tem um comentário'
  }
}, {
  sequelize,
  modelName: 'foo',

  // Usando `unique: true` em um atributo acima é exatamente o mesmo como criar um índice nas opções do model:
  indexes: [{ unique: true, fields: ['someUnique'] }]
});
```

## Tirando vantagem dos Models serem classes

Os models Sequelize são [ES6 classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes). Você pode muito facilmente adicionar instância personalizada ou métodos.

```js
class User extends Model {
  static classLevelMethod() {
    return 'foo';
  }
  instanceLevelMethod() {
    return 'bar';
  }
  getFullname() {
    return [this.firstname, this.lastname].join(' ');
  }
}
User.init({
  firstname: Sequelize.TEXT,
  lastname: Sequelize.TEXT
}, { sequelize });

console.log(User.classLevelMethod()); // 'foo'
const user = User.build({ firstname: 'Jane', lastname: 'Doe' });
console.log(user.instanceLevelMethod()); // 'bar'
console.log(user.getFullname()); // 'Jane Doe'
```