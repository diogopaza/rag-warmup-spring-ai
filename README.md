# 🧠 RAG Warm-up com Spring AI

---

## 🎯 CONTEXTO E OBJETIVO GERAL

Projeto curto de aquecimento antes da **Parte 16 (Pipeline RAG)** do
`desafio-completo-parte-3-arquitetura-genai-rag-mcp-llmops`. A ideia é entender os conceitos de RAG
na prática, com um projeto pequeno: embedding, chunk, vector store, busca por similaridade e
prompt aumentado.

**Stack:** Spring Boot 3.4 · Java 21 · Spring AI 1.0 · Ollama (LLM local, gratuito) ·
`SimpleVectorStore` (em memória) → **Neon** (Postgres serverless com `pgvector`).

### O que é RAG, em 4 frases

1. O LLM só sabe o que aprendeu no treino e **não conhece os documentos da sua empresa**.
2. **Ingestão:** os documentos são quebrados em pedaços (*chunks*). Cada chunk vira um vetor de
   números (*embedding*) que representa o significado dele, e os vetores ficam num **vector store**.
3. **Busca (Retrieval):** a pergunta também vira um vetor, e você busca os chunks de significado mais parecido.
4. **Geração:** o prompt fica "responda **só** com base neste contexto: [chunks] + pergunta".
   **R**etrieval **A**ugmented **G**eneration.

### Como vamos trabalhar

* Você escreve o código; eu aponto requisitos e bugs e dou **nota de 0 a 10** a cada etapa.
* As perguntas de cada etapa são respondidas **neste README**, abaixo da própria etapa.
* Toda correção é conferida direto no código antes de ser dada como fechada.
* **Os prazos são compromisso.** Cada etapa só é marcada como ✅ quando foi entregue e avaliada
  até a data. Se uma etapa atrasar, o atraso fica registrado na tabela (não se apaga a data original).

---

## 📅 PRAZOS

Início: **14/11/2026** (sábado).

| Etapa | Conteúdo | Prazo | Entregue em | Nota | Status |
|---|---|---|---|---|---|
| 1 | Setup: Ollama + projeto Spring + config | **16/11/2026** (seg) | | | ⏳ |
| 2 | Ingestão (ETL): documentos → chunks → embeddings | **18/11/2026** (qua) | | | ⏳ |
| 3 | Busca + geração (RAG manual) e comparação sem RAG | **21/11/2026** (sáb) | | | ⏳ |
| 4 | Experimentos: chunk size, topK, threshold | **23/11/2026** (seg) | | | ⏳ |
| 5 | `QuestionAnswerAdvisor` (a abstração do Spring AI) | **25/11/2026** (qua) | | | ⏳ |
| 6 | Neon: Postgres + `pgvector` na nuvem | **28/11/2026** (sáb) | | | ⏳ |
| 7 | Neon console: SQL de similaridade, metadata e branching | **01/12/2026** (ter) | | | ⏳ |
| 8 | Fechamento: filtros por metadata, README final | **03/12/2026** (qui) | | | ⏳ |

Legenda: ⏳ pendente · 🔨 em andamento · ✅ entregue no prazo · ⚠️ entregue com atraso · ❌ não entregue

---

## 🧩 ETAPA 1 — SETUP · prazo 16/11/2026

### 🎯 Objetivo

Deixar o ambiente pronto: um LLM local rodando e um projeto Spring que consegue falar com ele.

### 🧪 Desafio

* Instalar o Ollama (ollama.com) e baixar os dois modelos:
  ```bash
  ollama pull llama3.2
  ollama pull nomic-embed-text
  ```
* Criar o projeto Spring Boot neste repositório (start.spring.io: Maven, Java 21, **Spring Web**).
  Os documentos já estão em `src/main/resources/docs/`; não apague essa pasta ao extrair o projeto.
* Adicionar o Spring AI ao `pom.xml`:
  ```xml
  <dependencyManagement>
      <dependencies>
          <dependency>
              <groupId>org.springframework.ai</groupId>
              <artifactId>spring-ai-bom</artifactId>
              <version>1.0.0</version>
              <type>pom</type>
              <scope>import</scope>
          </dependency>
      </dependencies>
  </dependencyManagement>
  ```
  ```xml
  <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-starter-model-ollama</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-vector-store</artifactId>
  </dependency>
  ```
* `application.yml`:
  ```yaml
  spring:
    ai:
      ollama:
        base-url: http://localhost:11434
        chat:
          options:
            model: llama3.2
            temperature: 0.2
        embedding:
          options:
            model: nomic-embed-text
  ```
* Criar um endpoint `GET /sem-rag?q=...` que só repassa a pergunta para o LLM via `ChatClient`
  (prova de que a integração funciona).

### ❓ Perguntas

1. Por que usamos **dois** modelos diferentes (`llama3.2` e `nomic-embed-text`)? Qual é o papel de cada um?
2. O que é `temperature` e por que deixamos baixa (0.2) num assistente que responde sobre políticas da empresa?

### 🎯 Avaliação (0 a 10)

* Projeto sobe sem erro e responde no `/sem-rag`
* Configuração limpa (nada de URL/modelo hardcoded no Java)
* Respostas às perguntas

---

## 🧩 ETAPA 2 — INGESTÃO (ETL) · prazo 18/11/2026

### 🎯 Objetivo

Transformar os documentos em vetores guardados num vector store: **E**xtract (ler) → **T**ransform
(quebrar em chunks) → **L**oad (gerar embedding e guardar).

### 🧪 Desafio

* Criar um bean `VectorStore` usando `SimpleVectorStore.builder(embeddingModel).build()`.
* Criar um `ApplicationRunner` que, ao subir a aplicação:
  * lê todos os `classpath:docs/*.txt` com `TextReader`
  * guarda o nome do arquivo na metadata de cada documento (chave `arquivo`)
  * quebra em chunks com `TokenTextSplitter` (comece com ~200 tokens por chunk)
  * adiciona os chunks no `VectorStore`
  * loga quantos chunks cada arquivo gerou

### ❓ Perguntas

3. Em que momento exato o embedding é gerado: no `TextReader`, no `TokenTextSplitter` ou no `vectorStore.add`? Como você confirmou?
4. Quantas dimensões tem o vetor gerado pelo `nomic-embed-text`? Por que esse número importa quando formos para um banco de verdade?
5. Por que quebrar em chunks em vez de gerar um embedding por documento inteiro?

### 🎯 Avaliação (0 a 10)

* Ingestão funcionando e logada
* Metadata `arquivo` presente nos chunks
* Respostas às perguntas

---

## 🧩 ETAPA 3 — BUSCA + GERAÇÃO (RAG MANUAL) · prazo 21/11/2026

### 🎯 Objetivo

Montar o RAG **na mão**, sem abstração, para ver cada peça.

### 🧪 Desafio

* Criar `GET /com-rag?q=...` que:
  1. busca os 3 chunks mais parecidos com `vectorStore.similaritySearch(SearchRequest.builder().query(q).topK(3).build())`
  2. junta o texto deles (`document.getText()`) numa `String` de contexto
  3. monta um **system prompt** dizendo: responda SOMENTE com base no contexto; se a resposta não
     estiver no contexto, responda "não sei"
  4. chama o `ChatClient` com o system prompt e a pergunta
* Devolver junto da resposta **de quais arquivos** vieram os chunks usados (citação de fonte).
* Testar as perguntas abaixo nos **dois** endpoints (`/sem-rag` e `/com-rag`) e colar os resultados aqui:
  * `Quantos dias de antecedência preciso para pedir férias?`
  * `Qual o limite de hospedagem por noite?`
  * `Quais dias a engenharia vai ao escritório?`
  * `Qual o salário de um dev pleno?` (a resposta não está nos documentos)

### ❓ Perguntas

6. O que o `/sem-rag` respondeu sobre a antecedência de férias? Por que ele inventou (ou não)? Qual o nome desse fenômeno?
7. A pergunta do salário: o `/com-rag` disse "não sei"? Se não disse, o que no seu prompt falhou?
8. Por que devolver a fonte (nome do arquivo) é importante num sistema RAG real?

### 🎯 Avaliação (0 a 10)

* RAG manual funcionando
* Prompt que evita alucinação
* Citação de fonte
* Comparação documentada

---

## 🧩 ETAPA 4 — EXPERIMENTOS · prazo 23/11/2026

### 🎯 Objetivo

Entender os parâmetros que mais mudam a qualidade de um RAG, e o que acontece quando estão errados.

### 🧪 Desafio

* Rodar as 4 perguntas da Etapa 3 com: chunk de **30**, **200** e **2000** tokens. Registrar o resultado numa tabela aqui.
* Rodar com `topK` = **1**, **3** e **10**.
* Adicionar `similarityThreshold` na busca e achar um valor que faça a pergunta do salário não trazer nenhum chunk.
* Logar o **score** de similaridade de cada chunk retornado.

### ❓ Perguntas

9. O que piora com chunk muito pequeno? E com chunk muito grande?
10. `topK` = 1 quebrou alguma resposta? `topK` = 10 piorou alguma? Por quê?
11. Pare e suba a aplicação de novo. O que acontece com os vetores? Por que isso é um problema em produção?

### 🎯 Avaliação (0 a 10)

* Tabela de experimentos preenchida
* Threshold justificado com números (scores reais)
* Respostas às perguntas

---

## 🧩 ETAPA 5 — `QuestionAnswerAdvisor` · prazo 25/11/2026

### 🎯 Objetivo

Trocar o RAG manual pela abstração do Spring AI e saber dizer o que ela esconde.

### 🧪 Desafio

* Adicionar a dependência `spring-ai-advisors-vector-store`.
* Criar `GET /com-advisor?q=...` usando `QuestionAnswerAdvisor` no `ChatClient`.
* Manter o `/com-rag` manual funcionando, para comparar.
* Ativar o `SimpleLoggerAdvisor` e colar aqui o prompt final que o advisor montou.

### ❓ Perguntas

12. Compare o prompt que o advisor monta com o seu prompt manual. O que é igual e o que é diferente?
13. O que você deixou de escrever ao usar o advisor? Em que situação você voltaria para o RAG manual?

### 🎯 Avaliação (0 a 10)

* Advisor funcionando
* Prompt do advisor capturado e comparado
* Respostas às perguntas

---

## 🧩 ETAPA 6 — NEON: VECTOR STORE PERSISTENTE NA NUVEM · prazo 28/11/2026

### 🎯 Objetivo

Resolver o problema da pergunta 11: tirar os vetores da memória e guardar num Postgres com a
extensão **`pgvector`**, hospedado no **Neon** (Postgres serverless, com plano gratuito e console web).

### 🧪 Desafio

* Criar uma conta gratuita no neon.tech e um projeto (região mais próxima, ex.: `sa-east-1` São Paulo).
* No **SQL Editor** do console Neon, habilitar a extensão:
  ```sql
  CREATE EXTENSION IF NOT EXISTS vector;
  ```
* Trocar o `SimpleVectorStore` pelo PGVector:
  * remover o bean `SimpleVectorStore`
  * adicionar `spring-ai-starter-vector-store-pgvector` (ele traz o driver Postgres e o JDBC)
  * configurar:
    ```yaml
    spring:
      datasource:
        url: ${NEON_URL}          # jdbc:postgresql://<host-do-neon>/neondb?sslmode=require
        username: ${NEON_USER}
        password: ${NEON_PASSWORD}
      ai:
        vectorstore:
          pgvector:
            initialize-schema: true
            dimensions: 768          # confira com a sua resposta da pergunta 4
            distance-type: COSINE_DISTANCE
            index-type: HNSW
    ```
* **As credenciais NUNCA vão para o Git.** Use variáveis de ambiente (12-Factor, fator III: Config).
* Ajustar a ingestão para **não duplicar** os documentos a cada vez que a aplicação sobe.

### ❓ Perguntas

14. Por que o `sslmode=require` na URL do Neon? O que aconteceria sem ele?
15. O que acontece se `dimensions` não bater com o modelo de embedding?
16. Como você evitou a duplicação na ingestão? Que alternativas existiam?
17. Neon é "serverless": o banco "dorme" quando ninguém usa. Que efeito isso tem na primeira requisição depois de um tempo parado?

### 🎯 Avaliação (0 a 10)

* Vetores persistindo no Neon (reiniciar a aplicação não perde nada)
* Credenciais fora do repositório
* Ingestão idempotente
* Respostas às perguntas

---

## 🧩 ETAPA 7 — NEON CONSOLE: OLHANDO O VECTOR STORE POR DENTRO · prazo 01/12/2026

### 🎯 Objetivo

Usar o console web do Neon para ver que um vector store é "só" uma tabela, e fazer na mão, em SQL,
a busca que o Spring AI faz por você.

### 🧪 Desafio (tudo no SQL Editor do Neon, colando as queries e resultados aqui)

* Ver a estrutura da tabela criada pelo Spring AI (`vector_store`): colunas, tipos, índice HNSW.
* Contar os chunks por arquivo usando a coluna de metadata (`jsonb`):
  `metadata->>'arquivo'`.
* Ver um embedding cru e conferir a quantidade de dimensões com `vector_dims(embedding)`.
* Fazer uma **busca por similaridade em SQL puro**: pegue o embedding de um chunk sobre férias e
  ordene os outros chunks pela distância de cosseno (operador `<=>`). Os mais próximos são do mesmo
  arquivo?
* Rodar `EXPLAIN ANALYZE` na busca e ver se o índice HNSW foi usado.
* **Branching do Neon:** criar uma branch `experimento-chunk-500`, apontar a aplicação para ela,
  reingerir com chunk de 500 tokens e comparar as respostas com a branch `main`, sem tocar nos dados da main.

### ❓ Perguntas

18. O que significam os operadores `<=>`, `<->` e `<#>` do pgvector? Qual deles combina com `COSINE_DISTANCE`?
19. O que é o índice HNSW e por que ele é "aproximado"? Qual a troca entre velocidade e precisão?
20. Para que servem as branches do Neon num time de verdade? Compare com subir um Postgres no Docker para cada experimento.

### 🎯 Avaliação (0 a 10)

* Queries corretas, com resultados colados
* Busca de similaridade em SQL funcionando
* Branch criada e comparação documentada
* Respostas às perguntas

---

## 🧩 ETAPA 8 — FECHAMENTO · prazo 03/12/2026

### 🎯 Objetivo

Fechar o projeto com um recurso que RAG de verdade usa muito, e deixar o README pronto para portfólio.

### 🧪 Desafio

* Adicionar um parâmetro opcional `arquivo` no `/com-rag` que filtra a busca pela metadata
  (`filterExpression`, ex.: `arquivo == 'ferias.txt'`).
* Adicionar um quarto documento seu (inventado, sobre qualquer política da Nebulosa) e provar que
  ele entra no RAG sem mudar código.
* Atualizar este README: como rodar, arquitetura em um diagrama simples, e a tabela de prazos completa.

### ❓ Perguntas

21. Num sistema multi-empresa (multi-tenant), como o filtro por metadata impede que a empresa A veja documentos da empresa B? Por que filtrar só no prompt não basta?
22. Olhando para trás: em quais etapas você cumpriu o prazo e em quais não? O que causou cada atraso?

### 🎯 Avaliação (0 a 10)

* Filtro por metadata funcionando
* Novo documento ingerido sem mudar código
* README final
* Honestidade na pergunta 22

---

## 📚 Documentos do RAG

Ficam em `src/main/resources/docs/`. A empresa **Nebulosa Tecnologia** é **fictícia de propósito**:
se o LLM acertar as respostas, é porque o RAG funcionou e não porque ele "já sabia".

* `ferias.txt` — política de férias
* `reembolso.txt` — política de reembolso de despesas
* `home-office.txt` — política de trabalho híbrido
