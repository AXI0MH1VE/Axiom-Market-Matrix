# Axiom Market Matrix - Technical Specification

## Overview

This document provides technical specifications for key components of the Axiom Market Matrix trading system, with a focus on the Sentiment Monitor subsystem.

---

## Sentiment Monitor Specification

### Purpose

The Sentiment Monitor analyzes market sentiment from multiple data sources to generate trading signals and risk alerts. It aggregates sentiment indicators, applies temporal smoothing, and outputs structured signals for consumption by trading strategies.

### Data Sources

The Sentiment Monitor ingests data from the following sources:

#### 1. News Sentiment
- **Provider**: NewsAPI, Bloomberg Terminal API, Alpha Vantage News Sentiment
- **Update Frequency**: Real-time streaming + 15-minute batch updates
- **Metrics**:
  - Article sentiment score (-1.0 to +1.0)
  - Entity mentions and relevance scores
  - Source credibility weighting
  - Publication timestamp

#### 2. Social Media Signals
- **Provider**: Twitter/X API, Reddit API (r/wallstreetbets, r/stocks, r/investing)
- **Update Frequency**: Real-time streaming
- **Metrics**:
  - Tweet/post sentiment (-1.0 to +1.0) via NLP (VADER, FinBERT)
  - Volume of mentions per ticker/topic
  - Engagement metrics (likes, retweets, comments)
  - Verified account vs. general public segmentation

#### 3. Market Microstructure
- **Provider**: Exchange order book data (via broker API or market data feed)
- **Update Frequency**: Tick-level (as available) aggregated to 1-minute bars
- **Metrics**:
  - Bid-ask spread dynamics
  - Order flow imbalance
  - Trade aggression ratio (market buy vs. market sell volume)
  - Depth of book changes

#### 4. Options Market Implied Sentiment
- **Provider**: CBOE, broker options data
- **Update Frequency**: 5-minute snapshots
- **Metrics**:
  - Put/Call ratio
  - Implied volatility skew
  - Options volume by strike and expiry
  - Max pain analysis

#### 5. Fear & Greed Index / VIX
- **Provider**: CNN Fear & Greed Index, CBOE VIX
- **Update Frequency**: Daily (Fear & Greed), real-time (VIX)
- **Metrics**:
  - Composite fear/greed score (0-100)
  - VIX level and rate of change
  - VIX term structure (contango/backwardation)

### Signals Generated

The Sentiment Monitor produces the following signal types:

| Signal Name | Type | Range | Description |
|-------------|------|-------|-------------|
| `sentiment_composite` | Float | -1.0 to +1.0 | Weighted aggregate sentiment score across all sources |
| `sentiment_momentum` | Float | -1.0 to +1.0 | Rate of change of sentiment (delta over smoothing window) |
| `sentiment_divergence` | Float | 0.0 to 1.0 | Divergence between news sentiment and social sentiment (0=aligned, 1=max divergence) |
| `fear_greed_index` | Integer | 0 to 100 | CNN Fear & Greed Index (or equivalent composite) |
| `volatility_regime` | Enum | `LOW`, `NORMAL`, `HIGH`, `EXTREME` | Categorized VIX level |
| `social_volume_spike` | Boolean | `true`/`false` | Indicates abnormal social mention volume (>2 std dev above mean) |
| `order_flow_sentiment` | Float | -1.0 to +1.0 | Market microstructure sentiment (positive=bullish flow, negative=bearish) |

### Smoothing Windows

To reduce noise and false signals, temporal smoothing is applied:

| Window Type | Duration | Application |
|-------------|----------|-------------|
| **Fast EMA** | 5 minutes | Real-time trading signals, scalping strategies |
| **Medium EMA** | 15 minutes | Intraday momentum strategies |
| **Slow EMA** | 1 hour | Swing trading, daily rebalancing |
| **Daily SMA** | 24 hours (rolling) | Long-term trend assessment, filter out intraday noise |

**Smoothing Formula (EMA)**:
```
EMA(t) = α × Raw(t) + (1 - α) × EMA(t-1)
where α = 2 / (N + 1), N = number of periods
```

**Crossover Signals**:
- **Bullish**: Fast EMA crosses above Medium EMA
- **Bearish**: Fast EMA crosses below Medium EMA
- **Regime Change**: Medium EMA crosses Slow EMA or Daily SMA

### Alerts Schema

Alerts are emitted when predefined conditions are met. Alerts are published to:
- Internal message queue (RabbitMQ/Kafka topic: `sentiment.alerts`)
- Trading dashboard (WebSocket push)
- Optionally: Email/SMS/Slack webhook for critical alerts

#### Alert Structure (JSON)

```json
{
  "alert_id": "uuid-v4",
  "timestamp": "2025-10-31T17:12:34.567Z",
  "severity": "INFO | WARNING | CRITICAL",
  "alert_type": "SENTIMENT_SPIKE | SENTIMENT_DIVERGENCE | VOLATILITY_REGIME_CHANGE | SOCIAL_VOLUME_ANOMALY",
  "ticker": "AAPL",
  "signal_name": "sentiment_composite",
  "current_value": 0.78,
  "threshold": 0.75,
  "description": "Composite sentiment for AAPL exceeded bullish threshold (0.75). Current: 0.78",
  "metadata": {
    "sources_contributing": ["news", "social", "options"],
    "confidence": 0.85,
    "recommendation": "CONSIDER_LONG_ENTRY"
  }
}
```

#### Alert Trigger Conditions

| Alert Type | Condition | Severity |
|------------|-----------|----------|
| `SENTIMENT_SPIKE` | `sentiment_composite` > 0.75 or < -0.75 | WARNING |
| `SENTIMENT_DIVERGENCE` | `sentiment_divergence` > 0.6 | INFO |
| `VOLATILITY_REGIME_CHANGE` | `volatility_regime` changes from `NORMAL` to `HIGH` or `EXTREME` | CRITICAL |
| `SOCIAL_VOLUME_ANOMALY` | `social_volume_spike` == true AND `sentiment_momentum` > 0.5 | WARNING |
| `FEAR_GREED_EXTREME` | `fear_greed_index` < 20 (extreme fear) or > 80 (extreme greed) | WARNING |

### Integration with Trading Strategies

#### Signal Consumption

Strategies subscribe to sentiment signals via:
1. **Real-time pub/sub**: Kafka topic `sentiment.signals` (latest values)
2. **Time-series DB query**: InfluxDB/TimescaleDB for historical backtesting
3. **REST API**: `GET /api/v1/sentiment/{ticker}?window=15m` for on-demand queries

#### Example Strategy Integration

```python
# Pseudocode: Sentiment-enhanced momentum strategy
def generate_trade_signal(ticker, market_data, sentiment_data):
    price_momentum = calculate_momentum(market_data)
    sentiment_score = sentiment_data['sentiment_composite']
    
    # Only take long positions if sentiment is positive
    if price_momentum > 0 and sentiment_score > 0.3:
        return "BUY"
    # Only take short positions if sentiment is negative
    elif price_momentum < 0 and sentiment_score < -0.3:
        return "SELL"
    else:
        return "HOLD"
```

### Performance & Scaling

- **Latency Target**: <100ms from data ingestion to signal publication
- **Throughput**: Support 500+ tickers simultaneously
- **Availability**: 99.9% uptime during market hours
- **Data Retention**: Raw data 30 days, aggregated signals 1 year

### Tests & Validation

#### Unit Tests
- Data ingestion parsers (NewsAPI, Twitter, options data)
- Signal calculation functions (composite score, EMA smoothing, divergence)
- Alert trigger logic (threshold checks, severity assignment)

#### Integration Tests
- End-to-end data flow: ingestion → processing → signal publication → alert emission
- Message queue reliability (RabbitMQ/Kafka)
- Database writes and reads (InfluxDB/TimescaleDB)

#### Backtesting
- Historical sentiment data replay (2020-2024)
- Signal accuracy vs. actual market moves (correlation analysis)
- Alert precision/recall metrics (false positive rate)

#### Stress Tests
- High-volume social data spikes (e.g., earnings announcements)
- API rate limit handling and fallback logic
- Database write throughput under load

**Test Coverage Target**: >80% for all sentiment monitor modules

**Test Automation**: CI/CD pipeline runs tests on every commit (GitHub Actions)

### Monitoring & Observability

- **Metrics**: Prometheus metrics for latency, throughput, error rates
- **Dashboards**: Grafana dashboards for real-time signal visualization
- **Logging**: Structured JSON logs (ELK stack or Loki)
- **Tracing**: Distributed tracing with Jaeger for request flow analysis

### Configuration

Sentiment monitor configuration is managed via YAML config files:

```yaml
# config/sentiment_monitor.yaml
data_sources:
  news:
    enabled: true
    provider: "newsapi"
    api_key: "${NEWS_API_KEY}"
    update_interval: "15m"
  
  social:
    enabled: true
    providers:
      - twitter
      - reddit
    update_interval: "realtime"
  
  options:
    enabled: true
    provider: "tdameritrade"
    update_interval: "5m"

signals:
  sentiment_composite:
    weight_news: 0.4
    weight_social: 0.3
    weight_microstructure: 0.2
    weight_options: 0.1
  
  smoothing:
    fast_ema_periods: 5
    medium_ema_periods: 15
    slow_ema_periods: 60

alerts:
  enabled: true
  channels:
    - kafka
    - websocket
    - slack  # optional
  
  thresholds:
    sentiment_spike: 0.75
    divergence: 0.6
    fear_greed_extreme_low: 20
    fear_greed_extreme_high: 80
```

---

## Future Enhancements

- **LLM-based news summarization**: Use GPT-4 or Claude to summarize earnings calls and filings
- **Alternative data**: Satellite imagery, credit card transaction data, web traffic
- **Multi-asset sentiment**: Extend beyond equities to crypto, forex, commodities
- **Explainability**: SHAP values or LIME for signal attribution

---

## References

- [Alpha Vantage News Sentiment API](https://www.alphavantage.co/documentation/#news-sentiment)
- [FinBERT: Financial Sentiment Analysis](https://huggingface.co/ProsusAI/finbert)
- [CBOE VIX White Paper](https://cdn.cboe.com/resources/vix/vixwhite.pdf)
- [Market Microstructure Theory](https://www.investopedia.com/terms/m/microstructure.asp)

---

**Document Version**: 1.0  
**Last Updated**: October 2025  
**Owner**: Axiom Market Matrix Team
