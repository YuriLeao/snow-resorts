# Documentação do workspace

Documentação **viva** da plataforma (polyrepo workspace `snow-resorts`).

| Doc | Conteúdo |
|-----|----------|
| [../ARCHITECTURE.md](../ARCHITECTURE.md) | Arquitetura implementada + decisões de fundação do backend |
| [../LOCAL_DEV.md](../LOCAL_DEV.md) | Desenvolvimento local, App Store iOS, EAS |
| [../README.md](../README.md) | Visão dos repositórios e setup |

## SonarCloud (CI)

Cada repo polyrepo analisa no GitHub Actions com [SonarCloud](https://sonarcloud.io) (org `yurileao`).

1. SonarCloud → account → Security → gere **`SONAR_TOKEN`**
2. GitHub repo → Settings → Secrets → Actions → `SONAR_TOKEN` (ou secret na org)

| Repo | `sonar.projectKey` |
|------|--------------------|
| `snow-resorts-shared` | `yurileao_snow-resorts-shared` |
| `snow-resorts-*-service` | `yurileao_snow-resorts-<name>-service` |
| `snow-resorts-mobile` | `yurileao_snow-resorts-mobile` |
| `snow-resorts-infra` | `yurileao_snow-resorts-infra` |

Java: Maven `sonar-maven-plugin` no workflow. Mobile/infra: `sonar-project.properties`. Sem token, o step Sonar falha o job.

## O que fica fora desta pasta

- **READMEs por repositório** (`snow-resorts-*-service/README.md`, etc.) — específicos de cada Git repo.
- **[security-audit/](../security-audit/)** — relatórios pontuais de auditoria (evidência histórica).
