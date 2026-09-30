# Part 84: Machine Learning สำหรับ LINE Bots

## บทนำ

Machine Learning ช่วยให้ LINE Bot ให้บริการที่ personalized และ intelligent มากขึ้น บทนี้จะครอบคลุม ML Applications ที่สำคัญ ตั้งแต่ Recommendation Engine ไปจนถึง Churn Prediction โดยใช้ข้อมูลจาก LINE conversations

---

## 1. ML Architecture Overview

```
ML Pipeline for LINE Bot:

┌─────────────────────────────────────────────────────────────┐
│  Data Collection Layer                                       │
│  ├── LINE Webhook Events                                    │
│  ├── User Interactions                                      │
│  ├── Purchase History                                       │
│  └── Profile Data                                          │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│  Feature Engineering                                         │
│  ├── User Features (demographics, behavior)                 │
│  ├── Item Features (product attributes)                     │
│  ├── Interaction Features (time, context)                   │
│  └── Sequence Features (message patterns)                   │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│  Model Training (MLflow)                                     │
│  ├── Recommendation Models                                  │
│  ├── Churn Prediction                                       │
│  ├── Customer Segmentation                                  │
│  └── LTV Prediction                                         │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│  Model Serving                                              │
│  ├── FastAPI Inference Server                               │
│  ├── Feature Store (Redis)                                  │
│  └── Model Registry (MLflow)                                │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│  LINE Bot Integration                                        │
│  ├── Personalized Responses                                 │
│  ├── Product Recommendations                                │
│  └── Proactive Messaging                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Recommendation Engine

### 2.1 Collaborative Filtering

```python
# ml/recommendation/collaborative_filtering.py
import pandas as pd
import numpy as np
from sklearn.decomposition import TruncatedSVD
from sklearn.preprocessing import normalize
from scipy.sparse import csr_matrix
import mlflow
import mlflow.sklearn
import pickle
import redis
import json

class CollaborativeFilteringRecommender:
    """
    Matrix Factorization-based recommendation system
    ใช้ implicit feedback จาก LINE Bot interactions
    """
    
    def __init__(self, n_factors=50, n_iterations=100, regularization=0.01):
        self.n_factors = n_factors
        self.n_iterations = n_iterations
        self.regularization = regularization
        self.model = TruncatedSVD(n_components=n_factors, random_state=42)
        
        self.user_encoder = {}
        self.item_encoder = {}
        self.user_decoder = {}
        self.item_decoder = {}
        
        self.user_factors = None
        self.item_factors = None
    
    def build_interaction_matrix(self, interactions_df):
        """
        สร้าง User-Item interaction matrix จาก LINE bot data
        
        interactions_df columns: user_id, item_id, interaction_type, timestamp
        Interaction types: view, click, purchase
        """
        # กำหนด weights สำหรับแต่ละ interaction
        interaction_weights = {
            'view': 1.0,
            'click': 3.0,
            'add_to_cart': 5.0,
            'purchase': 10.0,
            'share': 7.0,
        }
        
        # คำนวณ weighted interaction score
        interactions_df['weight'] = interactions_df['interaction_type'].map(
            interaction_weights
        ).fillna(1.0)
        
        # Aggregate interactions
        agg_df = interactions_df.groupby(['user_id', 'item_id'])['weight'].sum().reset_index()
        
        # Encode users and items
        unique_users = agg_df['user_id'].unique()
        unique_items = agg_df['item_id'].unique()
        
        self.user_encoder = {u: i for i, u in enumerate(unique_users)}
        self.item_encoder = {it: i for i, it in enumerate(unique_items)}
        self.user_decoder = {i: u for u, i in self.user_encoder.items()}
        self.item_decoder = {i: it for it, i in self.item_encoder.items()}
        
        # Create sparse matrix
        user_indices = agg_df['user_id'].map(self.user_encoder).values
        item_indices = agg_df['item_id'].map(self.item_encoder).values
        weights = agg_df['weight'].values
        
        n_users = len(unique_users)
        n_items = len(unique_items)
        
        matrix = csr_matrix(
            (weights, (user_indices, item_indices)),
            shape=(n_users, n_items)
        )
        
        return matrix
    
    def train(self, interactions_df):
        """Train the recommendation model"""
        
        with mlflow.start_run(run_name="collaborative-filtering"):
            mlflow.log_params({
                'n_factors': self.n_factors,
                'n_iterations': self.n_iterations,
                'regularization': self.regularization,
            })
            
            # Build interaction matrix
            interaction_matrix = self.build_interaction_matrix(interactions_df)
            
            print(f"Matrix shape: {interaction_matrix.shape}")
            print(f"Non-zero entries: {interaction_matrix.nnz}")
            print(f"Sparsity: {1 - interaction_matrix.nnz / (interaction_matrix.shape[0] * interaction_matrix.shape[1]):.3f}")
            
            # Fit model
            self.user_factors = self.model.fit_transform(interaction_matrix)
            self.item_factors = self.model.components_.T
            
            # Normalize
            self.user_factors = normalize(self.user_factors)
            self.item_factors = normalize(self.item_factors)
            
            # Evaluate
            metrics = self.evaluate(interactions_df)
            mlflow.log_metrics(metrics)
            
            # Save model
            mlflow.sklearn.log_model(
                self.model,
                "collaborative_filter_model",
                registered_model_name="LineBot-Recommender"
            )
            
            print(f"Training complete. Metrics: {metrics}")
            
        return metrics
    
    def get_recommendations(self, user_id, n_recommendations=10, exclude_seen=True):
        """
        Get personalized product recommendations for a LINE user
        """
        if user_id not in self.user_encoder:
            # Cold start: ใช้ popularity-based recommendations
            return self.get_popular_items(n_recommendations)
        
        user_idx = self.user_encoder[user_id]
        user_vector = self.user_factors[user_idx]
        
        # Calculate scores ด้วย dot product
        scores = np.dot(self.item_factors, user_vector)
        
        if exclude_seen:
            # ลบ items ที่ user เคยเห็นแล้ว
            seen_items = self.get_seen_items(user_id)
            for item_id in seen_items:
                if item_id in self.item_encoder:
                    item_idx = self.item_encoder[item_id]
                    scores[item_idx] = -np.inf
        
        # Top-N recommendations
        top_indices = np.argsort(scores)[-n_recommendations:][::-1]
        
        recommendations = []
        for idx in top_indices:
            if scores[idx] > -np.inf:
                item_id = self.item_decoder[idx]
                recommendations.append({
                    'item_id': item_id,
                    'score': float(scores[idx]),
                })
        
        return recommendations
    
    def get_popular_items(self, n=10):
        """Fallback สำหรับ cold start"""
        # ดึงจาก feature store
        return []
    
    def get_seen_items(self, user_id):
        """ดึง items ที่ user เคยซื้อแล้ว"""
        return []
    
    def evaluate(self, test_interactions):
        """
        Evaluate model ด้วย Hit Rate และ NDCG
        """
        hit_rate_at_10 = 0
        ndcg_at_10 = 0
        n_users = 0
        
        for user_id in test_interactions['user_id'].unique()[:1000]:
            user_data = test_interactions[test_interactions['user_id'] == user_id]
            
            # แยก test purchases
            purchases = user_data[user_data['interaction_type'] == 'purchase']['item_id'].tolist()
            
            if not purchases:
                continue
            
            recs = self.get_recommendations(user_id, n_recommendations=10)
            rec_items = [r['item_id'] for r in recs]
            
            # Hit Rate@10
            hits = len(set(purchases) & set(rec_items))
            hit_rate_at_10 += 1 if hits > 0 else 0
            
            # NDCG@10
            for i, item in enumerate(rec_items):
                if item in purchases:
                    ndcg_at_10 += 1 / np.log2(i + 2)
            
            n_users += 1
        
        if n_users == 0:
            return {'hit_rate_at_10': 0, 'ndcg_at_10': 0}
        
        return {
            'hit_rate_at_10': hit_rate_at_10 / n_users,
            'ndcg_at_10': ndcg_at_10 / n_users,
        }


# Feature Store Integration
class RecommendationFeatureStore:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.ttl = 3600  # 1 hour cache
    
    def cache_recommendations(self, user_id, recommendations):
        key = f"recs:{user_id}"
        self.redis.setex(
            key,
            self.ttl,
            json.dumps(recommendations)
        )
    
    def get_cached_recommendations(self, user_id):
        key = f"recs:{user_id}"
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)
        return None
    
    def store_user_features(self, user_id, features):
        key = f"user_features:{user_id}"
        self.redis.setex(
            key,
            self.ttl,
            json.dumps(features)
        )
    
    def get_user_features(self, user_id):
        key = f"user_features:{user_id}"
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)
        return None
```

### 2.2 Content-Based Filtering

```python
# ml/recommendation/content_based.py
import numpy as np
import pandas as pd
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity
from sentence_transformers import SentenceTransformer

class ContentBasedRecommender:
    """
    Content-based filtering ใช้ product descriptions
    เหมาะสำหรับ cold start problem
    """
    
    def __init__(self, model_name='paraphrase-multilingual-MiniLM-L12-v2'):
        # Support ภาษาไทย
        self.sentence_model = SentenceTransformer(model_name)
        self.item_embeddings = {}
        self.item_metadata = {}
    
    def index_products(self, products_df):
        """
        Index product catalog
        products_df: id, name, description, category, tags
        """
        for _, row in products_df.iterrows():
            # รวม features เป็น text
            text = f"{row['name']} {row['description']} {row.get('tags', '')} {row.get('category', '')}"
            
            # คำนวณ embedding
            embedding = self.sentence_model.encode(text)
            
            self.item_embeddings[row['id']] = embedding
            self.item_metadata[row['id']] = {
                'name': row['name'],
                'price': row.get('price', 0),
                'category': row.get('category', ''),
                'image_url': row.get('image_url', ''),
            }
        
        print(f"Indexed {len(self.item_embeddings)} products")
    
    def get_similar_items(self, item_id, n=10):
        """หา items ที่คล้ายกัน"""
        if item_id not in self.item_embeddings:
            return []
        
        query_embedding = self.item_embeddings[item_id]
        
        similarities = {}
        for other_id, embedding in self.item_embeddings.items():
            if other_id != item_id:
                sim = cosine_similarity(
                    query_embedding.reshape(1, -1),
                    embedding.reshape(1, -1)
                )[0][0]
                similarities[other_id] = float(sim)
        
        top_items = sorted(similarities.items(), key=lambda x: x[1], reverse=True)[:n]
        
        return [
            {
                'item_id': item_id,
                'score': score,
                'metadata': self.item_metadata.get(item_id, {}),
            }
            for item_id, score in top_items
        ]
    
    def recommend_from_query(self, query_text, n=10):
        """แนะนำ products จาก natural language query (Thai support)"""
        query_embedding = self.sentence_model.encode(query_text)
        
        similarities = {}
        for item_id, embedding in self.item_embeddings.items():
            sim = cosine_similarity(
                query_embedding.reshape(1, -1),
                embedding.reshape(1, -1)
            )[0][0]
            similarities[item_id] = float(sim)
        
        top_items = sorted(similarities.items(), key=lambda x: x[1], reverse=True)[:n]
        
        return [
            {
                'item_id': item_id,
                'score': score,
                'metadata': self.item_metadata.get(item_id, {}),
            }
            for item_id, score in top_items
        ]
```

---

## 3. Churn Prediction

```python
# ml/churn/churn_predictor.py
import pandas as pd
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier, RandomForestClassifier
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import (
    classification_report, roc_auc_score, 
    precision_recall_curve, average_precision_score
)
import mlflow
import mlflow.sklearn
import shap

class ChurnPredictor:
    """
    Predict ว่า user จะ unfollow LINE OA หรือไม่
    """
    
    def __init__(self):
        self.model = GradientBoostingClassifier(
            n_estimators=200,
            max_depth=5,
            learning_rate=0.05,
            subsample=0.8,
            random_state=42,
        )
        self.scaler = StandardScaler()
        self.feature_names = []
    
    def prepare_features(self, user_df, events_df, purchases_df):
        """
        Feature engineering จาก LINE Bot data
        
        Features:
        - Recency: วันที่ส่ง message ล่าสุด
        - Frequency: จำนวน messages ต่อสัปดาห์
        - Monetary: ยอดซื้อรวม
        - Engagement: อัตราการ response
        - Session: เวลาเฉลี่ยต่อ session
        """
        features = []
        
        for user_id in user_df['user_id'].values:
            user_events = events_df[events_df['user_id'] == user_id]
            user_purchases = purchases_df[purchases_df['user_id'] == user_id] if purchases_df is not None else pd.DataFrame()
            
            now = pd.Timestamp.now()
            
            # Recency features
            last_message = user_events[user_events['event_type'] == 'message']['timestamp'].max()
            days_since_last_message = (now - pd.to_datetime(last_message)).days if pd.notna(last_message) else 999
            
            # Frequency features
            recent_events = user_events[
                user_events['timestamp'] >= (now - pd.Timedelta(days=30))
            ]
            messages_last_30d = len(recent_events[recent_events['event_type'] == 'message'])
            messages_last_7d = len(user_events[
                (user_events['event_type'] == 'message') &
                (user_events['timestamp'] >= (now - pd.Timedelta(days=7)))
            ])
            
            # Monetary features
            total_purchase = user_purchases['amount'].sum() if len(user_purchases) > 0 else 0
            purchase_count = len(user_purchases)
            avg_purchase_value = total_purchase / purchase_count if purchase_count > 0 else 0
            
            # Engagement features
            postbacks = len(user_events[user_events['event_type'] == 'postback'])
            engagement_rate = postbacks / max(messages_last_30d, 1)
            
            # Session features (approximate)
            if len(user_events) > 1:
                time_diffs = user_events['timestamp'].sort_values().diff().dt.total_seconds().dropna()
                session_gaps = time_diffs[time_diffs > 1800]  # 30 min gap = new session
                avg_session_duration = time_diffs[time_diffs <= 1800].sum() / max(len(session_gaps) + 1, 1)
            else:
                avg_session_duration = 0
            
            # Account age
            account_age_days = (now - pd.to_datetime(user_df[user_df['user_id'] == user_id]['created_at'].values[0])).days
            
            features.append({
                'user_id': user_id,
                'days_since_last_message': days_since_last_message,
                'messages_last_30d': messages_last_30d,
                'messages_last_7d': messages_last_7d,
                'total_purchase': total_purchase,
                'purchase_count': purchase_count,
                'avg_purchase_value': avg_purchase_value,
                'engagement_rate': engagement_rate,
                'postback_count': postbacks,
                'avg_session_duration_seconds': avg_session_duration,
                'account_age_days': account_age_days,
                'message_types_diversity': user_events['message_type'].nunique() if 'message_type' in user_events.columns else 0,
            })
        
        return pd.DataFrame(features)
    
    def train(self, features_df, labels_df):
        """
        Train churn prediction model
        
        labels_df: user_id, churned (1=churned, 0=retained)
        """
        
        with mlflow.start_run(run_name="churn-predictor"):
            # Merge features and labels
            df = features_df.merge(labels_df, on='user_id')
            
            self.feature_names = [c for c in features_df.columns if c != 'user_id']
            
            X = df[self.feature_names].fillna(0)
            y = df['churned']
            
            print(f"Dataset: {len(df)} users, {y.mean():.1%} churn rate")
            
            # Log dataset info
            mlflow.log_params({
                'n_samples': len(df),
                'churn_rate': float(y.mean()),
                'n_features': len(self.feature_names),
            })
            
            # Split
            X_train, X_test, y_train, y_test = train_test_split(
                X, y, test_size=0.2, random_state=42, stratify=y
            )
            
            # Scale
            X_train_scaled = self.scaler.fit_transform(X_train)
            X_test_scaled = self.scaler.transform(X_test)
            
            # Train
            self.model.fit(X_train_scaled, y_train)
            
            # Evaluate
            y_pred = self.model.predict(X_test_scaled)
            y_proba = self.model.predict_proba(X_test_scaled)[:, 1]
            
            auc_roc = roc_auc_score(y_test, y_proba)
            avg_precision = average_precision_score(y_test, y_proba)
            
            print(f"\nClassification Report:")
            print(classification_report(y_test, y_pred))
            print(f"AUC-ROC: {auc_roc:.4f}")
            print(f"Average Precision: {avg_precision:.4f}")
            
            mlflow.log_metrics({
                'auc_roc': auc_roc,
                'avg_precision': avg_precision,
            })
            
            # SHAP Feature Importance
            explainer = shap.TreeExplainer(self.model)
            shap_values = explainer.shap_values(X_test_scaled[:100])
            
            feature_importance = pd.DataFrame({
                'feature': self.feature_names,
                'importance': np.abs(shap_values).mean(0),
            }).sort_values('importance', ascending=False)
            
            print("\nTop Features:")
            print(feature_importance.head(10))
            
            # Log model
            mlflow.sklearn.log_model(
                self.model,
                "churn_model",
                registered_model_name="LineBot-ChurnPredictor"
            )
            
            return {
                'auc_roc': auc_roc,
                'avg_precision': avg_precision,
                'feature_importance': feature_importance.to_dict('records'),
            }
    
    def predict_churn_probability(self, user_features):
        """
        ทำนาย churn probability สำหรับ user
        """
        features = pd.DataFrame([user_features])[self.feature_names].fillna(0)
        features_scaled = self.scaler.transform(features)
        
        prob = self.model.predict_proba(features_scaled)[0][1]
        
        return {
            'churn_probability': float(prob),
            'risk_level': 'high' if prob > 0.7 else 'medium' if prob > 0.4 else 'low',
            'recommended_actions': self.get_recommended_actions(prob, user_features),
        }
    
    def get_recommended_actions(self, churn_prob, user_features):
        """
        แนะนำ actions เพื่อ retain user
        """
        actions = []
        
        if churn_prob > 0.7:
            actions.append({
                'action': 'send_personalized_offer',
                'priority': 'high',
                'message': 'ส่ง exclusive discount ให้ผู้ใช้ที่มีความเสี่ยงสูง',
            })
        
        if user_features.get('days_since_last_message', 0) > 7:
            actions.append({
                'action': 're_engagement_campaign',
                'priority': 'medium',
                'message': 'ส่ง re-engagement message ผ่าน broadcast',
            })
        
        if user_features.get('purchase_count', 0) == 0:
            actions.append({
                'action': 'first_purchase_incentive',
                'priority': 'medium',
                'message': 'ส่ง welcome coupon สำหรับการซื้อครั้งแรก',
            })
        
        return actions
    
    def batch_predict(self, features_df):
        """Predict churn สำหรับผู้ใช้จำนวนมาก"""
        X = features_df[self.feature_names].fillna(0)
        X_scaled = self.scaler.transform(X)
        
        probabilities = self.model.predict_proba(X_scaled)[:, 1]
        
        result = features_df[['user_id']].copy()
        result['churn_probability'] = probabilities
        result['risk_level'] = pd.cut(
            probabilities,
            bins=[0, 0.4, 0.7, 1.0],
            labels=['low', 'medium', 'high']
        )
        
        return result.sort_values('churn_probability', ascending=False)
```

---

## 4. Customer Segmentation

```python
# ml/segmentation/customer_segmenter.py
import pandas as pd
import numpy as np
from sklearn.cluster import KMeans
from sklearn.preprocessing import StandardScaler
from sklearn.decomposition import PCA
import matplotlib.pyplot as plt
import mlflow

class CustomerSegmenter:
    """
    RFM-based Customer Segmentation ด้วย K-Means
    """
    
    def __init__(self, n_segments=6):
        self.n_segments = n_segments
        self.kmeans = KMeans(n_clusters=n_segments, random_state=42, n_init=10)
        self.scaler = StandardScaler()
        self.segment_profiles = {}
    
    def calculate_rfm(self, events_df, purchases_df, reference_date=None):
        """
        คำนวณ RFM scores
        R: Recency (วันที่ซื้อล่าสุด)
        F: Frequency (ความถี่ในการซื้อ)
        M: Monetary (ยอดซื้อรวม)
        """
        if reference_date is None:
            reference_date = pd.Timestamp.now()
        
        rfm = purchases_df.groupby('user_id').agg({
            'timestamp': lambda x: (reference_date - pd.to_datetime(x.max())).days,
            'order_id': 'count',
            'amount': 'sum',
        }).reset_index()
        
        rfm.columns = ['user_id', 'recency', 'frequency', 'monetary']
        
        # Add engagement features from events
        engagement = events_df.groupby('user_id').agg({
            'event_type': 'count',
            'timestamp': lambda x: (reference_date - pd.to_datetime(x.max())).days,
        }).reset_index()
        
        engagement.columns = ['user_id', 'total_interactions', 'days_since_last_activity']
        
        rfm = rfm.merge(engagement, on='user_id', how='left')
        rfm = rfm.fillna(0)
        
        return rfm
    
    def segment_customers(self, rfm_df):
        """
        Segment customers ด้วย K-Means
        """
        
        with mlflow.start_run(run_name="customer-segmentation"):
            features = ['recency', 'frequency', 'monetary', 'total_interactions']
            X = rfm_df[features]
            
            # Log transform สำหรับ skewed distributions
            X_log = X.copy()
            X_log['monetary'] = np.log1p(X_log['monetary'])
            X_log['frequency'] = np.log1p(X_log['frequency'])
            X_log['total_interactions'] = np.log1p(X_log['total_interactions'])
            
            # Scale
            X_scaled = self.scaler.fit_transform(X_log)
            
            # Find optimal k ด้วย elbow method
            if self.n_segments is None:
                inertias = []
                for k in range(2, 11):
                    km = KMeans(n_clusters=k, random_state=42, n_init=10)
                    km.fit(X_scaled)
                    inertias.append(km.inertia_)
                self.n_segments = self.find_elbow(inertias) + 2
            
            # Fit K-Means
            self.kmeans.n_clusters = self.n_segments
            rfm_df['segment'] = self.kmeans.fit_predict(X_scaled)
            
            # Analyze segments
            self.segment_profiles = self.analyze_segments(rfm_df, features)
            
            mlflow.log_params({
                'n_segments': self.n_segments,
                'features': ','.join(features),
            })
            
            print("Customer Segments:")
            for seg_id, profile in self.segment_profiles.items():
                print(f"\nSegment {seg_id}: {profile['name']}")
                print(f"  Size: {profile['size']} users ({profile['size_pct']:.1%})")
                print(f"  Avg Recency: {profile['avg_recency']:.0f} days")
                print(f"  Avg Frequency: {profile['avg_frequency']:.1f} orders")
                print(f"  Avg Monetary: ฿{profile['avg_monetary']:,.0f}")
            
            return rfm_df
    
    def analyze_segments(self, rfm_df, features):
        """วิเคราะห์และตั้งชื่อ segments"""
        profiles = {}
        total = len(rfm_df)
        
        for seg_id in range(self.n_segments):
            seg_data = rfm_df[rfm_df['segment'] == seg_id]
            
            profile = {
                'size': len(seg_data),
                'size_pct': len(seg_data) / total,
                'avg_recency': seg_data['recency'].mean(),
                'avg_frequency': seg_data['frequency'].mean(),
                'avg_monetary': seg_data['monetary'].mean(),
                'total_revenue': seg_data['monetary'].sum(),
            }
            
            # ตั้งชื่อ segment อัตโนมัติ
            profile['name'] = self.name_segment(profile)
            profile['strategy'] = self.get_segment_strategy(profile)
            
            profiles[seg_id] = profile
        
        return profiles
    
    def name_segment(self, profile):
        """ตั้งชื่อ segment ตาม RFM pattern"""
        r = profile['avg_recency']
        f = profile['avg_frequency']
        m = profile['avg_monetary']
        
        if r < 30 and f > 5 and m > 5000:
            return "Champions"  # ซื้อบ่อย ซื้อล่าสุด ยอดสูง
        elif r < 60 and f > 3:
            return "Loyal Customers"
        elif r < 30 and f == 1:
            return "Recent Customers"
        elif r > 90 and f > 3:
            return "At Risk"  # เคยดีแต่หายไป
        elif r > 180:
            return "Hibernating"
        elif f == 0:
            return "Potential Loyalists"
        else:
            return "Needs Attention"
    
    def get_segment_strategy(self, profile):
        """กลยุทธ์สำหรับแต่ละ segment"""
        strategies = {
            "Champions": {
                "message": "ให้ rewards พิเศษ, เชิญ referral program",
                "frequency": "weekly",
                "offers": ["early_access", "vip_discount", "free_shipping"],
            },
            "Loyal Customers": {
                "message": "Loyalty program, personalized recommendations",
                "frequency": "bi-weekly",
                "offers": ["loyalty_points", "bundle_deals"],
            },
            "At Risk": {
                "message": "Win-back campaign ด้วย special offer",
                "frequency": "immediate",
                "offers": ["comeback_discount", "personalized_message"],
            },
            "Hibernating": {
                "message": "Re-engagement campaign",
                "frequency": "monthly",
                "offers": ["big_discount", "new_products"],
            },
        }
        
        return strategies.get(profile.get('name', ''), {
            "message": "Standard engagement",
            "frequency": "monthly",
            "offers": ["general_discount"],
        })
    
    def find_elbow(self, inertias):
        """หา optimal k จาก elbow point"""
        diffs = np.diff(inertias)
        diffs2 = np.diff(diffs)
        return diffs2.argmax() + 1
```

---

## 5. Lifetime Value Prediction

```python
# ml/ltv/ltv_predictor.py
import pandas as pd
import numpy as np
from sklearn.ensemble import RandomForestRegressor
from lifetimes import BetaGeoFitter, GammaGammaFitter
from lifetimes.utils import summary_data_from_transaction_data
import mlflow

class LTVPredictor:
    """
    Customer Lifetime Value Prediction
    ใช้ BG/NBD Model + Gamma-Gamma Model (Pareto/NBD approach)
    """
    
    def __init__(self, prediction_days=365):
        self.prediction_days = prediction_days
        self.bgf = BetaGeoFitter(penalizer_coef=0.0)
        self.ggf = GammaGammaFitter(penalizer_coef=0.0)
    
    def prepare_data(self, transactions_df, observation_period_end=None):
        """
        เตรียมข้อมูลสำหรับ BG/NBD Model
        
        transactions_df: user_id, order_id, timestamp, amount
        """
        if observation_period_end is None:
            observation_period_end = transactions_df['timestamp'].max()
        
        # สร้าง RFM summary
        summary = summary_data_from_transaction_data(
            transactions_df,
            customer_id_col='user_id',
            datetime_col='timestamp',
            monetary_value_col='amount',
            observation_period_end=observation_period_end,
            freq='D',
        )
        
        return summary
    
    def train(self, transactions_df):
        """Train LTV prediction models"""
        
        with mlflow.start_run(run_name="ltv-predictor"):
            summary = self.prepare_data(transactions_df)
            
            print(f"Training on {len(summary)} customers")
            print(f"Active customers: {(summary['frequency'] > 0).sum()}")
            
            # Train BG/NBD Model (ทำนายจำนวนการซื้อในอนาคต)
            print("Training BG/NBD model...")
            self.bgf.fit(
                summary['frequency'],
                summary['recency'],
                summary['T'],
                verbose=True,
            )
            
            # Train Gamma-Gamma Model (ทำนาย expected purchase value)
            # ใช้เฉพาะ customers ที่ซื้อมากกว่า 1 ครั้ง
            returning_customers = summary[summary['frequency'] > 0]
            
            print("Training Gamma-Gamma model...")
            self.ggf.fit(
                returning_customers['frequency'],
                returning_customers['monetary_value'],
            )
            
            # คำนวณ LTV
            summary['predicted_purchases'] = self.bgf.conditional_expected_number_of_purchases_up_to_time(
                self.prediction_days,
                summary['frequency'],
                summary['recency'],
                summary['T'],
            )
            
            summary['expected_avg_purchase'] = self.ggf.conditional_expected_average_profit(
                summary['frequency'],
                summary['monetary_value'],
            )
            
            summary['ltv'] = (
                summary['predicted_purchases'] * 
                summary['expected_avg_purchase']
            )
            
            # Log metrics
            mlflow.log_metrics({
                'avg_predicted_ltv': float(summary['ltv'].mean()),
                'median_predicted_ltv': float(summary['ltv'].median()),
                'total_predicted_revenue': float(summary['ltv'].sum()),
            })
            
            print(f"\nLTV Statistics:")
            print(f"Average LTV: ฿{summary['ltv'].mean():,.0f}")
            print(f"Median LTV: ฿{summary['ltv'].median():,.0f}")
            print(f"Total predicted revenue: ฿{summary['ltv'].sum():,.0f}")
            
            return summary
    
    def predict_user_ltv(self, user_id, user_frequency, user_recency, user_T, user_avg_purchase):
        """ทำนาย LTV สำหรับ user คนเดียว"""
        
        predicted_purchases = self.bgf.conditional_expected_number_of_purchases_up_to_time(
            self.prediction_days,
            user_frequency,
            user_recency,
            user_T,
        )
        
        expected_purchase_value = self.ggf.conditional_expected_average_profit(
            user_frequency,
            user_avg_purchase,
        ) if user_frequency > 0 else user_avg_purchase
        
        ltv = predicted_purchases * expected_purchase_value
        
        return {
            'user_id': user_id,
            'predicted_purchases': float(predicted_purchases),
            'expected_purchase_value': float(expected_purchase_value),
            'ltv': float(ltv),
            'ltv_tier': self.get_ltv_tier(ltv),
        }
    
    def get_ltv_tier(self, ltv):
        if ltv > 10000:
            return 'platinum'
        elif ltv > 5000:
            return 'gold'
        elif ltv > 1000:
            return 'silver'
        else:
            return 'bronze'
    
    def get_high_value_users(self, summary_df, top_n=100):
        """ดึง users ที่มี LTV สูงสุด"""
        return summary_df.nlargest(top_n, 'ltv')[
            ['user_id', 'ltv', 'predicted_purchases', 'expected_avg_purchase']
        ]
```

---

## 6. Spam Detection

```python
# ml/spam/spam_detector.py
import pandas as pd
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.pipeline import Pipeline
import re
import mlflow

class SpamDetector:
    """
    ตรวจจับ spam messages ใน LINE Bot
    """
    
    def __init__(self):
        self.pipeline = None
        self.spam_keywords = self.load_spam_keywords()
    
    def load_spam_keywords(self):
        return [
            # Thai spam keywords
            'รวยเร็ว', 'เงินด่วน', 'กู้เงิน', 'แทงหวย', 'คาสิโน',
            'ลงทุนได้กำไร', 'ไม่ต้องทำงาน', 'รายได้พิเศษ',
            # Common spam patterns
            'click here', 'free money', 'you won', 'congratulations',
            'act now', 'limited time',
        ]
    
    def extract_features(self, text):
        """Extract features สำหรับ spam detection"""
        features = {
            'has_url': bool(re.search(r'https?://\S+', text)),
            'url_count': len(re.findall(r'https?://\S+', text)),
            'has_phone': bool(re.search(r'\b0[0-9]{8,9}\b', text)),
            'has_email': bool(re.search(r'\S+@\S+\.\S+', text)),
            'caps_ratio': sum(c.isupper() for c in text) / max(len(text), 1),
            'text_length': len(text),
            'exclamation_count': text.count('!'),
            'has_spam_keyword': any(kw.lower() in text.lower() for kw in self.spam_keywords),
            'number_count': len(re.findall(r'\d+', text)),
        }
        return features
    
    def prepare_features(self, texts_df):
        """Prepare feature matrix"""
        feature_rows = []
        for _, row in texts_df.iterrows():
            features = self.extract_features(row['text'])
            feature_rows.append(features)
        return pd.DataFrame(feature_rows)
    
    def train(self, training_data):
        """
        training_data: DataFrame with 'text' and 'is_spam' columns
        """
        
        with mlflow.start_run(run_name="spam-detector"):
            X_features = self.prepare_features(training_data)
            y = training_data['is_spam']
            
            # TF-IDF for text features
            tfidf = TfidfVectorizer(
                max_features=5000,
                ngram_range=(1, 2),
                analyzer='char_wb',  # Character n-grams สำหรับภาษาไทย
            )
            
            X_tfidf = tfidf.fit_transform(training_data['text'])
            
            # Combine features
            import scipy.sparse as sp
            X_combined = sp.hstack([
                X_tfidf,
                sp.csr_matrix(X_features.values),
            ])
            
            # Train model
            model = GradientBoostingClassifier(
                n_estimators=100,
                learning_rate=0.1,
                random_state=42,
            )
            model.fit(X_combined, y)
            
            self.pipeline = {
                'tfidf': tfidf,
                'model': model,
            }
            
            mlflow.sklearn.log_model(model, "spam_detector")
            
        return self
    
    def predict(self, text):
        """ทำนายว่าเป็น spam หรือไม่"""
        if not self.pipeline:
            # Rule-based fallback
            return self.rule_based_check(text)
        
        import scipy.sparse as sp
        
        features = pd.DataFrame([self.extract_features(text)])
        tfidf_features = self.pipeline['tfidf'].transform([text])
        
        X_combined = sp.hstack([
            tfidf_features,
            sp.csr_matrix(features.values),
        ])
        
        prob = self.pipeline['model'].predict_proba(X_combined)[0][1]
        
        return {
            'is_spam': prob > 0.7,
            'spam_probability': float(prob),
            'features': {
                'has_url': bool(re.search(r'https?://\S+', text)),
                'has_spam_keyword': any(kw.lower() in text.lower() for kw in self.spam_keywords),
            },
        }
    
    def rule_based_check(self, text):
        """Rule-based spam detection สำหรับ fallback"""
        spam_signals = 0
        
        if any(kw.lower() in text.lower() for kw in self.spam_keywords):
            spam_signals += 2
        if len(re.findall(r'https?://\S+', text)) > 2:
            spam_signals += 1
        if text.count('!') > 3:
            spam_signals += 1
        if len(text) > 500:
            spam_signals += 1
        
        return {
            'is_spam': spam_signals >= 3,
            'spam_probability': min(spam_signals * 0.2, 1.0),
        }
```

---

## 7. MLflow Experiment Tracking

```python
# ml/tracking/experiment_manager.py
import mlflow
from mlflow.tracking import MlflowClient

class ExperimentManager:
    """จัดการ ML experiments ด้วย MLflow"""
    
    def __init__(self, tracking_uri=None):
        if tracking_uri:
            mlflow.set_tracking_uri(tracking_uri)
        self.client = MlflowClient()
    
    def setup_experiment(self, name):
        """สร้างหรือดึง experiment"""
        experiment = mlflow.get_experiment_by_name(name)
        if experiment is None:
            experiment_id = mlflow.create_experiment(
                name,
                tags={
                    'project': 'line-bot',
                    'team': 'ml',
                }
            )
        else:
            experiment_id = experiment.experiment_id
        
        mlflow.set_experiment(name)
        return experiment_id
    
    def get_best_model(self, experiment_name, metric='auc_roc'):
        """ดึง model ที่ดีที่สุดจาก experiment"""
        experiment = mlflow.get_experiment_by_name(experiment_name)
        if not experiment:
            return None
        
        runs = self.client.search_runs(
            experiment_ids=[experiment.experiment_id],
            order_by=[f"metrics.{metric} DESC"],
            max_results=1,
        )
        
        if not runs:
            return None
        
        best_run = runs[0]
        return {
            'run_id': best_run.info.run_id,
            'metric': best_run.data.metrics.get(metric),
            'model_uri': f"runs:/{best_run.info.run_id}/model",
            'params': best_run.data.params,
        }
    
    def promote_to_production(self, model_name, version):
        """Promote model version ไปยัง production"""
        self.client.transition_model_version_stage(
            name=model_name,
            version=version,
            stage="Production",
            archive_existing_versions=True,
        )
        print(f"Model {model_name} v{version} promoted to Production")
    
    def load_production_model(self, model_name):
        """Load production model"""
        model_uri = f"models:/{model_name}/Production"
        return mlflow.pyfunc.load_model(model_uri)
    
    def compare_models(self, experiment_name, metrics=['auc_roc', 'avg_precision']):
        """เปรียบเทียบ model versions"""
        experiment = mlflow.get_experiment_by_name(experiment_name)
        
        runs = self.client.search_runs(
            experiment_ids=[experiment.experiment_id],
            order_by=["start_time DESC"],
            max_results=10,
        )
        
        comparison = []
        for run in runs:
            row = {
                'run_id': run.info.run_id[:8],
                'start_time': run.info.start_time,
                'status': run.info.status,
            }
            for metric in metrics:
                row[metric] = run.data.metrics.get(metric, 'N/A')
            comparison.append(row)
        
        return pd.DataFrame(comparison)
```

---

## 8. A/B Testing ML Models

```javascript
// src/ml/ABTestService.js
class MLABTestService {
  constructor({ modelRegistry, analyticsService }) {
    this.modelRegistry = modelRegistry;
    this.analyticsService = analyticsService;
    
    this.experiments = new Map();
  }
  
  createExperiment(experimentId, { models, trafficSplit }) {
    this.experiments.set(experimentId, {
      models,
      trafficSplit,
      startTime: new Date().toISOString(),
      metrics: {},
    });
  }
  
  async getModelForUser(experimentId, userId) {
    const experiment = this.experiments.get(experimentId);
    if (!experiment) return null;
    
    // Consistent hashing สำหรับ user assignment
    const hash = this.hashUserId(userId);
    const bucket = hash % 100;
    
    let cumulative = 0;
    for (const [modelName, traffic] of Object.entries(experiment.trafficSplit)) {
      cumulative += traffic;
      if (bucket < cumulative) {
        // Track assignment
        await this.analyticsService.track(userId, 'ml_experiment_assigned', {
          experimentId,
          model: modelName,
        });
        
        return {
          modelName,
          model: await this.modelRegistry.getModel(modelName),
        };
      }
    }
    
    return null;
  }
  
  hashUserId(userId) {
    let hash = 0;
    for (let i = 0; i < userId.length; i++) {
      const char = userId.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash;
    }
    return Math.abs(hash);
  }
  
  async getExperimentResults(experimentId, startDate, endDate) {
    const results = await this.analyticsService.getABTestResults(
      experimentId,
      startDate,
      endDate
    );
    
    return this.calculateStatisticalSignificance(results);
  }
  
  calculateStatisticalSignificance(results) {
    // T-test สำหรับ continuous metrics
    // Z-test สำหรับ conversion rates
    return results.map(variant => ({
      ...variant,
      significance: variant.pValue < 0.05 ? 'significant' : 'not significant',
    }));
  }
}

module.exports = MLABTestService;
```

---

## 9. Complete ML Pipeline

```python
# ml/pipeline/training_pipeline.py

import pandas as pd
from datetime import datetime, timedelta
from sqlalchemy import create_engine
import schedule
import time

class MLTrainingPipeline:
    """
    Automated ML training pipeline
    """
    
    def __init__(self, config):
        self.config = config
        self.db_engine = create_engine(config['database_url'])
        self.mlflow_uri = config.get('mlflow_uri', 'http://mlflow:5000')
        
        # Models
        self.recommender = CollaborativeFilteringRecommender()
        self.churn_predictor = ChurnPredictor()
        self.segmenter = CustomerSegmenter()
        self.ltv_predictor = LTVPredictor()
    
    def extract_data(self, days_back=90):
        """Extract training data จาก database"""
        cutoff = datetime.now() - timedelta(days=days_back)
        
        events_df = pd.read_sql(
            f"SELECT * FROM events WHERE created_at >= '{cutoff}'",
            self.db_engine
        )
        
        purchases_df = pd.read_sql(
            f"SELECT * FROM purchases WHERE created_at >= '{cutoff}'",
            self.db_engine
        )
        
        users_df = pd.read_sql("SELECT * FROM users", self.db_engine)
        
        return events_df, purchases_df, users_df
    
    def run_full_pipeline(self):
        """Run full ML training pipeline"""
        print(f"Starting ML pipeline at {datetime.now()}")
        
        # 1. Extract data
        print("Extracting data...")
        events_df, purchases_df, users_df = self.extract_data()
        
        # 2. Train Recommender
        print("Training Recommender...")
        self.recommender.train(
            pd.concat([
                events_df[['user_id', 'item_id', 'event_type', 'timestamp']],
                purchases_df[['user_id', 'item_id']].assign(event_type='purchase', timestamp=purchases_df['created_at']),
            ], ignore_index=True).dropna(subset=['item_id'])
        )
        
        # 3. Train Churn Predictor
        print("Training Churn Predictor...")
        features_df = self.churn_predictor.prepare_features(
            users_df, events_df, purchases_df
        )
        
        # Label: user ที่ไม่ active ใน 30 วันล่าสุด = churned
        cutoff_30d = datetime.now() - timedelta(days=30)
        active_users = events_df[
            events_df['timestamp'] >= cutoff_30d.isoformat()
        ]['user_id'].unique()
        
        labels_df = users_df[['user_id']].copy()
        labels_df['churned'] = (~labels_df['user_id'].isin(active_users)).astype(int)
        
        self.churn_predictor.train(features_df, labels_df)
        
        # 4. Customer Segmentation
        print("Running Customer Segmentation...")
        rfm_df = self.segmenter.calculate_rfm(events_df, purchases_df)
        segmented_df = self.segmenter.segment_customers(rfm_df)
        
        # Save segments to DB
        segmented_df[['user_id', 'segment']].to_sql(
            'customer_segments',
            self.db_engine,
            if_exists='replace',
            index=False,
        )
        
        # 5. LTV Prediction
        print("Training LTV Predictor...")
        ltv_summary = self.ltv_predictor.train(purchases_df)
        
        ltv_summary[['user_id', 'ltv', 'predicted_purchases']].to_sql(
            'user_ltv',
            self.db_engine,
            if_exists='replace',
            index=False,
        )
        
        print(f"ML pipeline completed at {datetime.now()}")
        
        return {
            'status': 'success',
            'timestamp': datetime.now().isoformat(),
        }
    
    def schedule_training(self):
        """Schedule weekly training"""
        schedule.every().sunday.at("02:00").do(self.run_full_pipeline)
        
        print("ML training scheduled for every Sunday at 02:00")
        
        while True:
            schedule.run_pending()
            time.sleep(60)


# Training script
if __name__ == '__main__':
    import os
    
    config = {
        'database_url': os.environ['DATABASE_URL'],
        'mlflow_uri': os.environ.get('MLFLOW_TRACKING_URI', 'http://mlflow:5000'),
    }
    
    pipeline = MLTrainingPipeline(config)
    pipeline.run_full_pipeline()
```

---

## 10. Cost Analysis

```
ML Infrastructure Cost:

┌──────────────────────────┬──────────┬──────────┬──────────┐
│  Component               │  Small   │  Medium  │  Large   │
├──────────────────────────┼──────────┼──────────┼──────────┤
│  Training (weekly)       │          │          │          │
│  ├── GPU instance (p3.2xl│   $0     │  $150    │  $600    │
│  │   8 hrs/week)         │  (CPU)   │          │          │
│  └── Training data store │  $5      │  $50     │  $200    │
├──────────────────────────┼──────────┼──────────┼──────────┤
│  Serving (24/7)          │          │          │          │
│  ├── Inference API       │  $50     │  $200    │  $800    │
│  ├── Feature Store (Redis│  $50     │  $200    │  $800    │
│  └── MLflow server       │  $30     │  $100    │  $300    │
├──────────────────────────┼──────────┼──────────┼──────────┤
│  TOTAL/month             │  $135    │  $700    │  $2,700  │
└──────────────────────────┴──────────┴──────────┴──────────┘

ML ROI Analysis:
├── Recommendation Engine: +15-25% revenue lift
├── Churn Reduction: Save 5-10% of at-risk users
├── Personalization: +10-20% engagement rate
└── LTV Optimization: 20-30% better targeting ROI
```

---

## สรุป

ML สำหรับ LINE Bot ช่วยเพิ่ม value ในหลายด้าน:

1. **Recommendation Engine**: เพิ่ม revenue ด้วย personalized products
2. **Churn Prediction**: ลดการสูญเสีย users ด้วย proactive intervention
3. **Customer Segmentation**: วางกลยุทธ์ที่เหมาะสมกับแต่ละกลุ่ม
4. **LTV Prediction**: Focus resource ไปยัง high-value customers
5. **Spam Detection**: รักษาคุณภาพ customer experience

MLflow ช่วยในการ track experiments, version models, และ deploy อย่างเป็นระบบ

---

*จบ Part 84: Machine Learning สำหรับ LINE Bots*

---

## 11. Content Personalization

### 11.1 Personalization Engine

```python
# ml/personalization/content_personalizer.py
import pandas as pd
import numpy as np
from typing import List, Dict, Optional

class ContentPersonalizer:
    """
    Personalize LINE Bot content based on user behavior and segment
    """
    
    def __init__(self, recommender, segmenter, user_features_store):
        self.recommender = recommender
        self.segmenter = segmenter
        self.user_features = user_features_store
    
    def get_personalized_rich_menu(self, user_id: str) -> Dict:
        """
        เลือก Rich Menu ที่เหมาะกับ user
        """
        # Get user segment
        user_features = self.user_features.get(user_id)
        
        if not user_features:
            return self.get_default_rich_menu()
        
        segment = user_features.get('segment', 'unknown')
        purchase_count = user_features.get('purchase_count', 0)
        ltv = user_features.get('ltv', 0)
        
        # VIP users ได้ special menu
        if ltv > 10000 or purchase_count > 10:
            return self.get_vip_rich_menu()
        
        # First-time buyers
        if purchase_count == 0:
            return self.get_new_user_rich_menu()
        
        # Segment-based menus
        segment_menus = {
            'Champions': self.get_champions_rich_menu,
            'Loyal Customers': self.get_loyal_rich_menu,
            'At Risk': self.get_retention_rich_menu,
            'Hibernating': self.get_winback_rich_menu,
        }
        
        menu_fn = segment_menus.get(segment, self.get_default_rich_menu)
        return menu_fn()
    
    def get_personalized_broadcast_content(self, user_id: str, campaign_type: str) -> Dict:
        """
        สร้าง personalized content สำหรับ broadcast
        """
        user_features = self.user_features.get(user_id)
        recommendations = self.recommender.get_recommendations(user_id, n_recommendations=3)
        
        if campaign_type == 'product_recommendation':
            return self.build_recommendation_message(user_features, recommendations)
        elif campaign_type == 'winback':
            return self.build_winback_message(user_features)
        elif campaign_type == 'loyalty':
            return self.build_loyalty_message(user_features)
        
        return self.build_generic_message(user_features)
    
    def build_recommendation_message(self, user_features, recommendations):
        """สร้าง Flex Message สำหรับ product recommendations"""
        
        if not recommendations:
            return None
        
        name = user_features.get('display_name', 'คุณลูกค้า') if user_features else 'คุณลูกค้า'
        
        product_bubbles = []
        for rec in recommendations[:3]:
            product = rec.get('metadata', {})
            bubble = {
                "type": "bubble",
                "size": "micro",
                "hero": {
                    "type": "image",
                    "url": product.get('image_url', 'https://example.com/product.jpg'),
                    "size": "full",
                    "aspectRatio": "4:3",
                    "aspectMode": "cover"
                },
                "body": {
                    "type": "box",
                    "layout": "vertical",
                    "contents": [
                        {
                            "type": "text",
                            "text": product.get('name', 'สินค้า'),
                            "size": "sm",
                            "wrap": True
                        },
                        {
                            "type": "text",
                            "text": f"฿{product.get('price', 0):,.0f}",
                            "size": "lg",
                            "weight": "bold",
                            "color": "#FF0000"
                        }
                    ]
                },
                "footer": {
                    "type": "box",
                    "layout": "vertical",
                    "contents": [{
                        "type": "button",
                        "action": {
                            "type": "postback",
                            "label": "ดูรายละเอียด",
                            "data": f"action=view_product&product_id={product.get('id', '')}"
                        },
                        "style": "primary",
                        "height": "sm"
                    }]
                }
            }
            product_bubbles.append(bubble)
        
        return {
            "type": "flex",
            "altText": f"สินค้าแนะนำสำหรับ {name}",
            "contents": {
                "type": "carousel",
                "contents": product_bubbles
            }
        }
    
    def get_default_rich_menu(self):
        return {"richMenuId": "richmenu-default"}
    
    def get_vip_rich_menu(self):
        return {"richMenuId": "richmenu-vip"}
    
    def get_new_user_rich_menu(self):
        return {"richMenuId": "richmenu-new-user"}
    
    def get_loyal_rich_menu(self):
        return {"richMenuId": "richmenu-loyal"}
    
    def get_champions_rich_menu(self):
        return {"richMenuId": "richmenu-champions"}
    
    def get_retention_rich_menu(self):
        return {"richMenuId": "richmenu-retention"}
    
    def get_winback_rich_menu(self):
        return {"richMenuId": "richmenu-winback"}
    
    def build_winback_message(self, user_features):
        name = user_features.get('display_name', 'คุณลูกค้า') if user_features else 'คุณลูกค้า'
        return {
            "type": "flex",
            "altText": f"เราคิดถึงคุณ {name}!",
            "contents": {
                "type": "bubble",
                "body": {
                    "type": "box",
                    "layout": "vertical",
                    "contents": [
                        {"type": "text", "text": f"สวัสดีคุณ {name}!", "weight": "bold", "size": "xl"},
                        {"type": "text", "text": "เราไม่ได้เจอกันนานแล้ว! มาช็อปปิ้งด้วยกันไหม?", "wrap": True},
                        {"type": "text", "text": "รับส่วนลด 20% สำหรับการสั่งซื้อครั้งต่อไป", "color": "#FF0000", "weight": "bold"}
                    ]
                },
                "footer": {
                    "type": "box",
                    "layout": "vertical",
                    "contents": [{
                        "type": "button",
                        "action": {"type": "uri", "label": "ช็อปเลย!", "uri": "https://shop.example.com"},
                        "style": "primary"
                    }]
                }
            }
        }
    
    def build_loyalty_message(self, user_features):
        points = user_features.get('loyalty_points', 0) if user_features else 0
        return {
            "type": "text",
            "text": f"คุณมี {points} คะแนนสะสม! แลกรับของรางวัลได้เลยครับ"
        }
    
    def build_generic_message(self, user_features):
        return {
            "type": "text",
            "text": "สวัสดีครับ! มีสินค้าใหม่มาแนะนำ มาดูกันได้เลยนะครับ"
        }
```

---

## 12. Model Monitoring in Production

```python
# ml/monitoring/model_monitor.py
import pandas as pd
import numpy as np
from datetime import datetime, timedelta
from typing import Dict, List

class ModelDriftDetector:
    """
    ตรวจจับ data drift และ concept drift ใน ML models
    """
    
    def __init__(self, reference_data: pd.DataFrame):
        """
        reference_data: training data distribution เป็น baseline
        """
        self.reference_stats = self.compute_statistics(reference_data)
        self.drift_threshold = 0.1  # 10% drift threshold
    
    def compute_statistics(self, df: pd.DataFrame) -> Dict:
        """คำนวณ statistics ของ data"""
        stats = {}
        
        for col in df.select_dtypes(include=[np.number]).columns:
            stats[col] = {
                'mean': float(df[col].mean()),
                'std': float(df[col].std()),
                'min': float(df[col].min()),
                'max': float(df[col].max()),
                'p25': float(df[col].quantile(0.25)),
                'p50': float(df[col].quantile(0.50)),
                'p75': float(df[col].quantile(0.75)),
            }
        
        return stats
    
    def detect_drift(self, current_data: pd.DataFrame) -> Dict:
        """
        ตรวจจับ drift ระหว่าง reference และ current data
        ใช้ Population Stability Index (PSI)
        """
        current_stats = self.compute_statistics(current_data)
        drift_results = {}
        
        for col in self.reference_stats:
            if col not in current_stats:
                continue
            
            ref = self.reference_stats[col]
            curr = current_stats[col]
            
            # Normalized Mean Shift
            if ref['std'] > 0:
                mean_shift = abs(curr['mean'] - ref['mean']) / ref['std']
            else:
                mean_shift = 0
            
            # PSI calculation (simplified)
            psi = self.calculate_psi(
                [ref['p25'], ref['p50'], ref['p75']],
                [curr['p25'], curr['p50'], curr['p75']]
            )
            
            drift_results[col] = {
                'mean_shift': float(mean_shift),
                'psi': float(psi),
                'has_drift': psi > 0.2 or mean_shift > 2.0,
                'severity': 'high' if psi > 0.25 else 'medium' if psi > 0.1 else 'low',
            }
        
        return drift_results
    
    def calculate_psi(self, reference_bins, current_bins):
        """Calculate Population Stability Index"""
        psi = 0
        
        for ref_val, curr_val in zip(reference_bins, current_bins):
            if ref_val > 0 and curr_val > 0:
                psi += (curr_val - ref_val) * np.log(curr_val / ref_val)
        
        return abs(psi)
    
    def check_prediction_drift(self, predictions_df: pd.DataFrame) -> Dict:
        """
        ตรวจสอบ model prediction distribution drift
        """
        if len(predictions_df) < 100:
            return {'status': 'insufficient_data'}
        
        churn_rate = predictions_df['churn_probability'].mean()
        
        # Alert ถ้า predicted churn rate เปลี่ยนมากผิดปกติ
        baseline_churn_rate = 0.15  # 15% baseline
        
        if abs(churn_rate - baseline_churn_rate) > 0.1:
            return {
                'status': 'drift_detected',
                'current_churn_rate': float(churn_rate),
                'baseline_churn_rate': baseline_churn_rate,
                'deviation': float(abs(churn_rate - baseline_churn_rate)),
                'action': 'retrain_model',
            }
        
        return {
            'status': 'normal',
            'current_churn_rate': float(churn_rate),
        }


class MLMetricsCollector:
    """Collect ML metrics สำหรับ Prometheus"""
    
    def __init__(self):
        from prometheus_client import Counter, Histogram, Gauge
        
        self.inference_counter = Counter(
            'ml_inference_total',
            'Total ML inference calls',
            ['model_name', 'status']
        )
        
        self.inference_latency = Histogram(
            'ml_inference_duration_seconds',
            'ML inference duration',
            ['model_name'],
            buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.0]
        )
        
        self.prediction_gauge = Gauge(
            'ml_prediction_value',
            'Current ML prediction value',
            ['model_name', 'metric']
        )
        
        self.drift_gauge = Gauge(
            'ml_data_drift_psi',
            'Data drift PSI score',
            ['model_name', 'feature']
        )
    
    def record_inference(self, model_name: str, duration_s: float, success: bool):
        status = 'success' if success else 'error'
        self.inference_counter.labels(model_name=model_name, status=status).inc()
        self.inference_latency.labels(model_name=model_name).observe(duration_s)
    
    def record_drift(self, model_name: str, feature: str, psi: float):
        self.drift_gauge.labels(model_name=model_name, feature=feature).set(psi)
```

---

## สรุปสมบูรณ์ Machine Learning

ML stack สมบูรณ์สำหรับ LINE Bot:

| Use Case | Algorithm | Latency | Accuracy |
|----------|-----------|---------|---------|
| Intent Classification | WangchanBERTa | < 50ms | ~92% |
| Product Recommendation | Collaborative Filtering | < 20ms | HR@10: 0.35 |
| Churn Prediction | Gradient Boosting | < 10ms | AUC: 0.88 |
| Customer Segmentation | K-Means RFM | Batch | - |
| LTV Prediction | BG/NBD + Gamma-Gamma | < 10ms | MAE: ±15% |
| Spam Detection | TF-IDF + Gradient Boosting | < 5ms | F1: 0.95 |
| Content Personalization | Hybrid (CF + Content-based) | < 30ms | CTR +20% |

Pipeline ควรรัน:
- Real-time inference: ทุก request
- Daily batch: Segment update, LTV update
- Weekly training: Model retraining
- Monthly evaluation: Full model evaluation

---

*จบ Part 84: Machine Learning สำหรับ LINE Bots (ฉบับสมบูรณ์)*

---

## บทที่ 17: ระบบ A/B Testing สำหรับ Machine Learning Models

### การออกแบบ Multi-Armed Bandit System

```python
import numpy as np
from typing import Dict, List, Optional
import redis
import json
from datetime import datetime, timedelta

class ThompsonSamplingBandit:
    """
    Thompson Sampling Multi-Armed Bandit สำหรับ Online Learning
    ใช้สำหรับ optimize model selection แบบ real-time
    """
    
    def __init__(self, redis_client, experiment_id: str):
        self.redis = redis_client
        self.experiment_id = experiment_id
        self.key_prefix = f"bandit:{experiment_id}"
    
    def _get_arm_stats(self, arm_id: str) -> Dict:
        """ดึงสถิติของ arm จาก Redis"""
        key = f"{self.key_prefix}:arm:{arm_id}"
        data = self.redis.hgetall(key)
        if not data:
            return {"alpha": 1.0, "beta": 1.0, "total": 0, "wins": 0}
        return {
            "alpha": float(data.get(b"alpha", 1.0)),
            "beta": float(data.get(b"beta", 1.0)),
            "total": int(data.get(b"total", 0)),
            "wins": int(data.get(b"wins", 0))
        }
    
    def select_arm(self, available_arms: List[str]) -> str:
        """เลือก arm ด้วย Thompson Sampling"""
        samples = {}
        for arm_id in available_arms:
            stats = self._get_arm_stats(arm_id)
            # Sample จาก Beta distribution
            sample = np.random.beta(stats["alpha"], stats["beta"])
            samples[arm_id] = sample
        
        # เลือก arm ที่มี sample สูงสุด
        selected = max(samples, key=samples.get)
        return selected
    
    def record_result(self, arm_id: str, success: bool):
        """บันทึกผลลัพธ์และ update Beta distribution"""
        key = f"{self.key_prefix}:arm:{arm_id}"
        stats = self._get_arm_stats(arm_id)
        
        stats["total"] += 1
        if success:
            stats["wins"] += 1
            stats["alpha"] += 1  # Beta(alpha+1, beta) เมื่อ success
        else:
            stats["beta"] += 1   # Beta(alpha, beta+1) เมื่อ fail
        
        self.redis.hset(key, mapping={
            "alpha": stats["alpha"],
            "beta": stats["beta"],
            "total": stats["total"],
            "wins": stats["wins"],
            "updated_at": datetime.now().isoformat()
        })
        self.redis.expire(key, 86400 * 30)  # expire 30 วัน
    
    def get_statistics(self) -> Dict:
        """สรุปสถิติของทุก arm"""
        pattern = f"{self.key_prefix}:arm:*"
        keys = self.redis.keys(pattern)
        stats = {}
        
        for key in keys:
            arm_id = key.decode().split(":")[-1]
            arm_stats = self._get_arm_stats(arm_id)
            win_rate = arm_stats["wins"] / arm_stats["total"] if arm_stats["total"] > 0 else 0
            stats[arm_id] = {
                **arm_stats,
                "win_rate": win_rate,
                "confidence_interval": self._calculate_ci(arm_stats)
            }
        
        return stats
    
    def _calculate_ci(self, stats: Dict, confidence: float = 0.95) -> Dict:
        """คำนวณ Confidence Interval ของ Beta distribution"""
        from scipy import stats as scipy_stats
        alpha, beta = stats["alpha"], stats["beta"]
        
        # Wilson Score Interval
        lower = scipy_stats.beta.ppf((1 - confidence) / 2, alpha, beta)
        upper = scipy_stats.beta.ppf(1 - (1 - confidence) / 2, alpha, beta)
        
        return {"lower": lower, "upper": upper}


class MLModelABTestOrchestrator:
    """
    Orchestrator สำหรับ A/B Testing ระหว่าง ML Models
    รองรับ Statistical Significance Testing และ Early Stopping
    """
    
    def __init__(self, redis_client, db_pool, mlflow_client):
        self.redis = redis_client
        self.db = db_pool
        self.mlflow = mlflow_client
        self.bandit_cache = {}
    
    def create_experiment(
        self,
        experiment_id: str,
        models: List[Dict],
        metric: str = "user_engagement",
        min_samples: int = 1000,
        significance_level: float = 0.05
    ) -> Dict:
        """
        สร้าง Experiment ใหม่
        
        Args:
            models: รายการ model configs [{"id": "v1", "run_id": "mlflow_run_id", "traffic_pct": 50}, ...]
            metric: metric ที่ใช้วัดผล
            min_samples: จำนวน samples ขั้นต่ำก่อนสรุปผล
        """
        experiment_config = {
            "id": experiment_id,
            "models": models,
            "metric": metric,
            "min_samples": min_samples,
            "significance_level": significance_level,
            "status": "running",
            "created_at": datetime.now().isoformat()
        }
        
        # บันทึกลง Redis
        self.redis.setex(
            f"experiment:{experiment_id}",
            86400 * 30,
            json.dumps(experiment_config)
        )
        
        # สร้าง Bandit สำหรับ experiment นี้
        self.bandit_cache[experiment_id] = ThompsonSamplingBandit(
            self.redis, experiment_id
        )
        
        return experiment_config
    
    def get_model_for_user(self, experiment_id: str, user_id: str) -> Optional[str]:
        """
        เลือก Model สำหรับ User โดยใช้ Consistent Hashing + Thompson Sampling
        """
        exp_key = f"experiment:{experiment_id}"
        exp_data = self.redis.get(exp_key)
        if not exp_data:
            return None
        
        config = json.loads(exp_data)
        if config["status"] != "running":
            # ถ้า experiment จบแล้ว ใช้ winner เสมอ
            return config.get("winner")
        
        # ตรวจสอบว่า user นี้ถูก assign model ไปแล้วหรือยัง
        assignment_key = f"exp:{experiment_id}:user:{user_id}"
        assigned = self.redis.get(assignment_key)
        if assigned:
            return assigned.decode()
        
        # เลือก model ด้วย Bandit
        model_ids = [m["id"] for m in config["models"]]
        if experiment_id in self.bandit_cache:
            selected = self.bandit_cache[experiment_id].select_arm(model_ids)
        else:
            # Fallback: uniform random
            selected = np.random.choice(model_ids)
        
        # Assign และ cache
        self.redis.setex(assignment_key, 86400 * 7, selected)  # 7 วัน
        return selected
    
    def record_interaction(
        self,
        experiment_id: str,
        user_id: str,
        model_id: str,
        event_type: str,
        metric_value: float = 1.0
    ):
        """บันทึก interaction และ update Bandit"""
        success = event_type in ["click", "purchase", "positive_feedback"]
        
        if experiment_id in self.bandit_cache:
            self.bandit_cache[experiment_id].record_result(model_id, success)
        
        # Log ลง database สำหรับ analysis
        self.db.execute("""
            INSERT INTO ml_experiment_events 
            (experiment_id, user_id, model_id, event_type, metric_value, timestamp)
            VALUES ($1, $2, $3, $4, $5, NOW())
        """, experiment_id, user_id, model_id, event_type, metric_value)
        
        # ตรวจสอบ Early Stopping
        self._check_early_stopping(experiment_id)
    
    def _check_early_stopping(self, experiment_id: str):
        """ตรวจสอบว่าควรหยุด experiment เร็วไหม (Bayesian Sequential Testing)"""
        exp_data = self.redis.get(f"experiment:{experiment_id}")
        if not exp_data:
            return
        
        config = json.loads(exp_data)
        if config["status"] != "running":
            return
        
        bandit = self.bandit_cache.get(experiment_id)
        if not bandit:
            return
        
        stats = bandit.get_statistics()
        
        # ตรวจสอบว่ามี samples เพียงพอหรือยัง
        total_samples = sum(s["total"] for s in stats.values())
        if total_samples < config["min_samples"]:
            return
        
        # Bayesian Probability of Best Arm
        model_ids = list(stats.keys())
        if len(model_ids) < 2:
            return
        
        # Monte Carlo simulation
        n_simulations = 10000
        wins = {mid: 0 for mid in model_ids}
        
        for _ in range(n_simulations):
            samples = {
                mid: np.random.beta(stats[mid]["alpha"], stats[mid]["beta"])
                for mid in model_ids
            }
            winner = max(samples, key=samples.get)
            wins[winner] += 1
        
        probs = {mid: wins[mid] / n_simulations for mid in model_ids}
        best_arm = max(probs, key=probs.get)
        best_prob = probs[best_arm]
        
        # ถ้า probability > 95% หยุด experiment
        if best_prob >= 0.95:
            config["status"] = "completed"
            config["winner"] = best_arm
            config["winner_probability"] = best_prob
            config["completed_at"] = datetime.now().isoformat()
            config["total_samples"] = total_samples
            
            self.redis.setex(
                f"experiment:{experiment_id}",
                86400 * 90,  # เก็บ 90 วัน
                json.dumps(config)
            )
            
            print(f"Experiment {experiment_id} completed! Winner: {best_arm} (p={best_prob:.3f})")
    
    def get_experiment_report(self, experiment_id: str) -> Dict:
        """สร้าง Report สรุปผล Experiment"""
        exp_data = self.redis.get(f"experiment:{experiment_id}")
        if not exp_data:
            return {"error": "Experiment not found"}
        
        config = json.loads(exp_data)
        bandit = self.bandit_cache.get(experiment_id)
        
        report = {
            "experiment_id": experiment_id,
            "status": config["status"],
            "created_at": config["created_at"],
            "metric": config["metric"],
            "model_stats": {}
        }
        
        if bandit:
            stats = bandit.get_statistics()
            for model_id, model_stats in stats.items():
                report["model_stats"][model_id] = {
                    "total_users": model_stats["total"],
                    "conversions": model_stats["wins"],
                    "win_rate": model_stats["win_rate"],
                    "confidence_interval_95": model_stats["confidence_interval"]
                }
        
        if config["status"] == "completed":
            report["winner"] = config.get("winner")
            report["winner_probability"] = config.get("winner_probability")
            report["total_samples"] = config.get("total_samples")
        
        return report
```

### Integration กับ LINE Bot

```python
class MLPoweredLineBot:
    """LINE Bot ที่ใช้ ML สำหรับทุก feature หลัก"""
    
    def __init__(self, line_bot_api, ml_services):
        self.line_bot_api = line_bot_api
        self.recommender = ml_services["recommender"]
        self.intent_classifier = ml_services["intent_classifier"]
        self.churn_predictor = ml_services["churn_predictor"]
        self.ab_orchestrator = ml_services["ab_orchestrator"]
        self.personalized_content = ml_services["content_personalizer"]
    
    async def handle_message(self, event):
        """จัดการ Message Event ด้วย ML Pipeline"""
        user_id = event.source.user_id
        message_text = event.message.text
        
        # 1. Intent Classification
        intent_result = await self.intent_classifier.classify_async(message_text)
        intent = intent_result["intent"]
        confidence = intent_result["confidence"]
        
        # 2. เลือก Model version ตาม A/B Test
        model_version = self.ab_orchestrator.get_model_for_user(
            "response_model_v2_test", user_id
        )
        
        # 3. Route ไปยัง handler ที่เหมาะสม
        if intent == "product_inquiry" and confidence > 0.7:
            response = await self._handle_product_inquiry(
                user_id, message_text, model_version
            )
        elif intent == "complaint" and confidence > 0.6:
            response = await self._handle_complaint(user_id, message_text)
        elif intent == "order_status":
            response = await self._handle_order_status(user_id)
        else:
            response = await self._handle_general_query(
                user_id, message_text, model_version
            )
        
        # 4. บันทึก interaction สำหรับ A/B Test
        self.ab_orchestrator.record_interaction(
            "response_model_v2_test",
            user_id,
            model_version or "default",
            "message_sent"
        )
        
        # 5. ส่ง Response
        self.line_bot_api.reply_message(event.reply_token, response)
        
        # 6. Background: Churn Risk Check
        await self._check_churn_risk_async(user_id)
    
    async def _handle_product_inquiry(
        self, user_id: str, query: str, model_version: str
    ):
        """จัดการ Product Inquiry ด้วย Recommendations"""
        from linebot.models import FlexSendMessage
        
        # ดึง recommendations
        recommendations = self.recommender.get_recommendations(user_id, top_k=5)
        
        if not recommendations:
            # Fallback: แสดงสินค้ายอดนิยม
            recommendations = self.recommender.get_popular_items(top_k=5)
        
        # สร้าง Flex Message
        flex_content = self._build_product_carousel(recommendations)
        
        return FlexSendMessage(
            alt_text="สินค้าแนะนำสำหรับคุณ",
            contents=flex_content
        )
    
    async def _check_churn_risk_async(self, user_id: str):
        """ตรวจสอบ Churn Risk และส่ง Retention Message ถ้าจำเป็น"""
        import asyncio
        
        async def check():
            features = await self._get_user_features(user_id)
            churn_prob = self.churn_predictor.predict_proba(features)
            
            if churn_prob > 0.7:  # High churn risk
                # ส่ง Retention message หลัง 5 นาที
                await asyncio.sleep(300)
                retention_msg = await self.personalized_content.create_retention_message(
                    user_id, churn_prob
                )
                self.line_bot_api.push_message(user_id, retention_msg)
        
        asyncio.create_task(check())
    
    def _build_product_carousel(self, products: List[Dict]):
        """สร้าง Carousel Flex Message สำหรับสินค้า"""
        bubbles = []
        for product in products[:10]:  # LINE รองรับสูงสุด 10 bubbles
            bubble = {
                "type": "bubble",
                "hero": {
                    "type": "image",
                    "url": product.get("image_url", "https://via.placeholder.com/300"),
                    "size": "full",
                    "aspectRatio": "20:13",
                    "action": {
                        "type": "uri",
                        "uri": product.get("product_url", "#")
                    }
                },
                "body": {
                    "type": "box",
                    "layout": "vertical",
                    "contents": [
                        {
                            "type": "text",
                            "text": product.get("name", "สินค้า"),
                            "weight": "bold",
                            "size": "xl"
                        },
                        {
                            "type": "text",
                            "text": f"฿{product.get('price', 0):,.0f}",
                            "color": "#E74C3C",
                            "weight": "bold"
                        },
                        {
                            "type": "text",
                            "text": f"⭐ {product.get('rating', 0):.1f} ({product.get('reviews', 0)} รีวิว)",
                            "color": "#888888",
                            "size": "sm"
                        }
                    ]
                },
                "footer": {
                    "type": "box",
                    "layout": "vertical",
                    "contents": [
                        {
                            "type": "button",
                            "style": "primary",
                            "action": {
                                "type": "uri",
                                "label": "ดูสินค้า",
                                "uri": product.get("product_url", "#")
                            }
                        }
                    ]
                }
            }
            bubbles.append(bubble)
        
        return {
            "type": "carousel",
            "contents": bubbles
        }
```

---

## บทที่ 18: Federated Learning สำหรับ Privacy-Preserving ML

### ทำไมต้องใช้ Federated Learning

```
ปัญหาปัจจุบัน:
- ข้อมูลผู้ใช้ถูกเก็บส่วนกลาง → Privacy Risk
- PDPA ต้องการ data minimization
- Users ไม่ต้องการแชร์ข้อมูลส่วนตัว

Federated Learning แก้ปัญหา:
- Train model บน device ของ user
- ส่งเฉพาะ model weights (ไม่ใช่ raw data)
- Aggregate weights บน server
- Privacy โดย design
```

### Federated Learning Architecture

```
┌─────────────────────────────────────────────────────┐
│                 Federated Learning Flow              │
│                                                     │
│   Server          Client 1     Client 2   Client 3  │
│     │                │            │          │      │
│     │  Global Model  │            │          │      │
│     │─────────────►  │            │          │      │
│     │─────────────────────────►   │          │      │
│     │──────────────────────────────────────► │      │
│     │                │            │          │      │
│     │  Local Train   │            │          │      │
│     │                │ (private)  │(private) │      │
│     │                │            │          │      │
│     │  Update ΔW     │            │          │      │
│     │ ◄─────────────  │            │          │      │
│     │ ◄─────────────────────────   │          │      │
│     │ ◄──────────────────────────────────────│      │
│     │                │            │          │      │
│     │  Aggregate FedAvg           │          │      │
│     │  W = W + η*mean(ΔW)         │          │      │
│     │                │            │          │      │
└─────────────────────────────────────────────────────┘
```

```python
import torch
import torch.nn as nn
from typing import List, Dict
import copy

class FederatedLineBot:
    """
    Federated Learning สำหรับ LINE Bot Intent Classifier
    ใช้ FedAvg Algorithm (McMahan et al., 2017)
    """
    
    def __init__(self, global_model: nn.Module, n_rounds: int = 100):
        self.global_model = global_model
        self.n_rounds = n_rounds
        self.round = 0
        self.client_updates = []
    
    def broadcast_global_model(self) -> Dict:
        """ส่ง Global Model weights ไปยัง Clients"""
        return {
            "round": self.round,
            "weights": {
                name: param.data.clone()
                for name, param in self.global_model.named_parameters()
            }
        }
    
    def receive_client_update(
        self, 
        client_id: str, 
        model_update: Dict,
        n_samples: int
    ):
        """รับ Model Update จาก Client"""
        self.client_updates.append({
            "client_id": client_id,
            "weights": model_update["weights"],
            "n_samples": n_samples,
            "round": model_update["round"]
        })
    
    def aggregate_updates(self):
        """FedAvg: Weighted Average ตามจำนวน samples"""
        if not self.client_updates:
            return
        
        total_samples = sum(u["n_samples"] for u in self.client_updates)
        
        # Initialize aggregated weights
        aggregated = {}
        for name, param in self.global_model.named_parameters():
            aggregated[name] = torch.zeros_like(param.data)
        
        # Weighted sum
        for update in self.client_updates:
            weight = update["n_samples"] / total_samples
            for name, delta in update["weights"].items():
                aggregated[name] += weight * delta
        
        # Update Global Model
        with torch.no_grad():
            for name, param in self.global_model.named_parameters():
                param.data = aggregated[name]
        
        # Clear updates
        self.client_updates = []
        self.round += 1
        
        print(f"Round {self.round}: Aggregated {total_samples} samples from {len(self.client_updates)} clients")
    
    def add_differential_privacy(self, sensitivity: float = 1.0, epsilon: float = 1.0):
        """
        เพิ่ม Differential Privacy ด้วย Gaussian Mechanism
        ป้องกัน Membership Inference Attack
        """
        delta = 1e-5
        sigma = np.sqrt(2 * np.log(1.25 / delta)) * sensitivity / epsilon
        
        with torch.no_grad():
            for param in self.global_model.parameters():
                noise = torch.normal(0, sigma, size=param.data.shape)
                param.data += noise
        
        print(f"Added DP noise: σ={sigma:.4f} (ε={epsilon}, δ={delta})")


class ClientFederatedTrainer:
    """
    Client-side Training (ทำงานบน Edge/Mobile)
    """
    
    def __init__(self, user_id: str, local_data, learning_rate: float = 0.01):
        self.user_id = user_id
        self.local_data = local_data
        self.lr = learning_rate
        self.local_model = None
    
    def load_global_model(self, global_weights: Dict) -> nn.Module:
        """โหลด Global Model weights"""
        model = IntentClassifierModel()  # สร้าง model structure
        
        # Load weights
        model_state = {}
        for name, weight in global_weights.items():
            model_state[name] = weight.clone()
        
        model.load_state_dict(model_state)
        self.local_model = copy.deepcopy(model)
        return model
    
    def local_train(
        self, 
        global_weights: Dict, 
        n_epochs: int = 5,
        batch_size: int = 32
    ) -> Dict:
        """Train model ด้วย Local Data"""
        model = self.load_global_model(global_weights)
        optimizer = torch.optim.SGD(model.parameters(), lr=self.lr)
        criterion = nn.CrossEntropyLoss()
        
        # สร้าง DataLoader จาก local data
        loader = torch.utils.data.DataLoader(
            self.local_data, batch_size=batch_size, shuffle=True
        )
        
        model.train()
        for epoch in range(n_epochs):
            total_loss = 0
            for batch_x, batch_y in loader:
                optimizer.zero_grad()
                output = model(batch_x)
                loss = criterion(output, batch_y)
                loss.backward()
                
                # Gradient Clipping (สำหรับ DP)
                torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
                
                optimizer.step()
                total_loss += loss.item()
        
        # คำนวณ Delta Weights (ไม่ส่ง raw data!)
        delta_weights = {}
        for name, param in model.named_parameters():
            global_param = global_weights[name]
            delta_weights[name] = param.data - global_param
        
        return {
            "weights": delta_weights,
            "n_samples": len(self.local_data),
            "loss": total_loss / len(loader)
        }
```

---

## บทที่ 19: ML Model Versioning และ Governance

### Model Registry ด้วย MLflow

```python
import mlflow
import mlflow.sklearn
from mlflow.tracking import MlflowClient
from datetime import datetime
from typing import Optional

class LineMLModelRegistry:
    """
    Model Registry สำหรับ LINE Bot ML Models
    ครอบคลุม lifecycle ตั้งแต่ training จนถึง production
    """
    
    def __init__(self, mlflow_tracking_uri: str):
        mlflow.set_tracking_uri(mlflow_tracking_uri)
        self.client = MlflowClient()
    
    def register_model(
        self,
        run_id: str,
        model_name: str,
        model_artifact_path: str,
        tags: Optional[Dict] = None,
        description: Optional[str] = None
    ) -> str:
        """Register model และส่งคืน version"""
        # สร้าง Model Registry entry
        result = mlflow.register_model(
            f"runs:/{run_id}/{model_artifact_path}",
            model_name
        )
        
        # เพิ่ม tags
        if tags:
            for key, value in tags.items():
                self.client.set_model_version_tag(
                    model_name, result.version, key, value
                )
        
        # เพิ่ม description
        if description:
            self.client.update_model_version(
                name=model_name,
                version=result.version,
                description=description
            )
        
        print(f"Registered {model_name} v{result.version}")
        return result.version
    
    def promote_to_staging(self, model_name: str, version: str):
        """Promote model ไปยัง Staging environment"""
        self.client.transition_model_version_stage(
            name=model_name,
            version=version,
            stage="Staging",
            archive_existing_versions=False
        )
        
        # บันทึก approval workflow
        self.client.set_model_version_tag(
            model_name, version,
            "staging_promoted_at",
            datetime.now().isoformat()
        )
    
    def promote_to_production(
        self,
        model_name: str,
        version: str,
        approver: str,
        approval_notes: str = ""
    ):
        """Promote model ไปยัง Production (ต้องมี approval)"""
        # Archive previous production versions
        self.client.transition_model_version_stage(
            name=model_name,
            version=version,
            stage="Production",
            archive_existing_versions=True  # Archive old versions
        )
        
        # บันทึก audit trail
        self.client.set_model_version_tag(model_name, version, "approved_by", approver)
        self.client.set_model_version_tag(model_name, version, "approval_notes", approval_notes)
        self.client.set_model_version_tag(
            model_name, version,
            "production_promoted_at",
            datetime.now().isoformat()
        )
    
    def load_production_model(self, model_name: str):
        """โหลด Production Model version ล่าสุด"""
        model_uri = f"models:/{model_name}/Production"
        return mlflow.pyfunc.load_model(model_uri)
    
    def compare_models(
        self,
        model_name: str,
        versions: List[str],
        test_data
    ) -> Dict:
        """เปรียบเทียบ Model versions หลายๆ ตัว"""
        results = {}
        
        for version in versions:
            model_uri = f"models:/{model_name}/{version}"
            model = mlflow.pyfunc.load_model(model_uri)
            
            # รัน predictions
            predictions = model.predict(test_data["features"])
            
            # คำนวณ metrics
            from sklearn.metrics import accuracy_score, f1_score
            results[version] = {
                "accuracy": accuracy_score(test_data["labels"], predictions),
                "f1_macro": f1_score(test_data["labels"], predictions, average="macro"),
                "latency_ms": self._measure_latency(model, test_data["features"][:100])
            }
        
        return results
    
    def _measure_latency(self, model, samples, n_runs: int = 100) -> float:
        """วัด Inference Latency"""
        import time
        latencies = []
        
        for _ in range(n_runs):
            start = time.perf_counter()
            model.predict(samples)
            end = time.perf_counter()
            latencies.append((end - start) * 1000)
        
        return np.percentile(latencies, 95)  # P95 latency


class ModelGovernanceManager:
    """
    Model Governance: ตรวจสอบ Fairness, Bias, และ Compliance
    ตาม PDPA และ AI Ethics Guidelines
    """
    
    def __init__(self, db_pool):
        self.db = db_pool
    
    def audit_model_predictions(
        self,
        model_name: str,
        predictions: List[Dict],
        sensitive_attributes: List[str] = ["age_group", "gender", "region"]
    ) -> Dict:
        """
        ตรวจสอบ Bias ใน Model Predictions
        ตาม Demographic Parity และ Equal Opportunity
        """
        audit_results = {
            "model_name": model_name,
            "total_predictions": len(predictions),
            "bias_analysis": {},
            "recommendations": []
        }
        
        for attribute in sensitive_attributes:
            if attribute not in predictions[0]:
                continue
            
            # แบ่ง groups
            groups = {}
            for pred in predictions:
                group = pred[attribute]
                if group not in groups:
                    groups[group] = {"positive": 0, "total": 0}
                groups[group]["total"] += 1
                if pred["prediction"] == 1:  # Positive prediction
                    groups[group]["positive"] += 1
            
            # คำนวณ Positive Rate ต่อ group
            positive_rates = {
                group: data["positive"] / data["total"]
                for group, data in groups.items()
                if data["total"] > 0
            }
            
            # Demographic Parity Difference
            max_rate = max(positive_rates.values())
            min_rate = min(positive_rates.values())
            disparity = max_rate - min_rate
            
            audit_results["bias_analysis"][attribute] = {
                "positive_rates": positive_rates,
                "demographic_parity_difference": disparity,
                "is_fair": disparity < 0.1  # Threshold: 10%
            }
            
            if disparity >= 0.1:
                audit_results["recommendations"].append(
                    f"ตรวจพบ Bias ใน {attribute}: disparate impact = {disparity:.2%}. "
                    f"แนะนำให้ rebalance training data"
                )
        
        # บันทึก audit log
        self.db.execute("""
            INSERT INTO model_audit_logs 
            (model_name, audit_date, total_predictions, bias_results, created_at)
            VALUES ($1, NOW(), $2, $3, NOW())
        """, model_name, len(predictions), json.dumps(audit_results))
        
        return audit_results
    
    def generate_model_card(self, model_name: str, version: str) -> str:
        """
        สร้าง Model Card (เอกสารอธิบาย Model สำหรับ Transparency)
        ตาม Google Model Cards standard
        """
        # ดึงข้อมูลจาก MLflow
        client = MlflowClient()
        model_version = client.get_model_version(model_name, version)
        run = client.get_run(model_version.run_id)
        
        metrics = run.data.metrics
        params = run.data.params
        tags = run.data.tags
        
        card = f"""# Model Card: {model_name} v{version}

## Model Overview
- **Model Name**: {model_name}
- **Version**: {version}
- **Type**: {tags.get('model_type', 'Unknown')}
- **Created**: {model_version.creation_timestamp}
- **Author**: {tags.get('author', 'Unknown')}

## Intended Use
- **Primary Use Case**: {tags.get('use_case', 'LINE Bot Response Classification')}
- **Intended Users**: Customer Service Teams, Marketing Teams
- **Out-of-Scope Uses**: ไม่เหมาะสำหรับ Legal decisions หรือ Medical diagnosis

## Training Data
- **Dataset Size**: {params.get('training_samples', 'N/A')} samples
- **Languages**: Thai (Primary), English (Secondary)
- **Time Range**: {params.get('data_start_date', 'N/A')} ถึง {params.get('data_end_date', 'N/A')}
- **Data Collection**: เก็บจาก LINE Bot conversations (ขอ consent แล้ว)

## Performance Metrics
| Metric | Value |
|--------|-------|
| Accuracy | {metrics.get('accuracy', 'N/A')} |
| F1 Score (Macro) | {metrics.get('f1_macro', 'N/A')} |
| Precision | {metrics.get('precision', 'N/A')} |
| Recall | {metrics.get('recall', 'N/A')} |
| P95 Latency | {metrics.get('p95_latency_ms', 'N/A')} ms |

## Limitations
- ประสิทธิภาพลดลงสำหรับ dialect ภาษาไทยที่หายาก
- อาจมี Bias สำหรับ demographic groups ที่มีข้อมูลน้อย
- ไม่รองรับภาษา Code-switching ที่ซับซ้อน

## Ethical Considerations
- ข้อมูล Training ผ่าน anonymization และขอ consent แล้ว
- ทำ Bias audit ทุก 3 เดือน
- มี Human-in-the-loop สำหรับ high-stakes decisions
- ปฏิบัติตาม PDPA 2562

## Monitoring
- ตรวจสอบ Model Drift ทุกสัปดาห์
- Alert เมื่อ accuracy ลดลงมากกว่า 5%
- Retrain ทุกเดือนด้วย fresh data
"""
        return card
```

---

## สรุปบทที่ 19: Key Takeaways

### สิ่งที่เรียนรู้ในบท Machine Learning นี้

```
1. Recommendation System
   ├── Collaborative Filtering (User-Item Matrix)
   ├── Content-Based (Sentence Embeddings)
   └── Hybrid Approach สำหรับ Cold Start

2. Intent Classification
   ├── WangchanBERTa Fine-tuning
   ├── Thai Language Preprocessing
   └── Confidence Thresholds

3. Predictive Analytics
   ├── Churn Prediction (GradientBoosting)
   ├── Customer Segmentation (K-Means RFM)
   └── LTV Prediction (BG/NBD)

4. A/B Testing
   ├── Thompson Sampling Bandit
   ├── Bayesian Early Stopping
   └── Consistent Hashing for User Assignment

5. Federated Learning
   ├── FedAvg Algorithm
   ├── Differential Privacy
   └── Privacy-Preserving Training

6. Model Governance
   ├── MLflow Model Registry
   ├── Bias Detection (Demographic Parity)
   ├── Model Cards (Transparency)
   └── Audit Trail (Compliance)
```

### Production Checklist

```yaml
# ML Production Readiness Checklist

model_quality:
  - [ ] Accuracy > 85% บน test set
  - [ ] P95 latency < 100ms
  - [ ] Bias audit ผ่าน (disparity < 10%)
  - [ ] Robustness test (adversarial inputs)

infrastructure:
  - [ ] Model served ด้วย FastAPI/TorchServe
  - [ ] Auto-scaling ตาม load
  - [ ] Model versioning ใน MLflow Registry
  - [ ] A/B Testing framework พร้อม

monitoring:
  - [ ] Prometheus metrics ครบ
  - [ ] Data drift detection active
  - [ ] Model performance dashboard
  - [ ] Alerting rules configured

compliance:
  - [ ] Data consent recorded
  - [ ] PII anonymized
  - [ ] Model card เขียนแล้ว
  - [ ] PDPA audit trail complete
  - [ ] Retrain schedule defined
```

### เครื่องมือและ Library ที่ใช้

| เครื่องมือ | Version | วัตถุประสงค์ |
|-----------|---------|------------|
| scikit-learn | 1.3+ | ML Algorithms |
| PyTorch | 2.0+ | Deep Learning |
| transformers | 4.35+ | WangchanBERTa |
| MLflow | 2.8+ | Experiment Tracking |
| sentence-transformers | 2.2+ | Thai Embeddings |
| lightfm | 1.17 | Collaborative Filtering |
| lifetimes | 0.11 | LTV Prediction |
| SHAP | 0.42+ | Model Explainability |
| Redis | 7.0+ | Feature Cache |
| Prometheus | 2.47+ | Metrics |

