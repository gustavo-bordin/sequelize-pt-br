# Associações

Sequelize suporta as associações padrão [Um-Para-Um](https://en.wikipedia.org/wiki/One-to-one_%28data_model%29), [Um-Para-Muitos](https://en.wikipedia.org/wiki/One-to-many_%28data_model%29) e [Muitos-Para-Muitos](https://en.wikipedia.org/wiki/Many-to-many_%28data_model%29).

Para isso, Sequelize oferece **quatro** tipos de associações que devem ser combinadas para criá-las:

* A associação `HasOne`
* A associação `BelongsTo`
* A associação `HasMany` 
* A associação `BelongsToMany`

O guia começará explicando como definir esses quatros tipos de associações, e então irá prosseguir explicando como combina-las para definir os 3 tipos de associações ([Um-Para-Um](https://en.wikipedia.org/wiki/One-to-one_%28data_model%29), [Um-Para-Muitos](https://en.wikipedia.org/wiki/One-to-many_%28data_model%29) e [Muitos-Para-Muitos](https://en.wikipedia.org/wiki/Many-to-many_%28data_model%29)).

## Definindo as associações no Sequelize

Os quatros tipos de associações são definidos de forma muito parecida. Suponhamos que temos 2 modelos, `A` e `B`. Para dizer ao Sequelize que você quer uma associação entre os dois, basta chamar uma função:

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

A ordem em que a associação é definida é relevante. Em outras palavras, a ordem importa para os quatro casos. Em todos os exemplos acima, `A` é chamado de modelo **source (origem)** e `B` é chamado de modelo **target (alvo)**. Essa terminologia é importante.

A associação `A.hasOne(B)` significa que uma relação de Um-Para-Um existe entre `A` e `B`, com a chave estrangeira sendo definida no modelo alvo (`B`).

A associação `A.belongsTo(B)` significa que uma relação de Um-Para-Um existe entre `A` e `B`, com a chave estrangeira sendo definida no modelo origem (`A`).

A associação `A.hasMany(B)` signifia que uma relação de Um-Para-Muitos existe entre `A` e `B`, com a chave estrangeira sendo definida na modelo alvo (`B`).

Essas três chamadas irá fazer com que o Sequelize adicione automaticamente chaves estrangeiras para as modelos apropriados (A menos que elas ja estejam definidas).

A associação `A.belongsToMany(B, { through: 'C' })`  significa que um relacionamento de Muitos-Para-Muitos existe entre `A` e `B`, usando a tabela `C` como [Tabela de junção](https://en.wikipedia.org/wiki/Associative_entity), que irá ter as chaves estrangeiras (`aId` e `bId`, por exemplo). Sequeliza irá criar automaticamente o modelo `C` (a menos que ele ja existe) e definir as chaves estrangeiras apropriadas a ela.

*Nota: Nos exemplos de `belongsToMany` acima, uma string (`'C'`) foi passada para a opção through. Nesse caso, Sequelize gera automaticamente um modelo com esse nome. Porém, você também pode passar um modelo diretamente, se você já definiu um*

Essas são as ideias principais envolvidas em cada tipo de associação. Entretanto, esses relacionamentos são frequentemente usados em pares, para permitir um melhor uso com o Sequelize. Isso será visto mais tarde.

## Criando os relacionamentos padrão

Como mencionado, As associações no Sequelize são comumente definidas em pares. No sumário:

* Para criar o relacionamento de **Um-Para-Um** , as associações `hasOne` e `belongsTo` são usadas juntas;
* Para criar o relacionamento de **Um-Para-Muitos** , as associações `hasMany` e `belongsTo` são usadas juntas;
* Para criar o relacionamento de **Muitos-Para-Muitos** , duas chamadas à `belongsToMany` são usadas juntas
  * Nota: Também há um relacionamento *Super Muitos-Para-Muitos* , que usa seis associações ao mesmo tempo, e será discutido no [Guia avançado de relacionamento Muitos-Para-Muitos](advanced-many-to-many.html).

Tudo isso será visto em detalhes a seguir. As vantagens de usar pares em vez de uma associação individual serão discutidas no final desse capítulo.

## Relacionamentos de Um-Para-Um

### Filosofia

Antes de se aprofundar nos aspectos do uso do Sequelize, é recomendável dar um passo para trás para entender o que acontece por tras de um relacionamento individual.

Digamos que temos dois modelos, `Foo` e `Bar`. Queremos estabelecer um relacionamento de  Um-Para-Um Entre Foo e Bar. Sabemos que em um banco de dados relacional, isso seria feito criando uma chave estrangeira em uma das tabelas. Então nesse caso, uma pergunta muito relevante é: em qual dessas tabelas queremos que a chave estrangeira seja adicionada? Em outras palavras, queremos que `Foo` tenha uma coluna `barId`, ou ao invés disso, `Bar` é quem deveria ter uma coluna `fooId` ?

A principio, ambas opções são válidas ao estabelecer um relacionamento de Um-Para-Um entre Foo e Bar. Todavia, quando dizemos algo como *"há um relacionamento de Um-Para-Um entre Foo e Bar"*, não fica claro se o relacionamento é obrigatório ou opcional. Em outras palavras,  Foo pode existir sem Bar? Do mesmo modo, Bar pode existir sem Foo? As respostas para essas perguntas ajudam a definir onde queremos que a chave estrangeira seja aplicada.

### Objetivo

Para o restante desse exemplo, vamos supor que temos dois modelos, `Foo` e `Bar`. Queremos estabelecer um relacionamento de Um-Para-Um entre elas, de uma forma que `Bar` tenha uma coluna `fooId`.

### Implementação

A estrutura principal para atingir esse objetivo é a seguinte:

```js
Foo.hasOne(Bar);
Bar.belongsTo(Foo);
```

Já que não foi passado nenhuma opção como segundo parâmetro, Sequelize irá deduzir o que fazer a partir dos nomes dos modelos. Nesse caso, Sequelize sabe que uma coluna `fooId` precisa ser adicionada a `Bar`.

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

Por padrão, a associação é considerada opcional. Em outras palavras, no nosso exemplo, a coluna `fooId` pode ser nula, significando que Bar pode existir sem Foo. Para alterar isso basta definir `allowNull: false` nas opções de chave estrangeira:

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

As associações de Um-Para-Muitos são conectar uma origem com muitos alvos, enquanto todos
esses alvos são conectados apenas com essa origem.

Isso significa que, diferente da associação de Um-Para-Um, na qual tivemos que escolher onde a chave estrangeira seria colocada, há apenas uma opção no relacionamento de Um-Para-Muitos. Por exemplo, se um Foo tem muitos Bars (e dessa forma cada Bar pertence a um Foo), então a unica implementação sensata é ter uma coluna `fooId` na tabela `Bar`. O oposto é impossível , já que um Foo tem muitos Bars.

### Objetivo

Nesse exemplo, nós temos os modelos `Time` e `Jogador`. Queremos dizer ao Sequelize que existe um relacionamento de Um-Para-Muitos entre elas, significando que um Time tem muitos Jogadores, enquanto cada Jogador pertence a um único Time.

### Implementação

A principal forma de fazer isso é a seguinte:

```js
Time.hasMany(Jogador);
Jogador.belongsTo(Time);
```

De novo, como mencionado, a principal forma de fazer isso usou um par de associações do Sequelize (`hasMany` e `belongsTo`).

Por exemplo, no PostgreSQL, a configuração acima produzirá o seguinte SQL em `sync()`:

```sql
CREATE TABLE IF NOT EXISTS "Times" (
  /* ... */
);
CREATE TABLE IF NOT EXISTS "Jogadores" (
  /* ... */
  "TimeId" INTEGER REFERENCES "Times" ("id") ON DELETE SET NULL ON UPDATE CASCADE,
  /* ... */
);
```

### Opções

As opções a serem aplicadas nesse caso são as mesmas do relacionamento de Um-Para-Um. Por exemplo, para alterar o nome da chave estrangeira e ter certeza de que o relacionamento é obrigatório, podemos fazer:

```js
Time.hasMany(Jogador, {
  foreignKey: 'clubeId'
});
Jogador.belongsTo(Time);
```

Igual os relacionamentos de Um-Para-Um, `ON DELETE` é padronizado para `SET NULL` e `ON UPDATE` é padronizado para `CASCADE`.

## Relacionamentos de Muitos-Para-Muitos

### Filosofia

Associações de Muitos-Para-Muitos conectam uma origem com muitos alvos, enquanto todos esses alvos podem, por sua vez, serem conectados a outras fontes além da primeira.

Esse tipo de relacionamento não pode ser representado adicionando uma chave estrangeira em uma das tabelas, como os outros relacionamentos fizeram. Em vez disso, o conceito de um [Modelo de junção](https://en.wikipedia.org/wiki/Associative_entity) [é usado]. Isso será uma modelo extra (e uma tabela extra no banco de dados) que terá duas colunas de chaves estrangeiras e irá rastrear as associações. A tabela de junção também pode ser chamada de  *join table* ou *through table* ou *tabela pivô*.

### Objetivo

Para esse exemplo, vamos considerar os modelos `Filme` e `Ator`. Um ator participou de vários filmes, e um filme teve muitos atores envolvidos na produção. A tabela de junção que irá cuidar das associações será chamada de `AtorFilmes`, que irá conter as chaves estrangeiras `filmeId` e `atorId`.

### Implementação

A principal forma de fazer isso no Sequelize é a seguinte:

```js
const Filme = sequelize.define('Filme', { nome: DataTypes.STRING });
const Ator = sequelize.define('Ator', { name: DataTypes.STRING });
Filme.belongsToMany(Ator, { through: 'AtorFilmes' });
Ator.belongsToMany(Filme, { through: 'AtorFilmes' });
```

Já que uma string foi passada para a opção `through` da chamada de `belongsToMany`, Sequelize irá criar automaticamente o modelo `AtorFilmes` que irá atuar como modelo de junção. Por exemplo, no PostgreSQL:

```sql
CREATE TABLE IF NOT EXISTS "AtorFilmes" (
  "createdAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updatedAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "FilmeId" INTEGER REFERENCES "Filmes" ("id") ON DELETE CASCADE ON UPDATE CASCADE,
  "AtorId" INTEGER REFERENCES "Ators" ("id") ON DELETE CASCADE ON UPDATE CASCADE,
  PRIMARY KEY ("FilmeId","AtorId")
);
```

No lugar de uma string, também é suportado um modelo, e nesse caso o modelo dado será usado como modelo de junção (e nenhum modelo será criado automaticamente). Por exemplo:

```js
const Filme = sequelize.define('Filme', { name: DataTypes.STRING });
const Ator = sequelize.define('Ator', { name: DataTypes.STRING });
const AtorFilmes = sequelize.define('AtorFilmes', {
  FilmeId: {
    type: DataTypes.INTEGER,
    references: {
      model: Filme, // 'Filmes' também funcionaria
      key: 'id'
    }
  },
  AtorId: {
    type: DataTypes.INTEGER,
    references: {
      model: Ator, // 'Ators' também funcionaria
      key: 'id'
    }
  }
});
Filme.belongsToMany(Ator, { through: 'AtorFilmes' });
Ator.belongsToMany(Filme, { through: 'AtorFilmes' });
```

O exemplo mostrado acima irá produzir o seguinte SQL, que é correspondente ao código escrito:

```sql
CREATE TABLE IF NOT EXISTS "AtorFilmes" (
  "FilmeId" INTEGER NOT NULL REFERENCES "Filmes" ("id") ON DELETE RESTRICT ON UPDATE CASCADE,
  "AtorId" INTEGER NOT NULL REFERENCES "Ators" ("id") ON DELETE RESTRICT ON UPDATE CASCADE,
  "createdAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updatedAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  UNIQUE ("FilmeId", "AtorId"),     -- Note: Sequelize inseriu a especificação UNIQUE
  PRIMARY KEY ("FilmeId","AtorId")  -- isso é irrelevante ja que também é PRIMARY KEY
);
```

### Opções

Diferente do relacionamento de Um-Para-Um e Um-Para-Muitos, o padrão de `ON UPDATE` e `ON DELETE` é `CASCADE` para relacionamentos de Muitos-Para-Muitos.

BelongsToMany cria uma chave única no modelo de junção. O nome dessa chave única pode ser sobscrito usando a opção **uniqueKey**. Para omitir a criação dessa chave unica, use a opção **unique: false**.

```js
Projeto.belongsToMany(Usuario, { through: UsuarioProjetos, uniqueKey: 'minha_unique_customizada' })
```

## Consultas básicas envolvendo associações

Com os conceitos básicos de definições de associações já vistos, podemos prosseguir para as consultas envolvendo associações. As consultas mais comuns nesse assunto são as do tipo *read* (ou seja, os SELECTs). Posteriormente, outros tipos de consultas serão apresentados.

Para estudo, vamos considerar um exemplo em que temos Navios e Capitães, e uma relacionamento de Um-Para-Um entre eles. Irémos permitir valores nulos nas chaves estrangeiras (o padrão), significando que um Navio pode existir sem um Capitão e vice-versa.

```js
// Essa é a configuração das nossas tabelas para os exemplos abaixo
const Navio = sequelize.define('navio', {
  nome: DataTypes.TEXT,
  capacidadeTripulacao: DataTypes.INTEGER,
  quantidadeVelas: DataTypes.INTEGER
}, { timestamps: false });
const Capitao = sequelize.define('capitao', {
  nome: DataTypes.TEXT,
  nivelHabilidade: {
    type: DataTypes.INTEGER,
    validate: { min: 1, max: 10 }
  }
}, { timestamps: false });
Capitao.hasOne(Navio);
Navio.belongsTo(Capitao);
```

### Buscando associações - Eager Loading vs Lazy Loading

Os conceitos de Eager Loading e Lazy Loading são fundamentais para entender como a busca por associações funciona no Sequelize. Lazy Loading refere-se a tecnica de buscar os dados associados somente quando você realmente precisa deles; Eager Loading, por outro lado, refere-se a tecnica de buscar por tudo de uma vez desde o começo, com uma consulta maior.

#### Exemplo de Lazy Loading 

```js
const capitaoBiroliro = await Capitao.findOne({
  where: {
    nome: "Baldonaro"
  }
});
// Fazer coisas com o capitão buscado
console.log('Nome:', capitaoBiroliro.nome);
console.log('Nível de habilidade:', capitaoBiroliro.nivelHabilidade);
// Agora queremos informações sobre o navio dele
const navioDele = await capitaoBiroliro.getNavio();
// Fazer coisas com o navio
console.log('Nome do navio:', navioDele.nome);
console.log('Quantidade de velas:', navioDele.quantidadeVelas);
```

Observe que no exemplo acima fizemos duas buscas, buscando o navio associado apenas quando queriamos usa-lo. Isso pode ser muito útil se precisarmos ou não do navio, talvez possamos querer busca-lo apenas em alguns casos; Dessa forma podemos economizar tempo e memória buscando apenas quando necessário.

Nota: o método `getNavio()` usado acima é um dos métodos que o Sequelize adiciona automaticamente às instancias do `Capitao`. Há outros. Você aprenderá mais sobre eles posteriormente neste guia

#### Example de Eager Loading 

```js
const capitaoBiroliro = await Capitao.findOne({
  where: {
    nome: "Baldonaro"
  },
  include: Navio
});
// Agora o Navio vem junto
console.log('Nome:', capitaoBiroliro.nome);
console.log('Nível de habilidade:', capitaoBiroliro.nivelHabilidade);
console.log('Nome do navio:', capitaoBiroliro.navio.nome);
console.log('Quantidade de velas:', capitaoBiroliro.navio.quantidadeVelas);
```

Como mostrado acima, Eager Loading é feito no Sequelize usando a opção `include`. Observe que aqui apenas uma busca foi realizada no banco de dados (que traz os dados associados junto com a istância).

Essa foi apenas uma introdução rapida ao Eager Loading no Sequelize. Há muito mais, que você pode aprender em [Guia dedicado ao Eager Loading](eager-loading.html).

### Criando, atualizando e deletando

Os exemplos acima mostraram o básico de busca de dados envolvendo associações. Para criar, atualizar e deletar, você também pode:

* Use as consultas padrões de modelo diretamente:

  ```js
  // Exemplo: criando um modelo associado usando os métodos padrões
  Bar.create({
    name: 'Minha Bar',
    fooId: 5
  });
  // Isso cria uma Bar pertencendo ao Foo de ID 5 (desde que fooId seja uma coluna regular). Nada muito difícil acontecendo aqui
  ```

* Ou use os *[Métodos especíais/mixins](#special-methods-mixins-added-to-instances)* disponíveis para modelos associadas, que são explicados posteriormente nessa página.

**Nota:** O [método da instancia `save()`](../class/lib/model.js~Model.html#instance-method-save) não tem conhecimento das associações. Em outras palavras, se você alterar o valor de um objeto *filho* que foi carregado pela técnica Eager junto com o objeto *pai*, chamar a função `save()` no pai irá ignorar completamente a mudança que ocorreu no filho.

## Apelidos para associação & Chaves Estrangeiras personalizadas

Em todos os exemplos acima, Sequelize definiu automaticamente o nome das chaves estrangeiras. Por exemplo, no exemplo do Capitão e o Navio, o Sequelize automaticamente  definiu um campo `captainId` no modelo do Navio. Entretando, é facil especificar uma chave estrangeira personalizada.

Vamos considerar os modelos Navio e Capitao em uma forma mais simplificada, apenas para focar no tópico atual, como mostrado abaixo (menos campos):

```js
const Navio = sequelize.define('navio', { nome: DataTypes.TEXT }, { timestamps: false });
const Capitao = sequelize.define('capitao', { nome: DataTypes.TEXT }, { timestamps: false });
```

Há três formas de especificar um nome diferente para a chave estrangeira:

* Especificando o nome da chave estrangeira diretamente.
* Definindo um apelido (alias)
* Fazendo ambas as alternativas

### Recapitulando: a configuração padrão

Usando apenas `Navio.belongsTo(Capitao)`, Sequelize irá gerar o nome da chave estrangeira automaticamente:

```js
Navio.belongsTo(Capitao); // Isso cria a chave estrangeira 'capitaoId' no Navio

// Eager Loading é feito passando o modelo à opção 'include'
console.log((await Navio.findAll({ include: Capitao })).toJSON());
// Ou especificando o nome do modelo associado
console.log((await Navio.findAll({ include: 'captain' })).toJSON());

// Também, as instancias ganham o método `getCapitao()` para Lazy Loading:
const ship = Navio.findOne();
console.log((await ship.getCapitao()).toJSON());
```

### Especificando o nome da chave estrangeira diretamente

O nome da chave estrangeira pode ser especificado diretamente com uma opção na definição da associação, como abaixo:

```js
Navio.belongsTo(Capitao, { foreignKey: 'chefeId' }); // Isso cria a chave estrangeira  `chefeId` no Navio.

// Eager Loading é feito passando o modelo à opção 'include'
console.log((await Navio.findAll({ include: Capitao })).toJSON());
// Ou especificando o nome do modelo associado
console.log((await Navio.findAll({ include: 'Capitao' })).toJSON());

// Também, as instancias ganham o método `getCapitao()` para Lazy Loading:
const navio = Navio.findOne();
console.log((await navio.getCapitao()).toJSON());
```

### Definindo um apelido

Definir um apelido é mais poderoso do que simplesmente especificar um nome personalizado para a chave estrangeira. Isso é melhor entendido no exemplo:

<!-- NOTE: any change in this part might also require a change on advanced-many-to-many.md -->

```js
Navio.belongsTo(Capitao, { as: 'lider' }); // Isso cria a chave estrangeira 'liderId' no Navio

// Eager Loading não funciona mais passando o modelo ao include
console.log((await Navio.findAll({ include: Capitao })).toJSON()); // Gera um erro
// Agora, você deve passar o apelido:
console.log((await Navio.findAll({ include: 'lider' })).toJSON());
// Ou você pode passar um objeto com o modelo e o apelido:
console.log((await Navio.findAll({
  include: {
    model: Capitao,
    as: 'lider'
  }
})).toJSON());

// Também, as intâncias ganham o método `getLider()` para Lazy Loading:
const navio = Navio.findOne();
console.log((await navio.getLider()).toJSON());
```

Apelidos são especialmente úteis quando você precisa definir duas associaçoes diferentes entre os mesmos modelos. Por exemplo, Se tivermos os modelos `Mail` e `Pessoa`, podemos querer associá-los duas vezes, para representar o `remetente` e `destinatário` do e-Mail. Nesse caso, precisamos usar um apelido para cada associação, caso contrário, uma chamada como `mail.getPessoa()` seria ambígua. Com os apelidos `remetente` e `destinatario`, nós teriamos dois métodos disponíveis e funcionando: `mail.getRemetente()` e `mail.getDestinatario()`, ambos retornando uma `Promise<Pessoa>`.

Ao definir um apelido para uma associação `hasOne` ou `belongsTo`, você deve usar a forma singular de uma palavra (como `lider`, no exemplo acima). Por outro lado, ao definir um apelido para `hasMany` e` belongsToMany`, você deve usar a forma plural. A definição de apelidos para relacionamentos de Muitos-Para-Muitos (com `belongsToMany`) é abordada no [Guia avançado de associações de Muitos-Para-Muitos](advanced-many-to-many.html).

### Fazendo ambas as coisas

Podemos definir um apelido e também definir uma chave estrangeira diretamente:

```js
Navio.belongsTo(Capitao, { as: 'lider', foreignKey: 'chefeId' }); // Isso cria a chave estrangeira 'chefeId' no Navio

// Agora que um apelido foi definido, Eager Loading não funciona simplesmente passando o modelo no 'include'
console.log((await Navio.findAll({ include: Capitao })).toJSON()); // Gera um erro
// Agora, você deve passar o apelido:
console.log((await Navio.findAll({ include: 'lider' })).toJSON());
// Ou você pode passar um objeto definindo o modelo e o apelido
console.log((await Navio.findAll({
  include: {
    model: Capitao,
    as: 'lider'
  }
})).toJSON());

// Também, as intâncias ganham o método `getLeader()` para Lazy Loading:
const navio = Navio.findOne();
console.log((await navio.getLider()).toJSON());
```

## Métodos especiais/mixins adicionados as instâncias

Quando uma associação é definida entre dois modelos, as instâncias desses modelos ganham métodos especiais para interagir com seus correspondentes.

Por exemplo, se temos dois modelos, `Foo` e `Bar`, e eles estão associados, suas instancias  irão ter os seguintes métodos/mixins disponíveis, dependendo do tipo da associação:

### `Foo.hasOne(Bar)`

* `fooInstance.getBar()`
* `fooInstance.setBar()`
* `fooInstance.createBar()`

Exemplo:

```js
const foo = await Foo.create({ name: 'o foo' });
const bar1 = await Bar.create({ name: 'alguma-bar' });
const bar2 = await Bar.create({ name: 'outra-bar' });
console.log(await foo.getBar()); // null
await foo.setBar(bar1);
console.log((await foo.getBar()).name); // 'alguma-bar'
await foo.createBar({ name: 'ainda-outra-bar' });
const newlyAssociatedBar = await foo.getBar();
console.log(newlyAssociatedBar.name); // 'ainda-outra-bar'
await foo.setBar(null); // Desassociar
console.log(await foo.getBar()); // null
```

### `Foo.belongsTo(Bar)`

Os mesmos de `Foo.hasOne(Bar)`:

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

Exemplo:

```js
const foo = await Foo.create({ name: 'o-foo' });
const bar1 = await Bar.create({ name: 'alguma-bar' });
const bar2 = await Bar.create({ name: 'outra-bar' });
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
await foo.createBar({ name: 'ainda-outra-bar' });
console.log(await foo.countBars()); // 2
await foo.setBars([]); // Desassociar todas as Bars associadas anteriormente
console.log(await foo.countBars()); // 0
```

O método get aceita opções da mesma forma que os otros métodos de busca (como o `findAll`):

```js
const tarefasFaceis = await projeto.getTarefas({
  where: {
    dificuldade: {
      [Op.lte]: 5
    }
  }
});
const tituloTarefas = (await projeto.getTarefas({
  attributes: ['titulo'],
  raw: true
})).map(tarefa => tarefa.titulo);
```

### `Foo.belongsToMany(Bar, { through: Baz })`

Os mesmos de `Foo.hasMany(Bar)`:

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

### Note: Nomes dos métodos

Como apresentado no exemplo acima, os nomes que o Sequelize da para esses métodos especiais  são formados por um prefixo (ex: `get`, `add`, `set`) concatenado com o nome do modelo (com a primeira letra maiúscula). Quando necessário, o plural é usado, como em `fooInstance.setBars()`. Repetindo, plurais irregulares também são tratados automaticamente pelo Sequelize. Por exemplo, `Person` se torna `People` e `Hypothesis` se torna `Hypotheses`. O tratamento de plurais irregulares são aplicados apenas às palavras em inglês.

Se um apelido for definido, ele será usado no lugar do nome do modelo para formar os nomes dos métodos. Por exemplo:

```js
Tarefa.hasOne(Usuario, { as: 'Autor' });
```

* `instanciaTarefa.getAutor()`
* `instanciaTarefa.setAutor()`
* `instanciaTarefa.createAutor()`

## Por que as associações são definidas em pares?

Como mencionado anteriormente e mostrado em mais exemplos acima, as associações no Sequelize são normalmente definidas em pares:

* Para criar um relacionamento de **Um-Para-Um**, as associações `hasOne` e `belongsTo` são usadas juntas;
* Para criar um relacionamento de **Um-Para-Muitos**, as associações `hasMany` e `belongsTo` são usadas juntas;
* Para criar um relacionamento de **Muitos-Para-Muitos**, duas chamadas à `belongsToMany` são usadas juntas.

Quando uma associação no Sequelize é definida entre dois modelos, apenas o modelo *origem* *tem conhecimento sobre tal associação*. Então, por exemplo, ao usar `Foo.hasOne(Bar)` (então `Foo` é o modelo origem e `Bar` é o modelo alvo), apenas `Foo` sabe sobre a existência dessa associação. É por isso que nesse caso, como mostrado acima, as instâncias de `Foo` ganham os métodos `getBar()`, `setBar()` e `createBar()`, enquanto por outro lado as instâncias de `Bar` não ganham nada.

Similarmente, para `Foo.hasOne(Bar)`, desde que `Foo` saiba sobre o relacionamento, podemos realizar Eager Loading como em: `Foo.findOne({ include: Bar })`, porém não podemos fazer `Bar.findOne({ include: Foo })`.

Portanto, para trazer todo o poder ao uso do Sequelize, geralmente configuramos os relacionamentos em pares, pois então ambas as tabelas ganham o conhecimento sobre o relacionamento existente entre elas.

Demonstração prática:

* Se não definirmos o par de associações, chamando apenas `Foo.hasOne(Bar)`, por exemplo:

  ```js
  // Isso funciona...
  await Foo.findOne({ include: Bar });

  // Porém isso gera um erro:
  await Bar.findOne({ include: Foo });
  // SequelizeEagerLoadingError: foo is not associated to bar!
  ```

* Se definirmos o par de associações como recomendado, ou seja, ambas as associações `Foo.hasOne(Bar)` e `Bar.belongsTo(Foo)`:

  ```js
  // Isso funciona!
  await Foo.findOne({ include: Bar });

  // Isso também funciona!
  await Bar.findOne({ include: Foo });
  ```

## Associações multiplas envolvendo os mesmos modelos

No sequelize, é possível definir multiplas associações entres as mesmas tabelas. Você só precisa definir apelidos diferentes para cada associação:

```js
Time.hasOne(Jogo, { as: 'EmCasa', foreignKey: 'emCasaId' });
Time.hasOne(Jogo, { as: 'ForaDeCasa', foreignKey: 'foraDeCasaId' });
Jogo.belongsTo(Time);
```

## Criando associações referenciando um campo que não é a chave primaria

Em todos os exemplos acima, as associações foram definidas referênciando as chaves primárias dos modelos envolvidos (no nosso caso, seus IDs). No entanto, o Sequelize permite definir uma associação que usa outro campo, que não seja a chave primária, para estabelecer a associação

Esse outro campo deve unico, em outras palavras, a coluna deve recever a restrição **unique**, caso contrário não faria sentido.

### Para os relacionamentos `belongsTo`

Primeiro, lembre-se de que a associação `A.belongsTo(B)` coloca coloca a chave estrangeira no *modelo origem* (isto é, no `A`).

Vamos novamente usar o exemplo de navios e capitães. Dessa vez, assumiremos que os nomes dos capitães são unicos:

```js
const Navio = sequelize.define('navio', { name: DataTypes.TEXT }, { timestamps: false });
const Capitao = sequelize.define('capitao', {
  nome: { type: DataTypes.TEXT, unique: true }
}, { timestamps: false });
```

Dessa forma, em vez de manter `capitaoId` nos nossos Navios, podemos manter `capitaoNome` e usa-lo como nosso rastreador de associações. Em outras palavras, em vez de referenciar o `id` do modelo alvo (Capitao), nosso relacionamento irá referenciar outra coluna no modelo alvo: a coluna `nome`. Para especificar isso, temos que definir uma *targetKey*. Também devemos especificar um nome para a própria chave estrangeira.

```js
Navio.belongsTo(Capitao, { targetKey: 'nome', foreignKey: 'capitaoNome' });
// Isso cria uma chave estrangeira chamada 'capitaoNome' no modelo origem (Navio)
// que referencia o campo `nome` do modelo alvo (Capitao).
```

Agora podemos fazer coisas como:

```js
await Capitao.create({ nome: "Biroliro" });
const navio = await Navio.create({ nome: "De papel", capitaoNome: "Biroliro" });
console.log((await navio.getCapitao()).nome); // "Biroliro"
```

### Para os relacionamentos `hasOne` e `hasMany`

A exata mesma ideia pode ser aplicada as associações `hasOne` e `hasMany`, mas ao invés de especificar uma `targetKey`, especificamos `sourceKey` ao definir a associação. Isso porque diferente de `belongsTo`, as associações `hasOne` e `hasMany` mantem as chaves estrangeiras no modelo:

```js
const Foo = sequelize.define('foo', {
  nome: { type: DataTypes.TEXT, unique: true }
}, { timestamps: false });
const Bar = sequelize.define('bar', {
  titulo: { type: DataTypes.TEXT, unique: true }
}, { timestamps: false });
const Baz = sequelize.define('baz', { sumario: DataTypes.TEXT }, { timestamps: false });
Foo.hasOne(Bar, { sourceKey: 'nome', foreignKey: 'fooNome' });
Bar.hasMany(Baz, { sourceKey: 'titulo', foreignKey: 'barTitulo' });
// [...]
await Bar.setFoo("Nome do Foo aqui");
await Baz.addBar("Titulo do Bar aqui");
```

### Para os relacionamentos `belongsToMany`

A mesma ideia também pode ser aplicada aos relacionamentos `belongsToMany`. Porém, diferente das outras situações, na qual tivemos apenas uma chave estrangeira envolvida, o relacionamento `belongsToMany` envolve duas chaves estrangeiras que são mantidas em uma tabela extra(a tabela de junção)

Considere a seguinte configuração:

```js
const Foo = sequelize.define('foo', {
  nome: { type: DataTypes.TEXT, unique: true }
}, { timestamps: false });
const Bar = sequelize.define('bar', {
  titulo: { type: DataTypes.TEXT, unique: true }
}, { timestamps: false });
```

Há quatro casos a serem considerados:

* Podemos querer um relacionamento de Muitos-Para-Muitos usando as chaves primarias padrões para ambos `Foo` e `Bar`:

```js
Foo.belongsToMany(Bar, { through: 'foo_bar' });
// Isso cria uma tabela de junção `foo_bar` com campos `fooId` e `barId`
```

* Podemos querer um relacionamento de Muitos-Para-Muitos usando a chave primária padrão para `Foo` mas um campo diferente para `Bar`:

```js
Foo.belongsToMany(Bar, { through: 'foo_bar', targetKey: 'titulo' });
// Isso cria uma tabela de junção `foo_bar` com campos `fooId` e `barTitulo`
```

* Podemos querer um relacionamento de Muitos-Para-Muitos usando um campo diferente para `Foo` e a chave primária padrão para `Bar`:

```js
Foo.belongsToMany(Bar, { through: 'foo_bar', sourceKey: 'nome' });
// Isso cria uma tabela de junção `foo_bar` com campos `fooNome` e `barId`
```

* Podemos querer um relacionamento de Muitos-Para-Muitos usando campos diferentes para ambos `Foo` e `Bar`:

```js
Foo.belongsToMany(Bar, { through: 'foo_bar', sourceKey: 'nome', targetKey: 'titulo' });
// Isso cria uma tabela de junção `foo_bar` com campos `fooNome` e `barTitulo`
```

### Notas

Não se esqueça que o campo referenciado na associação precisa ser unico, ou seja, deve conter a restrição **unique**. Do contrário, um erro será gerado (e as vezes com uma mensagem de erro misteriosa - como `SequelizeDatabaseError: SQLITE_ERROR: foreign key mismatch - "foos" referencing "bars"` para SQLite).

O truque para decidir entre `sourceKey` e `targetKey` é apenas lembrar onde cada relacionamento coloca sua chave estrangeira. Como mencionado no início desse guia:

* `A.belongsTo(B)` mantém a chave estrangeira no modelo origem (`A`), portanto a chave referenciada está no modelo alvo, consequentemente o uso de `targetKey`.

* `A.hasOne(B)` e `A.hasMany(B)` mantém a chave estrangeira no modelo alvo (`B`), portanto a chave referenciada está no modelo origem, consequentemente o uso de `sourceKey`.

* `A.belongsToMany(B)` envolve um modelo extra (a tabela de junção), portnato ambos  `sourceKey` e `targetKey` são utilizáveis, com `sourceKey` correspondendo á algum campo em  `A` (a origem) e `targetKey` correspondendo á algum campo em `B` (o alvo).
