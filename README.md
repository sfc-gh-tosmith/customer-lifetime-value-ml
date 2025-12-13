# Customer Lifetime Value ML Models

## Cold Start CLV Model

The cold start model predicts customer lifetime value for new customers with little to no historical purchase data. It relies on initial attributes available at acquisition such as signup channel, demographic information, first purchase characteristics, and early engagement signals. This model helps prioritize marketing spend and customer experience investments before sufficient behavioral history accumulates.

**Training**: Trained on historical cohorts of customers using features available at their acquisition time (e.g., first 7-30 days). Labels are the actual lifetime value observed over a defined window (e.g., 12-24 months). Features include acquisition source, initial transaction amount, device type, geographic data, and early activity metrics.

## Continuous-Prediction CLV Model

The continuous model provides updated CLV predictions as customers accumulate transaction history and behavioral data. It leverages rich feature sets including purchase frequency, recency, average order value, product preferences, engagement patterns, and seasonality effects. This model enables dynamic segmentation and personalized retention strategies based on evolving customer value.

**Training**: Trained on customers with sufficient history (e.g., 3+ months of activity) using rolling time windows. Features include transactional aggregates (RFM metrics), behavioral sequences, customer service interactions, and temporal patterns. Labels are forward-looking value over a prediction horizon, with the model retrained regularly on recent cohorts to capture evolving behaviors.
