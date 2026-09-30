# Part 85: Custom AI Models สำหรับ LINE Bots

## บทนำ

การสร้าง Custom AI Model สำหรับ LINE Bot ช่วยให้ chatbot ตอบสนองได้ตรงกับ domain เฉพาะ ภาษาไทย และ context ของธุรกิจ บทนี้จะครอบคลุมการ Fine-tune LLMs, RAG Systems, และ Model Serving สำหรับ Production

---

## 1. Overview ของ Custom AI Approach

```
Custom AI Decision Tree:
┌─────────────────────────────────────────────────────────┐
│  ต้องการ AI ประเภทไหน?                                  │
│                                                         │
│  ① ใช้ API (GPT-4, Claude) ─── ง่าย, แพง, ไม่ private │
│  ② RAG + API ────────────── balance ดี สำหรับ knowledge│
│  ③ Fine-tuned Model ──────── domain-specific, ควบคุมได้ │
│  ④ Full Custom Model ─────── แพงมาก, เฉพาะองค์กรใหญ่   │
│                                                         │
│  แนะนำสำหรับ LINE OA:                                  │
│  ├── Customer Service Bot → RAG + API                   │
│  ├── Product Recommendation → ML Model                  │
│  ├── Thai Language NLP → Fine-tuned WangchanBERTa      │
│  └── Complex Conversations → Fine-tuned LLaMA/Mistral  │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Thai Language Models

### 2.1 Available Thai Language Models

```
Thai Language Models:

Model               Size    Use Case              Performance
──────────────────────────────────────────────────────────────
WangchanBERTa       178M    Classification/NER    ★★★★★
                            Sentiment Analysis    
──────────────────────────────────────────────────────────────
PhayaThaiBERT       110M    Text Classification   ★★★★☆
                            Token Classification  
──────────────────────────────────────────────────────────────
Typhoon-1.5B        1.5B    Thai Chatbot          ★★★★☆
(SCBX)                      Instruction Following 
──────────────────────────────────────────────────────────────
Typhoon-7B          7B      Complex Thai NLP      ★★★★★
                            Reasoning             
──────────────────────────────────────────────────────────────
SEA-LION-7B         7B      Southeast Asian NLP   ★★★★☆
(AI Singapore)              Thai + Multilingual   
──────────────────────────────────────────────────────────────
LLaMA-3-Thai-8B     8B      General Thai NLP      ★★★★☆
(Community)                 Fine-tuned on Thai    
```

### 2.2 WangchanBERTa สำหรับ Intent Classification

```python
# ml/thai_nlp/intent_classifier.py
from transformers import (
    AutoTokenizer,
    AutoModelForSequenceClassification,
    TrainingArguments,
    Trainer,
)
from datasets import Dataset
import torch
import numpy as np
import pandas as pd
from sklearn.metrics import classification_report
import mlflow

class ThaiIntentClassifier:
    """
    Intent Classification สำหรับ LINE Bot
    ใช้ WangchanBERTa ที่ fine-tune แล้ว
    """
    
    def __init__(self, model_name='airesearch/wangchanberta-base-att-spm-uncased'):
        self.model_name = model_name
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = None
        self.label2id = {}
        self.id2label = {}
    
    def prepare_dataset(self, texts, labels):
        """เตรียม dataset สำหรับ fine-tuning"""
        
        # Create label mappings
        unique_labels = sorted(set(labels))
        self.label2id = {label: i for i, label in enumerate(unique_labels)}
        self.id2label = {i: label for label, i in self.label2id.items()}
        
        # Tokenize
        def tokenize_function(examples):
            return self.tokenizer(
                examples['text'],
                padding='max_length',
                truncation=True,
                max_length=128,
            )
        
        dataset = Dataset.from_dict({
            'text': texts,
            'labels': [self.label2id[l] for l in labels],
        })
        
        return dataset.map(tokenize_function, batched=True)
    
    def fine_tune(self, train_texts, train_labels, eval_texts, eval_labels):
        """
        Fine-tune WangchanBERTa สำหรับ Intent Classification
        
        Intent categories:
        - greeting: สวัสดี, ดีครับ
        - product_inquiry: สอบถามสินค้า, ราคาเท่าไหร่
        - order_status: เช็คออร์เดอร์, ส่งถึงไหนแล้ว
        - complaint: ไม่พอใจ, สินค้าเสีย
        - support: ช่วยด้วย, ติดต่อเจ้าหน้าที่
        - farewell: ขอบคุณ, บาย
        - other: อื่นๆ
        """
        
        with mlflow.start_run(run_name="thai-intent-classifier"):
            n_classes = len(set(train_labels))
            
            self.model = AutoModelForSequenceClassification.from_pretrained(
                self.model_name,
                num_labels=n_classes,
                id2label=self.id2label,
                label2id=self.label2id,
            )
            
            train_dataset = self.prepare_dataset(train_texts, train_labels)
            eval_dataset = self.prepare_dataset(eval_texts, eval_labels)
            
            training_args = TrainingArguments(
                output_dir='./models/thai-intent-classifier',
                num_train_epochs=5,
                per_device_train_batch_size=16,
                per_device_eval_batch_size=32,
                warmup_steps=100,
                weight_decay=0.01,
                logging_dir='./logs',
                logging_steps=50,
                evaluation_strategy='epoch',
                save_strategy='epoch',
                load_best_model_at_end=True,
                metric_for_best_model='f1',
                report_to='mlflow',
                learning_rate=2e-5,
                fp16=torch.cuda.is_available(),
            )
            
            def compute_metrics(eval_pred):
                logits, labels = eval_pred
                predictions = np.argmax(logits, axis=-1)
                
                report = classification_report(
                    labels, predictions,
                    target_names=list(self.id2label.values()),
                    output_dict=True,
                )
                
                return {
                    'accuracy': report['accuracy'],
                    'f1': report['weighted avg']['f1-score'],
                    'precision': report['weighted avg']['precision'],
                    'recall': report['weighted avg']['recall'],
                }
            
            trainer = Trainer(
                model=self.model,
                args=training_args,
                train_dataset=train_dataset,
                eval_dataset=eval_dataset,
                compute_metrics=compute_metrics,
            )
            
            trainer.train()
            
            # Evaluate
            results = trainer.evaluate()
            print(f"\nFinal Results: {results}")
            
            # Save
            self.model.save_pretrained('./models/thai-intent-classifier/final')
            self.tokenizer.save_pretrained('./models/thai-intent-classifier/final')
            
            return results
    
    def predict(self, text, return_all=False):
        """ทำนาย intent จาก text ภาษาไทย"""
        
        if not self.model:
            raise ValueError("Model not loaded. Train or load a model first.")
        
        inputs = self.tokenizer(
            text,
            return_tensors='pt',
            padding=True,
            truncation=True,
            max_length=128,
        )
        
        with torch.no_grad():
            outputs = self.model(**inputs)
            logits = outputs.logits
            probabilities = torch.softmax(logits, dim=-1)[0]
        
        predicted_id = probabilities.argmax().item()
        predicted_label = self.id2label[predicted_id]
        confidence = probabilities[predicted_id].item()
        
        if return_all:
            return {
                'intent': predicted_label,
                'confidence': confidence,
                'all_intents': {
                    self.id2label[i]: float(p)
                    for i, p in enumerate(probabilities)
                },
            }
        
        return predicted_label, confidence


# Sample training data สำหรับ LINE Bot
THAI_INTENT_TRAINING_DATA = [
    # Greeting
    ("สวัสดีครับ", "greeting"),
    ("ดีครับ", "greeting"),
    ("หวัดดี", "greeting"),
    ("สวัสดีค่ะ พอดีอยากถามเรื่อง", "greeting"),
    
    # Product inquiry
    ("ราคาสินค้าตัวนี้เท่าไหร่ครับ", "product_inquiry"),
    ("มีสต็อกไหมครับ", "product_inquiry"),
    ("อยากได้ข้อมูลเพิ่มเติมเกี่ยวกับสินค้า", "product_inquiry"),
    ("รุ่นนี้มีสีอะไรบ้าง", "product_inquiry"),
    
    # Order status
    ("ออร์เดอร์ไปแล้วส่งถึงไหนแล้วครับ", "order_status"),
    ("เช็คสถานะพัสดุได้ไหม", "order_status"),
    ("สั่งซื้อไปแล้ว 3 วัน ยังไม่ได้รับเลย", "order_status"),
    ("ติดตามการจัดส่งได้อย่างไร", "order_status"),
    
    # Complaint
    ("สินค้าที่ได้รับมาเสียหายครับ", "complaint"),
    ("ได้รับสินค้าผิด", "complaint"),
    ("คุณภาพไม่ดีเลย", "complaint"),
    ("ไม่พอใจการบริการมาก", "complaint"),
    
    # Support
    ("ช่วยด้วยครับ ไม่รู้จะทำยังไง", "support"),
    ("อยากคุยกับเจ้าหน้าที่", "support"),
    ("ติดต่อ admin ได้อย่างไร", "support"),
    
    # Farewell
    ("ขอบคุณมากครับ", "farewell"),
    ("ขอบคุณนะคะ", "farewell"),
    ("โอเคครับ บาย", "farewell"),
    
    # Other
    ("555", "other"),
    ("อยากได้ส่วนลด", "other"),
]
```

---

## 3. Fine-tuning LLMs

### 3.1 Fine-tuning Typhoon สำหรับ Customer Service

```python
# ml/finetune/typhoon_finetuner.py
import torch
from transformers import AutoTokenizer, AutoModelForCausalLM, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, TaskType, prepare_model_for_kbit_training
from trl import SFTTrainer, SFTConfig
from datasets import Dataset
import json

class TyphoonFineTuner:
    """
    Fine-tune Typhoon-7B สำหรับ LINE Bot Customer Service
    ใช้ QLoRA (4-bit quantization + LoRA)
    """
    
    def __init__(self, 
                 base_model="scb10x/typhoon-7b",
                 output_dir="./models/typhoon-linebot"):
        self.base_model = base_model
        self.output_dir = output_dir
    
    def load_model(self):
        """Load model ด้วย 4-bit quantization เพื่อประหยัด memory"""
        
        # BnB config สำหรับ 4-bit quantization
        bnb_config = BitsAndBytesConfig(
            load_in_4bit=True,
            bnb_4bit_use_double_quant=True,
            bnb_4bit_quant_type="nf4",
            bnb_4bit_compute_dtype=torch.bfloat16,
        )
        
        print(f"Loading {self.base_model}...")
        
        self.tokenizer = AutoTokenizer.from_pretrained(self.base_model)
        self.tokenizer.pad_token = self.tokenizer.eos_token
        
        self.model = AutoModelForCausalLM.from_pretrained(
            self.base_model,
            quantization_config=bnb_config,
            device_map="auto",
            torch_dtype=torch.bfloat16,
        )
        
        # Prepare for k-bit training
        self.model = prepare_model_for_kbit_training(self.model)
        
        return self
    
    def add_lora_adapters(self):
        """เพิ่ม LoRA adapters"""
        
        lora_config = LoraConfig(
            task_type=TaskType.CAUSAL_LM,
            r=16,               # Rank
            lora_alpha=32,      # Alpha
            target_modules=[    # Modules to apply LoRA
                "q_proj", "k_proj", "v_proj", "o_proj",
                "gate_proj", "up_proj", "down_proj",
            ],
            lora_dropout=0.05,
            bias="none",
        )
        
        self.model = get_peft_model(self.model, lora_config)
        
        trainable_params = sum(p.numel() for p in self.model.parameters() if p.requires_grad)
        total_params = sum(p.numel() for p in self.model.parameters())
        
        print(f"Trainable parameters: {trainable_params:,} ({100*trainable_params/total_params:.2f}%)")
        
        return self
    
    def prepare_dataset(self, conversations):
        """
        เตรียม dataset จาก LINE Bot conversations
        
        Format: ChatML
        <|im_start|>system
        คุณคือผู้ช่วยของร้านค้า X ตอบคำถามลูกค้าด้วยภาษาไทยที่สุภาพ
        <|im_end|>
        <|im_start|>user
        สินค้าราคาเท่าไหร่
        <|im_end|>
        <|im_start|>assistant
        ราคาสินค้าเริ่มต้นที่ 299 บาทครับ...
        <|im_end|>
        """
        
        def format_conversation(conv):
            system_prompt = """คุณคือผู้ช่วย AI ของร้านค้า ตอบคำถามลูกค้าด้วยภาษาไทยที่สุภาพ 
กระชับ และตรงประเด็น ให้ข้อมูลที่ถูกต้องและเป็นประโยชน์"""
            
            formatted = f"<|im_start|>system\n{system_prompt}\n<|im_end|>\n"
            
            for turn in conv:
                role = turn['role']
                content = turn['content']
                formatted += f"<|im_start|>{role}\n{content}\n<|im_end|>\n"
            
            return formatted
        
        texts = [format_conversation(conv) for conv in conversations]
        
        return Dataset.from_dict({'text': texts})
    
    def train(self, conversations):
        """Train ด้วย SFT (Supervised Fine-tuning)"""
        
        dataset = self.prepare_dataset(conversations)
        
        training_config = SFTConfig(
            output_dir=self.output_dir,
            num_train_epochs=3,
            per_device_train_batch_size=4,
            gradient_accumulation_steps=4,
            learning_rate=2e-4,
            warmup_ratio=0.05,
            lr_scheduler_type="cosine",
            save_steps=100,
            logging_steps=25,
            fp16=False,
            bf16=True,
            max_seq_length=2048,
            dataset_text_field="text",
            packing=True,
            report_to="mlflow",
        )
        
        trainer = SFTTrainer(
            model=self.model,
            tokenizer=self.tokenizer,
            train_dataset=dataset,
            args=training_config,
        )
        
        print("Starting fine-tuning...")
        trainer.train()
        
        # Save adapter
        trainer.model.save_pretrained(f"{self.output_dir}/final-adapter")
        self.tokenizer.save_pretrained(f"{self.output_dir}/final-adapter")
        
        print(f"Fine-tuning complete. Saved to {self.output_dir}/final-adapter")
    
    def generate_response(self, user_message, system_prompt=None, max_tokens=500):
        """Generate response จาก fine-tuned model"""
        
        if system_prompt is None:
            system_prompt = "คุณคือผู้ช่วย AI ของร้านค้า ตอบคำถามด้วยภาษาไทยที่สุภาพ"
        
        prompt = f"""<|im_start|>system
{system_prompt}
<|im_end|>
<|im_start|>user
{user_message}
<|im_end|>
<|im_start|>assistant
"""
        
        inputs = self.tokenizer(
            prompt, return_tensors="pt"
        ).to(self.model.device)
        
        with torch.no_grad():
            outputs = self.model.generate(
                **inputs,
                max_new_tokens=max_tokens,
                temperature=0.7,
                top_p=0.9,
                do_sample=True,
                eos_token_id=self.tokenizer.eos_token_id,
                pad_token_id=self.tokenizer.eos_token_id,
                repetition_penalty=1.1,
            )
        
        response = self.tokenizer.decode(
            outputs[0][inputs['input_ids'].shape[1]:],
            skip_special_tokens=True
        )
        
        return response.strip()


# Sample fine-tuning conversations
SAMPLE_CONVERSATIONS = [
    [
        {"role": "user", "content": "สวัสดีครับ อยากสอบถามเรื่องสินค้า"},
        {"role": "assistant", "content": "สวัสดีครับ! ยินดีช่วยเหลือครับ มีอะไรอยากสอบถามเกี่ยวกับสินค้าได้เลยนะครับ"},
    ],
    [
        {"role": "user", "content": "สินค้าชิ้นนี้ราคาเท่าไหร่"},
        {"role": "assistant", "content": "สินค้าชิ้นนี้ราคา 599 บาทครับ หากซื้อ 2 ชิ้นขึ้นไปลด 10% เพิ่มเติมด้วยครับ มีสนใจสั่งซื้อไหมครับ?"},
    ],
    [
        {"role": "user", "content": "สั่งแล้วเมื่อวาน ยังไม่ได้รับเลยครับ"},
        {"role": "assistant", "content": "ขออภัยในความไม่สะดวกนะครับ รบกวนแจ้งหมายเลขออร์เดอร์ให้ผมทราบด้วยนะครับ จะตรวจสอบสถานะการจัดส่งให้ทันทีเลยครับ"},
    ],
    [
        {"role": "user", "content": "สินค้าที่ได้รับมาเสียหายครับ"},
        {"role": "assistant", "content": "ขอโทษอย่างยิ่งสำหรับปัญหาที่เกิดขึ้นนะครับ กรุณาถ่ายรูปสินค้าที่เสียหายส่งมาให้ผมดูด้วยครับ เราจะดำเนินการเปลี่ยนสินค้าให้ทันทีครับ"},
    ],
]
```

---

## 4. RLHF for Chatbots

```python
# ml/rlhf/reward_model.py
"""
RLHF (Reinforcement Learning from Human Feedback)
สำหรับ LINE Bot Chatbot
"""

import torch
from transformers import AutoTokenizer, AutoModelForSequenceClassification
from torch.optim import AdamW

class RewardModel:
    """
    Reward Model สำหรับประเมิน response quality
    """
    
    def __init__(self, base_model='airesearch/wangchanberta-base-att-spm-uncased'):
        self.tokenizer = AutoTokenizer.from_pretrained(base_model)
        self.model = AutoModelForSequenceClassification.from_pretrained(
            base_model,
            num_labels=1,  # Single reward score
        )
    
    def prepare_pairwise_data(self, pairs):
        """
        เตรียมข้อมูล pairwise comparisons
        pairs: list of (prompt, chosen_response, rejected_response)
        """
        chosen_inputs = []
        rejected_inputs = []
        
        for prompt, chosen, rejected in pairs:
            chosen_text = f"{prompt} [SEP] {chosen}"
            rejected_text = f"{prompt} [SEP] {rejected}"
            
            chosen_inputs.append(chosen_text)
            rejected_inputs.append(rejected_text)
        
        return chosen_inputs, rejected_inputs
    
    def train(self, pairs, epochs=3, lr=1e-5):
        """
        Train reward model ด้วย pairwise ranking loss
        """
        optimizer = AdamW(self.model.parameters(), lr=lr)
        
        chosen_inputs, rejected_inputs = self.prepare_pairwise_data(pairs)
        
        for epoch in range(epochs):
            total_loss = 0
            
            for chosen, rejected in zip(chosen_inputs, rejected_inputs):
                # Tokenize
                chosen_enc = self.tokenizer(
                    chosen, return_tensors='pt',
                    padding=True, truncation=True, max_length=512
                )
                rejected_enc = self.tokenizer(
                    rejected, return_tensors='pt',
                    padding=True, truncation=True, max_length=512
                )
                
                # Forward pass
                chosen_reward = self.model(**chosen_enc).logits
                rejected_reward = self.model(**rejected_enc).logits
                
                # Ranking loss: chosen > rejected
                loss = -torch.log(torch.sigmoid(chosen_reward - rejected_reward)).mean()
                
                optimizer.zero_grad()
                loss.backward()
                optimizer.step()
                
                total_loss += loss.item()
            
            avg_loss = total_loss / len(pairs)
            print(f"Epoch {epoch+1}/{epochs}, Loss: {avg_loss:.4f}")
    
    def score(self, prompt, response):
        """ให้คะแนน response"""
        text = f"{prompt} [SEP] {response}"
        
        inputs = self.tokenizer(
            text, return_tensors='pt',
            padding=True, truncation=True, max_length=512
        )
        
        with torch.no_grad():
            score = self.model(**inputs).logits[0].item()
        
        return score


# Sample preference data สำหรับ Thai LINE Bot
PREFERENCE_DATA = [
    (
        "สินค้าราคาเท่าไหร่",
        "สินค้าชิ้นนี้ราคา 299 บาทครับ ถ้าสนใจสั่งซื้อแจ้งได้เลยนะครับ",  # chosen
        "ผมไม่ทราบราคาครับ กรุณาติดต่อเจ้าหน้าที่",  # rejected
    ),
    (
        "ส่งสินค้าถึงจังหวัดชลบุรีได้ไหม",
        "ได้เลยครับ เราจัดส่งทั่วประเทศไทย ใช้เวลา 2-3 วันทำการครับ ค่าจัดส่ง 50 บาท หรือฟรีเมื่อซื้อครบ 500 บาท",  # chosen
        "ได้ครับ",  # rejected (too brief)
    ),
    (
        "คืนสินค้าได้ไหม",
        "ได้ครับ เรารับคืนสินค้าภายใน 7 วัน หากสินค้าไม่มีการใช้งานและมีสภาพสมบูรณ์ครับ กรุณาแจ้งเหตุผลการคืนด้วยนะครับ",  # chosen
        "ขึ้นอยู่กับเงื่อนไขครับ",  # rejected (vague)
    ),
]
```

---

## 5. RAG (Retrieval Augmented Generation)

### 5.1 Complete RAG System

```python
# ml/rag/rag_system.py
from typing import List, Dict, Optional
import json
import re

class LineRAGSystem:
    """
    RAG System สำหรับ LINE Bot
    ช่วยให้ Bot ตอบคำถามจาก knowledge base ของธุรกิจ
    """
    
    def __init__(self, config: Dict):
        self.config = config
        self.vector_store = None
        self.llm_client = None
        self.embedding_model = None
    
    def setup(self):
        """Initialize all components"""
        self._setup_embeddings()
        self._setup_vector_store()
        self._setup_llm()
        return self
    
    def _setup_embeddings(self):
        """Setup multilingual embeddings ที่รองรับภาษาไทย"""
        from sentence_transformers import SentenceTransformer
        
        # Model ที่รองรับภาษาไทย
        model_name = self.config.get(
            'embedding_model',
            'intfloat/multilingual-e5-base'
        )
        
        self.embedding_model = SentenceTransformer(model_name)
        print(f"Embedding model loaded: {model_name}")
    
    def _setup_vector_store(self):
        """Setup vector database"""
        vector_store_type = self.config.get('vector_store', 'chroma')
        
        if vector_store_type == 'chroma':
            import chromadb
            
            self.chroma_client = chromadb.PersistentClient(
                path=self.config.get('chroma_path', './chroma_db')
            )
            
            self.vector_store = self.chroma_client.get_or_create_collection(
                name="linebot_knowledge",
                metadata={"hnsw:space": "cosine"},
            )
            
        elif vector_store_type == 'weaviate':
            import weaviate
            
            self.weaviate_client = weaviate.Client(
                url=self.config.get('weaviate_url', 'http://localhost:8080'),
            )
            
            # Create schema
            schema = {
                "classes": [{
                    "class": "KnowledgeBase",
                    "vectorizer": "none",
                    "properties": [
                        {"name": "content", "dataType": ["text"]},
                        {"name": "source", "dataType": ["string"]},
                        {"name": "category", "dataType": ["string"]},
                        {"name": "metadata", "dataType": ["text"]},
                    ],
                }]
            }
            
            if not self.weaviate_client.schema.exists("KnowledgeBase"):
                self.weaviate_client.schema.create(schema)
                
        elif vector_store_type == 'pinecone':
            from pinecone import Pinecone, ServerlessSpec
            
            pc = Pinecone(api_key=self.config['pinecone_api_key'])
            
            index_name = self.config.get('pinecone_index', 'linebot-knowledge')
            
            if index_name not in pc.list_indexes().names():
                pc.create_index(
                    name=index_name,
                    dimension=768,  # for multilingual-e5-base
                    metric="cosine",
                    spec=ServerlessSpec(
                        cloud='aws',
                        region='ap-southeast-1'
                    ),
                )
            
            self.vector_store = pc.Index(index_name)
    
    def _setup_llm(self):
        """Setup LLM client"""
        llm_provider = self.config.get('llm_provider', 'openai')
        
        if llm_provider == 'openai':
            from openai import OpenAI
            self.llm_client = OpenAI(api_key=self.config['openai_api_key'])
            self.llm_model = self.config.get('llm_model', 'gpt-4o-mini')
            
        elif llm_provider == 'anthropic':
            import anthropic
            self.llm_client = anthropic.Anthropic(api_key=self.config['anthropic_api_key'])
            self.llm_model = self.config.get('llm_model', 'claude-3-haiku-20240307')
            
        elif llm_provider == 'local':
            # Ollama local model
            import ollama
            self.llm_client = ollama
            self.llm_model = self.config.get('llm_model', 'typhoon-7b')
    
    def add_documents(self, documents: List[Dict]):
        """
        เพิ่ม documents ลง knowledge base
        
        documents: list of {content, source, category, metadata}
        """
        
        texts = [doc['content'] for doc in documents]
        
        # Compute embeddings
        print(f"Computing embeddings for {len(texts)} documents...")
        embeddings = self.embedding_model.encode(
            texts,
            batch_size=32,
            show_progress_bar=True,
            normalize_embeddings=True,
        )
        
        # Store in Chroma
        ids = [f"doc_{i}" for i in range(len(documents))]
        
        self.vector_store.upsert(
            ids=ids,
            embeddings=embeddings.tolist(),
            documents=texts,
            metadatas=[{
                'source': doc.get('source', ''),
                'category': doc.get('category', ''),
                'metadata': json.dumps(doc.get('metadata', {})),
            } for doc in documents],
        )
        
        print(f"Added {len(documents)} documents to knowledge base")
    
    def retrieve(self, query: str, n_results: int = 5) -> List[Dict]:
        """
        ค้นหา relevant documents จาก vector store
        """
        query_embedding = self.embedding_model.encode(
            query,
            normalize_embeddings=True,
        )
        
        results = self.vector_store.query(
            query_embeddings=[query_embedding.tolist()],
            n_results=n_results,
            include=['documents', 'metadatas', 'distances'],
        )
        
        retrieved_docs = []
        for doc, metadata, distance in zip(
            results['documents'][0],
            results['metadatas'][0],
            results['distances'][0],
        ):
            retrieved_docs.append({
                'content': doc,
                'source': metadata.get('source', ''),
                'category': metadata.get('category', ''),
                'relevance_score': 1 - distance,  # Convert distance to similarity
            })
        
        return retrieved_docs
    
    def generate_response(self, user_query: str, chat_history: List = None) -> str:
        """
        Generate response ด้วย RAG
        """
        
        # 1. Retrieve relevant documents
        relevant_docs = self.retrieve(user_query, n_results=5)
        
        # Filter by relevance threshold
        relevant_docs = [d for d in relevant_docs if d['relevance_score'] > 0.5]
        
        # 2. Build context
        if relevant_docs:
            context = "\n\n".join([
                f"[{doc['category']}] {doc['content']}"
                for doc in relevant_docs
            ])
        else:
            context = "ไม่พบข้อมูลที่เกี่ยวข้อง"
        
        # 3. Build prompt
        system_prompt = """คุณคือผู้ช่วย AI ของร้านค้า ตอบคำถามลูกค้าด้วยภาษาไทยที่สุภาพ

กฎการตอบ:
1. ตอบตามข้อมูลที่ให้มาเท่านั้น
2. หากไม่มีข้อมูล บอกว่าไม่ทราบและแนะนำให้ติดต่อเจ้าหน้าที่
3. ตอบกระชับ ตรงประเด็น
4. ใช้ภาษาที่เป็นมิตร

ข้อมูลที่เกี่ยวข้อง:
{context}"""
        
        messages = [
            {"role": "system", "content": system_prompt.format(context=context)}
        ]
        
        # Add chat history
        if chat_history:
            messages.extend(chat_history[-6:])  # Last 3 turns
        
        messages.append({"role": "user", "content": user_query})
        
        # 4. Generate response
        llm_provider = self.config.get('llm_provider', 'openai')
        
        if llm_provider == 'openai':
            response = self.llm_client.chat.completions.create(
                model=self.llm_model,
                messages=messages,
                temperature=0.3,
                max_tokens=500,
            )
            return response.choices[0].message.content
            
        elif llm_provider == 'anthropic':
            response = self.llm_client.messages.create(
                model=self.llm_model,
                messages=[m for m in messages if m['role'] != 'system'],
                system=messages[0]['content'],
                max_tokens=500,
                temperature=0.3,
            )
            return response.content[0].text
            
        elif llm_provider == 'local':
            response = self.llm_client.chat(
                model=self.llm_model,
                messages=messages,
                options={'temperature': 0.3},
            )
            return response['message']['content']
        
        return "ขออภัยครับ เกิดข้อผิดพลาดในการประมวลผล"
    
    def add_feedback(self, query: str, response: str, rating: int, user_id: str):
        """บันทึก feedback เพื่อ improve ในอนาคต"""
        feedback = {
            'query': query,
            'response': response,
            'rating': rating,  # 1-5
            'user_id': user_id,
            'timestamp': __import__('datetime').datetime.now().isoformat(),
        }
        
        # Store feedback for RLHF training
        with open('./data/feedback.jsonl', 'a') as f:
            f.write(json.dumps(feedback, ensure_ascii=False) + '\n')
```

### 5.2 LINE Bot Integration กับ RAG

```javascript
// src/ai/RAGLineBot.js
const axios = require('axios');

class RAGLineBot {
  constructor({ ragApiUrl, lineClient, sessionManager }) {
    this.ragApi = axios.create({
      baseURL: ragApiUrl,
      timeout: 10000,
    });
    this.lineClient = lineClient;
    this.sessionManager = sessionManager;
  }
  
  async handleMessage(event) {
    const userId = event.source.userId;
    const userMessage = event.message.text;
    
    // Get chat history
    const session = await this.sessionManager.getSession(userId);
    const chatHistory = session?.chatHistory || [];
    
    // Show typing indicator
    await this.lineClient.showLoadingAnimation(userId, 5);
    
    try {
      // Call RAG API
      const response = await this.ragApi.post('/generate', {
        query: userMessage,
        user_id: userId,
        chat_history: chatHistory,
      });
      
      const aiResponse = response.data.response;
      const sources = response.data.sources || [];
      
      // Build LINE message
      const messages = this.buildResponseMessages(aiResponse, sources);
      
      // Reply
      await this.lineClient.replyMessage(event.replyToken, messages);
      
      // Update session
      await this.sessionManager.updateSession(userId, {
        chatHistory: [
          ...chatHistory.slice(-10), // Keep last 5 turns
          { role: 'user', content: userMessage },
          { role: 'assistant', content: aiResponse },
        ],
      });
      
    } catch (error) {
      console.error('RAG API error:', error);
      
      await this.lineClient.replyMessage(event.replyToken, {
        type: 'text',
        text: 'ขออภัยครับ กำลังประมวลผล กรุณาลองใหม่อีกครั้ง',
      });
    }
  }
  
  buildResponseMessages(response, sources) {
    const messages = [{
      type: 'text',
      text: response,
    }];
    
    // เพิ่ม source references ถ้ามี
    if (sources && sources.length > 0) {
      const quickReplies = sources
        .filter(s => s.url)
        .slice(0, 3)
        .map((source, i) => ({
          type: 'action',
          action: {
            type: 'uri',
            label: `อ่านเพิ่มเติม ${i + 1}`,
            uri: source.url,
          },
        }));
      
      if (quickReplies.length > 0) {
        messages[0].quickReply = { items: quickReplies };
      }
    }
    
    return messages;
  }
  
  async handleFeedback(event) {
    const userId = event.source.userId;
    const data = JSON.parse(event.postback.data);
    
    if (data.action === 'feedback') {
      await this.ragApi.post('/feedback', {
        query: data.query,
        response: data.response,
        rating: data.rating,
        user_id: userId,
      });
      
      await this.lineClient.replyMessage(event.replyToken, {
        type: 'text',
        text: 'ขอบคุณสำหรับ feedback ครับ จะนำไปปรับปรุงต่อไปครับ 🙏',
      });
    }
  }
}

module.exports = RAGLineBot;
```

---

## 6. Vector Databases Comparison

### 6.1 Comparison Matrix

```
Vector Database Comparison:

┌─────────────────┬───────────┬───────────┬───────────┬───────────┐
│  Feature        │  Chroma   │  Weaviate │  Pinecone │  Qdrant   │
├─────────────────┼───────────┼───────────┼───────────┼───────────┤
│  Type           │ Open      │ Open/     │ Managed   │ Open/     │
│                 │ Source    │ Managed   │ Cloud     │ Managed   │
├─────────────────┼───────────┼───────────┼───────────┼───────────┤
│  Setup          │ ★★★★★    │ ★★★☆☆    │ ★★★★★    │ ★★★★☆    │
│  Complexity     │ Easy      │ Medium    │ Easy      │ Easy      │
├─────────────────┼───────────┼───────────┼───────────┼───────────┤
│  Performance    │ ★★★☆☆    │ ★★★★☆    │ ★★★★★    │ ★★★★★    │
│  (1M vectors)   │ ~50ms     │ ~10ms     │ ~5ms      │ ~5ms      │
├─────────────────┼───────────┼───────────┼───────────┼───────────┤
│  Scalability    │ ★★☆☆☆    │ ★★★★☆    │ ★★★★★    │ ★★★★☆    │
├─────────────────┼───────────┼───────────┼───────────┼───────────┤
│  Cost           │ Free      │ Free/     │ $70+/mo   │ Free/     │
│                 │           │ $25+/mo   │           │ $9+/mo    │
├─────────────────┼───────────┼───────────┼───────────┼───────────┤
│  Thai Support   │ ✓         │ ✓         │ ✓         │ ✓         │
│  (via embedding)│           │           │           │           │
├─────────────────┼───────────┼───────────┼───────────┼───────────┤
│  Best For       │ Dev/Test  │ Production│ Enterprise│ Production│
│                 │ Prototype │ Medium    │ Large     │ Medium    │
└─────────────────┴───────────┴───────────┴───────────┴───────────┘

Recommendation for LINE Bot:
├── Development: Chroma (zero cost, easy setup)
├── Production Small: Qdrant (cost-effective, good performance)
├── Production Large: Pinecone (managed, scalable)
└── Self-hosted Enterprise: Weaviate or Qdrant
```

### 6.2 Qdrant Setup

```python
# ml/rag/qdrant_store.py
from qdrant_client import QdrantClient
from qdrant_client.models import (
    VectorParams, Distance, PointStruct,
    Filter, FieldCondition, MatchValue,
    SearchRequest, ScoredPoint,
)
import uuid

class QdrantKnowledgeStore:
    """
    Knowledge Store ด้วย Qdrant
    """
    
    def __init__(self, url="http://localhost:6333", collection_name="linebot_knowledge"):
        self.client = QdrantClient(url=url)
        self.collection_name = collection_name
        self.vector_size = 768  # multilingual-e5-base dimension
    
    def initialize(self):
        """สร้าง collection"""
        
        collections = self.client.get_collections().collections
        existing_names = [c.name for c in collections]
        
        if self.collection_name not in existing_names:
            self.client.create_collection(
                collection_name=self.collection_name,
                vectors_config=VectorParams(
                    size=self.vector_size,
                    distance=Distance.COSINE,
                ),
            )
            print(f"Created collection: {self.collection_name}")
        else:
            print(f"Collection already exists: {self.collection_name}")
    
    def upsert_documents(self, documents, embeddings):
        """
        เพิ่มหรืออัพเดต documents
        """
        points = []
        
        for doc, embedding in zip(documents, embeddings):
            point = PointStruct(
                id=str(uuid.uuid4()),
                vector=embedding.tolist(),
                payload={
                    'content': doc['content'],
                    'source': doc.get('source', ''),
                    'category': doc.get('category', ''),
                    'title': doc.get('title', ''),
                    'metadata': doc.get('metadata', {}),
                },
            )
            points.append(point)
        
        self.client.upsert(
            collection_name=self.collection_name,
            points=points,
        )
        
        print(f"Upserted {len(points)} documents")
    
    def search(self, query_embedding, n_results=5, category_filter=None):
        """ค้นหา relevant documents"""
        
        query_filter = None
        if category_filter:
            query_filter = Filter(
                must=[
                    FieldCondition(
                        key="category",
                        match=MatchValue(value=category_filter),
                    )
                ]
            )
        
        results = self.client.search(
            collection_name=self.collection_name,
            query_vector=query_embedding.tolist(),
            query_filter=query_filter,
            limit=n_results,
            with_payload=True,
        )
        
        return [
            {
                'content': r.payload['content'],
                'source': r.payload.get('source', ''),
                'category': r.payload.get('category', ''),
                'title': r.payload.get('title', ''),
                'score': r.score,
            }
            for r in results
        ]
    
    def delete_by_source(self, source):
        """ลบ documents ตาม source"""
        self.client.delete(
            collection_name=self.collection_name,
            points_selector=Filter(
                must=[
                    FieldCondition(
                        key="source",
                        match=MatchValue(value=source),
                    )
                ]
            ),
        )
```

---

## 7. Model Serving

### 7.1 Ollama Setup สำหรับ Local Models

```bash
# scripts/setup-ollama.sh
#!/bin/bash

# Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# Pull Thai-friendly models
ollama pull typhoon2-8b
ollama pull llama3.1:8b

# Create custom Modelfile สำหรับ LINE Bot
cat > /tmp/Modelfile << 'EOF'
FROM typhoon2-8b

# Set system prompt สำหรับ LINE Bot customer service
SYSTEM """
คุณคือผู้ช่วย AI ของร้านค้า Thai Shop 
ตอบคำถามลูกค้าด้วยภาษาไทยที่สุภาพและเป็นมิตร
ตอบกระชับ ตรงประเด็น และเป็นประโยชน์

หากไม่แน่ใจ ให้แนะนำให้ติดต่อเจ้าหน้าที่ที่ LINE: @thaishop
"""

# Tune parameters
PARAMETER temperature 0.3
PARAMETER top_p 0.9
PARAMETER top_k 40
PARAMETER repeat_penalty 1.1
PARAMETER num_ctx 4096
EOF

ollama create linebot-assistant -f /tmp/Modelfile

echo "Ollama setup complete"
```

### 7.2 vLLM Deployment

```yaml
# kubernetes/vllm-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-server
  namespace: linebot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vllm-server
  template:
    metadata:
      labels:
        app: vllm-server
    spec:
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        command:
          - python
          - -m
          - vllm.entrypoints.openai.api_server
          - --model
          - scb10x/typhoon-7b
          - --dtype
          - bfloat16
          - --max-model-len
          - "4096"
          - --tensor-parallel-size
          - "1"
          - --gpu-memory-utilization
          - "0.9"
          - --port
          - "8000"
        ports:
        - containerPort: 8000
        resources:
          limits:
            nvidia.com/gpu: "1"
            memory: "24Gi"
          requests:
            nvidia.com/gpu: "1"
            memory: "20Gi"
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 120
          periodSeconds: 10

---
apiVersion: v1
kind: Service
metadata:
  name: vllm-service
  namespace: linebot
spec:
  selector:
    app: vllm-server
  ports:
  - port: 8000
    targetPort: 8000
```

---

## 8. Cost Comparison

```
Cost Comparison: GPT-4 vs Custom Model

สมมติ: LINE Bot ที่มี 100,000 messages/วัน
       Average tokens: 500 input + 300 output per message

GPT-4o (gpt-4o):
├── Input: 100,000 × 500 tokens = 50M tokens/day
├── Output: 100,000 × 300 tokens = 30M tokens/day
├── Cost: 50M × $2.50/1M + 30M × $10/1M
├── Daily cost: $125 + $300 = $425/day
└── Monthly cost: ~$12,750/month

GPT-4o-mini:
├── Cost: 50M × $0.15/1M + 30M × $0.60/1M
├── Daily cost: $7.5 + $18 = $25.5/day
└── Monthly cost: ~$765/month

Custom Model (Typhoon-7B on vLLM):
├── GPU instance (A100 40GB): ~$3/hr
├── Can handle ~1000 req/min
├── For 100k req/day ≈ 70 req/min average
├── 1 GPU instance เพียงพอ
├── Monthly cost: ~$2,160/month (24/7)
├── But: Setup cost + maintenance
└── Break-even vs GPT-4o: ~3 months

Custom Model (Typhoon-7B Serverless):
├── Modal.com หรือ RunPod Serverless
├── Pay per request: ~$0.0001/token
├── For same usage: ~$800/month
└── No maintenance overhead

Recommendation:
├── < 10k msg/day: GPT-4o-mini (simplest, cheapest)
├── 10k-100k msg/day: Serverless GPU + local model
├── > 100k msg/day: Dedicated GPU + vLLM
└── Privacy-critical: Always local model
```

---

## 9. Evaluation Framework

```python
# ml/evaluation/eval_framework.py
from typing import List, Dict
import json
import re

class ChatbotEvaluator:
    """
    Evaluate chatbot quality ด้วย multiple metrics
    """
    
    def __init__(self, judge_model_client=None):
        self.judge_client = judge_model_client  # LLM as judge
    
    def evaluate_response(self, question: str, response: str, 
                           ground_truth: str = None) -> Dict:
        """
        Evaluate single response
        """
        metrics = {}
        
        # 1. Length check
        metrics['response_length'] = len(response)
        metrics['is_too_short'] = len(response) < 20
        metrics['is_too_long'] = len(response) > 1000
        
        # 2. Thai language check
        thai_chars = sum(1 for c in response if '฀' <= c <= '๿')
        metrics['thai_ratio'] = thai_chars / max(len(response), 1)
        metrics['has_thai'] = metrics['thai_ratio'] > 0.3
        
        # 3. Politeness markers
        polite_markers = ['ครับ', 'ค่ะ', 'นะครับ', 'นะคะ', 'ขอบคุณ', 'ยินดี']
        metrics['has_polite_marker'] = any(m in response for m in polite_markers)
        
        # 4. Question answering quality (if ground truth provided)
        if ground_truth:
            metrics['answer_overlap'] = self.calculate_answer_overlap(
                response, ground_truth
            )
        
        # 5. LLM-as-Judge evaluation
        if self.judge_client:
            judge_score = self.llm_judge(question, response)
            metrics.update(judge_score)
        
        # Overall score
        score = 0
        if not metrics['is_too_short']: score += 20
        if not metrics['is_too_long']: score += 10
        if metrics['has_thai']: score += 20
        if metrics['has_polite_marker']: score += 20
        if metrics.get('answer_overlap', 0) > 0.5: score += 30
        
        metrics['overall_score'] = score
        
        return metrics
    
    def calculate_answer_overlap(self, response, ground_truth):
        """คำนวณ token overlap ระหว่าง response กับ ground truth"""
        response_tokens = set(response.split())
        truth_tokens = set(ground_truth.split())
        
        if not truth_tokens:
            return 0
        
        overlap = len(response_tokens & truth_tokens)
        return overlap / len(truth_tokens)
    
    def llm_judge(self, question: str, response: str) -> Dict:
        """ใช้ LLM เพื่อประเมิน response quality"""
        
        prompt = f"""ประเมิน response ของ chatbot ตาม criteria ต่อไปนี้:
คำถาม: {question}
คำตอบ: {response}

ให้คะแนน 1-5 ในแต่ละด้าน:
1. ความถูกต้อง (Correctness)
2. ความเป็นมิตร (Friendliness)  
3. ความกระชับ (Conciseness)
4. ภาษาไทยที่ถูกต้อง (Thai Language)

ตอบในรูปแบบ JSON:
{{"correctness": 4, "friendliness": 5, "conciseness": 3, "thai_language": 5}}"""
        
        try:
            response_text = self.judge_client.chat.completions.create(
                model='gpt-4o-mini',
                messages=[{'role': 'user', 'content': prompt}],
                temperature=0,
                max_tokens=100,
            ).choices[0].message.content
            
            # Parse JSON
            json_match = re.search(r'\{.*\}', response_text, re.DOTALL)
            if json_match:
                scores = json.loads(json_match.group())
                scores['llm_judge_avg'] = sum(scores.values()) / len(scores)
                return scores
        except Exception as e:
            print(f"LLM judge error: {e}")
        
        return {}
    
    def evaluate_dataset(self, test_cases: List[Dict]) -> Dict:
        """
        Evaluate ชุด test cases
        
        test_cases: [{'question': ..., 'response': ..., 'ground_truth': ...}]
        """
        all_metrics = []
        
        for case in test_cases:
            metrics = self.evaluate_response(
                case['question'],
                case['response'],
                case.get('ground_truth'),
            )
            all_metrics.append(metrics)
        
        # Aggregate
        avg_metrics = {}
        for key in all_metrics[0].keys():
            if isinstance(all_metrics[0][key], (int, float)):
                avg_metrics[f'avg_{key}'] = sum(m[key] for m in all_metrics) / len(all_metrics)
        
        avg_metrics['n_samples'] = len(all_metrics)
        
        return avg_metrics
```

---

## 10. Complete RAG + LINE Bot System

### 10.1 Docker Compose

```yaml
# docker-compose.rag.yml
version: '3.8'

services:
  # Qdrant Vector Database
  qdrant:
    image: qdrant/qdrant:v1.6.4
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant-data:/qdrant/storage
    environment:
      QDRANT__SERVICE__HTTP_PORT: 6333
      QDRANT__SERVICE__GRPC_PORT: 6334

  # RAG API Server
  rag-api:
    build: ./services/rag-api
    ports:
      - "8000:8000"
    environment:
      QDRANT_URL: http://qdrant:6333
      EMBEDDING_MODEL: intfloat/multilingual-e5-base
      LLM_PROVIDER: openai
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      LLM_MODEL: gpt-4o-mini
    depends_on:
      - qdrant
    volumes:
      - ./data/knowledge:/app/data/knowledge
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 4G

  # LINE Bot Webhook
  linebot:
    build: ./services/linebot
    ports:
      - "3000:3000"
    environment:
      LINE_CHANNEL_ACCESS_TOKEN: ${LINE_CHANNEL_ACCESS_TOKEN}
      LINE_CHANNEL_SECRET: ${LINE_CHANNEL_SECRET}
      RAG_API_URL: http://rag-api:8000
      REDIS_URL: redis://redis:6379
    depends_on:
      - rag-api
      - redis

  # Redis for session management
  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

  # MLflow for experiment tracking
  mlflow:
    image: ghcr.io/mlflow/mlflow:v2.8.0
    ports:
      - "5000:5000"
    command: mlflow server --host 0.0.0.0 --port 5000 --backend-store-uri sqlite:///mlflow.db
    volumes:
      - mlflow-data:/mlflow

volumes:
  qdrant-data:
  redis-data:
  mlflow-data:
```

### 10.2 Knowledge Base Ingestion Script

```python
# scripts/ingest_knowledge.py
"""
Ingest knowledge base จากหลาย sources:
- PDF files
- Website pages
- CSV/Excel
- Database
"""

import os
import glob
from pathlib import Path
from rag_system import LineRAGSystem
from sentence_transformers import SentenceTransformer

def load_pdf_documents(pdf_dir):
    """Load documents จาก PDF files"""
    from pypdf import PdfReader
    
    documents = []
    pdf_files = glob.glob(f"{pdf_dir}/**/*.pdf", recursive=True)
    
    for pdf_path in pdf_files:
        reader = PdfReader(pdf_path)
        filename = Path(pdf_path).stem
        
        for page_num, page in enumerate(reader.pages):
            text = page.extract_text()
            if len(text.strip()) > 50:  # Skip empty pages
                documents.append({
                    'content': text,
                    'source': pdf_path,
                    'category': 'document',
                    'title': f"{filename} - Page {page_num + 1}",
                    'metadata': {'page': page_num + 1, 'filename': filename},
                })
    
    return documents

def load_website_content(urls):
    """Scrape content จาก website"""
    import requests
    from bs4 import BeautifulSoup
    
    documents = []
    
    for url in urls:
        try:
            response = requests.get(url, timeout=10)
            soup = BeautifulSoup(response.content, 'html.parser')
            
            # Remove scripts and styles
            for element in soup(['script', 'style', 'nav', 'footer']):
                element.decompose()
            
            text = soup.get_text(separator=' ', strip=True)
            
            if len(text) > 100:
                documents.append({
                    'content': text[:2000],  # Limit size
                    'source': url,
                    'category': 'website',
                    'title': soup.title.string if soup.title else url,
                })
        except Exception as e:
            print(f"Error loading {url}: {e}")
    
    return documents

def load_faq_from_csv(csv_path):
    """Load FAQ จาก CSV"""
    import pandas as pd
    
    df = pd.read_csv(csv_path)
    documents = []
    
    for _, row in df.iterrows():
        documents.append({
            'content': f"คำถาม: {row['question']}\nคำตอบ: {row['answer']}",
            'source': csv_path,
            'category': 'faq',
            'title': row['question'],
            'metadata': {'question': row['question']},
        })
    
    return documents

def main():
    config = {
        'vector_store': 'qdrant',
        'qdrant_url': os.environ.get('QDRANT_URL', 'http://localhost:6333'),
        'embedding_model': 'intfloat/multilingual-e5-base',
        'llm_provider': 'openai',
        'openai_api_key': os.environ['OPENAI_API_KEY'],
        'llm_model': 'gpt-4o-mini',
    }
    
    rag = LineRAGSystem(config)
    rag.setup()
    
    all_documents = []
    
    # Load from PDFs
    if os.path.exists('./data/knowledge/pdfs'):
        pdf_docs = load_pdf_documents('./data/knowledge/pdfs')
        all_documents.extend(pdf_docs)
        print(f"Loaded {len(pdf_docs)} PDF documents")
    
    # Load FAQ
    if os.path.exists('./data/knowledge/faq.csv'):
        faq_docs = load_faq_from_csv('./data/knowledge/faq.csv')
        all_documents.extend(faq_docs)
        print(f"Loaded {len(faq_docs)} FAQ items")
    
    # Load website content
    website_urls = [
        'https://yourshop.com/faq',
        'https://yourshop.com/about',
        'https://yourshop.com/shipping',
    ]
    web_docs = load_website_content(website_urls)
    all_documents.extend(web_docs)
    print(f"Loaded {len(web_docs)} web pages")
    
    # Add all documents to knowledge base
    print(f"\nTotal documents: {len(all_documents)}")
    print("Adding to knowledge base...")
    rag.add_documents(all_documents)
    
    print("Knowledge base ingestion complete!")
    
    # Test retrieval
    test_query = "ส่งสินค้าใช้เวลากี่วัน"
    results = rag.retrieve(test_query, n_results=3)
    
    print(f"\nTest query: {test_query}")
    for r in results:
        print(f"  Score: {r['relevance_score']:.3f} | {r['content'][:100]}...")

if __name__ == '__main__':
    main()
```

---

## 11. Monitoring AI Models

```python
# src/monitoring/ai_monitor.py

class AIModelMonitor:
    """Monitor AI model performance ใน production"""
    
    def __init__(self, metrics_client):
        self.metrics = metrics_client
    
    def track_inference(self, model_name, latency_ms, tokens_used, success):
        """Track inference metrics"""
        self.metrics.histogram(
            'ai_inference_latency_ms',
            latency_ms,
            tags={'model': model_name}
        )
        
        self.metrics.counter(
            'ai_inference_total',
            1,
            tags={'model': model_name, 'success': str(success)}
        )
        
        if tokens_used:
            self.metrics.histogram(
                'ai_tokens_used',
                tokens_used,
                tags={'model': model_name}
            )
    
    def track_rag_metrics(self, query, n_retrieved, avg_relevance_score):
        """Track RAG-specific metrics"""
        self.metrics.histogram('rag_documents_retrieved', n_retrieved)
        self.metrics.histogram('rag_avg_relevance_score', avg_relevance_score)
        
        # Alert if relevance too low
        if avg_relevance_score < 0.5:
            self.log_low_relevance_query(query, avg_relevance_score)
    
    def log_low_relevance_query(self, query, score):
        """Log queries ที่ไม่พบข้อมูลที่ relevant"""
        import logging
        logging.warning(f"Low relevance query (score={score:.2f}): {query}")
        # ส่งไป review เพื่อเพิ่ม knowledge base
```

---

## สรุป

Custom AI Models สำหรับ LINE Bot มีหลายระดับ:

1. **Intent Classification (WangchanBERTa)**: ราคาถูก fast inference รองรับภาษาไทย
2. **Fine-tuned LLM (Typhoon)**: ตอบสนองได้ตรง domain ควบคุมได้
3. **RAG System**: Balance ระหว่าง accuracy, cost, และ maintainability
4. **RLHF**: ปรับปรุง model จาก user feedback อย่างต่อเนื่อง

สำหรับ LINE OA ทั่วไป แนะนำเริ่มจาก RAG + GPT-4o-mini แล้วค่อยๆ migrate ไป custom model เมื่อ volume สูงขึ้น

```
Migration Path:
Phase 1: GPT-4o-mini + RAG        → Fast, accurate, ~$765/mo
Phase 2: Fine-tuned Typhoon + RAG → Cost ↓ 50%, accuracy ↑ for Thai
Phase 3: Full local deployment     → Cost ↓ 70%, full data privacy
```

---

*จบ Part 85: Custom AI Models สำหรับ LINE Bots*

---

## 12. Production Deployment Pipeline

### 12.1 Model Registry API

```python
# services/model-registry/src/registry.py
from fastapi import FastAPI, HTTPException, UploadFile
from pydantic import BaseModel
import mlflow
from mlflow.tracking import MlflowClient
import json

app = FastAPI(title="LINE Bot Model Registry")
client = MlflowClient()

class ModelDeployRequest(BaseModel):
    model_name: str
    version: int
    environment: str  # staging, production
    
class InferenceRequest(BaseModel):
    model_name: str
    inputs: dict
    user_id: str

@app.post("/models/deploy")
async def deploy_model(request: ModelDeployRequest):
    """Deploy model ไปยัง environment"""
    
    try:
        if request.environment == 'production':
            # Archive existing production model
            client.transition_model_version_stage(
                name=request.model_name,
                version=request.version,
                stage="Production",
                archive_existing_versions=True
            )
        elif request.environment == 'staging':
            client.transition_model_version_stage(
                name=request.model_name,
                version=request.version,
                stage="Staging"
            )
        
        return {
            "status": "success",
            "model_name": request.model_name,
            "version": request.version,
            "environment": request.environment
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/models/{model_name}/production")
async def get_production_model_info(model_name: str):
    """ดึงข้อมูล production model"""
    
    versions = client.get_latest_versions(model_name, stages=["Production"])
    
    if not versions:
        raise HTTPException(status_code=404, detail=f"No production model found: {model_name}")
    
    version = versions[0]
    
    return {
        "model_name": model_name,
        "version": version.version,
        "run_id": version.run_id,
        "created_at": version.creation_timestamp,
        "metrics": client.get_run(version.run_id).data.metrics,
        "params": client.get_run(version.run_id).data.params,
    }

@app.post("/models/predict")
async def predict(request: InferenceRequest):
    """Real-time prediction endpoint"""
    
    import time
    start = time.time()
    
    try:
        # Load model from registry
        model = mlflow.pyfunc.load_model(
            f"models:/{request.model_name}/Production"
        )
        
        # Predict
        result = model.predict(request.inputs)
        duration_ms = (time.time() - start) * 1000
        
        return {
            "prediction": result,
            "model_name": request.model_name,
            "duration_ms": duration_ms,
            "user_id": request.user_id
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8080)
```

### 12.2 Inference Service with Caching

```python
# services/ai-inference/src/inference_service.py
import asyncio
import time
import redis
import json
import hashlib
from typing import Optional, Dict, Any

class InferenceService:
    """
    High-performance inference service with caching
    """
    
    def __init__(self, model_registry, redis_client, config):
        self.registry = model_registry
        self.redis = redis_client
        self.config = config
        self.models = {}  # In-memory model cache
        
        # Cache TTLs
        self.cache_ttls = {
            'rag_response': 300,      # 5 minutes (conversation varies)
            'intent': 60,             # 1 minute
            'recommendation': 3600,   # 1 hour
            'churn_prediction': 86400, # 24 hours
        }
    
    async def predict_intent(self, text: str, user_id: str) -> Dict:
        """ทำนาย intent พร้อม caching"""
        
        cache_key = f"intent:{hashlib.md5(text.encode()).hexdigest()}"
        
        # Check cache
        cached = await self.get_cache(cache_key)
        if cached:
            return {**cached, 'from_cache': True}
        
        start = time.time()
        
        model = await self.get_model('ThaiIntentClassifier')
        intent, confidence = model.predict(text)
        
        result = {
            'intent': intent,
            'confidence': float(confidence),
            'latency_ms': (time.time() - start) * 1000,
            'from_cache': False,
        }
        
        # Cache the result
        await self.set_cache(cache_key, result, self.cache_ttls['intent'])
        
        return result
    
    async def get_recommendations(self, user_id: str, n: int = 5) -> Dict:
        """Get personalized recommendations"""
        
        cache_key = f"recs:{user_id}:{n}"
        
        cached = await self.get_cache(cache_key)
        if cached:
            return {**cached, 'from_cache': True}
        
        model = await self.get_model('Recommender')
        recommendations = model.get_recommendations(user_id, n)
        
        result = {
            'user_id': user_id,
            'recommendations': recommendations,
            'from_cache': False,
        }
        
        await self.set_cache(cache_key, result, self.cache_ttls['recommendation'])
        
        return result
    
    async def generate_rag_response(self, query: str, user_id: str, 
                                     chat_history: list = None) -> Dict:
        """Generate RAG response (not cached due to conversation context)"""
        
        start = time.time()
        
        rag_system = await self.get_model('RAGSystem')
        response = rag_system.generate_response(query, chat_history)
        
        return {
            'response': response,
            'user_id': user_id,
            'latency_ms': (time.time() - start) * 1000,
        }
    
    async def get_model(self, model_name: str):
        """Load model with in-memory caching"""
        
        if model_name not in self.models:
            self.models[model_name] = await self.load_model(model_name)
        
        return self.models[model_name]
    
    async def load_model(self, model_name: str):
        """Load model from registry"""
        import mlflow.pyfunc
        
        model_uri = f"models:/{model_name}/Production"
        return mlflow.pyfunc.load_model(model_uri)
    
    async def get_cache(self, key: str) -> Optional[Dict]:
        try:
            value = self.redis.get(key)
            if value:
                return json.loads(value)
        except Exception:
            pass
        return None
    
    async def set_cache(self, key: str, value: Dict, ttl: int):
        try:
            self.redis.setex(key, ttl, json.dumps(value))
        except Exception as e:
            print(f"Cache set failed: {e}")
```

---

## 13. Training Data Pipeline

### 13.1 Conversation Data Collector

```python
# ml/data/conversation_collector.py
"""
เก็บ conversation data จาก LINE Bot สำหรับ training
"""

import json
import asyncio
from datetime import datetime
from typing import List, Dict

class ConversationDataCollector:
    """
    เก็บ conversations ที่มีคุณภาพสำหรับ fine-tuning
    """
    
    def __init__(self, db, storage_path='./data/conversations'):
        self.db = db
        self.storage_path = storage_path
        self.quality_threshold = 4.0  # Rating 4/5 ขึ้นไป
    
    async def collect_training_conversations(self, 
                                              min_turns: int = 3,
                                              min_rating: float = 4.0) -> List[Dict]:
        """
        ดึง conversations ที่ผ่าน quality filter
        """
        result = await self.db.query("""
            SELECT 
                c.session_id,
                c.user_id,
                json_agg(
                    json_build_object(
                        'role', CASE WHEN m.direction = 'inbound' THEN 'user' ELSE 'assistant' END,
                        'content', m.message_content->>'text',
                        'timestamp', m.created_at
                    ) ORDER BY m.created_at
                ) as turns,
                avg(f.rating) as avg_rating,
                count(m.id) as turn_count
            FROM chat_sessions c
            JOIN messages m ON c.id = m.session_id
            LEFT JOIN conversation_feedback f ON c.session_id = f.session_id
            WHERE m.message_content->>'text' IS NOT NULL
            GROUP BY c.session_id, c.user_id
            HAVING count(m.id) >= $1
            AND (avg(f.rating) >= $2 OR avg(f.rating) IS NULL)
        """, [min_turns, min_rating])
        
        conversations = []
        
        for row in result:
            # Filter out personal information
            turns = self.anonymize_turns(row['turns'])
            
            conversations.append({
                'session_id': row['session_id'],
                'turns': turns,
                'avg_rating': row['avg_rating'],
                'turn_count': row['turn_count'],
            })
        
        return conversations
    
    def anonymize_turns(self, turns: List[Dict]) -> List[Dict]:
        """Remove PII จาก conversation turns"""
        import re
        
        anonymized = []
        for turn in turns:
            content = turn['content']
            if content:
                # Remove phone numbers
                content = re.sub(r'\b0[0-9]{8,9}\b', '[PHONE]', content)
                # Remove email
                content = re.sub(r'\S+@\S+\.\S+', '[EMAIL]', content)
                # Remove Thai ID card
                content = re.sub(r'\b[0-9]{13}\b', '[ID_CARD]', content)
                
                anonymized.append({
                    'role': turn['role'],
                    'content': content,
                })
        
        return anonymized
    
    def export_to_jsonl(self, conversations: List[Dict], output_path: str):
        """Export conversations สำหรับ fine-tuning"""
        
        with open(output_path, 'w', encoding='utf-8') as f:
            for conv in conversations:
                # Format for instruction tuning
                training_example = {
                    "messages": [
                        {
                            "role": "system",
                            "content": "คุณคือผู้ช่วย AI ของร้านค้า ตอบคำถามลูกค้าด้วยภาษาไทยที่สุภาพ"
                        }
                    ] + conv['turns']
                }
                f.write(json.dumps(training_example, ensure_ascii=False) + '\n')
        
        print(f"Exported {len(conversations)} conversations to {output_path}")
```

---

## 14. Thai Language Specific Considerations

### 14.1 Thai Text Processing

```python
# ml/thai_nlp/thai_processor.py
"""
Thai text preprocessing สำหรับ LINE Bot NLP
"""

import re
from typing import List

class ThaiTextProcessor:
    """
    Preprocess Thai text สำหรับ ML models
    """
    
    def __init__(self):
        # ลองใช้ pythainlp ถ้ามี
        try:
            from pythainlp.tokenize import word_tokenize
            from pythainlp.corpus.common import thai_stopwords
            self.tokenizer = word_tokenize
            self.stop_words = set(thai_stopwords())
            self.use_pythainlp = True
        except ImportError:
            print("pythainlp not available, using basic tokenization")
            self.use_pythainlp = False
    
    def clean_text(self, text: str) -> str:
        """Clean Thai text"""
        # Remove HTML
        text = re.sub(r'<[^>]+>', '', text)
        # Remove URLs
        text = re.sub(r'https?://\S+', '', text)
        # Remove phone numbers
        text = re.sub(r'\b0[0-9]{8,9}\b', '', text)
        # Remove excessive whitespace
        text = re.sub(r'\s+', ' ', text).strip()
        # Remove emoji (optional)
        # text = text.encode('ascii', 'ignore').decode('ascii')
        return text
    
    def tokenize(self, text: str) -> List[str]:
        """Tokenize Thai text"""
        if self.use_pythainlp:
            return self.tokenizer(text, engine='newmm')
        else:
            # Fallback: character-based
            return list(text)
    
    def remove_stopwords(self, tokens: List[str]) -> List[str]:
        """Remove Thai stopwords"""
        if hasattr(self, 'stop_words'):
            return [t for t in tokens if t not in self.stop_words and t.strip()]
        return tokens
    
    def normalize_thai(self, text: str) -> str:
        """Normalize Thai characters"""
        # แก้ไข common typos ในภาษาไทย
        corrections = {
            'กรุณ': 'กรุณา',
            'ขอบคุณมากๆ': 'ขอบคุณมาก',
            'โอเค': 'โอเค',
            'โอเคๆ': 'โอเค',
        }
        
        for wrong, correct in corrections.items():
            text = text.replace(wrong, correct)
        
        return text
    
    def detect_language(self, text: str) -> str:
        """ตรวจสอบภาษาของข้อความ"""
        thai_chars = sum(1 for c in text if '฀' <= c <= '๿')
        english_chars = sum(1 for c in text if c.isalpha() and ord(c) < 128)
        
        if thai_chars > english_chars:
            return 'th'
        elif english_chars > 0:
            return 'en'
        return 'unknown'
    
    def preprocess_for_bert(self, text: str, max_length: int = 128) -> str:
        """Preprocess สำหรับ BERT-based models"""
        text = self.clean_text(text)
        text = self.normalize_thai(text)
        
        # Truncate ถ้าข้อความยาวเกิน
        if len(text) > max_length * 4:  # Rough char to token ratio for Thai
            text = text[:max_length * 4]
        
        return text
    
    def extract_entities(self, text: str) -> dict:
        """Extract entities จาก Thai text"""
        entities = {
            'phone_numbers': re.findall(r'\b0[0-9]{8,9}\b', text),
            'prices': re.findall(r'฿[\d,]+|[\d,]+\s*บาท', text),
            'order_ids': re.findall(r'(?:ออร์เดอร์|order|ORD)[-#]?\s*([A-Z0-9]+)', text, re.I),
            'dates': re.findall(r'\d{1,2}[/-]\d{1,2}[/-]\d{2,4}', text),
        }
        return {k: v for k, v in entities.items() if v}
```

---

## สรุปสมบูรณ์ Custom AI Models

Custom AI Models เป็นการลงทุนที่คุ้มค่าสำหรับ LINE Bot:

```
AI Strategy Roadmap:

Month 1-3: Foundation
├── RAG System + GPT-4o-mini
├── Basic intent classification
└── Cost: ~$1,000/month

Month 4-6: Optimization  
├── Fine-tuned Thai intent model
├── Spam detection
└── Cost: ~$800/month (save 20%)

Month 7-12: Custom Models
├── Fine-tuned Typhoon for domain
├── Local vLLM deployment
└── Cost: ~$2,500/month (but unlimited scale)

Year 2+: Full Custom
├── Domain-specific LLM
├── RLHF training loop
└── Cost: ~$3,000/month (full privacy, max performance)
```

Key Takeaways:
1. **เริ่มง่ายๆ**: RAG + API สำหรับ prototype
2. **Scale smart**: Fine-tune เมื่อ volume สูงขึ้น
3. **Thai language first**: เลือก model ที่รองรับภาษาไทยดี
4. **Monitor ต่อเนื่อง**: Track model performance ใน production
5. **Feedback loop**: เก็บ user feedback เพื่อปรับปรุง model

---

*จบ Part 85: Custom AI Models สำหรับ LINE Bots (ฉบับสมบูรณ์)*
