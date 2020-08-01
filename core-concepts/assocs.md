# Associações

Sequelize suporta as associações padrões [Um-Para-Um](https://en.wikipedia.org/wiki/One-to-one_%28data_model%29), [Um-Para-Muitos](https://en.wikipedia.org/wiki/One-to-many_%28data_model%29) and [Muitos-Para-Muitos](https://en.wikipedia.org/wiki/Many-to-many_%28data_model%29).

Para isso, Sequelize oferece **quatro** tipos de associações que devem serem combinadas para criá-las:

* A associação `HasOne`
* A associação `BelongsTo`
* A associação `HasMany` 
* A associação `BelongsToMany`

O guia começará explicando como definir esses quatros tipos de associações, e então irá prosseguir explicando como combina-las para definir os 3 tipos de associações ([Um-Para-Um](https://en.wikipedia.org/wiki/One-to-one_%28data_model%29), [Um-Para-Muitos](https://en.wikipedia.org/wiki/One-to-many_%28data_model%29) e [Muitos-Para-Muitos](https://en.wikipedia.org/wiki/Many-to-many_%28data_model%29)).

## Definindo as associações no sequelize

Os quatros tipos de associações são definidos em uma forma muito parecida. Suponhamos que temos 2 tabelas, `A` e `B`. Para dizer ao sequelize que você quer uma associação entre as duas, basta chamar uma função:

```js
const A = sequelize.define('A', /* ... */);
const B = sequelize.define('B', /* ... */);

A.hasOne(B); // A Tem um B
A.belongsTo(B); // A pertence a B
A.hasMany(B); // A tem muitos B
A.belongsToMany(B, { through: 'C' }); // A pertence a muitos B através da tabela de junção C
```

Todos eles aceitam um objeto de opções como um segundo parâmetro (opcional para os três primeiros, obrigatório para belongsToMany contendo pelo menos a propriedade through):

```js
A.hasOne(B, { /* opções */ });
A.belongsTo(B, { /* opções */ });
A.hasMany(B, { /* opções */ });
A.belongsToMany(B, { through: 'C', /* opções */ });
```

A ordem em que a associação é definida é relevante. Em outras palavras, a ordem importa, para os quatro casos. Em todos os exemplos acima, `A` é chamado de tabela **source (origem)** e `B` é chamado de tabela **target (alvo)**. Essa terminologia é importante.

A associação `A.hasOne(B)` significa que uma relação de Um-Para-Um existe entre `A` e `B`, com a chave estrangeira sendo definida na tabela alvo (`B`).

A associação `A.belongsTo(B)` significa que uma relação de Um-Para-Um existe entre `A` e `B`, com a chave estrangeira sendo definida na tabela origem (`A`).

A associação `A.hasMany(B)`  signifia que uma relação de Um-Para-Muitos existe entre `A` e `B`, com a chave estrangeira sendo definida na tabela alvo (`B`).

Essas três chamadas irá fazer com que o Sequelize adicione automaticamente chaves estrangeiras  para as tabelas apropriadas (A menos que elas ja estejam definidas).

A associação `A.belongsToMany(B, { through: 'C' })`  significa que um relacionamento de Muitos-Para-Muitos existe entre `A` e `B`, usando a tabela `C` como [Tabela de junção](https://en.wikipedia.org/wiki/Associative_entity), que irá ter as chaves estrangeiras (`aId` e `bId`, por exemplo). Sequeliza irá criar automaticamente a tabela `C` (a menos que ela ja existe) e definir as chaves estrangeiras apropriadas a ela.

*Nota: Nos exemplos acima para `belongsToMany`, uma string (`'C'`) foi passada para a opção through. Nesse caso, Sequelize gera automaticamente uma tabela com esse nome. Entretanto, você também pode passar esse modelo diretamente, se você já o definiu.*

Essas são as ideias principais envolvidas em cada tipo de associação. Entretanto, esses relacionamentos são frequentemente usados em pares, para permitir um melhor uso com o Sequelize. Isso será visto mais tarde.

## Criando os relacionamentos padrões

Como mencionado, As associações no Sequelize são comumente definidas em pares. No sumário:

* Para criar o relacionamento de **Um-Para-Um** , as associações `hasOne` e `belongsTo` são usadas juntas;
* Para criar o relacionamento de **Um-Para-Muitos** , as associações `hasMany` e `belongsTo` são usadas juntas;
* Para criar o relacionamento de **Muitos-Para-Muitos** , duas chamadas à `belongsToMany` são usadas juntas
  * Nota: Também há um relacionamento *Super Muitos-Para-Muitos* , que usa seis associações ao mesmo tempo, e será discutido no [Guia avançado de relacionamento Muitos-Para-Muitos](advanced-many-to-many.html).

Tudo isso será visto em detalhes a seguir. As vantagens de usar pares em vez de uma associação individual serão discutidas no final desse capítulo.

## Relacionamentos de Um-Para-Um

### Filosofia

Antes de se aprofundar nos aspectos do uso do Sequelize, é recomendável dar um passo para trás para entender o que acontece por tras de um relacionamento individual.

Digamos que temos duas tabelas, `Foo` e `Bar`. Queremos estabelecer um relacionamento de  Um-Para-Um Entre Foo e Bar. Sabemos que em um banco de dados relacional, isso seria feito criando uma chave estrangeira em uma das tabelas. Então nesse caso, uma pergunta muito relevante é: em qual dessas tabelas queremos que a chave estrangeira seja adicionada? Em outras palavras, queremos que `Foo` tenha uma coluna `barId`, ou ao invés disso, `Bar` é quem deveria ter uma coluna `fooId` ?

A principio, ambas opções são válidas ao estabelecer um relacionamento de Um-Para-Um entre Foo e Bar. Todavia, quando dizemos algo como *"há um relacionamento de Um-Para-Um entre Foo e Bar"*, não fica claro se o relacionamento é obrigatório ou opcional. Em outras palavras, a tabela Foo pode existir sem a tabela Bar? A tabela Bar pode existir sem a tabela Foo? As respostas para essas perguntas ajudam a definir onde queremos que a chave estrangeira seja aplicada.

### Objetivo

Para o restante desse exemplo, vamos supor que temos duas tabelas, `Foo` e `Bar`. Queremos estabelecer um relacionamento de Um-Para-Um entre elas, de uma forma que `Bar` tenha uma coluna `fooId`.

### Implementação

A estrutura principal para atingir esse objetivo é a seguinte:

```js
Foo.hasOne(Bar);
Bar.belongsTo(Foo);
```

Já que não foi passado nenhuma opção como segundo parâmetro, Sequelize irá deduzir o que fazer a partir dos nomes das tabelas. Nesse caso, Sequelize sabe que uma coluna `fooId` precisa ser adicionada a `Bar`.

Dessa forma, chamar `Bar.sync()` após o procedimento acima produzirá o seguinte SQL (no PostgreSQL, por exemplo)

```sql
CREATE TABLE IF NOT EXISTS "foos" (
  /* ... */
);
CREATE TABLE IF NOT EXISTS "bars" (
  /* ... */
  "fooId" INTEGER REFERENCES "foos" ("id") ON DELETE SET NULL ON UPDATE CASCADE
  /* ... */
);
```

### Opções

Várias opções podem ser passadas como segundo parametro da função que realiza a associação

#### `onDelete` e `onUpdate`

Por exemplo, para configurar o comportamento `ON DELETE` e `ON UPDATE`

```js
Foo.hasOne(Bar, {
  onDelete: 'RESTRICT',
  onUpdate: 'RESTRICT'
});
Bar.belongsTo(Foo);
```

As possíveis escolhas são `RESTRICT`, `CASCADE`, `NO ACTION`, `SET DEFAULT` e `SET NULL`.

O padrão para associacões de Um-Para-Um são `SET NULL` para `ON DELETE` e `CASCADE` para `ON UPDATE`.

#### Customizando a chave estrangeira

Ambas as chamads `hasOne` e `belongsTo` mostradas acima irão deduzir que a chave estrangeira a ser criada deve ser chamada de `fooId`. Para usar um nome diferente, como `myFooId`:

```js
// Opção 1
Foo.hasOne(Bar, {
  foreignKey: 'myFooId'
});
Bar.belongsTo(Foo);

// Opção 2
Foo.hasOne(Bar, {
  foreignKey: {
    name: 'myFooId'
  }
});
Bar.belongsTo(Foo);

// Opção 3
Foo.hasOne(Bar);
Bar.belongsTo(Foo, {
  foreignKey: 'myFooId'
});

// Opção 4
Foo.hasOne(Bar);
Bar.belongsTo(Foo, {
  foreignKey: {
    name: 'myFooId'
  }
});
```

Como mostrado acima, a opção `foreignKey` aceita uma string ou um objeto. quando recebe um objeto, o mesmo será usado como definição para a coluna, da mesma forma que faria em uma chamada a função `sequelize.define`. Portanto, especificando opções como `type`, `allowNull`, `defaultValue`, etc, funciona.

Por exemplo, para usar `UUID` como tipo de dado da chave estrangeiro em vez do padrão (`INTEGER`), você pode simplesmente fazer:

```js
const { DataTypes } = require("Sequelize");

Foo.hasOne(Bar, {
  foreignKey: {
    // name: 'myFooId'
    type: DataTypes.UUID
  }
});
Bar.belongsTo(Foo);
```

#### Associações obrigatórias versus opcionais

Por padrão, a associação é considerada opcional. Em outras palavras, no nosso exemplo, a coluna `fooId` pode ser nula, significando que a tabela Bar pode existir sem a tabela Foo. Alterar isso é simplesmente definir `allowNull: false` nas opções de chave estrangeira:

```js
Foo.hasOne(Bar, {
  foreignKey: {
    allowNull: false
  }
});
// "fooId" INTEGER NOT NULL REFERENCES "foos" ("id") ON DELETE RESTRICT ON UPDATE RESTRICT
```

## Relacionamentos de Um-Para-Muitos

### Filosofia

One-To-Many associations are connecting one source with multiple targets, while all these targets are connected only with this single source.

This means that, unlike the One-To-One association, in which we had to choose where the foreign key would be placed, there is only one option in One-To-Many associations. For example, if one Foo has many Bars (and this way each Bar belongs to one Foo), then the only sensible implementation is to have a `fooId` column in the `Bar` table. The opposite is impossible, since one Foo has many Bars.

### Goal

In this example, we have the models `Team` and `Player`. We want to tell Sequelize that there is a One-To-Many relationship between them, meaning that one Team has many Players, while each Player belongs to a single Team.

### Implementation

The main way to do this is as follows:

```js
Team.hasMany(Player);
Player.belongsTo(Team);
```

Again, as mentioned, the main way to do it used a pair of Sequelize associations (`hasMany` and `belongsTo`).

For example, in PostgreSQL, the above setup will yield the following SQL upon `sync()`:

```sql
CREATE TABLE IF NOT EXISTS "Teams" (
  /* ... */
);
CREATE TABLE IF NOT EXISTS "Players" (
  /* ... */
  "TeamId" INTEGER REFERENCES "Teams" ("id") ON DELETE SET NULL ON UPDATE CASCADE,
  /* ... */
);
```

### Options

The options to be applied in this case are the same from the One-To-One case. For example, to change the name of the foreign key and make sure that the relationship is mandatory, we can do:

```js
Team.hasMany(Player, {
  foreignKey: 'clubId'
});
Player.belongsTo(Team);
```

Like One-To-One relationships, `ON DELETE` defaults to `SET NULL` and `ON UPDATE` defaults to `CASCADE`.

## Many-To-Many relationships

### Philosophy

Many-To-Many associations connect one source with multiple targets, while all these targets can in turn be connected to other sources beyond the first.

This cannot be represented by adding one foreign key to one of the tables, like the other relationships did. Instead, the concept of a [Junction Model](https://en.wikipedia.org/wiki/Associative_entity) is used. This will be an extra model (and extra table in the database) which will have two foreign key columns and will keep track of the associations. The junction table is also sometimes called *join table* or *through table*.

### Goal

For this example, we will consider the models `Movie` and `Actor`. One actor may have participated in many movies, and one movie had many actors involved with its production. The junction table that will keep track of the associations will be called `ActorMovies`, which will contain the foreign keys `movieId` and `actorId`.

### Implementation

The main way to do this in Sequelize is as follows:

```js
const Movie = sequelize.define('Movie', { name: DataTypes.STRING });
const Actor = sequelize.define('Actor', { name: DataTypes.STRING });
Movie.belongsToMany(Actor, { through: 'ActorMovies' });
Actor.belongsToMany(Movie, { through: 'ActorMovies' });
```

Since a string was given in the `through` option of the `belongsToMany` call, Sequelize will automatically create the `ActorMovies` model which will act as the junction model. For example, in PostgreSQL:

```sql
CREATE TABLE IF NOT EXISTS "ActorMovies" (
  "createdAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updatedAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "MovieId" INTEGER REFERENCES "Movies" ("id") ON DELETE CASCADE ON UPDATE CASCADE,
  "ActorId" INTEGER REFERENCES "Actors" ("id") ON DELETE CASCADE ON UPDATE CASCADE,
  PRIMARY KEY ("MovieId","ActorId")
);
```

Instead of a string, passing a model directly is also supported, and in that case the given model will be used as the junction model (and no model will be created automatically). For example:

```js
const Movie = sequelize.define('Movie', { name: DataTypes.STRING });
const Actor = sequelize.define('Actor', { name: DataTypes.STRING });
const ActorMovies = sequelize.define('ActorMovies', {
  MovieId: {
    type: DataTypes.INTEGER,
    references: {
      model: Movie, // 'Movies' would also work
      key: 'id'
    }
  },
  ActorId: {
    type: DataTypes.INTEGER,
    references: {
      model: Actor, // 'Actors' would also work
      key: 'id'
    }
  }
});
Movie.belongsToMany(Actor, { through: 'ActorMovies' });
Actor.belongsToMany(Movie, { through: 'ActorMovies' });
```

The above yields the following SQL in PostgreSQL, which is equivalent to the one shown above:

```sql
CREATE TABLE IF NOT EXISTS "ActorMovies" (
  "MovieId" INTEGER NOT NULL REFERENCES "Movies" ("id") ON DELETE RESTRICT ON UPDATE CASCADE,
  "ActorId" INTEGER NOT NULL REFERENCES "Actors" ("id") ON DELETE RESTRICT ON UPDATE CASCADE,
  "createdAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updatedAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  UNIQUE ("MovieId", "ActorId"),     -- Note: Sequelize generated this UNIQUE constraint but
  PRIMARY KEY ("MovieId","ActorId")  -- it is irrelevant since it's also a PRIMARY KEY
);
```

### Options

Unlike One-To-One and One-To-Many relationships, the defaults for both `ON UPDATE` and `ON DELETE` are `CASCADE` for Many-To-Many relationships.

Belongs-To-Many creates a unique key when primary key is not present on through model. This unique key name can be overridden using **uniqueKey** option.

```js
Project.belongsToMany(User, { through: UserProjects, uniqueKey: 'my_custom_unique' })
```

## Basics of queries involving associations

With the basics of defining associations covered, we can look at queries involving associations. The most common queries on this matter are the *read* queries (i.e. SELECTs). Later on, other types of queries will be shown.

In order to study this, we will consider an example in which we have Ships and Captains, and a one-to-one relationship between them. We will allow null on foreign keys (the default), meaning that a Ship can exist without a Captain and vice-versa.

```js
// This is the setup of our models for the examples below
const Ship = sequelize.define('ship', {
  name: DataTypes.TEXT,
  crewCapacity: DataTypes.INTEGER,
  amountOfSails: DataTypes.INTEGER
}, { timestamps: false });
const Captain = sequelize.define('captain', {
  name: DataTypes.TEXT,
  skillLevel: {
    type: DataTypes.INTEGER,
    validate: { min: 1, max: 10 }
  }
}, { timestamps: false });
Captain.hasOne(Ship);
Ship.belongsTo(Captain);
```

### Fetching associations - Eager Loading vs Lazy Loading

The concepts of Eager Loading and Lazy Loading are fundamental to understand how fetching associations work in Sequelize. Lazy Loading refers to the technique of fetching the associated data only when you really want it; Eager Loading, on the other hand, refers to the technique of fetching everything at once, since the beginning, with a larger query.

#### Lazy Loading example

```js
const awesomeCaptain = await Captain.findOne({
  where: {
    name: "Jack Sparrow"
  }
});
// Do stuff with the fetched captain
console.log('Name:', awesomeCaptain.name);
console.log('Skill Level:', awesomeCaptain.skillLevel);
// Now we want information about his ship!
const hisShip = await awesomeCaptain.getShip();
// Do stuff with the ship
console.log('Ship Name:', hisShip.name);
console.log('Amount of Sails:', hisShip.amountOfSails);
```

Observe that in the example above, we made two queries, only fetching the associated ship when we wanted to use it. This can be especially useful if we may or may not need the ship, perhaps we want to fetch it conditionally, only in a few cases; this way we can save time and memory by only fetching it when necessary.

Note: the `getShip()` instance method used above is one of the methods Sequelize automatically adds to `Captain` instances. There are others. You will learn more about them later in this guide.

#### Eager Loading Example

```js
const awesomeCaptain = await Captain.findOne({
  where: {
    name: "Jack Sparrow"
  },
  include: Ship
});
// Now the ship comes with it
console.log('Name:', awesomeCaptain.name);
console.log('Skill Level:', awesomeCaptain.skillLevel);
console.log('Ship Name:', awesomeCaptain.ship.name);
console.log('Amount of Sails:', awesomeCaptain.ship.amountOfSails);
```

As shown above, Eager Loading is performed in Sequelize by using the `include` option. Observe that here only one query was performed to the database (which brings the associated data along with the instance).

This was just a quick introduction to Eager Loading in Sequelize. There is a lot more to it, which you can learn at [the dedicated guide on Eager Loading](eager-loading.html).

### Creating, updating and deleting

The above showed the basics on queries for fetching data involving associations. For creating, updating and deleting, you can either:

* Use the standard model queries directly:

  ```js
  // Example: creating an associated model using the standard methods
  Bar.create({
    name: 'My Bar',
    fooId: 5
  });
  // This creates a Bar belonging to the Foo of ID 5 (since fooId is
  // a regular column, after all). Nothing very clever going on here.
  ```

* Or use the *[special methods/mixins](#special-methods-mixins-added-to-instances)* available for associated models, which are explained later on this page.

**Note:** The [`save()` instance method](../class/lib/model.js~Model.html#instance-method-save) is not aware of associations. In other words, if you change a value from a *child* object that was eager loaded along a *parent* object, calling `save()` on the parent will completely ignore the change that happened on the child.

## Association Aliases & Custom Foreign Keys

In all the above examples, Sequelize automatically defined the foreign key names. For example, in the Ship and Captain example, Sequelize automatically defined a `captainId` field on the Ship model. However, it is easy to specify a custom foreign key.

Let's consider the models Ship and Captain in a simplified form, just to focus on the current topic, as shown below (less fields):

```js
const Ship = sequelize.define('ship', { name: DataTypes.TEXT }, { timestamps: false });
const Captain = sequelize.define('captain', { name: DataTypes.TEXT }, { timestamps: false });
```

There are three ways to specify a different name for the foreign key:

* By providing the foreign key name directly
* By defining an Alias
* By doing both things

### Recap: the default setup

By using simply `Ship.belongsTo(Captain)`, sequelize will generate the foreign key name automatically:

```js
Ship.belongsTo(Captain); // This creates the `captainId` foreign key in Ship.

// Eager Loading is done by passing the model to `include`:
console.log((await Ship.findAll({ include: Captain })).toJSON());
// Or by providing the associated model name:
console.log((await Ship.findAll({ include: 'captain' })).toJSON());

// Also, instances obtain a `getCaptain()` method for Lazy Loading:
const ship = Ship.findOne();
console.log((await ship.getCaptain()).toJSON());
```

### Providing the foreign key name directly

The foreign key name can be provided directly with an option in the association definition, as follows:

```js
Ship.belongsTo(Captain, { foreignKey: 'bossId' }); // This creates the `bossId` foreign key in Ship.

// Eager Loading is done by passing the model to `include`:
console.log((await Ship.findAll({ include: Captain })).toJSON());
// Or by providing the associated model name:
console.log((await Ship.findAll({ include: 'Captain' })).toJSON());

// Also, instances obtain a `getCaptain()` method for Lazy Loading:
const ship = Ship.findOne();
console.log((await ship.getCaptain()).toJSON());
```

### Defining an Alias

Defining an Alias is more powerful than simply specifying a custom name for the foreign key. This is better understood with an example:

<!-- NOTE: any change in this part might also require a change on advanced-many-to-many.md -->

```js
Ship.belongsTo(Captain, { as: 'leader' }); // This creates the `leaderId` foreign key in Ship.

// Eager Loading no longer works by passing the model to `include`:
console.log((await Ship.findAll({ include: Captain })).toJSON()); // Throws an error
// Instead, you have to pass the alias:
console.log((await Ship.findAll({ include: 'leader' })).toJSON());
// Or you can pass an object specifying the model and alias:
console.log((await Ship.findAll({
  include: {
    model: Captain,
    as: 'leader'
  }
})).toJSON());

// Also, instances obtain a `getLeader()` method for Lazy Loading:
const ship = Ship.findOne();
console.log((await ship.getLeader()).toJSON());
```

Aliases are especially useful when you need to define two different associations between the same models. For example, if we have the models `Mail` and `Person`, we may want to associate them twice, to represent the `sender` and `receiver` of the Mail. In this case we must use an alias for each association, since otherwise a call like `mail.getPerson()` would be ambiguous. With the `sender` and `receiver` aliases, we would have the two methods available and working: `mail.getSender()` and `mail.getReceiver()`, both of them returning a `Promise<Person>`.

When defining an alias for a `hasOne` or `belongsTo` association, you should use the singular form of a word (such as `leader`, in the example above). On the other hand, when defining an alias for `hasMany` and `belongsToMany`, you should use the plural form. Defining aliases for Many-to-Many relationships (with `belongsToMany`) is covered in the [Advanced Many-to-Many Associations guide](advanced-many-to-many.html).

### Doing both things

We can define and alias and also directly define the foreign key:

```js
Ship.belongsTo(Captain, { as: 'leader', foreignKey: 'bossId' }); // This creates the `bossId` foreign key in Ship.

// Since an alias was defined, eager Loading doesn't work by simply passing the model to `include`:
console.log((await Ship.findAll({ include: Captain })).toJSON()); // Throws an error
// Instead, you have to pass the alias:
console.log((await Ship.findAll({ include: 'leader' })).toJSON());
// Or you can pass an object specifying the model and alias:
console.log((await Ship.findAll({
  include: {
    model: Captain,
    as: 'leader'
  }
})).toJSON());

// Also, instances obtain a `getLeader()` method for Lazy Loading:
const ship = Ship.findOne();
console.log((await ship.getLeader()).toJSON());
```

## Special methods/mixins added to instances

When an association is defined between two models, the instances of those models gain special methods to interact with their associated counterparts.

For example, if we have two models, `Foo` and `Bar`, and they are associated, their instances will have the following methods/mixins available, depending on the association type:

### `Foo.hasOne(Bar)`

* `fooInstance.getBar()`
* `fooInstance.setBar()`
* `fooInstance.createBar()`

Example:

```js
const foo = await Foo.create({ name: 'the-foo' });
const bar1 = await Bar.create({ name: 'some-bar' });
const bar2 = await Bar.create({ name: 'another-bar' });
console.log(await foo.getBar()); // null
await foo.setBar(bar1);
console.log((await foo.getBar()).name); // 'some-bar'
await foo.createBar({ name: 'yet-another-bar' });
const newlyAssociatedBar = await foo.getBar();
console.log(newlyAssociatedBar.name); // 'yet-another-bar'
await foo.setBar(null); // Un-associate
console.log(await foo.getBar()); // null
```

### `Foo.belongsTo(Bar)`

The same ones from `Foo.hasOne(Bar)`:

* `fooInstance.getBar()`
* `fooInstance.setBar()`
* `fooInstance.createBar()`

### `Foo.hasMany(Bar)`

* `fooInstance.getBars()`
* `fooInstance.countBars()`
* `fooInstance.hasBar()`
* `fooInstance.hasBars()`
* `fooInstance.setBars()`
* `fooInstance.addBar()`
* `fooInstance.addBars()`
* `fooInstance.removeBar()`
* `fooInstance.removeBars()`
* `fooInstance.createBar()`

Example:

```js
const foo = await Foo.create({ name: 'the-foo' });
const bar1 = await Bar.create({ name: 'some-bar' });
const bar2 = await Bar.create({ name: 'another-bar' });
console.log(await foo.getBars()); // []
console.log(await foo.countBars()); // 0
console.log(await foo.hasBar(bar1)); // false
await foo.addBars([bar1, bar2]);
console.log(await foo.countBars()); // 2
await foo.addBar(bar1);
console.log(await foo.countBars()); // 2
console.log(await foo.hasBar(bar1)); // true
await foo.removeBar(bar2);
console.log(await foo.countBars()); // 1
await foo.createBar({ name: 'yet-another-bar' });
console.log(await foo.countBars()); // 2
await foo.setBars([]); // Un-associate all previously associated bars
console.log(await foo.countBars()); // 0
```

The getter method accepts options just like the usual finder methods (such as `findAll`):

```js
const easyTasks = await project.getTasks({
  where: {
    difficulty: {
      [Op.lte]: 5
    }
  }
});
const taskTitles = (await project.getTasks({
  attributes: ['title'],
  raw: true
})).map(task => task.title);
```

### `Foo.belongsToMany(Bar, { through: Baz })`

The same ones from `Foo.hasMany(Bar)`:

* `fooInstance.getBars()`
* `fooInstance.countBars()`
* `fooInstance.hasBar()`
* `fooInstance.hasBars()`
* `fooInstance.setBars()`
* `fooInstance.addBar()`
* `fooInstance.addBars()`
* `fooInstance.removeBar()`
* `fooInstance.removeBars()`
* `fooInstance.createBar()`

### Note: Method names

As shown in the examples above, the names Sequelize gives to these special methods are formed by a prefix (e.g. `get`, `add`, `set`) concatenated with the model name (with the first letter in uppercase). When necessary, the plural is used, such as in `fooInstance.setBars()`. Again, irregular plurals are also handled automatically by Sequelize. For example, `Person` becomes `People` and `Hypothesis` becomes `Hypotheses`.

If an alias was defined, it will be used instead of the model name to form the method names. For example:

```js
Task.hasOne(User, { as: 'Author' });
```

* `taskInstance.getAuthor()`
* `taskInstance.setAuthor()`
* `taskInstance.createAuthor()`

## Why associations are defined in pairs?

As mentioned earlier and shown in most examples above, usually associations in Sequelize are defined in pairs:

* To create a **One-To-One** relationship, the `hasOne` and `belongsTo` associations are used together;
* To create a **One-To-Many** relationship, the `hasMany` and `belongsTo` associations are used together;
* To create a **Many-To-Many** relationship, two `belongsToMany` calls are used together.

When a Sequelize association is defined between two models, only the *source* model *knows about it*. So, for example, when using `Foo.hasOne(Bar)` (so `Foo` is the source model and `Bar` is the target model), only `Foo` knows about the existence of this association. This is why in this case, as shown above, `Foo` instances gain the methods `getBar()`, `setBar()` and `createBar()`, while on the other hand `Bar` instances get nothing.

Similarly, for `Foo.hasOne(Bar)`, since `Foo` knows about the relationship, we can perform eager loading as in `Foo.findOne({ include: Bar })`, but we can't do `Bar.findOne({ include: Foo })`.

Therefore, to bring full power to Sequelize usage, we usually setup the relationship in pairs, so that both models get to *know about it*.

Practical demonstration:

* If we do not define the pair of associations, calling for example just `Foo.hasOne(Bar)`:

  ```js
  // This works...
  await Foo.findOne({ include: Bar });

  // But this throws an error:
  await Bar.findOne({ include: Foo });
  // SequelizeEagerLoadingError: foo is not associated to bar!
  ```

* If we define the pair as recommended, i.e., both `Foo.hasOne(Bar)` and `Bar.belongsTo(Foo)`:

  ```js
  // This works!
  await Foo.findOne({ include: Bar });

  // This also works!
  await Bar.findOne({ include: Foo });
  ```

## Multiple associations involving the same models

In Sequelize, it is possible to define multiple associations between the same models. You just have to define different aliases for them:

```js
Team.hasOne(Game, { as: 'HomeTeam', foreignKey: 'homeTeamId' });
Team.hasOne(Game, { as: 'AwayTeam', foreignKey: 'awayTeamId' });
Game.belongsTo(Team);
```

## Creating associations referencing a field which is not the primary key

In all the examples above, the associations were defined by referencing the primary keys of the involved models (in our case, their IDs). However, Sequelize allows you to define an association that uses another field, instead of the primary key field, to establish the association.

This other field must have a unique constraint on it (otherwise, it wouldn't make sense).

### For `belongsTo` relationships

First, recall that the `A.belongsTo(B)` association places the foreign key in the *source model* (i.e., in `A`).

Let's again use the example of Ships and Captains. Additionally, we will assume that Captain names are unique:

```js
const Ship = sequelize.define('ship', { name: DataTypes.TEXT }, { timestamps: false });
const Captain = sequelize.define('captain', {
  name: { type: DataTypes.TEXT, unique: true }
}, { timestamps: false });
```

This way, instead of keeping the `captainId` on our Ships, we could keep a `captainName` instead and use it as our association tracker. In other words, instead of referencing the `id` from the target model (Captain), our relationship will reference another column on the target model: the `name` column. To specify this, we have to define a *target key*. We will also have to specify a name for the foreign key itself:

```js
Ship.belongsTo(Captain, { targetKey: 'name', foreignKey: 'captainName' });
// This creates a foreign key called `captainName` in the source model (Ship)
// which references the `name` field from the target model (Captain).
```

Now we can do things like:

```js
await Captain.create({ name: "Jack Sparrow" });
const ship = await Ship.create({ name: "Black Pearl", captainName: "Jack Sparrow" });
console.log((await ship.getCaptain()).name); // "Jack Sparrow"
```

### For `hasOne` and `hasMany` relationships

The exact same idea can be applied to the `hasOne` and `hasMany` associations, but instead of providing a `targetKey`, we provide a `sourceKey` when defining the association. This is because unlike `belongsTo`, the `hasOne` and `hasMany` associations keep the foreign key on the target model:

```js
const Foo = sequelize.define('foo', {
  name: { type: DataTypes.TEXT, unique: true }
}, { timestamps: false });
const Bar = sequelize.define('bar', {
  title: { type: DataTypes.TEXT, unique: true }
}, { timestamps: false });
const Baz = sequelize.define('baz', { summary: DataTypes.TEXT }, { timestamps: false });
Foo.hasOne(Bar, { sourceKey: 'name', foreignKey: 'fooName' });
Bar.hasMany(Baz, { sourceKey: 'title', foreignKey: 'barTitle' });
// [...]
await Bar.setFoo("Foo's Name Here");
await Baz.addBar("Bar's Title Here");
```

### For `belongsToMany` relationships

The same idea can also be applied to `belongsToMany` relationships. However, unlike the other situations, in which we have only one foreign key involved, the `belongsToMany` relationship involves two foreign keys which are kept on an extra table (the junction table).

Consider the following setup:

```js
const Foo = sequelize.define('foo', {
  name: { type: DataTypes.TEXT, unique: true }
}, { timestamps: false });
const Bar = sequelize.define('bar', {
  title: { type: DataTypes.TEXT, unique: true }
}, { timestamps: false });
```

There are four cases to consider:

* We might want a many-to-many relationship using the default primary keys for both `Foo` and `Bar`:

```js
Foo.belongsToMany(Bar, { through: 'foo_bar' });
// This creates a junction table `foo_bar` with fields `fooId` and `barId`
```

* We might want a many-to-many relationship using the default primary key for `Foo` but a different field for `Bar`:

```js
Foo.belongsToMany(Bar, { through: 'foo_bar', targetKey: 'title' });
// This creates a junction table `foo_bar` with fields `fooId` and `barTitle`
```

* We might want a many-to-many relationship using the a different field for `Foo` and the default primary key for `Bar`:

```js
Foo.belongsToMany(Bar, { through: 'foo_bar', sourceKey: 'name' });
// This creates a junction table `foo_bar` with fields `fooName` and `barId`
```

* We might want a many-to-many relationship using different fields for both `Foo` and `Bar`:

```js
Foo.belongsToMany(Bar, { through: 'foo_bar', sourceKey: 'name', targetKey: 'title' });
// This creates a junction table `foo_bar` with fields `fooName` and `barTitle`
```

### Notes

Don't forget that the field referenced in the association must have a unique constraint placed on it. Otherwise, an error will be thrown (and sometimes with a mysterious error message - such as `SequelizeDatabaseError: SQLITE_ERROR: foreign key mismatch - "ships" referencing "captains"` for SQLite).

The trick to deciding between `sourceKey` and `targetKey` is just to remember where each relationship places its foreign key. As mentioned in the beginning of this guide:

* `A.belongsTo(B)` keeps the foreign key in the source model (`A`), therefore the referenced key is in the target model, hence the usage of `targetKey`.

* `A.hasOne(B)` and `A.hasMany(B)` keep the foreign key in the target model (`B`), therefore the referenced key is in the source model, hence the usage of `sourceKey`.

* `A.belongsToMany(B)` involves an extra table (the junction table), therefore both `sourceKey` and `targetKey` are usable, with `sourceKey` corresponding to some field in `A` (the source) and `targetKey` corresponding to some field in `B` (the target).
