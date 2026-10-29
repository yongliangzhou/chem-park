# 化工园区信息化平台 · CI/CD GitLab Pipeline 示例

> **版本**：V1.0 | **日期**：2026-08-22
> **配套文档**：《化工园区信息化平台-微服务架构设计方案.md》《化工园区信息化平台-Kubernetes-Helm部署清单示例.md》
> **目标**：14 个微服务 + 中间件从代码提交到 K8s 私有化集群的自动化构建、扫描、部署、验证。

---

## 目录

1. [流水线总体设计](#一流水线总体设计)
2. [仓库与分支策略](#二仓库与分支策略)
3. [全局 .gitlab-ci.yml](#三全局-gitlab-ciyml)
4. [构建阶段（Maven + 镜像）](#四构建阶段maven--镜像)
5. [安全扫描阶段](#五安全扫描阶段)
6. [部署阶段（Helm 到 K8s）](#六部署阶段helm-到-k8s)
7. [验证与回滚](#七验证与回滚)
8. [关键变量与 Secrets](#八关键变量与-secrets)

---

## 一、流水线总体设计

```
代码提交 ──▶ [build] ──▶ [test] ──▶ [scan] ──▶ [package] ──▶ [deploy:dev]
                                                                      │
                                                          [deploy:stag] ─▶ [e2e] ─▶ [deploy:prod(手动+灰度)]
                                                                                      │
                                                                              [smoke] ─▶ [rollback(失败)]
```

| 阶段 | 动作 | 耗时目标 |
|------|------|----------|
| build | Maven 编译 + 单元测试 | ≤ 5 min |
| test | 集成测试 + 契约测试 | ≤ 10 min |
| scan | Trivy 镜像 + SonarQube + 依赖审计 | ≤ 8 min |
| package | 构建镜像推私有仓库 | ≤ 5 min |
| deploy | Helm upgrade（按环境） | ≤ 5 min |
| verify | 健康检查 + 冒烟 | ≤ 3 min |

---

## 二、仓库与分支策略

- **单仓多模块（monorepo）**：`platform/` 下 `services/*`、`middleware/`、`charts/`
- **分支**：`main`（生产）、`release/*`（预发）、`develop`（集成）、`feature/*`
- **环境映射**：`develop→dev`、`release/*→stag`、`main→prod（手动）`
- **标签**：`v1.x.x` 触发生产发布

---

## 三、全局 .gitlab-ci.yml

```yaml
stages:
  - build
  - test
  - scan
  - package
  - deploy
  - verify

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2 -B -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN"
  REGISTRY: "registry.chem-park.local"
  CHART_DIR: "charts/chem-park-platform"
  NAMESPACE: "chem-park-app"

include:
  - local: '/ci/build.yml'
  - local: '/ci/scan.yml'
  - local: '/ci/deploy.yml'

cache:
  paths:
    - .m2/repository
    - node_modules
```

---

## 四、构建阶段（Maven + 镜像）

```yaml
# ci/build.yml
.build:
  stage: build
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn -q clean compile
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH'

unit-test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn -q test
    - mvn jacoco:report
  coverage: '/Total.*?([0-9]{1,3})%/'
  artifacts:
    reports:
      junit: '**/target/surefire-reports/TEST-*.xml'
    paths:
      - '**/target/site/jacoco/'

contract-test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn -q -Pcontract-test verify   # Spring Cloud Contract 校验接口契约
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH =~ /^release/'

docker-build:
  stage: package
  image: docker:24
  services: [docker:24-dind]
  script:
    - |
      SERVICE=$SERVICE_NAME
      TAG=${CI_COMMIT_SHORT_SHA}
      docker build -t $REGISTRY/$SERVICE:$TAG -f services/$SERVICE/Dockerfile .
      docker push $REGISTRY/$SERVICE:$TAG
      echo "IMAGE=$REGISTRY/$SERVICE:$TAG" >> build.env
  artifacts:
    reports:
      dotenv: build.env
  parallel:
    matrix:
      - SERVICE_NAME: [iam-svc, mdm-svc, iot-ingest-svc, device-svc, safety-svc,
                       monitor-svc, closedmgmt-svc, env-svc, energy-svc, corridor-svc,
                       emergency-svc, operation-svc, ioc-svc, gateway-svc]
```

> 每个微服务 `Dockerfile` 基于 `eclipse-temurin:17-jre` 最小化镜像，非 root 运行。

---

## 五、安全扫描阶段

```yaml
# ci/scan.yml
trivy-image:
  stage: scan
  image: aquasec/trivy:0.50
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE
    - trivy image --format sarif --output trivy.sarif $IMAGE
  artifacts:
    reports: { sast: trivy.sarif }
  needs: [docker-build]

sonarqube:
  stage: scan
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn sonar:sonar -Dsonar.host.url=$SONAR_URL -Dsonar.token=$SONAR_TOKEN
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"'

dependency-audit:
  stage: scan
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn org.owasp:dependency-check-maven:check   # CVE 依赖审计
  allow_failure: true
```

> `trivy` 阻断 CRITICAL/HIGH；`dependency-check` 失败不阻塞但告警（可配置为阻断）。

---

## 六、部署阶段（Helm 到 K8s）

```yaml
# ci/deploy.yml
.deploy:
  stage: deploy
  image: bitnami/kubectl:1.29
  before_script:
    - kubectl config use-context $K8S_CONTEXT
  script:
    - |
      helm dependency build $CHART_DIR
      helm upgrade --install chem-park $CHART_DIR \
        -n $NAMESPACE --create-namespace \
        -f $CHART_DIR/values.yaml -f $CHART_DIR/values-$ENV.yaml \
        --set "services.$SERVICE.tag=$TAG" \
        --wait --timeout 300s

deploy-dev:
  extends: .deploy
  variables: { ENV: dev }
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy-stag:
  extends: .deploy
  variables: { ENV: stag }
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^release/'

deploy-prod:
  extends: .deploy
  variables: { ENV: prod }
  when: manual          # 生产手动触发
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'
```

> 灰度：生产先 `--set monitor.replicas=3` 滚动，配合 Gateway 权重路由小流量验证。

---

## 七、验证与回滚

```yaml
smoke-test:
  stage: verify
  image: curlimages/curl
  script:
    - curl -fsS https://$DOMAIN/api/ioc/overview -H "Authorization: Bearer $TEST_TOKEN" | grep -q code
    - curl -fsS https://$DOMAIN/actuator/health | grep -q UP
  rules:
    - if: '$CI_COMMIT_TAG'

rollback:
  stage: verify
  when: on_failure
  image: bitnami/kubectl:1.29
  script:
    - helm rollback chem-park -n $NAMESPACE
    - kubectl rollout status deploy/$SERVICE -n $NAMESPACE
  rules:
    - if: '$CI_COMMIT_TAG'
```

> 监控联动：部署后 Prometheus 检查 5xx 率/延迟，超阈值自动 `helm rollback`。

---

## 八、关键变量与 Secrets

| 变量 | 作用 | 存储 |
|------|------|------|
| `REGISTRY` | 私有镜像仓库地址 | CI Variable |
| `K8S_CONTEXT` | 集群 kubeconfig 上下文 | CI File Variable |
| `SONAR_TOKEN` | SonarQube 令牌 | masked secret |
| `DB_PASSWORD` / `TLS_KEY` | 数据库/证书 | Kubernetes Secret（由 Vault 注入） |
| `TEST_TOKEN` | 冒烟测试 token | masked secret |

> Secrets 经 GitLab CI Masked + Vault 注入，禁止明文入库；kubeconfig 使用最小权限 SA。

---

*文档结束 · 版本 V1.0*
