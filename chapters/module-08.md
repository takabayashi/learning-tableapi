## Módulo 8: Conectando a Sistemas Externos

Bem-vindo ao Módulo 8! Uma aplicação de processamento de dados raramente existe isoladamente. Ela precisa consumir dados de sistemas de origem (bancos de dados, *message queues*, sistemas de arquivos) e persistir ou expor seus resultados para outros sistemas (bancos de dados, *dashboards*, APIs). O Apache Flink fornece um framework de conectores rico e extensível para essa finalidade.

Na Table API e SQL, os conectores são usados para definir `Table Sources` (de onde ler dados) e `Table Sinks` (para onde escrever dados).

Neste módulo, exploraremos:
* Uma visão geral dos conectores e formatos no Flink.
* Como definir `Table Sources` e `Sinks` usando a abordagem declarativa DDL SQL.
* Detalhes sobre conectores comuns como FileSystem, Apache Kafka e JDBC.
* Como definir `Table Sources` e `Sinks` programaticamente usando `TableDescriptor`.
* Modelo de Conectores no Confluent Cloud for Apache Flink.
* Considerações importantes ao trabalhar com conectores.

---

### Visão Geral dos Conectores no Flink SQL e Table API

O Flink possui uma arquitetura de conectores que permite a leitura e escrita de dados de e para uma variedade de sistemas externos. Quando usamos a Table API ou SQL, interagimos com esses sistemas como se fossem tabelas.

* **`Table Sources`:** Representam tabelas que leem dados de sistemas externos. Uma `Table Source` pode ler um *stream* de dados (e ser uma `Dynamic Table`) ou um conjunto de dados em *batch*.
* **`Table Sinks`:** Representam tabelas que escrevem dados para sistemas externos. Uma `Table Sink` pode receber um *stream* de dados contínuo (incluindo *appends*, *updates*, e *deletes*) ou dados em *batch*.

**Formatos (`Format`):**

Além do conector que lida com a comunicação com o sistema externo, você geralmente precisa especificar um **formato** que define como os dados são serializados e desserializados. O Flink suporta muitos formatos comuns:
* **JSON:** `format = 'json'`
* **Avro:** `format = 'avro'` (muitas vezes com integração com Schema Registry, e.g., `format = 'avro-confluent-registry'`)
* **CSV:** `format = 'csv'`
* **Parquet:** `format = 'parquet'`
* **ORC:** `format = 'orc'`
* **Debezium JSON, Canal JSON, Maxwell JSON:** Para consumir *changelog streams* de CDC.
* E outros...

**Dependências Maven:**

Cada conector e, frequentemente, cada formato, requer a adição de dependências específicas ao seu projeto Maven (ou Gradle). Por exemplo, para usar o conector Kafka com formato JSON (usando as dependências unificadas do Flink 1.17+):
```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-sql-connector-kafka</artifactId>
    <version>${flink.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-json</artifactId>
    <version>${flink.version}</version>
</dependency>
```
*Nota: Verifique sempre a documentação da sua versão do Flink para as dependências corretas.*

**Onde Encontrar Conectores:**

A documentação oficial do Apache Flink é o melhor lugar para encontrar a lista de conectores suportados, suas opções de configuração e dependências. Muitos conectores também são mantidos pela comunidade.

---

### Definindo `Table Sources` e `Sinks` usando DDL SQL (`CREATE TABLE`)

A maneira mais comum e declarativa de definir `Table Sources` e `Sinks` na Table API e SQL é através de instruções DDL SQL `CREATE TABLE`.

**Estrutura Geral:**

```sql
CREATE TABLE nome_da_tabela (
  coluna1 TIPO_DE_DADO,
  coluna2 TIPO_DE_DADO,
  -- Metadados, Colunas Computadas, Atributos de Tempo, Watermarks
  event_time TIMESTAMP(3),
  processing_time AS PROCTIME(), -- Coluna de tempo de processamento
  coluna_computada AS coluna1 * 2,
  WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND,
  -- Chaves Primárias (importante para upsert sinks e alguns conectores)
  PRIMARY KEY (coluna1) NOT ENFORCED
) WITH (
  'connector' = 'tipo_do_conector', -- Obrigatório: kafka, filesystem, jdbc, etc.
  -- Opções específicas do conector:
  'propriedade_conector_1' = 'valor1',
  'propriedade_conector_2' = 'valor2',
  -- Opções de formato (se o conector não tiver um formato padrão ou se você quiser especificar um):
  'format' = 'tipo_do_formato', -- json, avro, csv, etc.
  -- Opções específicas do formato:
  'propriedade_formato_1' = 'valor_formato1'
);
```

---

### Conectores Comuns Detalhados (com exemplos DDL SQL)

**1. FileSystem Connector (`connector' = 'filesystem'`)**

Lê e escreve dados de/para sistemas de arquivos distribuídos (HDFS, S3) ou locais.

* **Formatos Comuns:** CSV, JSON, Parquet, Avro, ORC.
* **Principais Opções:**
    * `path`: URI do diretório ou arquivo.
    * `format`: Tipo do formato dos dados.
    * `source.monitor-interval` (Source): Intervalo para monitorar novos arquivos.
    * `sink.partition-commit.trigger`, `sink.partition-commit.policy.kind` (Sink): Configurações para commit de partições.

* **Exemplo: Lendo um CSV (Source) e Escrevendo como Parquet (Sink)**

    ```sql
    -- Fonte: Lendo arquivos CSV
    CREATE TABLE CsvSalesSource (
      product_id STRING,
      category STRING,
      amount DOUBLE,
      sale_ts BIGINT,
      event_time AS TO_TIMESTAMP_LTZ(sale_ts, 3),
      WATERMARK FOR event_time AS event_time - INTERVAL '10' SECOND
    ) WITH (
      'connector' = 'filesystem',
      'path' = 'file:///path/to/input/csv_sales/', -- Use um caminho válido
      'format' = 'csv'
    );

    -- Sink: Escrevendo em formato Parquet
    CREATE TABLE ParquetCategorySummarySink (
      category STRING,
      total_amount DOUBLE,
      window_end_time TIMESTAMP(3)
    ) WITH (
      'connector' = 'filesystem',
      'path' = 'file:///path/to/output/parquet_summary/', -- Use um caminho válido
      'format' = 'parquet'
      -- Opções de particionamento e commit podem ser adicionadas aqui
    );

    -- Query de exemplo para popular o sink (requer uma agregação em janela)
    -- INSERT INTO ParquetCategorySummarySink
    -- SELECT ... FROM CsvSalesSource ... ;
    ```

**2. Apache Kafka Connector (`connector' = 'kafka'` ou `'upsert-kafka'`)**

Lê e escreve dados em tópicos Apache Kafka.

* **Formatos Comuns:** JSON, Avro, CSV, Debezium-JSON, etc.
* **Principais Opções:**
    * `topic`: Nome do(s) tópico(s).
    * `properties.bootstrap.servers`: Lista de brokers Kafka.
    * `properties.group.id` (Source): ID do grupo de consumidores.
    * `scan.startup.mode` (Source): De onde começar a consumir.
    * `sink.partitioner` (Sink): Como particionar dados.
    * `key.format`, `key.fields` (Sink): Formato/campos da chave Kafka.
    * `value.format`: Formato do valor Kafka.

* **Exemplo: Lendo JSON e escrevendo JSON transformado**

    ```sql
    -- Fonte: Lendo eventos de usuário JSON
    CREATE TABLE UserActivityKafkaSource (
      user_id STRING,
      page_url STRING,
      activity_timestamp BIGINT,
      event_time AS TO_TIMESTAMP_LTZ(activity_timestamp, 3),
      WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
    ) WITH (
      'connector' = 'kafka',
      'topic' = 'user_activity_topic',
      'properties.bootstrap.servers' = 'localhost:9092',
      'properties.group.id' = 'flink_user_activity_consumer',
      'scan.startup.mode' = 'latest-offset',
      'format' = 'json'
    );

    -- Sink: Escrevendo contagem por página em JSON
    CREATE TABLE PageViewCountsKafkaSink (
      page_url STRING PRIMARY KEY NOT ENFORCED, -- Chave para Upsert/Compactação
      view_count BIGINT,
      window_end TIMESTAMP(3)
    ) WITH (
      'connector' = 'upsert-kafka', -- Usando upsert-kafka
      'topic' = 'page_view_counts_topic',
      'properties.bootstrap.servers' = 'localhost:9092',
      'key.format' = 'json',   -- Formato da chave
      'value.format' = 'json'  -- Formato do valor
    );

    -- Query para popular o sink
    -- INSERT INTO PageViewCountsKafkaSink
    -- SELECT
    //   page_url,
    //   COUNT(*) AS view_count,
    //   TUMBLE_END(event_time, INTERVAL '1' MINUTE) AS window_end
    // FROM UserActivityKafkaSource
    // GROUP BY
    //   page_url,
    //   TUMBLE(event_time, INTERVAL '1' MINUTE);
    ```

**3. JDBC Connector (`connector' = 'jdbc'`)**

Lê e escreve dados em bancos de dados relacionais.

* **Principais Opções:**
    * `url`: URL JDBC.
    * `table-name`: Nome da tabela no banco.
    * `username`, `password`: Credenciais.
    * `driver`: Classe do driver JDBC.
    * Opções de `lookup` para leitura e `sink` para escrita (buffer, retries, upsert).

* **Exemplo: Escrevendo resultados para PostgreSQL**

    ```sql
    -- Sink JDBC
    CREATE TABLE AggregatedResultsJdbcSink (
      product_category STRING,
      total_sales DOUBLE,
      report_time TIMESTAMP(3)
    ) WITH (
      'connector' = 'jdbc',
      'url' = 'jdbc:postgresql://localhost:5432/mydatabase',
      'table-name' = 'aggregated_results_flink',
      'username' = 'flinkuser',
      'password' = 'flinkpassword'
    );

    -- Suponha uma view 'HourlySalesView'
    -- INSERT INTO AggregatedResultsJdbcSink SELECT * FROM HourlySalesView;
    ```

**4. Outros Conectores (Breve Menção):**

Existem conectores para Elasticsearch, RabbitMQ, Pulsar, Kinesis, Cassandra, etc. Consulte a documentação do Flink.

---

### Definindo `Table Sources` e `Sinks` Programaticamente (`TableDescriptor`)

Alternativa ao DDL SQL, útil para construção dinâmica em Java.

```java
// import org.apache.flink.table.api.*;
// import static org.apache.flink.table.api.Expressions.$;

// Supondo tableEnv inicializado
// Definindo um Kafka Source programaticamente
// TableDescriptor kafkaSourceDescriptor = TableDescriptor.forConnector("kafka")
//     .schema(Schema.newBuilder()
//         .column("user_id", DataTypes.STRING())
//         .column("message", DataTypes.STRING())
//         .column("event_ts", DataTypes.BIGINT())
//         .columnByExpression("rowtime", "TO_TIMESTAMP_LTZ($('event_ts'), 3)")
//         .watermark("rowtime", "$('rowtime') - INTERVAL '5' SECOND")
//         .build())
//     .option("topic", "my-input-topic")
//     .option("properties.bootstrap.servers", "localhost:9092")
//     .option("properties.group.id", "my-flink-group")
//     .option("scan.startup.mode", "earliest-offset")
//     .format("json")
//     .build();
// tableEnv.createTemporaryTable("MyKafkaSourceProgrammatic", kafkaSourceDescriptor);

// Definindo um Print Sink programaticamente
// TableDescriptor printSinkDescriptor = TableDescriptor.forConnector("print")
//     .schema(Schema.newBuilder()
//         .column("user_id", DataTypes.STRING())
//         .column("transformed_message", DataTypes.STRING())
//         .build())
//     .build();
// tableEnv.createTemporaryTable("MyPrintSinkProgrammatic", printSinkDescriptor);
```

---

### Modelo de Conectores no Confluent Cloud for Apache Flink

Ao utilizar o Apache Flink dentro de um ambiente gerenciado como o Confluent Cloud, a forma como você interage com sistemas externos pode ter particularidades.

**Nota Importante:** O Confluent Cloud for Apache Flink é um produto comercial da Confluent. As características e funcionalidades estão sujeitas a alterações. **Consulte sempre a documentação oficial da Confluent.**

**1. Kafka como Hub Central de Dados**

No Confluent Cloud, o Apache Kafka é o centro do ecossistema. Assim:
* **A interação primária e mais integrada do Flink é com os tópicos Kafka gerenciados pela Confluent Cloud.** O conector Kafka é otimizado e a integração com Schema Registry e segurança é facilitada.
* Kafka frequentemente atua como o principal ponto de entrada (`source`) e saída (`sink`) para os jobs Flink.

**2. Acessando Outros Sistemas via Kafka Connect**

Para interagir com sistemas *não*-Kafka (JDBC, S3, Elasticsearch, etc.), o padrão no ecossistema Confluent é usar o **Kafka Connect** (também oferecido como serviço gerenciado pela Confluent).

* **O Fluxo de Dados Típico:**
    1.  **Fonte Externa -> Flink:** Sistema Externo -> Kafka Connect Source -> Tópico Kafka -> Flink.
    2.  **Flink -> Destino Externo:** Flink -> Tópico Kafka -> Kafka Connect Sink -> Sistema Externo.

* **Diagrama do Fluxo via Kafka Connect:**
    ```
    +------------------+      +---------------------+      +----------------+      +--------------+
    | Sistema Externo  | ---> | Kafka Connect       | ---> | Tópico Kafka   | ---> | Flink        |
    | (e.g., Banco DB) |      | (Source Connector)  |      | (Dados Brutos) |      | (SQL/Table)  |
    |                  |      | (Gerenciado em CC)  |      | (Em CC)        |      | (Em CC Flink)|
    +------------------+      +---------------------+      +----------------+      +--------------+
                                                                                           |
                                                                                           v
    +------------------+      +---------------------+      +------------------+     +--------------+
    | Sistema Externo  | <--- | Kafka Connect       | <--- | Tópico Kafka     | <---| (Processados)|
    | (e.g., Data Lake)|      | (Sink Connector)    |      | (Resultados)     |      +--------------+
    |                  |      | (Gerenciado em CC)  |      | (Em CC)        |
    +------------------+      +---------------------+      +------------------+
    ```

* **Vantagens deste Modelo:**
    * **Simplicidade para o Flink:** O job Flink foca na interação com Kafka.
    * **Ecossistema Kafka Connect:** Aproveita a vasta gama de conectores Connect, gerenciados pela Confluent.
    * **Desacoplamento:** Kafka atua como um buffer e ponto de desacoplamento.

**3. Implicações e Considerações:**

* **Latência:** Pode adicionar latência comparado a conectores Flink diretos.
* **Conectores Diretos no Flink (em CC):** A Confluent pode suportar outros conectores Flink nativos além do Kafka. **Verifique a documentação.**
* **Configuração:** Feita através das interfaces/APIs da Confluent Cloud.

**Conclusão da Seção:**

No Confluent Cloud for Flink, espere uma forte integração com Kafka. Para outros sistemas, o uso do Kafka Connect gerenciado é o padrão, simplificando o job Flink, mas introduzindo o Kafka como intermediário.

---

### Interagindo com `Sources` e `Sinks`

* **Lendo de `Sources`:** Use `tableEnv.from("NomeDaTabelaRegistrada")` ou refira-se ao nome diretamente em SQL.
* **Escrevendo para `Sinks`:** Use `table.executeInsert("NomeDaTabela Sink")` (Table API) ou `INSERT INTO NomeDaTabela Sink SELECT ...` (SQL).

---

### Considerações Importantes

* **Semântica de Entrega (`Delivery Semantics`):** `At-least-once` vs. `Exactly-once`. Depende dos conectores e da configuração do Flink (checkpointing). Busque *exactly-once* para consistência crítica.
* **Paralelismo:** Configure o paralelismo de *sources* e *sinks* para performance.
* **Tratamento de Erros:** Use opções de formato (e.g., `json.ignore-parse-errors`) e confie no mecanismo de recuperação de falhas do Flink.
* **Segurança:** Gerencie credenciais e configure conexões seguras.

---

### Exercícios

1.  **DDL para Kafka Source:** Escreva uma instrução DDL SQL `CREATE TABLE` para uma fonte Kafka chamada `SensorInput` que lê de um tópico `iot_sensor_data`. Os dados estão em formato JSON e contêm os campos: `sensor_id` (STRING), `temperature` (DOUBLE), e `timestamp_ms` (BIGINT). Defina um atributo de `event_time` a partir de `timestamp_ms` e um `watermark` com 5 segundos de tolerância a desordem.

2.  **DDL para FileSystem Sink:** Escreva uma instrução DDL SQL `CREATE TABLE` para um *sink* FileSystem chamado `AlertsOutput` que escreve dados para o caminho `file:///tmp/sensor_alerts/` em formato CSV. A tabela deve ter colunas `alert_sensor_id` (STRING) e `alert_message` (STRING).

3.  **Opções do Conector Kafka:** Qual opção na cláusula `WITH` de um `CREATE TABLE` para um conector Kafka especifica de onde no tópico o Flink deve começar a ler (e.g., do início, do fim, ou de *offsets* de grupo)?
    * a) `format`
    * b) `scan.startup.mode`
    * c) `sink.partitioner`
    * d) `properties.group.id`

4.  **`TableDescriptor` vs. DDL:** Em que cenário você poderia preferir usar `TableDescriptor` em vez de DDL SQL para definir uma `Table Source`?
    * a) Sempre, pois é mais conciso.
    * b) Apenas para conectar ao Kafka.
    * c) Quando a definição da tabela precisa ser construída dinamicamente em código Java com base em parâmetros ou lógica externa.
    * d) `TableDescriptor` não pode ser usado para `Table Sources`, apenas para `Sinks`.

5.  **Conectividade no Confluent Cloud for Flink:** De acordo com a discussão, qual é a abordagem padrão recomendada no Confluent Cloud para conectar um job Flink a um banco de dados relacional externo (como fonte ou sink)?
    * a) Usar o conector JDBC Flink diretamente no job Flink.
    * b) Usar o conector FileSystem Flink para ler/escrever arquivos intermediários.
    * c) Usar conectores Kafka Connect (gerenciados pela Confluent) para mover dados entre o banco de dados e tópicos Kafka, e então fazer o job Flink interagir com esses tópicos Kafka.
    * d) O Flink no Confluent Cloud não pode interagir com bancos de dados relacionais.

6.  **Semântica de Entrega:** Se um conector Kafka *sink* e o Flink estão configurados para *checkpointing* e o conector suporta transações Kafka, qual semântica de entrega é tipicamente alcançável?
    * a) `At-most-once`
    * b) `At-least-once`
    * c) `Exactly-once`
    * d) Nenhuma garantia de entrega.

---

Conectar o Flink a sistemas externos é uma parte fundamental da construção de *pipelines* de dados completos. No último módulo, discutiremos algumas melhores práticas e dicas de otimização para suas aplicações Flink Table API.