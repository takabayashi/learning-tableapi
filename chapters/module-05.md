## Módulo 5: Funções Definidas pelo Usuário (UDFs)

Bem-vindo ao Módulo 5! Embora a Table API e o SQL do Flink ofereçam um rico conjunto de funções embutidas, muitas vezes você precisará implementar lógicas de negócios específicas ou transformações de dados que não são cobertas nativamente. As Funções Definidas pelo Usuário (UDFs) permitem estender o Flink com seu próprio código customizado, que pode ser chamado diretamente em suas queries.

Neste módulo, exploraremos:
* Os diferentes tipos de UDFs: `ScalarFunction`, `TableFunction`, e `AggregateFunction`.
* Como implementar e registrar essas funções.
* Como chamar UDFs em suas queries da Table API e SQL.

---

### Tipos de UDFs

O Flink suporta diversos tipos de UDFs, cada uma adequada para um tipo diferente de operação:

**1. `ScalarFunction` (Função Escalar)**

* **Definição:** Uma `ScalarFunction` mapeia zero, um ou mais valores de entrada escalares para um único valor de saída escalar. Ela opera em cada linha individualmente, sem considerar outras linhas.
* **Casos de Uso Típicos:**
    * Formatação de strings (e.g., `UPPERCASE_WITH_PREFIX(string, prefix)`).
    * Cálculos matemáticos customizados (e.g., `LOG_BASE_N(value, base)`).
    * Lógica condicional complexa que seria verbosa em expressões SQL puras.
    * Funções de parsing ou validação de dados.
* **Implementação:**
    * Herde da classe `org.apache.flink.table.functions.ScalarFunction`.
    * Implemente um ou mais métodos públicos chamados `eval`. O Flink fará a correspondência dos tipos e número de argumentos da chamada da função com os métodos `eval` disponíveis.
* **Exemplo de Código Java:**

    ```java
    import org.apache.flink.table.functions.ScalarFunction;

    // UDF para calcular o comprimento de uma string, retornando 0 para null.
    public class StringLengthFunction extends ScalarFunction {
        // O nome do método DEVE ser 'eval'
        public int eval(String s) {
            return (s == null) ? 0 : s.length();
        }

        // Você pode sobrecarregar o método 'eval' com diferentes tipos de argumentos
        public int eval(Integer i) {
            return (i == null) ? 0 : String.valueOf(i).length();
        }
    }

    // UDF para adicionar um prefixo
    public static class PrefixNameFunction extends ScalarFunction {
        public String eval(String name, String prefix) {
            if (name == null || prefix == null) {
                return name;
            }
            return prefix + name;
        }
    }
    ```

**2. `TableFunction<T>` (Função de Tabela)**

* **Definição:** Uma `TableFunction` mapeia zero, um ou mais valores de entrada escalares para múltiplos registros (linhas) como saída. Cada linha de saída pode ter uma ou mais colunas. É similar a um `flatMap` na programação funcional ou ao operador `UNNEST` ou `LATERAL TABLE` em SQL.
* **Casos de Uso Típicos:**
    * "Explodir" (desagregar) um valor de array ou uma string delimitada em múltiplas linhas. (e.g., `split('hello,world', ',')` -> duas linhas: 'hello' e 'world').
    * Gerar múltiplos pares chave-valor a partir de uma entrada complexa.
* **Implementação:**
    * Herde da classe `org.apache.flink.table.functions.TableFunction<T>`, onde `T` é o tipo da linha de saída (pode ser `Row`, um POJO, ou um tipo primitivo se a função emitir apenas uma coluna).
    * Implemente um ou mais métodos públicos chamados `eval`.
    * Dentro do método `eval`, use o método `collect(T outputRow)` (fornecido pela classe base ou injetado como um `Collector<T>`) para emitir as linhas de resultado.
* **Exemplo de Código Java:**

    ```java
    import org.apache.flink.table.functions.TableFunction;
    import org.apache.flink.types.Row; // Para emitir linhas com múltiplas colunas

    // UDF para dividir uma string por um delimitador e emitir cada parte como uma nova linha com uma coluna.
    // Emite um Row com duas colunas: a palavra e seu comprimento.
    public class SplitStringFunction extends TableFunction<Row> {
        // O nome do método DEVE ser 'eval'
        public void eval(String text, String separator) {
            if (text == null || separator == null) {
                return;
            }
            for (String token : text.split(java.util.regex.Pattern.quote(separator))) {
                // Emite uma nova linha com a palavra e seu comprimento
                collect(Row.of(token, token.length()));
            }
        }
        // Opcional: para informar ao Flink os tipos de retorno, especialmente se não for Row
        // @Override
        // public org.apache.flink.table.types.DataType getResultType(Object[] arguments, Class[] argTypes) {
        //     return DataTypes.ROW(DataTypes.FIELD("word", DataTypes.STRING()), DataTypes.FIELD("length", DataTypes.INT()));
        // }
    }
    ```
    *Nota: Para `TableFunction`, o Flink geralmente pode inferir os tipos de resultado, mas fornecer `getResultType()` ou usar `DataTypeHint` pode ser necessário em casos mais complexos ou para melhor desempenho.*

**3. `AggregateFunction<T, ACC>` (Função de Agregação)**

* **Definição:** Uma `AggregateFunction` mapeia valores de múltiplas linhas de entrada (de um grupo) para um único valor de saída agregado.
* **Casos de Uso Típicos:**
    * Calcular uma média ponderada.
    * Concatenar strings de um grupo.
    * Implementar funções estatísticas customizadas (e.g., variância, desvio padrão se não disponíveis).
* **Implementação:**
    * Herde da classe `org.apache.flink.table.functions.AggregateFunction<T, ACC>`, onde `T` é o tipo do valor agregado final e `ACC` é o tipo do acumulador intermediário.
    * Implemente os seguintes métodos:
        * `ACC createAccumulator()`: Cria e inicializa o acumulador. Chamado uma vez por grupo.
        * `void accumulate(ACC accumulator, [Input Values...])`: Processa cada linha de entrada do grupo e atualiza o acumulador. O Flink fará o matching dos argumentos com os valores da query.
        * `T getValue(ACC accumulator)`: Retorna o resultado final da agregação a partir do acumulador. Chamado uma vez por grupo após todas as linhas terem sido processadas.
    * Métodos opcionais (importantes para certos cenários, como *retraction streams* ou otimizações de *merge*):
        * `void retract(ACC accumulator, [Input Values...])`: Desfaz o efeito de uma linha no acumulador (usado em *retraction streams*).
        * `void merge(ACC accumulator, Iterable<ACC> its)`: Mescla múltiplos acumuladores (e.g., em agregações de sessão ou otimizações de *two-phase aggregation*).
        * `void resetAccumulator(ACC accumulator)`: Reseta um acumulador para reutilização (raramente necessário).
* **Exemplo de Código Java:**

    ```java
    import org.apache.flink.table.functions.AggregateFunction;
    import java.util.Iterator;

    // Acumulador para a média ponderada
    public static class WeightedAvgAccumulator {
        public double sum = 0;  // soma dos valores * pesos
        public long count = 0; // soma dos pesos
    }

    // UDF para calcular a média ponderada de um valor, dado um peso
    public class WeightedAvgFunction extends AggregateFunction<Double, WeightedAvgAccumulator> {

        @Override
        public WeightedAvgAccumulator createAccumulator() {
            return new WeightedAvgAccumulator();
        }

        // Método 'accumulate' para processar cada linha
        // O Flink tentará corresponder os tipos e a ordem dos argumentos
        public void accumulate(WeightedAvgAccumulator acc, Double value, Integer weight) {
            if (value != null && weight != null) {
                acc.sum += value * weight;
                acc.count += weight;
            }
        }

        // Método 'getValue' para retornar o resultado final
        @Override
        public Double getValue(WeightedAvgAccumulator acc) {
            if (acc.count == 0) {
                return null; // Ou 0.0, dependendo da lógica desejada para grupos vazios/pesos zero
            } else {
                return acc.sum / acc.count;
            }
        }

        // Opcional: para dar dicas sobre os tipos do acumulador ao Flink
        // public org.apache.flink.table.types.DataType getAccumulatorType() {
        //    return DataTypes.STRUCTURED(...);
        // }

        // Opcional: para dar dicas sobre o tipo do resultado ao Flink
        // public org.apache.flink.table.types.DataType getResultType() {
        //    return DataTypes.DOUBLE();
        // }

        // Opcional: método retract para streams com retração
        public void retract(WeightedAvgAccumulator acc, Double value, Integer weight) {
            if (value != null && weight != null) {
                acc.sum -= value * weight;
                acc.count -= weight;
            }
        }

        // Opcional: método merge para otimizações (e.g., session windows)
        public void merge(WeightedAvgAccumulator acc, Iterable<WeightedAvgAccumulator> it) {
            Iterator<WeightedAvgAccumulator> iter = it.iterator();
            while (iter.hasNext()) {
                WeightedAvgAccumulator otherAcc = iter.next();
                acc.sum += otherAcc.sum;
                acc.count += otherAcc.count;
            }
        }
    }
    ```

**4. `TableAggregateFunction<T, ACC>` (Função de Agregação de Tabela)**

* **Definição:** Uma `TableAggregateFunction` é similar a uma `AggregateFunction`, mas em vez de retornar um único valor escalar, ela retorna uma tabela (múltiplas linhas e/ou colunas) como resultado da agregação.
* **Casos de Uso Típicos:**
    * Implementar uma função "Top N" que retorna as N principais linhas dentro de cada grupo.
    * Funções que emitem múltiplos resultados estatísticos para um grupo.
* **Implementação:**
    * Herde de `org.apache.flink.table.functions.TableAggregateFunction<T, ACC>`.
    * Métodos `createAccumulator()`, `accumulate()` (e opcionais `retract()`, `merge()`) são similares à `AggregateFunction`.
    * O método chave é `void emitValue(ACC accumulator, Collector<T> out)` ou `void emitUpdateWithRetract(ACC accumulator, RetractionCollector<T> out)` (para *retraction*), que é chamado para emitir os resultados tabulares.
* **Exemplo Conceitual (Top 2 por grupo):**
    ```java
    // Acumulador para Top2
    // public static class Top2Accumulator {
    //     public Integer first;
    //     public Integer second;
    // }
    // public class Top2Function extends TableAggregateFunction<Row, Top2Accumulator> {
    //     // ... implementar createAccumulator, accumulate ...
    //     public void emitValue(Top2Accumulator acc, Collector<Row> out) {
    //         if (acc.first != null) { out.collect(Row.of(acc.first)); }
    //         if (acc.second != null) { out.collect(Row.of(acc.second)); }
    //     }
    // }
    ```
    Devido à sua complexidade, as `TableAggregateFunctions` são geralmente usadas para cenários mais avançados.

---

### Implementando UDFs: Requisitos e Dicas

* **Visibilidade e Construtor:** A classe da UDF deve ser `public` e não `abstract`. Ela deve ter um construtor `public` sem argumentos, ou o Flink deve ser capaz de instanciá-la (e.g., se for uma classe interna estática).
* **Métodos de Avaliação:** Os métodos de processamento (`eval`, `accumulate`, `getValue`, `emitValue`, etc.) devem ser `public` e não estáticos.
* **Tipos de Dados:**
    * Use tipos primitivos Java (int, double, String etc.), tipos Java `Boxed` (Integer, Double), ou tipos de dados do Flink (`org.apache.flink.types.Row`, POJOs, `java.sql.Timestamp`, `java.time.LocalDate`, `java.time.LocalDateTime`, `java.math.BigDecimal`).
    * O Flink tentará mapear os tipos SQL das colunas de entrada para os tipos dos parâmetros dos seus métodos `eval`/`accumulate`.
    * Você pode usar anotações como `@DataTypeHint` nos parâmetros ou no tipo de retorno dos métodos para guiar explicitamente a inferência de tipos do Flink, o que é recomendado para robustez.
* **Determinismo:**
    * Por padrão, o Flink assume que as UDFs são determinísticas, ou seja, para a mesma entrada, elas sempre produzirão a mesma saída. Isso permite otimizações.
    * Se sua UDF não for determinística (e.g., usa `Math.random()`, `System.currentTimeMillis()`, ou depende de estado externo mutável), você deve sobrescrever o método `isDeterministic()` na sua UDF e retornar `false`.
    ```java
    // public class MyNonDeterministicFunc extends ScalarFunction {
    //     public String eval(Integer i) { return i + "_" + System.nanoTime(); }
    //     @Override
    //     public boolean isDeterministic() { return false; }
    // }
    ```

---

### Registrando UDFs

Antes que uma UDF possa ser usada em uma query da Table API ou SQL, ela precisa ser registrada no `TableEnvironment` com um nome único.

```java
import org.apache.flink.table.api.TableEnvironment;
// ... (inicialização do tableEnv) ...

// Registrar uma ScalarFunction
StringLengthFunction lenFunc = new StringLengthFunction();
tableEnv.createTemporarySystemFunction("myLength", lenFunc);
// ou tableEnv.createTemporaryFunction("myLength", lenFunc);

// Registrar uma TableFunction
SplitStringFunction splitFunc = new SplitStringFunction();
tableEnv.createTemporarySystemFunction("mySplit", splitFunc);

// Registrar uma AggregateFunction
WeightedAvgFunction weightedAvgFunc = new WeightedAvgFunction();
tableEnv.createTemporarySystemFunction("myWeightedAvg", weightedAvgFunc);

// Tipos de registro:
// - createTemporarySystemFunction: Registra no catálogo embutido, escopo de sessão, visível em todos os catálogos/BDs.
// - createTemporaryFunction: Registra no catálogo/BD atual, escopo de sessão.
// - createFunction: Registra permanentemente no catálogo (se o catálogo suportar).
```
O nome fornecido durante o registro ("myLength", "mySplit", "myWeightedAvg") é o nome que você usará para chamar a função em SQL ou na Table API.

---

### Chamando UDFs

**1. Na Table API:**

Use a função `call()` da API de expressões (`org.apache.flink.table.api.Expressions.call`) para invocar UDFs registradas.

* **`ScalarFunction`:**
    ```java
    // import static org.apache.flink.table.api.Expressions.$;
    // import static org.apache.flink.table.api.Expressions.call;
    // import static org.apache.flink.table.api.Expressions.lit;

    // Table inputTable = ... ; // Suponha uma tabela com colunas 'name' e 'city'
    // Table resultScalar = inputTable.select(
    //     $("name"),
    //     call("myLength", $("name")).as("name_length"), // Chamando a UDF 'myLength'
    //     call(PrefixNameFunction.class, $("city"), lit("CITY_")).as("prefixed_city") // Chamando por classe (requer que esteja no classpath)
    // );
    // resultScalar.execute().print();
    ```
    *Nota: Chamar por classe (`call(MinhaUDF.class, ...)` é uma opção se a UDF não estiver registrada com um nome, mas estiver no *classpath*. Registrar e chamar pelo nome é geralmente mais explícito e flexível.*

* **`TableFunction`:**
    `TableFunction`s são usadas com uma sintaxe de `join` especial: `joinLateral` ou `leftOuterJoinLateral`. O `LATERAL` indica que a função de tabela é chamada para cada linha da tabela de entrada.
    ```java
    // import static org.apache.flink.table.api.Expressions.$;
    // import static org.apache.flink.table.api.Expressions.call;

    // Table dataWithCsv = tableEnv.fromValues(
    //     Row.of(1, "apple,banana,orange"),
    //     Row.of(2, "lemon,grape")
    // ).as("id", "fruits_csv");

    // // Usando joinLateral: se mySplit não emitir nada para uma linha de entrada, a linha de entrada é descartada.
    // Table resultTableFunc = dataWithCsv
    //     .joinLateral(call("mySplit", $("fruits_csv"), lit(",")) // Chamando mySplit e o separador
    //                    .as("fruit", "fruit_len")) // Nomeando as colunas de saída da UDTF
    //     .select($("id"), $("fruit"), $("fruit_len"));
    // resultTableFunc.execute().print();

    // // Usando leftOuterJoinLateral: se mySplit não emitir nada, a linha de entrada é preservada com NULLs para as colunas da UDTF.
    // Table resultTableFuncOuter = dataWithCsv
    //     .leftOuterJoinLateral(call("mySplit", $("fruits_csv"), lit(","))
    //                             .as("fruit", "fruit_len"))
    //     .select($("id"), $("fruit"), $("fruit_len"));
    // resultTableFuncOuter.execute().print();
    ```

* **`AggregateFunction`:**
    Usada em uma cláusula `select` após um `groupBy`.
    ```java
    // import static org.apache.flink.table.api.Expressions.$;
    // import static org.apache.flink.table.api.Expressions.call;

    // Table sales = tableEnv.fromValues(
    //     Row.of("ProductA", 100.0, 10), // product, value, weight
    //     Row.of("ProductA", 120.0, 5),
    //     Row.of("ProductB", 200.0, 20)
    // ).as("product", "value", "weight");

    // Table resultAgg = sales
    //     .groupBy($("product"))
    //     .select(
    //         $("product"),
    //         call("myWeightedAvg", $("value"), $("weight")).as("w_avg") // Chamando a UDAF
    //     );
    // resultAgg.execute().print();
    ```

**2. Em SQL:**

Uma vez registradas, as UDFs podem ser chamadas em queries SQL usando o nome com o qual foram registradas.

* **`ScalarFunction`:**
    ```sql
    -- Supondo que a UDF 'myLength' e 'PrefixNameFunction' (registrada como 'addPrefix') estejam disponíveis
    -- SELECT
    --   name,
    //   myLength(name) AS name_length,
    //   addPrefix(city, 'CITY_') AS prefixed_city
    // FROM MyInputTable;
    ```

* **`TableFunction`:**
    Use `LATERAL TABLE(<NomeDaFuncaoTable>(...)) AS T(col1, col2, ...)`
    ```sql
    -- Supondo que a UDTF 'mySplit' esteja registrada
    -- SELECT
    //   T1.id,
    //   T2.fruit,
    //   T2.fruit_len
    // FROM MyDataWithCsv AS T1
    // LEFT JOIN LATERAL TABLE(mySplit(T1.fruits_csv, ',')) AS T2(fruit, fruit_len) ON TRUE;
    ```
    O `ON TRUE` é frequentemente usado com `LATERAL TABLE` quando não há condição de junção adicional entre a tabela externa e a saída da UDTF.

* **`AggregateFunction`:**
    ```sql
    -- Supondo que a UDAF 'myWeightedAvg' esteja registrada
    -- SELECT
    //   product,
    //   myWeightedAvg(value, weight) AS w_avg
    // FROM MySalesTable
    // GROUP BY product;
    ```

---

### Considerações Adicionais

* **Dependências:** Se suas UDFs usam bibliotecas de terceiros, esses JARs devem estar disponíveis no *classpath* dos TaskManagers do Flink durante a execução.
* **Testes:**
    * Teste a lógica da sua UDF como uma classe Java comum usando testes unitários.
    * O Flink também fornece utilitários de teste (e.g., em `org.apache.flink.table.planner.utils.JavaUserDefinedAggFunctions`) para testes mais integrados.
* **Performance:**
    * UDFs são uma forma poderosa de estender o Flink, mas código Java arbitrário pode ser uma caixa preta para o otimizador de queries.
    * Mantenha a lógica dentro das UDFs o mais eficiente possível.
    * Evite operações de I/O bloqueantes ou estado complexo não gerenciado pelo Flink dentro de UDFs, especialmente em `ScalarFunction`s que são chamadas por linha.
    * Para `AggregateFunction`, certifique-se de que os acumuladores sejam serializáveis e eficientes.

---

### Exercícios

1.  **`ScalarFunction` - Validação de Email:**
    Implemente uma `ScalarFunction` chamada `IsValidEmailFunction` que recebe uma `String` e retorna `true` se a string parece ser um email válido (e.g., contém "@" e "."), e `false` caso contrário. (Não precisa ser uma validação de email perfeita, apenas a lógica básica). Registre-a e use-a para filtrar uma tabela de usuários.

2.  **`TableFunction` - Gerador de Sequência:**
    Implemente uma `TableFunction<Row>` chamada `SequenceGeneratorFunction` que recebe um inteiro `n` e emite `n` linhas, cada uma contendo um único inteiro de 1 a `n`. Registre-a e use-a com `joinLateral` em uma tabela de entrada para "multiplicar" as linhas.

3.  **`AggregateFunction` - Concatenação de Strings em Grupo:**
    Implemente uma `AggregateFunction<String, StringBuilder>` chamada `StringConcatAggFunction` que concatena todos os valores de uma coluna de string dentro de um grupo, separados por uma vírgula.
    * `createAccumulator()`: retorna `new StringBuilder()`.
    * `accumulate(StringBuilder acc, String value)`: anexa o valor ao `StringBuilder`.
    * `getValue(StringBuilder acc)`: retorna `acc.toString()`.
    Registre-a e use-a para agregar nomes de produtos por categoria.

4.  **Registro e Chamada:**
    Qual a principal diferença na forma de chamar uma `ScalarFunction` e uma `TableFunction` na Table API (não SQL)?
    * a) Ambas são chamadas diretamente como métodos em expressões.
    * b) `ScalarFunction` é chamada diretamente em expressões, `TableFunction` requer `joinLateral` ou `leftOuterJoinLateral`.
    * c) `TableFunction` é chamada diretamente em expressões, `ScalarFunction` requer `joinLateral`.
    * d) Ambas requerem `joinLateral`.

5.  **Determinismo:** Por que é importante que uma UDF seja determinística, ou que o Flink saiba se ela não é?
    * a) Funções não determinísticas não podem ser registradas.
    * b) O determinismo afeta apenas a sintaxe de chamada da UDF.
    * c) O Flink pode aplicar otimizações (como subexpressão comum) se souber que uma função é determinística, e pode evitar resultados inconsistentes em reprocessamentos.
    * d) Funções não determinísticas só podem ser usadas em modo *batch*.

---

Com as UDFs, você tem o poder de adaptar o Flink precisamente às suas necessidades de processamento de dados! No próximo módulo, veremos como a Table API se integra com o SQL e como usar o catálogo do Flink.