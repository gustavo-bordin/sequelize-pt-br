# Raw Queries

Como há muitos casos de uso em que é mais facil executar uma query bruta / queries já preparadas, você pode usar o método [`sequelize.query`](../class/lib/sequelize.js~Sequelize.html#instance-method-query).

Por padrão a função retornará dois argumentos - um array de resultados e um objeto contendo metadados (como a quantidade de linhas afetadas, etc). Note que como se trata de uma query bruta, os metadados são específicos para cada dialeto. Alguns dialetos retornam os metadados dentro do objeto de resultado (como propriedados em um array). Contudo, dois argumentos sempre serão retornados, porém para MSSQL e MySQL será duas referências ao mesmo objeto.

```js
const [results, metadata] = await sequelize.query("UPDATE users SET y = 42 WHERE x = 12");
// O resultado será um array vazio e o metadado irá conter o numero de linhas afetadas.
```

Em casos que você não precise dos metadados, você pode dizer na query como o Sequelize deve formatar os resultados. Por exemplo, para uma simples query de SELECT, você pode fazer:

```js
const { QueryTypes } = require('sequelize');
const users = await sequelize.query("SELECT * FROM `users`", { type: QueryTypes.SELECT });
// Não precisamos desestruturar os resultados aqui - Os resultados serão retornados diretamente
```

Muitas outras queries estão disponíveis. [Veja no código para detalhes](https://github.com/sequelize/sequelize/blob/master/lib/query-types.js).

Uma segunda opção é o model. Se você definir um, os dados retornados serão instâncias desse model.

```js
// A definição do model abaixo é chamada de Calle. Isso permite mapear facilmente a query para um model predefinido.
const projects = await sequelize.query('SELECT * FROM projects', {
  model: Projects,
  mapToModel: true // pass true here if you have any mapped fields
});
// Agora cada elemento de 'projects' será uma instância de Projects
```

Veja mais opções em [Referência da API da query](../class/lib/sequelize.js~Sequelize.html#instance-method-query). Alguns exemplos:

```js
const { QueryTypes } = require('sequelize');
await sequelize.query('SELECT 1', {
  // Uma função (ou falso) para mostrar logs de sua query
  // Será chamada para todas as queries SQL enviadas ao servidor
  logging: console.log,

  // Se plain for true, então o sequelize retornará apenas o primeiro
  // registro do conjunto de registros. Se for falso, retornará todos
  plain: false,

  // Deixe isso como true se você não tiver uma definição de model para sua query.
  raw: false,

  // O tipo de query que você está executando. O tipo da query altera a forma em que o resultado é formatado antes de ser retornado.
  type: QueryTypes.SELECT
});

// Note o segundo argumento sendo 'null'!
// Mesmo se você declarar um callee aqui (definir um model), a opção 'raw: true' 
// iria sobrescrever e retornar um objeto bruto
console.log(await sequelize.query('SELECT * FROM projects', { raw: true }));
```

## Atributos "Dotted" e a opção `nest`

Se o nome de um atributo na tabela conter pontos, os objetos resultantes podem se tornar objetos aninhados definindo a opção `nest: true`. Isso é feito com [dottie.js](https://github.com/mickhansen/dottie.js/) por baixo dos panos. Veja abaixo:

* Sem `nest: true`:

  ```js
  const { QueryTypes } = require('sequelize');
  const records = await sequelize.query('select 1 as `foo.bar.baz`', {
    type: QueryTypes.SELECT
  });
  console.log(JSON.stringify(records[0], null, 2));
  ```

  ```json
  {
    "foo.bar.baz": 1
  }
  ```

* Com `nest: true`:

  ```js
  const { QueryTypes } = require('sequelize');
  const records = await sequelize.query('select 1 as `foo.bar.baz`', {
    nest: true,
    type: QueryTypes.SELECT
  });
  console.log(JSON.stringify(records[0], null, 2));
  ```

  ```json
  {
    "foo": {
      "bar": {
        "baz": 1
      }
    }
  }
  ```

## Substituições

Substituições na query pode ser feitas através de duas formas, usando parâmetros nomeados  (começando com `:`), ou não nomeados, representado por um `?`. Substituições são passadas no objeto de opções

* Se um array for passado, `?` será substituido pelo elemento de ordem correspondente.
* Se um objeto for passado, `:key` será substituido pelas chaves desse objeto. Se o objeto conter chaves que não existe na query, ou vice-versa, um erro será gerado.

```js
const { QueryTypes } = require('sequelize');

await sequelize.query(
  'SELECT * FROM projects WHERE status = ?',
  {
    replacements: ['active'],
    type: QueryTypes.SELECT
  }
);

await sequelize.query(
  'SELECT * FROM projects WHERE status = :status',
  {
    replacements: { status: 'active' },
    type: QueryTypes.SELECT
  }
);
```

As substituições de array serão tratadas automaticamente, a query a seguir procura em que o status corresponde com um array de valores:

```js
const { QueryTypes } = require('sequelize');

await sequelize.query(
  'SELECT * FROM projects WHERE status IN(:status)',
  {
    replacements: { status: ['active', 'inactive'] },
    type: QueryTypes.SELECT
  }
);
```

Para usar o operador curinga `%`, anexe-o ao seu substituto. A query a seguir corresponde a usuários com nomes que começam com 'ben'.

```js
const { QueryTypes } = require('sequelize');

await sequelize.query(
  'SELECT * FROM users WHERE name LIKE :search_name',
  {
    replacements: { search_name: 'ben%' },
    type: QueryTypes.SELECT
  }
);
```

## Parâmetro Bind

Parâmetros Bind são como substituições. Porém as substituições são escapadas e inseridas na consulta pelo Sequelize antes ela seja enviada ao banco de dados, enquanto parâmetros bind são enviadas ao banco de dados fora da string do SQL. Uma query pode ter tanto parâmetros bind quanto substituições. Parâmetros bind são referenciados por $1, $2, ... (numérico) ou $key (alpha-numérico). Isso para todos os bancos de dados.

* Se um array for passado, `$1` é ligado ao primeiro elemento do array (`bind[0]`)
* Se um objeto for passado, `$key` é ligado ao `object['key']`. Cada chave deve começar com um caractere não numérico. `$1` não é uma chave válida, mesmo se `object['1']` existir.
* em ambos os casos `$$` pode ser usado para escapar o caractere `$`.

O array ou o objeto deve ter todos os valores vinculados, ou o Sequelize lançará uma exceção.
Isso se aplica até mesmo aos casos em que o banco de dados ignore o(s) parâmetro(s) vinculado(s).

O banco de dados pode adicionar outras restrições a isso. Os parâmetros bind não podem ser palavras reservadas do SQL, nem tabelas e colunas. Eles também são ignorados em textos ou dados citados. No PostgreSQL, também pode ser necessário fazer o typecast deles, se o tipo não puder ser inferido a partir do contexto `$ 1 :: varchar`.

```js
const { QueryTypes } = require('sequelize');

await sequelize.query(
  'SELECT *, "text with literal $$1 and literal $$status" as t FROM projects WHERE status = $1',
  {
    bind: ['active'],
    type: QueryTypes.SELECT
  }
);

await sequelize.query(
  'SELECT *, "text with literal $$1 and literal $$status" as t FROM projects WHERE status = $status',
  {
    bind: { status: 'active' },
    type: QueryTypes.SELECT
  }
);
```
