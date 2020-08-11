# Consulta de Modelo - Localizadores

Metodos localizadores são aqueles que geram consultas do tipo `SELECT`.

Por padrão, os resultados de todos os métodos localizadores são instâncias da classe modelo (ao invés de serem apenas objetos JavaScript simples). Isso significa que após
a base de dados retornar os resultados, o Sequelize, automaticamente, realiza a junção de tudo em objetos de instância apropriados. em alguns casos, quando existem muitos resultados, a junção pode ser ineficiente. Para receber uma resposta simplifica e desabilitar essa funcão, é necessário passar `{ raw: true }` como uma opção para o método
de busca.

## `findAll`

O método `findAll` já é conhecido do tutorial anterior. Ele gera uma consulta padrão `SELECT` que vai recuperar todas as entradas da tabela (a menos que que exista uma restrição, como por exemplo um `where`).

## `findByPk`

O método `findByPk` retorna somente uma única entrada da table, utilizando a chave primária (primary key) fornecida.

```js
const project = await Project.findByPk(123);
if (project === null) {
  console.log('Not found!');
} else {
  console.log(project instanceof Project); // true
  // A chave primária é 123
}
```

## `findOne`

O método `findOne` retorna o primeiro objeto encontrado (que preenche as informações opcionais de consulta, caso sejam fornecidas).

```js
const project = await Project.findOne({ where: { title: 'My Title' } });
if (project === null) {
  console.log('Not found!');
} else {
  console.log(project instanceof Project); // true
  console.log(project.title); // 'My Title'
}
```

## `findOrCreate`

O método `findOrCreate` irá criar uma entrada na tabela a menino que encontre uma entrada que preencha as opções de consulta. Em ambos os casos, o metódo retornará uma instância (mesmo se encontrada ou criada uma instância) e um valor booleano, indicando se a instância foi criada ou se já existia

A opção whe`where`re é considerada para encontrar uma entrada, e a opção `defaults` é utilizada para definir o que deve ser criado em caso de nada ser encontrado. Se a opção
`defaults` não possui um valor para toda coluna, o Sequelize irá utilizar os valores fornecidos ao `where` (caso exista).

Vamos assumir que nós temos uma base de dados vazia com uma model `User` que possui um `username` e um `job`.

```js
const [user, created] = await User.findOrCreate({
  where: { username: 'sdepold' },
  defaults: {
    job: 'Technical Lead JavaScript'
  }
});
console.log(user.username); // 'sdepold'
console.log(user.job); // Isso pode ser ou não 'Technical Lead JavaScript'
console.log(created); // O valor booleano indica se a instância foi criada
if (created) {
  console.log(user.job); // Isso certamente será 'Technical Lead JavaScript'
}
```

## `findAndCountAll`

O método `findAndCountAll` é um método de conveção que combina outros dois, `findAll` e `count`. Tal método é útil quando lidamos com consultas relacionadas a paginação
nas quais você deseja recuperar dados um `limit` e `offset`, mas também o total de de registros que satisfaçam a consulta.

O método `findAndCountAll` retorna um objeto com duas propriedades:

* `count` - um inteiro - número total de registros que satisfaçam a consulta
* `rows` - um array de objetos - registros obtidos

```js
const { count, rows } = await Project.findAndCountAll({
  where: {
    title: {
      [Op.like]: 'foo%'
    }
  },
  offset: 10,
  limit: 2
});
console.log(count);
console.log(rows);
```