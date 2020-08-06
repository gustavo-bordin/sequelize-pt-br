# Sub Queries

Suponha que você tem dois modelos, `Postagem` e `Reacao`, com um relacionamento de Um-Para-Muitos configurado entre eles, então uma postagem tem muitas reações:

```js
const Postagem = sequelize.define('postagem', {
    conteudo: DataTypes.STRING
}, { timestamps: false });

const Reacao = sequelize.define('reacao', {
    type: DataTypes.STRING
}, { timestamps: false });

Postagem.hasMany(Reacao);
Reacao.belongsTo(Postagem);
```

*Nota: Desabilitamos os timestamps apenas para ter queries mais simples nos próximos exemplos.*

Vamos preencher nossas tabelas com alguns dados:

```js
async function criarPostagemComReacoes(conteudo, tiposReacao) {
    const postagem = await Postagem.create({ conteudo });
    await Reacao.bulkCreate(
        tiposReacao.map(tipo => ({ tipo, postagemId: postagem.id }))
    );
    return postagem;
}

await criarPostagemComReacoes('Olá Mundo', [
    'Like', 'Angry', 'Laugh', 'Like', 'Like', 'Angry', 'Sad', 'Like'
]);
await criarPostagemComReacoes('Minha segunda postagem', [
    'Laugh', 'Laugh', 'Like', 'Laugh'
]);
```

Agora, estamos prontos para os exemplos do poder das sub-queries.

Suponhamos que queremos calcular por SQL uma `contagemReacoesLaugh` para cada postagem. Podemos fazer isso com as sub-queries, como o seguinte:

```sql
SELECT
    *,
    (
        SELECT COUNT(*)
        FROM reacoes AS reacao
        WHERE
            reacao.postagemId = postagem.id
            AND
            reacao.tipo = "Laugh"
    ) AS contagemReacoesLaugh
FROM postagens AS postagem
```

Se rodarmos a query acima através do Sequelize, teremos:

```json
[
  {
    "id": 1,
    "conteudo": "Olá Mundo",
    "contagemReacoesLaugh": 1
  },
  {
    "id": 2,
    "content": "Minha segunda postagem",
    "contagemReacoesLaugh": 3
  }
]
```

Então como podemos ter o mesmo resultado acima com uma ajudinha do Sequelize, sem ter que escrever toda a query manualmente?

A resposta é: combinando a opção `attributes` dos métodos finders (como `findAll`) com a função utilitária `sequelize.literal`, isso te permite inserir diretamente conteudo arbitrário na query sem escape automático.

Isso significa que o Sequelize te ajudará com a maior e principal query, mas você ainda terá que escrever a subquery sozinho:

```js
Postagem.findAll({
    attributes: {
        include: [
            [
                // Note os parênteses envolvendo a chamada abaixo
                sequelize.literal(`(
                    SELECT COUNT(*)
                    FROM reacoes AS reacao
                    WHERE
                        reacao.postagemId = postagem.id
                        AND
                        reacao.type = "Laugh"
                )`),
                'contagemReacoesLaugh'
            ]
        ]
    }
});
```

*Nota importante: Já que `sequelize.literal` insere conteudo arbitrário sem escapar para a query, a função merece uma atenção redobrada, pois pode ser a principal fonte de vulnerabilidades. Não deveria ser usada em conteudos gerados pelo usuário.* Contudo, aqui, estamos usando `sequelize.literal` com uma string fixa, cuidadosamente escrita por nós (os programadores). Dessa forma está tudo bem, desde que sabemos o que estamos fazendo.

O exemplo acima gera o seguinte resultado:

```json
[
  {
    "id": 1,
    "conteudo": "Olá Mundo",
    "contagemReacoesLaugh": 1
  },
  {
    "id": 2,
    "content": "Minha segunda postagem",
    "contagemReacoesLaugh": 3
  }
]
```

Sucesso!

## Usando sub-queries para ordenar os resultados

Essa ideia pode ser usada para ordenar os resultados, desde a ordenação simples até a mais complexa, aqui, vamos ordenar as postagens pelo numero de reações 'Laugh' que elas obtiveram:

```js
Postagem.findAll({
    attributes: {
        include: [
            [
                sequelize.literal(`(
                    SELECT COUNT(*)
                    FROM reacoes AS reacao
                    WHERE
                        reacao.postagemId = postagem.id
                        AND
                        reacao.type = "Laugh"
                )`),
                'contagemReacoesLaugh'
            ]
        ]
    },
    order: [
        [sequelize.literal('contagemReacoesLaugh'), 'DESC']
    ]
});
```

Result:

```json
[
  {
    "id": 2,
    "conteudo": "Minha segunda postagem",
    "contagemReacoesLaugh": 3
  },
  {
    "id": 1,
    "conteudo": "Olá mundo",
    "contagemReacoesLaugh": 1
  }
]
```