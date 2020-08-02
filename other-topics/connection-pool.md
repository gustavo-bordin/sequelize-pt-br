# Conjunto de conexões

Se você está conectando ao banco de dados por apenas uma etapa, você deveria criar apenas uma instância do Sequelize. o Sequelize irá configurar um conjunto de conexões ao inicializar. Esse conjunto de conexões pode ser configurado através do parametro de opções do construtor (usando  `options.pool`), como mostrado no exemplo abaixo:

```js
const sequelize = new Sequelize(/* ... */, {
  // ...
  pool: {
    max: 5,
    min: 0,
    acquire: 30000,
    idle: 10000
  }
});
```

Veja mais na [Referência da API para o construtor do Sequelize](../class/lib/sequelize.js~Sequelize.html#instance-constructor-constructor). Se você está se conectando ao banco de dados por várias processos, você terá que criar uma instância para cada processo, porém cada instância deve ter limite para o conjunto de conexões, de forma que o tamanho máximo total seja respeitado. Por exemplo, Se você quiser um limite de 90 e você tem 3 processos, a instância do Sequelize para cada processo deve ter um limite para o conjunto de conexões de 30.
