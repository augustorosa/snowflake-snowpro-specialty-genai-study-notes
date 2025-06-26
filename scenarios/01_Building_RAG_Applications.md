[< Domain 4](../4_Snowflake_Document_AI/README.md) | **📘 Scenario 1: Building RAG Applications** | [Home >](../README.md)
***

# Scenario 1: Building RAG Applications

## Overview

This scenario walks through building a complete Retrieval-Augmented Generation (RAG) application using Snowflake Cortex. You'll learn to implement document ingestion, embedding generation, similarity search, and contextualized response generation.

## Business Context

**Company**: TechCorp Inc.  
**Challenge**: Build an intelligent knowledge base system for customer support that can answer questions based on product documentation, FAQs, and support articles.

**Requirements**:
- Ingest and process various document types (PDF, text, HTML)
- Enable semantic search across all documents
- Generate accurate, contextual responses to customer questions
- Track usage, costs, and performance
- Ensure data security and governance

## Architecture Overview

```
Documents → Preprocessing → Chunking → Embeddings → Vector Store
                                                         ↓
User Query → Query Embedding → Similarity Search → Context Retrieval → LLM Generation → Response
```

## Implementation Steps

### Step 1: Environment Setup

```sql
-- Create database and schema for RAG system
CREATE DATABASE IF NOT EXISTS customer_support_rag;
USE DATABASE customer_support_rag;
CREATE SCHEMA IF NOT EXISTS knowledge_base;
USE SCHEMA knowledge_base;

-- Create file format for document ingestion
CREATE OR REPLACE FILE FORMAT text_format
    TYPE = 'CSV'
    FIELD_DELIMITER = '|'
    SKIP_HEADER = 1
    FIELD_OPTIONALLY_ENCLOSED_BY = '"';
```

### Step 2: Document Ingestion and Storage

```sql
-- Create table for raw documents
CREATE OR REPLACE TABLE raw_documents (
    document_id STRING,
    document_type STRING,
    title STRING,
    content TEXT,
    source_url STRING,
    created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    metadata VARIANT
);

-- Insert sample documents
INSERT INTO raw_documents (document_id, document_type, title, content, source_url, metadata)
VALUES 
    ('doc_001', 'FAQ', 'Account Setup', 
     'How to set up your account: First, visit our registration page. Enter your email and create a strong password. Verify your email address by clicking the link we send you. Complete your profile with basic information.', 
     'https://support.techcorp.com/faq/account-setup',
     {'category': 'account', 'priority': 'high'}),
    
    ('doc_002', 'MANUAL', 'API Authentication', 
     'To authenticate with our API, you need to obtain an API key from your dashboard. Include the API key in the Authorization header of your requests using Bearer token format. API keys expire after 90 days and must be renewed.', 
     'https://docs.techcorp.com/api/auth',
     {'category': 'technical', 'priority': 'medium'}),
    
    ('doc_003', 'TROUBLESHOOTING', 'Connection Issues', 
     'If you are experiencing connection issues: Check your internet connection. Verify your firewall settings allow our service. Try clearing your browser cache. If problems persist, contact our support team with your error logs.', 
     'https://support.techcorp.com/troubleshooting/connection',
     {'category': 'technical', 'priority': 'high'});
```

### Step 3: Document Preprocessing and Chunking

```sql
-- Function to chunk documents into smaller pieces
CREATE OR REPLACE FUNCTION chunk_document(content TEXT, chunk_size INT DEFAULT 500)
RETURNS TABLE (chunk_id INT, chunk_text TEXT)
LANGUAGE SQL
AS $$
    WITH numbered_words AS (
        SELECT 
            ROW_NUMBER() OVER (ORDER BY SEQ) as word_position,
            VALUE as word
        FROM TABLE(SPLIT_TO_TABLE(content, ' '))
    ),
    chunks AS (
        SELECT 
            CEIL(word_position / chunk_size) as chunk_id,
            LISTAGG(word, ' ') WITHIN GROUP (ORDER BY word_position) as chunk_text
        FROM numbered_words
        GROUP BY CEIL(word_position / chunk_size)
    )
    SELECT chunk_id, chunk_text FROM chunks
$$;

-- Create table for document chunks
CREATE OR REPLACE TABLE document_chunks (
    chunk_id STRING,
    document_id STRING,
    chunk_number INT,
    chunk_text TEXT,
    chunk_metadata VARIANT,
    created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Populate chunks table
INSERT INTO document_chunks (chunk_id, document_id, chunk_number, chunk_text, chunk_metadata)
SELECT 
    document_id || '_chunk_' || chunk_id as chunk_id,
    document_id,
    chunk_id as chunk_number,
    chunk_text,
    OBJECT_CONSTRUCT(
        'document_type', document_type,
        'title', title,
        'source_url', source_url,
        'original_metadata', metadata
    ) as chunk_metadata
FROM raw_documents,
LATERAL TABLE(chunk_document(content, 200)) as chunks;
```

### Step 4: Generate Embeddings

```sql
-- Create table for embeddings
CREATE OR REPLACE TABLE document_embeddings (
    chunk_id STRING,
    document_id STRING,
    chunk_text TEXT,
    embedding VECTOR(FLOAT, 768),
    embedding_model STRING,
    created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    PRIMARY KEY (chunk_id)
);

-- Generate embeddings for all chunks
INSERT INTO document_embeddings (chunk_id, document_id, chunk_text, embedding, embedding_model)
SELECT 
    chunk_id,
    document_id,
    chunk_text,
    SNOWFLAKE.CORTEX.EMBED_TEXT_768('e5-base-v2', chunk_text) as embedding,
    'e5-base-v2' as embedding_model
FROM document_chunks;

-- Verify embeddings were created
SELECT 
    COUNT(*) as total_chunks,
    COUNT(embedding) as chunks_with_embeddings,
    AVG(ARRAY_SIZE(embedding::ARRAY)) as avg_embedding_dimension
FROM document_embeddings;
```

### Step 5: Implement Similarity Search

```sql
-- Create function for similarity search
CREATE OR REPLACE FUNCTION find_relevant_chunks(
    query_text TEXT, 
    top_k INT DEFAULT 5,
    similarity_threshold FLOAT DEFAULT 0.7
)
RETURNS TABLE (
    chunk_id STRING,
    document_id STRING,
    chunk_text TEXT,
    similarity FLOAT,
    chunk_metadata VARIANT
)
LANGUAGE SQL
AS $$
    WITH query_embedding AS (
        SELECT SNOWFLAKE.CORTEX.EMBED_TEXT_768('e5-base-v2', query_text) as query_vector
    ),
    similarities AS (
        SELECT 
            de.chunk_id,
            de.document_id,
            de.chunk_text,
            VECTOR_COSINE_SIMILARITY(de.embedding, qe.query_vector) as similarity,
            dc.chunk_metadata
        FROM document_embeddings de
        CROSS JOIN query_embedding qe
        INNER JOIN document_chunks dc ON de.chunk_id = dc.chunk_id
        WHERE VECTOR_COSINE_SIMILARITY(de.embedding, qe.query_vector) >= similarity_threshold
    )
    SELECT 
        chunk_id,
        document_id,
        chunk_text,
        similarity,
        chunk_metadata
    FROM similarities
    ORDER BY similarity DESC
    LIMIT top_k
$$;

-- Test similarity search
SELECT * FROM TABLE(find_relevant_chunks('How do I set up my account?', 3));
```

### Step 6: Build RAG Query Function

```sql
-- Create comprehensive RAG function
CREATE OR REPLACE FUNCTION rag_query(
    user_question TEXT,
    model_name STRING DEFAULT 'mistral-large',
    context_chunks INT DEFAULT 5,
    temperature FLOAT DEFAULT 0.3
)
RETURNS TABLE (
    response TEXT,
    sources ARRAY,
    context_used TEXT,
    confidence_score FLOAT
)
LANGUAGE SQL
AS $$
    WITH relevant_context AS (
        SELECT 
            chunk_text,
            similarity,
            chunk_metadata,
            chunk_id
        FROM TABLE(find_relevant_chunks(user_question, context_chunks))
    ),
    context_summary AS (
        SELECT 
            LISTAGG(chunk_text, '\n\n') as combined_context,
            ARRAY_AGG(chunk_id) as source_chunks,
            AVG(similarity) as avg_confidence
        FROM relevant_context
    ),
    generated_response AS (
        SELECT 
            SNOWFLAKE.CORTEX.COMPLETE(
                model_name,
                'You are a helpful customer support assistant. Answer the user question based on the provided context. If the context does not contain enough information, say so clearly.\n\n' ||
                'Context:\n' || combined_context || '\n\n' ||
                'Question: ' || user_question || '\n\n' ||
                'Answer:',
                OBJECT_CONSTRUCT('temperature', temperature, 'max_tokens', 500)
            ) as response,
            source_chunks,
            combined_context,
            avg_confidence
        FROM context_summary
    )
    SELECT 
        response,
        source_chunks as sources,
        combined_context as context_used,
        avg_confidence as confidence_score
    FROM generated_response
$$;

-- Test the RAG system
SELECT * FROM TABLE(rag_query('How do I authenticate with your API?'));
```

### Step 7: Create User Interface Functions

```sql
-- Simple chat interface
CREATE OR REPLACE FUNCTION chat_with_knowledge_base(
    user_message TEXT,
    conversation_history ARRAY DEFAULT ARRAY_CONSTRUCT()
)
RETURNS TABLE (
    response TEXT,
    sources ARRAY,
    updated_history ARRAY
)
LANGUAGE SQL
AS $$
    WITH rag_result AS (
        SELECT response, sources, context_used
        FROM TABLE(rag_query(user_message))
    ),
    updated_conversation AS (
        SELECT 
            ARRAY_APPEND(
                ARRAY_APPEND(conversation_history, 
                    OBJECT_CONSTRUCT('role', 'user', 'content', user_message)
                ),
                OBJECT_CONSTRUCT('role', 'assistant', 'content', response)
            ) as new_history
        FROM rag_result
    )
    SELECT 
        r.response,
        r.sources,
        c.new_history as updated_history
    FROM rag_result r
    CROSS JOIN updated_conversation c
$$;
```

### Step 8: Analytics and Monitoring

```sql
-- Create usage tracking table
CREATE OR REPLACE TABLE rag_usage_log (
    query_id STRING DEFAULT UUID_STRING(),
    user_question TEXT,
    response_text TEXT,
    sources_used ARRAY,
    confidence_score FLOAT,
    processing_time_ms INT,
    token_usage INT,
    timestamp TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Function to log RAG queries
CREATE OR REPLACE FUNCTION logged_rag_query(user_question TEXT)
RETURNS TABLE (response TEXT, sources ARRAY, query_id STRING)
LANGUAGE SQL
AS $$
    WITH start_time AS (
        SELECT CURRENT_TIMESTAMP() as start_ts
    ),
    rag_result AS (
        SELECT response, sources, context_used, confidence_score
        FROM TABLE(rag_query(user_question))
    ),
    end_time AS (
        SELECT CURRENT_TIMESTAMP() as end_ts
    ),
    log_entry AS (
        INSERT INTO rag_usage_log (
            user_question, response_text, sources_used, confidence_score, 
            processing_time_ms, token_usage
        )
        SELECT 
            user_question,
            response,
            sources,
            confidence_score,
            DATEDIFF('millisecond', start_ts, end_ts) as processing_time,
            SNOWFLAKE.CORTEX.COUNT_TOKENS('mistral-large', response) as tokens
        FROM rag_result, start_time, end_time
        RETURNING query_id, response_text, sources_used
    )
    SELECT response_text as response, sources_used as sources, query_id
    FROM log_entry
$$;

-- Analytics queries
-- Usage statistics
SELECT 
    DATE_TRUNC('day', timestamp) as day,
    COUNT(*) as query_count,
    AVG(confidence_score) as avg_confidence,
    AVG(processing_time_ms) as avg_processing_time_ms,
    SUM(token_usage) as total_tokens
FROM rag_usage_log
GROUP BY DATE_TRUNC('day', timestamp)
ORDER BY day DESC;

-- Most common questions
SELECT 
    LOWER(user_question) as question,
    COUNT(*) as frequency,
    AVG(confidence_score) as avg_confidence
FROM rag_usage_log
GROUP BY LOWER(user_question)
ORDER BY frequency DESC
LIMIT 10;
```

### Step 9: Performance Optimization

```sql
-- Create indexes for better performance
CREATE OR REPLACE TABLE document_embeddings_optimized (
    chunk_id STRING,
    document_id STRING,
    chunk_text TEXT,
    embedding VECTOR(FLOAT, 768),
    embedding_model STRING,
    created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    -- Add search tags for better filtering
    SEARCH OPTIMIZATION ON (chunk_text, document_id)
);

-- Batch processing for large document sets
CREATE OR REPLACE PROCEDURE process_documents_batch(batch_size INT DEFAULT 100)
RETURNS STRING
LANGUAGE SQL
AS $$
DECLARE
    processed_count INT DEFAULT 0;
    batch_cursor CURSOR FOR 
        SELECT document_id, content 
        FROM raw_documents 
        WHERE document_id NOT IN (SELECT DISTINCT document_id FROM document_embeddings)
        LIMIT batch_size;
BEGIN
    FOR record IN batch_cursor DO
        -- Process chunks and embeddings for each document
        INSERT INTO document_chunks (chunk_id, document_id, chunk_number, chunk_text)
        SELECT 
            record.document_id || '_chunk_' || chunk_id,
            record.document_id,
            chunk_id,
            chunk_text
        FROM TABLE(chunk_document(record.content));
        
        processed_count := processed_count + 1;
    END FOR;
    
    RETURN 'Processed ' || processed_count || ' documents';
END;
$$;
```

### Step 10: Security and Governance

```sql
-- Create RBAC for RAG system
CREATE ROLE IF NOT EXISTS rag_user;
CREATE ROLE IF NOT EXISTS rag_admin;

-- Grant permissions
GRANT USAGE ON DATABASE customer_support_rag TO ROLE rag_user;
GRANT USAGE ON SCHEMA knowledge_base TO ROLE rag_user;
GRANT SELECT ON TABLE document_embeddings TO ROLE rag_user;
GRANT USAGE ON FUNCTION rag_query(TEXT) TO ROLE rag_user;

-- Admin permissions
GRANT ALL ON DATABASE customer_support_rag TO ROLE rag_admin;
GRANT ALL ON SCHEMA knowledge_base TO ROLE rag_admin;

-- Data masking for sensitive content
CREATE OR REPLACE MASKING POLICY mask_sensitive_content AS (val STRING) 
RETURNS STRING ->
    CASE 
        WHEN CURRENT_ROLE() IN ('RAG_ADMIN') THEN val
        ELSE REGEXP_REPLACE(val, '[0-9]{4}-[0-9]{4}-[0-9]{4}-[0-9]{4}', 'XXXX-XXXX-XXXX-XXXX')
    END;

-- Apply masking to chunk text
ALTER TABLE document_chunks MODIFY COLUMN chunk_text 
SET MASKING POLICY mask_sensitive_content;
```

## Testing and Validation

### Test Cases

```sql
-- Test 1: Basic functionality
SELECT 'Test 1: Basic Query' as test_name,
       response, 
       ARRAY_SIZE(sources) as source_count
FROM TABLE(rag_query('How do I create an account?'));

-- Test 2: Technical question
SELECT 'Test 2: Technical Query' as test_name,
       response, 
       confidence_score
FROM TABLE(rag_query('What are the API authentication requirements?'));

-- Test 3: Unknown topic
SELECT 'Test 3: Unknown Topic' as test_name,
       response,
       confidence_score
FROM TABLE(rag_query('How do I bake a cake?'));

-- Test 4: Performance test
SELECT 'Test 4: Performance' as test_name,
       COUNT(*) as total_queries,
       AVG(processing_time_ms) as avg_time_ms
FROM rag_usage_log
WHERE timestamp >= CURRENT_TIMESTAMP - INTERVAL '1 hour';
```

## Success Metrics

- **Accuracy**: >85% of responses should be relevant and accurate
- **Response Time**: <2 seconds average processing time
- **User Satisfaction**: Confidence score >0.7 for 80% of queries
- **Coverage**: System should handle 90% of common support questions

## Common Issues and Solutions

1. **Low Similarity Scores**: Adjust embedding model or chunk size
2. **Slow Performance**: Optimize vector operations or increase warehouse size
3. **Irrelevant Responses**: Improve chunking strategy or increase similarity threshold
4. **High Costs**: Monitor token usage and optimize model selection

## Next Steps

1. Implement feedback collection system
2. Add multi-language support
3. Integrate with existing chat platforms
4. Implement continuous learning from user interactions
5. Add document versioning and update mechanisms

This scenario provides a complete RAG implementation that can be adapted for various use cases and scaled for production environments.

***
[< Domain 4](../4_Snowflake_Document_AI/README.md) | **📘 Scenario 1: Building RAG Applications** | [Home >](../README.md) 