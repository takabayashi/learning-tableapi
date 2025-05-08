Entendido! Você gostaria que eu apresentasse novamente o Módulo 3 completo, já com os diagramas e a tabela comparativa de `joins` que adicionamos.

Aqui está o Módulo 3 revisado e completo:

---

## Módulo 3: Operações Relacionais

Bem-vindo ao Módulo 3! Neste módulo, exploraremos as operações relacionais fundamentais disponíveis na Table API do Flink. Estas operações permitem combinar, agrupar, filtrar e ordenar dados de maneira poderosa e declarativa, similar ao que você faria com SQL tradicional, mas aplicado tanto a *streams* quanto a dados em *batch*.

Cobriremos:
* Junções (`Joins`) para combinar dados de múltiplas tabelas.
* Agrupamento e Agregação (`Grouping and Aggregation`) para sumarizar dados.
* Operações de Conjunto (`Set Operations`) como `UNION`, `INTERSECT` e `MINUS`.
* Ordenação e Limitação de Resultados (`Ordering and Limiting`).

---

### Junções (`Joins`)

Junções são usadas para combinar linhas de duas `Tables` com base em uma condição de junção relacionada. O Flink Table API suporta vários tipos de junções.

**Tipos Comuns de Junção:**

* **`INNER JOIN` (ou apenas `join`):** Retorna apenas as linhas que têm correspondência em ambas as `Tables`.
* **`LEFT OUTER JOIN` (ou `leftOuterJoin`):** Retorna todas as linhas da `Table` da esquerda e as linhas correspondentes da `Table` da direita. Se não houver correspondência, as colunas da `Table` da direita terão valor `NULL`.
* **`RIGHT OUTER JOIN` (ou `rightOuterJoin`):** Retorna todas as linhas da `Table` da direita e as linhas correspondentes da `Table` da esquerda. Se não houver correspondência, as colunas da `Table` da esquerda terão valor `NULL`.
* **`FULL OUTER JOIN` (ou `fullOuterJoin`):** Retorna todas as linhas de ambas as `Tables`. Se não houver correspondência em um dos lados, as colunas do outro lado terão valor `NULL`.

**Diagrama Conceitual de `Joins` (Exemplo com `INNER JOIN` e `LEFT JOIN`):**

Suponha as seguintes tabelas:

**`Customers` Table:**
| c_id | c_name  | c_city     |
|------|---------|------------|
| 1    | Alice   | New York   |
| 2    | Bob     | London     |
| 3    | Charlie | Paris      |
| 4    | David   | New York   |

**`Orders` Table:**
| o_id | o_customer_id | o_product | o_amount |
|------|---------------|-----------|----------|
| 101  | 1             | Laptop    | 1200.00  |
| 102  | 2             | Mouse     | 25.00    |
| 103  | 1             | Keyboard  | 75.00    |
| 104  | 3             | Monitor   | 300.00   |
| 105  | 5             | Webcam    | 50.00    |

1.  **`INNER JOIN` entre `Customers` e `Orders` em `c_id = o_customer_id`:**
    Apenas linhas com `c_id` correspondente em `o_customer_id` são incluídas. O cliente David (c_id=4) e o pedido 105 (o_customer_id=5) são excluídos.

    ```
    Customers (c_id)       Orders (o_customer_id)     Resultado do INNER JOIN (c_name, o_product)
    +---+                  +---+                      +-------------------+-------------------+
    | 1 |------------------| 1 |                      | Alice             | Laptop            |
    | 1 |------------------| 1 |                      | Alice             | Keyboard          |
    | 2 |------------------| 2 |                      | Bob               | Mouse             |
    | 3 |------------------| 3 |                      | Charlie           | Monitor           |
    | 4 | (sem par)        +---+
    +---+                  | 5 | (sem par)
                           +---+
    ```
    **Resultado:**
    | c_name  | o_product |
    |---------|-----------|
    | Alice   | Laptop    |
    | Alice   | Keyboard  |
    | Bob     | Mouse     |
    | Charlie | Monitor   |

2.  **`LEFT OUTER JOIN` de `Customers` (esquerda) com `Orders` (direita) em `c_id = o_customer_id`:**
    Todas as linhas de `Customers` são incluídas. Se não houver pedido correspondente, as colunas de `Orders` são `NULL`. O pedido 105 ainda é excluído, pois não tem cliente correspondente na tabela da esquerda.

    ```
    Customers (c_id)       Orders (o_customer_id)     Resultado do LEFT JOIN (c_name, o_product)
    +---+                  +---+                      +-------------------+-------------------+
    | 1 |------------------| 1 |                      | Alice             | Laptop            |
    | 1 |------------------| 1 |                      | Alice             | Keyboard          |
    | 2 |------------------| 2 |                      | Bob               | Mouse             |
    | 3 |------------------| 3 |                      | Charlie           | Monitor           |
    | 4 | (sem par em Orders)                       | David             | NULL              |
    +---+                  +---+
                           | 5 | (sem par em Customers)
                           +---+
    ```
    **Resultado:**
    | c_name  | o_product |
    |---------|-----------|
    | Alice   | Laptop    |
    | Alice   | Keyboard  |
    | Bob     | Mouse     |
    | Charlie | Monitor   |
    | David   | NULL      |

**Tabela Comparativa dos Tipos de `JOIN`:**

| Tipo de `JOIN`    | Descrição                                                                                                | Resultado Conceitual (Clientes e Pedidos)                                                                                                |
|-------------------|----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| `INNER JOIN`      | Retorna linhas apenas quando há uma correspondência em ambas as tabelas.                                   | Apenas clientes que fizeram pedidos e apenas pedidos de clientes existentes. (Cliente David e Pedido 105 são excluídos).                 |
| `LEFT OUTER JOIN` | Retorna todas as linhas da tabela da esquerda e as linhas correspondentes da tabela da direita.            | Todos os clientes. Se um cliente não tiver pedidos, as colunas do pedido são `NULL`. (Cliente David aparece; Pedido 105 é excluído).  |
| `RIGHT OUTER JOIN`| Retorna todas as linhas da tabela da direita e as linhas correspondentes da tabela da esquerda.           | Todos os pedidos. Se um pedido não tiver um cliente correspondente, as colunas do cliente são `NULL`. (Pedido 105 aparece; Cliente David é excluído). |
| `FULL OUTER JOIN` | Retorna todas as linhas de ambas as tabelas. Se não houver correspondência, as colunas do outro lado são `NULL`. | Todos os clientes e todos os pedidos. (Cliente David aparece com pedidos `NULL`; Pedido 105 aparece com cliente `NULL`).             |

**Sintaxe Básica:**
Para executar os exemplos de código Java, você precisará de um `TableEnvironment` configurado e das tabelas `customers` e `orders` criadas, por exemplo, usando `fromValues` como mostrado nos módulos anteriores e nos comentários do código abaixo.

```java
import org.apache.flink.table.api.*;
import org.apache.flink.types.Row;
import static org.apache.flink.table.api.Expressions.*; // Para $, lit, etc.

// --- Configuração do Ambiente e Tabelas de Exemplo (coloque dentro de um método main) ---
// EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().build();
// TableEnvironment tableEnv = TableEnvironment.create(settings);

// Tabela de Clientes (Customers)
// Table customers = tableEnv.fromValues(
//     DataTypes.ROW(
//         DataTypes.FIELD("c_id", DataTypes.INT().notNull()),
//         DataTypes.FIELD("c_name", DataTypes.STRING()),
//         DataTypes.FIELD("c_city", DataTypes.STRING())
//     ),
//     Row.of(1, "Alice", "New York"),
//     Row.of(2, "Bob", "London"),
//     Row.of(3, "Charlie", "Paris"),
//     Row.of(4, "David", "New York")
// ).as("c_id", "c_name", "c_city"); // Usar .as() aqui para nomear as colunas pós-fromValues

// Tabela de Pedidos (Orders)
// Table orders = tableEnv.fromValues(
//     DataTypes.ROW(
//         DataTypes.FIELD("o_id", DataTypes.INT().notNull()),
//         DataTypes.FIELD("o_customer_id", DataTypes.INT()),
//         DataTypes.FIELD("o_product", DataTypes.STRING()),
//         DataTypes.FIELD("o_amount", DataTypes.DECIMAL(10, 2))
//     ),
//     Row.of(101, 1, "Laptop", DataTypes.DECIMAL(10,2).toInternal(1200.00)),
//     Row.of(102, 2, "Mouse", DataTypes.DECIMAL(10,2).toInternal(25.00)),
//     Row.of(103, 1, "Keyboard", DataTypes.DECIMAL(10,2).toInternal(75.00)),
//     Row.of(104, 3, "Monitor", DataTypes.DECIMAL(10,2).toInternal(300.00)),
//     Row.of(105, 5, "Webcam", DataTypes.DECIMAL(10,2).toInternal(50.00)) // Pedido de um cliente inexistente na tabela customers
// ).as("o_id", "o_customer_id", "o_product", "o_amount");

// --- Inner Join ---
// Table innerJoined = customers
//     .join(orders, $("c_id").isEqual($("o_customer_id"))) // Condição de junção
//     .select($("c_name"), $("o_product"), $("o_amount")); // Selecionando colunas de interesse

// System.out.println("\n--- Inner Join (Clientes e seus Pedidos) ---");
// innerJoined.execute().print();

// --- Left Outer Join ---
// Table leftJoined = customers
//     .leftOuterJoin(orders, $("c_id").isEqual($("o_customer_id")))
//     .select($("c_id"), $("c_name"), $("o_id"), $("o_product"));

// System.out.println("\n--- Left Outer Join (Todos Clientes, e seus Pedidos se existirem) ---");
// leftJoined.execute().print(); // David (c_id=4) aparecerá com o_id e o_product NULL

// --- Right Outer Join ---
// Table rightJoined = customers
//     .rightOuterJoin(orders, $("c_id").isEqual($("o_customer_id")))
//     .select($("c_name"), $("o_id"), $("o_product"), $("o_customer_id"));

// System.out.println("\n--- Right Outer Join (Todos Pedidos, e seus Clientes se existirem) ---");
// rightJoined.execute().print(); // Pedido 105 (o_customer_id=5) aparecerá com c_name NULL

// --- Full Outer Join ---
// Table fullJoined = customers
//     .fullOuterJoin(orders, $("c_id").isEqual($("o_customer_id")))
//     .select($("c_name"), $("o_product"));

// System.out.println("\n--- Full Outer Join (Todos Clientes e Todos Pedidos) ---");
// fullJoined.execute().print();
```

**Junções Temporais (`Time-Windowed Joins / Interval Joins`):**

Em processamento de *streams*, frequentemente você precisa juntar eventos que ocorrem próximos no tempo. O Flink suporta:
* **Interval Joins:** Junta linhas de duas `Tables` se seus atributos de tempo estiverem dentro de um intervalo de tempo relativo um ao outro.
* **Windowed Joins (usando TVFs - Table-Valued Functions):** Junta agregações de janelas das duas `Tables`.

Exemplo de um `Interval Join` (conceitual, requer atributos de tempo de evento e *watermarks* definidos, que veremos no Módulo 4):

```java
// Suponha Table 'impressions' com (ad_id, impression_time AS ROWTIME)
// Suponha Table 'clicks' com (ad_id, click_time AS ROWTIME)

// Table intervalJoined = impressions.join(
//         clicks,
//         $("impressions.ad_id").isEqual($("clicks.ad_id")) // Chave de junção
//         .and($("impressions.impression_time").between(        // Condição de tempo
//                 $("clicks.click_time").minus(lit(1).hour()), // Limite inferior do intervalo
//                 $("clicks.click_time")                       // Limite superior do intervalo
//         ))
//     )
//     .select($("impressions.ad_id"), $("impression_time"), $("click_time"));
```
Este tipo de junção é crucial para correlação de eventos em *streams* e será mais explorado quando falarmos de tempo (Módulo 4).

---

### Agrupamento e Agregação (`Grouping and Aggregation`)

Estas operações permitem agrupar linhas com base em chaves comuns e calcular funções agregadas para cada grupo.

**Diagrama Conceitual de Agregação:**

Suponha a tabela `Orders` do exemplo anterior. Vamos agrupar por `o_customer_id` e calcular `COUNT(o_id)` e `SUM(o_amount)`.

**`Orders` Table (Entrada):**
| o_id | o_customer_id | o_product | o_amount |
|------|---------------|-----------|----------|
| 101  | 1             | Laptop    | 1200.00  |
| 102  | 2             | Mouse     | 25.00    |
| 103  | 1             | Keyboard  | 75.00    |
| 104  | 3             | Monitor   | 300.00   |
| 105  | 5             | Webcam    | 50.00    |

**Passo 1: `groupBy($("o_customer_id"))`**
As linhas são particionadas em grupos com base no `o_customer_id`.

```
Grupo o_customer_id = 1:
  (101, 1, "Laptop", 1200.00)
  (103, 1, "Keyboard", 75.00)

Grupo o_customer_id = 2:
  (102, 2, "Mouse", 25.00)

Grupo o_customer_id = 3:
  (104, 3, "Monitor", 300.00)

Grupo o_customer_id = 5:
  (105, 5, "Webcam", 50.00)
```

**Passo 2: `select($("o_customer_id"), $("o_id").count().as("num_orders"), $("o_amount").sum().as("total_amount"))`**
As funções de agregação são aplicadas a cada grupo.

**Resultado da Agregação:**
| o_customer_id | num_orders | total_amount |
|---------------|------------|--------------|
| 1             | 2          | 1275.00      |
| 2             | 1          | 25.00        |
| 3             | 1          | 300.00       |
| 5             | 1          | 50.00        |

* **`groupBy(Expression... fields)`:** Agrupa a `Table` pelas chaves especificadas.
* **Funções de Agregação:** Usadas dentro de uma operação `select` após um `groupBy`. Comuns incluem:
    * `sum(Expression field)`
    * `count(Expression field)` ou `count().distinct()`
    * `avg(Expression field)`
    * `min(Expression field)`
    * `max(Expression field)`
    * `listAgg(Expression field)`: Agrega valores em uma lista (string).
* **`select(Expression... fields)`:** Após o `groupBy`, você seleciona as chaves de agrupamento e as agregações.
* **`having(Expression predicate)`:** Filtra os grupos com base em uma condição sobre os resultados da agregação (similar ao `HAVING` do SQL).

```java
// Reutilizando a tabela 'orders' do exemplo de Joins
// (Suponha que 'tableEnv', 'customers' e 'orders' estão definidos e populados)

// Agregação simples: Contar pedidos e somar o total por cliente
// Table aggByCustomer = orders
//     .groupBy($("o_customer_id"))
//     .select(
//         $("o_customer_id"),
//         $("o_id").count().as("num_orders"), // Contagem de pedidos
//         $("o_amount").sum().as("total_amount") // Soma dos valores
//     );

// System.out.println("\n--- Agregação por Cliente (Contagem e Soma) ---");
// aggByCustomer.execute().print();

// Agregação com HAVING: Clientes com mais de 1 pedido
// Table aggWithHaving = orders
//     .groupBy($("o_customer_id"))
//     .select(
//         $("o_customer_id"),
//         $("o_id").count().as("num_orders"),
//         $("o_amount").sum().as("total_amount")
//     )
//     .having($("num_orders").isGreater(1)); // Condição HAVING

// System.out.println("\n--- Agregação com HAVING (Clientes com >1 Pedido) ---");
// aggWithHaving.execute().print();
```

**Agregação em Janelas (`Windowed Aggregation`):**

Para dados de *streaming*, as agregações são frequentemente calculadas sobre janelas de tempo (e.g., vendas por minuto).

```java
// Exemplo conceitual de agregação em janela (detalhes no Módulo 4)
// Supondo que 'orders' tenha uma coluna 'o_event_time' como ROWTIME (atributo de tempo de evento)

// Table windowedAgg = orders
//     .window(Tumble.over(lit(10).minutes()).on($("o_event_time")).as("w")) // Janela de Tumbling de 10 min
//     .groupBy($("w"), $("o_product")) // Agrupa pela janela e produto
//     .select(
//         $("o_product"),
//         $("w").start().as("window_start"), // Início da janela
//         $("w").end().as("window_end"),     // Fim da janela
//         $("o_amount").sum().as("sum_amount_in_window") // Soma na janela
//     );

// System.out.println("\n--- Agregação em Janela (Soma de Vendas por Produto a cada 10 min) ---");
// // A execução de uma agregação em janela em um stream normalmente envia para outro sink ou tabela.
// // windowedAgg.execute().print(); // Requer uma fonte de stream e configuração de tempo para ser significativo.
```
Isso define janelas (`Tumble`, `Slide`, `Session`) e então agrupa e agrega dentro dessas janelas. Veremos isso em detalhes no Módulo 4.

---

### Operações de Conjunto (`Set Operations`)

Estas operações combinam os resultados de duas `Tables` com `schemas` compatíveis (mesmo número e tipos de colunas).

* **`union(Table other)`:** Retorna linhas que aparecem em qualquer uma das `Tables`, removendo duplicatas.
* **`unionAll(Table other)`:** Retorna linhas que aparecem em qualquer uma das `Tables`, mantendo todas as duplicatas. Este é geralmente o comportamento padrão para `union` na Table API se a distinção não for crucial ou para melhor desempenho, especialmente em *streams*.
* **`intersect(Table other)`:** Retorna apenas as linhas que aparecem em ambas as `Tables`, removendo duplicatas.
* **`intersectAll(Table other)`:** Retorna linhas que aparecem em ambas as `Tables`, mantendo duplicatas (o comportamento exato pode depender da semântica de *stream* vs *batch*).
* **`minus(Table other)` (ou `except`):** Retorna linhas da primeira `Table` que não aparecem na segunda, removendo duplicatas.
* **`minusAll(Table other)` (ou `exceptAll`):** Retorna linhas da primeira `Table` que não aparecem na segunda, considerando duplicatas.

```java
// --- Tabelas de Exemplo para Operações de Conjunto ---
// (Suponha que 'tableEnv' está definido)
// Table set1 = tableEnv.fromValues(
//     DataTypes.ROW(DataTypes.FIELD("id", DataTypes.INT()), DataTypes.FIELD("val", DataTypes.STRING())),
//     Row.of(1, "A"),
//     Row.of(2, "B"),
//     Row.of(3, "C")
// ).as("id", "val");

// Table set2 = tableEnv.fromValues(
//     DataTypes.ROW(DataTypes.FIELD("id", DataTypes.INT()), DataTypes.FIELD("val", DataTypes.STRING())),
//     Row.of(2, "B"),
//     Row.of(3, "C_modified"), // Diferente de (3, "C")
//     Row.of(4, "D")
// ).as("id", "val");

// Table set3_with_duplicates = tableEnv.fromValues(
//     DataTypes.ROW(DataTypes.FIELD("id", DataTypes.INT()), DataTypes.FIELD("val", DataTypes.STRING())),
//     Row.of(1, "A"),
//     Row.of(1, "A"), // Duplicata intencional
//     Row.of(2, "B")
// ).as("id", "val");


// --- unionAll ---
// Table unionAllResult = set1.unionAll(set2);
// System.out.println("\n--- unionAll(set1, set2) ---");
// unionAllResult.execute().print(); // Resultado: (1,A), (2,B), (3,C), (2,B), (3,C_modified), (4,D)

// --- union (remove duplicatas do resultado combinado) ---
// Table unionResult = set1.union(set2);
// System.out.println("\n--- union(set1, set2) ---");
// unionResult.execute().print(); // Resultado: (1,A), (2,B), (3,C), (3,C_modified), (4,D) - (2,B) aparece uma vez

// --- intersect (linhas comuns, sem duplicatas) ---
// Table intersectResult = set1.intersect(set3_with_duplicates);
// System.out.println("\n--- intersect(set1, set3_with_duplicates) ---");
// intersectResult.execute().print(); // Resultado: (1,A), (2,B)

// --- intersectAll (linhas comuns, com multiplicidade - o Flink pode tratar como intersect normal para streams) ---
// Table intersectAllResult = set1.intersectAll(set3_with_duplicates);
// System.out.println("\n--- intersectAll(set1, set3_with_duplicates) ---");
// intersectAllResult.execute().print(); // Para Table API, o comportamento de ALL pode ser o mesmo que a versão não-ALL em alguns contextos.

// --- minus (set1 - set2, sem duplicatas) ---
// Table minusResult = set1.minus(set2);
// System.out.println("\n--- minus(set1, set2) ---");
// minusResult.execute().print(); // Resultado: (1,A), (3,C)

// --- minusAll (set1 - set2, com multiplicidade) ---
// Table table_A = tableEnv.fromValues(Row.of(1,"X"), Row.of(1,"X"), Row.of(2,"Y")).as("id", "val");
// Table table_B = tableEnv.fromValues(Row.of(1,"X"), Row.of(3,"Z")).as("id", "val");
// Table minusAllResult = table_A.minusAll(table_B);
// System.out.println("\n--- minusAll(table_A, table_B) ---");
// minusAllResult.execute().print(); // Resultado: (1,X), (2,Y) (uma ocorrência de (1,"X") de A é removida)
```
*Nota: O comportamento exato de `ALL` vs. não-`ALL` em operações de conjunto pode ter nuances dependendo da versão do Flink e se a operação é em *stream* ou *batch*. `unionAll` é a mais comum para *streams* devido à natureza de anexar dados.*

---

### Ordenação e Limitação de Resultados (`Ordering and Limiting`)

* **`orderBy(Expression... fields)`:** Ordena a `Table` com base nos campos especificados. Use `.asc()` (padrão) ou `.desc()`.
    * Em *streaming*, uma ordenação global sem janelas só é possível em `Tables` que são continuamente atualizadas (e.g., *upsert stream* para um *changelog*) ou em *streams* que são convertidos para `Tables` em modo *batch*. Para *streams* de apensamento puro, `orderBy` geralmente é usado com janelas ou `LIMIT` para produzir resultados significativos.
* **`limit(int fetch)` ou `Workspace(int fetch)`:** Retorna as primeiras `N` linhas.
    * Para *streams* de apensamento ilimitados, `LIMIT` sem uma ordenação ou janela pode não ser muito útil ou pode ter semântica de "as primeiras N que chegam". Com `ORDER BY`, pode se referir a um resultado que muda ao longo do tempo (Top-N contínuo).
* **`offset(int offset)`:** Pula as primeiras `N` linhas.

```java
// Reutilizando a tabela 'customers' do exemplo de Joins
// (Suponha que 'tableEnv' e 'customers' estão definidos e populados)

// --- Order By ---
// Table orderedCustomers = customers.orderBy($("c_city").asc(), $("c_name").desc());
// System.out.println("\n--- Clientes Ordenados por Cidade (ASC) e Nome (DESC) ---");
// orderedCustomers.execute().print(); // Em modo batch, ou para tabelas de resultado atualizáveis.

// --- Limit ---
// Table limitedCustomers = customers.orderBy($("c_id")).limit(2); // Adicionado orderBy para resultado determinístico
// System.out.println("\n--- Clientes Limitados aos 2 Primeiros (ordenado por ID) ---");
// limitedCustomers.execute().print();

// --- Order By com Limit (Top-N) ---
// Table top2ByName = customers
//     .orderBy($("c_name").asc()) // Ordena por nome
//     .limit(2);                 // Pega os 2 primeiros
// System.out.println("\n--- Top 2 Clientes por Nome (ASC) ---");
// top2ByName.execute().print();

// --- Offset com Limit (Paginação) ---
// Table paginatedCustomers = customers
//     .orderBy($("c_id").asc()) // Precisa de uma ordem estável para paginação
//     .offset(1)              // Pula o primeiro
//     .limit(2);               // Pega os próximos 2
// System.out.println("\n--- Paginação: Clientes 2 e 3 (ID ordenado) ---");
// paginatedCustomers.execute().print();
```
*Nota: A execução de `orderBy`, `limit`, `offset` em *streams* ilimitados pode exigir que o Flink mantenha um estado considerável, especialmente para `orderBy`. Em modo *streaming*, estas operações são frequentemente usadas em conjunto com janelas ou em cenários de *upsert/retraction streams* para produzir resultados significativos e gerenciáveis.*

---

### Exercícios

Use o `TableEnvironment` e crie `Tables` de exemplo com `fromValues` para resolver os exercícios.

**1. `INNER JOIN`:**
Dadas as `Tables` `Employees` (`emp_id` INT, `emp_name` STRING, `dept_id` INT) e `Departments` (`dept_id` INT, `dept_name` STRING):
    * `Employees`: (1, "Alice", 101), (2, "Bob", 102), (3, "Charlie", 101), (4, "David", 103)
    * `Departments`: (101, "Engineering"), (102, "Sales"), (104, "HR")
Crie uma `Table` que mostre o nome do empregado (`emp_name`) e o nome do departamento (`dept_name`) para todos os empregados que pertencem a um departamento listado. Qual empregado não aparecerá no resultado?
    * a) Alice
    * b) Bob
    * c) Charlie
    * d) David

**2. `Grouping and Aggregation`:**
Dada uma `Table` `Sales` (`product_id` STRING, `category` STRING, `amount` DOUBLE, `sale_date` DATE):
    * Crie dados de exemplo.
    * Calcule o total de vendas (`amount`) para cada `category`.
    * Depois, filtre para mostrar apenas as categorias cujo total de vendas seja superior a 1000.00.

**3. `Set Operation`:**
Dadas duas `Tables`, `StreamA` e `StreamB`, ambas com um `schema` (`id` INT, `value` STRING):
    * `StreamA`: (1, "apple"), (2, "banana")
    * `StreamB`: (2, "banana"), (3, "cherry")
Qual seria o resultado de `StreamA.unionAll(StreamB)`? E de `StreamA.union(StreamB)`? E de `StreamA.minus(StreamB)`?

**4. `Ordering and Limiting`:**
Dada a `Table` `Players` (`player_id` INT, `player_name` STRING, `score` INT):
    * `Players`: (10, "Zidane", 95), (20, "Ronaldo", 98), (30, "Messi", 99), (40, "Pele", 97)
    * Escreva uma consulta para obter os nomes dos 2 jogadores com as maiores pontuações (`score`), ordenados da maior para a menor pontuação.

**5. Tipo de Junção para "Todos os pedidos e, se houver, seus clientes":**
Se você tem uma tabela `Pedidos` e uma tabela `Clientes`, e quer listar todos os pedidos, incluindo as informações do cliente se o cliente existir, ou `NULL` para as informações do cliente caso contrário, qual tipo de junção você usaria (com `Pedidos` sendo a tabela "principal" no fluxo da consulta)?
    * a) `INNER JOIN`
    * b) `Clientes.leftOuterJoin(Pedidos, ...)`
    * c) `Pedidos.leftOuterJoin(Clientes, ...)`
    * d) `FULL OUTER JOIN`

---

Espero que estas adições tenham tornado o módulo ainda mais claro!

Podemos prosseguir para o **Módulo 4: Trabalhando com Tempo na Table API** quando você estiver pronto.