# Gitlab on small Kubernetes cluster

## Provision a Kubernetes cluster
Order and provision a cluster with ovh at this url:
https://labs.ovh.com/kubernetes-k8s

Configure kubectl to point to this cluster.
Configure your domain name to point to one of your nodes.

### Open these firewall ports

Priorité | Action | Protocole | IP | Port destination | Options
:---: | :---: | :---: | :---: | :---: | :---: |
0 | Autoriser | TCP | tous |  | established
1 | Autoriser | TCP | tous | 22 |
2 | Autoriser | TCP | tous | 80 |
3 | Autoriser | TCP | tous | 443 |
4 | Autoriser |	UDP | tous | 10000 |
5 | Autoriser |	ICMP | tous |  |
6 | Autoriser | TCP | tous | 30260 |
7 |	Autoriser | TCP |	tous | 31148 |
19 | Refuser | IPv4 | tous |  |

## Install Helm

Create the rbac rules for tiller:
```sh
kubectl apply -f tiller.yaml
```

Install tiller:
```sh
helm init --tiller-namespace kube-system --service-account tiller
```

## Install gitlab-helm chart

```sh
helm upgrade --install gitlab-small-team .
	--set gitlab.global.hosts.domain=<your-domain>
	--set gitlab.global.hosts.externalIP=<ip-address-your-domain-resolves-to>
	--set gitlab.certmanager-issuer.email=<email-address-for-certificates>
	--set gitlab.gitlab-runner.runners.cache.s3AccessKey=<some-access-key>
	--set gitlab.gitlab-runner.runners.cache.s3SecretKey=<some-secret-key>
	--set grafana.adminPassword=<initial-password-for-monitoring>
	--set grafana.ingress.hosts=monitoring.<your-domain>
	--set grafana.ingress.tls.hosts=monitoring.<your-domain>

```

# Usage

You can access your gitlab instance at the url `https://gitlab.<your-domain>`.

You can monitor your cluster at the url `https://monitoring.gitlab.<your-domain>`

You can create a docker images CI pipeline and Helm charts as shown in the `gitlab-ci.yaml` example file below:
```yaml
image: docker:latest

variables:
  RAILS_ENV: test
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: postgres
  POSTGRES_DB: app_test
  CONTAINER_IMAGE: $CI_REGISTRY_IMAGE:$CI_PIPELINE_ID

stages:
- build and test
- test
- review app
- staging
- production

# See https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#making-docker-in-docker-builds-faster-with-docker-layer-caching
build and test:
  stage: build and test
  image: docker:stable
  variables:
    DOCKER_HOST: 'tcp://localhost:2375'
    DOCKER_DRIVER: overlay2
  services:
  - name: docker:stable-dind
  - name: postgres:10-alpine
    alias: postgres
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker pull $CI_REGISTRY_IMAGE:latest || true
    - docker build --cache-from $CI_REGISTRY_IMAGE:latest -t $CONTAINER_IMAGE .
    - docker run --rm --network host 
        -v $CI_PROJECT_DIR/tmp/test-results:/app/tmp/test-results
        -v $CI_PROJECT_DIR/public/coverage:/app/public/coverage
        -v $CI_PROJECT_DIR/vendor/bundle:/app/vendor/bundle
        --name $CI_PROJECT_NAME $CONTAINER_IMAGE
        /bin/sh -c '
          RAILS_ENV=test bundle install --path vendor/bundle && 
          RAILS_ENV=test bundle exec rails db:create && 
          RAILS_ENV=test bundle exec rails db:migrate &&
          RAILS_ENV=test bundle exec rspec --format RspecJunitFormatter --out ./tmp/test-results/rspec.xml'
    - docker tag $CI_REGISTRY_IMAGE:latest $CONTAINER_IMAGE
    - docker push $CONTAINER_IMAGE
  artifacts:
    reports:
      junit: ./tmp/test-results/rspec.xml
    paths:
      - ./public/coverage
      - ./tmp/test-results
    name: artifact:test
    when: on_success
    expire_in: 1 weeks
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - vendor/bundle

.deploy:
  stage: review app
  image: dtzar/helm-kubectl:2.9.1
  script:
    - kubectl --kubeconfig=$KUBECONFIG create secret docker-registry read-registry-token
        --docker-username="$CI_DEPLOY_USER"
        --docker-password="$CI_DEPLOY_PASSWORD"
        --docker-email="$GITLAB_USER_EMAIL"
        --docker-server="$CI_REGISTRY"
        --namespace="$KUBE_NAMESPACE"
        --dry-run -o yaml | kubectl apply -f -
    - helm upgrade --install --wait --values=./<your-chart-folder>/values.yaml 
        --set image.url="$CI_REGISTRY_IMAGE"
        --set image.tag="$CI_PIPELINE_ID"
        --set urlPrefix="$URL_PREFIX"
        --namespace "$KUBE_NAMESPACE"
        "$CI_ENVIRONMENT_SLUG-$CI_PROJECT_NAME" ./<your-chart-folder>

.stop-deploy:
  stage: review app
  image: dtzar/helm-kubectl:2.9.1
  script:
    - helm delete --purge $CI_ENVIRONMENT_SLUG-$CI_PROJECT_NAME
    - kubectl --kubeconfig=$KUBECONFIG delete secret read-registry-token || true

review:
  extends: .deploy
  variables:
    URL_PREFIX: "staging-pr-$CI_COMMIT_REF_NAME.$CI_PROJECT_NAME"
  environment:
    name: review/$CI_COMMIT_REF_NAME
    url: "https://staging-pr-$CI_COMMIT_REF_NAME.$CI_PROJECT_NAME.<your-domain>"
    on_stop: stop review
  except:
    - master
  only:
    kubernetes: active

stop review:
  extends: .stop-deploy
  variables:
    GIT_STRATEGY: none
  when: manual
  environment:
    name: review/$CI_COMMIT_REF_NAME
    action: stop
  except:
    - master
  only:
    kubernetes: active

deploy to staging:
  extends: .deploy
  stage: staging
  variables:
    URL_PREFIX: staging.$CI_PROJECT_NAME
  environment:
    name: staging
    url: https://staging.$CI_PROJECT_NAME.<your-domain>
  only:
    refs:
      - master
    kubernetes: active

deploy to production:
  extends: .deploy
  when: manual
  stage: production
  variables:
    URL_PREFIX: $CI_PROJECT_NAME
  environment:
    name: production
    url: https://$CI_PROJECT_NAME.<your-domain>
  only:
    refs:
      - master
    kubernetes: active

```
