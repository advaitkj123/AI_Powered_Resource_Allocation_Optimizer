# AI Resource Allocation Optimizer

## 📋 Documentation

- **[Requirements Document](.kiro/specs/ai-resource-allocation/requirements.md)** - Detailed functional requirements and acceptance criteria
- **[Design Document](.kiro/specs/ai-resource-allocation/design.md)** - Technical architecture and implementation details

## Project Overview

The AI Resource Allocation Optimizer is a multi-agent AI system designed to revolutionize resource allocation for India's ~255,000 gram panchayats and 3.5M+ NGOs. Built on AWS serverless architecture, this system addresses critical inefficiencies in manual resource allocation that currently result in an estimated ₹750 Cr+ annual loss from MGNREGA's ~₹86,000 Cr budget.

## Problem Statement

India's rural resource allocation faces several critical challenges:

- **20-30% budget inefficiency** due to manual decision-making processes
- **Rural communities receive disproportionately lower allocations** despite greater need
- **No structured transparency** leading to corruption allegations
- **No measurable impact tracking** to evaluate allocation effectiveness
- **Decision cycles take ~4 weeks** causing delays in critical resource deployment

## Solution

A serverless, multi-agent AI system that:

1. **Automates allocation decisions** through 5 specialized AI agents that negotiate optimal resource distribution
2. **Balances multiple objectives** including equity, impact, budget constraints, urgency, and community preferences
3. **Generates transparent explanations** in vernacular languages for every allocation decision
4. **Operates offline-first** enabling usage in areas with poor connectivity
5. **Continuously learns** from allocation outcomes to improve future predictions
6. **Ensures fairness** through automated bias detection and hard-coded fairness constraints

## Key Features

### Multi-Agent Negotiation System

Five specialized AI agents work together to optimize allocations:

- **Equity Agent**: Minimizes Gini coefficient to ensure fair distribution
- **Budget Agent**: Enforces financial constraints and compliance
- **Impact Agent**: Maximizes predicted social outcome improvement
- **Urgency Agent**: Prioritizes emergency and time-sensitive requests
- **Community Agent**: Incorporates citizen preferences and feedback

### Responsible AI

- Monthly bias audits using Amazon SageMaker Clarify
- Counterfactual explanations for transparency
- Human-in-the-loop approval for final decisions
- Hard-coded fairness constraints in optimization model

### Offline-First Mobile Experience

- React Native mobile app with SQLite local storage
- Voice input support for low-literacy users
- SMS fallback for feature phones
- Automatic cloud synchronization when connectivity is available

### Vernacular Language Support

- Explanations generated in local languages using Amazon Bedrock
- Voice input transcription for multiple Indian languages
- Accessible interface for diverse user base

## Architecture

### AWS Services Used

**Frontend:**
- Amazon S3 + CloudFront (Web dashboard)
- React Native + SQLite (Mobile app)

**Compute:**
- AWS Lambda (AI agents and business logic)
- AWS Step Functions (Workflow orchestration)
- Amazon EventBridge (Event-driven triggers)

**AI/ML:**
- Amazon SageMaker (XGBoost model training and inference)
- Amazon Bedrock (Claude 3 Haiku for explanations)
- SageMaker Clarify (Bias detection)

**Data:**
- Amazon RDS PostgreSQL (Primary database)
- Amazon DynamoDB (Offline sync)
- Amazon S3 (Document storage and audit logs)
- Amazon OpenSearch Service (Analytics)

**Security:**
- AWS KMS (Encryption at rest)
- TLS 1.3 (Encryption in transit)
- IAM Roles (Access control)
- DPDP Act compliant architecture

## Cost Model

For 1000 Panchayats:

| Line Item | Monthly Cost |
|-----------|--------------|
| Compute (Lambda + Fargate) | ₹5,000 |
| ML/AI (SageMaker + Bedrock) | ₹13,000 |
| Storage (RDS + S3) | ₹11,000 |
| Monitoring & Transfer | ₹6,500 |
| **Gross Operating Cost** | **₹35,500** |
| Startup Credits | -₹27,500 |
| **Net Monthly Cost** | **₹8,000** |

## Optimization Model

The system uses a multi-objective optimization approach:

**Objective Function:**
```
Maximize: α × Impact + β × Equity + γ × Efficiency
```

**Constraints:**
- Total Allocation ≤ Budget
- Rural Initiatives ≥ 25% of Budget
- All Allocations ≥ 0
- Historical utilization limits respected

**Solver:** PuLP (Python Linear Programming)

## Workflow

1. **Submission**: Panchayat leaders submit needs via mobile app (voice or text)
2. **Negotiation**: 5 AI agents analyze and negotiate optimal allocations
3. **Optimization**: Multi-objective engine computes Pareto-optimal solutions
4. **Explanation**: Bedrock generates vernacular explanations
5. **Bias Check**: SageMaker Clarify detects potential bias
6. **Approval**: Government officials review and approve top 3 solutions
7. **Learning**: System tracks outcomes and retrains ML models

## Scaling Plan

**Phase 1 (Months 1-2):** Deploy to 10 pilot panchayats
- Validate core functionality
- Gather initial user feedback
- Refine ML models

**Phase 2 (Months 3-4):** Improve ML models
- Incorporate outcome data
- Optimize agent negotiation protocol
- Enhance explanations

**Phase 3 (Months 5-6):** State-level partnership
- Scale to 100 panchayats
- Establish monitoring and support processes
- Conduct bias audits

**Year 2:** National rollout
- Scale to 1000+ panchayats
- Continuous improvement and optimization
- Expand to additional states

## Documentation

This specification consists of two main documents:

### 1. Requirements Document (`requirements.md`)

Comprehensive requirements specification including:
- 12 major functional requirements
- User stories for each stakeholder
- Detailed acceptance criteria in EARS format
- Glossary of technical terms

**Key Requirements:**
- Mobile app with voice input and offline capabilities
- Multi-agent AI negotiation system
- Pareto-optimal solution generation
- Vernacular language explanations
- Budget constraint enforcement
- Continuous ML learning
- Bias detection and prevention
- Real-time monitoring dashboard
- Security and compliance (DPDP Act)
- Appeals mechanism
- Serverless architecture
- Offline-first data synchronization

### 2. Design Document (`design.md`)

Detailed technical design including:
- High-level architecture diagram
- Component specifications and interfaces
- Data models (TypeScript definitions)
- 12 correctness properties for testing
- Error handling strategies
- Comprehensive testing strategy

**Key Design Elements:**
- Serverless architecture using AWS Lambda and Step Functions
- Multi-agent system with 5 specialized agents
- Optimization engine using PuLP
- SageMaker for ML model training and inference
- Bedrock for natural language explanations
- Offline-first mobile architecture with SQLite
- Bidirectional data sync using AppSync and DynamoDB
- Real-time analytics using OpenSearch
- End-to-end encryption and access control

## Correctness Properties

The system is designed with 12 testable correctness properties:

1. Budget constraints are never violated
2. Rural allocations always meet 25% minimum
3. All allocations are non-negative
4. Offline-online data consistency is maintained
5. All 5 agents are invoked for every allocation
6. Generated solutions are Pareto-optimal
7. Explanations are generated for all allocations
8. Sensitive data is encrypted at rest
9. All data in transit uses TLS 1.3
10. Bias detection runs for every allocation
11. Outcome data feeds back into model training
12. Appeals mechanism is always available

These properties can be validated using property-based testing frameworks (Hypothesis for Python, fast-check for JavaScript).

## Security & Compliance

- **Encryption**: AWS KMS for data at rest, TLS 1.3 for data in transit
- **Access Control**: IAM roles with least privilege principle
- **Data Protection**: DPDP Act compliant architecture
- **Audit Trails**: All actions logged to S3 for compliance
- **Privacy**: Aadhaar data encrypted with auto-purge policy
- **Bias Prevention**: Monthly audits and hard-coded fairness constraints

## Technology Stack

**Mobile:**
- React Native
- SQLite (offline storage)
- AWS Amplify (authentication)

**Web:**
- React
- Amazon S3 + CloudFront

**Backend:**
- Python (Lambda functions)
- AWS Step Functions
- AWS AppSync (GraphQL)

**AI/ML:**
- XGBoost (impact prediction)
- PuLP (optimization)
- Amazon Bedrock (explanations)
- SageMaker Clarify (bias detection)

**Data:**
- PostgreSQL (RDS)
- DynamoDB
- Amazon S3
- OpenSearch

## Getting Started

### Prerequisites

- AWS Account with appropriate permissions
- AWS CLI configured
- Python 3.9+
- Node.js 16+
- React Native development environment

### Next Steps

1. Review the `requirements.md` document to understand all functional requirements
2. Study the `design.md` document for technical architecture details
3. Set up AWS infrastructure using Infrastructure as Code (Terraform/CloudFormation)
4. Implement the 5 AI agents as Lambda functions
5. Build the optimization engine using PuLP
6. Develop the mobile app using React Native
7. Create the web dashboard using React
8. Integrate SageMaker for ML model training
9. Set up monitoring and alerting
10. Conduct thorough testing (unit, integration, property-based)

## Impact

**Expected Outcomes:**
- Reduce allocation inefficiency from 20-30% to <5%
- Save ₹750+ Cr annually in MGNREGA budget
- Reduce decision cycles from 4 weeks to 48 hours
- Increase transparency and reduce corruption allegations
- Improve rural allocation equity by 40%
- Enable data-driven policy decisions

## Contributing

This project is designed for the AWS Hackathon. For questions or contributions, please refer to the requirements and design documents for detailed specifications.

## License

[To be determined based on project deployment]

