# 前言
在逛huggingface的时候，我看到了这个，有点意思，让我们来玩一玩。

<img width="736" height="374" alt="Image" src="https://github.com/user-attachments/assets/97e88cd7-32f5-42ef-b56b-d4c3fafa3c99" />

一些基础的知识，本文就不再补充了，请优先阅读前面两篇文章，这对于理解本篇文章更有帮助。

[在PostgreSQL中运行ONNX模型 —— pg_onnx](https://mp.weixin.qq.com/s/7Y4EmAUjwocC7r2dY1z3Vg)

[OnnxOCR_PG_ONNX：探讨在PostgreSQL中运行ONNX模型更多可能的验证性项目](https://mp.weixin.qq.com/s/gF1fiF15evmMwq5WCSTmJA)
 
# 环境准备
## onnxruntime-extensions 依赖
需要安装onnxruntime-extensions，我先前安装了onnxruntime 位于/usr/local/onnxruntime所以这里我也直接指向这个目录好了
```bash
git clone --recursive https://github.com/microsoft/onnxruntime-extensions.git
cd onnxruntime-extensions

mkdir build
cd build

cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr/local/onnxruntime \
  -DONNXRUNTIME_DIR=/usr/local/onnxruntime \
  -DOCOS_ENABLE_SPM_TOKENIZER=ON

nohup make &

sudo make install
```
## 下载text_to_embedding.onnx模型
点击[此处](https://huggingface.co/oga5/multilingual-e5-small-pg-onnx/blob/main/text_to_embedding.onnx)下载我们这次所需要的onnx模型，它是被转换过的，下面给出原模型和论文地址。 
原模型网址：https://huggingface.co/intfloat/multilingual-e5-small
论文：https://arxiv.org/pdf/2402.05672
#### 常见问题解答

<img width="779" height="757" alt="Image" src="https://github.com/user-attachments/assets/0db2123a-9e06-4d61-8866-e63bf6b1bd26" />

# 数据库内准备
## 拓展创建
这次我们需要用到两个拓展，一个是pg_onnx，一个是pgvector
```sql
create extension pg_onnx;
create extension vector;
```
#### 导入模型和创建所需函数
```sql
-- Embedding模型注册
SELECT pg_onnx_import_model(
    'e5-embedding',
    'v1',
    pg_read_binary_file('/home/postgres/model/pgonnx/text_to_embedding.onnx')::bytea,
    '{"ortextensions_path": "libortextensions.so"}'::jsonb,
    'e5 text to embedding'
);

-- 基本的Embedding生成函数
CREATE OR REPLACE FUNCTION e5_embedding(input_text text)
RETURNS vector(384)
AS $$
    SELECT array(
        SELECT jsonb_array_elements_text(
            pg_onnx_execute_session(
                'e5-embedding',
                'v1',
                jsonb_build_object('text', jsonb_build_array(input_text))
            )->'embedding'->0
        )::float
    )::vector(384);
$$
LANGUAGE sql
IMMUTABLE;

-- 用于生成passage的Embedding函数
-- E5型号使用“passage：”前缀
CREATE OR REPLACE FUNCTION e5_embedding_passage(input_text text)
RETURNS vector(384)
AS $$
    SELECT e5_embedding('passage: ' || input_text);
$$
LANGUAGE sql
IMMUTABLE;

-- 生成query的Embedding函数
-- E5模型使用“query：”前缀
CREATE OR REPLACE FUNCTION e5_embedding_query(input_text text)
RETURNS vector(384)
AS $$
    SELECT e5_embedding('query: ' || input_text);
$$
LANGUAGE sql
IMMUTABLE;
```
### 表和基础数据准备
```sql
-- 测试表创建
CREATE TABLE llm_test (
    i integer NOT NULL PRIMARY KEY,
    txt text,
    v vector(384)
);

-- HNSW索引创建
CREATE INDEX llm_test_v_idx ON llm_test USING hnsw (v vector_ip_ops);

-- 插入测试数据
INSERT INTO llm_test (i, txt) VALUES 
('1', 'Machine learning is a subfield of artificial intelligence'),
('2', 'A database is a system for managing data'),
('3', 'PostgreSQL is a powerful open-source database'),
('4', 'Vector search retrieves results by computing similarity'),
('5', 'ONNX is a standard format for machine learning models'),
('6', 'Natural language processing is a technology for handling text'),
('7', 'Embeddings convert text into vectors'),
('8', 'Cosine similarity measures similarity between vectors'),
('9', 'A tokenizer splits text into tokens'),
('10', 'Transformers are a modern neural network architecture'),
('11', 'SQL is a language for manipulating databases'),
('12', 'Indexes improve query performance'),
('13', 'pgvector is a vector extension for PostgreSQL'),
('14', 'Semantic search retrieves based on meaning'),
('15', 'Neural networks mimic the structure of the brain'),
('16', 'Deep learning uses multi-layer neural networks'),
('17', 'Batch processing handles multiple data at once'),
('18', 'Model inference performs prediction with a trained model'),
('19', 'Fine-tuning adapts an existing model to a specific task'),
('20', 'A cross-encoder evaluates the relevance between two texts');

-- 生成对应的Embedding结果
-- 也可以考虑创建触发器来完成这一部分工作
UPDATE llm_test SET v = e5_embedding_passage(txt);
```

# 简单测试

```sql
postgres=# WITH q AS (
    SELECT 'What is machine learning?' AS query
),
qv AS MATERIALIZED (
    SELECT e5_embedding_query(q.query) AS v FROM q
)
SELECT 
    i, 
    txt, 
    t.v <#> qv.v AS distance
FROM llm_test t, qv
ORDER BY distance
LIMIT 5;
 i  |                            txt                            |      distance       
----+-----------------------------------------------------------+---------------------
  1 | Machine learning is a subfield of artificial intelligence | -0.8541078567504883
  5 | ONNX is a standard format for machine learning models     | -0.8386106491088867
 16 | Deep learning uses multi-layer neural networks            | -0.8247002363204956
 10 | Transformers are a modern neural network architecture     | -0.8128323554992676
 18 | Model inference performs prediction with a trained model  |  -0.801337718963623
(5 rows)
```
也可以用中文（可能会对结果造成点影响，主要看模型训练）
```sql
postgres=# WITH q AS (
    SELECT '什么是机器学习？' AS query
),
qv AS MATERIALIZED (
    SELECT e5_embedding_query(q.query) AS v FROM q
)
SELECT 
    i, 
    txt, 
    t.v <#> qv.v AS distance
FROM llm_test t, qv
ORDER BY distance
LIMIT 5;
 i  |                            txt                            |      distance       
----+-----------------------------------------------------------+---------------------
  1 | Machine learning is a subfield of artificial intelligence | -0.7919192910194397
  5 | ONNX is a standard format for machine learning models     | -0.7906289100646973
 16 | Deep learning uses multi-layer neural networks            | -0.7704752087593079
 18 | Model inference performs prediction with a trained model  | -0.7630345821380615
  4 | Vector search retrieves results by computing similarity   | -0.7604728937149048
(5 rows)
```
不说是知识库吧，简易的问答系统感觉就这样子做完了（前提是将答案录进了数据库）。
