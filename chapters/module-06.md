## Módulo 6: Integração da Table API com SQL

Bem-vindo ao Módulo 6! O Apache Flink oferece uma poderosa sinergia entre sua Table API programática e a familiaridade do SQL. Você pode definir tabelas e executar transformações usando a Table API e, em seguida, consultar essas tabelas ou definir novas transformações usando SQL, ou vice-versa. Esta flexibilidade permite que equipes com diferentes conjuntos de habilidades colaborem efetivamente e escolham a melhor ferramenta para cada tarefa específica.

Neste módulo, vamos cobrir:
* O Catálogo do Flink e como registrar `Tables` nele.
* Como executar queries SQL diretamente no `TableEnvironment`.
* Como criar e utilizar `Views` para simplificar e modularizar suas queries.
* Estratégias para misturar operações da Table API com queries SQL.
* Table API/SQL em Ambientes Gerenciados: Confluent Cloud for Apache Flink.

---

### O Catálogo do Flink (`Catalog`)

O `Catalog` no Flink serve como um repositório de metadados. Ele armazena informações sobre objetos como `databases`, `tables`, `views`, funções definidas pelo usuário (UDFs), e conectores para fontes de dados externas. O uso de um catálogo permite que você referencie esses objetos por nome em suas queries SQL e operações da Table API, facilitando a organização e o reuso.

**Principais Conceitos do Catálogo:**

* **`Catalog`:** A interface principal para interagir com metadados. O Flink vem com um catálogo padrão em memória, mas suporta catálogos externos.
    * **`GenericInMemoryCatalog`:** Este é o catálogo padrão usado pelo Flink. Ele armazena todos os metadados (tabelas, views, UDFs) apenas na memória da sessão atual do `TableEnvironment`. Isso significa que os metadados são perdidos quando a sessão termina. É útil para desenvolvimento e testes.
* **`Database`:** Um namespace dentro de um catálogo para organizar tabelas e outros objetos. Cada catálogo tem um *database* padrão.
* **`Table`:** Representa uma tabela com dados. Pode ser uma tabela originada de um sistema externo (via conector), de um `DataStream`, ou de valores em memória.
* **`View`:** Uma tabela virtual definida por uma query. Não armazena dados fisicamente; a query é executada quando a *view* é referenciada.
* **`Function`:** Funções definidas pelo usuário (UDFs) que podem ser registradas no catálogo.

**Catálogos Suportados (além do padrão):**

* **`JdbcCatalog`:** Permite conectar a bancos de dados relacionais (como PostgreSQL, MySQL) e usar suas tabelas e views diretamente no Flink. Os metadados são persistidos no banco de dados conectado.
* **`HiveCatalog`:** Permite integrar com o Metastore do Apache Hive. Isso é extremamente útil se você já tem um ecossistema Hadoop/Hive, pois o Flink pode ler e escrever em tabelas Hive, usar UDFs Hive, etc. Os metadados são persistidos no Hive Metastore.
* Outros catálogos customizados podem ser implementados.

**Interagindo com o Catálogo Programaticamente:**

O `TableEnvironment` oferece métodos para gerenciar catálogos e *databases*:

```java
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.TableEnvironment;
import java.util.Arrays; // Para Arrays.toString

// Supondo que você tenha um método main ou um local para executar este código:
// public static void main(String[] args) {
//     EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().build();
//     TableEnvironment tableEnv = TableEnvironment.create(settings);

//     // Listar catálogos disponíveis
//     System.out.println("Catálogos disponíveis: " + Arrays.toString(tableEnv.listCatalogs()));

//     // Obter o catálogo atual
//     String currentCatalog = tableEnv.getCurrentCatalog();
//     System.out.println("Catálogo atual: " + currentCatalog);

//     // Obter o database atual no catálogo corrente
//     String currentDb = tableEnv.getCurrentDatabase();
//     System.out.println("Database atual: " + currentDb);

//     // Exemplo: Criar um novo database (se o catálogo suportar)
//     // try {
//     //     tableEnv.executeSql("CREATE DATABASE IF NOT EXISTS my_test_db");
//     //     tableEnv.useDatabase("my_test_db");
//     //     System.out.println("Database atual alterado para: " + tableEnv.getCurrentDatabase());
//     // } catch (Exception e) {
//     //     System.err.println("Erro ao criar/usar database: " + e.getMessage());
//     // }

//     // Listar databases no catálogo atual
//     System.out.println("Databases no catálogo '" + tableEnv.getCurrentCatalog() + "': " + Arrays.toString(tableEnv.listDatabases()));

//     // Listar tabelas e views no database atual
//     System.out.println("Tabelas e Views (inicialmente): " + Arrays.toString(tableEnv.listTables()));
// }
```

---

### Registrando `Tables` no Catálogo

Para que uma `Table` (ou `View`) seja acessível por nome em queries SQL ou em outras operações da Table API (`tableEnv.from("tableName")`), ela precisa ser registrada no catálogo.

**1. A Partir de um Objeto `Table` da Table API:**

Se você criou um objeto `Table` usando as operações da Table API (e.g., `tableEnv.fromValues(...)` ou transformações), você pode registrá-lo.

* **`tableEnv.createTemporaryView(String path, Table view)`:**
    * Registra um objeto `Table` existente como uma *view* temporária.
    * O "path" é o nome da *view* (e.g., "MyView" ou "my_database.MyView").
    * Essas *views* são válidas apenas para a sessão atual e não são persistidas em catálogos externos. Elas residem no catálogo em memória do Flink.
    * Apesar do nome `createTemporaryView`, este é o método principal para tornar um objeto `Table` (que pode ser uma tabela base ou o resultado de uma query da Table API) consultável por SQL.

    ```java
    import org.apache.flink.table.api.*;
    import org.apache.flink.types.Row;
    // import static org.apache.flink.table.api.Expressions.*; // Se for usar expressões na criação da tabela

    // Supondo tableEnv já inicializado
    // Table sourceTable = tableEnv.fromValues(
    //     DataTypes.ROW(DataTypes.FIELD("id", DataTypes.INT()), DataTypes.FIELD("name", DataTypes.STRING())),
    //     Row.of(1, "Alice"),
    //     Row.of(2, "Bob")
    // ); // .as("id", "name") é desnecessário se os nomes dos campos são definidos em DataTypes.FIELD

    // Registrando a 'sourceTable' como uma temporary view chamada "InputView"
    // tableEnv.createTemporaryView("InputView", sourceTable);

    // Agora "InputView" pode ser usada em SQL
    // try {
    //     Table resultFromSql = tableEnv.sqlQuery("SELECT UPPER(name) AS upper_name FROM InputView WHERE id > 1");
    //     System.out.println("\nResultado do SQL em InputView:");
    //     resultFromSql.execute().print();
    // } catch (Exception e) {
    //     e.printStackTrace();
    // }
    ```

**2. Usando `TableDescriptor` (para `Connectors`):**

Você pode registrar tabelas que se conectam a sistemas externos usando um `TableDescriptor`.

* **`tableEnv.createTemporaryTable(String path, TableDescriptor descriptor)`:** Registra uma tabela temporária baseada em um conector.
* **`tableEnv.createTable(String path, TableDescriptor descriptor)`:** Registra uma tabela no catálogo atual. Se o catálogo for persistente (e.g., `HiveCatalog`), a tabela será persistida.

    ```java
    // import org.apache.flink.table.api.DataTypes;
    // import org.apache.flink.table.api.Schema;
    // import org.apache.flink.table.api.TableDescriptor;

    // Supondo tableEnv já inicializado
    // TableDescriptor myFileSystemTableDescriptor = TableDescriptor.forConnector("filesystem")
    //     .schema(Schema.newBuilder()
    //         .column("word", DataTypes.STRING())
    //         .column("count", DataTypes.BIGINT())
    //         .build())
    //     .option("path", "/tmp/my_flink_data") // Use um caminho acessível para teste
    //     .option("source.monitor-interval", "1 s") // Para streaming de filesystem
    //     .format("csv") // Formato dos dados
    //     .build();

    // Crie um arquivo no caminho /tmp/my_flink_data/some_file.csv com conteúdo como:
    // hello,1
    // world,2

    // Registra uma tabela temporária
    // tableEnv.createTemporaryTable("MyCsvSource", myFileSystemTableDescriptor);
    // Table csvTable = tableEnv.from("MyCsvSource");
    // System.out.println("\nSchema de MyCsvSource:");
    // csvTable.printSchema();
    // System.out.println("Dados de MyCsvSource (pode não imprimir em streaming sem .execute().print() em uma query):");
    // tableEnv.sqlQuery("SELECT * FROM MyCsvSource").execute().print(); // Para testar
    ```

**3. Usando DDL SQL (`CREATE TABLE`):**

Esta é frequentemente a maneira mais conveniente de definir e registrar tabelas, especialmente aquelas conectadas a sistemas externos.

```sql
-- Executado via tableEnv.executeSql(...)
CREATE TABLE MyKafkaTable (
  user_id BIGINT,
  item_id STRING,
  event_time TIMESTAMP(3),
  WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
  'connector' = 'kafka',
  'topic' = 'user_events_topic', -- Nome do tópico
  'properties.bootstrap.servers' = 'localhost:9092', -- Endereço do Kafka
  'properties.group.id' = 'my-flink-consumer-group', -- Group ID
  'scan.startup.mode' = 'earliest-offset', -- Modo de início da leitura
  'format' = 'json', -- Formato da mensagem
  'json.fail-on-missing-field' = 'false' -- Tolerar campos ausentes no JSON
);
```
   ```java
   // Supondo tableEnv já inicializado
   // String kafkaDDL = "CREATE TABLE MyKafkaTable (" +
   //                   "  user_id BIGINT, " +
   //                   "  item_id STRING, " +
   //                   "  event_time TIMESTAMP(3), " +
   //                   "  WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND " +
   //                   ") WITH ( " +
   //                   "  'connector' = 'kafka', " +
   //                   "  'topic' = 'user_events_topic', " +
   //                   "  'properties.bootstrap.servers' = 'localhost:9092', " +
   //                   "  'properties.group.id' = 'my-flink-consumer-group', " +
   //                   "  'scan.startup.mode' = 'earliest-offset', " +
   //                   "  'format' = 'json', " +
   //                   "  'json.fail-on-missing-field' = 'false' " +
   //                   ")";
   // tableEnv.executeSql(kafkaDDL);
   // System.out.println("\nTabela MyKafkaTable registrada. Verificando tabelas:");
   // System.out.println(Arrays.toString(tableEnv.listTables())); // Deve listar MyKafkaTable
   // Table kafkaTable = tableEnv.from("MyKafkaTable");
   // kafkaTable.printSchema();
   ```

---

### Executando Queries SQL

Uma vez que suas tabelas estão registradas (ou você tem *views* definidas), você pode executar queries SQL sobre elas.

**1. `tableEnv.sqlQuery(String sqlQuery)`:**

* Usado para executar queries SQL `SELECT` que produzem um resultado.
* Retorna um objeto `Table`, que representa o resultado da query.
* Este objeto `Table` pode então ser usado em operações subsequentes da Table API, convertido para um `DataStream`, ou usado para popular outra tabela (via `executeInsert()`).

    ```java
    // import org.apache.flink.table.api.*;
    // import org.apache.flink.types.Row;
    // import static org.apache.flink.table.api.Expressions.$;

    // Supondo tableEnv já inicializado
    // Table source = tableEnv.fromValues(
    //     DataTypes.ROW(
    //         DataTypes.FIELD("id", DataTypes.INT()),
    //         DataTypes.FIELD("name", DataTypes.STRING()),
    //         DataTypes.FIELD("score", DataTypes.INT())
    //     ),
    //     Row.of(1, "Alice", 100),
    //     Row.of(2, "Bob", 200),
    //     Row.of(3, "Charlie", 150)
    // );
    // tableEnv.createTemporaryView("Players", source);

    // Executar uma query SQL SELECT
    // Table highScorers = tableEnv.sqlQuery(
    //     "SELECT name, score FROM Players WHERE score > 120 ORDER BY score DESC"
    // );

    // System.out.println("\nJogadores com score > 120 (via SQL):");
    // highScorers.execute().print();

    // 'highScorers' pode ser usado em operações da Table API
    // Table furtherProcessed = highScorers.select($("name").upperCase().as("upper_name"));
    // System.out.println("\nNomes em maiúsculas (Table API sobre resultado SQL):");
    // furtherProcessed.execute().print();
    ```

**2. `tableEnv.executeSql(String sqlStatement)`:**

* Usado para executar instruções SQL que **não** retornam um resultado na forma de um objeto `Table` diretamente para manipulação programática. Isso inclui:
    * DDL: `CREATE TABLE`, `DROP TABLE`, `ALTER TABLE`, `CREATE VIEW`, `DROP VIEW`.
    * DML: `INSERT INTO`, `INSERT OVERWRITE` (para *sinks*).
    * DQL (`SELECT`): Se você usar `tableEnv.executeSql("SELECT ...")`, ele retorna um `TableResult`. Você pode então chamar `.print()` neste `TableResult` para exibir os dados no console (útil para depuração ou demonstração).
    * Comandos de Catálogo: `USE CATALOG`, `USE DATABASE`, `SHOW TABLES`, etc.
* Retorna um objeto `TableResult`, que pode ser usado para obter informações sobre a execução do job (como o `JobClient`) ou para imprimir resultados.

    ```java
    // import org.apache.flink.table.api.*;
    // import org.apache.flink.types.Row; // Para Row.of

    // Supondo tableEnv já inicializado
    // Exemplo 1: Criar uma tabela sink usando DDL
    // tableEnv.executeSql(
    //     "CREATE TABLE OutputSink (" +
    //     "  player_name STRING, " +
    //     "  total_score BIGINT " +
    //     ") WITH (" +
    //     "  'connector' = 'print'" +
    //     ")"
    // );

    // Exemplo 2: Usar INSERT INTO com SELECT (DML)
    // String createSourceSQL = "CREATE TEMPORARY TABLE SourcePlayers (" +
    //         "  id INT, " +
    //         "  name STRING, " +
    //         "  score INT" +
    //         ") WITH ('connector' = 'values', 'data-id' = '" +
    //         // Para gerar data-id programaticamente para dados embutidos (requer flink-table-common)
    //         org.apache.flink.table.data. આપણેValuesData.generateDataId(java.util.Arrays.asList(
    //             Row.of(1, "Alice", 100), Row.of(2, "Bob", 200), Row.of(1, "Alice", 50)
    //         )) + "')";
    // tableEnv.executeSql(createSourceSQL);

    // TableResult insertResult = tableEnv.executeSql(
    //     "INSERT INTO OutputSink " +
    //     "SELECT name, SUM(score) FROM SourcePlayers GROUP BY name"
    // );
    // System.out.println("\nINSERT INTO submetido. Job ID (se aplicável): " + insertResult.getJobClient().map(jc -> jc.getJobID().toString()).orElse("N/A"));
    // O conector 'print' da OutputSink irá imprimir os resultados quando o job rodar.
    // Para ver a saída em um exemplo simples, o job precisa ser executado. Em um StreamExecutionEnvironment, seria com env.execute().
    // Com Table API pura, a chamada a executeSql para INSERT dispara a execução.

    // Exemplo 3: Executar um SELECT e imprimir com .print() de TableResult
    // TableResult selectAndPrintResult = tableEnv.executeSql("SELECT name, score FROM SourcePlayers WHERE score > 100");
    // System.out.println("\nResultado do SELECT direto com executeSql().print():");
    // selectAndPrintResult.print();
    ```

---

### Criando `Views`

Uma `View` é essencialmente uma query SQL armazenada que pode ser referenciada como uma tabela. Ela não armazena dados fisicamente; em vez disso, a query que define a *view* é executada (ou expandida) toda vez que a *view* é usada em outra query.

**Benefícios das `Views`:**

* **Simplificação:** Quebrar queries complexas em partes menores e mais gerenciáveis.
* **Reutilização:** Definir uma lógica de transformação uma vez e usá-la em múltiplas queries.
* **Abstração:** Esconder a complexidade das tabelas subjacentes ou das queries de origem.

**Tipos de `Views` no Flink:**

1.  **`Temporary View` (criada com `CREATE TEMPORARY VIEW` em SQL ou `tableEnv.createTemporaryView()` na Table API):**
    * **Escopo:** Válida apenas para a sessão atual do `TableEnvironment`.
    * **Persistência:** Não é armazenada em catálogos externos; reside no catálogo em memória do Flink.
    * **Uso Comum:** Para modularizar queries dentro de uma única aplicação Flink.

    ```sql
    -- SQL DDL para criar uma Temporary View
    -- Supondo que a tabela 'Players' já existe
    CREATE TEMPORARY VIEW HighScorePlayersView AS
    SELECT name, score
    FROM Players -- 'Players' deve ser uma tabela ou view existente
    WHERE score > 150;

    -- Agora podemos consultar a view
    -- SELECT name FROM HighScorePlayersView ORDER BY score DESC;
    ```
    ```java
    // Supondo tableEnv e a tabela 'Players' registrados
    // tableEnv.executeSql(
    //     "CREATE TEMPORARY VIEW HighScorePlayersViewSQL AS " +
    //     "SELECT name, score FROM Players WHERE score > 150"
    // );
    // tableEnv.sqlQuery("SELECT name FROM HighScorePlayersViewSQL ORDER BY score DESC").execute().print();

    // Criando a mesma Temporary View programaticamente a partir de um objeto Table
    // Table playersTable = tableEnv.from("Players");
    // Table highScores = playersTable.filter($("score").isGreater(150)).select($("name"), $("score"));
    // tableEnv.createTemporaryView("HighScorePlayersViewProgrammatic", highScores);
    // tableEnv.sqlQuery("SELECT name FROM HighScorePlayersViewProgrammatic ORDER BY score DESC").execute().print();
    ```

2.  **`View` (criada com `CREATE VIEW` em SQL, persistida no catálogo):**
    * **Escopo:** Se o catálogo atual suportar persistência (e.g., `HiveCatalog`, `JdbcCatalog`), a *view* é armazenada no catálogo e pode ser usada em diferentes sessões e até por outras ferramentas que acessam o mesmo catálogo.
    * **Persistência:** A definição da *view* é salva no catálogo.

    ```sql
    -- SQL DDL para criar uma View persistente (requer um catálogo que suporte, e.g., HiveCatalog)
    -- Supondo que estamos usando um catálogo como 'my_hive_catalog' e database 'my_hive_db'
    -- CREATE VIEW my_hive_catalog.my_hive_db.PersistentHighScoreView AS
    -- SELECT name, score
    -- FROM my_hive_catalog.my_hive_db.PlayersTable -- Nome completo da tabela base
    -- WHERE score > 150;
    ```

---

### Misturando Table API e SQL

A beleza da abordagem unificada do Flink é a capacidade de alternar entre a Table API e o SQL de forma transparente.

* **De Table API para SQL:**
    1. Crie ou manipule um objeto `Table` usando a Table API.
    2. Registre este objeto `Table` como uma *temporary view* usando `tableEnv.createTemporaryView("myViewName", myTableObject)`.
    3. Agora você pode referenciar `myViewName` em suas queries `tableEnv.sqlQuery(...)` ou `tableEnv.executeSql(...)`.

* **De SQL para Table API:**
    1. Execute uma query `SELECT` usando `tableEnv.sqlQuery("SELECT ...")`.
    2. Isso retorna um objeto `Table`.
    3. Você pode então aplicar mais transformações neste objeto `Table` usando os métodos da Table API (e.g., `.select()`, `.filter()`, `.join()`, etc.).

**Fluxo de Trabalho Comum:**

1.  **Definir Fontes:** Use DDL SQL (`CREATE TABLE ... WITH ('connector' = ...)`) para definir suas tabelas de origem conectadas a sistemas externos.
2.  **Pré-processamento/Transformações Iniciais:** Use SQL ou Table API, registrando resultados intermediários como *temporary views*.
3.  **Lógica de Negócios Principal:** Combine `Tables` e *views*, misturando SQL e Table API.
4.  **Definir *Sinks*:** Use DDL SQL para tabelas de *sink*.
5.  **Inserir Resultados:** Use `INSERT INTO mySinkTable SELECT ...` ou `myFinalTable.executeInsert("mySinkTable")`.

**Exemplo de Fluxo Misto:**
```java
// import org.apache.flink.table.api.*;
// import org.apache.flink.table.functions.ScalarFunction; // Para UDF
// import org.apache.flink.types.Row;
// import static org.apache.flink.table.api.Expressions.*;

// public class MixedPipelineExample {
//     public static class ParsePayloadFunction extends ScalarFunction { // UDF de exemplo
//         public String eval(String payload, String field) {
//             if (payload == null || field == null || !payload.startsWith("{") || !payload.endsWith("}")) return null;
//             String search = "\"" + field + "\":\"";
//             int start = payload.indexOf(search);
//             if (start == -1) return null;
//             start += search.length();
//             int end = payload.indexOf("\"", start);
//             if (end == -1) return null;
//             return payload.substring(start, end);
//         }
//     }

//     public static void main(String[] args) throws Exception {
//         EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().build();
//         TableEnvironment tableEnv = TableEnvironment.create(settings);

//         // 1. Definir fonte com Table API
//         Table raw_events = tableEnv.fromValues(
//             DataTypes.ROW(
//                 DataTypes.FIELD("user_id", DataTypes.INT()),
//                 DataTypes.FIELD("event_type", DataTypes.STRING()),
//                 DataTypes.FIELD("payload", DataTypes.STRING())
//             ),
//             Row.of(1, "click", "{\"page\":\"home\"}"),
//             Row.of(2, "view", "{\"item\":\"A\"}"),
//             Row.of(1, "purchase", "{\"item\":\"B\", \"amount\":100}")
//         );

//         // 2. Registrar como temporary view para uso em SQL
//         tableEnv.createTemporaryView("RawEventsView", raw_events);

//         // 3. Usar SQL para uma transformação inicial
//         Table clicks = tableEnv.sqlQuery(
//             "SELECT user_id, payload FROM RawEventsView WHERE event_type = 'click'"
//         );

//         // 4. Registrar e usar UDF com Table API no resultado do SQL
//         tableEnv.createTemporarySystemFunction("parseJsonField", new ParsePayloadFunction());
//         Table clicksWithPage = clicks.select(
//             $("user_id"),
//             call("parseJsonField", $("payload"), lit("page")).as("page_clicked")
//         );
//         System.out.println("\nCliques com página parseada (Table API sobre SQL):");
//         clicksWithPage.execute().print();

//         // 5. Definir sink com DDL e inserir usando SQL (requer que clicksWithPage seja uma view)
//         tableEnv.createTemporaryView("ClicksWithPageView", clicksWithPage);
//         tableEnv.executeSql(
//             "CREATE TABLE ClickPagesSink (" +
//             "  uid INT, " +
//             "  page STRING " +
//             ") WITH ('connector' = 'print')"
//         );
//         tableEnv.executeSql("INSERT INTO ClickPagesSink SELECT user_id, page_clicked FROM ClicksWithPageView")
//             .await(); // Em um job real, await() ou similar pode ser necessário para garantir a execução.
//     }
// }
```

---

### Table API/SQL em Ambientes Gerenciados: Confluent Cloud for Apache Flink

À medida que o Apache Flink ganha popularidade, plataformas de nuvem e serviços gerenciados como o Confluent Cloud for Apache Flink (às vezes referido como ccFlink) surgem para simplificar a implantação, o gerenciamento e a escalabilidade de aplicações Flink. Ao usar a Table API e o SQL nesses ambientes, existem algumas considerações e potenciais diferenças em relação a uma configuração Flink auto-gerenciada.

**Nota Importante:** O Confluent Cloud for Apache Flink é um produto comercial da Confluent. As características, funcionalidades, limitações e melhores práticas estão sujeitas a alterações e são continuamente atualizadas. **Consulte sempre a documentação oficial da Confluent para obter as informações mais recentes e precisas.**

**1. Visão Geral do Confluent Cloud for Apache Flink**

O Confluent Cloud for Apache Flink é um serviço totalmente gerenciado que permite executar aplicações de processamento de *stream* escritas com Flink SQL ou DataStream API sem a necessidade de gerenciar a infraestrutura subjacente do Flink (como *clusters*, *JobManagers*, *TaskManagers*). Ele se integra nativamente com outros serviços da Confluent Cloud, como Apache Kafka e Schema Registry.

**2. Usando Table API e SQL no Confluent Cloud for Apache Flink**

Normalmente, você interage com o Flink SQL no Confluent Cloud através de:

* **SQL Workspace / Studio da Confluent:** Uma interface web para desenvolver, testar e executar queries Flink SQL diretamente.
* **Submissão de Jobs Flink:** Você pode empacotar aplicações Flink (que podem usar Table API programaticamente ou executar SQL) como JARs e submetê-las ao serviço gerenciado.
* **Ferramentas CLI ou APIs:** A Confluent pode fornecer CLIs ou APIs para gerenciar e submeter suas queries ou computações Flink.

A Table API programática geralmente é usada dentro de um job Flink que você empacota e submete, enquanto o Flink SQL pode ser mais diretamente utilizado no SQL Workspace.

**3. Principais Considerações e Diferenças (Perspectiva da Table API/SQL)**

Ao trabalhar com Table API/SQL no Confluent Cloud for Flink, espere as seguintes características e possíveis diferenças:

* **Gerenciamento de Catálogo e Schema Registry:**
    * **Integração com Schema Registry:** Uma das grandes vantagens é a integração nativa e profunda com o Confluent Schema Registry. Ao definir tabelas conectadas a tópicos Kafka que usam Avro, Protobuf ou JSON Schema, o Flink no Confluent Cloud pode automaticamente usar o Schema Registry para ler e escrever esquemas, garantindo a governança e evolução dos dados.
    * **Catálogos:** O catálogo padrão pode ser o `GenericInMemoryCatalog`, mas o Confluent Cloud pode oferecer ou configurar automaticamente catálogos que facilitam a integração com seus serviços. Por exemplo, pode haver maneiras simplificadas de registrar tabelas do Kafka gerenciado pela Confluent.
    * A persistência de metadados de tabelas e *views* (além das temporárias) dependerá do suporte de catálogo oferecido e configurado no ambiente.

* **Conectores (`Connectors`):**
    * **Conectores Otimizados:** Espere conectores otimizados e bem integrados para os serviços da Confluent Cloud (especialmente Kafka).
    * **Disponibilidade:** A maioria dos conectores Flink *open-source* populares deve estar disponível. No entanto, a forma de incluir e gerenciar JARs de conectores customizados ou menos comuns pode ser específica da plataforma (e.g., upload de JARs através da UI ou CLI da Confluent).
    * **Configuração:** A configuração de conectores (e.g., autenticação com Kafka) será gerenciada através das configurações e segredos da Confluent Cloud, abstraindo algumas complexidades.

* **Funções Definidas pelo Usuário (UDFs):**
    * **Empacotamento e Registro:** UDFs escritas em Java (ou outras linguagens JVM) precisarão ser empacotadas em um JAR. O processo de disponibilizar este JAR para o ambiente Flink gerenciado e registrar as UDFs (seja via DDL SQL `CREATE FUNCTION` ou programaticamente em um job) será específico da plataforma Confluent Cloud.
    * **Limitações Potenciais:** Em ambientes gerenciados, pode haver restrições de segurança ou recursos sobre o que as UDFs podem fazer (e.g., acesso à rede, acesso ao sistema de arquivos local).

* **Ambiente de Execução e Versão do Flink:**
    * **Abstração:** Você não gerencia diretamente os *clusters* Flink. A Confluent lida com o provisionamento, escalonamento e manutenção.
    * **Versão do Flink:** O Confluent Cloud oferecerá suporte a versões específicas do Apache Flink. Você precisará desenvolver sua aplicação com base nessas versões suportadas. Atualizações para novas versões do Flink são gerenciadas pela Confluent.

* **SQL Dialect e Features:**
    * O objetivo é suportar o dialeto Flink SQL padrão. No entanto, podem existir otimizações específicas da plataforma ou, em raras ocasiões, um subconjunto de funcionalidades durante fases iniciais de adoção de novas *features* do Flink *open-source*.

**4. Limitações Potenciais e Melhores Práticas (baseado em informações até minha última atualização, sempre verifique a documentação atual)**

* **Sempre Verifique a Documentação Oficial:** Este é o ponto mais crucial. A plataforma evolui rapidamente.
* **Customização do Ambiente:** Ambientes gerenciados geralmente oferecem menos flexibilidade para customizações profundas do ambiente Flink (e.g., configurações de JVM muito específicas, instalação de bibliotecas no nível do sistema) em comparação com uma configuração auto-gerenciada.
* **Recursos e Cotas:** O uso de recursos (CPUs, memória, estado) será gerenciado dentro dos limites e cotas do seu plano de serviço na Confluent Cloud.
* **Acesso a APIs de Baixo Nível:** O acesso direto a APIs de muito baixo nível do Flink ou ao sistema de arquivos do *cluster* pode ser restrito por razões de segurança e operacionalidade.
* **Diagnóstico e Monitoramento:** As ferramentas de diagnóstico e monitoramento serão aquelas fornecidas pela Confluent Cloud, que podem ser diferentes das ferramentas Flink *open-source* padrão (como a Flink Web UI), embora geralmente forneçam integrações e visualizações ricas.
* **Melhores Práticas:**
    * Aproveite ao máximo a integração com o ecossistema Confluent (Kafka, Schema Registry, ksqlDB se aplicável).
    * Para UDFs e conectores customizados, siga as diretrizes da Confluent para empacotamento e upload de JARs.
    * Monitore seus jobs Flink usando as ferramentas fornecidas pela Confluent Cloud.
    * Entenda o modelo de precificação e como o uso de recursos Flink (como tamanho do estado, paralelismo) impacta os custos.

**Conclusão da Seção:**

Usar a Table API e o SQL no Confluent Cloud for Apache Flink pode acelerar significativamente o desenvolvimento e a implantação de aplicações de processamento de *stream*, removendo o fardo da gestão de infraestrutura. No entanto, é essencial entender as especificidades da plataforma, especialmente em relação à integração com outros serviços Confluent, gerenciamento de dependências (conectores, UDFs) e quaisquer limitações ou abstrações impostas pelo ambiente gerenciado. A documentação da Confluent será sempre sua melhor amiga.

---

### Exercícios

1.  **Registro e Consulta SQL:**
    * Crie uma `Table` chamada `Products` usando `tableEnv.fromValues()` com as colunas `id` (INT), `name` (STRING), `price` (DOUBLE).
    * Registre esta `Table` como uma *temporary view* chamada "ProductView".
    * Escreva e execute uma query SQL para selecionar o nome e o preço dos produtos com `price` maior que 50.0.

2.  **Criação de `View` SQL:**
    * Usando a "ProductView" do exercício anterior, crie uma nova *temporary view* SQL chamada "ExpensiveProductNamesView" que contenha apenas os nomes dos produtos com `price` maior que 100.0.
    * Escreva uma query SQL para selecionar todos os nomes da "ExpensiveProductNamesView".

3.  **Fluxo Misto:**
    * Crie uma `Table` `Orders` com `order_id` (INT) e `product_id` (INT).
    * Crie uma `Table` `ProductDetails` com `product_id` (INT) e `product_name` (STRING).
    * Registre ambas como *temporary views*.
    * Use uma query SQL para fazer um `INNER JOIN` entre `Orders` e `ProductDetails` em `product_id` para obter `order_id` e `product_name`. Chame o resultado de `JoinedView`.
    * Converta o resultado (`JoinedView`) de volta para um objeto `Table` da Table API.
    * Na Table API, filtre esta `Table` para um `product_name` específico e imprima o resultado.

4.  **`sqlQuery` vs `executeSql`:** Qual método do `TableEnvironment` você usaria para executar uma instrução `CREATE TABLE ... WITH (...)`?
    * a) `tableEnv.sqlQuery(...)`
    * b) `tableEnv.executeSql(...)`
    * c) `tableEnv.createTable(...)`
    * d) `tableEnv.registerTable(...)`

5.  **Escopo de `Temporary Views`:** Se você cria uma `Temporary View` usando `tableEnv.createTemporaryView("myView", tableObject)`, onde esta *view* está disponível?
    * a) Em todas as sessões Flink e é persistida no Hive Metastore.
    * b) Apenas na sessão atual do `TableEnvironment` e reside no catálogo em memória.
    * c) Apenas no *database* atual do catálogo em memória.
    * d) Fica disponível globalmente para outros clusters Flink.

---

A capacidade de misturar SQL e a Table API programática oferece grande flexibilidade e poder. No próximo módulo, exploraremos alguns conceitos avançados da Table API, como tabelas dinâmicas e tabelas temporais.