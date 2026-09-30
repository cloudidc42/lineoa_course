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
