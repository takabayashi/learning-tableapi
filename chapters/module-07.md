## Módulo 7: Conceitos Avançados da Table API

Bem-vindo ao Módulo 7! Após termos coberto os fundamentos, junções, agregações, tempo, janelas, UDFs e a integração SQL, é hora de mergulhar em conceitos mais avançados. Estes conceitos são cruciais para entender como o Flink lida com dados que mudam continuamente e como ele permite a construção de *pipelines* de dados reativos e com estado.

Neste módulo, focaremos em:
* **`Dynamic Tables` e `Continuous Queries`**: A base do processamento de *streams* no Flink SQL.
* **`Temporal Tables` e `Temporal Joins`**: Para enriquecer *streams* com dados versionados (como *Slowly Changing Dimensions*).
* **`Upsert Streams`**: Como o Flink lida com atualizações e deleções em *streams* e como isso se reflete nos conectores.

---

### `Dynamic Tables` & `Continuous Queries`

**1. O Conceito de `Dynamic Table`**

No Flink, uma `Table` não é apenas uma coleção estática de dados como em um banco de dados tradicional. Em vez disso, uma `Table` pode ser **dinâmica**: ela muda ao longo do tempo.

* **Analogia do Placar Esportivo Ao Vivo:** Pense em uma `Dynamic Table` como um placar de um jogo de futebol que é transmitido ao vivo. O placar (`Dynamic Table`) não é uma foto tirada no final do jogo (uma tabela estática). Ele é atualizado constantemente durante o jogo (o *stream* de eventos): gols são adicionados, cartões são mostrados, o tempo de jogo avança. A cada momento, o placar reflete o estado atual e cumulativo do jogo.

* Quando uma `Table` é derivada de um *stream* de dados contínuo (e.g., cliques de usuários, leituras de sensores, transações financeiras), ela é uma representação viva desses dados.
* À medida que novos eventos chegam ao *stream*, a `Dynamic Table` é atualizada.
* Essas atualizações podem ser:
    * **Appends:** Novas linhas são adicionadas à tabela (e.g., cada novo log de acesso a um site).
    * **Updates:** Linhas existentes na tabela são modificadas (e.g., o status de um pedido muda de "processando" para "enviado").
    * **Deletes:** Linhas existentes são removidas da tabela (e.g., um item de um carrinho de compras online é removido).
* O Flink lida com essa natureza dinâmica de forma transparente, seja a origem um *stream* de *append-only* ou um *changelog stream*.

**2. `Continuous Queries`**

Quando você executa uma query (seja SQL ou Table API) sobre uma ou mais `Dynamic Tables`, essa query é uma **`Continuous Query`**.

* Ao contrário de uma query em *batch* que processa um conjunto finito de dados e termina, uma `Continuous Query` opera sobre dados ilimitados e, teoricamente, nunca termina (para *streams* não limitados).
* O resultado de uma `Continuous Query` também é uma `Dynamic Table`. Esta tabela de resultado é continuamente atualizada à medida que os dados de entrada mudam.

**Como o Resultado de uma `Continuous Query` é Atualizado?**

O Flink precisa de uma maneira de propagar as mudanças das tabelas de entrada para a tabela de resultado.

* **Append Mode (Modo de Apensamento):**
    * Apenas novas linhas são adicionadas (inseridas) na tabela de resultado.
    * **Analogia do Feed de Notícias:** Um feed de notícias online onde novas matérias são simplesmente adicionadas ao topo ou ao final da lista. Resultados anteriores não são alterados.
    * Ocorre quando a query não precisa modificar resultados previamente emitidos (e.g., `SELECT ... FROM stream WHERE ...` sem agregações que mudam resultados passados, ou janelas de `tumbling` que só emitem quando a janela fecha).

* **Update Mode (Modo de Atualização):**
    * Linhas na tabela de resultado podem ser atualizadas ou deletadas.
    * **Analogia do Saldo Bancário:** Se você tem uma tabela que representa o saldo atual de contas de usuários (`user_id`, `balance`), e um *stream* de transações (`user_id`, `amount_change`), uma query contínua para manter o saldo atualizado produzirá *updates* na tabela de saldos.
    * O Flink representa essas mudanças usando:
        * **Retraction Stream:** Emite mensagens de `(-)` para a linha antiga que está sendo invalidada e `(+)` para a nova linha.
            * Exemplo (contagem de pedidos por usuário):
                * Usuário A faz 1º pedido: `+(user_A, count=1)`
                * Usuário A faz 2º pedido: `-(user_A, count=1), +(user_A, count=2)`
        * **Upsert Stream:** Emite a nova linha (atualizada), que deve substituir completamente a linha anterior que tem a mesma chave primária única.
            * Exemplo (último status do pedido por `order_id`):
                * Pedido 1 status "PROCESSANDO": `+(order_1, status='PROCESSANDO')`
                * Pedido 1 status "ENVIADO": `+(order_1, status='ENVIADO')` (substitui a anterior)

**Diagrama: `Continuous Query` com `GROUP BY`**

```
Stream de Eventos de Venda (OrderStream): (user, product, amount)
(Alice, P1, 10) -->
(Bob,   P2, 20) -->
(Alice, P3, 05) -->
(Charles,P1,30) -->
(Bob,   P1, 15) -->
                     +-------------------------------------+
                     | Continuous Query:                   |
                     | SELECT user, SUM(amount) AS total   |
                     | FROM OrderStream                    |
                     | GROUP BY user                       |
                     +-------------------------------------+
                                       |
                                       v
Dynamic Table de Resultado (UserTotalAmount):
Timestamp | Mudança Emitida (Upsert/Retraction) | Estado da Tabela UserTotalAmount
----------|-------------------------------------|---------------------------------
T1        | +(Alice, 10)                        | (Alice, 10)
T2        | +(Bob, 20)                          | (Alice, 10), (Bob, 20)
T3        | -(Alice, 10), +(Alice, 15)          | (Alice, 15), (Bob, 20)
T4        | +(Charles, 30)                      | (Alice, 15), (Bob, 20), (Charles, 30)
T5        | -(Bob, 20), +(Bob, 35)              | (Alice, 15), (Bob, 35), (Charles, 30)
```
Neste exemplo, a tabela `UserTotalAmount` é atualizada continuamente. Se o *sink* puder lidar com *retractions*, ele pode manter a soma correta. Se uma chave primária (`user`) for definida e o *sink* for um *upsert sink*, ele apenas aplicaria a última versão.

**3. Declarando Semântica de `Changelog`**

Para converter entre `DataStream` e `Table`, o Flink precisa saber a semântica do `DataStream`:
* `tableEnv.fromDataStream(dataStream)`: Assume que o `DataStream` é *append-only*.
* `tableEnv.fromChangelogStream(DataStream<Row> changelogStream, Schema schema, ChangelogMode changelogMode)`: Permite especificar que o `DataStream` contém inserções, atualizações e/ou deleções. `ChangelogMode` define os tipos de mudança (`INSERT_ONLY`, `UPSERT`, `ALL`).
* `tableEnv.toDataStream(table)`: Converte uma `Table` (que pode ser dinâmica) para um `DataStream`. Se a `Table` for atualizada, o `DataStream` resultante será um *changelog stream* (emitindo `RowKind`).
* `tableEnv.toChangelogStream(table)`: Similar, mas mais explícito sobre a natureza do *changelog*.

Conectores como Debezium (para Change Data Capture - CDC) ou o conector Upsert Kafka são projetados para produzir ou consumir *streams* com semântica de *changelog*.

---

### `Temporal Tables` & `Temporal Joins`

**1. O que é uma `Temporal Table`?**

Uma `Temporal Table` é uma `Table` que rastreia o histórico de suas versões ao longo do tempo. Ela não representa apenas o estado atual dos dados, mas também como esses dados eram em pontos específicos no passado.

* **Analogia do Livro de Histórico de Endereços:** Imagine que uma empresa mantém um livro com o histórico de endereços de seus clientes. Para cada cliente, o livro não mostra apenas o endereço atual, mas todos os endereços anteriores, cada um com um período de validade (e.g., "Rua A, nº 123, válido de 01/01/2020 até 15/05/2022"; "Rua B, nº 456, válido de 16/05/2022 até hoje"). Uma `Temporal Table` funciona de forma similar, permitindo "olhar para trás no tempo".

* **Por que usar?** É fundamental para enriquecimento de dados de *stream* com dados dimensionais que mudam lentamente (SCD - Slowly Changing Dimensions). Por exemplo, se você tem um *stream* de pedidos e os preços dos produtos mudam ocasionalmente, você vai querer juntar cada pedido com o preço do produto que estava válido *no momento daquele pedido*, não o preço atual.

**2. `Versioning` e `Temporal Table Function`**

Para usar uma tabela como `Temporal Table`, ela precisa ser **versionada**. Isso significa que:
1.  Deve ter uma **chave primária** (e.g., `product_id`).
2.  Deve ter um **atributo de tempo de versionamento** (e.g., `price_valid_from_timestamp`) que indica quando uma determinada versão do registro (com aquela chave primária) se tornou válida. O Flink usa esse atributo para determinar qual versão do registro estava ativa em um dado ponto no tempo.

Uma `Temporal Table Function` é uma abstração que o Flink pode gerar a partir de uma tabela de histórico versionada. Quando você realiza um `Temporal Join`, essa função é invocada com o tempo do evento do *stream* principal para buscar a versão correta do registro na tabela versionada.

**3. `Temporal Join`**

Um `Temporal Join` junta um *stream* de eventos (tabela de fatos, e.g., `Pedidos`) com uma `Temporal Table` (tabela de dimensões versionada, e.g., `PrecosHistoricosProdutos`).

* **Analogia do Detetive de Preços:** Um detetive (`Temporal Join`) está investigando pedidos antigos. Para cada pedido, ele consulta um catálogo antigo de preços de produtos (`Temporal Table`). Ele não usa o catálogo de preços de hoje, mas sim a versão do catálogo que estava em vigor na data exata de cada pedido, garantindo que o preço associado a cada item do pedido seja historicamente correto.

* **Sintaxe SQL (usando `FOR SYSTEM_TIME AS OF`):**
  Esta é a forma padrão e mais clara de expressar um `Temporal Join`.

  ```sql
  -- Suponha um stream de Pedidos (OrdersStream)
  -- OrdersStream: order_id STRING, currency_code STRING, amount DECIMAL(10,2), order_time TIMESTAMP(3) (ROWTIME)

  -- Suponha uma tabela de histórico de Taxas de Câmbio (CurrencyRatesHistory)
  -- CurrencyRatesHistory: currency_code STRING, rate DECIMAL(10,4), update_time TIMESTAMP(3) (ROWTIME), PRIMARY KEY(currency_code)
  -- Onde update_time é o tempo a partir do qual a taxa é válida e é o atributo de tempo para versionamento.

  CREATE TABLE CurrencyRatesHistory (
    currency_code STRING,
    rate DECIMAL(10, 4),
    update_time TIMESTAMP(3), -- Tempo do evento da atualização da taxa
    PRIMARY KEY (currency_code) NOT ENFORCED,
    WATERMARK FOR update_time AS update_time -- Define 'update_time' como o versioning time
  ) WITH (
    'connector' = '...', -- e.g., kafka com changelog de taxas, ou jdbc com histórico
    -- ... outras opções
  );


  -- Query com Temporal Join
  SELECT
    o.order_id,
    o.currency_code,
    o.amount,
    o.order_time,
    r.rate AS exchange_rate_at_order_time,
    o.amount * r.rate AS amount_in_usd
  FROM OrdersStream AS o
  JOIN CurrencyRatesHistory FOR SYSTEM_TIME AS OF o.order_time AS r -- Cláusula Temporal Join
  ON o.currency_code = r.currency_code;
  ```
  Nesta query, para cada pedido `o` de `OrdersStream`, o Flink busca a taxa (`r.rate`) da `CurrencyRatesHistory` onde `r.currency_code` corresponde e cuja versão era válida no momento `o.order_time`.

* **Table API:**
  Na Table API, `Temporal Joins` são geralmente implementados usando uma `TemporalTableFunction` (criada a partir da tabela de histórico) e um `joinLateral`, ou, mais modernamente, através de *lookup joins* em fontes de tabelas configuradas como versionadas. A sintaxe SQL `FOR SYSTEM_TIME AS OF` também é suportada em *lookup joins* na Table API para fontes versionadas.

  ```java
  // import org.apache.flink.table.functions.TemporalTableFunction;
  // import static org.apache.flink.table.api.Expressions.$;
  // import static org.apache.flink.table.api.Expressions.call;

  // Suponha 'ordersStream' (Table object)
  // Suponha 'currencyRatesHistoryTable' (Table object com 'update_time' como time attribute e 'currency_code' como primary key)

  // 1. Criar a TemporalTableFunction
  // TemporalTableFunction ratesFunction = currencyRatesHistoryTable
  //     .createTemporalTableFunction($("update_time"), $("currency_code"));

  // 2. Registrar (se for chamar pelo nome)
  // tableEnv.createTemporarySystemFunction("RatesTemporalFunc", ratesFunction);

  // 3. Executar o joinLateral (uma forma de realizar o join temporal)
  // Table result = ordersStream
  //     .joinLateral(
  //         call("RatesTemporalFunc", $("order_time")).as("r_currency_code", "r_rate", "r_update_time"),
  //         $("currency_code").isEqual($("r_currency_code"))
  //     )
  //     .select(
  //         $("order_id"),
  //         $("amount"),
  //         $("r_rate").as("exchange_rate_at_order_time")
  //     );
  // result.execute().print();
  ```
  *Nota: A abordagem recomendada para versões mais recentes do Flink é usar fontes de tabela explicitamente versionadas (com `VERSIONED BY` em DDL quando o conector suporta) e então usar a sintaxe SQL `FOR SYSTEM_TIME AS OF` ou a semântica equivalente de lookup join na Table API.*

* **Diagrama:**
  ```
  OrdersStream (eventos ao longo do tempo)
  order_time | currency | amount
  ---------------------------------
  10:00:05   | EUR      | 100
  10:00:15   | GBP      | 50
  10:00:25   | EUR      | 200

  CurrencyRatesHistory (taxas mudam ao longo do tempo, versionadas por 'update_time')
  currency | rate | update_time (valid_from)
  -------------------------------------
  EUR      | 1.10 | 10:00:00  <-- Versão 1 do EUR
  GBP      | 1.30 | 10:00:00
  EUR      | 1.12 | 10:00:20  <-- Versão 2 do EUR

  Resultado do Temporal Join (FOR SYSTEM_TIME AS OF ordersStream.order_time):
  order_time | currency | amount | exchange_rate_at_order_time
  --------------------------------------------------------------
  10:00:05   | EUR      | 100    | 1.10  (usa a Versão 1 do EUR, válida às 10:00:05)
  10:00:15   | GBP      | 50     | 1.30
  10:00:25   | EUR      | 200    | 1.12  (usa a Versão 2 do EUR, válida às 10:00:25)
  ```

* **Fontes para `Temporal Tables`:**
    Uma `Temporal Table` é tipicamente construída a partir de uma tabela que captura o histórico de mudanças. Isso pode ser:
    * Um *changelog stream* proveniente de um sistema de Change Data Capture (CDC) como Debezium.
    * Uma tabela de histórico com colunas `valid_from`/`valid_to` gerenciadas pela aplicação.
    * No Flink, a tabela de origem precisa de uma `PRIMARY KEY` e um atributo de tempo que atue como o *versioning time attribute* (e.g., `event_time` da atualização).

---

### `Upsert Streams` e Conectores

**1. O que é um `Upsert Stream`?**

Um `Upsert Stream` é um tipo de *changelog stream* que contém operações de `INSERT` e `UPDATE`.
* Se um novo evento chega com uma chave que não existe na tabela de resultado (ou no *sink*), é uma inserção.
* Se um novo evento chega com uma chave que já existe, o registro existente é atualizado com os novos valores.
* Deleções (`DELETE`) podem ser representadas por mensagens especiais (e.g., um registro com valor `null` para uma chave específica em um tópico Kafka compactado).

* **Analogia do Perfil de Usuário Online:** Pense no seu perfil em uma rede social. Quando você atualiza sua foto ou status, você não está criando um novo perfil; você está atualizando o existente. O sistema que recebe essa atualização está processando um evento "upsert": atualize o perfil para `user_id = 'seu_id'` com os novos dados. Se você fosse um novo usuário, seria uma inserção. O fluxo de todas essas atualizações de perfis para o banco de dados da rede social pode ser visto como um `Upsert Stream`.

**2. Como o Flink lida com `Upsert Streams`:**

* Muitas queries no Flink naturalmente produzem `Dynamic Tables` cujas mudanças podem ser interpretadas como um `Upsert Stream` se uma chave primária (ou única) for definida. Por exemplo, uma agregação `GROUP BY` sem janelas em um *stream* produz atualizações para os valores agregados de cada grupo.
* A abordagem moderna é definir a semântica de *upsert* no *sink* do conector, que então consome o *changelog stream* produzido pela query Flink.

**3. Conectores que Suportam `Upsert` (Sinks):**

* **Upsert Kafka Connector (`connector' = 'upsert-kafka'`):**
    * Escreve um *changelog stream* para um tópico Kafka. A chave primária da `Table` é usada como a chave da mensagem Kafka.
    * **Detalhe sobre Compactação:** Se o tópico Kafka estiver configurado para **compactação de log** (`log.cleanup.policy=compact`), o Kafka garantirá que apenas a última mensagem (o estado mais recente) para cada chave seja retida (após os períodos de retenção de limpeza). Isso transforma seu tópico Kafka em uma espécie de "K-Table" durável e consultável do estado mais recente de cada chave.
    * Deleções podem ser sinalizadas por mensagens Kafka com valor `null` (tombstones).

* **JDBC Connector:** Pode ser configurado para operar em modo *upsert*.

**4. Definindo Chaves Primárias:**

Para que um *sink* possa interpretar um *stream* como *upsert*, a `Table` que está sendo escrita geralmente precisa ter uma chave primária definida: `PRIMARY KEY (col1, col2, ...) NOT ENFORCED`.

**Exemplo com `Upsert Kafka Sink` (Agregação Contínua):**

```sql
-- Tabela de origem (stream de atividades do usuário)
CREATE TABLE UserActivityStream (
    user_id STRING,
    activity_time TIMESTAMP(3),
    action_type STRING,
    WATERMARK FOR activity_time AS activity_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'datagen',
    'fields.user_id.kind' = 'random', 'fields.user_id.length' = '3',
    'fields.action_type.values' = 'CLICK,VIEW,PURCHASE',
    'rows-per-second' = '1'
);

-- View que calcula a contagem de atividades e o tempo da última atividade por usuário
CREATE TEMPORARY VIEW UserAggregatesView AS
SELECT
    user_id,
    COUNT(*) AS event_count,
    MAX(activity_time) AS latest_activity
FROM UserActivityStream
GROUP BY user_id;

-- Tabela Sink para Upsert Kafka
CREATE TABLE UserAggregatesKafkaSink (
    user_id STRING PRIMARY KEY NOT ENFORCED, -- Chave primária para o upsert
    event_count BIGINT,
    latest_activity TIMESTAMP(3)
) WITH (
    'connector' = 'upsert-kafka',
    'topic' = 'user_aggregates_topic',
    'properties.bootstrap.servers' = 'localhost:9092', -- Seu broker Kafka
    'key.format' = 'json',   -- Formato da chave Kafka (baseado na PRIMARY KEY)
    'value.format' = 'json'  -- Formato do valor Kafka
);

-- Inserir os resultados da agregação contínua no sink upsert-kafka
-- Esta query produzirá um changelog stream que o upsert-kafka sink pode processar.
INSERT INTO UserAggregatesKafkaSink
SELECT * FROM UserAggregatesView;
```
Neste exemplo, à medida que novos eventos chegam em `UserActivityStream`, a `UserAggregatesView` (que é uma `Dynamic Table`) é continuamente atualizada. O `INSERT INTO UserAggregatesKafkaSink` envia essas atualizações (upserts) para o tópico Kafka `user_aggregates_topic`.

---

### Tabela Comparativa: `Dynamic Table` vs. `Temporal Table`

| Característica         | `Dynamic Table` (Geral)                                                                 | `Temporal Table` (para `Temporal Join`)                                                                                                |
|------------------------|-----------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| **Descrição Principal** | Uma tabela cujos conteúdos mudam ao longo do tempo devido a um *stream* de dados contínuo. | Uma tabela que rastreia e fornece acesso a diferentes versões de seus registros ao longo do tempo (histórico).                                |
| **Natureza dos Dados** | Representa o estado atual (ou em evolução) de um *stream* ou o resultado de uma `Continuous Query`. Pode ser *append-only* ou um *changelog stream* (com inserts, updates, deletes). | Tipicamente uma tabela de dimensões com histórico de versões (SCD). Cada versão tem um período de validade.                                     |
| **Como é Usada** | Como entrada ou saída de `Continuous Queries`. Pode ser materializada em *sinks* que entendem *changelogs* (e.g., *upsert sinks*). | Usada no lado "dimensional" (lado direito) de um `Temporal Join` (com `FOR SYSTEM_TIME AS OF ...`) para enriquecer um *stream* com dados versionados. |
| **Objetivo Principal** | Processar e transformar *streams* de dados em tempo real, produzindo resultados que também evoluem continuamente. | Fornecer uma visão "point-in-time" de uma tabela de referência para que eventos de um *stream* possam ser juntados com os dados corretos de seu tempo. |
| **Exemplo de Caso de Uso** | Contagem contínua de eventos por usuário; filtragem de um *stream* de logs; resultado de uma agregação em janela que se atualiza. | Juntar um *stream* de pedidos com uma tabela de preços de produtos que mudam, usando o preço válido no momento de cada pedido.         |
| **Analogia Chave** | Placar esportivo ao vivo; feed de notícias contínuo; saldo bancário que se atualiza.        | Livro de histórico de endereços de clientes; catálogo de preços de produtos com datas de validade para cada preço.                          |
| **Chave para Entender**| Conceito de *stream* de mudanças (inserts, updates, deletes) aplicado a uma tabela. As queries são contínuas e os resultados são dinâmicos.        | Conceito de versionamento de dados e a capacidade de "viajar no tempo" para buscar um estado passado específico para fins de junção.     |

---

### Exercícios

1.  **`Dynamic Table`:** Usando a analogia do "Placar Esportivo Ao Vivo", quais seriam os "eventos" do *stream* que atualizam o placar, e quais informações no placar representariam o estado da `Dynamic Table`?

2.  **`Temporal Join`:** Você tem um *stream* de leituras de sensores IoT, onde cada leitura tem `sensor_id`, `reading_value`, e `reading_timestamp`. Você também tem uma tabela `SensorConfiguration` que armazena os limiares de alerta para cada sensor (`sensor_id`, `warning_threshold`, `critical_threshold`, `config_update_time`), e esses limiares podem mudar ao longo do tempo. Como um `Temporal Join` ajudaria a verificar se uma leitura de sensor excedeu o limiar correto que estava em vigor no momento da leitura?

3.  **`Upsert Stream` vs. `Append-only Stream`:** Qual é a principal diferença na informação que um `Upsert Stream` carrega em comparação com um `Append-only Stream`?
    * a) `Upsert Streams` só contêm novas inserções, enquanto `Append-only Streams` podem conter atualizações.
    * b) `Upsert Streams` podem representar atualizações ou inserções para registros existentes (baseado em uma chave), enquanto `Append-only Streams` apenas adicionam novos registros independentes.
    * c) `Append-only Streams` são usados para *sinks* Kafka, e `Upsert Streams` para *sinks* JDBC.
    * d) Não há diferença funcional, apenas no nome.

4.  **Resultado de `Continuous Query`:** Se você executa uma query `SELECT user_id, COUNT(*) FROM clicks GROUP BY user_id` sobre um *stream* contínuo de cliques, o que você espera que seja a natureza da `Dynamic Table` resultante?
    * a) Uma tabela estática calculada uma única vez.
    * b) Uma tabela que só recebe novas linhas (append-only) à medida que novos usuários aparecem.
    * c) Uma tabela que é continuamente atualizada (upsert/retraction) à medida que novos cliques chegam para usuários existentes, modificando suas contagens.
    * d) Uma tabela vazia, pois `COUNT(*)` não é permitido em *streams*.

5.  **Chave Primária em `Upsert Sinks`:** Por que a definição de uma chave primária (`PRIMARY KEY ... NOT ENFORCED`) é importante ao usar um *sink* que espera `Upsert Streams` (como o `upsert-kafka`)?

---

Estes conceitos avançados são a chave para desbloquear todo o potencial do Flink para aplicações de processamento de *stream* complexas e com estado. No próximo módulo, veremos como conectar o Flink a sistemas externos usando `connectors`.