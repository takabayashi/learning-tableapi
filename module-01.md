## Módulo 1: Introdução ao Apache Flink e à Table API

Bem-vindo ao primeiro módulo do nosso curso sobre a Table API do Apache Flink! Neste módulo, vamos explorar os fundamentos do Apache Flink, entender o que é a Table API, como configurar seu ambiente de desenvolvimento e como o Flink se integra com a plataforma Confluent, especialmente quando se trata de Kafka.

---

### O que é o Apache Flink?

O Apache Flink é um framework de processamento de dados distribuído, de código aberto, projetado para processar grandes volumes de dados, tanto em modo **streaming (fluxo contínuo)** quanto em modo **batch (lote)**. Ele se destaca por sua alta performance, escalabilidade e capacidade de lidar com o estado da aplicação de forma robusta e tolerante a falhas.

**Principais Características do Flink:**

* **Processamento Unificado de Stream e Batch:** O Flink trata o processamento em batch como um caso especial do processamento de stream, onde o stream é finito. Isso simplifica o desenvolvimento e permite a reutilização de código e lógica para ambos os cenários.
* **Alto Desempenho e Baixa Latência:** Projetado para fornecer resultados rápidos, o Flink é capaz de processar milhões de eventos por segundo com latências na casa dos milissegundos.
* **Gerenciamento de Estado Avançado:** O Flink oferece um sistema sofisticado para gerenciar o estado das aplicações de streaming. Ele garante consistência exatamente uma vez (exactly-once semantics) para o estado, mesmo em caso de falhas.
* **Tolerância a Falhas:** Mecanismos robustos de checkpointing e recuperação garantem que as aplicações possam se recuperar de falhas sem perda de dados ou com mínima interrupção.
* **Escalabilidade:** Aplicações Flink podem ser escaladas para rodar em milhares de nós, processando petabytes de dados.
* **Rica Biblioteca de Conectores e APIs:** O Flink oferece APIs de diferentes níveis de abstração (DataStream API, Table API, SQL) e uma vasta gama de conectores para diversas fontes e coletores de dados (Kafka, HDFS, Cassandra, Elasticsearch, etc.).
* **Suporte a Tempo de Evento e Tempo de Processamento:** Permite lidar com dados que chegam fora de ordem e realizar cálculos baseados no tempo em que os eventos realmente ocorreram.

**Casos de Uso Comuns:**

* **Análise em Tempo Real:** Detecção de fraudes, monitoramento de sistemas, personalização em tempo real.
* **Processamento de Eventos Complexos (CEP):** Identificação de padrões em fluxos de eventos.
* **ETL (Extração, Transformação e Carga) de Dados:** Tanto para streaming quanto para batch.
* **Análise de Grafos em Tempo Real.**
* **Recomendações de Produtos em Tempo Real.**
* **Monitoramento de Qualidade de Dados.**

---

### Introdução à Table API do Flink

A Table API é uma API relacional de alto nível para o Apache Flink que permite aos desenvolvedores usar uma linguagem de consulta declarativa, semelhante ao SQL, para processar dados. Ela oferece uma maneira mais abstrata e concisa de escrever lógicas de processamento de dados em comparação com a DataStream API, que é mais imperativa.

**Arquitetura e Posição no Ecossistema Flink:**

A Table API (e a SQL API, que é intimamente ligada a ela) fica acima da DataStream API e da DataSet API (para processamento em batch legado). As consultas escritas na Table API ou SQL são traduzidas por um otimizador de consultas do Flink em programas DataStream ou DataSet eficientes.

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
* **Unificação de APIs:** Permite usar a mesma lógica para processamento de stream e batch.
* **Interoperabilidade:** Tabelas podem ser facilmente convertidas de/para DataStreams ou DataSets, permitindo combinar a Table API com as APIs de nível inferior quando necessário.
* **Linguagem Familiar:** A sintaxe é intuitiva para quem já tem experiência com SQL ou sistemas relacionais.

**Comparação com a DataStream API:**

* **DataStream API:**
    * **Nível de Abstração:** Mais baixo, oferece controle granular sobre as transformações de dados e o gerenciamento de estado.
    * **Modelo de Programação:** Imperativo. Você define cada passo da transformação.
    * **Casos de Uso:** Lógica de processamento complexa que não se encaixa bem no modelo relacional, gerenciamento de estado muito específico, interações de baixo nível com tempo e janelas.
* **Table API/SQL:**
    * **Nível de Abstração:** Mais alto, declarativo. Você especifica o resultado desejado.
    * **Modelo de Programação:** Declarativo.
    * **Casos de Uso:** A maioria das tarefas de análise de dados, transformações relacionais, agregações, junções. Ideal para prototipagem rápida e quando a lógica pode ser expressa de forma relacional.

**Conceitos Chave da Table API:**

* **`TableEnvironment`:** O ponto de entrada principal para a Table API e SQL. É usado para registrar tabelas, executar consultas SQL, e configurar propriedades da execução.
* **`Table`:** A estrutura de dados fundamental da Table API. Representa uma tabela (potencialmente um stream de dados em constante mudança) com um esquema definido (nomes e tipos de colunas).
* **Tabelas Dinâmicas (Dynamic Tables):** Um conceito central no processamento de streams com a Table API. Tabelas dinâmicas mudam ao longo do tempo. Consultas em tabelas dinâmicas produzem outras tabelas dinâmicas.

---

### Configurando um Ambiente de Desenvolvimento Flink

Para começar a desenvolver aplicações Flink com a Table API em Java, você precisará configurar seu ambiente de desenvolvimento.

**Pré-requisitos:**

1.  **Java Development Kit (JDK):** O Flink requer Java 8 ou 11. Certifique-se de que o JDK está instalado e configurado corretamente (variável de ambiente `JAVA_HOME`).
2.  **Ferramenta de Build (Maven ou Gradle):** Usaremos Maven para os exemplos, mas Gradle também é totalmente suportado.
    * **Maven:** Versão 3.0.4 ou superior.
    * **Gradle:** Versão 7.x ou superior.
3.  **IDE (Opcional, mas Recomendado):** IntelliJ IDEA, Eclipse ou VS Code com extensões Java.

**Configuração do Projeto (Maven):**

Crie um novo projeto Maven e adicione as seguintes dependências ao seu arquivo `pom.xml`. Estas são as dependências essenciais para começar com a Table API em um ambiente de streaming.

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
* Use `<scope>provided</scope>` para dependências que já estarão disponíveis no classpath do cluster Flink (como `flink-streaming-java`, `flink-table-api-java-bridge`, `flink-table-planner-loader`, `flink-clients`). Isso reduz o tamanho do seu JAR.
* Use `<scope>compile</scope>` (ou omita o escopo, pois é o padrão) se você estiver construindo um JAR "fat" que inclua todas as dependências, ou para rodar diretamente na sua IDE. Para rodar localmente na IDE de forma simples, você pode mudar o escopo para `compile` e remover o plugin `maven-shade-plugin` ou configurá-lo adequadamente se for gerar um JAR para submissão.

**Exemplo Simples de Código (para verificar a configuração):**

```java
package com.example.flink;

import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.TableEnvironment;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.types.Row;

public class SimpleTableApp {

    public static void main(String[] args) throws Exception {
        // 1. Configurar o ambiente de execução de stream
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 2. Criar o TableEnvironment
        // Para aplicações de streaming, use StreamTableEnvironment
        // A partir do Flink 1.14, EnvironmentSettings é a forma preferida de criar o TableEnvironment
        EnvironmentSettings settings = EnvironmentSettings.newInstance()
            .inStreamingMode() // ou .inBatchMode()
            .build();
        TableEnvironment tableEnv = TableEnvironment.create(settings);

        // Alternativamente, se você precisar da interoperabilidade com DataStream API mais diretamente:
        // StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

        // 3. Criar uma tabela de origem (exemplo: a partir de uma coleção de dados)
        Table sourceTable = tableEnv.fromValues(
            Row.of("Alice", 12),
            Row.of("Bob", 10),
            Row.of("Alice", 100)
        ).as("name", "score");

        // 4. Registrar a tabela para poder referenciá-la em SQL (opcional para API de Tabela)
        tableEnv.createTemporaryView("InputTable", sourceTable);

        // 5. Executar uma consulta simples com a Table API
        Table resultTable = sourceTable
            .filter(org.apache.flink.table.api.Expressions.$("score").isGreater(50))
            .groupBy(org.apache.flink.table.api.Expressions.$("name"))
            .select(
                org.apache.flink.table.api.Expressions.$("name"),
                org.apache.flink.table.api.Expressions.$("score").sum().as("total_score")
            );

        // 6. Imprimir os resultados (para fins de demonstração)
        // Para converter uma Table para um DataStream e imprimir:
        // tableEnv.toDataStream(resultTable).print(); // Requer StreamTableEnvironment se usar esta conversão explícita

        // Usando executeSql para mostrar resultados (mais simples para visualização rápida)
        System.out.println("\nResultados da Table API:");
        resultTable.execute().print();

        // Exemplo com SQL
        System.out.println("\nResultados do SQL:");
        tableEnv.executeSql("SELECT name, SUM(score) FROM InputTable WHERE score > 50 GROUP BY name")
                .print();


        // Nota: Em uma aplicação real, você normalmente não chama env.execute() aqui
        // se estiver usando apenas Table API/SQL e conectores.
        // A chamada a tableEnv.executeSql(...) ou table.execute() já submete o job.
        // Se houver operações DataStream, env.execute("Nome do Job") seria chamado no final.
    }
}
```
*Observação:* Para executar este código diretamente na IDE e ver a saída no console, as dependências com `<scope>provided</scope>` precisam ser alteradas para `<scope>compile</scope>`. Para submeter a um cluster, `provided` é o ideal.

---

### Flink Table API e Confluent

A Confluent Platform é uma plataforma de streaming de dados construída em torno do Apache Kafka. Ela oferece ferramentas e serviços adicionais que podem complementar e aprimorar as capacidades do Apache Flink, especialmente em cenários que envolvem Kafka como fonte ou destino de dados.

**Como a Confluent Aprimora o Flink (especialmente com Kafka):**

1.  **Confluent Schema Registry:**
    * **Gerenciamento Centralizado de Esquemas:** O Schema Registry armazena e gerencia os esquemas dos dados (Avro, Protobuf, JSON Schema) que fluem pelo Kafka.
    * **Evolução de Esquemas:** Permite que os esquemas dos tópicos do Kafka evoluam ao longo do tempo de forma compatível.
    * **Integração com Flink:** O Flink pode se integrar com o Schema Registry para serializar e desserializar dados de/para o Kafka. O conector Kafka do Flink e os formatos (como `flink-avro-confluent-registry`) facilitam essa integração, garantindo que o Flink use os esquemas corretos e lide com a evolução dos esquemas de forma robusta. Isso é crucial para a governança de dados e para evitar erros de desserialização.

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
    * Embora o Flink possua seus próprios conectores Kafka, a Confluent também investe em garantir uma boa interoperabilidade e, em alguns casos, pode fornecer conhecimento especializado ou ferramentas que ajudam na configuração e monitoramento de pipelines Kafka-Flink.

3.  **Confluent Cloud:**
    * Oferece Kafka gerenciado (Confluent Cloud for Apache Kafka) e, potencialmente, um ambiente onde o Flink pode ser implantado ou integrado mais facilmente com serviços Kafka gerenciados. Isso simplifica a infraestrutura de operações.

4.  **ksqlDB (como alternativa ou complemento):**
    * ksqlDB é um banco de dados de streaming de eventos que permite construir aplicações de processamento de stream usando SQL diretamente sobre o Kafka.
    * **Comparação/Complemento:**
        * **Flink Table API/SQL:** Um motor de processamento de propósito geral mais poderoso e flexível, com estado gerenciado sofisticado, janelamento avançado e uma gama mais ampla de conectores além do Kafka. Ideal para transformações complexas, lógica de negócios com estado e integração com diversos sistemas.
        * **ksqlDB:** Focado primariamente em Kafka, excelente para transformações e agregações mais simples diretamente nos fluxos do Kafka, materializando visualizações e servindo queries sobre esses fluxos. Pode ser mais rápido para casos de uso mais simples centrados em Kafka.
    * Em algumas arquiteturas, ksqlDB pode ser usado para pré-processamento ou pós-processamento de dados no Kafka, enquanto o Flink cuida das lógicas mais complexas.

**Integração Específica com Kafka e Schema Registry:**

Ao usar a Table API do Flink com Kafka e o Confluent Schema Registry, você normalmente usaria um formato como `avro-confluent-registry` ou `protobuf-confluent-registry`.

Exemplo de como definir uma tabela conectada ao Kafka com Schema Registry (será detalhado no Módulo 8):

```sql
-- Este é um exemplo de DDL SQL que seria executado via tableEnv.executeSql(...)
CREATE TABLE meu_topico_kafka (
  `user_id` BIGINT,
  `item_id` STRING,
  `timestamp` TIMESTAMP(3) METADATA FROM 'timestamp',
  WATERMARK FOR `timestamp` AS `timestamp` - INTERVAL '5' SECOND
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

1.  **Conceitual:** Descreva com suas próprias palavras a principal diferença entre processamento de stream e processamento em batch. Como o Apache Flink aborda ambos?
2.  **Conceitual:** Quais são os três principais benefícios de usar a Table API do Flink em vez da DataStream API para um novo projeto de análise de dados? Em que cenário a DataStream API ainda seria preferível?
3.  **Configuração:**
    * Crie um novo projeto Maven em sua máquina.
    * Adicione as dependências básicas do Flink (`flink-streaming-java`, `flink-table-api-java-bridge`, `flink-table-planner-loader`, `flink-clients`) ao seu `pom.xml`, utilizando a versão mais recente do Flink que você encontrar na documentação oficial. Configure o escopo das dependências para `compile` para poder rodar o exemplo diretamente na IDE.
    * Copie e cole o código `SimpleTableApp.java` fornecido anteriormente no seu projeto.
    * Execute a classe `SimpleTableApp`. Você conseguiu ver as saídas no console? (Não se preocupe se houver muitos logs do Flink, foque nas seções "Resultados da Table API" e "Resultados do SQL").
4.  **Conceitual:** Como o Confluent Schema Registry pode ajudar a evitar problemas em uma pipeline de dados que usa Flink para processar mensagens do Kafka?

---

No próximo módulo, mergulharemos nos conceitos centrais da Table API, como `TableEnvironment`, criação de tabelas a partir de diversas fontes e as operações básicas que podemos realizar sobre elas.