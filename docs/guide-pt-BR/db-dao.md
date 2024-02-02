Database Access Objects (Objeto de acesso a dados)
=======================

Construído em cima do [PDO](https://www.php.net/manual/en/book.pdo.php), Yii DAO (Database Access Objects) fornece uma
API orientada a objetos para acessar bancos de dados relacionais. Ele é a base para outros métodos de acesso a bancos de dados
mais avançados, incluindo [query builder](db-query-builder.md) and [active record](db-active-record.md).

Ao usar Yii DAO, você precisará principalmente lidar com SQLs simples e arrays PHP. Como resultado, é a maneira mais eficiente
de acessar bancos de dados. Apesar disso, pela sintaxe SQL variar para bancos de dados diferentes, utilizar Yii DAO também significa 
que você precisa ter esforço extra para criar uma aplicação independente do tipo de banco de dados. 

No Yii 2.0, DAO suporta os seguintes bancos de dados por padrão:

- [MySQL](https://www.mysql.com/)
- [MariaDB](https://mariadb.com/)
- [SQLite](https://sqlite.org/)
- [PostgreSQL](https://www.postgresql.org/): versão 8.4 ou maior.
- [CUBRID](https://www.cubrid.org/): versão 9.3 ou maior.
- [Oracle](https://www.oracle.com/database/)
- [MSSQL](https://www.microsoft.com/en-us/sqlserver/default.aspx): versão 2008 ou maior.

> Observação: A nova versão do pdo_oci para PHP 7 atualmente existe só como código fonte. Siga
  [instrução fornecida pela comunidade](https://github.com/yiisoft/yii2/issues/10975#issuecomment-248479268)
  para compilar ou use [PDO emulation layer](https://github.com/taq/pdooci).

## Criando Conexões a BDs <span id="creating-db-connections"></span>

Para acessar um banco de dados, primeiro você precisa conectar a ele criando uma instância de [[yii\db\Connection]]:

```php
$db = new yii\db\Connection([
    'dsn' => 'mysql:host=localhost;dbname=example',
    'username' => 'root',
    'password' => '',
    'charset' => 'utf8',
]);
```

Como uma conexão a BD geralmente precisa ser acessada em lugares diferentes, é uma prática comum configurá-la como
um [componente de aplicação](structure-application-components.md) como o seguinte:

```php
return [
    // ...
    'components' => [
        // ...
        'db' => [
            'class' => 'yii\db\Connection',
            'dsn' => 'mysql:host=localhost;dbname=example',
            'username' => 'root',
            'password' => '',
            'charset' => 'utf8',
        ],
    ],
    // ...
];
```

Você pode então acessar a conexão BD pela expressão `Yii::$app->db`.

> Dica: Você pode configurar múltiplos componentes de aplicação BD se sua aplicação precisar de acesso a múltiplos bancos de dados.

Ao configurar uma conexão de BD, você deve sempre especificar seu Data Source Name (DSN) por meio da propriedade [[yii\db\Connection::dsn|dsn]]. 
O formato do DSN varia para bancos de dados diferentes. Por favor leia o [PHP manual](https://www.php.net/manual/en/pdo.construct.php) 
para mais detalhes. Abaixo estão alguns exemplos:
 
* MySQL, MariaDB: `mysql:host=localhost;dbname=mydatabase`
* SQLite: `sqlite:/path/to/database/file`
* PostgreSQL: `pgsql:host=localhost;port=5432;dbname=mydatabase`
* CUBRID: `cubrid:dbname=demodb;host=localhost;port=33000`
* MS SQL Server (via sqlsrv driver): `sqlsrv:Server=localhost;Database=mydatabase`
* MS SQL Server (via dblib driver): `dblib:host=localhost;dbname=mydatabase`
* MS SQL Server (via mssql driver): `mssql:host=localhost;dbname=mydatabase`
* Oracle: `oci:dbname=//localhost:1521/mydatabase`

Perceba que se você estiver conectando num banco de dados via ODBC, você deve configurar a propriedade [[yii\db\Connection::driverName]]
 para que o Yii possa saber o tipo do banco de dados. Por exemplo,

```php
'db' => [
    'class' => 'yii\db\Connection',
    'driverName' => 'mysql',
    'dsn' => 'odbc:Driver={MySQL};Server=localhost;Database=test',
    'username' => 'root',
    'password' => '',
],
```

Além da propriedade [[yii\db\Connection::dsn|dsn]], você frequentemente precisará configurar [[yii\db\Connection::username|username]]
e [[yii\db\Connection::password|password]]. Por favor consulte [[yii\db\Connection]] para a lista completa de propriedades configuráveis.

> Informação: Quando você cria uma instância de conexão com o banco de dados, a conexão com o banco não é estabelecida até que 
  você execute o primeiro SQL ou chame o método [[yii\db\Connection::open()|open()]] explicitamente.

> Dica: Às vezes você pode querer executar algumas consultas logo após a conexão com o banco de dados ser estabelecida para inicializar
> algumas variáveis ​​de ambiente (por exemplo, para definir o fuso horário ou conjunto de caracteres). Você pode fazer isso registrando um manipulador de eventos 
> para o evento [[yii\db\Connection::EVENT_AFTER_OPEN|afterOpen]] 
> da conexão com o banco de dados. Você pode registrar o manipulador diretamente na configuração do aplicativo assim:
> 
> ```php
> 'db' => [
>     // ...
>     'on afterOpen' => function($event) {
>         // $event->sender se refere a conexão BD
>         $event->sender->createCommand("SET time_zone = 'UTC'")->execute();
>     }
> ],
> ```

Para o MS SQL Server, é necessária uma opção de conexão adicional para o tratamento adequado de dados binários:

```php
'db' => [
 'class' => 'yii\db\Connection',
    'dsn' => 'sqlsrv:Server=localhost;Database=mydatabase',
    'attributes' => [
        \PDO::SQLSRV_ATTR_ENCODING => \PDO::SQLSRV_ENCODING_SYSTEM
    ]
],
```


## Executando Consultas SQL <span id="executing-sql-queries"></span>

Uma vez que você tem uma instância de conexão com o banco de dados, você pode executar uma consulta SQL pelas seguintes etapas:
 
1. Crie um [[yii\db\Command]] com uma consulta SQL simples;
2. Vincule os parâmetros (opcional);
3. Chame um dos métodos de execução SQL em [[yii\db\Command]].

O exemplo a seguir mostra várias maneiras de buscar dados de um banco de dados:
 
```php
// retorna um conjunto de linhas. cada linha é uma array associativa de nomes e valores.
// um array vazio é retornado se a consulta não encontrar resultados
$posts = Yii::$app->db->createCommand('SELECT * FROM post')
            ->queryAll();

// retorna uma única linha (a primeira linha)
// false é retornado se a consulta não encontrar resultados
$post = Yii::$app->db->createCommand('SELECT * FROM post WHERE id=1')
           ->queryOne();

// retorna uma única coluna (a primeira coluna)
// um array vazio é retornado se a consulta não encontrar resultados
$titles = Yii::$app->db->createCommand('SELECT title FROM post')
             ->queryColumn();

// retorna um valor escalar
// false é retornado se a consulta não encontrar resultados
$count = Yii::$app->db->createCommand('SELECT COUNT(*) FROM post')
             ->queryScalar();
```

> Observação: Para preservar a precisão, os dados obtidos dos bancos de dados são todos representados como strings, mesmo que a coluna
  do banco correspondente tiver tipo numérico.


### Vinculando Parâmetros <span id="binding-parameters"></span>

Ao criar um BD command de um SQL com parâmetros, você quase sempre deve usar a abordagem de vincular parâmetros
para prevenir ataques de injeção de SQL. Por exemplo,

```php
$post = Yii::$app->db->createCommand('SELECT * FROM post WHERE id=:id AND status=:status')
           ->bindValue(':id', $_GET['id'])
           ->bindValue(':status', 1)
           ->queryOne();
```

Na instrução SQL, você pode incorporar um ou vários espaços reservados dos parâmetros (`:id` no exemplo acima). Um espaço
reservado do parâmetro deve ser uma string começando com dois pontos. Você pode então chamar um dos seguintes métodos de vinculação
para vincular os valores dos parâmetros:

* [[yii\db\Command::bindValue()|bindValue()]]: vincular um único valor de parâmetro 
* [[yii\db\Command::bindValues()|bindValues()]]:  vincular múltiplos valores de parâmetro em uma chamada 
* [[yii\db\Command::bindParam()|bindParam()]]: similar ao [[yii\db\Command::bindValue()|bindValue()]] mas também
  suporta referências aos parâmetros vinculados.

O exemplo a seguir mostra meios alternativos de vincular parâmetros:

```php
$params = [':id' => $_GET['id'], ':status' => 1];

$post = Yii::$app->db->createCommand('SELECT * FROM post WHERE id=:id AND status=:status')
           ->bindValues($params)
           ->queryOne();
           
$post = Yii::$app->db->createCommand('SELECT * FROM post WHERE id=:id AND status=:status', $params)
           ->queryOne();
```

A vinculação de parâmetros é implementada via [prepared statements](https://www.php.net/manual/en/mysqli.quickstart.prepared-statements.php).
Além de prevenir ataques de injeção de SQL, também pode melhorar o desempenho preparando uma instrução SQL apenas uma vez e
executando-a várias vezes com parâmetros diferentes. Por exemplo,

```php
$command = Yii::$app->db->createCommand('SELECT * FROM post WHERE id=:id');

$post1 = $command->bindValue(':id', 1)->queryOne();
$post2 = $command->bindValue(':id', 2)->queryOne();
// ...
```

Como [[yii\db\Command::bindParam()|bindParam()]] suporta referências aos parâmetros vinculados, o código acima
também pode ser escrito da seguinte forma:

```php
$command = Yii::$app->db->createCommand('SELECT * FROM post WHERE id=:id')
              ->bindParam(':id', $id);

$id = 1;
$post1 = $command->queryOne();

$id = 2;
$post2 = $command->queryOne();
// ...
```

Note que você vincula um substituto à variável `$id` antes da execução, e então muda o valor daquela variável 
antes de cada execução subsequente (isso geralmente é feito com loops). Executar consultas dessa maneira pode ser muito mais
eficiente do que executar uma nova consulta para cada valor de parâmetro diferente.

> Informação: A vinculação de parâmetros só é usada em locais onde valores precisam ser inseridos em strings que contêm SQL simples.
> Em muitos lugares em camadas de abstração superiores, como [query builder](db-query-builder.md) e [active record](db-active-record.md)
> você geralmente especifica um array de valores que será transformado em SQL. Nestes locais a vinculação de parâmetros é feita internamente pelo Yii,
> portanto é necessário especificar parâmetros manualmente.

### Executando Consultas Não-SELECT <span id="non-select-queries"></span>

Os métodos `queryXyz()` introduzidos nas seções anteriores lidam com consultas SELECT que buscam dados em bancos de dados.
Para consultas que não trazem dados de volta, você deve chamar o método [[yii\db\Command::execute()]]. Por exemplo,

```php
Yii::$app->db->createCommand('UPDATE post SET status=1 WHERE id=1')
   ->execute();
```

O método [[yii\db\Command::execute()]] retorna o número de linhas afetadas pela execução do SQL.

Para consultas INSERT, UPDATE e DELETE, em vez de escrever SQLs simples, você pode chamar, respectivamente, [[yii\db\Command::insert()|insert()]],
[[yii\db\Command::update()|update()]], [[yii\db\Command::delete()|delete()]], para construir os SQLs correspondentes.
 Esses métodos citarão corretamente os nomes das tabelas e colunas e vincularão os valores dos parâmetros. Por exemplo,

```php
// INSERT (nome da tabela, valores das colunas)
Yii::$app->db->createCommand()->insert('user', [
    'name' => 'Sam',
    'age' => 30,
])->execute();

// UPDATE (nome da tabela, valores das colunas, condição)
Yii::$app->db->createCommand()->update('user', ['status' => 1], 'age > 30')->execute();

// DELETE (nome da tabela, condição)
Yii::$app->db->createCommand()->delete('user', 'status = 0')->execute();
```

Você também pode chamar [[yii\db\Command::batchInsert()|batchInsert()]] para inserir múltiplas linhas de uma só vez, o que é
muito mais eficiente do que inserir uma linha de cada vez:

```php
// nome da tabela, nomes das colunas, valores das colunas
Yii::$app->db->createCommand()->batchInsert('user', ['name', 'age'], [
    ['Tom', 30],
    ['Jane', 20],
    ['Linda', 25],
])->execute();
```

Outro método útil é o [[yii\db\Command::upsert()|upsert()]]. Upsert é uma operação atômica que insere linhas
em uma tabela de banco de dados se elas ainda não existirem (correspondendo a restrições exclusivas) ou as atualiza se existirem:

```php
Yii::$app->db->createCommand()->upsert('pages', [
    'name' => 'Front page',
    'url' => 'https://example.com/', // url is unique
    'visits' => 0,
], [
    'visits' => new \yii\db\Expression('visits + 1'),
], $params)->execute();
```

O código acima irá inserir um novo registro de página ou incrementar seu contador de visitas atomicamente.

Observe que os métodos mencionados acima apenas criam a consulta e você sempre terá que chamar [[yii\db\Command::execute()|execute()]]
para realmente executá-los.


## Citando Nomes de Tabelas e Colunas  <span id="quoting-table-and-column-names"></span>

Ao escrever código independente de banco de dados, citar corretamente nomes de tabelas e colunas costuma ser uma dor de cabeça
porque bancos de dados diferentes têm regras de citação de nomes diferentes. Para superar esse problema, você pode usar a seguinte
sintaxe de citação introduzida pelo Yii:

* `[[column name]]`: coloque um nome da coluna a ser citada entre colchetes duplos; 
* `{{table name}}`: coloque o nome da tabela a ser citada entre chaves duplas.

Yii DAO irá converter automaticamente tais constructos nas colunas ou nomes de tabelas entre aspas correspondentes usando
sintaxe específica do SGBD (Sistema de gerenciamento de banco de dados).
Por exemplo,

```php
// executa esse SQL para MySQL: SELECT COUNT(`id`) FROM `employee`
$count = Yii::$app->db->createCommand("SELECT COUNT([[id]]) FROM {{employee}}")
            ->queryScalar();
```


### Usando Prefixo de Tabela <span id="using-table-prefix"></span>

Se a maioria dos nomes de suas tabelas de banco de dados compartilham um prefixo comum, você pode usar o recurso de prefixo de tabela fornecido
por Yii DAO.

Primeiro, especifique o prefixo da tabela por meio da propriedade [[yii\db\Connection::tablePrefix]] nas configurações da aplicação:

```php
return [
    // ...
    'components' => [
        // ...
        'db' => [
            // ...
            'tablePrefix' => 'tbl_',
        ],
    ],
];
```

Então, no seu código, sempre que precisar se referir a uma tabela cujo nome contém esse prefixo, use a sintaxe
`{{%table_name}}`. O caractere de porcentagem será automaticamente substituído pelo prefixo da tabela que você especificou
ao configurar a conexão do banco de dados. Por exemplo,

```php
// executa esse SQL para MySQL: SELECT COUNT(`id`) FROM `tbl_employee`
$count = Yii::$app->db->createCommand("SELECT COUNT([[id]]) FROM {{%employee}}")
            ->queryScalar();
```


## Realizando Transações <span id="performing-transactions"></span>

Ao executar várias consultas relacionadas em uma sequência, pode ser necessário envolvê-las em uma transação para garantir a integridade
e consistência do seu banco de dados. Se alguma das consultas falhar, o banco de dados será revertido para o estado anterior, e será como se
nenhuma dessas consultas tivesse sido executada.
 
O código a seguir mostra uma maneira comum de usar transações:

```php
Yii::$app->db->transaction(function($db) {
    $db->createCommand($sql1)->execute();
    $db->createCommand($sql2)->execute();
    // ... executing other SQL statements ...
});
```

O código acima é equivalente ao seguinte, que dá mais controle sobre o código de tratamento de erros:

```php
$db = Yii::$app->db;
$transaction = $db->beginTransaction();
try {
    $db->createCommand($sql1)->execute();
    $db->createCommand($sql2)->execute();
    // ... executing other SQL statements ...
    
    $transaction->commit();
} catch(\Exception $e) {
    $transaction->rollBack();
    throw $e;
} catch(\Throwable $e) {
    $transaction->rollBack();
    throw $e;
}
```

Ao chamar o método [[yii\db\Connection::beginTransaction()|beginTransaction()]], uma nova transação é iniciada.
A transação é representada como um objeto [[yii\db\Transaction]] salvo na variável `$transaction`. Então,
as consultas executadas são colocadas em um bloco `try...catch...`. Se todas as consultas forem executadas com sucesso,
o método [[yii\db\Transaction::commit()|commit()]] é chamado para confirmar a transação. Caso contrário, se uma exceção
for lançada e capturada, o método [[yii\db\Transaction::rollBack()|rollBack()]] é chamado para reverter
as mudanças feitas pelas consultas anteriores à consulta que falhou na transação. `throw $e` então lançará novamente a
exceção como se não a tivéssemos detectado, logo o processo normal de tratamento de erros cuidará dela.

> Observação: no código acima temos dois blocos catch para compatibilidade
> com PHP 5.x e PHP 7.x. `\Exception` implementa [`\Throwable` interface](https://www.php.net/manual/en/class.throwable.php)
> desde PHP 7.0, então você pode pular a parte com `\Exception` se seu app usar apenas PHP 7.0 ou superior.


### Especificando Níveis de Isolamento <span id="specifying-isolation-levels"></span>

Yii também suporta configuração de [isolation levels] para suas transações. Por padrão, ao iniciar uma nova transação,
ele usará o nível de isolamento padrão definido pelo seu sistema de banco de dados. Você pode substituir o nível de isolamento padrão da seguinte maneira,

```php
$isolationLevel = \yii\db\Transaction::REPEATABLE_READ;

Yii::$app->db->transaction(function ($db) {
    ....
}, $isolationLevel);
 
// ou alternativamente

$transaction = Yii::$app->db->beginTransaction($isolationLevel);
```

Yii fornece quatro constantes para os níveis de isolamento mais comuns:

- [[\yii\db\Transaction::READ_UNCOMMITTED]] - no nível mais fraco, podem ocorrer leituras sujas, leituras não repetíveis e fantasmas.
- [[\yii\db\Transaction::READ_COMMITTED]] - evita leituras sujas.
- [[\yii\db\Transaction::REPEATABLE_READ]] - evita leituras sujas e leituras não repetiveis.
- [[\yii\db\Transaction::SERIALIZABLE]] - o nível mais forte, evita todos os problemas citados acima.

Além de usar as constantes acima para especificar níveis de isolamento, você também pode usar strings com uma sintaxe válida suportada
pelo SGBD que você estiver utilizando. Por exemplo, no PostgreSQL, você pode usar `"SERIALIZABLE READ ONLY DEFERRABLE"`.

Observe que alguns SGBDs permitem definir o nível de isolamento apenas para a conexão inteira. Quaisquer transações subsequentes
terão o mesmo nível de isolamento mesmo se você não especificar nenhum. Ao usar este recurso
pode ser necessário definir explicitamente o nível de isolamento para todas as transações para evitar configurações conflitantes.
No momento da redação deste artigo, apenas MSSQL e SQLite são afetados por esta limitação.

> Observação: SQLite suporta apenas dois níveis de isolamento, então você só pode usar `READ UNCOMMITTED` e `SERIALIZABLE`.
O uso de outros níveis resultará no lançamento de uma exceção.

> Observação: O PostgreSQL não permite definir o nível de isolamento antes do início da transação, portanto você não pode
especificar o nível de isolamento diretamente ao iniciar a transação.
Você deve chamar neste caso [[yii\db\Transaction::setIsolationLevel()]] após o início da transação.

[isolation levels]: https://en.wikipedia.org/wiki/Isolation_%28database_systems%29#Isolation_levels


### Aninhando Transações <span id="nesting-transactions"></span>

Se o seu SGBD suportar Savepoint, você poderá aninhar várias transações como a seguir:

```php
Yii::$app->db->transaction(function ($db) {
    // transação externa
    
    $db->transaction(function ($db) {
        // transação interna
    });
});
```

Ou alternativamente,

```php
$db = Yii::$app->db;
$outerTransaction = $db->beginTransaction();
try {
    $db->createCommand($sql1)->execute();

    $innerTransaction = $db->beginTransaction();
    try {
        $db->createCommand($sql2)->execute();
        $innerTransaction->commit();
    } catch (\Exception $e) {
        $innerTransaction->rollBack();
        throw $e;
    } catch (\Throwable $e) {
        $innerTransaction->rollBack();
        throw $e;
    }

    $outerTransaction->commit();
} catch (\Exception $e) {
    $outerTransaction->rollBack();
    throw $e;
} catch (\Throwable $e) {
    $outerTransaction->rollBack();
    throw $e;
}
```


## Replication and Read-Write Splitting <span id="read-write-splitting"></span>

Many DBMS support [database replication](https://en.wikipedia.org/wiki/Replication_(computing)#Database_replication)
to get better database availability and faster server response time. With database replication, data are replicated
from the so-called *master servers* to *slave servers*. All writes and updates must take place on the master servers,
while reads may also take place on the slave servers.

To take advantage of database replication and achieve read-write splitting, you can configure a [[yii\db\Connection]]
component like the following:

```php
[
    'class' => 'yii\db\Connection',

    // configuration for the master
    'dsn' => 'dsn for master server',
    'username' => 'master',
    'password' => '',

    // common configuration for slaves
    'slaveConfig' => [
        'username' => 'slave',
        'password' => '',
        'attributes' => [
            // use a smaller connection timeout
            PDO::ATTR_TIMEOUT => 10,
        ],
    ],

    // list of slave configurations
    'slaves' => [
        ['dsn' => 'dsn for slave server 1'],
        ['dsn' => 'dsn for slave server 2'],
        ['dsn' => 'dsn for slave server 3'],
        ['dsn' => 'dsn for slave server 4'],
    ],
]
```

The above configuration specifies a setup with a single master and multiple slaves. One of the slaves will
be connected and used to perform read queries, while the master will be used to perform write queries.
Such read-write splitting is accomplished automatically with this configuration. For example,

```php
// create a Connection instance using the above configuration
Yii::$app->db = Yii::createObject($config);

// query against one of the slaves
$rows = Yii::$app->db->createCommand('SELECT * FROM user LIMIT 10')->queryAll();

// query against the master
Yii::$app->db->createCommand("UPDATE user SET username='demo' WHERE id=1")->execute();
```

> Info: Queries performed by calling [[yii\db\Command::execute()]] are considered as write queries, while
  all other queries done through one of the "query" methods of [[yii\db\Command]] are read queries.
  You can get the currently active slave connection via `Yii::$app->db->slave`.

The `Connection` component supports load balancing and failover between slaves.
When performing a read query for the first time, the `Connection` component will randomly pick a slave and
try connecting to it. If the slave is found "dead", it will try another one. If none of the slaves is available,
it will connect to the master. By configuring a [[yii\db\Connection::serverStatusCache|server status cache]],
a "dead" server can be remembered so that it will not be tried again during a
[[yii\db\Connection::serverRetryInterval|certain period of time]].

> Info: In the above configuration, a connection timeout of 10 seconds is specified for every slave.
  This means if a slave cannot be reached in 10 seconds, it is considered as "dead". You can adjust this parameter
  based on your actual environment.


You can also configure multiple masters with multiple slaves. For example,


```php
[
    'class' => 'yii\db\Connection',

    // common configuration for masters
    'masterConfig' => [
        'username' => 'master',
        'password' => '',
        'attributes' => [
            // use a smaller connection timeout
            PDO::ATTR_TIMEOUT => 10,
        ],
    ],

    // list of master configurations
    'masters' => [
        ['dsn' => 'dsn for master server 1'],
        ['dsn' => 'dsn for master server 2'],
    ],

    // common configuration for slaves
    'slaveConfig' => [
        'username' => 'slave',
        'password' => '',
        'attributes' => [
            // use a smaller connection timeout
            PDO::ATTR_TIMEOUT => 10,
        ],
    ],

    // list of slave configurations
    'slaves' => [
        ['dsn' => 'dsn for slave server 1'],
        ['dsn' => 'dsn for slave server 2'],
        ['dsn' => 'dsn for slave server 3'],
        ['dsn' => 'dsn for slave server 4'],
    ],
]
```

The above configuration specifies two masters and four slaves. The `Connection` component also supports
load balancing and failover between masters just as it does between slaves. A difference is that when none 
of the masters are available an exception will be thrown.

> Note: When you use the [[yii\db\Connection::masters|masters]] property to configure one or multiple
  masters, all other properties for specifying a database connection (e.g. `dsn`, `username`, `password`)
  with the `Connection` object itself will be ignored.


By default, transactions use the master connection. And within a transaction, all DB operations will use
the master connection. For example,

```php
$db = Yii::$app->db;
// the transaction is started on the master connection
$transaction = $db->beginTransaction();

try {
    // both queries are performed against the master
    $rows = $db->createCommand('SELECT * FROM user LIMIT 10')->queryAll();
    $db->createCommand("UPDATE user SET username='demo' WHERE id=1")->execute();

    $transaction->commit();
} catch(\Exception $e) {
    $transaction->rollBack();
    throw $e;
} catch(\Throwable $e) {
    $transaction->rollBack();
    throw $e;
}
```

If you want to start a transaction with the slave connection, you should explicitly do so, like the following:

```php
$transaction = Yii::$app->db->slave->beginTransaction();
```

Sometimes, you may want to force using the master connection to perform a read query. This can be achieved
with the `useMaster()` method:

```php
$rows = Yii::$app->db->useMaster(function ($db) {
    return $db->createCommand('SELECT * FROM user LIMIT 10')->queryAll();
});
```

You may also directly set `Yii::$app->db->enableSlaves` to be `false` to direct all queries to the master connection.


## Working with Database Schema <span id="database-schema"></span>

Yii DAO provides a whole set of methods to let you manipulate the database schema, such as creating new tables,
dropping a column from a table, etc. These methods are listed as follows:

* [[yii\db\Command::createTable()|createTable()]]: creating a table
* [[yii\db\Command::renameTable()|renameTable()]]: renaming a table
* [[yii\db\Command::dropTable()|dropTable()]]: removing a table
* [[yii\db\Command::truncateTable()|truncateTable()]]: removing all rows in a table
* [[yii\db\Command::addColumn()|addColumn()]]: adding a column
* [[yii\db\Command::renameColumn()|renameColumn()]]: renaming a column
* [[yii\db\Command::dropColumn()|dropColumn()]]: removing a column
* [[yii\db\Command::alterColumn()|alterColumn()]]: altering a column
* [[yii\db\Command::addPrimaryKey()|addPrimaryKey()]]: adding a primary key
* [[yii\db\Command::dropPrimaryKey()|dropPrimaryKey()]]: removing a primary key
* [[yii\db\Command::addForeignKey()|addForeignKey()]]: adding a foreign key
* [[yii\db\Command::dropForeignKey()|dropForeignKey()]]: removing a foreign key
* [[yii\db\Command::createIndex()|createIndex()]]: creating an index
* [[yii\db\Command::dropIndex()|dropIndex()]]: removing an index

These methods can be used like the following:

```php
// CREATE TABLE
Yii::$app->db->createCommand()->createTable('post', [
    'id' => 'pk',
    'title' => 'string',
    'text' => 'text',
]);
```

The above array describes the name and types of the columns to be created. For the column types, Yii provides
a set of abstract data types, that allow you to define a database agnostic schema. These are converted to
DBMS specific type definitions dependent on the database, the table is created in.
Please refer to the API documentation of the [[yii\db\Command::createTable()|createTable()]]-method for more information.

Besides changing the database schema, you can also retrieve the definition information about a table through
the [[yii\db\Connection::getTableSchema()|getTableSchema()]] method of a DB connection. For example,

```php
$table = Yii::$app->db->getTableSchema('post');
```

The method returns a [[yii\db\TableSchema]] object which contains the information about the table's columns,
primary keys, foreign keys, etc. All this information is mainly utilized by [query builder](db-query-builder.md) 
and [active record](db-active-record.md) to help you write database-agnostic code. 
