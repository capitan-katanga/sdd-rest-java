---
name: azure-container-apps-deployment
description: "Deploy Spring Boot 4 services to Azure Container Apps with ACR, Bicep IaC, and OIDC-authenticated GitHub Actions. Read before provisioning Azure infra or wiring CI/CD to azd / Bicep. Triggers: Azure Container Apps, ACA, ACR, Azure Container Registry, bicep, azd, az containerapp, OIDC GitHub Actions Azure, azure/login@v2, microsoft.app/containerapps, managed identity, ingress.external, dapr, KEDA scale rules."
version: 0.1.0
license: Apache-2.0
---

# Azure Container Apps Deployment

**Signals**: `*.bicep`, `azd` CLI references, `microsoft.app/containerApps`, GitHub Actions step `azure/login@v2` with `client-id`/`tenant-id`/`subscription-id`, ACR push targets, KEDA scale rules.

## Tested With

- Spring Boot 4.x containerized as JVM or native image
- Azure Container Apps API 2024-03-01+
- Bicep 0.27+
- GitHub Actions OIDC federated credentials (no static secrets)

## Do NOT Use This Skill When

- Building Dockerfiles or multi-stage images → use `containerization-docker`
- Building GraalVM native images → use `graalvm-native-image`
- Deploying to AWS (ECS, Fargate, EKS, App Runner, Lambda) → use the relevant `aws-sdk-*` / `aws-lambda-*` skills
- Configuring application logging → use `observability-logging`
- Wiring app secrets/config inside the JVM → use `core-setup`

## When to Read References

| Situation | Read |
|-----------|------|
| Provisioning Container Apps environment, ACR, managed identity with Bicep | `references/azure-deployment.md` |
| OIDC trust between GitHub Actions and Azure (no client secrets) | `references/azure-deployment.md` |
| `azd up` / `azd deploy` workflows, environment promotion | `references/azure-deployment.md` |
| Ingress (external/internal), custom domains, HTTPS, mTLS, Dapr sidecar | `references/azure-deployment.md` |
| KEDA scale rules (HTTP, queue length, CPU), min/max replicas | `references/azure-deployment.md` |
| Health probes (liveness/readiness/startup) wired to Spring Actuator | `references/azure-deployment.md` |

## Anti-patterns

- Don't use storage-account connection strings in plaintext — bind via managed identity or Key Vault references.
- Don't push images with the `latest` tag in production — use immutable SHA-pinned tags.
- Don't put `AZURE_CLIENT_SECRET` in GitHub Actions — use OIDC federated credentials.
- Don't expose Actuator endpoints publicly — keep ingress on `/actuator/*` internal-only.
