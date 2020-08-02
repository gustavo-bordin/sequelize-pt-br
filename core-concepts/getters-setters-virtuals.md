# Getters, Setters & Virtuals

Sequelize permite você definir getters e setters customizados para os atributos de seus modelos.

Sequelize também permite você especificar os chamados *atributos virtuais*, que são os atributos do modelo no Sequelize que não existe na tabela SQL subjacente, mas, ao invés, são preenchidos automaticamente pelo Sequelize. Eles são muito úteis para simplificar o código, por exemplo.

## Getters

Um getter é uma função `get()` definida para uma coluna na definição do modelo.

```js
const Usuario = sequelize.define('usuario', {

  // Vamos dizer que nós queremos ver todo o 'nomeUsuario' em maiúsculo, embora
  // não sejam maiúsculos no próprio banco de dados
  nomeUsuario: {
    type: DataTypes.STRING,
    get() {
      const nomeCru = this.getDataValue(nomeUsuario);
      return nomeCru ? nomeCru.toUpperCase() : null;
    }
  }
});
```

Esse getter, apenas como um getter padrão do Javascript, é chamado automaticamente quando o valor do campo é lido.

```js
const usuario = Usuario.build({ nomeUsuario: 'SuperUsuario123' });
console.log(usuario.nomeUsuario); // 'SUPERUSUARIO123'
console.log(usuario.getDataValue(nomeUsuario)); // 'SuperUsuario123'
```

Note que, embora `SUPERUSUARIO123` foi registrado acima, o valor verdadeiro armazenado no banco de dados ainda é `SuperUsuario123`. Usamos `this.getDataValue(nomeUsuario)` para obter esse valor, e o convertemos para letras maiúsculas.

Se em vez disso tivéssemos tentado usar `this.nomeUsuario` no getter, teríamos um loop inifinito! É por isso que o Sequelize nos da o método `getDataValue`.

## Setters

Um setter é uma função `set()` definida para uma coluna na definição do modelo. Ele recebe um valor que está sendo definido:

```js
const Usuario = sequelize.define('usuario', {
  nomeUsuario: DataTypes.STRING,
  senha: {
    type: DataTypes.STRING,
    set(value) {

      // Armazenar senhas em texto puro no banco de dados é terrível.
      // Criptografar o valor com uma função apropriada de hash é melhor.
      this.setDataValue('senha', hash(value));
    }
  }
});
```

```js
const usuario = Usuario.build({ nomeUsuario: 'alguem', senha: 'NotSo§tr0ngP4$SW0RD!' });
console.log(usuario.senha); // '7cfc84b8ea898bb72462e78b4643cfccd77e9f05678ec2ce78754147ba947acc'
console.log(usuario.getDataValue(senha)); // '7cfc84b8ea898bb72462e78b4643cfccd77e9f05678ec2ce78754147ba947acc'
```

Observe que o Sequelize chamou o setter automaticamente, antes mesmo de enviar os dados para o banco de dados. O único dado que o banco de dados recebeu foi o valor já criptografado.

Se nós quisermos envolver um outro campo da instância do modelo dentro do calculo, isso é possível e muito fácil!

```js
const Usuario = sequelize.define('usuario', {
  nomeUsuario: DataTypes.STRING,
  senha: {
    type: DataTypes.STRING,
    set(value) {

      // Armazenar senhas em texto puro no banco de dados é terrível.
      // Criptografar o valor com uma função apropriada de hash é melhor.
      // Usar o nomeUsuario concatenado é melhor.
      
      this.setDataValue('senha', hash(this.nomeUsuario + value));
    }
  }
});
```

**Nota:** Os exemplos acima envolvendo gerenciamento de senhas, embora muito melhor do que simplesmente armazenar a senha em texto puro, estão longe de ser a segurança perfeita. Gerenciar senhas apropriadamente é difícil, tudo aqui é apenas para um exemplo cujo intuito é mostrar funcionalidades do Sequelize. Sugerimos a participação de um especialista em segurança cibernética a ler [OWASP](https://www.owasp.org/) e/ou visitar o [InfoSec StackExchange](https://security.stackexchange.com/).

## Combinando getters e setters

Getters e setters podem ser definidos no mesmo campo.

Para um exemplo, vamos dizer que nós estamos modelando um `Post`, cujo `conteudo` é um texto de tamanho ilimitado. Para melhorar o uso de memória, vamos dizer que nós queremos armazenar uma versão gzipped do conteúdo.

*Nota: Banco de dados modernos devem fazer alguma compressão automaticamente nesses casos. Por favor, note que isso é apenas para um exemplo.*

```js
const { gzipSync, gunzipSync } = require('zlib');

const Post = sequelize.define('post', {
  conteudo: {
    type: DataTypes.TEXT,
    get() {
      const valorGuardado = this.getDataValue('conteudo');
      const gzippedBuffer = Buffer.from(valorGuardado, 'base64');
      const unzippedBuffer = gunzipSync(gzippedBuffer);
      return unzippedBuffer.toString();
    },
    set(value) {
      const gzippedBuffer = gzipSync(value);
      this.setDataValue('conteudo', gzippedBuffer.toString('base64'));
    }
  }
});
```

Com a configuração acima, sempre que nós tentarmos interagir com o campo `conteudo` do nosso model `Post`, Sequelize vai automaticamente gerenciar o getter e setter personalizado. Por exemplo:

```js
const post = await Post.create({ conteudo: 'Olá pessoal!' });

console.log(post.conteudo); // 'Olá pessoal!'

// Tudo está acontecendo por baixo dos panos, então podemos até esquecer que o
// content está na verdade sendo armazenado como um gzipped base64 string!

// No entanto, se nós somos realmente curiosos, podemos pegar o dado 'cru'...
console.log(post.getDataValue('conteudo'));
// Saída: 'H4sIAAAAAAAACvNIzcnJV0gtSy2qzM9LVQQAUuk9jQ8AAAA='
```

## Campos virtuais

Campos virtuais são campos que o Sequelize preenche por baixo dos panos, mas na
realidade eles nem existem no banco de dados.

Por exemplo, vamos dizer que nós temos os atributos `primeiroNome` e `ultimoNome` para um Usuario.

*De novo, isso é [somente um exemplo](https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/)* 

Seria bom ter uma maneira simples para obter o *nome completo* diretamente! Nós podemos combinar a ideia de `getters` com tipo de dado especial que Sequelize providencia para esse tipo de situação: `DataTypes.VIRTUAL`:

```js
const { DataTypes } = require("sequelize");

const Usuario = sequelize.define('usuario', {
  primeiroNome: DataTypes.TEXT,
  ultimoNome: DataTypes.TEXT,
  nomeCompleto: {
    type: DataTypes.VIRTUAL,
    get() {
      return `${this.primeiroNome} ${this.ultimoNome}`;
    },
    set(value) {
      throw new Error('não tente atribuir o valor `nomeCompleto`!');
    }
  }
});
```

O campo `VIRTUAL` não cria uma coluna na tabela. Em outras palavras, o modelo acima não vai ter uma coluna `nomeCompleto`. No entanto, vai parecer ter uma!

```js
const usuario = await Usuario.create({ primeiroNome: 'John', ultimoNome: 'Doe' });
console.log(usuario.nomeCompleto); // 'John Doe'
```

## `getterMethods` e `setterMethods`

Sequelize também providencia as opções `getterMethods` e `setterMethods` na definição do modelo para especificar coisas que parecem atributos virtuais, mas não são exatamente iguais.

Exemplo:

```js
const { Sequelize, DataTypes } = require('sequelize');
const sequelize = new Sequelize('sqlite::memory:');

const Usuario = sequelize.define('usuario', {
  primeiroNome: DataTypes.STRING,
  ultimoNome: DataTypes.STRING
}, {
  getterMethods: {
    nomeCompleto() {
      return this.primeiroNome + ' ' + this.ultimoNome;
    }
  },
  setterMethods: {
    nomeCompleto(value) {

      // Nota: Isso é apenas para demonstração
      // Veja: https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/
      const nomes = value.split(' ');
      const primeiroNome = nomes[0];
      const ultimoNome = nomes.slice(1).join(' ');
      this.setDataValue('primeiroNome', primeiroNome);
      this.setDataValue('ultimoNome', primeiroNome);
    }
  }
});

(async () => {
  await sequelize.sync();
  let usuario = await Usuario.create({ primeiroNome: 'John',  ultimoNome: 'Doe' });
  console.log(usuario.nomeCompleto); // 'John Doe'
  usuario.nomeCompleto = 'Algum Outro';
  await usuario.save();
  usuario = await Usuario.findOne();
  console.log(usuario.primeiroNome); // 'Algum'
  console.log(usuario.ultimoNome); // 'Outro'
})();
```
