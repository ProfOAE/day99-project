# Day 99 - Hello from EKS via OIDC

Production-style mini-project: code -> GitHub Actions (OIDC, no long-lived keys) -> ECR (SHA tag) -> EKS -> ALB Ingress -> ACM HTTPS -> Route 53.

Live proof (before teardown): https://app.prof-ohene-game.eu - showed `Day 99 - Hello from EKS via OIDC - Step 5 LIVE!` with padlock.

## Architecture

```
Dev (git push main)
  -> GitHub Actions: day99-deploy.yml (OIDC AssumeRoleWithWebIdentity)
  -> ECR: hello-app:<commit-SHA> (immutable tag, not latest)
  -> EKS cluster day99 (eu-central-1, VPC + 2 AZs, NAT)
  -> Deployment hello-app (replicas: 2, readinessProbe GET /)
  -> Service hello-app (ClusterIP 80)
  -> Ingress hello-app (ALB, internet-facing, target-type ip)
  -> ACM cert for app.prof-ohene-game.eu (DNS validated, ISSUED)
  -> Route 53 Alias A-record app.prof-ohene-game.eu -> ALB
  -> Browser: https://app.prof-ohene-game.eu (HTTP/2 200, padlock)
```

## Code-to-live path (proven in Step 5)

1. Edit `app/index.html` - change heading to `Step 5 LIVE!`
2. `git commit + push` to main - triggers day99-deploy
3. Workflow builds image, tags with full commit SHA, pushes to ECR
4. `kubectl set image deployment/hello-app hello-app=<ECR>:<new-SHA>`
5. `kubectl rollout status` - 2 new pods Running 1/1, old pods gone
6. Refresh https://app.prof-ohene-game.eu - new heading visible

Old SHA: 28ff78abf6b39713a8ab4aed32589a884d560159
New SHA: da6ea7399818bf5d88878c5ed10c2c57b0006bda

## OIDC lesson - the bug that failed twice (READ THIS BEFORE YOU FORGET)

**Rule: Always check the real `sub` with a debug workflow first. Never guess it from a tutorial. Tutorials are old, GitHub changes the format.**

- Old format (tutorials): `repo:ProfOAE/day99-project:ref:refs/heads/main`
- New format (what GitHub actually sends in 2025-2026): `repo:ProfOAE@28925065/day99-project@1470335531:ref:refs/heads/main`
- Why IDs? To stop impersonation. If someone reuses your name or repo name later, their user ID / repo ID is different, so they cannot reuse your trust.
- Our mistake: trust policy used old format without IDs. AWS compared trust vs token, saw `ProfOAE` then expected `/` but got `@`, said `Not authorized to perform sts:AssumeRoleWithWebIdentity`.
- Fix: debug-oidc workflow prints real `sub`, copy that format into trust with `*` for IDs: `repo:ProfOAE@*/day99-project@*:ref:refs/heads/main`
- Debug workflow file was deleted before finish - it was a debug tool, not part of final portfolio.

## 3 trade-offs

1. **SHA tag vs `latest`**: SHA is longer to type, but you can prove exactly which code is running (`kubectl get deploy -o wide` shows SHA). `latest` is easy but you never know which code is live. Chose SHA.
2. **ALB (L7) vs NLB (L4)**: ALB costs more, but does HTTPS termination + host routing + SSL redirect with ACM. NLB is cheaper/faster but cannot do HTTPS host routing. Chose ALB for HTTPS.
3. **NAT Gateway vs NAT Instance / no NAT**: NAT Gateway costs ~$0.045/hr + data, but is managed + highly available. NAT Instance is cheaper but you manage it. No NAT means private pods cannot pull images. Chose NAT Gateway for Day 99, would revisit for cost in production (see Improvements).

## Cost per hour (eu-central-1, while running)

- EKS control plane: ~$0.10/hr
- EC2 nodes (2x t3.medium, on-demand): ~$0.08/hr total (approx, depends on size)
- NAT Gateway: ~$0.045/hr + data processing
- ALB: ~$0.025/hr + LCU
- Route 53 Hosted Zone: $0.50/month (~$0.0007/hr)
- ACM certificate: $0 (free)
- ECR storage: ~$0.10/GB/month (small image, cents)
- **Total while running: ~$0.25-$0.30/hr. Teardown brings it to $0.**

## Teardown (reverse order, proven)

1. `kubectl delete -f k8s/ingress.yaml` (deletes ALB) -> deployment + service
2. Route 53: delete Alias A-record for app... + ACM validation CNAME, then delete hosted zone
3. ACM: delete certificate (after ALB gone)
4. Terraform destroy: addons (4) -> eks (35) -> vpc (19) -> ecr-oidc (4) = 62 destroyed
5. ECR: delete images first (batch-delete-image by digest, including untagged), then destroy repo - Terraform refuses to delete non-empty repo
6. Sweep: `~/end-of-lab.sh` -> ALL CLEAR

## Improvements (if doing again)

- Terraform: single root with modules + remote state (S3 + DynamoDB lock) instead of 4 separate roots
- CD: ArgoCD / Flux to auto-sync EKS from ECR SHA, instead of manual `kubectl set image`
- Cost: Spot nodes, remove NAT if pods use VPC endpoints, use CloudFront + ACM in us-east-1 for global
- Security: least-privilege IAM (ECR push only, not `*`), OIDC `sub` pinned to exact IDs, not `*`, Pod Security Standards, NetworkPolicy
- Observability: ALB access logs to S3, Container Insights, alert on rollout failure

## Screenshots (in repo / portfolio)

- `day99-workflow-green.png` - GitHub 2 green checks
- `day99-https-padlock.png` - Browser https://app.prof-ohene-game.eu first heading
- `day99-step5-live.png` - Browser https://app.prof-ohene-game.eu Step 5 LIVE! heading
- `day99-sweep-all-clear.png` - `~/end-of-lab.sh` ALL CLEAR

## Repo layout

- `app/index.html` + `Dockerfile` - app
- `.github/workflows/day99-deploy.yml` - CI/CD (OIDC)
- `k8s/deployment.yaml` + `service.yaml` + `ingress.yaml` - K8s
- `vpc/` + `eks/` + `addons/` + `ecr-oidc/` - Terraform
- `README.md` - this file

