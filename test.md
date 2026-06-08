You are a Principal Cloud Architect, Staff Software Engineer, FinOps Expert, and Product Designer.

Design an MVP called "Cloud Intelligence Copilot".

Goal:
Build a web application that helps engineering teams analyze AWS infrastructure across multiple AWS accounts and regions, identify optimization opportunities, explain findings using AI, and recommend improvements.

The solution should be realistic for a hackathon implementation.

Requirements:

Frontend:
- Modern dashboard UI
- Upload CSV/Excel option
- Future support for AWS API integration
- Chat-style assistant panel
- Findings dashboard
- Service-level recommendations page

Supported AWS Services:
- EC2
- ECS
- Lambda
- Glue
- S3
- RDS
- CloudWatch
- Cost Explorer

Inputs:
1. Resource inventory exports
2. CloudWatch metrics exports
3. Cost reports
4. Glue job statistics
5. Lambda execution statistics
6. ECS utilization metrics
7. S3 bucket metadata

System Responsibilities:

Step 1:
Parse uploaded files.

Step 2:
Normalize data into a common model.

Step 3:
Run deterministic analysis.

Examples:

EC2:
- Detect underutilized instances
- Recommend downsizing
- Estimate savings

Lambda:
- Detect memory overprovisioning
- Detect high error rates
- Detect excessive cold starts

Glue:
- Detect oversized DPU allocation
- Detect long-running jobs
- Recommend DPU changes

S3:
- Detect buckets not accessed recently
- Detect lifecycle policy opportunities
- Recommend Glacier migration

ECS:
- Detect CPU and memory overprovisioning
- Recommend task resizing

RDS:
- Detect low utilization
- Recommend instance optimization

Step 4:
Generate an optimization score per service.

Step 5:
Calculate estimated monthly and annual savings.

Step 6:
Pass only analyzed findings to the LLM.

The LLM should:
- Explain findings
- Prioritize recommendations
- Answer user questions
- Generate executive summaries

The LLM must NOT perform cost calculations.

Architecture Requirements:
- Spring Boot backend
- React frontend
- AWS deployment option
- OpenAI or Claude integration
- Modular analysis engine
- Future support for MCP tools

Provide:

1. System architecture diagram
2. Database schema
3. API design
4. UI wireframes
5. Service analysis rules
6. LLM prompts
7. Project structure
8. Hackathon MVP scope
9. Nice-to-have future enhancements
10. Demo script for judges

Focus on practical implementation rather than theoretical AI concepts.
