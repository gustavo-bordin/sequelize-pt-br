# Getters, Setters & Virtuals

Sequelize permite você definir getters e setters customizados para os atributos dos seus models.

Sequelize também permite você especificar os assim chamados *atributos virtuais*, no qual são os atributos no Sequelize Model que não realmente existe na tabela SQL subjacente, mas, ao invés, são populadas automaticamente pelo Sequelize. Eles são muito úteis para simplificar o código, por exemplo.

## Getters

Um getter é uma função `get()` definida para uma coluna na definição do model.

```js
const User = sequelize.define('user', {

  // Vamos dizer que nós queremos ver todo username em maiúsculo, embora
  // não sejam maiúsculos no próprio bando de dados
  username: {
    type: DataTypes.STRING,
    get() {
      const rawValue = this.getDataValue(username);
      return rawValue ? rawValue.toUpperCase() : null;
    }
  }
});
```

Esse getter, apenas como um getter padrão do Javascript, é chamado automaticamente quando o valor do campo é lido.

```js
const user = User.build({ username: 'SuperUser123' });
console.log(user.username); // 'SUPERUSER123'
console.log(user.getDataValue(username)); // 'SuperUser123'
```

Note que, embora `SUPERUSER123` foi registrado acima, o valor verdadeiro armazenado no banco de dados ainda é `SuperUser123`. Usamos `this.getDataValue(username)` para obter esse valor, e o convertemos para maiúsculas.

Se tivéssemos tentado user `this.username` no getter ao invés, teríamos um loop inifinito! Isso é porque Sequelize providencia o método `getDataValue`.

## Setters

Um setter é uma função `set()` definada para uma coluna na definição do model. Ele recebe um valor que está sendo definido:

```js
const User = sequelize.define('user', {
  username: DataTypes.STRING,
  password: {
    type: DataTypes.STRING,
    set(value) {

      // Armazenar senhas em texto puro no banco de dados é terrível.
      // Hashear o valor com uma apropriada função de hash criptográfica é melhor.
      this.setDataValue('password', hash(value));
    }
  }
});
```

```js
const user = User.build({ username: 'someone', password: 'NotSo§tr0ngP4$SW0RD!' });
console.log(user.password); // '7cfc84b8ea898bb72462e78b4643cfccd77e9f05678ec2ce78754147ba947acc'
console.log(user.getDataValue(password)); // '7cfc84b8ea898bb72462e78b4643cfccd77e9f05678ec2ce78754147ba947acc'
```

Observe que o Sequelize chamou o setter automaticamente, antes mesmo de enviar os dados para o banco de dados. O único dado que o banco de dados viu foi o valor já hasheado.

Se nós quisermos envolver um outro campo da nossa instância do model na computação, isso é possível e muito fácil!

```js
const User = sequelize.define('user', {
  username: DataTypes.STRING,
  password: {
    type: DataTypes.STRING,
    set(value) {

      // Armazenar senhas em texto puro no banco de dados é terrível.
      // Hashear o valor com uma apropriada função de hash criptográfica é melhor.
      // Usar o username como um salt é melhor.
      
      this.setDataValue('password', hash(this.username + value));
    }
  }
});
```

**Nota:** Os exemplos acima envolvendo gerenciamento de senhas, embora muito melhor do que simplesmente armazenar a senha em texto puro, estão longe de ser a segurança perfeita. Gerenciar senhas apropriadamente é difícil, tudo aqui é apenas para um exemplo para mostrar funcionalidades do Sequelize. Sugerimos a participação de um especialista em segurança cibernética e/ou ler os documentos de [OWASP](https://www.owasp.org/) e/ou visitar o [InfoSec StackExchange](https://security.stackexchange.com/).

## Combinando getters e setters

Getters e setters pode ser ambos definidos no mesmo campo.

Para um exemplo, vamos dizer que nós estamos modelando um `Post`, cujo `content` é um texto de tamanho ilimitado. Para melhorar o uso de memória, vamos dizer que nós queremos armazenar uma versão gzipped do conteúdo.

*Nota: Banco de dados modernos devem fazer alguma compressão automaticamente nesses casos. Por favor, note que isso é apenas para um exemplo.*

```js
const { gzipSync, gunzipSync } = require('zlib');

const Post = sequelize.define('post', {
  content: {
    type: DataTypes.TEXT,
    get() {
      const storedValue = this.getDataValue('content');
      const gzippedBuffer = Buffer.from(storedValue, 'base64');
      const unzippedBuffer = gunzipSync(gzippedBuffer);
      return unzippedBuffer.toString();
    },
    set(value) {
      const gzippedBuffer = gzipSync(value);
      this.setDataValue('content', gzippedBuffer.toString('base64'));
    }
  }
});
```

Com a configuração acima, sempre que nós tentarmos interagir com o campo `content` do nosso model `Post`, Sequelize vai automaticamente gerenciar o getter e setter personalizados. Por exemplo:

```js
const post = await Post.create({ content: 'Hello everyone!' });

console.log(post.content); // 'Hello everyone!'

// Tudo está acontecendo embaixo do capô, então nós podemos até esquecer que o
// content está na verdade sendo armazenado como um gzipped base64 string!

// No entanto, se nós somos realmente curiosos, podemos pegar o dado 'cru'...
console.log(post.getDataValue('content'));
// Saída: 'H4sIAAAAAAAACvNIzcnJV0gtSy2qzM9LVQQAUuk9jQ8AAAA='
```

## Campos virtuais

Campos virtuais são campos que o Sequelize preenche embaixo do capô, mas na
realidade eles nem mesmo existem no banco de dados.

Por exemplo, vamos dizer que nós temos os atributos `firstName` e `lastName` para um User.

*De novo, isso é [somente um exemplo](https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/)* 

Seria bom de ter uma maneira simples para obter o *full name* diretamente! Nós podemos combinar a ideia de `getters` com tipo de dado especial que Sequelize providencia para esse tipo de situação: `DataTypes.VIRTUAL`:

```js
const { DataTypes } = require("sequelize");

const User = sequelize.define('user', {
  firstName: DataTypes.TEXT,
  lastName: DataTypes.TEXT,
  fullName: {
    type: DataTypes.VIRTUAL,
    get() {
      return `${this.firstName} ${this.lastName}`;
    },
    set(value) {
      throw new Error('Do not try to set the `fullName` value!');
    }
  }
});
```

O campo `VIRTUAL` não causa uma coluna na tabela existir. Em outras palavras, o model acima não vai ter uma coluna `fullName`. No entanto, vai parecer ter uma!

```js
const user = await User.create({ firstName: 'John', lastName: 'Doe' });
console.log(user.fullName); // 'John Doe'
```

## `getterMethods` e `setterMethods`

Sequelize também providencia as opções `getterMethods` e `setterMethods` na definição do model para especificar coisas que se parecem, mas não são exatamente iguais aos atributos virtuais.

Exemplo:

```js
const { Sequelize, DataTypes } = require('sequelize');
const sequelize = new Sequelize('sqlite::memory:');

const User = sequelize.define('user', {
  firstName: DataTypes.STRING,
  lastName: DataTypes.STRING
}, {
  getterMethods: {
    fullName() {
      return this.firstName + ' ' + this.lastName;
    }
  },
  setterMethods: {
    fullName(value) {

      // Nota: Isso é apenas para demonstração
      // Veja: https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/
      const names = value.split(' ');
      const firstName = names[0];
      const lastName = names.slice(1).join(' ');
      this.setDataValue('firstName', firstName);
      this.setDataValue('lastName', lastName);
    }
  }
});

(async () => {
  await sequelize.sync();
  let user = await User.create({ firstName: 'John',  lastName: 'Doe' });
  console.log(user.fullName); // 'John Doe'
  user.fullName = 'Someone Else';
  await user.save();
  user = await User.findOne();
  console.log(user.firstName); // 'Someone'
  console.log(user.lastName); // 'Else'
})();
```