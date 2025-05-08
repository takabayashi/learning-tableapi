## Módulo 2: Conceitos Fundamentais da Table API

Bem-vindo ao Módulo 2! Neste módulo, vamos aprofundar nos conceitos centrais da Table API do Apache Flink. Abordaremos o `TableEnvironment`, as diversas formas de criar `Tables` (incluindo a definição de seus `schemas`) e as operações básicas que você pode realizar sobre elas, como `projection`, `filtering` e `renaming`.

---

### `TableEnvironment`: O Ponto de Partida

O `TableEnvironment` é a classe central e o ponto de entrada para todas as funcionalidades da Table API e SQL do Flink. Ele é usado para:

* Registrar `Tables` e `Views`.
* Executar consultas SQL.
* Converter `DataStreams` ou `DataSets` em `Tables`.
* Converter `Tables` de volta para `DataStreams` ou `DataSets`.
* Configurar propriedades de execução e otimização.
* Registrar funções definidas pelo usuário (UDFs).
* Gerenciar o catálogo de metadados.

**Obtendo uma Instância do `TableEnvironment`:**

Existem algumas maneiras de obter uma instância do `TableEnvironment`, dependendo do seu caso de uso:

1.  **Para Aplicações Puras de Table API/SQL (Recomendado):**
    Use `EnvironmentSettings` para definir o modo de execução (*streaming* ou *batch*) e o *planner* desejado.

    ```java
    import org.apache.flink.table.api.EnvironmentSettings;
    import org.apache.flink.table.api.TableEnvironment;

    // Para modo de streaming
    EnvironmentSettings settings = EnvironmentSettings.newInstance()
        .inStreamingMode()
        .build();
    TableEnvironment tableEnv = TableEnvironment.create(settings);

    // Para modo de batch (exemplo)
    // EnvironmentSettings batchSettings = EnvironmentSettings.newInstance()
    //    .inBatchMode()
    //    .build();
    // TableEnvironment batchTableEnv = TableEnvironment.create(batchSettings);
    ```

2.  **Para Aplicações que Mesclam DataStream API e Table API:**
    Se você precisa de interoperabilidade com a `DataStreamAPI` (por exemplo, converter um `DataStream` existente para uma `Table` ou vice-versa), você deve usar o `StreamTableEnvironment`.

    ```java
    import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
    import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
    import org.apache.flink.table.api.EnvironmentSettings;

    // Configuração básica do StreamExecutionEnvironment
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

    // Criando o StreamTableEnvironment a partir do StreamExecutionEnvironment
    // Usar EnvironmentSettings é opcional aqui se as configurações padrão do planner forem suficientes
    // EnvironmentSettings streamingSettings = EnvironmentSettings.newInstance().inStreamingMode().build();
    // StreamTableEnvironment streamTableEnv = StreamTableEnvironment.create(env, streamingSettings);

    // Forma mais comum e simples quando já se tem o 'env'
    StreamTableEnvironment streamTableEnv = StreamTableEnvironment.create(env);
    ```

**Configurações Importantes:**

O `TableEnvironment` permite configurar diversos aspectos da execução através de sua `TableConfig`.
```java
import java.time.ZoneId;
// Exemplo de acesso à TableConfig (supondo que 'tableEnv' já foi criado)
// tableEnv.getConfig().setLocalTimeZone(ZoneId.of("America/Sao_Paulo"));
```
Iremos explorar configurações mais avançadas em módulos posteriores.

---

### Criando `Tables`

Uma `Table` no Flink é uma representação lógica de dados estruturados. Ela possui um `schema` bem definido, composto por nomes e tipos de colunas. `Tables` podem ser "dinâmicas" (*dynamic tables*), o que significa que elas mudam ao longo do tempo em aplicações de *streaming*.

**1. A Partir de Coleções em Memória (`fromValues`)**

Útil para testes, prototipagem ou para dados pequenos e fixos.

```java
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.TableEnvironment;
import org.apache.flink.table.api.DataTypes;
import org.apache.flink.types.Row;
import static org.apache.flink.table.api.Expressions.$; // Para usar a sintaxe $("")

// Supondo que tableEnv já foi inicializado como TableEnvironment.create(EnvironmentSettings.inStreamingMode().build());

Table simpleTable = tableEnv.fromValues(
    DataTypes.ROW( // Definindo o tipo da linha explicitamente
        DataTypes.FIELD("name", DataTypes.STRING()),
        DataTypes.FIELD("score", DataTypes.INT()),
        DataTypes.FIELD("event_time_long", DataTypes.BIGINT()) // Usando BIGINT para o timestamp longo
    ),
    Row.of("Alice", 12, System.currentTimeMillis()),
    Row.of("Bob", 10, System.currentTimeMillis() - 3600_000L), // 1 hora atrás
    Row.of("Charlie", 7, System.currentTimeMillis() - 7200_000L) // 2 horas atrás
);
// Note que .as() não é mais usado com fromValues quando DataTypes.ROW é fornecido.
// Os nomes dos campos são definidos em DataTypes.FIELD.

System.out.println("Schema da simpleTable:");
simpleTable.printSchema();
System.out.println("\nDados da simpleTable:");
simpleTable.execute().print();
```

**2. A Partir de `DataStreams` (Interoperabilidade)**

Você pode converter um `DataStream` de POJOs (Plain Old Java Objects), Tuples do Flink, ou `Row` em uma `Table`. Esta operação requer um `StreamTableEnvironment`.

* **Com POJOs:** O Flink infere o `schema` a partir dos campos do POJO.

    ```java
    import org.apache.flink.streaming.api.datastream.DataStream;
    import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
    import org.apache.flink.table.api.Table;
    import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
    import static org.apache.flink.table.api.Expressions.$;

    // Definição do POJO
    public static class UserActivity {
        public String userId;
        public String pageUrl;
        public Long visitTime; // epoch milliseconds

        // Construtor vazio é necessário para o Flink
        public UserActivity() {}

        public UserActivity(String userId, String pageUrl, Long visitTime) {
            this.userId = userId;
            this.pageUrl = pageUrl;
            this.visitTime = visitTime;
        }

        @Override
        public String toString() {
            return "UserActivity{" +
                   "userId='" + userId + '\'' +
                   ", pageUrl='" + pageUrl + '\'' +
                   ", visitTime=" + visitTime +
                   '}';
        }
    }

    // Configuração do ambiente
    // StreamExecutionEnvironment execEnv = StreamExecutionEnvironment.getExecutionEnvironment();
    // StreamTableEnvironment TEnv = StreamTableEnvironment.create(execEnv);

    // DataStream<UserActivity> userStream = execEnv.fromElements(
    //     new UserActivity("user1", "/home", System.currentTimeMillis()),
    //     new UserActivity("user2", "/product", System.currentTimeMillis() - 10000L),
    //     new UserActivity("user1", "/cart", System.currentTimeMillis() - 20000L)
    // );

    // Convertendo o DataStream para Table (inferência de schema a partir do POJO)
    // Table userActivityTable = TEnv.fromDataStream(userStream);

    // System.out.println("\nSchema da userActivityTable (inferido do POJO):");
    // userActivityTable.printSchema();
    // System.out.println("\nDados da userActivityTable:");
    // userActivityTable.execute().print(); // Cuidado: em um job real, isso é um sink de teste.

    // Convertendo com projeção e manipulação de campos (ex: definindo atributos de tempo)
    // Atributos de tempo como 'rowtime' serão abordados em detalhes no Módulo 4.
    // Table userActivityTableWithTime = TEnv.fromDataStream(
    //     userStream,
    //     $("userId"),
    //     $("pageUrl"),
    //     $("visitTime").as("ts"), // Renomeando campo
    //     $("proctime").proctime() // Adicionando uma coluna de tempo de processamento
    //      // Para tempo de evento: $("event_ts").rowtime() se 'visitTime' fosse o campo de tempo de evento
    // );
    // System.out.println("\nSchema da userActivityTableWithTime (com proctime):");
    // userActivityTableWithTime.printSchema();
    // System.out.println("\nDados da userActivityTableWithTime:");
    // userActivityTableWithTime.execute().print();
    ```
    *Nota:* Para executar o código acima, você precisaria de um método `main` e descomentar as inicializações de `execEnv` e `TEnv`. O `execute().print()` é um *sink* que imprime no console e finaliza jobs de *streaming* finitos ou executa continuamente para os infinitos (cancelamento manual necessário).

* **Definindo o `Schema` Programaticamente com `Schema.newBuilder()`:**
    Oferece controle total sobre nomes, tipos e até mesmo *computed columns* ou *watermarks*.

    ```java
    import org.apache.flink.streaming.api.datastream.DataStream;
    import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
    import org.apache.flink.table.api.DataTypes;
    import org.apache.flink.table.api.Schema;
    import org.apache.flink.table.api.Table;
    import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
    import org.apache.flink.types.Row;
    import static org.apache.flink.table.api.Expressions.$;
    import static org.apache.flink.table.api.Expressions.lit; // Para literais

    // StreamExecutionEnvironment execEnvForSchema = StreamExecutionEnvironment.getExecutionEnvironment();
    // StreamTableEnvironment TEnvForSchema = StreamTableEnvironment.create(execEnvForSchema);

    // DataStream<Row> genericStream = execEnvForSchema.fromElements(
    //     Row.of("user3", "/cart", System.currentTimeMillis()),
    //     Row.of("user1", "/checkout", System.currentTimeMillis() - 5000L)
    // );

    // Table eventsTable = TEnvForSchema.fromDataStream(
    //     genericStream,
    //     Schema.newBuilder()
    //         .column("f0", DataTypes.STRING().notNull()) // Campos do Row são acessados por f0, f1, ...
    //         .column("f1", DataTypes.STRING())
    //         .column("f2", DataTypes.BIGINT())
    //         // Definindo 'computed columns' a partir dos campos originais
    //         .columnByExpression("user_id", $("f0"))
    //         .columnByExpression("url", $("f1"))
    //         .columnByExpression("ts_millis", $("f2"))
    //         // Convertendo para TIMESTAMP_LTZ e definindo como atributo de tempo
    //         // TO_TIMESTAMP_LTZ(numeric_epoch, precision)
    //         .columnByExpression("event_time", "TO_TIMESTAMP_LTZ(ts_millis, 3)")
    //         // Watermarks serão cobertos em detalhes no Módulo 4
    //         .watermark("event_time", "$('event_time') - INTERVAL '5' SECOND")
    //         .build()
    // )
    // // Selecionando apenas as colunas desejadas para a tabela final
    // .select($("user_id"), $("url"), $("event_time"));


    // System.out.println("\nSchema da eventsTable (com Schema.newBuilder):");
    // eventsTable.printSchema();
    // System.out.println("\nDados da eventsTable:");
    // eventsTable.execute().print();
    ```
    Aqui, `Schema.newBuilder()` é usado para dar nomes significativos aos campos `f0`, `f1`, `f2` de `Row`, converter um `BIGINT` para `TIMESTAMP_LTZ` e declarar um *watermark* (uma prévia do Módulo 4).

**3. A Partir de `DataSets` (Para Processamento em Batch Legado)**

De forma análoga ao `DataStream`, você pode converter `DataSets` para `Tables` usando um `BatchTableEnvironment`. A conversão é similar: `batchTableEnv.fromDataSet(...)`. Este curso foca na API unificada para *streaming* e *batch* através do `TableEnvironment` e `StreamTableEnvironment`.

**4. A Partir de `Connectors` (Fontes Externas)**

A maneira mais comum de popular `Tables` em produção é através de `connectors` que leem de sistemas externos como Apache Kafka, S3, bancos de dados JDBC, etc. Você define essas `Tables` usando SQL DDL ou programaticamente.

* **SQL DDL (Data Definition Language):**

    ```java
    // Supondo que tableEnv já foi inicializado
    // tableEnv.executeSql("DROP TABLE IF EXISTS Orders"); // Para remover se já existir em testes

    // Exemplo de DDL para uma Table conectada ao Kafka (requer o conector Kafka e dependências de formato como flink-json)
    // Detalhes sobre conectores e formatos no Módulo 8.
    // A execução deste DDL registra a 'Table' no catálogo do TableEnvironment.
    // Nenhuma leitura de dados ocorre até que uma query use esta tabela e um job seja submetido.
    tableEnv.executeSql(
        "CREATE TABLE Orders (\n" +
        "  `order_id` STRING,\n" +
        "  `user_id` STRING,\n" +
        "  `product` STRING,\n" +
        "  `amount` DECIMAL(10, 2),\n" +
        "  `order_time_str` STRING, -- Supondo que o tempo venha como string do Kafka JSON\n" +
        "  `order_time` AS TO_TIMESTAMP_LTZ(CAST(order_time_str AS BIGINT), 3), -- Computed column para converter\n" +
        "  WATERMARK FOR `order_time` AS `order_time` - INTERVAL '10' SECOND\n" +
        ") WITH (\n" +
        "  'connector' = 'kafka',\n" +
        "  'topic' = 'orders_topic',\n" +
        "  'properties.bootstrap.servers' = 'localhost:9092',\n" +
        "  'properties.group.id' = 'flink_orders_consumer',\n" +
        "  'scan.startup.mode' = 'earliest-offset',\n" +
        "  'format' = 'json',\n" +
        "  'json.fail-on-missing-field' = 'false',\n" +
        "  'json.ignore-parse-errors' = 'true'\n" +
        ")"
    );

    // Após a criação, você pode inspecionar o schema da Table registrada:
    // Table orders = tableEnv.from("Orders");
    // System.out.println("\nSchema da Tabela 'Orders' (registrada via DDL Kafka):");
    // orders.printSchema();
    // Para realmente processar dados desta tabela, você a usaria em uma query e executaria um sink:
    // orders.filter($("amount").isGreater(100)).executeInsert("some_sink_table");
    // ou para teste (requer Kafka rodando e produzindo dados):
    // orders.limit(5).execute().print(); // 'limit' é útil para não imprimir um stream infinito.
    ```
    Este DDL cria uma `Table` chamada `Orders`. O Flink usará o `connector` Kafka para ler dados. A coluna `order_time` é uma *computed column* derivada de `order_time_str`.

* **Programaticamente usando `TableDescriptor`:**

    ```java
    import org.apache.flink.table.api.TableDescriptor;
    import org.apache.flink.table.api.Schema;
    import org.apache.flink.table.api.DataTypes;
    // ... (tableEnv inicializado)

    // TableDescriptor kafkaOrdersDescriptor = TableDescriptor.forConnector("kafka")
    //     .schema(Schema.newBuilder()
    //         .column("order_id", DataTypes.STRING())
    //         .column("user_id", DataTypes.STRING())
    //         .column("product", DataTypes.STRING())
    //         .column("amount", DataTypes.DECIMAL(10, 2))
    //         .column("order_time_str", DataTypes.STRING()) // Tempo como string do JSON
    //         // Definindo uma computed column para o timestamp
    //         .columnByExpression("order_time", "TO_TIMESTAMP_LTZ(CAST(order_time_str AS BIGINT), 3)")
    //         .watermark("order_time", "$('order_time') - INTERVAL '10' SECOND")
    //         .build())
    //     .option("topic", "orders_topic_prog")
    //     .option("properties.bootstrap.servers", "localhost:9092")
    //     .option("properties.group.id", "flink_orders_consumer_prog")
    //     .option("scan.startup.mode", "earliest-offset")
    //     .format("json") // Para formatos mais complexos: .format(new Json().failOnMissingField(false)...)
    //     .build();

    // tableEnv.createTemporaryTable("ProgrammaticOrders", kafkaOrdersDescriptor);
    // Table progOrders = tableEnv.from("ProgrammaticOrders");
    // System.out.println("\nSchema da Tabela 'ProgrammaticOrders' (registrada via TableDescriptor):");
    // progOrders.printSchema();
    ```
    O `TableDescriptor` oferece uma forma programática de definir tabelas conectadas a sistemas externos, similar ao DDL.

---

### Operações Básicas em `Table`

Uma vez que você tem um objeto `Table`, você pode aplicar operações relacionais. Estas são declarativas e otimizadas pelo Flink antes da execução.

*(Reutilizando `simpleTable` do exemplo `fromValues` para as operações básicas)*
```java
// Certifique-se que simpleTable está disponível e populada como no exemplo fromValues:
Table simpleTable = tableEnv.fromValues(
    DataTypes.ROW(
        DataTypes.FIELD("name", DataTypes.STRING()),
        DataTypes.FIELD("score", DataTypes.INT()),
        DataTypes.FIELD("event_time_long", DataTypes.BIGINT())
    ),
    Row.of("Alice", 12, System.currentTimeMillis()),
    Row.of("Bob", 10, System.currentTimeMillis() - 3600_000L),
    Row.of("Charlie", 7, System.currentTimeMillis() - 7200_000L)
);
```

**1. `Projection` (`select`, `addOrReplaceColumns`, `dropColumns`)**

* **`select(Expression... fields)`:**

    ```java
    // Supondo 'simpleTable' com colunas: name, score, event_time_long
    Table projectedTable = simpleTable.select($("name"), $("score"));
    System.out.println("\nSchema após select('name', 'score'):");
    projectedTable.printSchema();
    System.out.println("Dados:");
    projectedTable.execute().print();

    Table calculatedTable = simpleTable.select(
        $("name").upperCase().as("NAME_UPPER"),
        $("score").plus(100).as("adjusted_score")
    );
    System.out.println("\nSchema após select com cálculo e alias:");
    calculatedTable.printSchema();
    System.out.println("Dados:");
    calculatedTable.execute().print();
    ```

* **`addOrReplaceColumns(Expression... fields)`:**

    ```java
    Table withNewColumn = simpleTable.addOrReplaceColumns(
        $("score").multipliedBy(2).as("doubled_score"), // Nova coluna
        $("name").lowerCase().as("name") // Substitui a coluna 'name'
    );
    System.out.println("\nSchema após addOrReplaceColumns:");
    withNewColumn.printSchema();
    System.out.println("Dados:");
    withNewColumn.execute().print();
    ```

* **`dropColumns(Expression... fields)`:**

    ```java
    Table withoutEventTime = simpleTable.dropColumns($("event_time_long"));
    System.out.println("\nSchema após dropColumns('event_time_long'):");
    withoutEventTime.printSchema();
    System.out.println("Dados:");
    withoutEventTime.execute().print();
    ```

**2. `Filtering` (`filter`, `where`)**

* **`filter(Expression predicate)` ou `where(Expression predicate)`:**

    ```java
    Table highScorePlayers = simpleTable.filter($("score").isGreater(10));
    System.out.println("\nJogadores com score > 10:");
    highScorePlayers.execute().print();

    Table aliceOnly = simpleTable.filter($("name").isEqual("Alice")
                                        .and($("score").isGreaterOrEqual(10)));
    System.out.println("\nApenas Alice com score >= 10:");
    aliceOnly.execute().print();
    ```

**3. `Renaming` (`as`, `renameColumns`)**

* **`as(String... fields)`:** Renomeia todas as colunas.

    ```java
    Table renamedAll = simpleTable.as("jogador", "pontuacao", "hora_evento_ms");
    System.out.println("\nSchema após .as('jogador', 'pontuacao', 'hora_evento_ms'):");
    renamedAll.printSchema();
    System.out.println("Dados:");
    renamedAll.execute().print();
    ```

* **`renameColumns(Expression... newFieldNames)`:** Renomeia colunas específicas.

    ```java
    Table renamedSpecific = simpleTable.renameColumns($("event_time_long").as("timestamp_ms"));
    System.out.println("\nSchema após renameColumns('event_time_long' para 'timestamp_ms'):");
    renamedSpecific.printSchema();
    System.out.println("Dados:");
    renamedSpecific.execute().print();
    ```

**Diagrama Conceitual das Operações Básicas:**
(Mesmo diagrama do Módulo 2 anterior, pois o conceito não mudou)
```
Fonte de Dados (e.g., Kafka, Collection, DataStream)
        |
        v
+-------------------+
| TableEnvironment  |  Cria/Registra
+-------------------+
        |
        v
+-------------------------------------------+
| Tabela Original (com schema completo)     |
| (e.g., col1, col2, col3, col4)            |
+-------------------------------------------+
        |
        |--- filter($("col2").isGreater(10)) --> +-----------------------------------+
        |                                       | Tabela Filtrada                   |
        |                                       | (mesmo schema, menos linhas)      |
        |                                       +-----------------------------------+
        |
        |--- select($("col1"), $("col3").as("nova_col3")) --> +-----------------------------+
        |                                                    | Tabela Projetada/Renomeada  |
        |                                                    | (schema: col1, nova_col3)   |
        |                                                    +-----------------------------+
        |
        v
Operações Posteriores / Sink
```

---

### Exercícios

Para os exercícios, crie um `TableEnvironment` (`TableEnvironment.create(EnvironmentSettings.inStreamingMode().build())`) e use `tableEnv.fromValues(...)` para as `Tables` de teste, como mostrado no exemplo `simpleTable` ajustado.

**1. Configuração do Ambiente:**
Qual é a maneira recomendada de instanciar um `TableEnvironment` para uma aplicação de *streaming* que usará primariamente a Table API e SQL, sem necessidade imediata de interoperar com `DataStreams`?
    * a) `new TableEnvironmentImpl()`
    * b) `StreamTableEnvironment.create(StreamExecutionEnvironment.getExecutionEnvironment())`
    * c) `TableEnvironment.create(EnvironmentSettings.newInstance().inStreamingMode().build())`
    * d) `TableEnvironment.create(new Configuration())`

**2. Criação de `Table` e `Schema`:**
Crie uma `Table` chamada `products` a partir de uma coleção de dados em memória (`fromValues`) com as seguintes colunas e dados:
    * `product_id` (STRING)
    * `product_name` (STRING)
    * `price` (DOUBLE)
    * `category` (STRING)

    Dados:
    * ("p001", "Laptop Pro", 1200.50, "Electronics")
    * ("p002", "Coffee Maker", 89.99, "Appliances")
    * ("p003", "Desk Chair", 150.00, "Furniture")
    * ("p004", "USB Cable", 9.99, "Electronics")

    Imprima o `schema` da `Table` e seus dados.
    *Dica: Use `DataTypes.ROW` com `DataTypes.FIELD` para definir o schema em `fromValues`.*

**3. `Projection` e `Renaming`:**
A partir da `Table` `products` criada no exercício anterior:
    * a) Selecione apenas as colunas `product_name` e `price`.
    * b) Na `Table` resultante da seleção, renomeie `product_name` para `item` e `price` para `cost` usando o método `as()`.
    * Imprima o `schema` e os dados da `Table` final.

**4. `Filtering`:**
A partir da `Table` `products` original:
    * a) Filtre os produtos que pertencem à categoria "Electronics". Imprima o resultado.
    * b) A partir do resultado anterior (produtos de "Electronics"), filtre aqueles que custam menos de 100.00. Imprima o resultado final.

**5. Operação Combinada:**
Qual das seguintes sequências de operações na Table API resultaria em uma `Table` contendo apenas os nomes (renomeados para `itemName`) de produtos da categoria "Electronics" com preço superior a 50.00?
    Suponha uma `Table` inicial `sourceProducts` com colunas `id` (STRING), `name` (STRING), `price` (DOUBLE), `category` (STRING).

    * a) `sourceProducts.filter($("category").isEqual("Electronics").and($("price").isGreater(50.00))).select($("name").as("itemName"))`
    * b) `sourceProducts.select($("name").as("itemName")).filter($("category").isEqual("Electronics").and($("price").isGreater(50.00)))`
    * c) `sourceProducts.filter($("category").isEqual("Electronics")).select($("name").as("itemName")).filter($("price").isGreater(50.00))`
    * d) `sourceProducts.select($("name")).as("itemName").filter($("category").isEqual("Electronics").and($("price").isGreater(50.00)))`

---

No próximo módulo, exploraremos as operações relacionais mais complexas, como junções (`joins`), agrupamentos (`group by`) e agregações.