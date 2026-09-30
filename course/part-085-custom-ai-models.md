# ตอนที่ 85: Custom AI Models สำหรับ LINE Bots

## บทนำ

การสร้าง AI Model ที่ปรับแต่งเฉพาะสำหรับธุรกิจของคุณช่วยให้ LINE Bot ฉลาดขึ้น เข้าใจบริบทของธุรกิจได้ดีขึ้น และให้คำตอบที่ตรงกับความต้องการของลูกค้ามากขึ้น

## สถาปัตยกรรม AI System

```
┌────────────────────────────────────────────────────────────────────┐
│                    Custom AI System for LINE Bot                    │
│                                                                      │
│  ┌────────────┐  ┌─────────────────────────────────────────────┐   │
│  │  LINE OA   │  │              AI Processing Layer             │   │
│  │  Webhook   │──▶  ┌──────────┐  ┌──────────┐  ┌──────────┐  │   │
│  └────────────┘  │  │ Semantic │  │   RAG    │  │   LLM    │  │   │
│                  │  │ Search   │  │ Context  │  │ Generate │  │   │
│                  │  └──────────┘  └──────────┘  └──────────┘  │   │
│                  └─────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Storage Layer                             │   │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────┐  ┌──────────┐ │   │
│  │  │ Pinecone │  │Weaviate  │  │   Chroma   │  │  MySQL   │ │   │
│  │  │(Vector)  │  │(Vector)  │  │  (Local)   │  │  (Data)  │ │   │
│  │  └──────────┘  └──────────┘  └────────────┘  └──────────┘ │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    LLM Options                               │   │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────┐  ┌──────────┐ │   │
│  │  │  GPT-4   │  │ Claude 3 │  │   Ollama   │  │Fine-tune │ │   │
│  │  │(OpenAI)  │  │(Anthropic│  │  (Local)   │  │  Model   │ │   │
│  │  └──────────┘  └──────────┘  └────────────┘  └──────────┘ │   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

## 1. Thai Language Models Overview

### ภาษาไทยกับ AI
ภาษาไทยมีความท้าทายพิเศษ:
- ไม่มีช่องว่างระหว่างคำ (Word Segmentation ยาก)
- มี Tone markers ที่เปลี่ยนความหมาย
- มีคำยืมจากภาษาอื่นมาก
- สคริปต์พิเศษที่ต้องการ Tokenizer เฉพาะ

### Models ที่รองรับภาษาไทยได้ดี

| Model | Provider | ภาษาไทย | Cost | Speed |
|-------|----------|---------|------|-------|
| GPT-4o | OpenAI | ดีมาก | สูง | กลาง |
| Claude 3.5 Sonnet | Anthropic | ดีมาก | กลาง | เร็ว |
| Llama 3.1 70B | Meta | ดี | ต่ำ | ช้า |
| WangchanX | NECTEC | ดีมาก | ฟรี | ช้า |
| OpenThaiGPT | Thai | ดีมาก | ต่ำ | กลาง |
| Typhoon | SCB 10X | ดีมาก | ต่ำ | เร็ว |

## 2. Training Data Preparation จาก LINE Conversations

```python
# scripts/prepare_training_data.py
import json
import mysql.connector
import pandas as pd
from datetime import datetime, timedelta
from typing import List, Dict, Tuple
import re

class TrainingDataPreparer:
    def __init__(self, db_config: dict):
        self.db = mysql.connector.connect(**db_config)
        self.cursor = self.db.cursor(dictionary=True)
    
    def extract_conversations(self, days_back: int = 90) -> List[Dict]:
        """ดึงการสนทนาจาก Database"""
        start_date = datetime.now() - timedelta(days=days_back)
        
        self.cursor.execute("""
            SELECT 
                a.contact_id,
                a.direction,
                a.body,
                a.occurred_at,
                c.display_name
            FROM activities a
            JOIN contacts c ON a.contact_id = c.id
            WHERE 
                a.type = 'line_message'
                AND a.occurred_at >= %s
                AND a.body IS NOT NULL
                AND LENGTH(a.body) > 3
            ORDER BY a.contact_id, a.occurred_at ASC
        """, (start_date,))
        
        return self.cursor.fetchall()
    
    def group_into_conversations(self, messages: List[Dict], 
                                  session_timeout_minutes: int = 30) -> List[List[Dict]]:
        """จัดกลุ่มข้อความเป็น Conversations"""
        if not messages:
            return []
        
        conversations = []
        current_conv = [messages[0]]
        
        for i in range(1, len(messages)):
            curr_msg = messages[i]
            prev_msg = messages[i-1]
            
            # ตรวจสอบว่าเป็น Contact เดิมและอยู่ใน Session เดียวกัน
            time_diff = (curr_msg['occurred_at'] - prev_msg['occurred_at']).total_seconds() / 60
            same_contact = curr_msg['contact_id'] == prev_msg['contact_id']
            
            if same_contact and time_diff <= session_timeout_minutes:
                current_conv.append(curr_msg)
            else:
                if len(current_conv) >= 2:  # ต้องมีอย่างน้อย 2 ข้อความ
                    conversations.append(current_conv)
                current_conv = [curr_msg]
        
        if len(current_conv) >= 2:
            conversations.append(current_conv)
        
        return conversations
    
    def create_qa_pairs(self, conversations: List[List[Dict]]) -> List[Dict]:
        """สร้าง Q&A Pairs จากการสนทนา"""
        qa_pairs = []
        
        for conv in conversations:
            for i in range(len(conv) - 1):
                msg = conv[i]
                next_msg = conv[i + 1]
                
                # ต้องการ: คำถามจาก User -> คำตอบจาก Bot
                if msg['direction'] == 'inbound' and next_msg['direction'] == 'outbound':
                    # กรองข้อความที่ไม่ดี
                    if self.is_valid_pair(msg['body'], next_msg['body']):
                        qa_pairs.append({
                            'question': self.clean_text(msg['body']),
                            'answer': self.clean_text(next_msg['body']),
                            'context': self.get_conversation_context(conv[:i])
                        })
        
        return qa_pairs
    
    def is_valid_pair(self, question: str, answer: str) -> bool:
        """ตรวจสอบความถูกต้องของ Q&A Pair"""
        # ตรวจสอบความยาว
        if len(question) < 3 or len(answer) < 5:
            return False
        
        # กรองข้อความที่เป็นแค่ Sticker/Image
        invalid_patterns = [
            r'^\[sticker\]$',
            r'^\[image\]$',
            r'^\[video\]$',
            r'^(สวัสดี|hello|hi|ok|โอเค|ได้เลย|ขอบคุณ)$'
        ]
        
        for pattern in invalid_patterns:
            if re.match(pattern, question.lower()) or re.match(pattern, answer.lower()):
                return False
        
        return True
    
    def clean_text(self, text: str) -> str:
        """ทำความสะอาดข้อความ"""
        # ลบ Whitespace ซ้ำ
        text = re.sub(r'\s+', ' ', text).strip()
        # ลบ URL
        text = re.sub(r'https?://\S+', '[URL]', text)
        # ลบ Phone numbers
        text = re.sub(r'(?:\+66|0)[0-9]{8,9}', '[PHONE]', text)
        return text
    
    def get_conversation_context(self, previous_messages: List[Dict]) -> str:
        """ดึง Context จากข้อความก่อนหน้า"""
        if not previous_messages:
            return ""
        
        context_msgs = previous_messages[-3:]  # เอาแค่ 3 ข้อความล่าสุด
        context = "\n".join([
            f"{'User' if m['direction'] == 'inbound' else 'Bot'}: {m['body']}"
            for m in context_msgs
        ])
        return context
    
    def export_for_fine_tuning(self, qa_pairs: List[Dict], 
                                output_file: str = 'training_data.jsonl',
                                format: str = 'openai') -> int:
        """Export ข้อมูลในรูปแบบที่ใช้ Fine-tuning"""
        
        with open(output_file, 'w', encoding='utf-8') as f:
            for pair in qa_pairs:
                if format == 'openai':
                    # OpenAI Fine-tuning Format
                    record = {
                        "messages": [
                            {
                                "role": "system",
                                "content": "คุณเป็น AI Assistant สำหรับ LINE Bot ของร้านเรา คุณตอบคำถามเกี่ยวกับสินค้าและบริการเป็นภาษาไทย"
                            }
                        ]
                    }
                    
                    # เพิ่ม Context ถ้ามี
                    if pair['context']:
                        record['messages'].append({
                            "role": "user",
                            "content": f"บริบทการสนทนาก่อนหน้า:\n{pair['context']}"
                        })
                        record['messages'].append({
                            "role": "assistant",
                            "content": "รับทราบครับ"
                        })
                    
                    record['messages'].extend([
                        {"role": "user", "content": pair['question']},
                        {"role": "assistant", "content": pair['answer']}
                    ])
                    
                elif format == 'alpaca':
                    # Alpaca Format สำหรับ Local Models
                    record = {
                        "instruction": "ตอบคำถามของลูกค้าในฐานะ Customer Service ของร้านค้า",
                        "input": pair['question'],
                        "output": pair['answer']
                    }
                
                f.write(json.dumps(record, ensure_ascii=False) + '\n')
        
        return len(qa_pairs)
    
    def run(self):
        """รัน Pipeline ทั้งหมด"""
        print("กำลังดึงข้อมูลการสนทนา...")
        messages = self.extract_conversations(days_back=90)
        print(f"ดึงข้อมูลได้ {len(messages)} ข้อความ")
        
        print("กำลังจัดกลุ่มเป็น Conversations...")
        conversations = self.group_into_conversations(messages)
        print(f"จัดกลุ่มได้ {len(conversations)} conversations")
        
        print("กำลังสร้าง Q&A Pairs...")
        qa_pairs = self.create_qa_pairs(conversations)
        print(f"สร้างได้ {len(qa_pairs)} Q&A pairs")
        
        print("กำลัง Export ข้อมูล...")
        count = self.export_for_fine_tuning(
            qa_pairs, 
            'training_data.jsonl',
            format='openai'
        )
        print(f"Export สำเร็จ: {count} records")
        
        # สถิติ
        df = pd.DataFrame(qa_pairs)
        print("\nสถิติข้อมูล:")
        print(f"  ความยาวเฉลี่ยคำถาม: {df['question'].str.len().mean():.0f} ตัวอักษร")
        print(f"  ความยาวเฉลี่ยคำตอบ: {df['answer'].str.len().mean():.0f} ตัวอักษร")


# รัน Script
if __name__ == '__main__':
    db_config = {
        'host': 'localhost',
        'user': 'root',
        'password': 'your_password',
        'database': 'line_crm'
    }
    
    preparer = TrainingDataPreparer(db_config)
    preparer.run()
```

## 3. RAG (Retrieval Augmented Generation) สำหรับ LINE Bots

```javascript
// src/ai/rag-system.js
const { OpenAI } = require('openai');
const { Pinecone } = require('@pinecone-database/pinecone');
const db = require('../database/mysql');

class RAGSystem {
    constructor() {
        this.openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
        this.pinecone = new Pinecone({ apiKey: process.env.PINECONE_API_KEY });
        this.index = this.pinecone.index(process.env.PINECONE_INDEX_NAME || 'line-bot-knowledge');
        
        this.embeddingModel = 'text-embedding-3-small';
        this.chatModel = process.env.CHAT_MODEL || 'gpt-4o-mini';
        this.maxContextLength = 3000;
        this.topK = 5;
    }

    // สร้าง Embedding สำหรับข้อความ
    async createEmbedding(text) {
        const response = await this.openai.embeddings.create({
            model: this.embeddingModel,
            input: text.slice(0, 8000) // จำกัดความยาว
        });
        
        return response.data[0].embedding;
    }

    // ค้นหา Context ที่เกี่ยวข้อง
    async retrieveContext(query, topK = null) {
        const queryEmbedding = await this.createEmbedding(query);
        
        const results = await this.index.query({
            vector: queryEmbedding,
            topK: topK || this.topK,
            includeMetadata: true
        });
        
        return results.matches
            .filter(m => m.score >= 0.7) // กรอง Relevance Score
            .map(m => ({
                content: m.metadata.content,
                source: m.metadata.source,
                title: m.metadata.title,
                score: m.score
            }));
    }

    // สร้าง Response โดยใช้ RAG
    async generateResponse(userMessage, conversationHistory = [], userProfile = {}) {
        try {
            // 1. ดึง Context ที่เกี่ยวข้อง
            const contexts = await this.retrieveContext(userMessage);
            
            // 2. สร้าง Context String
            let contextStr = '';
            if (contexts.length > 0) {
                contextStr = "ข้อมูลที่เกี่ยวข้องจากฐานความรู้:\n\n";
                contexts.forEach((ctx, i) => {
                    contextStr += `[${i + 1}] ${ctx.title || 'ข้อมูล'}\n${ctx.content}\n\n`;
                });
            }
            
            // 3. สร้าง System Prompt
            const systemPrompt = this.buildSystemPrompt(userProfile, contextStr);
            
            // 4. สร้าง Messages Array
            const messages = [
                { role: 'system', content: systemPrompt },
                ...conversationHistory.slice(-6), // เอาแค่ 3 รอบล่าสุด
                { role: 'user', content: userMessage }
            ];
            
            // 5. เรียก LLM
            const response = await this.openai.chat.completions.create({
                model: this.chatModel,
                messages,
                max_tokens: 500,
                temperature: 0.7,
                stream: false
            });
            
            const answer = response.choices[0].message.content;
            
            // 6. บันทึก Usage สำหรับ Cost Tracking
            await this.trackUsage(response.usage);
            
            return {
                answer,
                contexts: contexts.length > 0 ? contexts : null,
                model: this.chatModel,
                tokens: response.usage
            };
        } catch (error) {
            console.error('RAG generation error:', error);
            throw error;
        }
    }

    // สร้าง System Prompt
    buildSystemPrompt(userProfile, context) {
        let prompt = `คุณเป็น AI Assistant ที่ช่วยเหลือลูกค้าของ ${process.env.COMPANY_NAME || 'บริษัทของเรา'}
คุณต้องตอบคำถามเป็นภาษาไทยอย่างสุภาพและเป็นมืออาชีพ

กฎการตอบ:
1. ตอบตรงประเด็น กระชับ ไม่เกิน 5 ประโยค
2. ใช้ภาษาไทยที่เป็นทางการแต่เป็นกันเอง
3. ถ้าไม่รู้คำตอบ ให้บอกว่าจะส่งต่อให้เจ้าหน้าที่
4. อย่าสร้างข้อมูลที่ไม่มีในฐานความรู้`;

        if (userProfile.name) {
            prompt += `\n\nข้อมูลลูกค้า: ชื่อ ${userProfile.name}`;
        }
        
        if (context) {
            prompt += `\n\n${context}`;
        }
        
        return prompt;
    }

    // ติดตาม API Usage
    async trackUsage(usage) {
        await db.query(`
            INSERT INTO ai_usage_log (
                model, prompt_tokens, completion_tokens, total_tokens,
                estimated_cost, occurred_at
            ) VALUES (?, ?, ?, ?, ?, NOW())
        `, [
            this.chatModel,
            usage.prompt_tokens,
            usage.completion_tokens,
            usage.total_tokens,
            this.calculateCost(usage)
        ]);
    }

    // คำนวณค่าใช้จ่าย
    calculateCost(usage) {
        const pricing = {
            'gpt-4o': { input: 0.0025, output: 0.01 },
            'gpt-4o-mini': { input: 0.00015, output: 0.0006 },
            'gpt-3.5-turbo': { input: 0.0005, output: 0.0015 }
        };
        
        const model = pricing[this.chatModel] || pricing['gpt-4o-mini'];
        const cost = (usage.prompt_tokens / 1000 * model.input) + 
                     (usage.completion_tokens / 1000 * model.output);
        
        return Math.round(cost * 10000) / 10000; // USD
    }
}

module.exports = new RAGSystem();
```

## 4. Vector Databases

### 4.1 Pinecone Integration

```javascript
// src/ai/knowledge-base/pinecone-manager.js
const { Pinecone } = require('@pinecone-database/pinecone');
const { OpenAI } = require('openai');
const { v4: uuidv4 } = require('uuid');
const db = require('../../database/mysql');

class PineconeKnowledgeBase {
    constructor() {
        this.pinecone = new Pinecone({ apiKey: process.env.PINECONE_API_KEY });
        this.openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
        this.indexName = process.env.PINECONE_INDEX_NAME || 'line-bot-kb';
    }

    // สร้าง Index ใหม่
    async createIndex() {
        await this.pinecone.createIndex({
            name: this.indexName,
            dimension: 1536, // สำหรับ text-embedding-3-small
            metric: 'cosine',
            spec: {
                serverless: {
                    cloud: 'aws',
                    region: 'us-east-1'
                }
            }
        });
        
        console.log(`Created Pinecone index: ${this.indexName}`);
    }

    // เพิ่ม Documents เข้า Knowledge Base
    async addDocuments(documents) {
        const index = this.pinecone.index(this.indexName);
        const batchSize = 100;
        
        for (let i = 0; i < documents.length; i += batchSize) {
            const batch = documents.slice(i, i + batchSize);
            
            // สร้าง Embeddings
            const embeddings = await this.createEmbeddings(batch.map(d => d.content));
            
            // เตรียม Vectors
            const vectors = batch.map((doc, j) => ({
                id: doc.id || uuidv4(),
                values: embeddings[j],
                metadata: {
                    content: doc.content.slice(0, 1000), // จำกัดขนาด Metadata
                    title: doc.title || '',
                    source: doc.source || '',
                    category: doc.category || '',
                    created_at: new Date().toISOString()
                }
            }));
            
            // Upsert ไป Pinecone
            await index.upsert(vectors);
            
            console.log(`Uploaded ${i + batch.length}/${documents.length} documents`);
        }
    }

    // สร้าง Embeddings แบบ Batch
    async createEmbeddings(texts) {
        const response = await this.openai.embeddings.create({
            model: 'text-embedding-3-small',
            input: texts
        });
        
        return response.data.map(d => d.embedding);
    }

    // โหลด FAQ จาก Database เข้า Pinecone
    async loadFAQFromDatabase() {
        const faqs = await db.query(`
            SELECT 
                id, question, answer, category,
                CONCAT(question, '\n\nคำตอบ: ', answer) as content
            FROM knowledge_base
            WHERE is_active = 1
            ORDER BY priority DESC
        `);
        
        const documents = faqs.map(faq => ({
            id: `faq-${faq.id}`,
            content: faq.content,
            title: faq.question,
            source: 'faq',
            category: faq.category
        }));
        
        await this.addDocuments(documents);
        console.log(`Loaded ${documents.length} FAQs into Pinecone`);
    }

    // โหลด Product Info
    async loadProductsFromDatabase() {
        const products = await db.query(`
            SELECT 
                id, name, description, price, category,
                CONCAT(
                    'สินค้า: ', name, '\n',
                    'คำอธิบาย: ', COALESCE(description, ''), '\n',
                    'ราคา: ', price, ' บาท\n',
                    'หมวดหมู่: ', category
                ) as content
            FROM products
            WHERE is_active = 1
        `);
        
        const documents = products.map(p => ({
            id: `product-${p.id}`,
            content: p.content,
            title: p.name,
            source: 'product_catalog',
            category: p.category
        }));
        
        await this.addDocuments(documents);
        console.log(`Loaded ${documents.length} products into Pinecone`);
    }

    // ลบ Document
    async deleteDocument(docId) {
        const index = this.pinecone.index(this.indexName);
        await index.deleteOne(docId);
    }

    // ดึงสถิติ Index
    async getIndexStats() {
        const index = this.pinecone.index(this.indexName);
        return await index.describeIndexStats();
    }
}

module.exports = new PineconeKnowledgeBase();
```

### 4.2 Chroma (Local Vector Database)

```javascript
// src/ai/knowledge-base/chroma-manager.js
const { ChromaClient, OpenAIEmbeddingFunction } = require('chromadb');

class ChromaKnowledgeBase {
    constructor() {
        this.client = new ChromaClient({
            path: process.env.CHROMA_URL || 'http://localhost:8000'
        });
        
        this.embeddingFunction = new OpenAIEmbeddingFunction({
            openai_api_key: process.env.OPENAI_API_KEY,
            openai_model: 'text-embedding-3-small'
        });
        
        this.collectionName = 'line_bot_knowledge';
    }

    // สร้างหรือดึง Collection
    async getOrCreateCollection() {
        return await this.client.getOrCreateCollection({
            name: this.collectionName,
            embeddingFunction: this.embeddingFunction,
            metadata: {
                description: 'LINE Bot Knowledge Base',
                'hnsw:space': 'cosine'
            }
        });
    }

    // เพิ่มเอกสาร
    async addDocuments(documents) {
        const collection = await this.getOrCreateCollection();
        
        const ids = documents.map((d, i) => d.id || `doc-${i}-${Date.now()}`);
        const contents = documents.map(d => d.content);
        const metadatas = documents.map(d => ({
            title: d.title || '',
            source: d.source || '',
            category: d.category || ''
        }));
        
        await collection.add({
            ids,
            documents: contents,
            metadatas
        });
        
        console.log(`Added ${documents.length} documents to Chroma`);
    }

    // ค้นหา
    async search(query, nResults = 5) {
        const collection = await this.getOrCreateCollection();
        
        const results = await collection.query({
            queryTexts: [query],
            nResults
        });
        
        if (!results.documents[0].length) return [];
        
        return results.documents[0].map((doc, i) => ({
            content: doc,
            metadata: results.metadatas[0][i],
            distance: results.distances[0][i]
        }));
    }

    // ลบ Documents ทั้งหมด
    async clearCollection() {
        try {
            await this.client.deleteCollection(this.collectionName);
            console.log('Collection cleared');
        } catch (e) {
            console.log('Collection not found, creating new one');
        }
    }
}

module.exports = new ChromaKnowledgeBase();
```

## 5. Building Knowledge Base จาก FAQ

```javascript
// src/ai/knowledge-base/faq-loader.js
const db = require('../../database/mysql');
const pdfParse = require('pdf-parse');
const fs = require('fs');
const path = require('path');
const mammoth = require('mammoth');

class FAQLoader {
    constructor(vectorDB) {
        this.vectorDB = vectorDB; // Pinecone หรือ Chroma
    }

    // โหลด FAQ จาก Database
    async loadFromDatabase() {
        const faqs = await db.query(`
            SELECT * FROM knowledge_base 
            WHERE is_active = 1 
            ORDER BY category, priority DESC
        `);
        
        return faqs.map(faq => ({
            id: `faq-${faq.id}`,
            content: `คำถาม: ${faq.question}\n\nคำตอบ: ${faq.answer}`,
            title: faq.question,
            source: 'faq_database',
            category: faq.category
        }));
    }

    // โหลดจาก JSON File
    async loadFromJSON(filePath) {
        const data = JSON.parse(fs.readFileSync(filePath, 'utf8'));
        
        return data.map((item, i) => ({
            id: item.id || `json-${i}`,
            content: `คำถาม: ${item.question}\n\nคำตอบ: ${item.answer}`,
            title: item.question,
            source: 'json_file',
            category: item.category || 'general'
        }));
    }

    // โหลดจาก PDF
    async loadFromPDF(filePath) {
        const pdfBuffer = fs.readFileSync(filePath);
        const pdfData = await pdfParse(pdfBuffer);
        
        // แบ่งเป็น Chunks
        const chunks = this.splitIntoChunks(pdfData.text, 500, 50);
        const fileName = path.basename(filePath, '.pdf');
        
        return chunks.map((chunk, i) => ({
            id: `pdf-${fileName}-${i}`,
            content: chunk,
            title: `${fileName} - ส่วนที่ ${i + 1}`,
            source: 'pdf',
            category: 'document'
        }));
    }

    // โหลดจาก Word Document
    async loadFromWord(filePath) {
        const result = await mammoth.extractRawText({ path: filePath });
        const chunks = this.splitIntoChunks(result.value, 500, 50);
        const fileName = path.basename(filePath, '.docx');
        
        return chunks.map((chunk, i) => ({
            id: `docx-${fileName}-${i}`,
            content: chunk,
            title: `${fileName} - ส่วนที่ ${i + 1}`,
            source: 'word_document',
            category: 'document'
        }));
    }

    // แบ่งข้อความเป็น Chunks
    splitIntoChunks(text, chunkSize = 500, overlap = 50) {
        const chunks = [];
        const sentences = text.split(/(?<=[.!?。])\s+/);
        
        let currentChunk = '';
        let currentSize = 0;
        
        for (const sentence of sentences) {
            if (currentSize + sentence.length > chunkSize && currentChunk) {
                chunks.push(currentChunk.trim());
                
                // Overlap: เริ่ม Chunk ใหม่ด้วยส่วนท้ายของ Chunk เก่า
                const words = currentChunk.split(' ');
                const overlapWords = words.slice(-Math.floor(overlap / 5));
                currentChunk = overlapWords.join(' ') + ' ' + sentence;
                currentSize = currentChunk.length;
            } else {
                currentChunk += (currentChunk ? ' ' : '') + sentence;
                currentSize += sentence.length;
            }
        }
        
        if (currentChunk.trim()) {
            chunks.push(currentChunk.trim());
        }
        
        return chunks.filter(c => c.length > 20);
    }

    // โหลดทุกแหล่งข้อมูล
    async loadAll(config = {}) {
        const allDocuments = [];
        
        // จาก Database FAQ
        const dbDocs = await this.loadFromDatabase();
        allDocuments.push(...dbDocs);
        console.log(`Loaded ${dbDocs.length} FAQ from database`);
        
        // จาก JSON Files
        if (config.jsonFiles) {
            for (const file of config.jsonFiles) {
                const docs = await this.loadFromJSON(file);
                allDocuments.push(...docs);
                console.log(`Loaded ${docs.length} documents from ${file}`);
            }
        }
        
        // จาก PDFs
        if (config.pdfFiles) {
            for (const file of config.pdfFiles) {
                const docs = await this.loadFromPDF(file);
                allDocuments.push(...docs);
                console.log(`Loaded ${docs.length} chunks from ${file}`);
            }
        }
        
        // อัพโหลดทั้งหมดไป Vector DB
        await this.vectorDB.addDocuments(allDocuments);
        console.log(`Total documents loaded: ${allDocuments.length}`);
        
        return allDocuments.length;
    }
}

module.exports = FAQLoader;
```

## 6. Ollama สำหรับ Local Model Hosting

```javascript
// src/ai/ollama-client.js
const axios = require('axios');

class OllamaClient {
    constructor() {
        this.baseUrl = process.env.OLLAMA_URL || 'http://localhost:11434';
        this.defaultModel = process.env.OLLAMA_MODEL || 'typhoon2-8b-instruct';
        
        this.client = axios.create({
            baseURL: this.baseUrl,
            timeout: 60000
        });
    }

    // สร้าง Chat Completion
    async chat(messages, options = {}) {
        const response = await this.client.post('/api/chat', {
            model: options.model || this.defaultModel,
            messages,
            stream: false,
            options: {
                temperature: options.temperature || 0.7,
                top_p: options.topP || 0.9,
                num_predict: options.maxTokens || 500,
                num_ctx: 4096
            }
        });
        
        return {
            content: response.data.message.content,
            model: response.data.model,
            tokens: {
                prompt_tokens: response.data.prompt_eval_count,
                completion_tokens: response.data.eval_count,
                total_tokens: (response.data.prompt_eval_count || 0) + (response.data.eval_count || 0)
            }
        };
    }

    // สร้าง Embedding (สำหรับ RAG)
    async createEmbedding(text) {
        const response = await this.client.post('/api/embeddings', {
            model: 'nomic-embed-text',
            prompt: text
        });
        
        return response.data.embedding;
    }

    // Stream Chat
    async* chatStream(messages, options = {}) {
        const response = await this.client.post('/api/chat', {
            model: options.model || this.defaultModel,
            messages,
            stream: true,
            options: {
                temperature: options.temperature || 0.7,
                num_predict: options.maxTokens || 500
            }
        }, {
            responseType: 'stream'
        });
        
        for await (const chunk of response.data) {
            const lines = chunk.toString().split('\n').filter(l => l.trim());
            for (const line of lines) {
                try {
                    const data = JSON.parse(line);
                    if (data.message?.content) {
                        yield data.message.content;
                    }
                } catch (e) { /* ข้ามบรรทัดที่ parse ไม่ได้ */ }
            }
        }
    }

    // ตรวจสอบ Models ที่มี
    async listModels() {
        const response = await this.client.get('/api/tags');
        return response.data.models;
    }

    // Pull Model
    async pullModel(modelName) {
        console.log(`Pulling model: ${modelName}`);
        const response = await this.client.post('/api/pull', {
            name: modelName,
            stream: true
        }, {
            responseType: 'stream',
            timeout: 600000 // 10 นาที
        });
        
        return new Promise((resolve, reject) => {
            response.data.on('data', chunk => {
                const data = JSON.parse(chunk.toString());
                if (data.status) console.log(`  ${data.status}`);
            });
            response.data.on('end', resolve);
            response.data.on('error', reject);
        });
    }

    // Health Check
    async isAvailable() {
        try {
            await this.client.get('/api/version', { timeout: 3000 });
            return true;
        } catch {
            return false;
        }
    }
}

module.exports = new OllamaClient();
```

### Docker Compose สำหรับ Ollama

```yaml
# docker-compose.ollama.yml
version: '3.8'

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_ORIGINS=*
    # GPU Support (uncomment ถ้ามี NVIDIA GPU)
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: 1
    #           capabilities: [gpu]
    restart: unless-stopped
    
  # Web UI สำหรับ Ollama
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    depends_on:
      - ollama
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes:
      - open_webui_data:/app/backend/data
    restart: unless-stopped

volumes:
  ollama_data:
  open_webui_data:
```

```bash
# ติดตั้ง Ollama และ pull Models
docker-compose -f docker-compose.ollama.yml up -d

# Pull Thai-capable models
docker exec ollama ollama pull typhoon2-8b-instruct
docker exec ollama ollama pull llama3.1:8b
docker exec ollama ollama pull nomic-embed-text  # สำหรับ Embeddings

# ทดสอบ
curl http://localhost:11434/api/generate -d '{
  "model": "typhoon2-8b-instruct",
  "prompt": "สวัสดี คุณช่วยอธิบายสินค้าของเราได้ไหม?"
}'
```

## 7. OpenAI Assistant API Integration

```javascript
// src/ai/openai-assistant.js
const { OpenAI } = require('openai');
const db = require('../database/mysql');

class OpenAIAssistantManager {
    constructor() {
        this.openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
        this.assistantId = process.env.OPENAI_ASSISTANT_ID;
        this.threadCache = new Map(); // lineUserId -> threadId
    }

    // สร้าง Assistant ใหม่
    async createAssistant(config = {}) {
        const assistant = await this.openai.beta.assistants.create({
            name: config.name || 'LINE Bot Customer Service',
            instructions: config.instructions || `
คุณเป็น Customer Service Agent สำหรับ LINE Bot
คุณตอบคำถามเป็นภาษาไทยอย่างสุภาพ
ถ้าไม่รู้คำตอบ ให้บอกว่าจะส่งต่อให้เจ้าหน้าที่

สิ่งที่คุณทำได้:
1. ตอบคำถามเกี่ยวกับสินค้าและบริการ
2. ช่วยเรื่องการสั่งซื้อ
3. แก้ไขปัญหาเบื้องต้น
            `.trim(),
            model: config.model || 'gpt-4o-mini',
            tools: [
                { type: 'file_search' },
                {
                    type: 'function',
                    function: {
                        name: 'get_product_info',
                        description: 'ดึงข้อมูลสินค้าจาก Database',
                        parameters: {
                            type: 'object',
                            properties: {
                                product_name: {
                                    type: 'string',
                                    description: 'ชื่อสินค้าที่ต้องการค้นหา'
                                }
                            },
                            required: ['product_name']
                        }
                    }
                },
                {
                    type: 'function',
                    function: {
                        name: 'check_order_status',
                        description: 'ตรวจสอบสถานะคำสั่งซื้อ',
                        parameters: {
                            type: 'object',
                            properties: {
                                order_id: {
                                    type: 'string',
                                    description: 'หมายเลขคำสั่งซื้อ'
                                }
                            },
                            required: ['order_id']
                        }
                    }
                }
            ]
        });
        
        console.log(`Created assistant: ${assistant.id}`);
        return assistant.id;
    }

    // ดึงหรือสร้าง Thread สำหรับ User
    async getOrCreateThread(lineUserId) {
        // ตรวจสอบ Cache ก่อน
        if (this.threadCache.has(lineUserId)) {
            return this.threadCache.get(lineUserId);
        }
        
        // ตรวจสอบ Database
        const stored = await db.query(
            'SELECT thread_id FROM ai_threads WHERE line_user_id = ? AND is_active = 1',
            [lineUserId]
        );
        
        if (stored.length > 0) {
            this.threadCache.set(lineUserId, stored[0].thread_id);
            return stored[0].thread_id;
        }
        
        // สร้าง Thread ใหม่
        const thread = await this.openai.beta.threads.create();
        
        await db.query(`
            INSERT INTO ai_threads (id, line_user_id, thread_id, is_active, created_at)
            VALUES (UUID(), ?, ?, 1, NOW())
            ON DUPLICATE KEY UPDATE thread_id = ?, is_active = 1, updated_at = NOW()
        `, [lineUserId, thread.id, thread.id]);
        
        this.threadCache.set(lineUserId, thread.id);
        return thread.id;
    }

    // ส่งข้อความและรับการตอบ
    async chat(lineUserId, userMessage) {
        const threadId = await this.getOrCreateThread(lineUserId);
        
        // เพิ่มข้อความของ User
        await this.openai.beta.threads.messages.create(threadId, {
            role: 'user',
            content: userMessage
        });
        
        // รัน Assistant
        let run = await this.openai.beta.threads.runs.create(threadId, {
            assistant_id: this.assistantId
        });
        
        // รอจนกว่าจะเสร็จ
        run = await this.waitForCompletion(threadId, run.id);
        
        // ดึงข้อความตอบกลับ
        const messages = await this.openai.beta.threads.messages.list(threadId);
        const lastMessage = messages.data.find(m => m.role === 'assistant');
        
        if (!lastMessage) throw new Error('No response from assistant');
        
        const responseText = lastMessage.content
            .filter(c => c.type === 'text')
            .map(c => c.text.value)
            .join('\n');
        
        return {
            response: responseText,
            threadId,
            runId: run.id,
            usage: run.usage
        };
    }

    // รอให้ Run เสร็จ
    async waitForCompletion(threadId, runId, maxAttempts = 30) {
        for (let i = 0; i < maxAttempts; i++) {
            const run = await this.openai.beta.threads.runs.retrieve(threadId, runId);
            
            if (run.status === 'completed') return run;
            
            if (run.status === 'requires_action') {
                // จัดการ Function Calls
                return await this.handleFunctionCalls(threadId, run);
            }
            
            if (['failed', 'cancelled', 'expired'].includes(run.status)) {
                throw new Error(`Run ${run.status}: ${run.last_error?.message}`);
            }
            
            await new Promise(resolve => setTimeout(resolve, 1000));
        }
        
        throw new Error('Run timeout');
    }

    // จัดการ Function Calls จาก Assistant
    async handleFunctionCalls(threadId, run) {
        const toolCalls = run.required_action.submit_tool_outputs.tool_calls;
        const toolOutputs = [];
        
        for (const toolCall of toolCalls) {
            const args = JSON.parse(toolCall.function.arguments);
            let output;
            
            switch (toolCall.function.name) {
                case 'get_product_info':
                    output = await this.getProductInfo(args.product_name);
                    break;
                case 'check_order_status':
                    output = await this.checkOrderStatus(args.order_id);
                    break;
                default:
                    output = { error: 'Function not found' };
            }
            
            toolOutputs.push({
                tool_call_id: toolCall.id,
                output: JSON.stringify(output)
            });
        }
        
        // ส่ง Tool Outputs กลับ
        const updatedRun = await this.openai.beta.threads.runs.submitToolOutputs(
            threadId,
            run.id,
            { tool_outputs: toolOutputs }
        );
        
        return await this.waitForCompletion(threadId, updatedRun.id);
    }

    // Function: ดึงข้อมูลสินค้า
    async getProductInfo(productName) {
        const products = await db.query(
            'SELECT name, description, price, stock FROM products WHERE name LIKE ? LIMIT 5',
            [`%${productName}%`]
        );
        
        return products.length > 0 ? products : { message: 'ไม่พบสินค้าที่ค้นหา' };
    }

    // Function: ตรวจสอบ Order
    async checkOrderStatus(orderId) {
        const orders = await db.query(
            'SELECT id, status, total_amount, created_at FROM orders WHERE id = ?',
            [orderId]
        );
        
        return orders.length > 0 ? orders[0] : { message: 'ไม่พบ Order นี้' };
    }
}

module.exports = new OpenAIAssistantManager();
```

## 8. Model Evaluation และ Comparison

```javascript
// src/ai/model-evaluator.js
const { OpenAI } = require('openai');
const OllamaClient = require('./ollama-client');
const RAGSystem = require('./rag-system');
const db = require('../database/mysql');

class ModelEvaluator {
    constructor() {
        this.openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
        this.ollama = OllamaClient;
        
        // Test Questions (ภาษาไทย)
        this.testQuestions = [
            {
                question: 'สินค้าของคุณมีการรับประกันนานเท่าไหร่?',
                expectedKeywords: ['รับประกัน', 'ปี', 'เดือน'],
                category: 'product_warranty'
            },
            {
                question: 'ฉันต้องการยกเลิกคำสั่งซื้อ ทำอย่างไร?',
                expectedKeywords: ['ยกเลิก', 'ติดต่อ', 'โทร', 'LINE'],
                category: 'order_cancellation'
            },
            {
                question: 'มีบริการส่งด่วนไหม? ค่าใช้จ่ายเท่าไหร่?',
                expectedKeywords: ['ส่งด่วน', 'บาท', 'วัน', 'ชั่วโมง'],
                category: 'shipping'
            }
        ];
    }

    // ทดสอบ Model หลายๆ ตัว
    async compareModels(models = ['gpt-4o-mini', 'gpt-3.5-turbo', 'ollama:typhoon2-8b-instruct']) {
        const results = {};
        
        for (const model of models) {
            console.log(`\nEvaluating ${model}...`);
            results[model] = await this.evaluateModel(model);
        }
        
        // สรุปผล
        this.printComparisonTable(results);
        
        // บันทึกผลใน Database
        await this.saveEvaluationResults(results);
        
        return results;
    }

    // ประเมิน Model
    async evaluateModel(modelName) {
        const metrics = {
            model: modelName,
            totalQuestions: this.testQuestions.length,
            correctAnswers: 0,
            avgResponseTime: 0,
            avgTokens: 0,
            totalCost: 0,
            responses: []
        };
        
        let totalTime = 0;
        let totalTokens = 0;
        
        for (const testCase of this.testQuestions) {
            const startTime = Date.now();
            
            try {
                let response;
                
                if (modelName.startsWith('ollama:')) {
                    const ollamaModel = modelName.replace('ollama:', '');
                    response = await this.ollama.chat([
                        { role: 'system', content: 'คุณเป็น Customer Service ตอบภาษาไทย' },
                        { role: 'user', content: testCase.question }
                    ], { model: ollamaModel });
                } else {
                    const completion = await this.openai.chat.completions.create({
                        model: modelName,
                        messages: [
                            { role: 'system', content: 'คุณเป็น Customer Service ตอบภาษาไทย' },
                            { role: 'user', content: testCase.question }
                        ],
                        max_tokens: 300
                    });
                    
                    response = {
                        content: completion.choices[0].message.content,
                        tokens: completion.usage
                    };
                    
                    // คำนวณราคา
                    metrics.totalCost += this.calculateCost(modelName, completion.usage);
                }
                
                const responseTime = Date.now() - startTime;
                totalTime += responseTime;
                totalTokens += response.tokens?.total_tokens || 0;
                
                // ตรวจสอบความถูกต้อง
                const isCorrect = this.checkResponse(
                    response.content, 
                    testCase.expectedKeywords
                );
                
                if (isCorrect) metrics.correctAnswers++;
                
                metrics.responses.push({
                    question: testCase.question,
                    answer: response.content,
                    responseTime,
                    isCorrect,
                    category: testCase.category
                });
                
            } catch (error) {
                console.error(`Error with ${modelName}:`, error.message);
                metrics.responses.push({
                    question: testCase.question,
                    error: error.message,
                    isCorrect: false
                });
            }
        }
        
        metrics.avgResponseTime = Math.round(totalTime / this.testQuestions.length);
        metrics.avgTokens = Math.round(totalTokens / this.testQuestions.length);
        metrics.accuracy = (metrics.correctAnswers / metrics.totalQuestions * 100).toFixed(1);
        
        return metrics;
    }

    // ตรวจสอบ Response
    checkResponse(response, expectedKeywords) {
        if (!response) return false;
        
        const lowerResponse = response.toLowerCase();
        const matchedKeywords = expectedKeywords.filter(kw => 
            lowerResponse.includes(kw.toLowerCase())
        );
        
        return matchedKeywords.length >= Math.ceil(expectedKeywords.length / 2);
    }

    // คำนวณค่าใช้จ่าย
    calculateCost(model, usage) {
        const pricing = {
            'gpt-4o': { input: 0.0025, output: 0.01 },
            'gpt-4o-mini': { input: 0.00015, output: 0.0006 },
            'gpt-3.5-turbo': { input: 0.0005, output: 0.0015 }
        };
        
        const p = pricing[model];
        if (!p || !usage) return 0;
        
        return (usage.prompt_tokens / 1000 * p.input) + 
               (usage.completion_tokens / 1000 * p.output);
    }

    // แสดงตารางเปรียบเทียบ
    printComparisonTable(results) {
        console.log('\n=== Model Comparison Results ===');
        console.log('Model'.padEnd(30), 'Accuracy'.padEnd(12), 'Avg Time'.padEnd(12), 'Cost/Q'.padEnd(12));
        console.log('-'.repeat(70));
        
        for (const [model, metrics] of Object.entries(results)) {
            const costPerQ = (metrics.totalCost / metrics.totalQuestions).toFixed(4);
            console.log(
                model.padEnd(30),
                `${metrics.accuracy}%`.padEnd(12),
                `${metrics.avgResponseTime}ms`.padEnd(12),
                `$${costPerQ}`.padEnd(12)
            );
        }
    }

    // บันทึกผลลัพธ์
    async saveEvaluationResults(results) {
        for (const [model, metrics] of Object.entries(results)) {
            await db.query(`
                INSERT INTO model_evaluations (
                    id, model_name, accuracy, avg_response_time_ms, 
                    avg_tokens, total_cost_usd, evaluated_at
                ) VALUES (UUID(), ?, ?, ?, ?, ?, NOW())
            `, [
                model,
                metrics.accuracy,
                metrics.avgResponseTime,
                metrics.avgTokens,
                metrics.totalCost
            ]);
        }
    }
}

module.exports = new ModelEvaluator();
```

## 9. Cost Analysis: GPT-4 vs Fine-tuned vs Local Model

```javascript
// src/ai/cost-analyzer.js
const db = require('../database/mysql');

class CostAnalyzer {
    // วิเคราะห์ค่าใช้จ่ายย้อนหลัง
    async analyzeHistoricalCost(days = 30) {
        const data = await db.query(`
            SELECT 
                model,
                COUNT(*) as requests,
                SUM(total_tokens) as total_tokens,
                SUM(estimated_cost) as total_cost_usd,
                AVG(total_tokens) as avg_tokens_per_request,
                SUM(estimated_cost) * 34 as total_cost_thb
            FROM ai_usage_log
            WHERE occurred_at >= DATE_SUB(NOW(), INTERVAL ? DAY)
            GROUP BY model
            ORDER BY total_cost_usd DESC
        `, [days]);
        
        return data;
    }

    // คำนวณ Break-even Analysis
    calculateBreakEven(config = {}) {
        const {
            dailyRequests = 1000,
            avgTokensPerRequest = 500
        } = config;
        
        const monthlyRequests = dailyRequests * 30;
        const monthlyTokens = monthlyRequests * avgTokensPerRequest;
        
        const models = {
            'GPT-4o': {
                inputCostPer1k: 0.0025,
                outputCostPer1k: 0.01,
                setupCost: 0,
                monthlyCost: 0
            },
            'GPT-4o-mini': {
                inputCostPer1k: 0.00015,
                outputCostPer1k: 0.0006,
                setupCost: 0,
                monthlyCost: 0
            },
            'Fine-tuned GPT-3.5': {
                inputCostPer1k: 0.003,
                outputCostPer1k: 0.006,
                setupCost: 50,      // ค่า Fine-tuning
                monthlyCost: 0
            },
            'Ollama (Local GPU)': {
                inputCostPer1k: 0,
                outputCostPer1k: 0,
                setupCost: 0,
                monthlyCost: 150,   // ค่าเครื่อง/ค่าไฟ ต่อเดือน
                note: 'ต้องการ GPU Server'
            },
            'Ollama (Cloud GPU)': {
                inputCostPer1k: 0,
                outputCostPer1k: 0,
                setupCost: 0,
                monthlyCost: 300,   // ค่า Cloud GPU (เช่น RunPod)
                note: 'RunPod/Vast.ai'
            }
        };
        
        // คำนวณค่าใช้จ่าย
        const results = {};
        for (const [name, model] of Object.entries(models)) {
            const tokenCostUSD = (monthlyTokens / 1000) * 
                ((model.inputCostPer1k + model.outputCostPer1k) / 2);
            
            const totalMonthlyUSD = tokenCostUSD + model.monthlyCost;
            const totalMonthlyTHB = totalMonthlyUSD * 34;
            const costPerRequestTHB = (totalMonthlyTHB / monthlyRequests).toFixed(4);
            
            results[name] = {
                monthlyRequestCostUSD: tokenCostUSD.toFixed(2),
                monthlyServerCostUSD: model.monthlyCost.toFixed(2),
                totalMonthlyCostUSD: totalMonthlyUSD.toFixed(2),
                totalMonthlyCostTHB: Math.round(totalMonthlyTHB).toLocaleString(),
                costPerRequestTHB,
                setupCostUSD: model.setupCost,
                note: model.note || ''
            };
        }
        
        return {
            assumptions: {
                dailyRequests,
                monthlyRequests,
                avgTokensPerRequest
            },
            models: results
        };
    }

    // แสดงรายงานค่าใช้จ่าย
    async generateCostReport() {
        const historical = await this.analyzeHistoricalCost(30);
        const breakEven = this.calculateBreakEven({ dailyRequests: 2000 });
        
        console.log('\n=== AI Cost Report (30 วันที่ผ่านมา) ===');
        console.log('\nค่าใช้จ่ายจริง:');
        historical.forEach(model => {
            console.log(`  ${model.model}:`);
            console.log(`    Requests: ${model.requests.toLocaleString()}`);
            console.log(`    Tokens: ${model.total_tokens.toLocaleString()}`);
            console.log(`    Cost: $${model.total_cost_usd.toFixed(2)} (฿${Math.round(model.total_cost_thb).toLocaleString()})`);
        });
        
        console.log('\n=== Break-Even Analysis (2,000 requests/วัน) ===');
        Object.entries(breakEven.models).forEach(([name, data]) => {
            console.log(`\n${name}:`);
            console.log(`  ค่าใช้จ่ายต่อเดือน: ฿${data.totalMonthlyCostTHB}`);
            console.log(`  ค่าใช้จ่ายต่อ Request: ฿${data.costPerRequestTHB}`);
            if (data.note) console.log(`  หมายเหตุ: ${data.note}`);
        });
        
        return { historical, breakEven };
    }
}

module.exports = new CostAnalyzer();
```

## 10. Complete RAG + LINE Bot System

```javascript
// src/ai/line-ai-handler.js
const RAGSystem = require('./rag-system');
const OllamaClient = require('./ollama-client');
const OpenAIAssistantManager = require('./openai-assistant');
const db = require('../database/mysql');
const lineClient = require('../line/client');

class LineAIHandler {
    constructor() {
        this.ragSystem = RAGSystem;
        this.ollama = OllamaClient;
        this.assistant = OpenAIAssistantManager;
        
        // เลือก AI Engine
        this.engine = process.env.AI_ENGINE || 'rag'; // rag, ollama, assistant
        
        // Cache สำหรับ Conversation History
        this.conversationCache = new Map();
    }

    // จัดการข้อความจาก LINE
    async handleMessage(lineUserId, userMessage, replyToken) {
        const startTime = Date.now();
        
        try {
            // ดึง Conversation History
            const history = this.getConversationHistory(lineUserId);
            
            // ดึงข้อมูล User
            const contact = await db.query(
                'SELECT * FROM contacts WHERE line_user_id = ?',
                [lineUserId]
            );
            const userProfile = contact.length > 0 ? contact[0] : {};
            
            // เรียก AI ตาม Engine
            let result;
            
            switch (this.engine) {
                case 'rag':
                    result = await this.ragSystem.generateResponse(
                        userMessage, history, userProfile
                    );
                    break;
                    
                case 'ollama':
                    const isOllamaAvailable = await this.ollama.isAvailable();
                    
                    if (isOllamaAvailable) {
                        const ollamaMessages = [
                            {
                                role: 'system',
                                content: `คุณเป็น AI Assistant สำหรับ ${process.env.COMPANY_NAME || 'บริษัทเรา'} ตอบภาษาไทย`
                            },
                            ...history,
                            { role: 'user', content: userMessage }
                        ];
                        
                        const response = await this.ollama.chat(ollamaMessages);
                        result = { answer: response.content, model: response.model };
                    } else {
                        // Fallback ไป RAG
                        result = await this.ragSystem.generateResponse(
                            userMessage, history, userProfile
                        );
                    }
                    break;
                    
                case 'assistant':
                    const assistantResult = await this.assistant.chat(lineUserId, userMessage);
                    result = { answer: assistantResult.response, model: 'openai-assistant' };
                    break;
                    
                default:
                    throw new Error(`Unknown AI engine: ${this.engine}`);
            }
            
            const responseTime = Date.now() - startTime;
            
            // อัพเดท Conversation History
            this.updateConversationHistory(lineUserId, userMessage, result.answer);
            
            // ตรวจสอบว่าควร Escalate ไป Human
            const shouldEscalate = this.shouldEscalateToHuman(result.answer, userMessage);
            
            if (shouldEscalate) {
                await this.escalateToHuman(lineUserId, userMessage, replyToken);
            } else {
                // ส่งคำตอบ
                await lineClient.replyMessage(replyToken, {
                    type: 'text',
                    text: result.answer
                });
            }
            
            // บันทึก Log
            await this.logInteraction({
                lineUserId,
                userMessage,
                botResponse: result.answer,
                model: result.model,
                responseTime,
                tokensUsed: result.tokens?.total_tokens,
                shouldEscalate
            });
            
        } catch (error) {
            console.error('AI Handler error:', error);
            
            // ส่งข้อความ Error
            await lineClient.replyMessage(replyToken, {
                type: 'text',
                text: 'ขออภัย เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง หรือติดต่อเจ้าหน้าที่'
            });
        }
    }

    // ดึง Conversation History
    getConversationHistory(lineUserId, maxTurns = 5) {
        const history = this.conversationCache.get(lineUserId) || [];
        return history.slice(-maxTurns * 2); // รวม User + Assistant messages
    }

    // อัพเดท History
    updateConversationHistory(lineUserId, userMessage, assistantResponse) {
        const history = this.conversationCache.get(lineUserId) || [];
        
        history.push(
            { role: 'user', content: userMessage },
            { role: 'assistant', content: assistantResponse }
        );
        
        // เก็บแค่ 10 รอบล่าสุด
        while (history.length > 20) {
            history.splice(0, 2);
        }
        
        this.conversationCache.set(lineUserId, history);
        
        // TTL: ลบหลัง 30 นาที
        setTimeout(() => {
            this.conversationCache.delete(lineUserId);
        }, 30 * 60 * 1000);
    }

    // ตรวจสอบว่าควร Escalate ไป Human
    shouldEscalateToHuman(botResponse, userMessage) {
        const escalationKeywords = [
            'คุย', 'เจ้าหน้าที่', 'คน', 'พนักงาน',
            'โกรธ', 'ไม่พอใจ', 'แย่มาก', 'ห่วยแตก',
            'ขอคืนเงิน', 'refund', 'ฟ้อง', 'แจ้งความ'
        ];
        
        const lowerMsg = userMessage.toLowerCase();
        const containsEscalation = escalationKeywords.some(kw => lowerMsg.includes(kw));
        
        const botIsUnsure = botResponse.toLowerCase().includes('ไม่แน่ใจ') ||
                            botResponse.toLowerCase().includes('ส่งต่อ') ||
                            botResponse.toLowerCase().includes('เจ้าหน้าที่');
        
        return containsEscalation || botIsUnsure;
    }

    // Escalate ไป Human Agent
    async escalateToHuman(lineUserId, userMessage, replyToken) {
        // บันทึกใน Queue สำหรับ Human
        await db.query(`
            INSERT INTO human_handover_queue (id, line_user_id, last_message, created_at)
            VALUES (UUID(), ?, ?, NOW())
            ON DUPLICATE KEY UPDATE last_message = ?, updated_at = NOW()
        `, [lineUserId, userMessage, userMessage]);
        
        // แจ้งเตือน Admin
        const adminIds = process.env.ADMIN_LINE_USER_IDS?.split(',') || [];
        for (const adminId of adminIds) {
            await lineClient.pushMessage(adminId.trim(), {
                type: 'text',
                text: `⚠️ ต้องการความช่วยเหลือจาก Human Agent\nUser: ${lineUserId}\nข้อความ: ${userMessage}`
            });
        }
        
        // ตอบ User
        await lineClient.replyMessage(replyToken, {
            type: 'text',
            text: 'กำลังส่งต่อให้เจ้าหน้าที่ดูแลคุณ กรุณารอสักครู่นะครับ/ค่ะ 🙏'
        });
    }

    // บันทึก Interaction Log
    async logInteraction(data) {
        await db.query(`
            INSERT INTO ai_interaction_logs (
                id, line_user_id, user_message, bot_response,
                model_used, response_time_ms, tokens_used, 
                escalated_to_human, occurred_at
            ) VALUES (UUID(), ?, ?, ?, ?, ?, ?, ?, NOW())
        `, [
            data.lineUserId,
            data.userMessage,
            data.botResponse,
            data.model,
            data.responseTime,
            data.tokensUsed || 0,
            data.shouldEscalate ? 1 : 0
        ]);
    }
}

module.exports = new LineAIHandler();
```

## 11. Docker Setup สำหรับ Complete AI Stack

```yaml
# docker-compose.ai.yml
version: '3.8'

services:
  # ChromaDB Vector Database
  chromadb:
    image: chromadb/chroma:latest
    container_name: chromadb
    ports:
      - "8000:8000"
    volumes:
      - chroma_data:/chroma/.chroma/index
    environment:
      - ALLOW_RESET=TRUE
      - CHROMA_SERVER_AUTH_CREDENTIALS_PROVIDER=chromadb.auth.token.TokenConfigServerAuthCredentialsProvider
      - CHROMA_SERVER_AUTH_TOKEN_TRANSPORT_HEADER=X-Chroma-Token
      - CHROMA_SERVER_AUTH_CREDENTIALS=${CHROMA_TOKEN:-admin-token}
    restart: unless-stopped
    
  # Ollama Local LLM
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    environment:
      - OLLAMA_KEEP_ALIVE=24h
      - OLLAMA_NUM_PARALLEL=2
    restart: unless-stopped
    
  # Redis สำหรับ Cache
  redis:
    image: redis:7-alpine
    container_name: redis_ai
    ports:
      - "6379:6379"
    command: redis-server --requirepass ${REDIS_PASSWORD:-password}
    volumes:
      - redis_data:/data
    restart: unless-stopped
    
  # LINE AI Bot
  line-ai-bot:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: line-ai-bot
    ports:
      - "3000:3000"
    environment:
      - LINE_CHANNEL_ACCESS_TOKEN=${LINE_CHANNEL_ACCESS_TOKEN}
      - LINE_CHANNEL_SECRET=${LINE_CHANNEL_SECRET}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - PINECONE_API_KEY=${PINECONE_API_KEY}
      - CHROMA_URL=http://chromadb:8000
      - OLLAMA_URL=http://ollama:11434
      - REDIS_URL=redis://:${REDIS_PASSWORD:-password}@redis:6379
      - AI_ENGINE=rag  # rag, ollama, assistant
      - CHAT_MODEL=gpt-4o-mini
    depends_on:
      - chromadb
      - ollama
      - redis
    restart: unless-stopped
    volumes:
      - ./knowledge:/app/knowledge

volumes:
  chroma_data:
  ollama_data:
  redis_data:
```

## 12. Database Tables สำหรับ AI

```sql
-- ตาราง knowledge_base
CREATE TABLE knowledge_base (
    id VARCHAR(36) PRIMARY KEY DEFAULT (UUID()),
    question TEXT NOT NULL,
    answer TEXT NOT NULL,
    category VARCHAR(100),
    tags JSON,
    priority INT DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    view_count INT DEFAULT 0,
    helpful_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ตาราง ai_usage_log
CREATE TABLE ai_usage_log (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    model VARCHAR(100) NOT NULL,
    prompt_tokens INT DEFAULT 0,
    completion_tokens INT DEFAULT 0,
    total_tokens INT DEFAULT 0,
    estimated_cost DECIMAL(10, 6) DEFAULT 0,
    occurred_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_model (model),
    INDEX idx_occurred_at (occurred_at)
);

-- ตาราง ai_threads (สำหรับ OpenAI Assistant)
CREATE TABLE ai_threads (
    id VARCHAR(36) PRIMARY KEY,
    line_user_id VARCHAR(100) UNIQUE,
    thread_id VARCHAR(100) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ตาราง ai_interaction_logs
CREATE TABLE ai_interaction_logs (
    id VARCHAR(36) PRIMARY KEY,
    line_user_id VARCHAR(100),
    user_message TEXT,
    bot_response TEXT,
    model_used VARCHAR(100),
    response_time_ms INT,
    tokens_used INT DEFAULT 0,
    escalated_to_human BOOLEAN DEFAULT FALSE,
    occurred_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_line_user_id (line_user_id),
    INDEX idx_occurred_at (occurred_at)
);

-- ตาราง model_evaluations
CREATE TABLE model_evaluations (
    id VARCHAR(36) PRIMARY KEY,
    model_name VARCHAR(100),
    accuracy DECIMAL(5, 2),
    avg_response_time_ms DECIMAL(10, 2),
    avg_tokens DECIMAL(10, 2),
    total_cost_usd DECIMAL(10, 4),
    evaluated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ตาราง human_handover_queue
CREATE TABLE human_handover_queue (
    id VARCHAR(36) PRIMARY KEY,
    line_user_id VARCHAR(100) NOT NULL,
    last_message TEXT,
    status ENUM('waiting', 'assigned', 'resolved') DEFAULT 'waiting',
    assigned_to VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

## สรุป

ในบทนี้เราสร้าง Custom AI System สำหรับ LINE Bot ที่ประกอบด้วย:

1. **Thai Language Models** - ภาพรวม Models ที่รองรับภาษาไทย
2. **Training Data Preparation** - การเตรียมข้อมูลสำหรับ Fine-tuning จากการสนทนาจริง
3. **RAG System** - ระบบค้นหาข้อมูลที่เกี่ยวข้องก่อนสร้างคำตอบ
4. **Vector Databases** - Pinecone (Cloud) และ Chroma (Local)
5. **Knowledge Base** - การโหลดข้อมูลจาก FAQ, PDF, และ Database
6. **Ollama** - การรัน Local LLM โดยไม่ต้องพึ่ง Cloud API
7. **OpenAI Assistant** - การใช้ Stateful Conversations ผ่าน Threads
8. **Model Evaluation** - การเปรียบเทียบประสิทธิภาพและค่าใช้จ่าย
9. **Cost Analysis** - การวิเคราะห์ ROI ของแต่ละ Model
10. **Complete System** - การรวมทุกส่วนเข้าด้วยกัน
