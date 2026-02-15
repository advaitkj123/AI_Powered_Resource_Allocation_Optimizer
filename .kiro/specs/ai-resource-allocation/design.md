# AI Resource Allocation Optimizer - Design Document

## Overview

The AI Resource Allocation Optimizer is a serverless, event-driven system built on AWS that uses multi-agent negotiation and multi-objective optimization to allocate government resources efficiently and equitably. The system architecture follows microservices principles with clear separation between data ingestion, agent negotiation, optimization, explainability, and human approval workflows.

The design prioritizes:
- **Offline-first operation** for rural connectivity challenges
- **Serverless scalability** to handle variable load cost-effectively
- **Responsible AI** with bias detection and human oversight
- **Explainability** through vernacular language generation
- **Security and compliance** with DPDP Act requirements

## Architecture

### High-Level Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                        Client Layer                           │
├──────────────────┬──────────────────┬─────────────────────────┤
│  Mobile App      │  Web Dashboard   │  SMS Gateway            │
│  (React Native)  │  (React)         │  (Twilio/SNS)           │
│  + SQLite        │  + CloudFront    │                         │
└────────┬─────────┴────────┬─────────┴──────────┬──────────────┘
         │                  │                     │
         │                  │                     │
┌────────▼──────────────────▼─────────────────────▼──────────────┐
│                     API Gateway Layer                          │
├────────────────────────────────────────────────────────────────┤
│  AWS AppSync (GraphQL)  │  API Gateway (REST)  │  EventBridge  │
└────────┬─────────────────┴──────────┬───────────┴──────────────┘
         │                            │
         │                            │
┌────────▼────────────────────────────▼───────────────────────────┐
│                   Orchestration Layer                           │
├─────────────────────────────────────────────────────────────────┤
│              AWS Step Functions (State Machine)                 │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐       │
│  │ Ingest   │ Agents   │ Optimize │ Explain  │ Approve  │       │ 
│  └──────────┴──────────┴──────────┴──────────┴──────────┘       │
└────────┬────────────────────────────────────────────────────────┘
         │
         │
┌────────▼─────────────────────────────────────────────────────────┐
│                    Compute Layer (AWS Lambda)                    │
├──────────────────────────────────────────────────────────────────┤
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐        │
│  │ Equity   │ Budget   │ Impact   │ Urgency  │Community │        │ 
│  │ Agent    │ Agent    │ Agent    │ Agent    │ Agent    │        │
│  └──────────┴──────────┴──────────┴──────────┴──────────┘        │
│  ┌──────────────────┬──────────────────┬──────────────────┐      │
│  │ Optimization     │ Explainability   │ Bias Detection   │      │
│  │ Engine (PuLP)    │ Engine (Bedrock) │ (Clarify)        │      │
│  └──────────────────┴──────────────────┴──────────────────┘      │ 
└────────┬─────────────────────────────────────────────────────────┘
         │
         │
┌────────▼──────────────────────────────────────────────────────────┐
│                      ML/AI Layer                                  │
├───────────────────────────────────────────────────────────────────┤
│  SageMaker          │  Bedrock           │  Transcribe/Polly      │
│  (XGBoost Model)    │  (Claude 3 Haiku)  │  (Voice I/O)           │
│  + Clarify          │                    │  + Bhashini            │
└────────┬──────────────────────────────────────────────────────────┘
         │
         │
┌────────▼──────────────────────────────────────────────────────────┐
│                      Data Layer                                   │
├───────────────────────────────────────────────────────────────────┤
│  RDS PostgreSQL  │  DynamoDB      │  S3           │  OpenSearch   │
│  (Structured)    │  (Sync/Cache)  │  (Documents)  │  (Analytics)  │
└───────────────────────────────────────────────────────────────────┘
```

### Event-Driven Workflow

```
Request Submission → EventBridge → Step Function → Agent Negotiation
                                                  ↓
                                            Optimization
                                                  ↓
                                            Explainability
                                                  ↓
                                            Human Approval
                                                  ↓
                                            Notification
```

### Offline-First Sync Architecture

```
Mobile Device (SQLite) ←→ AWS AppSync ←→ DynamoDB ←→ Lambda ←→ RDS
                              ↓
                        Conflict Resolution
                        (Last-Writer-Wins)
```

## Components and Interfaces

### 1. Request Ingestion Service

**Responsibility**: Accept and validate allocation requests from multiple channels

**Components**:
- `RequestValidator`: Validates request schema and business rules
- `VoiceProcessor`: Transcribes and translates voice input
- `SMSParser`: Parses structured SMS messages
- `DocumentUploader`: Handles file attachments to S3

**Interfaces**:
```python
class RequestIngestionService:
    def submit_mobile_request(request: AllocationRequest) -> RequestID
    def submit_voice_request(audio: AudioFile, language: str) -> RequestID
    def submit_sms_request(sms: SMSMessage) -> RequestID
    def get_request_status(request_id: RequestID) -> RequestStatus
```

**AWS Services**:
- Lambda functions for processing
- S3 for audio/document storage
- Transcribe for speech-to-text
- Comprehend for language detection
- EventBridge for workflow triggering

### 2. Multi-Agent Negotiation Service

**Responsibility**: Coordinate five specialized agents to negotiate optimal allocations

**Components**:
- `EquityAgent`: Minimizes Gini coefficient
- `BudgetAgent`: Enforces budget constraints
- `ImpactAgent`: Maximizes predicted social impact
- `UrgencyAgent`: Prioritizes emergency needs
- `CommunityAgent`: Incorporates citizen preferences
- `NegotiationCoordinator`: Manages agent interactions

**Interfaces**:
```python
class Agent(ABC):
    @abstractmethod
    def propose(context: AllocationContext) -> Proposal
    
    @abstractmethod
    def evaluate(proposal: Proposal) -> Score
    
    @abstractmethod
    def counter_propose(proposals: List[Proposal]) -> Proposal

class NegotiationCoordinator:
    def run_negotiation(requests: List[AllocationRequest], 
                       budget: Budget) -> List[Proposal]
    def check_convergence(proposals: List[Proposal]) -> bool
    def resolve_conflicts(proposals: List[Proposal]) -> Proposal
```

**AWS Services**:
- Lambda functions (3GB memory) for each agent
- Step Functions for coordination
- DynamoDB for state management
- SageMaker for Impact Agent ML model

### 3. Optimization Engine

**Responsibility**: Compute Pareto-optimal allocations using multi-objective optimization

**Components**:
- `ObjectiveFunction`: Implements α×Impact + β×Equity + γ×Efficiency
- `ConstraintManager`: Enforces all constraints
- `ParetoFrontierGenerator`: Generates top 3 solutions
- `SolutionRanker`: Ranks solutions by multiple criteria

**Interfaces**:
```python
class OptimizationEngine:
    def optimize(proposals: List[Proposal], 
                constraints: List[Constraint],
                weights: Weights) -> List[Solution]
    
    def generate_pareto_frontier(solutions: List[Solution]) -> List[Solution]
    
    def rank_solutions(solutions: List[Solution]) -> List[RankedSolution]
```

**AWS Services**:
- Lambda function (3GB memory, 15min timeout)
- PuLP library for linear programming
- CloudWatch for performance monitoring

### 4. Explainability Engine

**Responsibility**: Generate human-readable explanations in vernacular languages

**Components**:
- `ExplanationGenerator`: Creates explanations using LLM
- `LanguageTranslator`: Translates to vernacular languages
- `CounterfactualAnalyzer`: Generates "what-if" scenarios
- `VisualizationGenerator`: Creates charts and graphs

**Interfaces**:
```python
class ExplainabilityEngine:
    def generate_explanation(solution: Solution, 
                           language: str) -> Explanation
    
    def generate_counterfactual(solution: Solution, 
                               change: Change) -> Explanation
    
    def generate_visualization(solution: Solution) -> Visualization
```

**AWS Services**:
- Lambda function for orchestration
- Bedrock (Claude 3 Haiku) for text generation
- Polly for text-to-speech
- Bhashini API for translation
- S3 for storing generated content

### 5. Bias Detection Service

**Responsibility**: Detect and flag biased allocations

**Components**:
- `BiasDetector`: Calculates fairness metrics
- `ProtectedAttributeAnalyzer`: Analyzes allocation by demographics
- `ThresholdChecker`: Flags allocations exceeding bias thresholds
- `AuditReportGenerator`: Generates monthly bias reports

**Interfaces**:
```python
class BiasDetectionService:
    def detect_bias(solution: Solution) -> BiasMetrics
    def check_thresholds(metrics: BiasMetrics) -> List[Alert]
    def generate_audit_report(period: DateRange) -> AuditReport
```

**AWS Services**:
- Lambda function for orchestration
- SageMaker Clarify for bias analysis
- S3 for audit reports
- SNS for alerts

### 6. Approval Workflow Service

**Responsibility**: Manage human review and approval of allocations

**Components**:
- `ApprovalRouter`: Routes solutions to appropriate officials
- `NotificationManager`: Sends notifications via email/SMS
- `FeedbackCollector`: Collects approval/rejection feedback
- `ReoptimizationTrigger`: Triggers re-optimization on rejection

**Interfaces**:
```python
class ApprovalWorkflowService:
    def submit_for_approval(solutions: List[Solution]) -> ApprovalRequest
    def record_approval(request_id: str, decision: Decision) -> None
    def request_reoptimization(request_id: str, 
                              constraints: List[Constraint]) -> None
```

**AWS Services**:
- Step Functions for workflow
- Lambda for business logic
- SNS for notifications
- SES for email
- DynamoDB for state tracking

### 7. Continuous Learning Service

**Responsibility**: Track outcomes and retrain ML models

**Components**:
- `OutcomeTracker`: Collects actual outcome data
- `ModelRetrainer`: Retrains Impact Agent model
- `PerformanceEvaluator`: Evaluates model performance
- `ModelDeployer`: Deploys improved models

**Interfaces**:
```python
class ContinuousLearningService:
    def track_outcome(allocation_id: str, outcome: Outcome) -> None
    def trigger_retraining() -> TrainingJob
    def evaluate_model(model: Model) -> Metrics
    def deploy_model(model: Model) -> Endpoint
```

**AWS Services**:
- Lambda for orchestration
- SageMaker for training and deployment
- S3 for training data
- EventBridge for scheduled retraining

### 8. Data Synchronization Service

**Responsibility**: Sync data between mobile devices and cloud

**Components**:
- `SyncCoordinator`: Manages sync operations
- `ConflictResolver`: Resolves data conflicts
- `BatchProcessor`: Batches updates for efficiency
- `QueueManager`: Manages failed sync operations

**Interfaces**:
```python
class DataSynchronizationService:
    def sync_from_device(device_id: str, 
                        changes: List[Change]) -> SyncResult
    def sync_to_device(device_id: str, 
                      since: Timestamp) -> List[Change]
    def resolve_conflict(conflict: Conflict) -> Resolution
```

**AWS Services**:
- AppSync for GraphQL API
- DynamoDB for sync data
- Lambda resolvers for business logic
- Cognito for authentication

## Data Models

### Core Domain Models

```python
@dataclass
class AllocationRequest:
    request_id: str
    panchayat_id: str
    submitted_by: str
    submission_date: datetime
    amount_requested: Decimal
    purpose: str
    category: AllocationCategory
    urgency_level: UrgencyLevel
    supporting_documents: List[str]  # S3 URLs
    status: RequestStatus
    
@dataclass
class Panchayat:
    panchayat_id: str
    name: str
    state: str
    district: str
    population: int
    rural_classification: bool
    demographics: Demographics
    historical_allocations: List[Allocation]
    utilization_rate: float
    
@dataclass
class Demographics:
    sc_population: int
    st_population: int
    obc_population: int
    general_population: int
    literacy_rate: float
    income_level: IncomeLevel
    
@dataclass
class Proposal:
    agent_name: str
    allocations: Dict[str, Decimal]  # panchayat_id -> amount
    score: float
    rationale: str
    constraints_satisfied: List[str]
    
@dataclass
class Solution:
    solution_id: str
    allocations: Dict[str, Decimal]
    objective_value: float
    impact_score: float
    equity_score: float
    efficiency_score: float
    constraints_satisfied: List[str]
    pareto_rank: int
    
@dataclass
class Explanation:
    solution_id: str
    language: str
    text: str
    audio_url: Optional[str]  # S3 URL for audio
    visualizations: List[str]  # S3 URLs
    agent_contributions: Dict[str, float]
    binding_constraints: List[str]
    
@dataclass
class BiasMetrics:
    demographic_parity_difference: float
    equal_opportunity_difference: float
    disparate_impact_ratio: float
    rural_allocation_percentage: float
    gini_coefficient: float
    flagged: bool
    
@dataclass
class Outcome:
    allocation_id: str
    actual_impact: float
    predicted_impact: float
    employment_generated: int
    infrastructure_completed: int
    services_delivered: int
    citizen_satisfaction: float
    measurement_date: datetime
```

### Database Schema (RDS PostgreSQL)

```sql
-- Panchayats table
CREATE TABLE panchayats (
    panchayat_id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    state VARCHAR(50) NOT NULL,
    district VARCHAR(100) NOT NULL,
    population INTEGER NOT NULL,
    rural_classification BOOLEAN NOT NULL,
    sc_population INTEGER,
    st_population INTEGER,
    obc_population INTEGER,
    general_population INTEGER,
    literacy_rate DECIMAL(5,2),
    income_level VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Allocation requests table
CREATE TABLE allocation_requests (
    request_id VARCHAR(50) PRIMARY KEY,
    panchayat_id VARCHAR(50) REFERENCES panchayats(panchayat_id),
    submitted_by VARCHAR(100) NOT NULL,
    submission_date TIMESTAMP NOT NULL,
    amount_requested DECIMAL(15,2) NOT NULL,
    purpose TEXT NOT NULL,
    category VARCHAR(50) NOT NULL,
    urgency_level VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Allocations table
CREATE TABLE allocations (
    allocation_id VARCHAR(50) PRIMARY KEY,
    request_id VARCHAR(50) REFERENCES allocation_requests(request_id),
    panchayat_id VARCHAR(50) REFERENCES panchayats(panchayat_id),
    amount_allocated DECIMAL(15,2) NOT NULL,
    solution_id VARCHAR(50) NOT NULL,
    approved_by VARCHAR(100) NOT NULL,
    approval_date TIMESTAMP NOT NULL,
    fiscal_year VARCHAR(10) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Outcomes table
CREATE TABLE outcomes (
    outcome_id VARCHAR(50) PRIMARY KEY,
    allocation_id VARCHAR(50) REFERENCES allocations(allocation_id),
    actual_impact DECIMAL(10,2),
    predicted_impact DECIMAL(10,2),
    employment_generated INTEGER,
    infrastructure_completed INTEGER,
    services_delivered INTEGER,
    citizen_satisfaction DECIMAL(3,2),
    measurement_date TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bias audit logs table
CREATE TABLE bias_audit_logs (
    audit_id VARCHAR(50) PRIMARY KEY,
    solution_id VARCHAR(50) NOT NULL,
    demographic_parity_difference DECIMAL(5,4),
    equal_opportunity_difference DECIMAL(5,4),
    disparate_impact_ratio DECIMAL(5,4),
    rural_allocation_percentage DECIMAL(5,2),
    gini_coefficient DECIMAL(5,4),
    flagged BOOLEAN NOT NULL,
    audit_date TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Appeals table
CREATE TABLE appeals (
    appeal_id VARCHAR(50) PRIMARY KEY,
    allocation_id VARCHAR(50) REFERENCES allocations(allocation_id),
    appellant_name VARCHAR(200) NOT NULL,
    appeal_reason TEXT NOT NULL,
    status VARCHAR(20) NOT NULL,
    submitted_date TIMESTAMP NOT NULL,
    resolved_date TIMESTAMP,
    resolution TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for performance
CREATE INDEX idx_requests_panchayat ON allocation_requests(panchayat_id);
CREATE INDEX idx_requests_status ON allocation_requests(status);
CREATE INDEX idx_allocations_panchayat ON allocations(panchayat_id);
CREATE INDEX idx_allocations_fiscal_year ON allocations(fiscal_year);
CREATE INDEX idx_outcomes_allocation ON outcomes(allocation_id);
CREATE INDEX idx_appeals_allocation ON appeals(allocation_id);
CREATE INDEX idx_appeals_status ON appeals(status);
```

### DynamoDB Tables

```python
# Mobile sync table
{
    "TableName": "mobile_sync",
    "KeySchema": [
        {"AttributeName": "device_id", "KeyType": "HASH"},
        {"AttributeName": "timestamp", "KeyType": "RANGE"}
    ],
    "AttributeDefinitions": [
        {"AttributeName": "device_id", "AttributeType": "S"},
        {"AttributeName": "timestamp", "AttributeType": "N"}
    ],
    "ProvisionedThroughput": {
        "ReadCapacityUnits": 0,  # On-demand
        "WriteCapacityUnits": 0
    },
    "StreamSpecification": {
        "StreamEnabled": True,
        "StreamViewType": "NEW_AND_OLD_IMAGES"
    },
    "TimeToLiveSpecification": {
        "Enabled": True,
        "AttributeName": "ttl"
    }
}

# Audit logs table
{
    "TableName": "audit_logs",
    "KeySchema": [
        {"AttributeName": "entity_id", "KeyType": "HASH"},
        {"AttributeName": "timestamp", "KeyType": "RANGE"}
    ],
    "AttributeDefinitions": [
        {"AttributeName": "entity_id", "AttributeType": "S"},
        {"AttributeName": "timestamp", "AttributeType": "N"},
        {"AttributeName": "action_type", "AttributeType": "S"}
    ],
    "GlobalSecondaryIndexes": [
        {
            "IndexName": "action_type_index",
            "KeySchema": [
                {"AttributeName": "action_type", "KeyType": "HASH"},
                {"AttributeName": "timestamp", "KeyType": "RANGE"}
            ],
            "Projection": {"ProjectionType": "ALL"}
        }
    ]
}
```

## Error Handling

### Error Categories

1. **Validation Errors**: Invalid input data
   - Return 400 Bad Request with detailed error message
   - Log to CloudWatch for monitoring

2. **Business Logic Errors**: Constraint violations, insufficient budget
   - Return 422 Unprocessable Entity with explanation
   - Trigger notification to administrators

3. **External Service Errors**: AWS service failures, API timeouts
   - Implement exponential backoff retry (3 attempts)
   - Fall back to degraded functionality if possible
   - Alert on-call engineer via SNS

4. **Optimization Errors**: No feasible solution, timeout
   - Relax constraints incrementally
   - Request human intervention if still infeasible
   - Log detailed diagnostics

5. **ML Model Errors**: Prediction failures, model unavailable
   - Fall back to rule-based scoring
   - Alert ML team for investigation
   - Continue with degraded predictions

### Retry Strategy

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    retry=retry_if_exception_type((ConnectionError, TimeoutError))
)
def call_external_service(request):
    # Service call logic
    pass
```

### Circuit Breaker Pattern

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpenError()
        
        try:
            result = func(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            raise e
```

### Graceful Degradation

| Component | Failure Mode | Degraded Functionality |
|-----------|--------------|------------------------|
| Voice Input | Transcribe unavailable | Fall back to text input |
| Translation | Bhashini API down | Use English explanations |
| Impact Model | SageMaker endpoint down | Use rule-based scoring |
| Bias Detection | Clarify unavailable | Use hard-coded constraints only |
| Optimization | Timeout after 15min | Return best solution found so far |
| Notifications | SNS failure | Queue for retry, log to CloudWatch |

## Testing Strategy

### Unit Testing

**Scope**: Individual functions and classes

**Framework**: pytest

**Coverage Target**: >80%

**Key Areas**:
- Agent proposal generation logic
- Constraint validation
- Gini coefficient calculation
- Conflict resolution algorithms
- Data validation and sanitization
- Error handling paths

**Example**:
```python
def test_equity_agent_proposal():
    agent = EquityAgent()
    context = AllocationContext(
        requests=[...],
        budget=1000000,
        panchayats=[...]
    )
    proposal = agent.propose(context)
    
    assert proposal.score >= 0 and proposal.score <= 100
    assert sum(proposal.allocations.values()) <= context.budget
    assert all(amount >= 0 for amount in proposal.allocations.values())
```

### Integration Testing

**Scope**: Component interactions

**Framework**: pytest + moto (AWS mocking)

**Key Areas**:
- End-to-end request submission to allocation
- Agent negotiation coordination
- Database read/write operations
- S3 file upload/download
- AppSync sync operations
- Step Functions workflow execution

**Example**:
```python
@mock_aws
def test_request_submission_workflow():
    # Setup mocked AWS services
    s3 = boto3.client('s3')
    dynamodb = boto3.resource('dynamodb')
    
    # Submit request
    request = AllocationRequest(...)
    request_id = submit_request(request)
    
    # Verify stored in database
    item = dynamodb.Table('requests').get_item(Key={'request_id': request_id})
    assert item['Item']['status'] == 'PENDING'
    
    # Verify workflow triggered
    # ... assertions
```

### Property-Based Testing

**Framework**: Hypothesis

**Purpose**: Verify universal properties across all inputs

**Key Properties**:
- Budget constraint satisfaction
- Non-negativity of allocations
- Rural allocation minimum (25%)
- Gini coefficient bounds (0-1)
- Pareto optimality of solutions

**Example**:
```python
from hypothesis import given, strategies as st

@given(
    requests=st.lists(st.builds(AllocationRequest), min_size=1, max_size=100),
    budget=st.decimals(min_value=1000000, max_value=100000000)
)
def test_budget_constraint_always_satisfied(requests, budget):
    """Property: Total allocation must never exceed budget"""
    solution = optimize(requests, budget)
    total_allocated = sum(solution.allocations.values())
    assert total_allocated <= budget
```

### End-to-End Testing

**Scope**: Complete user workflows

**Framework**: Selenium + pytest

**Key Scenarios**:
- Panchayat leader submits request via mobile app
- Government official reviews and approves allocation
- System generates explanation in Hindi
- Appeal is submitted and resolved
- Bias is detected and allocation flagged

### Performance Testing

**Tool**: Locust

**Scenarios**:
- 1000 concurrent mobile app users
- 100 simultaneous allocation computations
- 10,000 requests/minute API load
- Large dataset optimization (1000+ panchayats)

**Metrics**:
- Response time (p50, p95, p99)
- Throughput (requests/second)
- Error rate
- Resource utilization (Lambda memory, RDS connections)

### Security Testing

**Activities**:
- Penetration testing (bi-annually)
- Vulnerability scanning (weekly)
- Dependency auditing (automated)
- IAM policy validation
- Encryption verification
- DPDP Act compliance audit

### Chaos Engineering

**Tool**: AWS Fault Injection Simulator

**Experiments**:
- Lambda function failures
- RDS failover
- DynamoDB throttling
- Network latency injection
- AZ failure simulation

**Success Criteria**:
- System remains available (degraded OK)
- No data loss
- Automatic recovery within SLA


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system-essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Budget Constraint Satisfaction

*For any* set of allocation requests and available budget, the sum of all allocations in any generated solution must be less than or equal to the available budget.

**Validates: Requirements 3.2.2, 3.3.2**

### Property 2: Non-Negativity of Allocations

*For any* generated solution, every individual allocation amount must be greater than or equal to zero.

**Validates: Requirements 3.3.2**

### Property 3: Rural Allocation Minimum

*For any* generated solution, the percentage of total allocation going to rural initiatives must be greater than or equal to 25%.

**Validates: Requirements 3.2.1, 3.3.2, 3.7.3**

### Property 4: Gini Coefficient Bounds

*For any* allocation proposal, the calculated Gini coefficient must be a value between 0 and 1 inclusive.

**Validates: Requirements 3.2.1**

### Property 5: Pareto Optimality

*For any* solution in the Pareto frontier, there should not exist another solution that improves one objective without worsening at least one other objective.

**Validates: Requirements 3.3.3**

### Property 6: Explanation Completeness

*For any* generated solution, the explanation must include contribution information from all five agents (Equity, Budget, Impact, Urgency, Community).

**Validates: Requirements 3.4.1**

### Property 7: Conflict Detection

*For any* two modifications to the same data entity with different timestamps, the synchronization system must detect a conflict.

**Validates: Requirements 3.9.1**

### Property 8: Last-Writer-Wins Resolution

*For any* detected conflict with two conflicting writes, the synchronization system must resolve by keeping the write with the later timestamp.

**Validates: Requirements 3.9.1**

### Property 9: Bias Metric Calculation

*For any* generated solution, bias metrics (demographic parity difference, disparate impact ratio) must be calculated and stored.

**Validates: Requirements 3.7.1**

### Property 10: Bias Threshold Flagging

*For any* solution where demographic parity difference exceeds 0.1 or disparate impact ratio is below 0.8, the solution must be flagged for manual review.

**Validates: Requirements 3.7.1, 6.2.2**

### Property 11: Outcome Tracking Completeness

*For any* finalized allocation, both predicted impact and actual impact (once measured) must be stored in the system.

**Validates: Requirements 3.6.1**

### Property 12: Appeal Metadata Completeness

*For any* submitted appeal, the system must store timestamp, appellant identity, and appeal reason.

**Validates: Requirements 3.8.1**

### Property 13: Appeal Notification Timeliness

*For any* recorded appeal, a notification must be sent to the responsible official within 24 hours of submission.

**Validates: Requirements 3.8.2**

### Property 14: Language Fallback

*For any* explanation generation request where translation to the requested vernacular language fails, the system must provide the explanation in English.

**Validates: Requirements 3.4.3**

### Property 15: Model Drift Detection

*For any* deployed ML model, when prediction accuracy (MAPE) on recent data exceeds 15%, the system must detect this as model drift and trigger an alert.

**Validates: Requirements 3.6.3, 6.1.4**

## Deployment Strategy

### Infrastructure as Code

All infrastructure will be defined using Terraform:

```hcl
# Example Terraform configuration
resource "aws_lambda_function" "equity_agent" {
  filename      = "equity_agent.zip"
  function_name = "ai-allocation-equity-agent"
  role          = aws_iam_role.lambda_exec.arn
  handler       = "main.handler"
  runtime       = "python3.11"
  memory_size   = 3072
  timeout       = 900
  
  environment {
    variables = {
      RDS_ENDPOINT = aws_db_instance.main.endpoint
      DYNAMODB_TABLE = aws_dynamodb_table.sync.name
    }
  }
  
  vpc_config {
    subnet_ids         = aws_subnet.private[*].id
    security_group_ids = [aws_security_group.lambda.id]
  }
}
```

### CI/CD Pipeline

**GitHub Actions Workflow**:

```yaml
name: Deploy AI Allocation System

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      - name: Run unit tests
        run: pytest tests/unit --cov=src --cov-report=xml
      - name: Run property tests
        run: pytest tests/property --hypothesis-profile=ci
      - name: Upload coverage
        uses: codecov/codecov-action@v3
  
  deploy-dev:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      - name: Terraform Init
        run: terraform init
      - name: Terraform Plan
        run: terraform plan -out=tfplan
      - name: Terraform Apply
        run: terraform apply tfplan
      - name: Deploy Lambda functions
        run: |
          ./scripts/package-lambdas.sh
          ./scripts/deploy-lambdas.sh dev
```

### Blue-Green Deployment

1. Deploy new version to "green" environment
2. Run smoke tests against green environment
3. Gradually shift traffic from blue to green (10% → 50% → 100%)
4. Monitor error rates and latency
5. Rollback to blue if issues detected
6. Keep blue environment for 24 hours before decommissioning

### Rollback Strategy

**Automated Rollback Triggers**:
- Error rate > 5% for 5 minutes
- P99 latency > 10 seconds for 5 minutes
- Any critical alarm triggered

**Rollback Process**:
1. CloudWatch alarm triggers SNS notification
2. Lambda function executes rollback script
3. Traffic shifted back to previous version
4. On-call engineer notified
5. Post-mortem scheduled

### Database Migration Strategy

**Flyway for Schema Migrations**:

```sql
-- V1__initial_schema.sql
CREATE TABLE panchayats (...);
CREATE TABLE allocation_requests (...);

-- V2__add_appeals_table.sql
CREATE TABLE appeals (...);
ALTER TABLE allocations ADD COLUMN appeal_id VARCHAR(50);
```

**Migration Process**:
1. Test migration on dev environment
2. Backup production database
3. Run migration during maintenance window
4. Verify data integrity
5. Monitor application health

### Monitoring and Alerting

**CloudWatch Alarms**:

```python
# Critical alarms
alarms = [
    {
        "name": "HighErrorRate",
        "metric": "Errors",
        "threshold": 5,  # 5% error rate
        "evaluation_periods": 2,
        "action": "page_oncall"
    },
    {
        "name": "HighLatency",
        "metric": "Duration",
        "statistic": "p99",
        "threshold": 10000,  # 10 seconds
        "evaluation_periods": 2,
        "action": "page_oncall"
    },
    {
        "name": "BiasDetected",
        "metric": "BiasViolations",
        "threshold": 1,
        "evaluation_periods": 1,
        "action": "notify_admin"
    }
]
```

**Dashboards**:
- System health (error rates, latency, throughput)
- Business metrics (allocations/day, approval rate, bias metrics)
- Cost tracking (Lambda invocations, SageMaker usage, data transfer)
- ML model performance (prediction accuracy, drift metrics)

### Disaster Recovery

**RTO**: 2 hours
**RPO**: 24 hours

**DR Plan**:
1. **Data Replication**: 
   - RDS: Multi-AZ with automated backups
   - S3: Cross-region replication to secondary region
   - DynamoDB: Global tables for multi-region

2. **Failover Process**:
   - Detect primary region failure via health checks
   - Update Route53 to point to secondary region
   - Verify data consistency in secondary region
   - Resume operations

3. **Regular DR Drills**: Quarterly failover tests

### Security Hardening

**Network Security**:
- All databases in private subnets
- Lambda functions in VPC for private resource access
- Security groups with least-privilege rules
- VPC Flow Logs enabled
- WAF rules on API Gateway

**Application Security**:
- Input validation on all endpoints
- SQL injection prevention via parameterized queries
- XSS prevention via output encoding
- CSRF tokens for state-changing operations
- Rate limiting on all APIs

**Data Security**:
- Encryption at rest with KMS
- Encryption in transit with TLS 1.3
- Aadhaar data encrypted with separate key
- Automatic key rotation (annual)
- Data masking in logs

**Access Control**:
- IAM roles with least-privilege policies
- MFA required for admin access
- Service accounts for automation
- Regular access reviews (quarterly)
- Audit logging of all access

### Cost Optimization

**Strategies**:
1. **Serverless First**: Lambda and SageMaker Serverless for zero idle costs
2. **Right-Sizing**: Monitor and adjust Lambda memory, RDS instance size
3. **Reserved Capacity**: RDS Reserved Instances for 30% savings
4. **Storage Tiering**: S3 Intelligent-Tiering, Glacier for archives
5. **Caching**: CloudFront and ElastiCache to reduce compute
6. **Spot Instances**: For non-critical batch jobs
7. **Budget Alerts**: Alert at 80% and 100% of monthly budget

**Cost Monitoring**:
- Daily cost reports by service
- Cost allocation tags for tracking
- Anomaly detection for unexpected spikes
- Monthly cost optimization reviews

### Compliance and Audit

**DPDP Act Compliance**:
- Data minimization: Collect only necessary data
- Purpose limitation: Use data only for stated purpose
- Consent management: Track user consent
- Right to deletion: Automated data deletion on request
- Data portability: Export user data on request
- Breach notification: Alert within 72 hours

**Audit Trail**:
- All API calls logged to CloudTrail
- All data access logged to DynamoDB
- Immutable audit logs in S3 with Object Lock
- 7-year retention for compliance
- Quarterly audit reviews

**Certifications** (Future):
- ISO 27001 for information security
- SOC 2 Type II for service organization controls

## Operational Runbooks

### Runbook 1: High Error Rate

**Symptoms**: Error rate > 5% for 5 minutes

**Investigation**:
1. Check CloudWatch Logs for error patterns
2. Check X-Ray traces for failing components
3. Check external service status (AWS, Bhashini)
4. Check database connection pool

**Resolution**:
- If Lambda timeout: Increase timeout or optimize code
- If external service down: Enable circuit breaker, use fallback
- If database issue: Check connections, consider read replica
- If code bug: Rollback to previous version

### Runbook 2: Bias Alert

**Symptoms**: Allocation flagged for bias violation

**Investigation**:
1. Review bias metrics in CloudWatch
2. Check allocation distribution by demographics
3. Review agent proposals and weights
4. Check for data quality issues

**Resolution**:
- Notify bias review committee
- Hold allocation for manual review
- Adjust agent weights if systematic issue
- Retrain model if data drift detected

### Runbook 3: Model Performance Degradation

**Symptoms**: MAPE > 15% on recent predictions

**Investigation**:
1. Check prediction accuracy trends
2. Analyze feature distributions for drift
3. Review recent outcome data quality
4. Check for seasonal patterns

**Resolution**:
- Trigger model retraining
- Review feature engineering
- Collect more recent training data
- Consider ensemble models

### Runbook 4: Sync Failures

**Symptoms**: High rate of sync failures from mobile devices

**Investigation**:
1. Check AppSync API error logs
2. Check DynamoDB throttling metrics
3. Check network connectivity patterns
4. Review conflict resolution logs

**Resolution**:
- Increase DynamoDB capacity if throttled
- Optimize sync batch sizes
- Improve conflict resolution logic
- Add retry logic with exponential backoff

## Future Enhancements

### Phase 2 (Months 3-6)

1. **Advanced Analytics Dashboard**
   - Predictive analytics for future allocation needs
   - Geographic heatmaps of allocation distribution
   - Trend analysis and forecasting
   - Comparative benchmarking across districts

2. **Enhanced ML Models**
   - Ensemble models for improved accuracy
   - Deep learning for complex pattern recognition
   - Reinforcement learning for optimal agent strategies
   - Transfer learning from other regions

3. **Blockchain Integration**
   - Immutable audit trail on blockchain
   - Smart contracts for automated approvals
   - Transparent allocation history
   - Tamper-proof record keeping

4. **Citizen Portal**
   - Public dashboard for allocation transparency
   - Citizen feedback and ratings
   - Community discussion forums
   - Allocation impact stories

### Phase 3 (Months 7-12)

1. **Natural Language Interface**
   - Chatbot for allocation queries
   - Voice-based Q&A system
   - Natural language policy specification
   - Conversational explanation generation

2. **Advanced Optimization**
   - Multi-period optimization (quarterly, annual)
   - Stochastic optimization for uncertainty
   - Robust optimization for worst-case scenarios
   - Game-theoretic fair division

3. **Integration Ecosystem**
   - Integration with payment systems (PFMS)
   - Integration with Aadhaar for authentication
   - Integration with GeM for procurement
   - Integration with state government systems

4. **Mobile App Enhancements**
   - Offline ML inference on device
   - AR visualization of allocation impact
   - Gamification for engagement
   - Social sharing of success stories

### Long-Term Vision (Year 2+)

1. **Scale to National Level**
   - Support for all 255,000 panchayats
   - Integration with 3.5M+ NGOs
   - Multi-state coordination
   - Central government dashboard

2. **AI-Driven Policy Recommendations**
   - Automated policy impact analysis
   - Optimal policy parameter suggestions
   - Scenario planning for policy changes
   - Evidence-based policy making

3. **Climate and Sustainability**
   - Carbon footprint tracking
   - Climate resilience scoring
   - Sustainable development goal alignment
   - Environmental impact assessment

4. **International Expansion**
   - Adaptation for other developing nations
   - Multi-currency support
   - Localization for different governance structures
   - Open-source community edition

## Conclusion

The AI Resource Allocation Optimizer represents a significant advancement in government resource allocation, combining multi-agent AI, optimization algorithms, and responsible AI practices to deliver equitable, efficient, and transparent allocation decisions. The serverless AWS architecture ensures cost-effectiveness and scalability, while the offline-first design addresses the connectivity challenges of rural India.

The system's emphasis on explainability, bias detection, and human oversight ensures that AI augments rather than replaces human decision-making, maintaining accountability and trust. Through continuous learning from outcomes, the system will improve over time, driving better social impact and reducing the ₹750 Cr+ annual inefficiency in resource allocation.

With a phased rollout starting from 10 panchayats and scaling to 1000+ in Year 2, the system will demonstrate its value incrementally while allowing for refinement based on real-world feedback. The comprehensive testing strategy, including property-based testing for correctness guarantees, ensures the system meets its ambitious goals of reducing inefficiency from 20-30% to under 5% while maintaining fairness and transparency.
