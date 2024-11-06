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

