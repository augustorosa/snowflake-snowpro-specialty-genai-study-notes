[< Scenarios](../scenarios/01_Building_RAG_Applications.md) | **📘 Cortex Functions Quick Reference** | [Exam Prep >](../exam_prep/Exam_Strategy.md)
***

# Snowflake Cortex Functions Quick Reference

## Core Function Syntax

### **Text Generation**
```sql
-- Basic text completion
SNOWFLAKE.CORTEX.COMPLETE(model_name, prompt, [options])

-- Conversational AI
SNOWFLAKE.CORTEX.CHAT(model_name, messages_array, [options])
```

### **Text Analysis**
```sql
-- Sentiment analysis (-1 to 1)
SNOWFLAKE.CORTEX.SENTIMENT(text)

-- Extract answers from context
SNOWFLAKE.CORTEX.EXTRACT_ANSWER(question, context)

-- Text classification
SNOWFLAKE.CORTEX.CLASSIFY(text, categories_array)
```

### **Text Transformation**
```sql
-- Text summarization
SNOWFLAKE.CORTEX.SUMMARIZE(text, [length])

-- Language translation
SNOWFLAKE.CORTEX.TRANSLATE(text, source_lang, target_lang)
```

### **Embeddings**
```sql
-- 768-dimensional embeddings
SNOWFLAKE.CORTEX.EMBED_TEXT_768(model_name, text)

-- 1024-dimensional embeddings
SNOWFLAKE.CORTEX.EMBED_TEXT_1024(model_name, text)
```

### **Utilities**
```sql
-- Token counting
SNOWFLAKE.CORTEX.COUNT_TOKENS(model_name, text)
```

### **AI Functions (Preview - Not on Exam)**
```sql
-- ⚠️ PREVIEW ONLY - These functions are not on the certification exam
-- Text completion with advanced features
AI_COMPLETE(model_name, prompt, [options])

-- Classification with multi-label and image support
AI_CLASSIFY(input, labels, [options])

-- Boolean filtering with natural language
AI_FILTER(condition, data)

-- Aggregate insights across multiple rows
AI_AGG(operation, data_column, [context])

-- Calculate embedding similarity
AI_SIMILARITY(input1, input2, [options])
```

## Available Models

### **LLM Models**

**Large Models (Best Performance):**
- `'llama3.1-405b'` - Meta's largest model for complex reasoning
- `'claude-3.5-sonnet'` - Anthropic's advanced model 
- `'gpt-4o'` - OpenAI's flagship multimodal model
- `'mistral-large2'` - Mistral's most capable model

**Medium Models (Balanced Performance/Cost):**
- `'llama3.1-70b'` - Strong performance for most use cases
- `'claude-3-haiku'` - Fast and cost-effective from Anthropic
- `'gpt-4o-mini'` - OpenAI's efficient model
- `'mixtral-8x7b'` - Mixture of experts model
- `'mistral-7b'` - General-purpose model

**Small Models (Cost-Optimized):**
- `'llama3.1-8b'` - Lightweight but capable
- `'gemma-7b'` - Google's efficient model
- `'llama2-70b-chat'` - Optimized for conversations
- `'llama2-7b-chat'` - Lightweight conversational model
- `'deepseek-coder-6.7b'` - Specialized for code generation

### **Embedding Models**
- `'snowflake-arctic-embed-l'` - Snowflake's large embedding model
- `'snowflake-arctic-embed-m'` - Snowflake's medium embedding model
- `'e5-base-v2'` - General purpose 768-dimensional embeddings
- `'multilingual-e5-large'` - Multi-language embedding support

## Vector Operations

### **Vector Functions**
```sql
-- Cosine similarity
VECTOR_COSINE_SIMILARITY(vector1, vector2)

-- Euclidean distance
VECTOR_L2_DISTANCE(vector1, vector2)

-- Dot product
VECTOR_DOT_PRODUCT(vector1, vector2)
```

### **Vector Data Types**
```sql
-- Float vector
VECTOR(FLOAT, 768)

-- Integer vector
VECTOR(INT, 256)
```

## Common Patterns

### **RAG Implementation**
```sql
-- 1. Create embeddings
CREATE TABLE docs AS
SELECT id, content,
       SNOWFLAKE.CORTEX.EMBED_TEXT_768('e5-base-v2', content) AS embedding
FROM source_docs;

-- 2. Similarity search
WITH query_vec AS (
    SELECT SNOWFLAKE.CORTEX.EMBED_TEXT_768('e5-base-v2', 'query') AS vec
)
SELECT id, VECTOR_COSINE_SIMILARITY(embedding, vec) AS similarity
FROM docs, query_vec
ORDER BY similarity DESC LIMIT 5;

-- 3. Generate response
SELECT SNOWFLAKE.CORTEX.COMPLETE(
    'mistral-large',
    'Answer based on context: ' || context || '\nQuestion: ' || question
);
```

### **Batch Processing**
```sql
-- Process multiple records efficiently
SELECT id, 
       SNOWFLAKE.CORTEX.SENTIMENT(review_text) AS sentiment,
       SNOWFLAKE.CORTEX.CLASSIFY(review_text, ['positive', 'negative', 'neutral']) AS category
FROM reviews
WHERE processed_date IS NULL;
```

### **Cost Optimization**
```sql
-- Token counting before processing
WITH token_check AS (
    SELECT id, text,
           SNOWFLAKE.CORTEX.COUNT_TOKENS('mistral-large', text) AS tokens
    FROM input_data
)
SELECT * FROM token_check WHERE tokens <= 1000;
```

## Language Codes

### **Supported Languages**
- `'en'` - English
- `'es'` - Spanish
- `'fr'` - French
- `'de'` - German
- `'it'` - Italian
- `'pt'` - Portuguese
- `'zh'` - Chinese
- `'ja'` - Japanese
- `'ko'` - Korean
- `'ru'` - Russian
- `'ar'` - Arabic

## Function Options

### **COMPLETE() Options**
```sql
{
    'temperature': 0.7,        -- Creativity (0.0-1.0)
    'max_tokens': 500,         -- Maximum response length
    'top_p': 0.9,             -- Nucleus sampling
    'stop': ['\n', '.']       -- Stop sequences
}
```

### **Chat Message Format**
```sql
[
    {'role': 'system', 'content': 'You are a helpful assistant'},
    {'role': 'user', 'content': 'User message'},
    {'role': 'assistant', 'content': 'AI response'},
    {'role': 'user', 'content': 'Follow-up question'}
]
```

## Error Handling

### **Try Functions**
```sql
-- Safe function calls
SELECT id,
       TRY_TO_TEXT(SNOWFLAKE.CORTEX.SENTIMENT(text)) AS sentiment,
       CASE WHEN TRY_TO_TEXT(SNOWFLAKE.CORTEX.SENTIMENT(text)) IS NULL 
            THEN 'Processing Error' 
            ELSE 'Success' END AS status
FROM input_data;
```

### **Common Error Patterns**
```sql
-- Handle null/empty inputs
SELECT CASE 
    WHEN text IS NULL OR LENGTH(text) = 0 THEN NULL
    ELSE SNOWFLAKE.CORTEX.SENTIMENT(text)
END AS sentiment
FROM reviews;
```

## Performance Tips

1. **Batch Operations**: Process multiple rows in single query
2. **Model Selection**: Use smallest model that meets requirements
3. **Caching**: Leverage Snowflake's result caching
4. **Token Management**: Monitor token usage for cost control
5. **Parallel Processing**: Use appropriate warehouse size

## Monitoring Queries

### **Usage Tracking**
```sql
-- Monitor Cortex usage
SELECT query_text, execution_time, credits_used
FROM information_schema.query_history
WHERE query_text ILIKE '%CORTEX%'
ORDER BY start_time DESC;
```

### **Cost Analysis**
```sql
-- Analyze costs by user
SELECT user_name, 
       COUNT(*) AS query_count,
       SUM(credits_used) AS total_credits
FROM snowflake.account_usage.query_history
WHERE query_text ILIKE '%CORTEX%'
GROUP BY user_name;
```

***
[< Scenarios](../scenarios/01_Building_RAG_Applications.md) | **📘 Cortex Functions Quick Reference** | [Exam Prep >](../exam_prep/Exam_Strategy.md) 