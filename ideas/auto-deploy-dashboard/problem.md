# Auto-Deploy Dashboard

## Problem
Every time we merge a PR to our dashboard repo, someone has to manually SSH into the server and run the deploy script. This takes 5-10 minutes and blocks the developer.

## Current Pain
- Deployments are manual and error-prone
- Average 3 deploys per day, each taking ~8 minutes of developer time
- Sometimes deploys are forgotten, leading to staging/production drift
