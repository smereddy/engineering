# Solution Ideas

## Option A: GitHub Actions CD Pipeline
- Trigger on merge to main
- Build Docker image, push to ECR
- Deploy via ECS task update
- Estimated: 2-3 days

## Option B: Simple webhook + deploy script
- GitHub webhook on push to main
- Lightweight server that runs deploy.sh
- Estimated: 1 day

## Preference
Leaning toward Option A for reliability, but Option B is simpler to start.
