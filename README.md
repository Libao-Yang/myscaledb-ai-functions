# 配置 text-embeddings-inference

## Docker 启动 text-embeddings-inference

选择合适的 embedding model， 并启动 Docker Embedding 服务。具体请参考: https://github.com/huggingface/text-embeddings-inference

本次使用的是 Alibaba-NLP/gte-base-en-v1.5, 也可以按照 Air gapped deployment 来部署
https://huggingface.co/docs/text-embeddings-inference/quick_tour#air-gapped-deployment

# 启动 MyScale

## Docker Compose 启动 MyScaleDB
```bash
docker compose up -d
```

## 进入 MyScaleDB 验证 text-embeddings-inference 连接情况
```bash
docker exec -ti myscaledb bash

# IP 和 PORT 替换为 text-embeddings-inference 对应 Docker 的 IP 和 PORT, 返回向量这表示 embedding 服务连接正常
# 根据 model 不同，请求体格式也存在一定差异, 本地使用的 model：Alibaba-NLP/gte-large-en-v1.5
curl 10.1.2.118:8080/embed \
    -X POST \
    -d '{"inputs":"What is Deep Learning?"}' \
    -H 'Content-Type: application/json'

# 验证 MyScale 的 EmbedText 函数
clickhouse-client
## 进入 clickhouse-client 后执行 query, 替换 IP 和 PORT, 和对应的 query text
SELECT EmbedText(concat('inputs: ', 'What is Deep Learning?'), 'HuggingFace', 'http://10.1.2.118:8080/embed', 'hf_key', '')
```

## 执行 Vector, Text and Hybrid Search

具体参考：https://myscale.com/docs/en/hybrid-search/#perform-vector-text-and-hybrid-search

# MyScaleDB RAG with Ollama

> 本 repo 为支持 Ollama 新增/扩展了几个 UDF：
```sql
SELECT EmbedText('test', 'Ollama', 'http://10.1.2.118:11434/api/embed', 'hf_key', '{"model": "all-minilm"}');

SELECT chat_ollama('http://10.2.2.144:11434', 'qwen2.5:1.5b', 'Hello');

SELECT split_text(2000, 'longlong text ...');
```

## 1. 创建 doc_table

```sql
CREATE TABLE doc_table (
    id UInt64,
    content String
) ENGINE = MergeTree() ORDER BY id;

INSERT INTO doc_table (id, content) VALUES
    (1, 'Artificial Intelligence (AI) is transforming industries around the world. It allows for automation of tasks, improved decision-making, and new capabilities. Companies in healthcare, finance, and even art are exploring the benefits of AI.'),
    (2, 'Climate change is an urgent issue facing humanity. Reducing carbon emissions, transitioning to renewable energy, and protecting forests are key ways to mitigate its effects. Governments, corporations, and individuals all have roles to play.'),
    (3, 'Space exploration has been a topic of fascination for decades. From the Apollo missions to Mars rovers, humanity continues to push the boundaries of what we can achieve. The recent efforts to establish a permanent presence on the Moon and eventually Mars show our ambition to become an interplanetary species.'),
    (4, 'The history of computer science dates back to the invention of mechanical calculating devices. Over the years, advancements like the development of programming languages, algorithms, and computer hardware have shaped the modern digital world.'),
    (5, 'Quantum computing is an emerging technology that promises to revolutionize computation. Unlike classical computers, quantum computers use qubits and can perform certain types of calculations much faster. This has applications in fields like cryptography, chemistry, and optimization problems.');
```

## 2. 创建 embedding_table

```sql
CREATE TABLE embedding_table (
    doc_id UInt64,
    chunk_index UInt32,
    chunk String,
    vector Array(Float32),
    CONSTRAINT check_length CHECK length(vector) = 384
) ENGINE = MergeTree() ORDER BY doc_id;
```

## 3. chunk & embedding

```sql
INSERT INTO embedding_table SELECT
    id,
    chunk.1,
    chunk.2,
    EmbedText(chunk.2, 'Ollama', 'http://10.1.2.118:11434/api/embed', 'no_need', '{"model": "all-minilm"}')
FROM
(
    SELECT
        id,
        arrayJoin(split_text(60, content)) AS chunk
    FROM doc_table
);
```

## 4. build vector index for embedding_table

```sql
ALTER TABLE embedding_table
    ADD VECTOR INDEX vec_ind vector TYPE DEFAULT;
```

## 5. semantic search

```sql
WITH 'Please introduce quantum computing and artificial intelligence.' AS question
SELECT
    doc_id,
    chunk_index,
    chunk,
    distance(vector, EmbedText(question, 'Ollama', 'http://10.1.2.118:11434/api/embed', 'no_need', '{"model": "all-minilm"}')) AS distance
FROM embedding_table
ORDER BY distance ASC
LIMIT 5
```

## 6. RAG

```sql
WITH 'Please introduce quantum computing and artificial intelligence.' AS question
SELECT chat_ollama('http://10.2.2.144:11434', 'qwen2.5:1.5b', chat_content) AS Ollama_Answer
FROM
(
    SELECT concat(arrayStringConcat(groupArray(chunk)), '\n Based on the context above \n Q: ', question, '\n A: The Answer is ') AS chat_content
    FROM
    (
        SELECT
            doc_id,
            chunk_index,
            chunk,
            distance(vector, EmbedText(question, 'Ollama', 'http://10.1.2.118:11434/api/embed', 'no_need', '{"model": "all-minilm"}')) AS distance
        FROM embedding_table
        ORDER BY distance ASC
        LIMIT 5
    )
)

┌─Ollama_Answer──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Q: Please introduce quantum computing and artificial intelligence.

A: Quantum computing and artificial intelligence (AI) are two rapidly advancing technologies that hold significant promise in driving innovation across various industries. Here’s a brief overview of each:

### Quantum Computing

Quantum computing leverages the principles of quantum mechanics to perform calculations much faster than classical computers can. Traditional computers use bits, which are 0s or 1s, but quantum computers use qubits that represent both states (0 and 1) simultaneously thanks to superposition. This property allows them to solve certain problems exponentially faster.

Key features of quantum computing include:
- **Superposition**: Qubits exist in multiple states at once.
- **Entanglement**: The state of one qubit can depend on the state of another, even when they are separated by large distances.
- **Quantum Entanglement**: A pair or group of qubits is entangled such that observing one affects the state of the others regardless of distance.

### Artificial Intelligence

Artificial intelligence (AI) refers to machines and systems that can perform tasks requiring human-like cognition. AI involves various techniques including machine learning, natural language processing, computer vision, and deep learning.

Key features of AI include:
- **Machine Learning**: Algorithms that allow machines to learn from data without explicit programming.
- **Natural Language Processing (NLP)**: Technologies that enable computers to understand, interpret, and respond to human language.
- **Computer Vision**: Systems that can interpret images or video frames for recognition purposes.
- **Deep Learning**: A subset of machine learning focused on artificial neural networks with multiple layers.

### Interplay Between Quantum Computing and AI

Quantum computing has the potential to significantly enhance various aspects of AI, including:
1. **Enhanced Processing Power**: By leveraging quantum bits (qubits), these machines can process data much faster than traditional computers.
2. **Improved Machine Learning Algorithms**: Quantum algorithms may optimize machine learning models more efficiently than current methods.

### Potential Applications

- **Drug Discovery**: Speeding up the discovery of new drugs by simulating complex molecular interactions.
- **Financial Modeling**: Forecasting financial markets with unprecedented accuracy and speed.
- **Cryptography**: Enhancing cybersecurity defenses through quantum-resistant encryption techniques.
- **Climate Science**: Providing more accurate simulations for climate modeling.

However, there are also challenges in integrating these technologies, such as maintaining the integrity of data due to the unpredictable nature of quantum states. These advancements bring us closer to a future where machines can potentially surpass human capabilities, paving new paths for scientific research and technological innovation. │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```