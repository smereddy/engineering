# INTENT: Auto-Deploy Dashboard

## Problem Statement

Deploying the dashboard application is a manual, error-prone process. Every PR merge requires a developer to SSH into the server and run a deploy script, taking 5-10 minutes per deploy. With an average of 3 deploys per day (~24 minutes of blocked developer time daily), this adds up to meaningful productivity loss. Worse, deploys are sometimes forgotten entirely, causing staging and production environments to drift out of sync.

## Proposed Solution

Implement a GitHub Actions CD pipeline (Option A from solution sketch) that automatically builds and deploys both services on merge to main. The pipeline will build Docker images, push them to ECR, and update ECS task definitions to trigger zero-downtime rolling deployments. This approach is preferred over a simple webhook-based solution because it provides better reliability, built-in logging, and native integration with the existing GitHub workflow.

## Key Requirements

1. Automatic deployment triggered on merge to the `main` branch
2. Zero-downtime deployments using ECS rolling updates
3. Rollback capability to revert to a previous deployment
4. Support for both services: Next.js frontend and Python API backend
5. Integration with existing AWS ECS infrastructure and ECR registry
6. Build and push Docker images as part of the pipeline
7. Deployment status visibility (success/failure notifications)

## Technical Considerations

**Architecture:**
- Two separate Docker builds and ECS service updates are needed (Next.js app, Python API)
- Pipeline must handle both services in a coordinated manner, since they are deployed independently
- ECS service update via `aws ecs update-service` with new task definition revisions

**Constraints:**
- Must use the existing AWS ECS setup; no infrastructure migration
- Zero-downtime is non-negotiable, so ECS rolling update or blue/green deployment strategy is required
- Rollback should be achievable by re-deploying a previous task definition revision

**Trade-offs:**
- Option A (GitHub Actions) takes 2-3 days to implement vs 1 day for a webhook approach, but provides better auditability, retry logic, and no additional infrastructure to maintain
- GitHub Actions has built-in secrets management for AWS credentials, avoiding the need to manage credentials on a webhook server

## Open Questions

1. **Deploy ordering:** Do the frontend and backend need to be deployed in a specific order, or can they roll out independently?
2. **Environment strategy:** Is the pipeline needed for staging only, production only, or both? Should staging auto-deploy on merge and production require manual approval?
3. **Notification channel:** Where should deploy success/failure notifications go (Slack, email, GitHub status)?
4. **Rollback trigger:** Should rollback be manual (re-run a previous workflow) or should there be automated health checks that trigger rollback on failure?
5. **Shared dependencies:** Do the two services share any infrastructure (database migrations, shared config) that must be coordinated during deploy?

## Source Files

- `problem.md` - Problem description and current pain points
- `constraints.md` - Technical constraints and service architecture
- `solution-sketch.md` - Solution options and initial preference
