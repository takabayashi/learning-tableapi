
# 🚀 Curso Abrangente de Apache Flink: Dominando a Table API e SQL

Bem-vindo ao repositório oficial do nosso curso completo sobre Apache Flink, com foco especial em sua poderosa Table API e SQL! Este material foi cuidadosamente elaborado para guiar você, desenvolvedor ou engenheiro de dados, desde os conceitos fundamentais até tópicos avançados no universo do processamento de dados em larga escala, tanto em *streaming* quanto em *batch*.

🇧🇷 Este curso é totalmente em **Português do Brasil**.

[![Apache Flink](https://img.shields.io/badge/Apache%20Flink-1.19+-E6526F?logo=apacheflink&logoColor=white)](https://flink.apache.org/)
[![License](https://img.shields.io/badge/License-PENDING-blue)](#licença) [![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md) ---

## 🎯 Sobre Este Curso

No mundo atual, a capacidade de processar e analisar vastos volumes de dados em tempo real é crucial. O Apache Flink se destaca como uma plataforma líder para esses desafios, e este curso tem como objetivo principal capacitar você a utilizar a **Table API e o SQL do Flink** de forma eficaz e produtiva.

Se você é um desenvolvedor buscando construir aplicações de processamento de dados robustas, um engenheiro de dados querendo dominar ETL de *streams* e análises em tempo real, ou um analista que deseja aplicar SQL a *streams* de dados, este curso é para você!

**Ao final deste curso, você será capaz de:**
* Entender profundamente o Apache Flink, sua arquitetura e seus casos de uso.
* Desenvolver aplicações Flink usando a Table API e SQL para processamento de *streams* e *batch*.
* Implementar transformações de dados complexas, junções, agregações e trabalhar com janelas temporais.
* Estender o Flink com Funções Definidas pelo Usuário (UDFs).
* Integrar o Flink com sistemas externos como Kafka, bancos de dados e sistemas de arquivos.
* Aplicar melhores práticas de otimização, debugging e entender o funcionamento do Flink em ambientes gerenciados como o Confluent Cloud.

---

## 📚 Estrutura do Curso (Visão Geral dos Módulos)

Este curso é dividido em módulos progressivos, cada um focado em um aspecto específico do Flink e sua Table API, culminando com um módulo bônus que oferece uma visão ainda mais ampla.

* **[Introdução do Curso](./chapters/module-00.md)**: Apresenta os objetivos, o público-alvo e o roteiro completo do que você aprenderá. (Este arquivo pode conter o texto completo da introdução que elaboramos).

* **[Módulo 1: Introdução ao Flink e à Table API](./chapters/module-01.md)**: Uma introdução ao Apache Flink, suas capacidades de *streaming* e *batch*, e a configuração do seu ambiente de desenvolvimento, com um olhar sobre a integração com o ecossistema Confluent.

* **[Módulo 2: Conceitos Fundamentais da Table API](./chapters/module-02.md)**: Foca nos blocos de construção essenciais: o `TableEnvironment`, a criação de `Tables` a partir de diversas fontes (incluindo `DataStreams`), e as operações básicas de manipulação.

* **[Módulo 3: Operações Relacionais](./chapters/module-03.md)**: Expande seu conhecimento para operações relacionais complexas, como diferentes tipos de `joins`, agrupamentos, agregações, operações de conjunto e ordenação de resultados.

* **[Módulo 4: Trabalhando com Tempo na Table API](./chapters/module-04.md)**: Um mergulho crucial no tratamento do tempo no Flink, cobrindo `event time`, `processing time`, `watermarks` e os diversos tipos de janelas (`tumbling`, `sliding`, `session`, `cumulative`).

* **[Módulo 5: Funções Definidas pelo Usuário (UDFs)](./chapters/module-05.md)**: Ensina como estender as funcionalidades nativas do Flink, implementando `scalar`, `table` e `aggregate functions` customizadas.

* **[Módulo 6: Integração da Table API com SQL](./chapters/module-06.md)**: Explora a sinergia entre a Table API programática e o poder declarativo do SQL, cobrindo o `Catalog`, execução de queries SQL, criação de `Views` e o uso em ambientes gerenciados.

* **[Módulo 7: Conceitos Avançados da Table API](./chapters/module-07.md)**: Aborda tópicos sofisticados como `Dynamic Tables`, `Continuous Queries`, `Temporal Tables` e `Temporal Joins` para enriquecimento de dados versionados, e `Upsert Streams`.

* **[Módulo 8: Conectando a Sistemas Externos](./chapters/module-08.md)**: Foca na integração do Flink com o mundo exterior através de `Connectors` e `Formats`, detalhando fontes e sinks comuns como FileSystem, Kafka e JDBC, incluindo o modelo no Confluent Cloud.

* **[Módulo 9: Arquitetura, Melhores Práticas, Otimização e Contribuição](./chapters/module-09.md)**: Um módulo final abrangente que explora a arquitetura interna da Table API, como entender planos de execução, otimizar performance, depurar aplicações, comparar Flink OSS com seu uso na Confluent Cloud, e como contribuir para o projeto Flink.

* **[Módulo Bônus: Apache Flink - Uma Visão Abrangente](./chapters/module-bonus.md)**: Oferece um mergulho profundo na história do Flink, sua arquitetura fundamental, a poderosa parceria com Apache Kafka, casos de uso ideais que destacam seu poder, mais detalhes sobre Flink gerenciado pela Confluent e um FAQ.

---

## 🔧 Pré-requisitos

Para o melhor aproveitamento deste curso, recomendamos:
* Familiaridade básica com **Java** (os exemplos de código são nesta linguagem).
* Entendimento de conceitos **SQL**.
* Alguma experiência prévia com processamento de dados (*batch* ou *stream*) é útil, mas não estritamente obrigatória para os módulos iniciais.
* Um ambiente onde você possa instalar Java, Maven e seu IDE preferido para executar os exemplos.

---

## 📖 Como Usar Este Material

* **Siga a Ordem:** Os módulos foram desenhados para construir conhecimento progressivamente.
* **Pratique, Pratique, Pratique:** A melhor forma de aprender é colocando a mão na massa! Execute os exemplos, modifique-os e tente os exercícios.
* **Consulte a Documentação Oficial:** Este curso é um guia, mas a [documentação oficial do Apache Flink](https://flink.apache.org/docs/) é a fonte definitiva.
* **Seja Curioso:** Explore, questione e divirta-se aprendendo!

---

## ✍️ Sobre os Autores

Este material de treinamento foi cuidadosamente elaborado através de uma colaboração especial:

* **Daniel Takabayashi:** Staff Solutions Engineer na Confluent, Daniel traz sua vasta experiência prática ajudando clientes a construir e otimizar soluções de processamento de *streams* de dados em larga escala com Apache Flink e Apache Kafka. Sua expertise no ecossistema Confluent e seu profundo conhecimento das capacidades do Flink foram fundamentais para moldar o conteúdo prático e relevante deste curso.

* **Gemini:** Seu amigo e assistente de IA do Google. Gemini colaborou na estruturação do conteúdo, na geração de explicações claras e exemplos, e na refinação do material para garantir que seja didático, completo e envolvente. Esta parceria buscou combinar o conhecimento técnico especializado com a capacidade de apresentar informações complexas de forma acessível.

Esperamos que esta colaboração resulte em uma experiência de aprendizado rica e proveitosa para você!

---

## 🙌 Como Contribuir para Este Curso

Sua contribuição é muito bem-vinda! Se você encontrar erros, tiver sugestões de melhoria, novos exemplos, ou quiser ajudar de alguma forma, por favor:
1.  Abra uma **Issue** para discutir a mudança ou reportar o erro.
2.  Se desejar, faça um **Fork** do repositório, crie um *branch* com suas modificações e submeta um **Pull Request**.

Consulte nosso arquivo `CONTRIBUTING.md` (se você criar um) para mais detalhes sobre como contribuir.

---

## 📜 Licença

*(Adicione aqui a licença sob a qual você está disponibilizando este material. Ex: MIT, Apache 2.0, etc.)*

Exemplo:
Este projeto está licenciado sob a Licença MIT - veja o arquivo `LICENSE.md` (a ser criado) para detalhes.

---

Agradecemos seu interesse e esperamos que este curso seja um divisor de águas na sua jornada com Apache Flink!
```

Agora a seção "Estrutura do Curso" tem links que devem funcionar se você organizar seus arquivos Markdown conforme o padrão `chapters/module-XX.md`. O link para a introdução também foi adicionado como `chapters/module-00.md`.

Acho que com isso seu `README.md` está pronto para receber os leitores no GitHub!