# Design Document

## Overview

The AI Resource Allocation Optimizer is a serverless, multi-agent system built on AWS that optimizes resource allocation for Indian Gram Panchayats/Organizations. The system employs five specialized AI agents that negotiate allocations through a structured protocol, ultimately generating Pareto-optimal solutions. The architecture prioritizes offline-first mobile capabilities, responsible AI practices, and cost efficiency through serverless computing.

## Architecture

### High-Level Architecture

```
┌─────────────────┐         ┌──────────────────┐
│  Mobile App     │◄────────┤  Web Dashboard   │
│  (React Native) │         │  (React/S3)      │
└────────┬────────┘         └────────┬─────────┘
         │                           │
         │                           │
         ▼                           ▼
┌─────────────────────────────────────────────┐
│         AWS AppSync + API Gateway           │
└────────┬────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│          AWS Step Functions                 │
│     (Orchestration Workflow)                │
└────────┬────────────────────────────────────┘
         │
         ├──────────┬──────────┬──────────┬────────┐
         ▼          ▼          ▼          ▼        ▼
    ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
    │Equity  │ │Budget  │ │Impact  │ │Urgency │ │Community│
    │Agent   │ │Agent   │ │Agent   │ │Agent   │ │Agent   │
    │(Lambda)│ │(Lambda)│ │(Lambda)│ │(Lambda)│ │(Lambda)│
    └────┬───┘ └────┬───┘ └────┬───┘ └────┬───┘ └────┬───┘
         │          │          │          │          │
         └──────────┴──────────┴──────────┴──────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │  Optimization    │
                    │  Engine (PuLP)   │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  SageMaker       │
                    │  (Impact Model)  │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  Bedrock         │
                    │  (Explanations)  │
                    └────────┬─────────┘
                             │
         ┌───────────────────┴───────────────────┐
         ▼                   ▼                   ▼
    ┌─────────┐      ┌──────────────┐    ┌──────────┐
    │   RDS   │      │  DynamoDB    │    │    S3    │
    │(Primary)│      │(Offline Sync)│    │(Storage) │
    └─────────┘      └──────────────┘    └──────────┘
```

### Component Layers

1. **Presentation Layer**: React Native mobile app, React web dashboard
2. **API Layer**: AWS AppSync (GraphQL), API Gateway (REST)
3. **Orchestration Layer**: AWS Step Functions, EventBridge
4. **Compute Layer**: AWS Lambda functions (Python-based agents)
5. **AI/ML Layer**: SageMaker (training/inference), Bedrock (explanations)
6. **Data Layer**: RDS (PostgreSQL), DynamoDB, S3
7. **Analytics Layer**: OpenSearch Service
8. **Security Layer**: KMS, IAM, TLS 1.3

## Components and Interfaces

### 1. Mobile Application (React Native)

**Responsibilities:**
- Capture community needs via forms and voice input
- Store data locally using SQLite for offline operation
- Sync data with cloud when connectivity is available
- Display allocation decisions and explanations

**Interfaces:**
- `submitNeed(needData: NeedSubmission): Promise<SubmissionResult>`
- `syncOfflineData(): Promise<SyncResult>`
- `getVoiceInput(): Promise<TranscribedText>`
- `viewAllocation(allocationId: string): Promise<AllocationDetails>`

### 2. Web Dashboard (React + S3/CloudFront)

**Responsibilities:**
- Display real-time analytics and allocation status
- Allow government officials to approve/reject allocations
- Show bias audit reports and compliance metrics
- Provide search and filtering capabilities

**Interfaces:**
- `getAnalytics(filters: AnalyticsFilter): Promise<AnalyticsData>`
- `approveAllocation(allocationId: string): Promise<ApprovalResult>`
- `viewBiasReport(period: DateRange): Promise<BiasReport>`

### 3. AI Agents (AWS Lambda)

Each agent is implemented as a separate Lambda function:

**Equity Agent:**
- Calculates Gini coefficient for proposed allocations
- Proposes adjustments to minimize inequality
- Interface: `evaluateEquity(allocations: Allocation[]): EquityScore`

**Budget Agent:**
- Validates total allocation against available budget
- Enforces minimum rural allocation (25%)
- Interface: `validateBudget(allocations: Allocation[]): BudgetValidation`

**Impact Agent:**
- Predicts social impact using SageMaker XGBoost model
- Ranks allocations by expected outcome improvement
- Interface: `predictImpact(allocation: Allocation): ImpactScore`

**Urgency Agent:**
- Scores requests based on emergency criteria
- Prioritizes time-sensitive needs
- Interface: `assessUrgency(need: Need): UrgencyScore`

**Community Agent:**
- Incorporates citizen preferences and feedback
- Weights allocations by community priority
- Interface: `evaluatePreference(allocation: Allocation): PreferenceScore`

### 4. Orchestration Workflow (Step Functions)

**Workflow Steps:**
1. Receive need submission (EventBridge trigger)
2. Validate input data
3. Invoke all 5 agents in parallel (proposal phase)
4. Run negotiation protocol (debate phase)
5. Execute optimization engine
6. Generate Pareto frontier
7. Invoke Bedrock for explanations
8. Run bias detection (SageMaker Clarify)
9. Store results and await human approval

**Interface:**
- `startAllocationWorkflow(needId: string): Promise<WorkflowExecution>`

### 5. Optimization Engine (PuLP)

**Responsibilities:**
- Solve multi-objective optimization problem
- Generate Pareto-optimal solutions
- Apply hard constraints (budget, rural minimum, non-negativity)

**Objective Function:**
```
Maximize: α × Impact + β × Equity + γ × Efficiency
```

**Constraints:**
```
∑ allocations ≤ Budget
∑ rural_allocations ≥ 0.25 × Budget
allocation_i ≥ 0 ∀i
allocation_i ≤ historical_limit_i ∀i
```

**Interface:**
- `optimize(agentScores: AgentScores[], constraints: Constraints): Solution[]`

### 6. Impact Prediction Model (SageMaker)

**Responsibilities:**
- Train XGBoost model on historical allocation outcomes
- Predict social impact scores for proposed allocations
- Continuously improve through outcome feedback

**Interface:**
- `trainModel(trainingData: OutcomeData[]): ModelArtifact`
- `predict(features: AllocationFeatures): ImpactPrediction`

### 7. Explanation Generator (Bedrock - Claude 3 Haiku)

**Responsibilities:**
- Generate human-readable explanations for allocations
- Translate explanations to vernacular languages
- Create counterfactual scenarios

**Interface:**
- `generateExplanation(allocation: Allocation, language: string): Explanation`
- `generateCounterfactual(allocation: Allocation): CounterfactualScenario`

### 8. Data Synchronization (AppSync + DynamoDB)

**Responsibilities:**
- Manage bidirectional sync between mobile and cloud
- Resolve conflicts in offline-first scenarios
- Maintain data consistency

**Interface:**
- `syncData(localChanges: Change[]): SyncResult`
- `resolveConflict(conflict: DataConflict): Resolution`

## Data Models

### Need Submission
```typescript
interface NeedSubmission {
  id: string;
  panchayatId: string;
  submittedBy: string;
  submissionDate: Date;
  needType: 'infrastructure' | 'healthcare' | 'education' | 'employment' | 'other';
  description: string;
  requestedAmount: number;
  urgencyLevel: 1 | 2 | 3 | 4 | 5;
  beneficiaryCount: number;
  location: GeoLocation;
  voiceRecordingUrl?: string;
  attachments?: string[];
  status: 'pending' | 'processing' | 'approved' | 'rejected';
}
```

### Allocation
```typescript
interface Allocation {
  id: string;
  needId: string;
  panchayatId: string;
  allocatedAmount: number;
  allocationDate: Date;
  approvedBy?: string;
  approvalDate?: Date;
  scores: {
    equity: number;
    budget: number;
    impact: number;
    urgency: number;
    community: number;
  };
  explanation: string;
  explanationLanguage: string;
  status: 'proposed' | 'approved' | 'rejected' | 'implemented';
}
```

### Agent Score
```typescript
interface AgentScore {
  agentType: 'equity' | 'budget' | 'impact' | 'urgency' | 'community';
  score: number; // 0-100
  confidence: number; // 0-1
  reasoning: string;
  proposedAdjustment?: number;
}
```

### Optimization Solution
```typescript
interface OptimizationSolution {
  id: string;
  allocations: Allocation[];
  objectiveValue: number;
  impactScore: number;
  equityScore: number;
  efficiencyScore: number;
  isParetoOptimal: boolean;
  rank: number; // 1-3 for top solutions
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Budget Constraint Preservation
*For any* set of allocations generated by the optimization engine, the sum of all allocated amounts must be less than or equal to the available budget.

**Validates: Requirements 5.1**

### Property 2: Rural Allocation Minimum
*For any* allocation solution, the sum of allocations to rural initiatives must be at least 25% of the total budget.

**Validates: Requirements 5.2**

### Property 3: Non-Negative Allocations
*For any* allocation in the system, the allocated amount must be greater than or equal to zero.

**Validates: Requirements 5.3**

### Property 4: Offline-Online Data Consistency
*For any* data submitted offline and later synced, the cloud database must contain the same data after successful synchronization.

**Validates: Requirements 1.3, 1.4, 12.3**

### Property 5: Agent Invocation Completeness
*For any* allocation workflow execution, all five agents (Equity, Budget, Impact, Urgency, Community) must be invoked and return scores.

**Validates: Requirements 2.2, 2.3, 2.4, 2.5, 2.6**

### Property 6: Pareto Optimality
*For any* solution in the top 3 presented to officials, there should not exist another feasible solution that improves one objective without worsening another.

**Validates: Requirements 3.1, 3.2**

### Property 7: Explanation Generation
*For any* approved allocation, an explanation must be generated and stored in the system.

**Validates: Requirements 4.1, 4.4**

### Property 8: Encryption at Rest
*For any* sensitive data stored in the system, the data must be encrypted using AWS KMS.

**Validates: Requirements 9.1**

### Property 9: Encryption in Transit
*For any* data transmission between system components, the connection must use TLS 1.3.

**Validates: Requirements 9.2**

### Property 10: Bias Detection Execution
*For any* allocation decision, the SageMaker Clarify bias detection must be executed and results stored.

**Validates: Requirements 7.1, 7.3**

### Property 11: Model Retraining with Outcomes
*For any* implemented allocation with tracked outcomes, the outcome data must be added to the training dataset for future model updates.

**Validates: Requirements 6.1, 6.2**

### Property 12: Appeal Mechanism Availability
*For any* allocation decision, the system must provide an appeals mechanism accessible through the mobile app.

**Validates: Requirements 10.1, 10.2**

## Error Handling

### 1. Network Failures
- **Mobile App**: Queue operations locally, retry with exponential backoff
- **Lambda Functions**: Implement retry logic with dead-letter queues (DLQ)
- **Step Functions**: Configure automatic retries with backoff

### 2. Agent Failures
- **Timeout**: Set Lambda timeout to 5 minutes, fail workflow if exceeded
- **Invalid Scores**: Validate agent output, use default conservative scores if invalid
- **Partial Failure**: If 4/5 agents succeed, log warning and proceed with available scores

### 3. Optimization Failures
- **Infeasible Solution**: Relax constraints incrementally and retry
- **Timeout**: Return best solution found within time limit
- **No Pareto Frontier**: Return single best solution with warning

### 4. Data Sync Conflicts
- **Last-Write-Wins**: For simple fields, use timestamp-based resolution
- **Manual Resolution**: For critical fields (allocation amounts), flag for human review
- **Conflict Logging**: Store all conflicts in S3 for audit

### 5. Model Prediction Failures
- **Fallback**: Use historical average impact score
- **Monitoring**: Alert if prediction failure rate exceeds 5%
- **Graceful Degradation**: Continue workflow with fallback scores

### 6. Bias Detection Failures
- **Non-Blocking**: Log failure but don't block allocation workflow
- **Alert**: Notify compliance team of detection failure
- **Manual Review**: Flag allocation for manual bias review

## Testing Strategy

### Unit Testing

**Scope:**
- Individual agent logic (equity calculation, budget validation, etc.)
- Optimization constraint validation
- Data model serialization/deserialization
- Encryption/decryption functions

**Framework:** pytest for Python components, Jest for JavaScript/TypeScript

**Key Test Cases:**
- Equity Agent correctly calculates Gini coefficient
- Budget Agent rejects allocations exceeding budget
- Impact Agent handles missing features gracefully
- Offline sync resolves conflicts correctly

### Property-Based Testing

**Framework:** Hypothesis (Python), fast-check (JavaScript)

**Configuration:** Minimum 100 iterations per property test

**Property Tests:**

1. **Budget Constraint Property** (Property 1)
   - **Feature: ai-resource-allocation, Property 1: Budget Constraint Preservation**
   - Generate random allocation sets, verify sum ≤ budget

2. **Rural Minimum Property** (Property 2)
   - **Feature: ai-resource-allocation, Property 2: Rural Allocation Minimum**
   - Generate random allocations, verify rural ≥ 25%

3. **Non-Negativity Property** (Property 3)
   - **Feature: ai-resource-allocation, Property 3: Non-Negative Allocations**
   - Generate random allocations, verify all amounts ≥ 0

4. **Sync Consistency Property** (Property 4)
   - **Feature: ai-resource-allocation, Property 4: Offline-Online Data Consistency**
   - Generate random offline changes, sync, verify cloud matches

5. **Agent Completeness Property** (Property 5)
   - **Feature: ai-resource-allocation, Property 5: Agent Invocation Completeness**
   - Execute workflow with random inputs, verify all 5 agents invoked

6. **Explanation Existence Property** (Property 7)
   - **Feature: ai-resource-allocation, Property 7: Explanation Generation**
   - Generate random allocations, verify explanation exists

### Integration Testing

**Scope:**
- End-to-end workflow from submission to approval
- Mobile app to cloud synchronization
- Agent negotiation protocol
- SageMaker model inference

**Key Scenarios:**
- Submit need offline, sync when online, verify in RDS
- Execute full allocation workflow, verify Pareto solutions generated
- Approve allocation, verify explanation generated in vernacular
- Detect bias, verify flagging mechanism works

### Load Testing

**Tool:** AWS Distributed Load Testing Solution

**Scenarios:**
- 1000 concurrent need submissions
- 100 simultaneous allocation workflows
- 10,000 mobile app sync operations

**Success Criteria:**
- 95th percentile latency < 3 seconds
- Error rate < 1%
- Lambda cold start < 2 seconds

### Security Testing

**Scope:**
- Penetration testing of API endpoints
- Encryption verification (at rest and in transit)
- IAM policy validation
- DPDP Act compliance audit

**Tools:** AWS Security Hub, AWS Inspector, third-party security audit
