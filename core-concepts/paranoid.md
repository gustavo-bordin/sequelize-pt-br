# Paranoid

Sequelize suporta o conceito de tabelas *paranoid*. Uma tabela *paranoid* é aquela que, quando dito para deletar um registro, ela não o deleta de verdade. Em vez disso, uma coluna especial chamada de `deletedAt` receberá o valor do momento em que a requisição da deleção foi feita.

Isso significa que tabelas *paranoid* executam uma *deleção suave* dos registros, ao invés de uma *deleção bruta*

## Definindo um modelo como paranoid

Para dizer que um modelo é paranoid, você deve passar a opção `paranoid: true` para a definição. As tabelas paranoid precisam que a opção 'timestamps' esteja como 'true' (ou seja, não irá funcionar se você definir `timestamps: false`).

Você também pode alterar o nome padrão da coluna (que é `deletedAt`) para algum outro.

```js
class Post extends Model {}
Post.init({ /* os atributos vão aqui */ }, {
  sequelize,
  paranoid: true,

  // Se você quiser dar outro nome à coluna 'deletedAt'
  deletedAt: 'momentoDestruido'
});
```

## Deletando

Quando você chama o método `destroy`, uma deleção-suave irá acontecer:

```js
await Post.destroy({
  where: {
    id: 1
  }
});
// UPDATE "posts" SET "deletedAt"=[timestamp] WHERE "deletedAt" IS NULL AND "id" = 1
```

Se você realmente quiser uma deleção-bruta e seu modelo for paranoid, você pode forçar com a opção `force: true`

```js
await Post.destroy({
  where: {
    id: 1
  },
  force: true
});
// DELETE FROM "posts" WHERE "id" = 1
```

Os exemplos acima usaram o método estático `destroy` , como (`Post.destroy`), porém, tudo funciona da mesma forma com as instâncias:

```js
const post = await Post.create({ titulo: 'teste' });
console.log(post instanceof Post); // true
await post.destroy(); // Apenas definiria a coluna 'deletedAt'
await post.destroy({ force: true }); // Apagaria o registro de verdade
```

## Restaurando

Para restaurar registros que sofreram uma deleção-suave, você pode usar o método `restore`, que funciona em ambas opções; no próprio modelo ou na instância do modelo:

```js
// Exemplo mostrando o uso de 'restore' na instância
// Criamos um post, executamos uma deleção-suave e então o restauramos
const post = await Post.create({ titulo: 'teste' });
console.log(post instanceof Post); // true
await post.destroy();
console.log('foi deletado suavemente');
await post.restore();
console.log('foi restaurado!');

// Exemplo mostrando o método estático de 'restore', ou seja, no próprio modelo
// Restaurando todos os registros suavemente deletados que tenham mais de 100 likes
await Post.restore({
  where: {
    likes: {
      [Op.gt]: 100
    }
  }
});
```

## Comportamento com outras queries

Toda query executada pelo Sequelize irá ignorar automaticamente os registros que foram deletados suavementes (exceto as queries cruas, é claro).

Isso significa que, por exemplo, o método `findAll` não irá ver os registros deletados suavementes, buscando apenas aqueles que não foram deletados.

Mesmo se você chamar `findByPk` passando a chave primária de um registro que foi deletado suavamente, o resultado será `null`, como se aquele registro não existisse.

Se você realmente quiser que as queries vejam os registros deletados suavemente, você pode passar a opção `paranoid: false` para o método da query. Por exemplo:

```js
await Post.findByPk(123); // Isso retornará 'null' se o registro de 123 foi deletado suavemente
await Post.findByPk(123, { paranoid: false }); // Isso retornará o registro

await Post.findAll({
  where: { foo: 'bar' }
}); // Isso não retornará registros deletados suavemente

await Post.findAll({
  where: { foo: 'bar' },
  paranoid: false
}); // Isso retornará registros deletados suavemente
```