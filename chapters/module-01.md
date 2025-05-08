## Módulo 1: Introdução ao Apache Flink e à Table API

Bem-vindo ao primeiro módulo do nosso curso sobre a Table API do Apache Flink! Neste módulo, vamos explorar os fundamentos do Apache Flink, entender o que é a Table API, como configurar seu ambiente de desenvolvimento e como o Flink se integra com a plataforma Confluent, especialmente quando se trata de Kafka.

---

### O que é o Apache Flink?

O Apache Flink é um *framework* de processamento de dados distribuído, de código aberto, projetado para processar grandes volumes de dados, tanto em modo **streaming (fluxo contínuo)** quanto em modo **batch (lote)**. Ele se destaca por sua alta performance, escalabilidade e capacidade de lidar com o estado da aplicação de forma robusta e tolerante a falhas.

**Principais Características do Flink:**

* **Processamento Unificado de Stream e Batch:** O Flink trata o processamento em *batch* como um caso especial do processamento de *stream*, onde o *stream* é finito. Isso simplifica o desenvolvimento e permite a reutilização de código e lógica para ambos os cenários.
* **Alto Desempenho e Baixa Latência:** Projetado para fornecer resultados rápidos, o Flink é capaz de processar milhões de eventos por segundo com latências na casa dos milissegundos.
* **Gerenciamento de Estado Avançado:** O Flink oferece um sistema sofisticado para gerenciar o estado das aplicações de *streaming*. Ele garante consistência exatamente uma vez (*exactly-once semantics*) para o estado, mesmo em caso de falhas.
* **Tolerância a Falhas:** Mecanismos robustos de *checkpointing* e recuperação garantem que as aplicações possam se recuperar de falhas sem perda de dados ou com mínima interrupção.
* **Escalabilidade:** Aplicações Flink podem ser escaladas para rodar em milhares de nós, processando petabytes de dados.
* **Rica Biblioteca de Conectores e APIs:** O Flink oferece APIs de diferentes níveis de abstração (DataStream API, Table API, SQL) e uma vasta gama de conectores para diversas fontes e coletores de dados (Kafka, HDFS, Cassandra, Elasticsearch, etc.).
* **Suporte a Tempo de Evento e Tempo de Processamento:** Permite lidar com dados que chegam fora de ordem e realizar cálculos baseados no tempo em que os eventos realmente ocorreram.

**Casos de Uso Comuns:**

* Análise em Tempo Real: Detecção de fraudes, monitoramento de sistemas, personalização em tempo real.
* Processamento de Eventos Complexos (CEP): Identificação de padrões em fluxos de eventos.
* ETL (Extração, Transformação e Carga) de Dados: Tanto para *streaming* quanto para *batch*.
* Análise de Grafos em Tempo Real.
* Recomendações de Produtos em Tempo Real.
* Monitoramento de Qualidade de Dados.

---

### Introdução à Table API do Flink

A Table API é uma API relacional de alto nível para o Apache Flink que permite aos desenvolvedores usar uma linguagem de consulta declarativa, semelhante ao SQL, para processar dados. Ela oferece uma maneira mais abstrata e concisa de escrever lógicas de processamento de dados em comparação com a DataStream API, que é mais imperativa.

**Arquitetura e Posição no Ecossistema Flink:**

A Table API (e a SQL API, que é intimamente ligada a ela) fica acima da DataStream API e da DataSet API (para processamento em *batch* legado). As consultas escritas na Table API ou SQL são traduzidas por um otimizador de consultas do Flink em programas DataStream ou DataSet eficientes.

```
+-----------------------------------+
| Aplicações Flink                  |
+-----------------------------------+
| Table API        | SQL            |  <-- Abstração Relacional
+------------------+----------------+
| DataStream API   | DataSet API    |  <-- APIs de Baixo Nível
+-----------------------------------+
| Runtime do Flink (Core Engine)    |
+-----------------------------------+
```

**Benefícios da Table API:**

* **Abstração de Alto Nível:** Permite focar no "o quê" fazer com os dados, em vez do "como" fazer, simplificando o desenvolvimento.
* **Otimização Automática:** O Flink possui um otimizador de consultas que analisa as operações da Table API/SQL e gera planos de execução eficientes.
* **Unificação de APIs:** Permite usar a mesma lógica para processamento de *stream* e *batch*.
* **Interoperabilidade:** *Tables* podem ser facilmente convertidas de/para DataStreams ou DataSets, permitindo combinar a Table API com as APIs de nível inferior quando necessário.
* **Linguagem Familiar:** A sintaxe é intuitiva para quem já tem experiência com SQL ou sistemas relacionais.

**Comparação com a DataStream API:**

* **DataStream API:**
    * **Nível de Abstração:** Mais baixo, oferece controle granular sobre as transformações de dados e o gerenciamento de estado.
    * **Modelo de Programação:** Imperativo. Você define cada passo da transformação.
    * **Casos de Uso:** Lógica de processamento complexa que não se encaixa bem no modelo relacional, gerenciamento de estado muito específico, interações de baixo nível com tempo e *windows* (janelas).
* **Table API/SQL:**
    * **Nível de Abstração:** Mais alto, declarativo. Você especifica o resultado desejado.
    * **Modelo de Programação:** Declarativo.
    * **Casos de Uso:** A maioria das tarefas de análise de dados, transformações relacionais, agregações, junções. Ideal para prototipagem rápida e quando a lógica pode ser expressa de forma relacional.

**Conceitos Chave da Table API:**

* **`TableEnvironment`:** O ponto de entrada principal para a Table API e SQL. É usado para registrar *tables*, executar consultas SQL, e configurar propriedades da execução.
* **`Table`:** A estrutura de dados fundamental da Table API. Representa uma tabela (potencialmente um *stream* de dados em constante mudança) com um esquema definido (nomes e tipos de colunas).
* ***Dynamic Tables***: Um conceito central no processamento de *streams* com a Table API. *Dynamic tables* mudam ao longo do tempo. Consultas em *dynamic tables* produzem outras *dynamic tables*.

---

### Configurando um Ambiente de Desenvolvimento Flink

Para começar a desenvolver aplicações Flink com a Table API em Java, você precisará configurar seu ambiente de desenvolvimento.

**Pré-requisitos:**

1.  **Java Development Kit (JDK):** O Flink requer Java 8 ou 11 (Java 17 também é suportado nas versões mais recentes do Flink, verifique a documentação da sua versão do Flink). Certifique-se de que o JDK está instalado e configurado corretamente (variável de ambiente `JAVA_HOME`).
2.  **Ferramenta de Build (Maven ou Gradle):** Usaremos Maven para os exemplos, mas Gradle também é totalmente suportado.
    * **Maven:** Versão 3.0.4 ou superior.
    * **Gradle:** Versão 7.x ou superior.
3.  **IDE (Opcional, mas Recomendado):** IntelliJ IDEA, Eclipse ou VS Code com extensões Java.

**Configuração do Projeto (Maven):**

Crie um novo projeto Maven e adicione as seguintes dependências ao seu arquivo `pom.xml`. Estas são as dependências essenciais para começar com a Table API em um ambiente de *streaming*.

```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <flink.version>1.19.0</flink.version> <java.version>11</java.version> <maven.compiler.source>${java.version}</maven.compiler.source>
    <maven.compiler.target>${java.version}</maven.compiler.target>
    <log4j.version>2.17.1</log4j.version> </properties>

<dependencies>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-streaming-java</artifactId>
        <version>${flink.version}</version>
        <scope>provided</scope> </dependency>

    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-table-api-java-bridge</artifactId>
        <version>${flink.version}</version>
        <scope>provided</scope>
    </dependency>

    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-table-planner-loader</artifactId>
        <version>${flink.version}</version>
        <scope>provided</scope>
    </dependency>

    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-clients</artifactId>
        <version>${flink.version}</version>
        <scope>provided</scope>
    </dependency>

    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-slf4j-impl</artifactId>
        <version>${log4j.version}</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-api</artifactId>
        <version>${log4j.version}</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <version>${log4j.version}</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.1</version>
            <configuration>
                <source>${java.version}</source>
                <target>${java.version}</target>
            </configuration>
        </plugin>
        </plugins>
</build>
```

**Nota sobre Escopo das Dependências:**
* Use `<scope>provided</scope>` para dependências que já estarão disponíveis no *classpath* do cluster Flink. Isso reduz o tamanho do seu *JAR*.
* Use `<scope>compile</scope>` (ou omita o escopo, pois é o padrão) se você estiver construindo um *JAR "fat"* que inclua todas as dependências, ou para rodar diretamente na sua IDE. Para rodar localmente na IDE de forma simples, você pode mudar o escopo para `compile`.

**Exemplo Simples de Código (para verificar a configuração):**

```java
package com.example.flink;

import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.TableEnvironment;
// Import para Expressions se for usar a forma estática.
import static org.apache.flink.table.api.Expressions.$;
import org.apache.flink.types.Row;

public class SimpleTableApp {

    public static void main(String[] args) throws Exception {
        // 1. Configurar o ambiente de execução de stream
        // StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // Para aplicações puras de Table API, obter o StreamExecutionEnvironment pode não ser sempre necessário
        // inicialmente, pois o TableEnvironment pode encapsular isso.

        // 2. Criar o TableEnvironment
        EnvironmentSettings settings = EnvironmentSettings.newInstance()
            .inStreamingMode() // ou .inBatchMode()
            .build();
        TableEnvironment tableEnv = TableEnvironment.create(settings);

        // Se você precisar de interoperabilidade com DataStream API, você pode criar o StreamTableEnvironment
        // a partir de um StreamExecutionEnvironment:
        // StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env, settings);


        // 3. Criar uma tabela de origem (exemplo: a partir de uma coleção de dados)
        Table sourceTable = tableEnv.fromValues(
            Row.of("Alice", 12),
            Row.of("Bob", 10),
            Row.of("Alice", 100)
        ).as("name", "score"); // Define os nomes das colunas

        // 4. Registrar a tabela como uma temporary view para poder referenciá-la em SQL
        tableEnv.createTemporaryView("InputTable", sourceTable);

        // 5. Executar uma consulta simples com a Table API
        Table resultTableApi = sourceTable
            .filter($("score").isGreater(50))
            .groupBy($("name"))
            .select(
                $("name"),
                $("score").sum().as("total_score")
            );

        // 6. Imprimir os resultados (para fins de demonstração)
        System.out.println("\nResultados da Table API:");
        resultTableApi.execute().print();

        // Exemplo com SQL
        System.out.println("\nResultados do SQL:");
        Table resultTableSql = tableEnv.sqlQuery("SELECT name, SUM(score) as total_score FROM InputTable WHERE score > 50 GROUP BY name");
        resultTableSql.execute().print();

        // Nota: Em uma aplicação real, a execução (como .execute().print() ou .executeInsert())
        // é o que dispara o job Flink.
        // Se você misturar DataStream API, um env.execute("Nome do Job") seria chamado no final.
        // Para Table API pura, cada .execute() em uma Table ou .executeSql() em uma DML/DQL
        // que produz resultados ou insere em um sink pode disparar a execução.
    }
}
```
*Observação:* Para executar este código diretamente na IDE e ver a saída no console, as dependências com `<scope>provided</scope>` precisam ser alteradas para `<scope>compile</scope>`. Para submeter a um cluster, `provided` é o ideal. Para o import `static org.apache.flink.table.api.Expressions.$;`, ele simplifica a escrita de expressões.

---

### Flink Table API e Confluent

A Confluent Platform é uma plataforma de *streaming* de dados construída em torno do Apache Kafka. Ela oferece ferramentas e serviços adicionais que podem complementar e aprimorar as capacidades do Apache Flink, especialmente em cenários que envolvem Kafka como fonte ou destino de dados.

**Como a Confluent Aprimora o Flink (especialmente com Kafka):**

1.  **Confluent Schema Registry:**
    * **Gerenciamento Centralizado de Esquemas:** O *Schema Registry* armazena e gerencia os esquemas dos dados (Avro, Protobuf, JSON Schema) que fluem pelo Kafka.
    * **Evolução de Esquemas:** Permite que os esquemas dos tópicos do Kafka evoluam ao longo do tempo de forma compatível.
    * **Integração com Flink:** O Flink pode se integrar com o *Schema Registry* para serializar e desserializar dados de/para o Kafka. O *Flink connector* para Kafka e os formatos (como `flink-avro-confluent-registry`) facilitam essa integração, garantindo que o Flink use os esquemas corretos e lide com a evolução dos esquemas de forma robusta. Isso é crucial para a governança de dados e para evitar erros de desserialização.

    ```
    +-----------------+     Registra/Recupera Esquema     +-----------------------+
    | Produtor Flink  | <-------------------------------> | Confluent Schema      |
    | (ou outro)      |                                   | Registry              |
    +-----------------+                                   +-----------------------+
          |                                                           ^
          | Dados Serializados (com ID do Esquema)                    |
          v                                                           |
    +-----------------+                                               |
    | Apache Kafka    |                                               |
    +-----------------+                                               |
          |                                                           |
          | Dados Serializados                                        |
          v                                                           |
    +-----------------+     Recupera Esquema (via ID)   +-----------------------+
    | Consumidor Flink| <-----------------------------> | Confluent Schema      |
    | (Table API)     |                                 | Registry              |
    +-----------------+                                 +-----------------------+
    ```

2.  **Conectores Otimizados e Confiáveis:**
    * Embora o Flink possua seus próprios *connectors* Kafka, a Confluent também investe em garantir uma boa interoperabilidade e, em alguns casos, pode fornecer conhecimento especializado ou ferramentas que ajudam na configuração e monitoramento de *pipelines* Kafka-Flink.

3.  **Confluent Cloud:**
    * Oferece Kafka gerenciado (Confluent Cloud for Apache Kafka) e, potencialmente, um ambiente onde o Flink pode ser implantado ou integrado mais facilmente com serviços Kafka gerenciados. Isso simplifica a infraestrutura de operações.

4.  **ksqlDB (como alternativa ou complemento):**
    * ksqlDB é um banco de dados de *streaming* de eventos que permite construir aplicações de processamento de *stream* usando SQL diretamente sobre o Kafka.
    * **Comparação/Complemento:**
        * **Flink Table API/SQL:** Um motor de processamento de propósito geral mais poderoso e flexível, com estado gerenciado sofisticado, *windowing* avançado e uma gama mais ampla de *connectors* além do Kafka. Ideal para transformações complexas, lógica de negócios com estado e integração com diversos sistemas.
        * **ksqlDB:** Focado primariamente em Kafka, excelente para transformações e agregações mais simples diretamente nos fluxos do Kafka, materializando *views* e servindo *queries* sobre esses fluxos. Pode ser mais rápido para casos de uso mais simples centrados em Kafka.
    * Em algumas arquiteturas, ksqlDB pode ser usado для pré-processamento ou pós-processamento de dados no Kafka, enquanto o Flink cuida das lógicas mais complexas.

**Integração Específica com Kafka e Schema Registry:**

Ao usar a Table API do Flink com Kafka e o Confluent *Schema Registry*, você normalmente usaria um formato como `avro-confluent-registry` ou `protobuf-confluent-registry`.

Exemplo de como definir uma tabela conectada ao Kafka com *Schema Registry* (será detalhado no Módulo 8):

```sql
-- Este é um exemplo de DDL SQL que seria executado via tableEnv.executeSql(...)
CREATE TABLE meu_topico_kafka (
  `user_id` BIGINT,
  `item_id` STRING,
  `event_timestamp` TIMESTAMP(3) METADATA FROM 'timestamp', -- 'timestamp' é um campo mágico do conector Kafka
  WATERMARK FOR `event_timestamp` AS `event_timestamp` - INTERVAL '5' SECOND
) WITH (
  'connector' = 'kafka',
  'topic' = 'nome_do_topico',
  'properties.bootstrap.servers' = 'kafka-broker:9092',
  'properties.group.id' = 'meu_grupo_flink',
  'scan.startup.mode' = 'earliest-offset',
  'format' = 'avro-confluent-registry', -- Especifica o formato
  'avro-confluent-registry.url' = 'http://schema-registry:8081', -- URL do Schema Registry
  'avro-confluent-registry.subject' = 'nome_do_topico-value' -- Opcional: padrão é <topic_name>-value
);
```

Essa integração garante que os dados lidos do Kafka sejam desserializados corretamente de acordo com o esquema registrado, e os dados escritos no Kafka sejam serializados com um esquema compatível e registrado.

---

### Exercícios

1.  **Conceitual:** Qual das seguintes afirmações descreve melhor como o Apache Flink lida com processamento de *stream* e *batch*?
    * a) Flink usa motores separados para *stream* e *batch*, otimizados individualmente.
    * b) Flink trata o processamento em *batch* como um caso especial de processamento de *stream*, usando um *runtime* unificado.
    * c) Flink é primariamente um motor de *batch*, com capacidades de *stream* adicionadas posteriormente.
    * d) Flink foca exclusivamente em processamento de *stream* de baixa latência.

2.  **Conceitual:** Ao comparar a Table API com a DataStream API do Flink, qual das seguintes é uma vantagem principal da Table API?
    * a) Oferece controle mais granular sobre o estado e o tempo.
    * b) Permite escrever lógica de processamento de forma declarativa, com otimizações automáticas de consulta.
    * c) É mais adequada para interações de baixo nível com fontes e *sinks* de dados.
    * d) Requer conhecimento aprofundado de Java para todas as operações.

3.  **Configuração:**
    * Crie um novo projeto Maven em sua máquina.
    * Adicione as dependências básicas do Flink (`flink-streaming-java`, `flink-table-api-java-bridge`, `flink-table-planner-loader`, `flink-clients`) ao seu `pom.xml`, utilizando uma versão recente e estável do Flink (ex: 1.19.0). Configure o escopo das dependências para `compile` para poder rodar o exemplo diretamente na IDE.
    * Copie e cole o código `SimpleTableApp.java` fornecido anteriormente no seu projeto.
    * Execute a classe `SimpleTableApp`. Você conseguiu ver as saídas no console? (Não se preocupe se houver muitos *logs* do Flink, foque nas seções "Resultados da Table API" e "Resultados do SQL").

4.  **Conceitual:** Em uma *pipeline* de dados que usa Flink para processar mensagens do Kafka, qual o principal benefício de integrar o Confluent *Schema Registry*?
    * a) Aumentar a taxa de transferência (throughput) do Kafka.
    * b) Garantir a governança e a evolução compatível dos esquemas de dados, prevenindo erros de serialização/desserialização.
    * c) Substituir a necessidade de usar a Table API do Flink.
    * d) Reduzir a latência no processamento de mensagens individuais pelo Flink.

---

No próximo módulo, mergulharemos nos conceitos centrais da Table API, como `TableEnvironment`, criação de *tables* a partir de diversas fontes e as operações básicas que podemos realizar sobre elas.