## Módulo 4: Trabalhando com Tempo na Table API

Bem-vindo ao Módulo 4! Neste módulo, vamos explorar como o Apache Flink lida com o conceito de tempo em suas Table API e SQL. Entender e utilizar corretamente os diferentes atributos de tempo e os mecanismos de janelamento (*windowing*) é essencial para construir aplicações de *streaming* precisas e robustas.

Cobriremos:
* Atributos de Tempo: `Processing Time` e `Event Time`.
* O papel vital dos `Watermarks` no processamento em `Event Time`.
* Conceitos de Janelas (*Windowing Concepts*): `Tumbling`, `Sliding`, `Session` e `Cumulative windows`.
* Como aplicar funções de agregação sobre essas janelas.

---

### Atributos de Tempo (`Time Attributes`)

No Flink, o tempo pode ser interpretado de duas maneiras principais quando se processa *streams* de dados:

**1. `Processing Time` (Tempo de Processamento)**

* **Definição:** Refere-se ao tempo do sistema da máquina que está executando a respectiva operação (o tempo do relógio do *worker* do Flink).
* **Características:**
    * **Simplicidade:** Não requer nenhuma coordenação especial ou extração de *timestamps* dos dados. É o mais fácil de usar.
    * **Melhor Performance:** Geralmente oferece a menor latência, pois os eventos são processados assim que chegam.
    * **Não Determinístico:** Os resultados podem variar dependendo da velocidade de chegada dos dados e da performance do sistema. Se houver uma falha e os dados forem reprocessados, ou se a carga do sistema variar, os mesmos dados podem cair em janelas diferentes, levando a resultados diferentes.
    * **Inadequado para Dados Históricos ou Fora de Ordem:** Não consegue lidar corretamente com eventos que chegam com atraso significativo ou em desordem em relação ao tempo em que realmente ocorreram.
* **Como Definir:**
    * **Em `DataStream` para `Table`:** Usando a função `.proctime()` ao definir o `schema`.
        ```java
        // Suponha que tableEnv é um StreamTableEnvironment e dataStream é um DataStream<Row>
        // import static org.apache.flink.table.api.Expressions.$;

        // Table myTable = tableEnv.fromDataStream(
        //     dataStream, // Supondo que dataStream tenha uma coluna 'f0'
        //     $("f0").as("data_field"),
        //     $("processing_time_col").proctime() // Define uma coluna de tempo de processamento
        // );
        // System.out.println("Schema com Processing Time:");
        // myTable.printSchema();
        ```
    * **Em DDL SQL:** Usando a função `PROCTIME()` como uma *computed column*.
        ```sql
        CREATE TABLE MySourceTable (
          `data_field` STRING,
          `processing_time_col` AS PROCTIME() -- Define uma coluna de tempo de processamento
        ) WITH (
          'connector' = 'datagen' -- Exemplo com conector datagen
          -- outras opções do conector
        );
        ```
        A coluna `processing_time_col` pode então ser usada em operações baseadas em tempo, como janelas.

**2. `Event Time` (Tempo do Evento)**

* **Definição:** Refere-se ao tempo em que o evento realmente ocorreu em seu dispositivo de origem ou foi gerado. Este *timestamp* é tipicamente embarcado no próprio registro do evento.
* **Características:**
    * **Determinístico e Consistente:** Os resultados das análises são consistentes e reproduzíveis, independentemente de quando ou como os dados são processados. Os eventos são atribuídos a janelas com base em seus próprios *timestamps*.
    * **Lida com Dados Fora de Ordem:** Permite que o sistema espere por eventos atrasados (até um certo ponto, gerenciado por *watermarks*).
    * **Requer `Watermarks`:** Para informar ao sistema até que ponto no tempo do evento ele pode esperar por mais dados.
    * **Pode Introduzir Latência:** O sistema pode precisar esperar por `watermarks` para progredir, o que pode introduzir alguma latência em comparação com o `processing time`.
* **Como Definir:**
    * É necessário um campo na sua `Table` que represente o `event time`. Este campo deve ser do tipo `TIMESTAMP(p)` ou `TIMESTAMP_LTZ(p)`.
    * Você deve declarar este campo como o atributo de tempo de evento e definir uma estratégia de `Watermark`.
    * **Em `DataStream` para `Table` (exemplo conceitual):**
        ```java
        // import org.apache.flink.streaming.api.datastream.DataStream;
        // import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
        // import org.apache.flink.table.api.DataTypes;
        // import org.apache.flink.table.api.Schema;
        // import org.apache.flink.table.api.Table;
        // import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
        // import org.apache.flink.types.Row;
        // import static org.apache.flink.table.api.Expressions.$;

        // StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
        // // Supondo que Row tenha (String data_field, Long event_ts_millis)
        // DataStream<Row> dataStreamWithTimestamp = env.fromElements(
        //     Row.of("A", 1000L), Row.of("B", 2000L), Row.of("C", 1500L) // Evento C está fora de ordem
        // );

        // Table myEventTimeTable = tableEnv.fromDataStream(
        //     dataStreamWithTimestamp,
        //     Schema.newBuilder()
        //         .column("f0", DataTypes.STRING()) // Acessando campos do Row
        //         .column("f1", DataTypes.BIGINT()) // O timestamp original em milissegundos
        //         // Converte o BIGINT para TIMESTAMP_LTZ(3) e o designa como ROWTIME
        //         .columnByExpression("event_time_col", "TO_TIMESTAMP_LTZ(f1, 3)")
        //         // Define a estratégia de watermark para a coluna event_time_col
        //         .watermark("event_time_col", "$('event_time_col') - INTERVAL '5' SECOND")
        //         .build()
        // )
        // .select($("f0").as("data_field"), $("event_time_col")); // Seleciona as colunas finais

        // System.out.println("Schema com Event Time e Watermark:");
        // myEventTimeTable.printSchema();
        ```
    * **Em DDL SQL:**
        ```sql
        CREATE TABLE MyEventSourceTable (
          `data_field` STRING,
          `event_ts_millis` BIGINT, -- Timestamp original da fonte
          -- Define 'event_time_col' como uma computed column convertida para TIMESTAMP
          `event_time_col` AS TO_TIMESTAMP_LTZ(`event_ts_millis`, 3),
          -- Declara 'event_time_col' como o atributo de tempo de evento
          -- e define a estratégia de watermark
          WATERMARK FOR `event_time_col` AS `event_time_col` - INTERVAL '5' SECOND
        ) WITH (
          'connector' = 'datagen', -- Exemplo
          'fields.event_ts_millis.kind' = 'sequence',
          'fields.event_ts_millis.start' = '0',
          'fields.event_ts_millis.end' = '1000000'
          -- outras opções do conector
        );
        ```

**3. `Ingestion Time` (Tempo de Ingestão)**

* É um caso especial de `event time`. O *timestamp* do evento é atribuído pela fonte do Flink no momento em que o evento entra no *pipeline* Flink.
* Pode ser uma boa opção se os *timestamps* originais não são confiáveis ou não existem, e você ainda quer alguma forma de ordenação baseada no tempo de chegada ao Flink, mas com mais consistência que o `processing time` puro.
* A definição é similar ao `event time`, mas o *timestamp* é gerado na borda do Flink.

**`Watermarks` (Marcas d'Água): O Guia do Tempo de Evento**

No processamento em `event time`, onde os eventos são processados com base nos *timestamps* que eles carregam, o sistema enfrenta dois desafios principais: os *streams* são infinitos e os eventos podem chegar fora de ordem. Como, então, o Flink pode saber que todos os eventos para um determinado período de tempo (por exemplo, uma janela de 10:00 AM a 10:05 AM) já chegaram e que a janela pode ser processada e seu resultado finalizado? É aqui que entram os `watermarks`.

* **O que são?**
    `Watermarks` são um mecanismo fundamental no Flink que atua como um **sinalizador do progresso do `event time`**. Um `Watermark(t)` é uma declaração do Flink de que ele não espera mais ver eventos com um *timestamp* `ts <= t`. Em outras palavras, ele diz: "Até onde sabemos, todos os eventos que ocorreram até o tempo `t` já foram observados pelo sistema." Isso permite que o Flink feche janelas de `event time` que terminam em ou antes de `t` e processe os dados acumulados.

    Pense nos `watermarks` como uma espécie de "relógio lógico" para o `event time` do *stream*. Enquanto o relógio de parede (processing time) avança constantemente, este relógio lógico do `event time` só avança quando o Flink tem evidências (através dos *timestamps* dos eventos e da lógica de geração de `watermarks`) de que o tempo real dos eventos progrediu.

* **Por que são Essenciais?**
    1.  **Fechamento de Janelas:** Em *streams* infinitos, sem `watermarks`, o Flink nunca saberia quando uma janela de `event time` (ex: "todos os eventos entre 10:00 e 10:05") está completa. Ele poderia esperar indefinidamente por eventos que podem ter se atrasado. `Watermarks` fornecem o gatilho para calcular e emitir o resultado de uma janela.
    2.  **Manuseio de Dados Fora de Ordem:** Eventos nem sempre chegam na ordem em que foram gerados. `Watermarks` permitem que o sistema espere por um período configurável por esses eventos "atrasados" antes de finalizar uma janela.
    3.  **Consistência:** Ao usar `event time` e `watermarks`, você obtém resultados consistentes e reproduzíveis, mesmo que os dados sejam reprocessados ou a velocidade de ingestão varie.

* **Analogia: A Partida do Trem**

    Imagine uma estação de trem onde um trem está programado para partir às 10:00 AM (este é o fim de uma "janela de embarque").
    * **Passageiros (Eventos):** Os passageiros chegam à estação com bilhetes que têm um horário (o `event time`). Alguns chegam cedo, outros em cima da hora, e alguns podem se atrasar um pouco.
    * **Plataforma de Embarque (Operador Flink/Janela):** Acumula os passageiros para o trem das 10:00 AM.
    * **Chefe da Estação (Gerador de `Watermark`):** O chefe da estação tem uma política para lidar com atrasos. Ele sabe que alguns passageiros podem se atrasar até, digamos, 5 minutos.
    * **O `Watermark`:** Às 10:00 AM (fim da janela), o chefe da estação não fecha as portas imediatamente. Ele espera. Ele observa os passageiros chegando. Quando o relógio da estação (que ele usa para sua decisão, não o relógio de cada passageiro) marca 10:05 AM, ele declara: "Não esperamos mais passageiros para o trem das 10:00 AM. Todos que deveriam ter chegado até as 10:00 AM (e tiveram uma tolerância de 5 minutos) e quiseram pegar este trem já deveriam estar aqui." Este anúncio é o `Watermark(10:00 AM)`.
    * **Partida do Trem (Processamento da Janela):** Com base no `watermark` do chefe da estação, as portas do trem das 10:00 AM são fechadas, e o trem parte (a janela é processada).
    * **Passageiros Muito Atrasados:** Se um passageiro com bilhete para as 10:00 AM chegar às 10:06 AM, ele perdeu o trem (o evento é considerado "tardio" para aquela janela e pode ser descartado ou tratado de forma especial).

    Nesta analogia:
    * O `event time` é o horário no bilhete do passageiro.
    * O `processing time` seria o relógio real da estação no momento em que cada passageiro *passa pela catraca*.
    * O `maxOutOfOrderness` (ou a folga) é os 5 minutos que o chefe da estação espera.
    * O `watermark` é a declaração do chefe da estação sobre o quão longe o "tempo do bilhete" progrediu de forma confiável.

* **Estratégias de Geração de `Watermarks`:**

    O Flink precisa de uma `WatermarkStrategy` para saber como gerar esses `watermarks` a partir dos *timestamps* dos eventos. As duas estratégias mais comuns são:

    1.  **`WatermarkStrategy.forBoundedOutOfOrderness(Duration maxOutOfOrderness)`:**
        * Esta é a estratégia mais utilizada na prática. Ela assume que os eventos podem chegar fora de ordem, mas dentro de um limite de atraso máximo conhecido (`maxOutOfOrderness`).
        * **Como funciona:** O gerador de `watermark` rastreia o maior *timestamp* de evento (`maxTimestamp`) visto até agora no *stream* (por partição). O `watermark` emitido é então `maxTimestamp - maxOutOfOrderness`.
        * **Exemplo:** Se `maxOutOfOrderness` é de 5 segundos, e o maior *timestamp* de evento visto até agora é `10:00:30`, o `watermark` gerado será `10:00:25`. Isso significa que o Flink assume que todos os eventos que ocorreram às `10:00:25` ou antes já chegaram.

    2.  **`WatermarkStrategy.forMonotonousTimestamps()`:**
        * Esta estratégia é usada quando os *timestamps* dos eventos são garantidamente monotônicos crescentes (ou seja, não há eventos fora de ordem em relação ao *timestamp*).
        * **Como funciona:** O `watermark` gerado é simplesmente o *timestamp* do evento atual (ou `maxTimestamp` visto, que será o do evento atual).
        * É mais simples e pode ter menor latência, mas é menos robusta a desordens reais nos dados.

    O Flink também permite a implementação de `WatermarkGenerator` personalizados para lógicas mais complexas, se necessário (geralmente usado com a DataStream API).

* **Diagrama: Progressão de `Watermarks` com `BoundedOutOfOrderness`**

    Suponha `maxOutOfOrderness = 2 segundos`.
    Os `Watermarks` são tipicamente gerados periodicamente (e.g., a cada 200ms por padrão) ou por evento, dependendo da configuração da `WatermarkStrategy`.

    ```
    Eventos (timestamp em segundos):
    E1(2s)  E2(1s)  E3(3s)     E4(5s)  E5(4s)         E6(8s)  E7(6s) E8(10s)
    |-------|-------|-------|-------|-------|-------|-------|-------|-------|-----> Tempo Real (Processing Time)

    Instante | Evento Chegando | max_event_ts_visto | Watermark Emitido (max_ts - 2s) | Janelas que podem fechar
    ---------|-----------------|--------------------|---------------------------------|-----------------------------------
    T1       | E1 (ts=2s)      | 2s                 | 0s                              | Janelas com fim <= 0s
    T2       | E2 (ts=1s)      | 2s (1s não é > 2s) | 0s                              | Janelas com fim <= 0s
    T3       | E3 (ts=3s)      | 3s                 | 1s                              | Janelas com fim <= 1s
    T4       | (sem evento)    | 3s                 | 1s (emissão periódica)          | Janelas com fim <= 1s
    T5       | E4 (ts=5s)      | 5s                 | 3s                              | Janelas com fim <= 3s (ex: [0s-3s])
    T6       | E5 (ts=4s)      | 5s (4s não é > 5s) | 3s                              | Janelas com fim <= 3s
    T7       | (sem evento)    | 5s                 | 3s (emissão periódica)          | Janelas com fim <= 3s
    T8       | E6 (ts=8s)      | 8s                 | 6s                              | Janelas com fim <= 6s (ex: [3s-6s])
    T9       | E7 (ts=6s)      | 8s (6s não é > 8s) | 6s                              | Janelas com fim <= 6s
    T10      | E8 (ts=10s)     | 10s                | 8s                              | Janelas com fim <= 8s (ex: [6s-8s])
    ```
    Neste diagrama:
    * `max_event_ts_visto` é o maior *timestamp* de evento encontrado até aquele ponto na partição.
    * O `Watermark Emitido` "persegue" o `max_event_ts_visto`, mas com uma defasagem igual ao `maxOutOfOrderness`.
    * Quando o `Watermark` ultrapassa o horário de término de uma janela (ex: uma janela de 0s a 5s), essa janela pode ser processada se o `Watermark` for `> 5s`.

* **Como `Watermarks` são Definidos na Table API/SQL:**

    Como vimos anteriormente, você define `watermarks` no `schema` da sua `Table`:

    * **DDL SQL:**
        ```sql
        CREATE TABLE EventStream (
          user_id STRING,
          event_data STRING,
          event_ts BIGINT, -- Timestamp original em milissegundos
          -- Converte para TIMESTAMP e define como o atributo de tempo do evento
          row_time AS TO_TIMESTAMP_LTZ(event_ts, 3),
          -- Define a estratégia de watermark para a coluna row_time
          WATERMARK FOR row_time AS row_time - INTERVAL '5' SECOND
        ) WITH (...);
        ```
        Aqui, `row_time - INTERVAL '5' SECOND` implementa uma estratégia similar à `BoundedOutOfOrderness` com uma folga de 5 segundos.

    * **Programaticamente com `Schema.newBuilder()` (ao converter `DataStream`):**
        ```java
        // import org.apache.flink.table.api.Schema;
        // import org.apache.flink.table.api.DataTypes;
        // import static org.apache.flink.table.api.Expressions.$;

        // Schema eventSchema = Schema.newBuilder()
        //     .column("event_ts_long", DataTypes.BIGINT())
        //     .columnByExpression("row_time", "TO_TIMESTAMP_LTZ($('event_ts_long'), 3)")
        //     .watermark("row_time", "$('row_time') - INTERVAL '5' SECOND")
        //     .build();
        ```
    Uma vez que o atributo de `event time` e a estratégia de `watermark` são definidos no `schema`, o Flink os utiliza automaticamente em operações de janela que operam sobre essa coluna de tempo.

* **Impacto e Considerações:**
    * **Correção:** `Watermarks` são a base para a correção em janelas de `event time`.
    * **Completude vs. Latência:** A escolha da estratégia de `watermark` (especialmente o valor de `maxOutOfOrderness`) é um equilíbrio:
        * Um `maxOutOfOrderness` **maior** (mais folga) permite que mais eventos fora de ordem sejam incluídos corretamente, aumentando a completude, mas também aumenta a latência, pois o sistema espera mais antes de fechar as janelas.
        * Um `maxOutOfOrderness` **menor** (menos folga) reduz a latência, mas pode fazer com que mais eventos sejam considerados "tardios" e potencialmente descartados das janelas.
    * **Por Partição:** `Watermarks` são geralmente gerados e processados por partição/paralelismo do *stream*. O `watermark` "global" que o Flink considera para operadores *downstream* (como uma junção de dois *streams*) é tipicamente o mínimo dos `watermarks` de todas as suas partições de entrada.

---

### Conceitos de Janelas (`Windowing Concepts`)

Janelas dividem um *stream* (potencialmente infinito) em "baldes" finitos sobre os quais podemos aplicar cálculos (como agregações). A Table API e SQL do Flink suportam diferentes tipos de janelas, principalmente através de *Table-Valued Functions (TVFs)*.

**1. `Tumbling Windows` (Janelas de Salto / Janelas Fixas Não Sobrepostas)**

* **Definição:** Dividem o *stream* em janelas de tamanho fixo, contíguas e que não se sobrepõem. Cada evento pertence a exatamente uma janela.
* **Parâmetros:** Tamanho da janela (`size`).
* **Casos de Uso:** Contagens por minuto, soma de vendas por hora.
* **Sintaxe (Table API usando TVF `Tumble`):**
    ```java
    // import static org.apache.flink.table.api.Expressions.lit;
    // import org.apache.flink.table.api.Tumble;
    // Suponha 'inputStream' com coluna 'event_time' como ROWTIME
    // Table result = inputStream.window(Tumble.over(lit(10).minutes()) // Tamanho da janela: 10 minutos
    //                              .on($("event_time"))      // Coluna de tempo (event ou processing)
    //                              .as("w"))                 // Nome da referência da janela
    //                    .groupBy($("w"), $("key_col"))      // Agrupa pela janela e chave
    //                    .select(
    //                        $("key_col"),
    //                        $("w").start().as("window_start"),
    //                        $("w").end().as("window_end"),
    //                        $("value_col").sum().as("sum_val")
    //                    );
    ```
* **Sintaxe (SQL usando `TUMBLE` UDTF):**
    ```sql
    SELECT
      key_col,
      TUMBLE_START(event_time, INTERVAL '10' MINUTE) AS window_start,
      TUMBLE_END(event_time, INTERVAL '10' MINUTE) AS window_end,
      SUM(value_col) AS sum_val
    FROM MyEventSourceTable -- Tabela com 'event_time' como atributo de tempo
    GROUP BY
      key_col,
      TUMBLE(event_time, INTERVAL '10' MINUTE) -- Define a janela de tumbling
    ```
* **Diagrama:**
    ```
    Stream: |e1|e2|e3|e4|e5|e6|e7|e8|... (eX = evento)
    Tempo:  ---------------------------------->
    Janelas:
    [------- W1 -------]
            (size)
                       [------- W2 -------]
                               (size)
                                          [------- W3 -------]
                                                  (size)
    Eventos em W1: e1, e2, e3
    Eventos em W2: e4, e5, e6
    Eventos em W3: e7, e8
    ```

**2. `Sliding Windows` (Janelas Deslizantes)**

* **Definição:** Janelas de tamanho fixo que "deslizam" pelo *stream* em um intervalo fixo (o *slide*). Se o *slide* for menor que o tamanho da janela, as janelas se sobrepõem. Um evento pode pertencer a múltiplas janelas.
* **Parâmetros:** Tamanho da janela (`size`), intervalo de deslize (`slide`).
* **Casos de Uso:** Média móvel dos últimos 5 minutos, atualizada a cada minuto.
* **Sintaxe (Table API usando TVF `Slide`):**
    ```java
    // import org.apache.flink.table.api.Slide;
    // Table result = inputStream.window(Slide.over(lit(10).minutes())  // Tamanho da janela: 10 minutos
    //                               .every(lit(5).minutes())   // Desliza a cada 5 minutos
    //                               .on($("event_time"))
    //                               .as("w"))
    //                    .groupBy($("w"), $("key_col"))
    //                    .select(...); // Similar ao tumbling
    ```
* **Sintaxe (SQL usando `HOP` UDTF - atenção, nome diferente do Table API):**
    ```sql
    SELECT
      key_col,
      HOP_START(event_time, INTERVAL '5' MINUTE, INTERVAL '10' MINUTE) AS window_start, -- (timeCol, slide, size)
      HOP_END(event_time, INTERVAL '5' MINUTE, INTERVAL '10' MINUTE) AS window_end,
      SUM(value_col) AS sum_val
    FROM MyEventSourceTable
    GROUP BY
      key_col,
      HOP(event_time, INTERVAL '5' MINUTE, INTERVAL '10' MINUTE) -- Define a janela deslizante
    ```
* **Diagrama (size=10min, slide=5min):**
    ```
    Stream: |e1|e2|e3|e4|e5|e6|e7|e8|...
    Tempo:  ---------------------------------->
    Janelas:
    [----------- W1 (0-10min) -----------]
                (size)
        [----------- W2 (5-15min) -----------]
                    (slide)
            [----------- W3 (10-20min) -----------]

    Eventos em W1: e1..eX (que caem entre 0 e 10 min)
    Eventos em W2: eY..eZ (que caem entre 5 e 15 min) - eY pode ser igual a algum evento em W1
    ```

**3. `Session Windows` (Janelas de Sessão)**

* **Definição:** Agrupam eventos com base em períodos de atividade, separados por um "gap" (intervalo de inatividade). Não têm tamanho fixo nem início/fim fixos; são dinâmicas. Uma nova sessão começa quando um evento chega após o `session_gap` ter expirado desde o último evento para uma determinada chave.
* **Parâmetros:** Intervalo de inatividade (`session_gap`).
* **Casos de Uso:** Análise de comportamento do usuário (e.g., cliques em um site), monitoramento de sensores onde dados só chegam quando há atividade.
* **Sintaxe (Table API usando TVF `Session`):**
    ```java
    // import org.apache.flink.table.api.Session;
    // Table result = inputStream.window(Session.withGap(lit(30).minutes()) // Gap de inatividade: 30 minutos
    //                                 .on($("event_time"))
    //                                 .as("w"))
    //                    .groupBy($("w"), $("user_id")) // Agrupa pela janela de sessão e usuário
    //                    .select(...);
    ```
* **Sintaxe (SQL usando `SESSION` UDTF):**
    ```sql
    SELECT
      user_id,
      SESSION_START(event_time, INTERVAL '30' MINUTE) AS session_start,
      SESSION_END(event_time, INTERVAL '30' MINUTE) AS session_end,
      COUNT(*) AS events_in_session
    FROM MyEventSourceTable
    GROUP BY
      user_id,
      SESSION(event_time, INTERVAL '30' MINUTE) -- Define a janela de sessão
    ```
* **Diagrama (gap=X):**
    ```
    Stream (para um user_id): |e1| |e2|e3| | | |e4|e5| |e6|...
    Tempo:  -------------------------------------------------->
             ^  ^   ^        ^   ^   ^
             ts1 ts2 ts3      ts4 ts5 ts6

    Sessões (gap=X):
    [---- S1 ----]  se (ts2-ts1 < X) e (ts3-ts2 < X)
                   (ts4-ts3 > X) -> S1 fecha, S2 começa com e4
                             [---- S2 ----] se (ts5-ts4 < X)
                                            (ts6-ts5 > X) -> S2 fecha, S3 começa com e6
                                                      [-- S3 --] ...
    Eventos em S1: e1, e2, e3
    Eventos em S2: e4, e5
    ```

**4. `Cumulative Windows` (Janelas Cumulativas - Flink 1.18+)**

* **Definição:** Produzem múltiplos painéis de resultados por janela, acumulando desde o início de um período maior até um máximo, com um tamanho de passo. Por exemplo, para uma janela máxima diária com passo horário, você teria resultados para a primeira hora, depois para as duas primeiras horas acumuladas, depois três primeiras horas, e assim por diante, até o acumulado do dia.
* **Parâmetros:** Tamanho máximo da janela (`max_size`), tamanho do passo (`step`).
* **Casos de Uso:** Relatórios que precisam mostrar totais parciais e o total acumulado ao longo de um período maior (e.g., vendas acumuladas por hora dentro de um dia).
* **Sintaxe (Table API usando TVF `Cumulate`):**
    ```java
    // import org.apache.flink.table.api.Cumulate;
    // Table result = inputStream.window(Cumulate.over(lit(1).day())     // Tamanho máximo da janela: 1 dia
    //                                   .step(lit(1).hour())    // Passo de acumulação: 1 hora
    //                                   .on($("event_time"))
    //                                   .as("w"))
    //                    .groupBy($("w"), $("key_col"))
    //                    .select(...);
    ```
* **Sintaxe (SQL usando `CUMULATE` UDTF):**
    ```sql
    SELECT
      key_col,
      CUMULATE_START(event_time, INTERVAL '1' HOUR, INTERVAL '1' DAY) AS window_start, -- (timeCol, step, maxSize)
      CUMULATE_END(event_time, INTERVAL '1' HOUR, INTERVAL '1' DAY) AS window_end,
      SUM(value_col) AS cumulative_sum
    FROM MyEventSourceTable
    GROUP BY
      key_col,
      CUMULATE(event_time, INTERVAL '1' HOUR, INTERVAL '1' DAY)
    ```
* **Diagrama (max_size=1day, step=1hour):**
    ```
    Tempo: |----H1----|----H2----|----H3----| ... |----H24----| (Dia)
    Resultados para uma chave (a cada hora, o resultado é o acumulado desde o início do dia até o fim daquela hora):
    - Fim de H1: Acumulado de H1 (janela: DiaInício a H1Fim)
    - Fim de H2: Acumulado de H1+H2 (janela: DiaInício a H2Fim)
    - Fim de H3: Acumulado de H1+H2+H3 (janela: DiaInício a H3Fim)
    ...
    - Fim de H24: Acumulado de H1+...+H24 (janela: DiaInício a H24Fim - total do dia)
    ```

---

### Funções de Janela (`Window Functions` / `Windowed Aggregation`)

Após definir uma janela, você tipicamente aplica funções de agregação (como `SUM()`, `COUNT()`, `AVG()`, `MAX()`, `MIN()`) aos eventos que caem dentro de cada instância da janela para cada chave de agrupamento.

**Acesso a Metadados da Janela:**
Dentro da cláusula `select` de uma agregação em janela, você pode acessar propriedades da janela usando o nome que você deu a ela (e.g., `w` no exemplo `as("w")`):
* `w.start`: Retorna o *timestamp* de início da janela.
* `w.end`: Retorna o *timestamp* de fim da janela.
* `w.rowtime`: Retorna o *timestamp* de `event time` da janela (geralmente igual ao `w.end` para `Tumbling` e `Sliding`).
* `w.proctime`: Retorna o *timestamp* de `processing time` da janela (se a janela for baseada em `processing time`).

**Exemplos de Código com Agregação em Janela:**

Vamos usar uma `Table` `InputStream` com `user_id` (STRING), `amount` (INT), e `event_time` (TIMESTAMP_LTZ(3) como ROWTIME).

```java
import org.apache.flink.table.api.*;
import org.apache.flink.types.Row;
import static org.apache.flink.table.api.Expressions.*;

// Configuração do ambiente e da tabela de entrada (exemplo)
// EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().build();
// TableEnvironment tableEnv = TableEnvironment.create(settings);

// Table inputStream = tableEnv.fromValues(
//     DataTypes.ROW(
//         DataTypes.FIELD("user_id", DataTypes.STRING()),
//         DataTypes.FIELD("amount", DataTypes.INT()),
//         DataTypes.FIELD("event_time_long", DataTypes.BIGINT())
//     ),
//     Row.of("user1", 10, 1000L),  // 00:00:01
//     Row.of("user2", 20, 2000L),  // 00:00:02
//     Row.of("user1", 15, 6000L),  // 00:00:06 (para janela 5-10s)
//     Row.of("user1", 25, 7000L),  // 00:00:07 (para janela 5-10s)
//     Row.of("user2", 30, 11000L), // 00:00:11 (para janela 10-15s)
//     Row.of("user1", 5, 12000L)   // 00:00:12 (para janela 10-15s)
// )
// .map( // Mapeia para definir event_time e watermarks
//     Schema.newBuilder()
//         .column("user_id", DataTypes.STRING())
//         .column("amount", DataTypes.INT())
//         .column("event_time_long", DataTypes.BIGINT()) // Coluna original não usada diretamente após
//         .columnByExpression("event_time", "TO_TIMESTAMP_LTZ(event_time_long, 3)") // Convertida
//         .watermark("event_time", "$('event_time') - INTERVAL '1' SECOND") // Watermark
//         .build()
// )
// .select($("user_id"), $("amount"), $("event_time")); // Seleciona as colunas finais

// System.out.println("Schema do InputStream:");
// inputStream.printSchema();


// 1. Tumbling Window (soma de 'amount' por 'user_id' a cada 5 segundos)
// Table tumblingWindowAgg = inputStream
//     .window(Tumble.over(lit(5).seconds()).on($("event_time")).as("w"))
//     .groupBy($("w"), $("user_id")) // Agrupa pela janela e user_id
//     .select(
//         $("user_id"),
//         $("w").start().as("window_start"),
//         $("w").end().as("window_end"),
//         $("amount").sum().as("total_amount")
//     );
// System.out.println("\nTumbling Window Aggregation (5 segundos):");
// tumblingWindowAgg.execute().print();
// Resultados esperados (aproximados, dependem da exata emissão de watermarks e dados):
// +I[user1, 1970-01-01T00:00:00Z, 1970-01-01T00:00:05Z, 10]
// +I[user2, 1970-01-01T00:00:00Z, 1970-01-01T00:00:05Z, 20]
// +I[user1, 1970-01-01T00:00:05Z, 1970-01-01T00:00:10Z, 40]  (15+25)
// +I[user2, 1970-01-01T00:00:10Z, 1970-01-01T00:00:15Z, 30]
// +I[user1, 1970-01-01T00:00:10Z, 1970-01-01T00:00:15Z, 5]


// 2. Sliding Window (soma de 'amount' nos últimos 10 segundos, deslizando a cada 5 segundos)
// Table slidingWindowAgg = inputStream
//     .window(Slide.over(lit(10).seconds()).every(lit(5).seconds()).on($("event_time")).as("w"))
//     .groupBy($("w"), $("user_id"))
//     .select(
//         $("user_id"),
//         $("w").start().as("window_start"),
//         $("w").end().as("window_end"),
//         $("amount").sum().as("total_amount")
//     );
// System.out.println("\nSliding Window Aggregation (10s size, 5s slide):");
// slidingWindowAgg.execute().print();

// 3. Session Window (contagem de eventos por 'user_id' com gap de 3 segundos de inatividade)
// Table sessionWindowAgg = inputStream
//     .window(Session.withGap(lit(3).seconds()).on($("event_time")).as("w"))
//     .groupBy($("w"), $("user_id")) // Precisa agrupar pela chave da sessão (user_id aqui)
//     .select(
//         $("user_id"),
//         $("w").start().as("session_start"),
//         $("w").end().as("session_end"), // O fim da sessão é o timestamp do último evento na sessão + gap
//         $("amount").count().as("event_count")
//     );
// System.out.println("\nSession Window Aggregation (3s gap):");
// sessionWindowAgg.execute().print();

// 4. Cumulative Window (soma de 'amount' acumulada a cada 5 segundos, até um máximo de 1 minuto)
// Table cumulativeWindowAgg = inputStream
//     .window(Cumulate.over(lit(1).minute()).step(lit(5).seconds()).on($("event_time")).as("w"))
//     .groupBy($("w"), $("user_id")) // Agrupa pela janela e user_id
//     .select(
//         $("user_id"),
//         $("w").start().as("window_start"), // Início do período de acumulação (e.g., início do minuto)
//         $("w").end().as("window_end"),     // Fim do painel atual (e.g., fim dos primeiros 5s, 10s, etc.)
//         $("amount").sum().as("cumulative_amount")
//     );
// System.out.println("\nCumulative Window Aggregation (max 1min, step 5s):");
// cumulativeWindowAgg.execute().print();

```
*Nota: Para executar estes exemplos, descomente-os e coloque-os em um método `main` com a inicialização apropriada do `TableEnvironment` e da `Table` `inputStream`. Os resultados exatos podem variar ligeiramente com base na temporização dos `watermarks` e na natureza do `DataStream` de origem.*

**Latência e Dados Atrasados (`Late Data`):**

* **Conceito:** `Late data` são eventos que chegam ao sistema após o `watermark` já ter passado o tempo do evento do próprio dado. Por exemplo, se o `watermark` atual é `10:05:00Z`, um evento com *timestamp* `10:04:58Z` é considerado atrasado se chegar neste momento.
* **Impacto:** Por padrão, no Flink Table API/SQL, eventos que chegam após o `watermark` ter ultrapassado o fim da janela à qual pertenceriam são **descartados** e não são incluídos na computação da janela.
* **Lidando com Dados Atrasados (Table API/SQL):**
    * **`Watermarks` Adequados:** A configuração correta do `maxOutOfOrderness` na sua estratégia de `watermark` é a principal forma de mitigar o problema de dados moderadamente atrasados, permitindo que eles sejam incluídos.
    * **`TableConfig.setIdleStateRetentionTime()`:** Esta configuração define por quanto tempo o estado de uma janela (ou de outras operações com estado, como `joins`) é mantido após o `watermark` passar. Se o estado for limpo muito cedo, mesmo que um dado não estivesse "tão atrasado" em relação ao `watermark`, a janela para ele já não existiria. Isso não "processa" dados atrasados na janela original, mas controla a limpeza do estado.
    * **Soluções Avançadas:** A Table API em si tem opções limitadas para processamento dedicado de dados atrasados (como o `allowedLateness()` da DataStream API que permite que uma janela permaneça ativa por mais tempo após o `watermark`, ou `sideOutputLateData()` para capturar dados atrasados em um *stream* separado). Para cenários que exigem um tratamento mais sofisticado de dados muito atrasados, pode ser necessário:
        * Usar a DataStream API para essas partes específicas da lógica.
        * Projetar a lógica de negócios para lidar com correções ou atualizações posteriores, se os dados atrasados forem armazenados em outro lugar.

---

### Exercícios

1.  **`Event Time` vs `Processing Time`:** Qual das seguintes afirmações NÃO é verdadeira sobre `event time`?
    * a) Garante resultados determinísticos e reproduzíveis.
    * b) Requer o uso de `watermarks` para lidar com dados fora de ordem.
    * c) É sempre o mais fácil de implementar e oferece a menor latência.
    * d) Permite que o sistema processe dados com base no momento em que os eventos realmente ocorreram.

2.  **Definição de `Watermark`:**
    Você tem uma `Table` `SensorReadings` com uma coluna `ts_millis` (BIGINT) representando o *timestamp* do evento em milissegundos. Escreva a parte da DDL SQL (`CREATE TABLE`) que define uma coluna `event_time` a partir de `ts_millis` e declara um `watermark` para `event_time` assumindo que os dados podem chegar com até 10 segundos de atraso.

3.  **Escolha da Janela Correta:**
    Para cada cenário abaixo, qual tipo de janela (`Tumbling`, `Sliding`, `Session`, `Cumulative`) seria mais apropriado?
    * a) Calcular o número total de visualizações de página a cada hora.
    * b) Analisar a duração das interações de um usuário com um aplicativo, onde uma interação termina após 30 minutos de inatividade do usuário.
    * c) Gerar um relatório diário que mostre o total de vendas por hora e também o total de vendas acumulado desde o início do dia para cada hora.
    * d) Calcular a média de temperatura dos últimos 15 minutos, com o cálculo sendo atualizado a cada minuto.

4.  **Agregação em Janela `Tumbling`:**
    Considerando a `Table` `InputStream` definida nos exemplos (com `user_id`, `amount`, `event_time`):
    Escreva uma consulta da Table API para calcular a contagem (`COUNT`) de eventos por `user_id` em janelas de `tumbling` de 7 segundos baseadas em `event_time`. Selecione o `user_id`, o início da janela, o fim da janela e a contagem.

5.  **Watermarks:** O que um `Watermark(T)` efetivamente sinaliza para o sistema Flink?
    * a) Todos os eventos com *timestamp* exatamente igual a `T` já chegaram.
    * b) O sistema não processará mais nenhum evento no *stream* após o `processing time` atingir `T`.
    * c) O sistema considera que todos os eventos que ocorreram no `event time` `ts <= T` já foram recebidos.
    * d) O `Watermark` é apenas um *timestamp* de referência e não afeta o processamento da janela.

---

Parabéns por completar o Módulo 4! Trabalhar com tempo é um dos aspectos mais poderosos e, por vezes, desafiadores do processamento de *streams*. No próximo módulo, exploraremos como estender a Table API com Funções Definidas pelo Usuário (UDFs).