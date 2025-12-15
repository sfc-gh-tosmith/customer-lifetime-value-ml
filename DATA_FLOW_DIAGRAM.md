# Customer Lifetime Value ML Pipeline - Data Flow

```mermaid
graph TB
    subgraph "RAW DATA GENERATION"
        A1[generate_coldstart_data.ipynb]
        A2[generate_continuous_data.ipynb]
        
        A1 --> B1[(COLDSTART_CUSTOMERS)]
        A2 --> B2[(CONTINUOUS_CUSTOMERS_PROFILE)]
        A2 --> B3[(CONTINUOUS_TRANSACTIONS)]
        A2 --> B4[(CONTINUOUS_INTERACTIONS)]
    end
    
    subgraph "COLD START PATH"
        B1 --> C1[train_coldstart_model.ipynb]
        C1 --> |Feature Engineering| C2[Early signals:<br/>- Acquisition channel<br/>- First purchase<br/>- 30d engagement<br/>- Demographics]
        C2 --> |XGBoost + HPO| C3[COLDSTART_CLV_MODEL<br/>Version: V1<br/>Target: 12M LTV]
        C3 --> D1[(ML Registry)]
        C3 --> D2[(COLDSTART_CUSTOMERS_STAGING)]
        D2 --> D3[(COLDSTART_CLV_PREDICTIONS<br/>Dynamic Table<br/>Refresh: 5 min)]
    end
    
    subgraph "CONTINUOUS PATH - FEATURE STORE"
        B2 --> E1[feature_engineering_continuous.ipynb]
        B3 --> E1
        B4 --> E1
        
        E1 --> |Feature Store| E2[(CLV_FEATURE_STORE Schema)]
        
        E2 --> F1[(RFM_FEATURES<br/>Feature View<br/>- RECENCY_DAYS<br/>- FREQUENCY<br/>- MONETARY_TOTAL<br/>- MONETARY_AVG<br/>Refresh: 1 day)]
        
        E2 --> F2[(PURCHASE_PATTERNS<br/>Feature View<br/>- UNIQUE_CATEGORIES<br/>- RECENT_30D_AMOUNT<br/>- RECENT_90D_AMOUNT<br/>Refresh: 1 day)]
        
        E2 --> F3[(ENGAGEMENT_FEATURES<br/>Feature View<br/>- WEBSITE_VISITS<br/>- EMAIL_OPENS<br/>- EMAIL_CLICKS<br/>Refresh: 1 day)]
        
        E1 --> |Calculate Target| F4[(CONTINUOUS_CUSTOMERS_PROFILE_WITH_TARGET<br/>+ FUTURE_12M_LTV)]
        
        F1 --> G1
        F2 --> G1
        F3 --> G1
        F4 --> G1
        
        G1[(CONTINUOUS_TRAINING_DATA_WITH_TARGET<br/>Feature Store Output)]
    end
    
    subgraph "CONTINUOUS PATH - TRAINING & INFERENCE"
        G1 --> H1[train_continuous_model.ipynb]
        H1 --> |Advanced Features| H2[Derived:<br/>- RFM scores<br/>- Lifecycle stage<br/>- Velocity indicators<br/>- Cohort comparisons]
        H2 --> |XGBoost + HPO| H3[CONTINUOUS_CLV_MODEL<br/>Version: V1<br/>Target: 12M LTV]
        H3 --> I1[(ML Registry)]
        
        H1 --> |Inference Setup| I2[Staging Tables:<br/>TRANSACTIONS_STAGING<br/>INTERACTIONS_STAGING]
        I2 --> |Merge| B2
        I2 --> |Merge| B3
        I2 --> |Merge| B4
        
        B2 -.->|Auto Refresh| F1
        B3 -.->|Auto Refresh| F1
        B3 -.->|Auto Refresh| F2
        B4 -.->|Auto Refresh| F3
        
        H1 --> |fs.retrieve_feature_values| I3[Read Feature Views:<br/>RFM_FEATURES<br/>PURCHASE_PATTERNS<br/>ENGAGEMENT_FEATURES]
        F1 -.-> I3
        F2 -.-> I3
        F3 -.-> I3
        
        I3 --> I4[Model Scoring<br/>Python Pipeline]
        I4 --> I5[(CONTINUOUS_CLV_PREDICTIONS<br/>Table<br/>Scheduled Update)]
    end
    
    subgraph "MODEL DEPLOYMENT"
        D1 --> J1[SQL Inference:<br/>WAREHOUSE Platform]
        D1 --> J2[Python Inference:<br/>SPCS Platform]
        I1 --> J1
        I1 --> J2
    end
    
    style A1 fill:#e1f5ff
    style A2 fill:#e1f5ff
    style B1 fill:#fff4e6
    style B2 fill:#fff4e6
    style B3 fill:#fff4e6
    style B4 fill:#fff4e6
    style E2 fill:#f0e6ff
    style F1 fill:#ffe6f0
    style F2 fill:#ffe6f0
    style F3 fill:#ffe6f0
    style F4 fill:#fff4e6
    style G1 fill:#e6fff0
    style C3 fill:#ffcccc
    style H3 fill:#ffcccc
    style D1 fill:#ccffcc
    style I1 fill:#ccffcc
    style D3 fill:#ffffcc
    style I5 fill:#ffffcc
```

## Data Flow Summary

### Cold Start Pipeline
**Purpose**: Predict CLV for new customers (0-30 days)

1. **Data Generation** (`generate_coldstart_data.ipynb`)
   - Input: Configuration parameters
   - Output: `COLDSTART_CUSTOMERS` (50K records)
   - Features: Demographics, acquisition, early engagement

2. **Model Training** (`train_coldstart_model.ipynb`)
   - Input: `COLDSTART_CUSTOMERS`
   - Transformations:
     - Binary indicators (first purchase)
     - Channel quality scoring
     - Engagement composite scores
     - Engagement rates
   - Output: `COLDSTART_CLV_MODEL` (Snowflake ML Registry)
   - Metrics: RMSE, MAE, R², MAPE
   - HPO: 20 trials, Bayesian optimization

3. **Inference** (Dynamic Table)
   - Staging: `COLDSTART_CUSTOMERS_STAGING`
   - Predictions: `COLDSTART_CLV_PREDICTIONS`
   - Refresh: Every 5 minutes

---

### Continuous Pipeline
**Purpose**: Predict CLV for established customers (90+ days)

1. **Data Generation** (`generate_continuous_data.ipynb`)
   - Input: Configuration parameters
   - Outputs:
     - `CONTINUOUS_CUSTOMERS_PROFILE` (50K customers)
     - `CONTINUOUS_TRANSACTIONS` (~500K transactions)
     - `CONTINUOUS_INTERACTIONS` (~5M interactions)
   - Vectorized generation for performance

2. **Feature Engineering** (`feature_engineering_continuous.ipynb`)
   - Input: Raw tables (customers, transactions, interactions)
   - **Feature Store Architecture**:
     - Schema: `CLV_FEATURE_STORE`
     - Entity: `CUSTOMER` (join key: CUSTOMER_ID)
   - **Feature Views** (Dynamic Tables, 1-day refresh):
     - `RFM_FEATURES`: Recency, Frequency, Monetary, Tenure
     - `PURCHASE_PATTERNS`: Category diversity, 30d/90d windows
     - `ENGAGEMENT_FEATURES`: Website, email, support metrics
   - Transformations:
     - Aggregate transactions by customer
     - Calculate time windows (30d, 90d)
     - Compute engagement rates
     - Join multiple data sources
   - Target Creation: `FUTURE_12M_LTV` (synthetic formula)
   - Output: `CONTINUOUS_TRAINING_DATA_WITH_TARGET`

3. **Model Training** (`train_continuous_model.ipynb`)
   - Input: `CONTINUOUS_TRAINING_DATA_WITH_TARGET`
   - Additional Transformations:
     - RFM score normalization (quintiles)
     - Lifecycle stage classification
     - Velocity indicators (30d vs 90d trends)
     - Cohort-based benchmarking
     - Engagement ratios
   - Output: `CONTINUOUS_CLV_MODEL` (Snowflake ML Registry)
   - Metrics: RMSE, MAE, R², MAPE
   - HPO: 20 trials, Bayesian optimization

4. **Inference** (Feature Store Integration)
   - Staging tables for new data:
     - `CONTINUOUS_TRANSACTIONS_STAGING`
     - `CONTINUOUS_INTERACTIONS_STAGING`
   - New data merged into base tables → Feature views auto-refresh (1 day)
   - Inference via `fs.retrieve_feature_values()` using **same** feature views as training:
     - `RFM_FEATURES` (v1.0)
     - `PURCHASE_PATTERNS` (v1.0)
     - `ENGAGEMENT_FEATURES` (v1.0)
   - Model scoring in Python with full pipeline
   - Output: `CONTINUOUS_CLV_PREDICTIONS` table
   - **Key benefit**: Zero training-serving skew - identical feature logic

---

## Key Transformations

### Feature Store Pattern
- **Raw Data → Feature Views**: Automated aggregations with scheduled refresh
- **Point-in-Time Correctness**: Features aligned with observation dates
- **Lineage Tracking**: Automatic documentation of feature derivations
- **Reusability**: Same features for training and inference

### Model Training Pattern
- **Temporal Splits**: 70% train / 15% validation / 15% test
- **Preprocessing Pipeline**: StandardScaler + OneHotEncoder
- **HPO**: Bayesian optimization with 20 trials
- **Regularization**: Tree depth limits, subsampling, L1/L2 penalties
- **Deployment**: Both WAREHOUSE (SQL) and SPCS (Python) platforms

### Inference Pattern (Feature Store)
- **Staging Tables**: Ingest new transaction/interaction data
- **Base Tables**: Merge/append staging data to CONTINUOUS_TRANSACTIONS, CONTINUOUS_INTERACTIONS
- **Feature Store**: Feature views automatically refresh (1-day schedule)
- **Retrieve Features**: `fs.retrieve_feature_values()` using same feature views as training
- **Model Scoring**: Python pipeline prediction (scheduled notebook execution)
- **Output Table**: CONTINUOUS_CLV_PREDICTIONS with predictions and timestamps
- **Consistency**: Guaranteed identical features for training and inference

---

## Object Naming Convention

**Raw Tables**: `{SCENARIO}_{TYPE}` (e.g., CONTINUOUS_TRANSACTIONS)  
**Feature Views**: `{FEATURE_CATEGORY}` (e.g., RFM_FEATURES)  
**Training Data**: `{SCENARIO}_TRAINING_DATA_WITH_TARGET`  
**Models**: `{SCENARIO}_CLV_MODEL`  
**Predictions**: `{SCENARIO}_CLV_PREDICTIONS`  
**Staging**: `{SCENARIO}_{TYPE}_STAGING`

---

## Technology Stack

- **Data Generation**: Pandas, NumPy (vectorized operations)
- **Feature Store**: Snowflake ML Feature Store
- **Feature Views**: Dynamic Tables (1-day refresh)
- **Model Training**: XGBoost, scikit-learn
- **HPO**: Snowflake ML tune (Bayesian Optimization)
- **Model Registry**: Snowflake ML Registry
- **Inference**: Dynamic Tables + SQL UDFs
- **Platforms**: Snowflake WAREHOUSE + SPCS
