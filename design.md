# Design Document: Vyapari Mitra AI

## Overview

Vyapari Mitra AI is a voice-first business assistant designed to empower small and rural retailers in India with AI-driven insights for inventory management, demand forecasting, and pricing intelligence. The system addresses a critical gap in the market by providing an accessible interface for semi-literate users who are not comfortable with traditional dashboard-based analytics tools.

The solution leverages AWS cloud services to deliver a scalable, secure, and cost-effective platform that processes natural language queries in Hindi and regional languages, providing actionable business insights through voice responses. The system is built on responsible AI principles, clearly communicating that all predictions are probabilistic estimates to support decision-making rather than replace human judgment.

## Design Principles

1. **Voice-First Experience**: All interactions are optimized for natural speech in regional languages, eliminating the need for reading or typing.

2. **Simplicity and Accessibility**: Responses are delivered in conversational language appropriate for semi-literate users, using familiar cultural references and common units of measurement.

3. **Transparency and Trust**: The system clearly communicates that predictions are probabilistic estimates, not guarantees, and explains the key factors influencing each recommendation.

4. **Privacy by Design**: The system uses only synthetic POS datasets and publicly available data for training, with secure handling of retailer-specific information.

5. **Scalability and Cost-Effectiveness**: Serverless AWS architecture enables automatic scaling based on demand while minimizing operational costs.

6. **Extensibility**: Modular design allows easy addition of new regional languages and business features without architectural changes.

## High-Level AWS Architecture

![Architecture Diagram](https://file%2B.vscode-resource.vscode-cdn.net/Users/yerramsetti_meghana%40optum.com/Documents/Learning/vyapari-mitra-ai/hackathon_architecture.jpg?version%3D1770888552152)

## Component Descriptions

### Amazon API Gateway
Serves as the entry point for all client requests, providing secure RESTful APIs for voice input submission and response retrieval. The gateway handles request routing, throttling, and integrates with AWS IAM for authentication and authorization. Rate limiting is configured to prevent abuse and ensure fair resource allocation across retailers.

### AWS Transcribe (Speech-to-Text)
Converts retailer voice queries from audio to text in Hindi and one additional regional language. AWS Transcribe supports Indian English and Hindi natively, with custom vocabulary configuration to improve accuracy for business-specific terms like product names and units. The service provides confidence scores for each transcription, enabling the system to request clarification when accuracy is uncertain.

### AWS Polly (Text-to-Speech)
Converts system-generated text responses into natural-sounding speech in the retailer's preferred language. AWS Polly offers neural TTS voices for Hindi that sound conversational and natural. The service supports SSML (Speech Synthesis Markup Language) for controlling pronunciation, emphasis, and pacing to ensure responses are clear and easy to understand.

### Query Understanding Lambda Function
Processes the transcribed text to extract intent and entities using natural language understanding techniques. This function identifies what the retailer is asking (demand forecast, reorder recommendation, pricing inquiry, etc.) and extracts relevant entities such as product names, time periods, and quantities. The function handles code-mixing between languages and uses context from previous queries to improve understanding in multi-turn conversations.

### Business Orchestrator Lambda Function
Acts as the central coordinator that routes queries to appropriate analytics services, aggregates results, and applies business rules. This function manages the conversation flow, retrieves retailer context from the database, checks the cache for frequently asked queries, and ensures responses are generated within the 5-second latency requirement. The orchestrator implements graceful degradation strategies when services are unavailable.

### Response Generator Lambda Function
Transforms structured analytics results into natural language responses appropriate for the retailer's language and literacy level. This function uses template-based generation with dynamic slot filling, formats numbers using culturally appropriate conventions, and ensures all predictions include probabilistic disclaimers. The generator adapts language complexity and uses familiar cultural references to make insights accessible.

### Amazon SageMaker - Demand Forecasting Model
Hosts machine learning models that predict next-day and next-week demand for inventory items based on historical POS data. The forecasting approach uses time series analysis techniques to identify trends, seasonality, and patterns. Models are trained on synthetic POS datasets and continuously updated as new retailer data becomes available. The service provides prediction intervals (ranges) rather than point estimates to communicate uncertainty.

### Amazon SageMaker - Seasonal Pattern Detection
Analyzes historical sales data to identify recurring patterns associated with festivals, holidays, agricultural cycles, and weekly trends. This model detects patterns relevant to the Indian retail context, such as increased demand during Diwali, harvest seasons, and weekends. Detected patterns are used to adjust demand forecasts and provide proactive recommendations for upcoming seasonal events.

### Amazon SageMaker - Pricing Intelligence Model
Generates pricing recommendations by analyzing government mandi price data and open retail datasets. The model considers regional price variations, seasonal trends, and the retailer's cost structure when provided. Recommendations are presented as price ranges to account for market variability and local conditions.

### Amazon RDS (PostgreSQL)
Stores structured data including retailer profiles, product catalogs, POS transactions, inventory snapshots, and market price data. The database is configured with read replicas for query performance and automated backups for data durability. Time-series data is optimized using partitioning strategies to enable efficient historical analysis.

### Amazon S3
Stores synthetic POS datasets used for model training, trained model artifacts, and public market datasets. S3 provides durable, cost-effective storage with lifecycle policies to archive older datasets. The service integrates seamlessly with SageMaker for model training and deployment.

### Amazon ElastiCache (Redis)
Caches frequently accessed data such as recent forecasts, market prices, and common query responses to reduce latency and database load. The cache implements a time-to-live (TTL) strategy to ensure data freshness while maximizing cache hit rates. This is critical for meeting the 5-second response time requirement.

### AWS CloudWatch
Provides comprehensive monitoring and logging for all system components. CloudWatch tracks key metrics including API request rates, Lambda function execution times, SageMaker endpoint latency, error rates, and cache hit rates. Alarms are configured to notify operators of performance degradation or service failures. Logs are retained for debugging and compliance purposes.

### AWS IAM
Manages authentication and authorization for API access, ensuring only authorized clients can submit queries. IAM roles are used to grant Lambda functions and SageMaker endpoints least-privilege access to required AWS resources. This implements defense-in-depth security principles.

## End-to-End Data Flow

### Demand Forecast Query Flow

When a retailer asks "कल चावल की कितनी बिक्री होगी?" (How much rice will sell tomorrow?), the system processes the request through the following steps:

1. **Voice Input**: The mobile app or IVR system captures the audio and sends it to Amazon API Gateway.

2. **Authentication**: API Gateway validates the request using AWS IAM credentials.

3. **Speech-to-Text**: AWS Transcribe converts the Hindi audio to text: "कल चावल की कितनी बिक्री होगी" with a confidence score.

4. **Query Understanding**: The Query Understanding Lambda function analyzes the text, identifying the intent as "demand_forecast" and extracting entities (product: rice, time_period: tomorrow).

5. **Cache Check**: The Business Orchestrator Lambda checks ElastiCache Redis for a recent forecast for this retailer and product.

6. **Forecast Generation** (if not cached): The orchestrator invokes the SageMaker demand forecasting endpoint, which retrieves 60 days of historical sales data from RDS, applies time series analysis with seasonal adjustments, and generates a prediction range (15-20 kg) with confidence score (0.78).

7. **Response Generation**: The Response Generator Lambda creates a natural language response in Hindi: "कल चावल की बिक्री लगभग 15 से 20 किलो हो सकती है। यह अनुमान पिछले दो महीने की बिक्री पर आधारित है। कृपया ध्यान दें कि यह अनुमान है, पक्की गारंटी नहीं।" (Tomorrow's rice sales could be approximately 15-20 kg. This estimate is based on the last two months of sales. Please note this is an estimate, not a guarantee.)

8. **Text-to-Speech**: AWS Polly converts the Hindi text to natural-sounding speech audio.

9. **Response Delivery**: The audio is streamed back through API Gateway to the client application.

Total end-to-end latency: approximately 4 seconds.

### Reorder Recommendation Query Flow

When a retailer asks "मुझे दाल कब मंगानी चाहिए?" (When should I order lentils?), the system follows a similar flow but involves multiple analytics components:

1. The query is transcribed and understood as a "reorder_recommendation" intent for lentils.

2. The Business Orchestrator retrieves the current inventory level from RDS (5 kg of lentils).

3. The orchestrator invokes the demand forecasting endpoint to get predicted daily demand (3 kg/day).

4. Using the forecast, current stock, and configured lead time (2 days), the system calculates the reorder quantity: (3 kg/day × 2 days) + safety stock (1.5 kg) - current stock (5 kg) = 2.5 kg, rounded to 3 kg.

5. The response is generated: "आपके पास अभी 5 किलो दाल है। अगले 2 दिन में 6 किलो बिक सकती है। आज 3 किलो दाल मंगा लें ताकि स्टॉक खत्म न हो।" (You currently have 5 kg of lentils. About 6 kg may sell in the next 2 days. Order 3 kg of lentils today so stock doesn't run out.)

6. The response is converted to speech and delivered to the retailer.



## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Voice Processing Properties

**Property 1: Speech-to-text accuracy for business queries**
*For any* business-related voice query in Hindi or the supported regional language, when converted to speech and then processed through STT, the resulting text should preserve the business intent with sufficient accuracy for query understanding.
**Validates: Requirements 1.1**

**Property 2: Low-confidence STT triggers clarification**
*For any* speech input that produces STT output with confidence below the threshold, the system should request the retailer to repeat the question rather than proceeding with potentially incorrect text.
**Validates: Requirements 1.2**

**Property 3: Response language matches query language**
*For any* query in a supported language, the system's response should be generated and delivered in the same language as the input query.
**Validates: Requirements 2.1**

**Property 4: Numerical and unit formatting consistency**
*For any* response containing numerical information, the system should format numbers using appropriate rounding and use units that are familiar to the retailer's regional context.
**Validates: Requirements 2.3, 4.4**

**Property 5: Probabilistic disclaimers in predictions**
*For any* response containing predictions or forecasts, the system should include clear language communicating that the output is probabilistic and not a guarantee.
**Validates: Requirements 2.5, 9.1**

### Forecasting Properties

**Property 6: Demand forecasts generated for valid requests**
*For any* valid inventory item with historical POS data, when a retailer requests a demand forecast, the forecasting engine should produce next-day and next-week predictions.
**Validates: Requirements 3.1**

**Property 7: Minimum historical data usage**
*For any* forecasting request where at least 30 days of historical sales data is available, the forecasting engine should use at least that amount of data in generating predictions.
**Validates: Requirements 3.2**

**Property 8: Seasonal pattern incorporation**
*For any* product with identifiable seasonal patterns in historical data, the demand forecast should reflect adjustments based on those seasonal trends.
**Validates: Requirements 3.4**

**Property 9: Range-based predictions**
*For any* forecast or pricing recommendation, the system should express the output as a range (minimum and maximum values) rather than a single exact number.
**Validates: Requirements 3.5, 7.4**

### Inventory Management Properties

**Property 10: Reorder recommendations generated**
*For any* inventory item with current stock level and demand forecast, when a retailer requests reorder guidance, the system should calculate and provide a reorder quantity recommendation.
**Validates: Requirements 4.1**

**Property 11: Reorder calculation incorporates key factors**
*For any* reorder recommendation, the calculated quantity should appropriately change when forecasted demand, current stock level, or lead time changes, following the relationship: higher demand or longer lead time increases reorder quantity, while higher current stock decreases it.
**Validates: Requirements 4.2**

**Property 12: Storage capacity constraints respected**
*For any* reorder recommendation where the retailer has provided storage capacity constraints, the recommended quantity should not exceed the available storage capacity.
**Validates: Requirements 4.3**

**Property 13: Low-stock alerts generated**
*For any* inventory item where current quantity falls below the threshold calculated from forecasted demand and lead time, the system should generate a low-stock alert.
**Validates: Requirements 5.1**

**Property 14: Overstock alerts generated**
*For any* inventory item where current quantity exceeds optimal levels based on forecasted demand and shelf life considerations, the system should generate an overstock alert.
**Validates: Requirements 5.2**

**Property 15: Alert prioritization by urgency**
*For any* set of inventory alerts, the system should order them by urgency level, with critical alerts appearing before high, medium, and low priority alerts.
**Validates: Requirements 5.3, 5.4**

**Property 16: Alert queries return correct results**
*For any* query for alerts on a specific item or category, the system should return only alerts that match the specified item or belong to the specified category.
**Validates: Requirements 5.5**

### Seasonal and Pricing Properties

**Property 17: Seasonal pattern detection**
*For any* sales data containing recurring patterns with consistent timing and magnitude, the system should identify and describe those patterns as seasonal trends.
**Validates: Requirements 6.1**

**Property 18: Festival and agricultural cycle detection**
*For any* sales data with spikes corresponding to known festivals, holidays, or agricultural cycles in the retailer's region, the system should detect and attribute those patterns to the relevant events.
**Validates: Requirements 6.2**

**Property 19: Seasonal recommendations generated**
*For any* upcoming seasonal event identified in the system, the system should provide actionable inventory recommendations related to that event.
**Validates: Requirements 6.4**

**Property 20: Pricing recommendations generated**
*For any* inventory item, when a retailer requests pricing guidance, the system should provide pricing intelligence based on available public market data.
**Validates: Requirements 7.1**

**Property 21: Market data sources utilized**
*For any* pricing recommendation, the system should query and incorporate data from government mandi prices or open retail datasets when generating the recommendation.
**Validates: Requirements 7.2**

**Property 22: Retailer cost and margin incorporated**
*For any* pricing recommendation where the retailer has provided their cost and desired margin, the suggested price range should ensure the margin requirement is met at the recommended prices.
**Validates: Requirements 7.3**

### Transparency and Explainability Properties

**Property 23: Influencing factors explained**
*For any* recommendation or prediction, the system's response should include a list or description of the key factors that influenced the output.
**Validates: Requirements 9.2**

**Property 24: Low-confidence warnings**
*For any* prediction with confidence below a defined threshold, the system should explicitly inform the retailer that the prediction has lower confidence.
**Validates: Requirements 9.4**

### Multilingual Properties

**Property 25: Context preservation across language switches**
*For any* conversation where the retailer switches from one supported language to another, the system should maintain the conversation context and continue providing relevant responses.
**Validates: Requirements 10.2**

### API Properties

**Property 26: Authentication enforcement**
*For any* API request without valid authentication credentials, the system should reject the request and return an authentication error.
**Validates: Requirements 12.2**

**Property 27: Descriptive error messages**
*For any* failed API request, the system should return an error response that includes a descriptive message explaining the reason for failure.
**Validates: Requirements 12.4**

**Property 28: Rate limiting enforcement**
*For any* client making API requests, when the number of requests exceeds the defined rate limit within the time window, subsequent requests should be rejected with a rate limit error.
**Validates: Requirements 12.5**

## Error Handling

### Error Categories

The system handles errors across multiple layers:

1. **Voice Processing Errors**
   - STT failure or low confidence
   - TTS service unavailable
   - Unsupported language detected
   - Audio quality too poor to process

2. **Query Understanding Errors**
   - Ambiguous intent (cannot determine what retailer is asking)
   - Missing required entities (e.g., product name not specified)
   - Unsupported query type
   - Code-mixing that cannot be parsed

3. **Data Errors**
   - Insufficient historical data for forecasting
   - Missing product information
   - Stale or outdated market data
   - Database connection failures

4. **Business Logic Errors**
   - Invalid inventory state (negative stock)
   - Forecasting model failure
   - Calculation errors (division by zero, etc.)
   - Constraint violations (reorder exceeds storage)

5. **External Service Errors**
   - Cloud STT/TTS API failures
   - Public dataset API unavailable
   - Network timeouts
   - Rate limiting from external services

### Error Handling Strategies

**Graceful Degradation:**
```python
def get_demand_forecast(product_id: str, retailer_id: str) -> DemandForecast:
    try:
        # Try primary forecasting model (Prophet)
        return prophet_forecast(product_id, retailer_id)
    except InsufficientDataError:
        # Fall back to simple moving average
        return moving_average_forecast(product_id, retailer_id)
    except ModelFailureError:
        # Fall back to regional average from public data
        return regional_average_forecast(product_id)
    except Exception as e:
        # Log error and return error response
        logger.error(f"Forecast failed: {e}")
        raise ForecastingUnavailableError("Unable to generate forecast")
```

**User-Friendly Error Messages:**

Instead of technical errors, the system translates errors into natural language:

```python
ERROR_MESSAGES = {
    "insufficient_data": {
        "hi": "माफ़ करें, इस उत्पाद के लिए पर्याप्त बिक्री डेटा नहीं है। कृपया कुछ हफ्ते बाद फिर से पूछें।",
        "en": "Sorry, there isn't enough sales data for this product. Please try again in a few weeks."
    },
    "product_not_found": {
        "hi": "मुझे यह उत्पाद नहीं मिला। कृपया उत्पाद का नाम फिर से बताएं।",
        "en": "I couldn't find this product. Please tell me the product name again."
    },
    "service_unavailable": {
        "hi": "माफ़ करें, अभी सेवा उपलब्ध नहीं है। कृपया कुछ देर बाद फिर से कोशिश करें।",
        "en": "Sorry, the service is currently unavailable. Please try again in a few moments."
    }
}
```

**Retry Logic:**

For transient failures (network issues, temporary service unavailability):

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    retry=retry_if_exception_type(TransientError)
)
def call_external_api(endpoint: str, params: dict) -> dict:
    response = requests.get(endpoint, params=params, timeout=5)
    response.raise_for_status()
    return response.json()
```

**Circuit Breaker Pattern:**

For external dependencies that may fail repeatedly:

```python
class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, timeout: int = 60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "closed"  # closed, open, half_open
    
    def call(self, func, *args, **kwargs):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "half_open"
            else:
                raise CircuitBreakerOpenError("Service unavailable")
        
        try:
            result = func(*args, **kwargs)
            if self.state == "half_open":
                self.state = "closed"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "open"
            raise e
```

### Logging and Monitoring

All errors are logged with appropriate context for debugging:

```python
logger.error(
    "Forecasting failed",
    extra={
        "retailer_id": retailer_id,
        "product_id": product_id,
        "error_type": type(e).__name__,
        "error_message": str(e),
        "stack_trace": traceback.format_exc()
    }
)
```

Metrics tracked:
- Error rate by error type
- STT/TTS success rate
- Forecasting model accuracy
- API response times
- Cache hit rates

## Testing Strategy

### Dual Testing Approach

The system requires both unit testing and property-based testing for comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs

Both approaches are complementary and necessary. Unit tests catch concrete bugs and validate specific scenarios, while property tests verify general correctness across a wide range of inputs.

### Unit Testing

**Focus Areas:**
- Specific examples demonstrating correct behavior
- Edge cases (empty data, boundary values, special characters)
- Error conditions and exception handling
- Integration points between components
- Mock external dependencies (STT/TTS APIs, databases)

**Example Unit Tests:**

```python
def test_demand_forecast_with_30_days_data():
    """Test that forecast uses 30 days of data when available."""
    historical_data = generate_sales_data(days=45)
    forecast = forecasting_engine.forecast_demand(
        product_id="rice_001",
        retailer_id="ret_123",
        forecast_horizon="next_day"
    )
    assert forecast.data_quality == "sufficient"
    assert forecast.predicted_quantity_min > 0
    assert forecast.predicted_quantity_max > forecast.predicted_quantity_min

def test_reorder_respects_storage_capacity():
    """Test that reorder recommendation doesn't exceed storage."""
    recommendation = inventory_analyzer.calculate_reorder_quantity(
        product_id="rice_001",
        retailer_id="ret_123",
        current_stock=10.0,
        demand_forecast=DemandForecast(predicted_quantity_mean=50.0),
        lead_time_days=2,
        storage_capacity=30.0
    )
    assert recommendation.recommended_quantity <= 30.0

def test_error_message_for_insufficient_data():
    """Test that appropriate error is returned for insufficient data."""
    historical_data = generate_sales_data(days=5)  # Less than 30 days
    with pytest.raises(InsufficientDataError) as exc_info:
        forecasting_engine.forecast_demand(
            product_id="new_product",
            retailer_id="ret_123",
            forecast_horizon="next_week"
        )
    assert "insufficient" in str(exc_info.value).lower()
```

### Property-Based Testing

**Testing Library:** Use Hypothesis (Python) for property-based testing

**Configuration:**
- Minimum 100 iterations per property test
- Each test tagged with feature name and property number
- Tag format: `# Feature: vyapari-mitra-ai, Property N: [property text]`

**Example Property Tests:**

```python
from hypothesis import given, strategies as st
import hypothesis

# Feature: vyapari-mitra-ai, Property 9: Range-based predictions
@given(
    historical_data=st.lists(
        st.floats(min_value=0, max_value=1000),
        min_size=30,
        max_size=365
    )
)
@hypothesis.settings(max_examples=100)
def test_forecasts_are_ranges(historical_data):
    """Property: All forecasts should be expressed as ranges."""
    forecast = forecasting_engine.forecast_demand(
        product_id="test_product",
        retailer_id="test_retailer",
        forecast_horizon="next_day"
    )
    assert forecast.predicted_quantity_min < forecast.predicted_quantity_max
    assert forecast.predicted_quantity_mean >= forecast.predicted_quantity_min
    assert forecast.predicted_quantity_mean <= forecast.predicted_quantity_max

# Feature: vyapari-mitra-ai, Property 11: Reorder calculation incorporates key factors
@given(
    current_stock=st.floats(min_value=0, max_value=100),
    daily_demand=st.floats(min_value=1, max_value=50),
    lead_time=st.integers(min_value=1, max_value=7)
)
@hypothesis.settings(max_examples=100)
def test_reorder_quantity_relationships(current_stock, daily_demand, lead_time):
    """Property: Reorder quantity should increase with demand and lead time."""
    forecast = DemandForecast(predicted_quantity_mean=daily_demand)
    
    rec1 = inventory_analyzer.calculate_reorder_quantity(
        product_id="test",
        retailer_id="test",
        current_stock=current_stock,
        demand_forecast=forecast,
        lead_time_days=lead_time
    )
    
    # Increase demand, reorder should increase
    forecast_higher = DemandForecast(predicted_quantity_mean=daily_demand * 1.5)
    rec2 = inventory_analyzer.calculate_reorder_quantity(
        product_id="test",
        retailer_id="test",
        current_stock=current_stock,
        demand_forecast=forecast_higher,
        lead_time_days=lead_time
    )
    
    assert rec2.recommended_quantity >= rec1.recommended_quantity

# Feature: vyapari-mitra-ai, Property 5: Probabilistic disclaimers in predictions
@given(
    query_text=st.text(min_size=10, max_size=100),
    forecast_data=st.builds(DemandForecast)
)
@hypothesis.settings(max_examples=100)
def test_responses_include_disclaimers(query_text, forecast_data):
    """Property: All prediction responses should include probabilistic disclaimers."""
    response = response_generator.generate_response(
        business_response=BusinessResponse(forecast=forecast_data),
        language="hi",
        retailer_context=RetailerContext()
    )
    
    # Check for disclaimer keywords in Hindi
    disclaimer_keywords = ["अनुमान", "संभावना", "हो सकता है", "लगभग"]
    assert any(keyword in response for keyword in disclaimer_keywords)

# Feature: vyapari-mitra-ai, Property 15: Alert prioritization by urgency
@given(
    alerts=st.lists(
        st.builds(
            InventoryAlert,
            urgency=st.sampled_from(["critical", "high", "medium", "low"])
        ),
        min_size=2,
        max_size=20
    )
)
@hypothesis.settings(max_examples=100)
def test_alerts_sorted_by_urgency(alerts):
    """Property: Alerts should be ordered by urgency level."""
    sorted_alerts = inventory_analyzer.prioritize_alerts(alerts)
    
    urgency_order = {"critical": 0, "high": 1, "medium": 2, "low": 3}
    
    for i in range(len(sorted_alerts) - 1):
        current_priority = urgency_order[sorted_alerts[i].urgency]
        next_priority = urgency_order[sorted_alerts[i + 1].urgency]
        assert current_priority <= next_priority

# Feature: vyapari-mitra-ai, Property 26: Authentication enforcement
@given(
    endpoint=st.sampled_from(["/forecast", "/reorder", "/pricing"]),
    has_valid_token=st.booleans()
)
@hypothesis.settings(max_examples=100)
def test_api_authentication_required(endpoint, has_valid_token):
    """Property: All API requests without valid auth should be rejected."""
    headers = {}
    if has_valid_token:
        headers["Authorization"] = "Bearer valid_token_123"
    
    response = api_client.get(endpoint, headers=headers)
    
    if has_valid_token:
        assert response.status_code != 401
    else:
        assert response.status_code == 401
```

### Integration Testing

**Focus Areas:**
- End-to-end voice query flow
- Database interactions
- External API integrations (with test/staging endpoints)
- Multi-component workflows

**Example Integration Test:**

```python
def test_end_to_end_demand_forecast_query():
    """Test complete flow from voice input to voice output."""
    # Simulate voice input
    audio_input = generate_test_audio("कल चावल की कितनी बिक्री होगी?")
    
    # Process through system
    stt_result = voice_interface.speech_to_text(audio_input, language_hint="hi")
    assert stt_result.confidence > 0.7
    
    parsed_query = query_understanding.parse_query(
        stt_result.text,
        language="hi",
        retailer_context=test_retailer_context
    )
    assert parsed_query.intent == "demand_forecast"
    
    business_response = orchestrator.process_query(
        parsed_query,
        test_retailer_context
    )
    assert business_response.forecast is not None
    
    response_text = response_generator.generate_response(
        business_response,
        language="hi",
        retailer_context=test_retailer_context
    )
    assert "अनुमान" in response_text  # Contains disclaimer
    
    audio_output = voice_interface.text_to_speech(response_text, language="hi")
    assert audio_output.duration > 0
```

### Performance Testing

**Load Testing:**
- Simulate 100+ concurrent voice sessions
- Measure end-to-end latency (target: <5 seconds)
- Monitor resource usage and auto-scaling behavior

**Tools:**
- Locust or JMeter for load generation
- Prometheus + Grafana for monitoring
- Cloud provider monitoring dashboards

### Test Data Management

**Synthetic POS Data Generation:**

```python
def generate_synthetic_pos_data(
    num_retailers: int = 100,
    num_products: int = 50,
    days: int = 365
) -> List[Transaction]:
    """Generate realistic synthetic POS transactions."""
    transactions = []
    
    for retailer_id in range(num_retailers):
        for product_id in range(num_products):
            # Generate daily sales with seasonal patterns
            for day in range(days):
                date = datetime.now() - timedelta(days=days - day)
                
                # Base demand with weekly and seasonal patterns
                base_demand = random.uniform(5, 20)
                weekly_factor = 1.2 if date.weekday() in [5, 6] else 1.0
                seasonal_factor = 1.5 if date.month in [10, 11] else 1.0
                
                quantity = base_demand * weekly_factor * seasonal_factor
                quantity += random.gauss(0, quantity * 0.1)  # Add noise
                
                if quantity > 0:
                    transactions.append(Transaction(
                        transaction_id=f"txn_{len(transactions)}",
                        retailer_id=f"ret_{retailer_id}",
                        product_id=f"prod_{product_id}",
                        quantity=round(quantity, 2),
                        unit_price=random.uniform(10, 100),
                        total_amount=round(quantity * random.uniform(10, 100), 2),
                        timestamp=date,
                        payment_method=random.choice(["cash", "upi", "card"])
                    ))
    
    return transactions
```

### Continuous Testing

**CI/CD Pipeline:**
1. Run unit tests on every commit
2. Run property tests on every pull request
3. Run integration tests on staging deployment
4. Run performance tests weekly or before major releases

**Test Coverage Goals:**
- Unit test coverage: >80% for business logic
- Property test coverage: All properties from design document
- Integration test coverage: All major user flows
- API test coverage: All endpoints

### Monitoring and Observability in Production

**Key Metrics:**
- Voice query success rate
- Average response latency
- Forecasting accuracy (compare predictions to actual sales)
- User satisfaction (feedback ratings)
- Error rates by type
- Cache hit rates

**Alerting:**
- Alert if error rate exceeds 5%
- Alert if average latency exceeds 7 seconds
- Alert if STT/TTS success rate drops below 90%
- Alert if database connections fail


## Model Design Approach

### Demand Forecasting Model

The demand forecasting model predicts next-day and next-week sales for inventory items using time series analysis. The approach combines statistical methods with machine learning to handle the diverse patterns found in small retail businesses.

**Data Preprocessing**: Historical POS transaction data is cleaned to handle missing values and outliers. Daily sales are aggregated by product, and features are engineered including day of week, month, proximity to festivals, and historical averages. For products with insufficient individual history, category-level patterns are used.

**Model Architecture**: The system uses an ensemble approach with multiple forecasting techniques. For products with at least 30 days of history, time series decomposition identifies trend, seasonality, and residual components. Statistical models like ARIMA or exponential smoothing capture short-term patterns, while machine learning models (gradient boosting) incorporate external features like festivals and regional events. For products with limited history, simple moving averages or category-based forecasts provide baseline predictions.

**Seasonal Adjustment**: The model explicitly accounts for Indian retail patterns including weekly cycles (higher weekend sales), monthly cycles (salary-driven purchasing), festival seasons (Diwali, Holi, Eid), agricultural cycles (harvest seasons), and regional events. A calendar of Indian festivals and regional events is maintained and used to adjust forecasts.

**Uncertainty Quantification**: Rather than providing point estimates, the model generates prediction intervals representing the range of likely outcomes. This is achieved through quantile regression or bootstrapping techniques. The confidence level is communicated to retailers to set appropriate expectations.

**Model Training and Updates**: Models are initially trained on synthetic POS datasets that simulate realistic Indian retail patterns. As retailer-specific data accumulates, models are fine-tuned using transfer learning. Models are retrained weekly to incorporate recent trends, with automated performance monitoring to detect degradation.

### Seasonal Pattern Detection

This component identifies recurring patterns in sales data that correspond to predictable events. The detection algorithm analyzes historical sales to find statistically significant spikes or dips that recur at consistent intervals.

**Pattern Types**: The system detects weekly patterns (weekend vs weekday), monthly patterns (beginning vs end of month), festival patterns (Diwali, Holi, Eid, regional festivals), agricultural patterns (sowing and harvest seasons), and holiday patterns (national and regional holidays).

**Detection Method**: Time series decomposition separates trend, seasonal, and residual components. Peaks in the seasonal component are matched against a calendar of known events. Statistical tests (like autocorrelation analysis) confirm the significance of detected patterns. Machine learning clustering identifies previously unknown recurring patterns.

**Regional Customization**: The system maintains region-specific event calendars. For example, Pongal is emphasized for Tamil Nadu retailers, while Onam is highlighted for Kerala retailers. Agricultural cycles are customized based on the dominant crops in each region.

**Actionable Insights**: Detected patterns are translated into proactive recommendations. For example, if Diwali is approaching and historical data shows a 2x increase in sweets sales, the system proactively suggests increasing sweets inventory.

### Pricing Intelligence Model

The pricing intelligence component helps retailers set competitive prices while maintaining profitability. It analyzes public market data to provide context-aware pricing recommendations.

**Data Sources**: The model integrates government mandi (wholesale market) price data from the Agmarknet portal, which provides daily prices for agricultural commodities across India. Open retail price datasets and surveys provide retail-level pricing context. Regional price indices account for geographic variations in pricing.

**Recommendation Logic**: For each product, the model retrieves recent market prices from the retailer's region. If the retailer provides their cost and desired margin, the model ensures recommendations meet profitability requirements. Prices are presented as ranges (e.g., ₹40-45 per kg) rather than exact values to account for local market conditions and quality variations.

**Competitive Positioning**: The model considers the retailer's business type and location. Rural retailers typically have different pricing dynamics than urban retailers. The system provides context like "This price is slightly above the regional average" to help retailers understand their positioning.

**Trend Analysis**: The model tracks price trends over time, alerting retailers to significant increases or decreases in market prices. This helps retailers adjust their purchasing and pricing strategies proactively.

## Scalability Strategy

The AWS serverless architecture enables automatic scaling to handle varying loads without manual intervention.

**Lambda Auto-Scaling**: AWS Lambda functions automatically scale by running multiple concurrent instances as request volume increases. Each function is configured with appropriate memory allocation (typically 512MB-1GB) and timeout settings (5-10 seconds) to balance performance and cost.

**SageMaker Endpoint Scaling**: SageMaker model endpoints are configured with auto-scaling policies based on invocation rate and latency metrics. During peak hours (typically morning and evening when retailers are most active), additional instances are automatically provisioned. During off-peak hours, the system scales down to minimize costs.

**Database Optimization**: Amazon RDS uses read replicas to distribute query load. Frequently accessed data (recent transactions, active inventory) is cached in ElastiCache Redis with appropriate TTL settings. Database queries are optimized with proper indexing on commonly filtered columns (retailer_id, product_id, timestamp).

**API Gateway Throttling**: API Gateway implements rate limiting to prevent abuse and ensure fair resource allocation. Burst limits allow temporary spikes while sustained rate limits prevent any single client from overwhelming the system.

**Cost Optimization**: The serverless architecture means costs scale with actual usage. During the initial rollout with limited retailers, costs remain low. As adoption grows, the system scales automatically while maintaining per-query costs through efficient resource utilization.

**Geographic Distribution**: For future expansion, AWS CloudFront can be used to cache static content and reduce latency for geographically distributed retailers. Multi-region deployment can be implemented if needed for disaster recovery or to serve retailers across India with lower latency.

## Security and Data Privacy

Security is implemented through multiple layers of defense to protect retailer data and ensure system integrity.

**Authentication and Authorization**: API Gateway requires authentication using AWS IAM credentials or API keys. Each retailer is assigned unique credentials that grant access only to their own data. JWT tokens with short expiration times are used for session management.

**Data Encryption**: All data in transit is encrypted using TLS 1.2 or higher. Data at rest in RDS and S3 is encrypted using AWS KMS (Key Management Service) with customer-managed keys. Sensitive fields like phone numbers are additionally encrypted at the application layer.

**Access Control**: AWS IAM roles implement least-privilege access. Lambda functions can only access the specific RDS tables and S3 buckets they need. SageMaker endpoints are accessible only from authorized Lambda functions. Database access is restricted to specific IP ranges and requires authentication.

**Data Isolation**: Each retailer's data is logically isolated using retailer_id as a partition key. Database queries always include retailer_id in the WHERE clause to prevent accidental cross-retailer data access. Multi-tenancy is implemented securely without physical data separation.

**Audit Logging**: All API requests, database queries, and model invocations are logged to CloudWatch. Logs include timestamps, user identifiers, actions performed, and outcomes. Logs are retained for compliance and security auditing purposes.

**Data Retention and Deletion**: Retailers can request deletion of their data at any time. The system implements a data deletion workflow that removes all retailer-specific data from RDS, clears cached data from Redis, and deletes any stored artifacts from S3. Deletion is completed within 30 days of the request.

**Synthetic Data for Training**: All machine learning models are initially trained on synthetic POS datasets that do not contain any real retailer information. This ensures no privacy risks during model development and testing. Retailer-specific models are trained only on that retailer's own data.

**Compliance**: The system is designed to comply with Indian data protection regulations. Personal information (phone numbers, names) is minimized and protected. Retailers provide explicit consent for data processing during onboarding.

## Responsible AI Implementation

The system is designed with responsible AI principles to ensure transparency, fairness, and appropriate use of AI predictions.

**Probabilistic Communication**: All forecasts and recommendations are explicitly communicated as probabilistic estimates, not guarantees. Responses include phrases like "could be approximately" and "this is an estimate, not a guarantee" in the retailer's language. Confidence scores are translated into qualitative terms (high confidence, moderate confidence, low confidence) that are easier for semi-literate users to understand.

**Explainability**: Each recommendation includes a simple explanation of the key factors that influenced it. For example, "This forecast is based on last month's sales and the upcoming festival season." Explanations avoid technical jargon and use familiar concepts. The system identifies the top 2-3 influencing factors rather than overwhelming users with complex details.

**Decision Support, Not Automation**: The system is positioned as a tool to support retailer decision-making, not to make decisions automatically. Retailers always have the final say on inventory purchases and pricing. The system provides information and recommendations, but never takes actions like placing orders without explicit retailer approval.

**Confidence Thresholds**: When prediction confidence is low (due to insufficient data or high uncertainty), the system explicitly warns the retailer. Low-confidence predictions include additional disclaimers and may suggest alternative approaches like consulting with suppliers or using personal experience.

**Bias Mitigation**: Models are trained on diverse synthetic data representing different business types, regions, and scales. Performance is monitored across different retailer segments to detect and address any systematic biases. The system avoids making assumptions based on retailer demographics.

**Feedback Loops**: Retailers can provide feedback on recommendation accuracy. This feedback is used to improve models over time and to identify cases where the system is not performing well. Negative feedback triggers review by human operators to understand failure modes.

**Human Oversight**: The system includes monitoring dashboards for operators to track model performance, error rates, and user satisfaction. Anomalous predictions or system behavior trigger alerts for human review. Regular audits ensure the system continues to operate as intended.

**Limitations Disclosure**: The system clearly communicates its limitations. For example, it cannot predict unexpected events like supply chain disruptions or sudden changes in consumer preferences. It works best for products with stable demand patterns and sufficient historical data.

## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Voice Processing Properties

**Property 1: Speech-to-text accuracy for business queries**
*For any* business-related voice query in Hindi or the supported regional language, when converted to speech and then processed through STT, the resulting text should preserve the business intent with sufficient accuracy for query understanding.
**Validates: Requirements 1.1**

**Property 2: Low-confidence STT triggers clarification**
*For any* speech input that produces STT output with confidence below the threshold, the system should request the retailer to repeat the question rather than proceeding with potentially incorrect text.
**Validates: Requirements 1.2**

**Property 3: Response language matches query language**
*For any* query in a supported language, the system's response should be generated and delivered in the same language as the input query.
**Validates: Requirements 2.1**

**Property 4: Numerical and unit formatting consistency**
*For any* response containing numerical information, the system should format numbers using appropriate rounding and use units that are familiar to the retailer's regional context.
**Validates: Requirements 2.3, 4.4**

**Property 5: Probabilistic disclaimers in predictions**
*For any* response containing predictions or forecasts, the system should include clear language communicating that the output is probabilistic and not a guarantee.
**Validates: Requirements 2.5, 9.1**

### Forecasting Properties

**Property 6: Demand forecasts generated for valid requests**
*For any* valid inventory item with historical POS data, when a retailer requests a demand forecast, the forecasting engine should produce next-day and next-week predictions.
**Validates: Requirements 3.1**

**Property 7: Minimum historical data usage**
*For any* forecasting request where at least 30 days of historical sales data is available, the forecasting engine should use at least that amount of data in generating predictions.
**Validates: Requirements 3.2**

**Property 8: Seasonal pattern incorporation**
*For any* product with identifiable seasonal patterns in historical data, the demand forecast should reflect adjustments based on those seasonal trends.
**Validates: Requirements 3.4**

**Property 9: Range-based predictions**
*For any* forecast or pricing recommendation, the system should express the output as a range (minimum and maximum values) rather than a single exact number.
**Validates: Requirements 3.5, 7.4**

### Inventory Management Properties

**Property 10: Reorder recommendations generated**
*For any* inventory item with current stock level and demand forecast, when a retailer requests reorder guidance, the system should calculate and provide a reorder quantity recommendation.
**Validates: Requirements 4.1**

**Property 11: Reorder calculation incorporates key factors**
*For any* reorder recommendation, the calculated quantity should appropriately change when forecasted demand, current stock level, or lead time changes, following the relationship: higher demand or longer lead time increases reorder quantity, while higher current stock decreases it.
**Validates: Requirements 4.2**

**Property 12: Storage capacity constraints respected**
*For any* reorder recommendation where the retailer has provided storage capacity constraints, the recommended quantity should not exceed the available storage capacity.
**Validates: Requirements 4.3**

**Property 13: Low-stock alerts generated**
*For any* inventory item where current quantity falls below the threshold calculated from forecasted demand and lead time, the system should generate a low-stock alert.
**Validates: Requirements 5.1**

**Property 14: Overstock alerts generated**
*For any* inventory item where current quantity exceeds optimal levels based on forecasted demand and shelf life considerations, the system should generate an overstock alert.
**Validates: Requirements 5.2**

**Property 15: Alert prioritization by urgency**
*For any* set of inventory alerts, the system should order them by urgency level, with critical alerts appearing before high, medium, and low priority alerts.
**Validates: Requirements 5.3, 5.4**

**Property 16: Alert queries return correct results**
*For any* query for alerts on a specific item or category, the system should return only alerts that match the specified item or belong to the specified category.
**Validates: Requirements 5.5**

### Seasonal and Pricing Properties

**Property 17: Seasonal pattern detection**
*For any* sales data containing recurring patterns with consistent timing and magnitude, the system should identify and describe those patterns as seasonal trends.
**Validates: Requirements 6.1**

**Property 18: Festival and agricultural cycle detection**
*For any* sales data with spikes corresponding to known festivals, holidays, or agricultural cycles in the retailer's region, the system should detect and attribute those patterns to the relevant events.
**Validates: Requirements 6.2**

**Property 19: Seasonal recommendations generated**
*For any* upcoming seasonal event identified in the system, the system should provide actionable inventory recommendations related to that event.
**Validates: Requirements 6.4**

**Property 20: Pricing recommendations generated**
*For any* inventory item, when a retailer requests pricing guidance, the system should provide pricing intelligence based on available public market data.
**Validates: Requirements 7.1**

**Property 21: Market data sources utilized**
*For any* pricing recommendation, the system should query and incorporate data from government mandi prices or open retail datasets when generating the recommendation.
**Validates: Requirements 7.2**

**Property 22: Retailer cost and margin incorporated**
*For any* pricing recommendation where the retailer has provided their cost and desired margin, the suggested price range should ensure the margin requirement is met at the recommended prices.
**Validates: Requirements 7.3**

### Transparency and Explainability Properties

**Property 23: Influencing factors explained**
*For any* recommendation or prediction, the system's response should include a list or description of the key factors that influenced the output.
**Validates: Requirements 9.2**

**Property 24: Low-confidence warnings**
*For any* prediction with confidence below a defined threshold, the system should explicitly inform the retailer that the prediction has lower confidence.
**Validates: Requirements 9.4**

### Multilingual Properties

**Property 25: Context preservation across language switches**
*For any* conversation where the retailer switches from one supported language to another, the system should maintain the conversation context and continue providing relevant responses.
**Validates: Requirements 10.2**

### API Properties

**Property 26: Authentication enforcement**
*For any* API request without valid authentication credentials, the system should reject the request and return an authentication error.
**Validates: Requirements 12.2**

**Property 27: Descriptive error messages**
*For any* failed API request, the system should return an error response that includes a descriptive message explaining the reason for failure.
**Validates: Requirements 12.4**

**Property 28: Rate limiting enforcement**
*For any* client making API requests, when the number of requests exceeds the defined rate limit within the time window, subsequent requests should be rejected with a rate limit error.
**Validates: Requirements 12.5**

## Limitations and Assumptions

**Data Availability**: The system assumes retailers can provide or will accumulate at least 30 days of POS transaction data for accurate forecasting. For new products or new retailers, predictions will have lower confidence until sufficient history is available.

**Internet Connectivity**: The system requires internet connectivity for voice processing and analytics. Retailers in areas with poor connectivity may experience degraded performance. A future enhancement could include offline mode with periodic synchronization.

**Voice Recognition Accuracy**: While AWS Transcribe supports Hindi, accuracy may vary based on accent, dialect, and audio quality. Background noise in busy retail environments may affect transcription quality. The system includes confidence thresholds and clarification requests to mitigate this.

**Language Support**: The initial version supports Hindi and one additional regional language. Expanding to all Indian languages will require additional development and testing. The architecture is designed to make language addition straightforward.

**Prediction Accuracy**: Demand forecasts are probabilistic estimates based on historical patterns. Unexpected events (supply chain disruptions, sudden trend changes, local events) cannot be predicted. The system is designed as decision support, not a replacement for retailer judgment.

**Market Data Coverage**: Pricing intelligence depends on the availability and quality of public market data. Not all products may have corresponding mandi prices. The system handles missing data gracefully by informing the retailer.

**Synthetic Training Data**: Initial models are trained on synthetic data that simulates realistic patterns but may not capture all real-world complexities. Model performance will improve as real retailer data is incorporated.

**Literacy Assumptions**: While designed for semi-literate users, the system assumes basic familiarity with mobile phones or IVR systems. User training and support may be needed during initial rollout.

**Business Model Assumptions**: The system is optimized for small retail businesses with relatively stable product catalogs. Businesses with highly variable inventory or specialized products may require customization.

## Future Enhancements

**Expanded Language Support**: Add support for additional Indian regional languages including Tamil, Telugu, Marathi, Bengali, Gujarati, Kannada, Malayalam, and others based on user demand.

**Offline Mode**: Develop a lightweight offline mode that caches recent forecasts and allows basic queries without internet connectivity, with synchronization when connection is restored.

**Supplier Integration**: Integrate with supplier systems to enable automated reordering based on recommendations, with retailer approval. This would streamline the procurement process.

**Visual Dashboard**: Provide an optional web dashboard for retailers who are comfortable with visual interfaces, showing trends, forecasts, and alerts in graphical format.

**Multi-Store Support**: Extend the system to support retailers with multiple store locations, providing consolidated analytics and inventory management across locations.

**Advanced Analytics**: Add features like customer segmentation, product affinity analysis (which products are often bought together), and profitability analysis.

**Weather Integration**: Incorporate weather data to improve demand forecasting for weather-sensitive products like beverages, umbrellas, and seasonal clothing.

**Credit and Financial Insights**: Provide insights on cash flow, credit management, and financial health based on sales and inventory data.

**Community Features**: Enable retailers to share best practices and insights (anonymized) with other retailers in their region, fostering a community of learning.

**Voice-Based Inventory Updates**: Allow retailers to update inventory levels through voice commands, making data entry easier and more accurate.

**Proactive Alerts**: Implement proactive voice calls or messages to alert retailers about critical inventory situations, upcoming seasonal opportunities, or significant market price changes.

**Integration with Payment Systems**: Integrate with UPI and other payment systems to automatically capture transaction data, reducing manual data entry.
