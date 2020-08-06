# Estratégias de nomenclatura

## A opção `underscored`

O Sequelize oferece a opção `underscored` para o modelo. Quando for `true`, essa opção definirá o nome de todos os atributos para a versão [snake_case](https://en.wikipedia.org/wiki/Snake_case) de seu nome. Isso também se aplica à chaves estrangeiras geradas automaticamente por relacionamentos e outros campos também gerados automaticamente. Exemplo:

```js
const User = sequelize.define('task', { username: Sequelize.STRING }, {
  underscored: true
});
const Task = sequelize.define('task', { title: Sequelize.STRING }, {
  underscored: true
});
User.hasMany(Task);
Task.belongsTo(User);
```

Acima temos dois modelos, User e Task, ambos usando a opção `underscored`. Também temos um relacionamento de Um-Para-Muitos entre eles. Lembre-se que como `timestamps` é true por padrão, esperamos os campos `createdAt` e `updatedAt` serem criados automaticamente.

Sem a opção `underscored`, O Sequelize definiria automaticamente:

* Um atributo `createdAt` para cada modelo, apontando para uma coluna `createdAt` em cada tabela.
* Um atributo `updatedAt` para cada modelo, apontando para uma coluna `updatedAt` em cada tabela.
* Um atributo `userId` no modelo `Task`, apontando para uma coluna de nome `userId` na tabela Task.

Com a opção `underscored` ativada, Sequelize definiria:

* Um atributo `createdAt` para cada modelo, apontando para uma coluna `created_at` em cada tabela.
* Um atributo `updatedAt` para cada modelo, apontando para uma coluna `updated_at` em cada tabela.
* Um atributo `userId` no modelo `Task`, apontando para uma coluna de nome `user_id` na tabela Task.

Note que em ambos os casos os campos continuam no padrão [camelCase](https://en.wikipedia.org/wiki/Camel_case) no lado do javascript; essa opção só muda a forma em que os campos são mapeados no banco de dados em si. A opção `field` de todos os atributos é definida para sua versão `snake_case`, porém o atributo continua no padrão `camelCase`.

Dessa forma, chamando `sync()` no código acima gera o seguinte:

```sql
CREATE TABLE IF NOT EXISTS "users" (
  "id" SERIAL,
  "username" VARCHAR(255),
  "created_at" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL,
  PRIMARY KEY ("id")
);
CREATE TABLE IF NOT EXISTS "tasks" (
  "id" SERIAL,
  "title" VARCHAR(255),
  "created_at" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL,
  "user_id" INTEGER REFERENCES "users" ("id") ON DELETE SET NULL ON UPDATE CASCADE,
  PRIMARY KEY ("id")
);
```

## Singular vs. Plural

À primeira vista, pode ser confuso se a forma singular ou plural de um nome deve ser usada no Sequelize. Essa seção visa esclarecer isso um pouco.

Lembre-se de que o Sequelize usa uma biblioteca chamada [inflection](https://www.npmjs.com/package/inflection) por baixo dos panos, então esses plurais irregulares (como `person -> people`) são calculados corretamente. Porém, se você estiver trabalhando em outro idioma sem ser o inglês, talvez você queira definir a forma plural e singular dos nomes diretamente; O Sequelize te permite fazer isso com algumas opções:

### Ao definir os modelos

Os modelos devem ser definidos na forma singular do nome:

```js
sequelize.define('foo', { name: DataTypes.STRING });
```

Acima, o nome do modelo é `foo` (singular), e sua respectiva tabela é `foos`, já que o Sequelize transforma o nome em plural automaticamente.

### Ao definir uma chave de referência no modelo

```js
sequelize.define('foo', {
  name: DataTypes.STRING,
  barId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    references: {
      model: "bars",
      key: "id"
    },
    onDelete: "CASCADE"
  },
});
```

No exemplo acima estamos definindo manualmente uma chave que referencia outro modelo. Não é comum fazer isso, mas se você precisar, você deve usar o nome da tabela na opção `model`, e não o do modelo. Isso porque a referência é criada após os dados serem computados. No exemplo acima, a forma plural foi usada (`bars`), assumingo que o modelo `bar` foi criado com as configurações padrão (tornando sua respectiva tabela pluralizada).

### Ao receber dados de um Eager loading

Quando você coloca `include` em uma query, os dados incluidos serão adicionados em um campo extra no objeto de resultados, de acordo com as seguintes regras:

* Quando incluindo dados de uma associação única (`hasOne` ou `belongsTo`) - o nome do campo será a versão singular do nome de seu modelo;
* Quando incluindo dados de uma associação múltipla (`hasMany` ou `belongsToMany`) - o nome do campo será a versão plural do nome de seu modelo;

Em resumo, o nome do campo seguirá a forma mais lógica para cada situação.

Exemplos:

```js
// Assumindo que Foo.hasMany(Bar)
const foo = Foo.findOne({ include: Bar });
// foo.bars será um array
// foo.bar não existirá, já que não faz sentido

// Assumindo que Foo.hasOne(Bar)
const foo = Foo.findOne({ include: Bar });
// foo.bar será um objeto (possivelmente nulo se não houver modelo associado)
// foo.bars não existirá, já que não faz sentido

// E assim por diante.
```

### Sobrescrevendo singulares e plurais ao definir apelidos

Ao definir um apelido para uma associação, ao invés de usar simplesmente `{ as: 'meuApelido' }`, você pode passar um objeto para especificar a forma singular e plural:

```js
Project.belongsToMany(User, {
  as: {
    singular: 'líder',
    plural: 'líderes'
  }
});
```

Se você tem certeza de que o modelo sempre usará o mesmo apelido nas associações, você pode definir a forma plural e singular diretamente no modelo:

```js
const User = sequelize.define('user', { /* ... */ }, {
  name: {
    singular: 'líder',
    plural: 'líderes',
  }
});
Project.belongsToMany(User);
```

Os mixins adicionados às instâncias do usuário usarão as formas corretos. Por exemplo, em vez de  `project.addUser()`, o Sequelize irá oferecer `project.getLíder()`. Também, em vez de `project.setUsers()`, o Sequelize irá oferecer `project.setLíderes()`.

Nota: lembre-se que usar `as` para alterar o nome de uma associação, também alterará o nome da chave estrangeira. Portanto, é recomendável especificar também as chaves estrangeiras envolvidas diretamente nesse caso.

```js
// Exemplo de possível erro
Invoice.belongsTo(Subscription, { as: 'TheSubscription' });
Subscription.hasMany(Invoice);
```

A primeira chamada acima definirá uma chava estrangeira chamada `theSubscriptionId` em `Invoice`. Contudo, a segunda chamada também irá gerar uma chave estrangeira em `Invoice` (desde que sabemos, as chamadas à `hasMany` define a chave estrangeira nos modelos alvo) - porém, ela será chamada de  `subscriptionId`. Dessa forma você terá ambas as colunas `subscriptionId` e `theSubscriptionId`.

A melhor abordagem é escolher um nome para a chave estrangeira e colocá-lo explicitamente nas duas chamadas. Por exemplo, se `subscription_id` foi escolhido:

```js
// Exemplo fixo
Invoice.belongsTo(Subscription, { as: 'TheSubscription', foreignKey: 'subscription_id' });
Subscription.hasMany(Invoice, { foreignKey: 'subscription_id' });
```