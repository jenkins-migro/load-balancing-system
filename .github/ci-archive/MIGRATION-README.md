# Jenkins to GitHub Actions Migration Report

## Summary

Migrated four declarative Jenkins pipelines to a pinned GitHub Actions workflow at `.github/workflows/jenkins-migration.yml` and archived the original Jenkinsfiles under `.github/ci-archive/`.

## Migrated Jenkins sources

| Original path | Archived path | GitHub Actions job(s) |
| --- | --- | --- |
| `Jenkinsfile` | `.github/ci-archive/Jenkinsfile` | `mulesoft-root` |
| `dockermaven/Jenkinsfile` | `.github/ci-archive/dockermaven/Jenkinsfile` | `docker-maven` |
| `mavendockerparallel/Jenkinsfile` | `.github/ci-archive/mavendockerparallel/Jenkinsfile` | `maven-docker-parallel-build`, `maven-docker-integration-test`, `maven-docker-sonar-scan`, `maven-docker-publish-image` |
| `mulesoftanypoint/Jenkinsfile` | `.github/ci-archive/mulesoftanypoint/Jenkinsfile` | `mulesoft-anypoint` |

## Pipeline analysis

- All source Jenkinsfiles used declarative `pipeline {}` syntax.
- No shared library declarations or `vars/` functions were present, so no shared-library expansion was required.
- Jenkins `sh` steps were converted to GitHub Actions `run` steps.
- Jenkins Docker agents using `maven:3.5.0-jdk-8` were converted to job-level `container` definitions.
- Jenkins `archiveArtifacts` and `publishTestResults` behavior was represented with `actions/upload-artifact` for generated JARs and Surefire XML files.
- Jenkins `parallel` quality-analysis stages were converted into separate dependent GitHub Actions jobs.
- The Jenkins `when { branch 'master' }` Docker publish condition was converted to `if: ${{ github.ref_name == 'master' }}`.
- Jenkins post-success and post-failure messages were converted to steps with `success()` and `failure()` conditions.

## Workflow trigger

The migrated workflow uses `workflow_dispatch` because the Jenkinsfiles did not define explicit SCM, cron, or webhook triggers and the repository does not include the Maven project files required for these commands to pass automatically on pull requests. The workflow can be extended with `push` or `pull_request` triggers when the expected application sources are present.

## Required secrets and variables

| Jenkins credential or variable | GitHub Actions equivalent | Used by |
| --- | --- | --- |
| `credentials('anypoint.credentials')` username | `secrets.ANYPOINT_USERNAME` | `mulesoft-root`, `mulesoft-anypoint` |
| `credentials('anypoint.credentials')` password | `secrets.ANYPOINT_PASSWORD` | `mulesoft-root`, `mulesoft-anypoint` |
| `credentials('sonar')` password/token | `secrets.SONAR_TOKEN` | `maven-docker-sonar-scan` |
| `IMAGE = readMavenPom().getArtifactId()` | Derived with `mvn help:evaluate -Dexpression=project.artifactId` | `maven-docker-parallel-build`, `maven-docker-publish-image` |
| `VERSION = readMavenPom().getVersion()` | Derived with `mvn help:evaluate -Dexpression=project.version` | `maven-docker-parallel-build`, `maven-docker-publish-image` |

## Action versions

All third-party workflow actions are official GitHub actions pinned to commit SHAs:

- `actions/checkout@11d5960a326750d5838078e36cf38b85af677262` (`v4`)
- `actions/setup-java@cf277c60eb25467037889841efdb72551f06f6c3` (`v4`)
- `actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02` (`v4`)

## Validation

- `actionlint` should be run against `.github/workflows/jenkins-migration.yml` after migration.
- Project-level Maven validation was not run because the repository only contains Jenkins examples and no `pom.xml` files.

## Notes and follow-up

- The source Jenkinsfiles contained redacted AnyPoint password arguments (`-Danypoint.******`). The migration maps those to `-Danypoint.password` using `secrets.ANYPOINT_PASSWORD`; confirm the exact Maven property name if this project expects a different MuleSoft plugin parameter.
- Docker publish behavior was migrated exactly from Jenkins, but no Docker registry login was defined in the Jenkins source. Add an authenticated login step if the target registry requires credentials.
