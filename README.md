stages:
  - setup
  - lint
  - deploy

variables:
  ORG: "turing"
  SOLMA_ID: "235909"
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
  VERSION: willBeReadFromPom
  R2D2_USER: $TURING_AUTOMATION_USERNAME
  R2D2_PASSWORD: $TURING_AUTOMATION_PASSWORD_PROD
  CHECKMARX_USER: $TURING_APP_SEC_USER_USERNAME
  CHECKMARX_PASSWORD: $TURING_APP_SEC_USER_PASSWORD
  GIT_USER: ${BOT_TURING_AUTOMATION_ID}
  GIT_PASSWORD: ${BOT_TURING_AUTOMATION}
  CAAS_NAMESPACE: cache-utility
  CI_REGISTRY: registry.sfgitlab.opr.statefarm.org
  CI_REGISTRY_IMAGE: /$ORG/k8s-development-test/pcf-to-caas
  JFROG_REGISTRY: packages.ic1.statefarm
  VERSION_C: 7.0.0-TEST
  scanner_id: $BOT_TURING_AUTOMATION_ID
  scanner_password: $BOT_TURING_AUTOMATION
  BUILD_IMAGE: registry.sfgitlab.opr.statefarm.org/registry/sfcommunity/docker/alpine:24
  NO_PROXY: .statefarm.org,.ic1.statefarm,localhost,127.0.0.1
  NAMESPACE: cache-test
  HELM_REPO_URL: https://sfgitlab.opr.statefarm.org/$ORG/repo.git
  HELM_REPO_PATH: redis-orchestrator
  HELM_RELEASE_NAME: redis-orchestrator
  HELM_CHART_PATH: ./redis-orchestrator/
  HELM_KUBE_CONTEXT: your_kube_context
  HELM_BRANCH: master

# Stage 1: Setup - Copy Helm Chart
copy_helm_chart:
  stage: setup
  image: $HELM3
  before_script:
    - helm repo add $ORG https://$ORG.sfgitlab.opr.statefarm.org/repo/
    - helm repo update
  script:
    - helm search repo $ORG
    - helm pull $ORG/$HELM_CHART_PATH --untar
    - helm template --install $HELM_RELEASE_NAME $HELM_CHART_PATH --namespace $CAAS_NAMESPACE \
        --set redisOrchestrator.image=$CI_REGISTRY/$ORG/k8s-development-test/pcf-to-caas:${VERSION_C}-SNAPSHOT \
        --set fullnameOverride=pcf-to-caas \
        --wait
    - ls
  tags:
    - cache-utility-test
  except:
    - pushes
    - merge_requests

# Stage 2: Lint Helm Chart
helm_lint:
  stage: lint
  image: registry.sfgitlab.opr.statefarm.org/registry/sfcommon/helm:3.15.4-alpine
  dependencies:
    - copy_helm_chart
  script:
    - helm lint $HELM_CHART_PATH
  tags:
    - cache-utility-test
  except:
    - pushes
    - merge_requests

# Stage 3: Deploy - Install Helm Chart
install_helm:
  stage: deploy
  image: $HELM3
  dependencies:
    - copy_helm_chart
  script:
    - echo "Installing Helm chart with image $CI_REGISTRY/$ORG/k8s-development-test/pcf-to-caas:${VERSION_C}-SNAPSHOT"
    - helm upgrade --install $HELM_RELEASE_NAME $HELM_CHART_PATH --namespace $CAAS_NAMESPACE \
        --set redisOrchestrator.image=$CI_REGISTRY/$ORG/k8s-development-test/pcf-to-caas:${VERSION_C}-SNAPSHOT \
        --set fullnameOverride=pcf-to-caas \
        --wait
    - helm list
  tags:
    - cache-utility-test
  only:
    variables:
      - $HELM_INSTALL
  except:
    - pushes
    - merge_requests

# Stage 3: Deploy - Upgrade Helm Chart
upgrade_helm:
  stage: deploy
  image: $HELM3
  dependencies:
    - copy_helm_chart
  script:
    - echo "Upgrading Helm chart with image $CI_REGISTRY/$ORG/k8s-development-test/pcf-to-caas:${VERSION_C}-SNAPSHOT"
    - helm upgrade --install $HELM_RELEASE_NAME $HELM_CHART_PATH --namespace $CAAS_NAMESPACE \
        --set redisOrchestrator.image=$CI_REGISTRY/$ORG/k8s-development-test/pcf-to-caas:${VERSION_C}-SNAPSHOT \
        --set fullnameOverride=pcf-to-caas \
        --wait
    - helm list
  tags:
    - cache-utility-test
  only:
    variables:
      - $HELM_UPGRADE
  except:
    - pushes
    - merge_requests
