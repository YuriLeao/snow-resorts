# SonarCloud setup

Each polyrepo repo analyzes on GitHub Actions with **SonarCloud** (`https://sonarcloud.io`, org `yurileao`).

## Secret

Add repository secret **`SONAR_TOKEN`** (or an org-level secret available to all snow-resorts repos):

1. SonarCloud → account → [Security](https://sonarcloud.io/account/security) → generate token  
2. GitHub repo → Settings → Secrets and variables → Actions → `SONAR_TOKEN`

Without the token, the Sonar step fails the job.

## Projects

Create (or import) one SonarCloud project per GitHub repo. Keys used by CI:

| Repo | `sonar.projectKey` | How it scans |
|------|--------------------|--------------|
| `snow-resorts-shared` | `yurileao_snow-resorts-shared` | Maven `sonar-maven-plugin` (`-Dsonar.*` in workflow) |
| `snow-resorts-*-service` | `yurileao_snow-resorts-<name>-service` | Same Maven plugin in `ci-cd.yml` |
| `snow-resorts-mobile` | `yurileao_snow-resorts-mobile` | `sonar-project.properties` + `sonarqube-scan-action` |
| `snow-resorts-infra` | `yurileao_snow-resorts-infra` | `sonar-project.properties` + `sonarqube-scan-action` |

Java services put Sonar issue suppressions in **`pom.xml`** (`sonar.issue.ignore.multicriteria`), not in `sonar-project.properties`.

## Local (optional)

```bash
# Java (from a service or shared root), after ./mvnw verify:
export SONAR_TOKEN=…
./mvnw -B org.sonarsource.scanner.maven:sonar-maven-plugin:5.1.0.4751:sonar \
  -Dsonar.projectKey=yurileao_snow-resorts-auth-service \
  -Dsonar.organization=yurileao \
  -Dsonar.host.url=https://sonarcloud.io

# Mobile: npm run test:ci then sonar-scanner with sonar-project.properties
```

## Related CI

- Gitleaks / Trivy (Java + shared), npm audit (mobile), tfsec (infra) run alongside Sonar in the same workflows.
