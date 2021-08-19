# Gitlab Auto DevOps(组件)
> 上一篇已经介绍了安装,接下来介绍一下每个组件以及组件如何自定义  

## 相关组件
> 官方也支持每个组件的自定义,如下  
```
image: alpine:latest
variables:
  // KUBE_INGRESS_BASE_DOMAIN is the application deployment domain and should be set as a variable at the group or project level.
  // KUBE_INGRESS_BASE_DOMAIN: domain.example.com
  POSTGRES_USER: user
  POSTGRES_PASSWORD: testing-password
  POSTGRES_ENABLED: "true"
  POSTGRES_DB: $CI_ENVIRONMENT_SLUG
  POSTGRES_VERSION: 9.6.2
  DOCKER_DRIVER: overlay2
  ROLLOUT_RESOURCE_TYPE: deployment
  DOCKER_TLS_CERTDIR: ""  # https://gitlab.com/gitlab-org/gitlab-runner/issues/4501
stages:
  - build
  - test
  - deploy  # dummy stage to follow the template guidelines
  - review
  - dast
  - staging
  - canary
  - production
  - incremental rollout 10%
  - incremental rollout 25%
  - incremental rollout 50%
  - incremental rollout 100%
  - performance
  - cleanup
include:
  - template: Jobs/Build.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Jobs/Build.gitlab-ci.yml
  - template: Jobs/Test.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Jobs/Test.gitlab-ci.yml
  - template: Jobs/Code-Quality.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Jobs/Code-Quality.gitlab-ci.yml
  - template: Jobs/Deploy.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Jobs/Deploy.gitlab-ci.yml
  - template: Jobs/DAST-Default-Branch-Deploy.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Jobs/DAST-Default-Branch-Deploy.gitlab-ci.yml
  - template: Jobs/Browser-Performance-Testing.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Jobs/Browser-Performance-Testing.gitlab-ci.yml
  - template: Security/DAST.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Security/DAST.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Security/Container-Scanning.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/License-Management.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Security/License-Management.gitlab-ci.yml
  - template: Security/SAST.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Security/SAST.gitlab-ci.yml
```

### build
```
build:
  stage: build
  image: “registry.gitlab.com/gitlab-org/cluster-integration/auto-build-image/master:stable"
  variables:
    DOCKER_TLS_CERTDIR: ""
  services:
    - docker:stable-dind
  script:
## 这里定义了应用的repo和tag
    - |
      if [[ -z "$CI_COMMIT_TAG" ]]; then
        export CI_APPLICATION_REPOSITORY=${CI_APPLICATION_REPOSITORY:-$CI_REGISTRY_IMAGE/$CI_COMMIT_REF_SLUG}
        export CI_APPLICATION_TAG=${CI_APPLICATION_TAG:-$CI_COMMIT_SHA}
      else
        export CI_APPLICATION_REPOSITORY=${CI_APPLICATION_REPOSITORY:-$CI_REGISTRY_IMAGE}
        export CI_APPLICATION_TAG=${CI_APPLICATION_TAG:-$CI_COMMIT_TAG}  
      fi
    - /build/build.sh
  only:
    - branches
    - tags

```

> 在build的ci里我们发现他依赖了registry.gitlab.com/gitlab-org/cluster-integration/auto-build-image/master:stable 这个镜像来自动build  
> 我们来看一下/build/build.sh  
```
#!/bin/bash -e
# build stage script for Auto-DevOps
if ! docker info &>/dev/null; then
  if [ -z "$DOCKER_HOST" ] && [ "$KUBERNETES_PORT" ]; then
    export DOCKER_HOST='tcp://localhost:2375'
  fi
fi
##判断仓库权限
if [[ -n "$CI_REGISTRY" && -n "$CI_REGISTRY_USER" ]]; then
  echo "Logging to GitLab Container Registry with CI credentials..."
  docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
fi
##判断dockerfile如果没有则用herokuish
if [[ -f Dockerfile ]]; then
  echo "Building Dockerfile-based application..."
else
  echo "Building Heroku-based application using gliderlabs/herokuish docker image..."
#使用镜像里的Dockerfile模板
  erb -T - /build/Dockerfile.erb > Dockerfile  
fi
##获取build相关参数
build_secret_args=''
if [[ -n "$AUTO_DEVOPS_BUILD_IMAGE_FORWARDED_CI_VARIABLES" ]]; then
  build_secret_file_path=/tmp/auto-devops-build-secrets
  "$(dirname "$0")"/export-build-secrets > "$build_secret_file_path"
  build_secret_args="--secret id=auto-devops-build-secrets,src=$build_secret_file_path"
  echo 'Activating Docker BuildKit to forward CI variables with --secret'
  export DOCKER_BUILDKIT=1
fi
##拉取镜像
# pull images for cache - this is required, otherwise --cache-from will not work
docker image pull "$CI_APPLICATION_REPOSITORY:$CI_COMMIT_BEFORE_SHA" || \
docker image pull "$CI_APPLICATION_REPOSITORY:latest" || \
true
##build镜像并且push
# shellcheck disable=SC2154 # missing variable warning for the lowercase variables
# shellcheck disable=SC2086 # double quoting for globbing warning for $build_secret_args and $AUTO_DEVOPS_BUILD_IMAGE_EXTRA_ARGS
docker build \
  --cache-from "$CI_APPLICATION_REPOSITORY:$CI_COMMIT_BEFORE_SHA" \
  --cache-from "$CI_APPLICATION_REPOSITORY:latest" \
  $build_secret_args \
  --build-arg BUILDPACK_URL="$BUILDPACK_URL" \
  --build-arg HTTP_PROXY="$HTTP_PROXY" \
  --build-arg http_proxy="$http_proxy" \
  --build-arg HTTPS_PROXY="$HTTPS_PROXY" \
  --build-arg https_proxy="$https_proxy" \
  --build-arg FTP_PROXY="$FTP_PROXY" \
  --build-arg ftp_proxy="$ftp_proxy" \
  --build-arg NO_PROXY="$NO_PROXY" \
  --build-arg no_proxy="$no_proxy" \
  $AUTO_DEVOPS_BUILD_IMAGE_EXTRA_ARGS \
  --tag "$CI_APPLICATION_REPOSITORY:$CI_APPLICATION_TAG" \
  --tag "$CI_APPLICATION_REPOSITORY:latest" .
docker push "$CI_APPLICATION_REPOSITORY:$CI_APPLICATION_TAG"
docker push "$CI_APPLICATION_REPOSITORY:latest"
```


### test 
#### 代码testing
```
test:
  services:
    - postgres:latest
  variables:
    POSTGRES_DB: test
  stage: test
  image: gliderlabs/herokuish:latest
  script:
    - |
      if [ -z ${KUBERNETES_PORT+x} ]; then
        DB_HOST=postgres
      else
        DB_HOST=localhost
      fi
    - export DATABASE_URL="postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${DB_HOST}:5432/${POSTGRES_DB}"
    - cp -R . /tmp/app
    - /bin/herokuish buildpack test
  only:
    - branches
    - tags
  except:
    variables:
      - $TEST_DISABLED
```
> 这里我们可以看到依赖的镜像是gliderlabs/herokuish,数据库依赖postgres  
> /bin/herokuish buildpack test 利用了该镜像执行test  

#### Code-Quality(代码质量)
```
code_quality:
  stage: test
  image: docker:stable
  allow_failure: true
  services:
    - docker:stable-dind
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
    CODE_QUALITY_IMAGE: "registry.gitlab.com/gitlab-org/security-products/codequality:12-5-stable"
  script:
    - |
      if ! docker info &>/dev/null; then
        if [ -z "$DOCKER_HOST" -a "$KUBERNETES_PORT" ]; then
          export DOCKER_HOST='tcp://localhost:2375'
        fi
      fi
    - docker pull --quiet "$CODE_QUALITY_IMAGE"
    - docker run
        --env SOURCE_CODE="$PWD"
        --volume "$PWD":/code
        --volume /var/run/docker.sock:/var/run/docker.sock
        "$CODE_QUALITY_IMAGE" /code
  artifacts:
    reports:
      codequality: gl-code-quality-report.json
    expire_in: 1 week
  dependencies: []
  only:
    refs:
      - branches
      - tags
  except:
    variables:
      - $CODE_QUALITY_DISABLED
```
> 使用dind方式启动了一个registry.gitlab.com/gitlab-org/security-products/codequality:12-5-stable 这个镜像 这个CI将分析代码并将生成的gl-code-quality-report.json上载为artifact,然后GitLab将检查此文件并在合并请求中显示信息 这个镜像里使用了codeclimate/codeclimate这个镜像来测试  

#### container_scanning(镜像扫描)

```
#Read more about this feature here: https://docs.gitlab.com/ee/user/application_security/container_scanning/
variables:
  CS_MAJOR_VERSION: 1
container_scanning:
  stage: test
  image:
    name: registry.gitlab.com/gitlab-org/security-products/analyzers/klar:$CS_MAJOR_VERSION
    entrypoint: []
  variables:
    # By default, use the latest clair vulnerabilities database, however, allow it to be overridden here with a specific image
    # to enable container scanning to run offline, or to provide a consistent list of vulnerabilities for integration testing purposes
    CLAIR_DB_IMAGE_TAG: "latest"
    CLAIR_DB_IMAGE: "arminc/clair-db:$CLAIR_DB_IMAGE_TAG"
    # Override the GIT_STRATEGY variable in your `.gitlab-ci.yml` file and set it to `fetch` if you want to provide a `clair-whitelist.yml`
    # file. See https://docs.gitlab.com/ee/user/application_security/container_scanning/index.html#overriding-the-container-scanning-template
    # for details
    GIT_STRATEGY: none
  allow_failure: true
  services:
    - name: $CLAIR_DB_IMAGE
      alias: clair-vulnerabilities-db
  script:
    # the kubernetes executor currently ignores the Docker image entrypoint value, so the start.sh script must
    # be explicitly executed here in order for this to work with both the kubernetes and docker executors
    # see this issue for more details https://gitlab.com/gitlab-org/gitlab-runner/issues/4125
    - /container-scanner/start.sh
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json
  dependencies: []
  only:
    refs:
      - branches
    variables:
      - $GITLAB_FEATURES =~ /\bcontainer_scanning\b/
  except:
    variables:
      - $CONTAINER_SCANNING_DISABLED
```
> 和Harbor一样通过这个clair这个镜像来扫描镜像

#### dependency_scanning(相关性扫描)
```
# Read more about this feature here: https://docs.gitlab.com/ee/user/application_security/dependency_scanning/
#
# Configure the scanning tool through the environment variables.
# List of the variables: https://gitlab.com/gitlab-org/security-products/dependency-scanning#settings
# How to set: https://docs.gitlab.com/ee/ci/yaml/#variables
variables:
  DS_ANALYZER_IMAGE_PREFIX: "registry.gitlab.com/gitlab-org/security-products/analyzers"
  DS_DEFAULT_ANALYZERS: "bundler-audit, retire.js, gemnasium, gemnasium-maven, gemnasium-python"
  DS_MAJOR_VERSION: 2
  DS_DISABLE_DIND: "false"
dependency_scanning:
  stage: test
  image: docker:stable
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
  allow_failure: true
  services:
    - docker:stable-dind
  script:
    - export DS_VERSION=${SP_VERSION:-$(echo "$CI_SERVER_VERSION" | sed 's/^\([0-9]*\)\.\([0-9]*\).*/\1-\2-stable/')}
    - |
      if ! docker info &>/dev/null; then
        if [ -z "$DOCKER_HOST" -a "$KUBERNETES_PORT" ]; then
          export DOCKER_HOST='tcp://localhost:2375'
        fi
      fi
    - | # this is required to avoid undesirable reset of Docker image ENV variables being set on build stage
      function propagate_env_vars() {
        CURRENT_ENV=$(printenv)
        for VAR_NAME; do
          echo $CURRENT_ENV | grep "${VAR_NAME}=" > /dev/null && echo "--env $VAR_NAME "
        done
      }
    - |
      docker run \
        $(propagate_env_vars \
          DS_ANALYZER_IMAGES \
          DS_ANALYZER_IMAGE_PREFIX \
          DS_ANALYZER_IMAGE_TAG \
          DS_DEFAULT_ANALYZERS \
          DS_EXCLUDED_PATHS \
          DEP_SCAN_DISABLE_REMOTE_CHECKS \
          DS_DOCKER_CLIENT_NEGOTIATION_TIMEOUT \
          DS_PULL_ANALYZER_IMAGE_TIMEOUT \
          DS_RUN_ANALYZER_TIMEOUT \
          DS_PYTHON_VERSION \
          DS_PIP_DEPENDENCY_PATH \
          PIP_INDEX_URL \
          PIP_EXTRA_INDEX_URL \
          MAVEN_CLI_OPTS \
        ) \
        —volume “$PWD:/code” \
        —volume /var/run/docker.sock:/var/run/docker.sock \
        "registry.gitlab.com/gitlab-org/security-products/dependency-scanning:$DS_VERSION" /code
  artifacts:
    reports:
      dependency_scanning: gl-dependency-scanning-report.json
  dependencies: []
  only:
    refs:
      - branches
    variables:
      - $GITLAB_FEATURES =~ /\bdependency_scanning\b/
  except:
    variables:
      - $DEPENDENCY_SCANNING_DISABLED
      - $DS_DISABLE_DIND == 'true'
.analyzer:
  extends: dependency_scanning
  services: []
  except:
    variables:
      - $DS_DISABLE_DIND == 'false'
  script:
    - /analyzer run
gemnasium-dependency_scanning:
  extends: .analyzer
  image:
    name: "$DS_ANALYZER_IMAGE_PREFIX/gemnasium:$DS_MAJOR_VERSION"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bdependency_scanning\b/ &&
        $DS_DEFAULT_ANALYZERS =~ /gemnasium/ &&
        $CI_PROJECT_REPOSITORY_LANGUAGES =~ /ruby|javascript|php/
gemnasium-maven-dependency_scanning:
  extends: .analyzer
  image:
    name: "$DS_ANALYZER_IMAGE_PREFIX/gemnasium-maven:$DS_MAJOR_VERSION"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bdependency_scanning\b/ &&
        $DS_DEFAULT_ANALYZERS =~ /gemnasium-maven/ &&
        $CI_PROJECT_REPOSITORY_LANGUAGES =~ /\bjava\b/
gemnasium-python-dependency_scanning:
  extends: .analyzer
  image:
    name: "$DS_ANALYZER_IMAGE_PREFIX/gemnasium-python:$DS_MAJOR_VERSION"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bdependency_scanning\b/ &&
        $DS_DEFAULT_ANALYZERS =~ /gemnasium-python/ &&
        $CI_PROJECT_REPOSITORY_LANGUAGES =~ /python/
bundler-audit-dependency_scanning:
  extends: .analyzer
  image:
    name: "$DS_ANALYZER_IMAGE_PREFIX/bundler-audit:$DS_MAJOR_VERSION"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bdependency_scanning\b/ &&
        $DS_DEFAULT_ANALYZERS =~ /bundler-audit/ &&
        $CI_PROJECT_REPOSITORY_LANGUAGES =~ /ruby/
retire-js-dependency_scanning:
  extends: .analyzer
  image:
    name: "$DS_ANALYZER_IMAGE_PREFIX/retire.js:$DS_MAJOR_VERSION"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bdependency_scanning\b/ &&
        $DS_DEFAULT_ANALYZERS =~ /retire.js/ &&
        $CI_PROJECT_REPOSITORY_LANGUAGES =~ /javascript/
```
> 依赖关系扫描有助于在开发和测试应用程序时（例如，当应用程序使用已知易受攻击的外部（开源）库时）在依赖关系中自动查找安全漏洞  
> 相关性扫描使用这个镜像dependency-scanning  

#### 自动许可合规
```
variables:
  LICENSE_MANAGEMENT_SETUP_CMD: ‘’  # If needed, specify a command to setup your environment with a custom package manager.
license_management:
  stage: test
  image:
    name: "registry.gitlab.com/gitlab-org/security-products/license-management:$CI_SERVER_VERSION_MAJOR-$CI_SERVER_VERSION_MINOR-stable"
    entrypoint: [""]
  variables:
    SETUP_CMD: $LICENSE_MANAGEMENT_SETUP_CMD
  allow_failure: true
  script:
    - /run.sh analyze .
  artifacts:
    reports:
      license_management: gl-license-management-report.json
  dependencies: []
  only:
    refs:
      - branches
    variables:
      - $GITLAB_FEATURES =~ /\blicense_management\b/
  except:
    variables:
      - $LICENSE_MANAGEMENT_DISABLED

```
> 判断依赖是否存在license_management  

#### SAST(静态应用程序安全测试)
```
variables:
  SAST_ANALYZER_IMAGE_PREFIX: “registry.gitlab.com/gitlab-org/security-products/analyzers"
  SAST_DEFAULT_ANALYZERS: "bandit, brakeman, gosec, spotbugs, flawfinder, phpcs-security-audit, security-code-scan, nodejs-scan, eslint, tslint, secrets, sobelow, pmd-apex, kubesec"
  SAST_ANALYZER_IMAGE_TAG: 2
  SAST_DISABLE_DIND: "false"
  SCAN_KUBERNETES_MANIFESTS: "false"
sast:
  stage: test
  allow_failure: true
  artifacts:
    reports:
      sast: gl-sast-report.json
  only:
    refs:
      - branches
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/
  image: docker:stable
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
  services:
    - docker:stable-dind
  script:
    - export SAST_VERSION=${SP_VERSION:-$(echo "$CI_SERVER_VERSION" | sed 's/^\([0-9]*\)\.\([0-9]*\).*/\1-\2-stable/')}
    - |
      if ! docker info &>/dev/null; then
        if [ -z "$DOCKER_HOST" -a "$KUBERNETES_PORT" ]; then
          export DOCKER_HOST='tcp://localhost:2375'
        fi
      fi
    - |
      printenv | grep -E '^(DOCKER_|CI|GITLAB_|FF_|HOME|PWD|OLDPWD|PATH|SHLVL|HOSTNAME)' | cut -d'=' -f1 | \
        (while IFS='\\n' read -r VAR; do unset -v "$VAR"; done; /bin/printenv > .env)
    - |
      docker run \
        --env-file .env \
        --volume "$PWD:/code" \
        --volume /var/run/docker.sock:/var/run/docker.sock \
        "registry.gitlab.com/gitlab-org/security-products/sast:$SAST_VERSION" /app/bin/run /code
  except:
    variables:
      - $SAST_DISABLED
      - $SAST_DISABLE_DIND == 'true'

.analyzer:
  extends: sast
  services: []
  except:
    variables:
      - $SAST_DISABLE_DIND == 'false'
  script:
    - /analyzer run
bandit-sast:
  extends: .analyzer
  image:
    name: "$SAST_ANALYZER_IMAGE_PREFIX/bandit:$SAST_ANALYZER_IMAGE_TAG"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /bandit/&&
          $CI_PROJECT_REPOSITORY_LANGUAGES =~ /python/
brakeman-sast:
  extends: .analyzer
  image:
    name: "$SAST_ANALYZER_IMAGE_PREFIX/brakeman:$SAST_ANALYZER_IMAGE_TAG"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /brakeman/ &&
          $CI_PROJECT_REPOSITORY_LANGUAGES =~ /ruby/
eslint-sast:
  extends: .analyzer
  image:
    name: "$SAST_ANALYZER_IMAGE_PREFIX/eslint:$SAST_ANALYZER_IMAGE_TAG"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /eslint/ &&
          $CI_PROJECT_REPOSITORY_LANGUAGES =~ /javascript/
flawfinder-sast:
  extends: .analyzer
  image:
    name: “$SAST_ANALYZER_IMAGE_PREFIX/flawfinder:$SAST_ANALYZER_IMAGE_TAG”
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /flawfinder/ &&
          $CI_PROJECT_REPOSITORY_LANGUAGES =~ /\b(c\+\+|c)\b/
kubesec-sast:
  extends: .analyzer
  image:
    name: "$SAST_ANALYZER_IMAGE_PREFIX/kubesec:$SAST_ANALYZER_IMAGE_TAG"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /kubesec/ &&
          $SCAN_KUBERNETES_MANIFESTS == 'true'
gosec-sast:
  extends: .analyzer
  image:
    name: "$SAST_ANALYZER_IMAGE_PREFIX/gosec:$SAST_ANALYZER_IMAGE_TAG"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /gosec/ &&
          $CI_PROJECT_REPOSITORY_LANGUAGES =~ /\bgo\b/
nodejs-scan-sast:
  extends: .analyzer
  image:
    name: "$分析源代码中的已知漏洞/nodejs-scan:$SAST_ANALYZER_IMAGE_TAG"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /nodejs-scan/ &&
          $CI_PROJECT_REPOSITORY_LANGUAGES =~ /javascript/
phpcs-security-audit-sast:
  extends: .analyzer
  image:
    name: “$SAST_ANALYZER_IMAGE_PREFIX/phpcs-security-audit:$SAST_ANALYZER_IMAGE_TAG”
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /phpcs-security-audit/ &&
          $CI_PROJECT_REPOSITORY_LANGUAGES =~ /php/
pmd-apex-sast:
  extends: .analyzer
  image:
    name: "$SAST_ANALYZER_IMAGE_PREFIX/pmd-apex:$SAST_ANALYZER_IMAGE_TAG"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /pmd-apex/ &&
          $CI_PROJECT_REPOSITORY_LANGUAGES =~ /apex/
secrets-sast:
  extends: .analyzer
  image:
    name: "$SAST_ANALYZER_IMAGE_PREFIX/secrets:$SAST_ANALYZER_IMAGE_TAG"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /secrets/
security-code-scan-sast:
  extends: .analyzer
  image:
    name: "$SAST_ANALYZER_IMAGE_PREFIX/security-code-scan:$SAST_ANALYZER_IMAGE_TAG"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /security-code-scan/ &&
          $CI_PROJECT_REPOSITORY_LANGUAGES =~ /\b(c\#|visual basic\b)/
sobelow-sast:
  extends: .analyzer
  image:
    name: “$SAST_ANALYZER_IMAGE_PREFIX/sobelow:$SAST_ANALYZER_IMAGE_TAG”
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /sobelow/ &&
          $CI_PROJECT_REPOSITORY_LANGUAGES =~ /elixir/
spotbugs-sast:
  extends: .analyzer
  image:
    name: "$SAST_ANALYZER_IMAGE_PREFIX/spotbugs:$SAST_ANALYZER_IMAGE_TAG"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /spotbugs/ &&
          $CI_PROJECT_REPOSITORY_LANGUAGES =~ /java\b/
tslint-sast:
  extends: .analyzer
  image:
    name: "$SAST_ANALYZER_IMAGE_PREFIX/tslint:$SAST_ANALYZER_IMAGE_TAG"
  only:
    variables:
      - $GITLAB_FEATURES =~ /\bsast\b/ &&
          $SAST_DEFAULT_ANALYZERS =~ /tslint/ &&
          $CI_PROJECT_REPOSITORY_LANGUAGES =~ /typescript/
```

> 使用registry.gitlab.com/gitlab-org/security-products/analyzers (bandit, brakeman, gosec, spotbugs, flawfinder, phpcs-security-audit, security-code-scan, nodejs-scan, eslint, tslint, secrets, sobelow, pmd-apex, kubesec) 镜像分析源代码中的已知漏洞,GitLab检查SAST报告,比较发现的源分支和目标分支之间的漏洞,并在合并请求中直接显示信息  

### Review(自动审核)
```
.auto-deploy:
  image: “registry.gitlab.com/gitlab-org/cluster-integration/auto-deploy-image:v0.7.0”
review:
  extends: .auto-deploy
  stage: review
  script:
    - auto-deploy check_kube_domain
    - auto-deploy download_chart
    - auto-deploy ensure_namespace
    - auto-deploy initialize_tiller
    - auto-deploy create_secret
    - auto-deploy deploy
    - auto-deploy persist_environment_url
  environment:
    name: review/$CI_COMMIT_REF_NAME
    url: http://$CI_PROJECT_ID-$CI_ENVIRONMENT_SLUG.$KUBE_INGRESS_BASE_DOMAIN
    on_stop: stop_review
  artifacts:
    paths: [environment_url.txt]
  only:
    refs:
      - branches
      - tags
    kubernetes: active
  except:
    refs:
      - master
    variables:
      - $REVIEW_DISABLED
stop_review:
  extends: .auto-deploy
  stage: cleanup
  variables:
    GIT_STRATEGY: none
  script:
    - auto-deploy initialize_tiller
    - auto-deploy delete
  environment:
    name: review/$CI_COMMIT_REF_NAME
    action: stop
  when: manual
  allow_failure: true
  only:
    refs:
      - branches
      - tags
    kubernetes: active
  except:
    refs:
      - master
    variables:
      - $REVIEW_DISABLED
# Staging deploys are disabled by default since
# continuous deployment to production is enabled by default
# If you prefer to automatically deploy to staging and
# only manually promote to production, enable this job by setting
# STAGING_ENABLED.
staging:
  extends: .auto-deploy
  stage: staging
  script:
    - auto-deploy check_kube_domain
    - auto-deploy download_chart
    - auto-deploy ensure_namespace
    - auto-deploy initialize_tiller
    - auto-deploy create_secret
    - auto-deploy deploy
  environment:
    name: staging
    url: http://$CI_PROJECT_PATH_SLUG-staging.$KUBE_INGRESS_BASE_DOMAIN
  only:
    refs:
      - master
    kubernetes: active
    variables:
      - $STAGING_ENABLED
# Canaries are disabled by default, but if you want them,
# and know what the downsides are, you can enable this by setting
# CANARY_ENABLED.
canary:
  extends: .auto-deploy
  stage: canary
  script:
    - auto-deploy check_kube_domain
    - auto-deploy download_chart
    - auto-deploy ensure_namespace
    - auto-deploy initialize_tiller
    - auto-deploy create_secret
    - auto-deploy deploy canary
  environment:
    name: production
    url: http://$CI_PROJECT_PATH_SLUG.$KUBE_INGRESS_BASE_DOMAIN
  when: manual
  only:
    refs:
      - master
    kubernetes: active
    variables:
      - $CANARY_ENABLED
.production: &production_template
  extends: .auto-deploy
  stage: production
  script:
    - auto-deploy check_kube_domain
    - auto-deploy download_chart
    - auto-deploy ensure_namespace
    - auto-deploy initialize_tiller
    - auto-deploy create_secret
    - auto-deploy deploy
    - auto-deploy delete canary
    - auto-deploy delete rollout
    - auto-deploy persist_environment_url
  environment:
    name: production
    url: http://$CI_PROJECT_PATH_SLUG.$KUBE_INGRESS_BASE_DOMAIN
  artifacts:
    paths: [environment_url.txt]
production:
  <<: *production_template
  only:
    refs:
      - master
    kubernetes: active
  except:
    variables:
      - $STAGING_ENABLED
      - $CANARY_ENABLED
      - $INCREMENTAL_ROLLOUT_ENABLED
      - $INCREMENTAL_ROLLOUT_MODE
production_manual:
  <<: *production_template
  when: manual
  allow_failure: false
  only:
    refs:
      - master
    kubernetes: active
    variables:
      - $STAGING_ENABLED
      - $CANARY_ENABLED
  except:
    variables:
      - $INCREMENTAL_ROLLOUT_ENABLED
      - $INCREMENTAL_ROLLOUT_MODE
# This job implements incremental rollout on for every push to `master`.
.rollout: &rollout_template
  extends: .auto-deploy
  script:
    - auto-deploy check_kube_domain
    - auto-deploy download_chart
    - auto-deploy ensure_namespace
    - auto-deploy initialize_tiller
    - auto-deploy create_secret
    - auto-deploy deploy rollout $ROLLOUT_PERCENTAGE
    - auto-deploy scale stable $((100-ROLLOUT_PERCENTAGE))
    - auto-deploy delete canary
    - auto-deploy persist_environment_url
  environment:
    name: production
    url: http://$CI_PROJECT_PATH_SLUG.$KUBE_INGRESS_BASE_DOMAIN
  artifacts:
    paths: [environment_url.txt]
.manual_rollout_template: &manual_rollout_template
  <<: *rollout_template
  stage: production
  when: manual
  # This selectors are backward compatible mode with $INCREMENTAL_ROLLOUT_ENABLED (before 11.4)
  only:
    refs:
      - master
    kubernetes: active
    variables:
      - $INCREMENTAL_ROLLOUT_MODE == "manual"
      - $INCREMENTAL_ROLLOUT_ENABLED
  except:
    variables:
      - $INCREMENTAL_ROLLOUT_MODE == "timed"
.timed_rollout_template: &timed_rollout_template
  <<: *rollout_template
  when: delayed
  start_in: 5 minutes
  only:
    refs:
      - master
    kubernetes: active
    variables:
      - $INCREMENTAL_ROLLOUT_MODE == "timed"
timed rollout 10%:
  <<: *timed_rollout_template
  stage: incremental rollout 10%
  variables:
    ROLLOUT_PERCENTAGE: 10
timed rollout 25%:
  <<: *timed_rollout_template
  stage: incremental rollout 25%
  variables:
    ROLLOUT_PERCENTAGE: 25
timed rollout 50%:
  <<: *timed_rollout_template
  stage: incremental rollout 50%
  variables:
    ROLLOUT_PERCENTAGE: 50
timed rollout 100%:
  <<: *timed_rollout_template
  <<: *production_template
  stage: incremental rollout 100%
  variables:
    ROLLOUT_PERCENTAGE: 100
rollout 10%:
  <<: *manual_rollout_template
  variables:
    ROLLOUT_PERCENTAGE: 10
rollout 25%:
  <<: *manual_rollout_template
  variables:
    ROLLOUT_PERCENTAGE: 25
rollout 50%:
  <<: *manual_rollout_template
  variables:
    ROLLOUT_PERCENTAGE: 50
rollout 100%:
  <<: *manual_rollout_template
  <<: *production_template
  allow_failure: false

```
> 总共有几步骤  
- 检查ingress ip
- 下载chart
- 创建命名空间
- 初始化helm
- 创建secret
- 部署chart
- 获取域名

### DAST(动态应用程序安全性测试)[此阶段依赖review阶段]
```
.dast-auto-deploy:
  image: “registry.gitlab.com/gitlab-org/cluster-integration/auto-deploy-image:v0.6.0”
dast_environment_deploy:
  extends: .dast-auto-deploy
  stage: review
  script:
    - auto-deploy check_kube_domain
    - auto-deploy download_chart
    - auto-deploy ensure_namespace
    - auto-deploy initialize_tiller
    - auto-deploy create_secret
    - auto-deploy deploy
    - auto-deploy persist_environment_url
  environment:
    name: dast-default
    url: http://dast-$CI_PROJECT_ID-$CI_ENVIRONMENT_SLUG.$KUBE_INGRESS_BASE_DOMAIN
    on_stop: stop_dast_environment
  artifacts:
    paths: [environment_url.txt]
  only:
    refs:
      - branches
    variables:
      - $GITLAB_FEATURES =~ /\bdast\b/
    kubernetes: active
  except:
    variables:
      - $CI_DEFAULT_BRANCH != $CI_COMMIT_REF_NAME
      - $DAST_DISABLED || $DAST_DISABLED_FOR_DEFAULT_BRANCH
      - $DAST_WEBSITE  # we don't need to create a review app if a URL is already given
stop_dast_environment:
  extends: .dast-auto-deploy
  stage: cleanup
  variables:
    GIT_STRATEGY: none
  script:
    - auto-deploy initialize_tiller
    - auto-deploy delete
  environment:
    name: dast-default
    action: stop
  needs: ["dast"]
  only:
    refs:
      - branches
    variables:
      - $GITLAB_FEATURES =~ /\bdast\b/
    kubernetes: active
  except:
    variables:
      - $CI_DEFAULT_BRANCH != $CI_COMMIT_REF_NAME
      - $DAST_DISABLED || $DAST_DISABLED_FOR_DEFAULT_BRANCH
      - $DAST_WEBSITE
```

```
stages:
  - build
  - test
  - deploy
  - dast
dast:
  stage: dast
  image:
    name: "registry.gitlab.com/gitlab-org/security-products/dast:$CI_SERVER_VERSION_MAJOR-$CI_SERVER_VERSION_MINOR-stable"
  variables:
  # URL to scan:
  # DAST_WEBSITE: https://example.com/
  #
  # Time limit for target availability (scan is attempted even when timeout):
  # DAST_TARGET_AVAILABILITY_TIMEOUT: 60
  #
  # Set these variables to scan with an authenticated user:
  # DAST_AUTH_URL: https://example.com/sign-in
  # DAST_USERNAME: john.doe@example.com
  # DAST_PASSWORD: john-doe-password
  # DAST_USERNAME_FIELD: session[user] # the name of username field at the sign-in HTML form
  # DAST_PASSWORD_FIELD: session[password] # the name of password field at the sign-in HTML form
  # DAST_AUTH_EXCLUDE_URLS: http://example.com/sign-out,http://example.com/sign-out-2 # optional: URLs to skip during the authenticated scan; comma-separated, no spaces in between
  #
  # Perform ZAP Full Scan, which includes both passive and active scanning:
  # DAST_FULL_SCAN_ENABLED: "true"
  allow_failure: true
  script:
    - export DAST_WEBSITE=${DAST_WEBSITE:-$(cat environment_url.txt)}
    - /analyze -t $DAST_WEBSITE
  artifacts:
    reports:
      dast: gl-dast-report.json
  only:
    refs:
      - branches
    variables:
      - $GITLAB_FEATURES =~ /\bdast\b/
  except:
    variables:
      - $DAST_DISABLED
      - $DAST_DISABLED_FOR_DEFAULT_BRANCH && $CI_DEFAULT_BRANCH == $CI_COMMIT_REF_NAME
```
> 使用流行的开源工具 [OWASP ZAProxy](https://github.com/zaproxy/zaproxy)  对当前代码进行分析并检查潜在的安全性问题  

### performance(自动浏览器性能测试)
```
performance:
  stage: performance
  image: docker:stable
  allow_failure: true
  variables:
    DOCKER_TLS_CERTDIR: ""
  services:
    - docker:stable-dind
  script:
    - |
      if ! docker info &>/dev/null; then
        if [ -z "$DOCKER_HOST" -a "$KUBERNETES_PORT" ]; then
          export DOCKER_HOST='tcp://localhost:2375'
        fi
      fi
    - export CI_ENVIRONMENT_URL=$(cat environment_url.txt)
    - mkdir gitlab-exporter
    - wget -O gitlab-exporter/index.js https://gitlab.com/gitlab-org/gl-performance/raw/10-5/index.js
    - mkdir sitespeed-results
    - |
      if [ -f .gitlab-urls.txt ]
      then
        sed -i -e 's@^@'"$CI_ENVIRONMENT_URL"'@' .gitlab-urls.txt
        docker run --shm-size=1g --rm -v "$(pwd)":/sitespeed.io sitespeedio/sitespeed.io:6.3.1 --plugins.add ./gitlab-exporter --outputFolder sitespeed-results .gitlab-urls.txt
      else
        docker run --shm-size=1g --rm -v "$(pwd)":/sitespeed.io sitespeedio/sitespeed.io:6.3.1 --plugins.add ./gitlab-exporter --outputFolder sitespeed-results "$CI_ENVIRONMENT_URL"
      fi
    - mv sitespeed-results/data/performance.json performance.json
  artifacts:
    paths:
      - performance.json
      - sitespeed-results/
  only:
    refs:
      - branches
      - tags
    kubernetes: active
  except:
    variables:
      - $PERFORMANCE_DISABLED
```
> 使用sitespeedio/sitespeed.io:6.3.1这个镜像来衡量网页的性能。将创建JSON报告并将其作为工件上传，其中包括每个页面的总体性能得分。默认情况下，将测试“Review”和“Production”环境的根页面。如果要添加其他URL进行测试，只需将路径添加到.gitlab-urls.txt根目录中命名的文件中  

### deploy(review,staging,production)
> 总共有几步骤  
- 检查ingress ip
- 下载chart
- 创建命名空间
- 初始化helm
- 创建secret
- 部署chart
- 获取域名
- 清除相关

### monitoring(自动监控)
> 一旦部署了应用程序，“自动监视”便可以立即监视应用程序的服务器和响应指标。自动监控使用 [Prometheus](https://docs.gitlab.com/ce/user/project/integrations/prometheus.html) 直接从 [Kubernetes](https://docs.gitlab.com/ce/user/project/integrations/prometheus_library/kubernetes.html) 获取系统指标，例如CPU和内存使用情况 ，以及从 [NGINX服务器获取](https://docs.gitlab.com/ce/user/project/integrations/prometheus_library/nginx_ingress.html) 响应指标，例如HTTP错误率，延迟和吞吐量   

### 其他
1. DB_INITIALIZE和DB_MIGRATE(在没有自定义Dockerfile的情况下)
> 自动迁移数据库和脚本  

## auto-deploy 脚本说明
```
#!/bin/bash -e
[[ "$TRACE" ]] && set -x
export RELEASE_NAME=${HELM_RELEASE_NAME:-$CI_ENVIRONMENT_SLUG}
auto_database_url=postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${RELEASE_NAME}-postgres:5432/${POSTGRES_DB}
export DATABASE_URL=${DATABASE_URL-$auto_database_url}
export TILLER_NAMESPACE=$KUBE_NAMESPACE
export HELM_HOST="localhost:44134"
function check_kube_domain() {
  if [[ -z "$KUBE_INGRESS_BASE_DOMAIN" ]]; then
    echo "In order to deploy or use Review Apps,"
    echo "KUBE_INGRESS_BASE_DOMAIN variables must be set"
    echo "From 11.8, you can set KUBE_INGRESS_BASE_DOMAIN in cluster settings"
    echo "or by defining a variable at group or project level."
    echo "You can also manually add it in .gitlab-ci.yml"
    false
  else
    true
  fi
}
function download_chart() {
  if [[ ! -d chart ]]; then
    auto_chart=${AUTO_DEVOPS_CHART:-gitlab/auto-deploy-app}
    # shellcheck disable=SC2086 # double quote variables to prevent globbing
    auto_chart_name=$(basename $auto_chart)
    auto_chart_name=${auto_chart_name%.tgz}
    auto_chart_name=${auto_chart_name%.tar.gz}
  else
    auto_chart="chart"
    auto_chart_name="chart"
  fi
  helm init --client-only
  # shellcheck disable=SC2086 # double quote variables to prevent globbing
  # shellcheck disable=SC2140 # ambiguous quoting warning
  helm repo add ${AUTO_DEVOPS_CHART_REPOSITORY_NAME:-gitlab} ${AUTO_DEVOPS_CHART_REPOSITORY:-https://charts.gitlab.io} ${AUTO_DEVOPS_CHART_REPOSITORY_USERNAME:+"--username" "$AUTO_DEVOPS_CHART_REPOSITORY_USERNAME"} ${AUTO_DEVOPS_CHART_REPOSITORY_PASSWORD:+"--password" "$AUTO_DEVOPS_CHART_REPOSITORY_PASSWORD"}
  if [[ ! -d "$auto_chart" ]]; then
    helm fetch ${auto_chart} --untar
  fi
  if [ "$auto_chart_name" != "chart" ]; then
    mv ${auto_chart_name} chart
  fi
  helm dependency update chart/
  helm dependency build chart/
}
function ensure_namespace() {
  kubectl get namespace "$KUBE_NAMESPACE" || kubectl create namespace "$KUBE_NAMESPACE"
}
function initialize_tiller() {
  echo "Checking Tiller..."
  nohup tiller -listen ${HELM_HOST} -alsologtostderr >/dev/null 2>&1 &
  echo "Tiller is listening on ${HELM_HOST}"
  if ! helm version --debug; then
    echo "Failed to init Tiller."
    return 1
  fi
  echo ""
}
function create_secret() {
  echo "Create secret..."
  if [[ "$CI_PROJECT_VISIBILITY" == "public" ]]; then
    return
  fi
  kubectl create secret -n "$KUBE_NAMESPACE" \
    docker-registry gitlab-registry \
    --docker-server="$CI_REGISTRY" \
    --docker-username="${CI_DEPLOY_USER:-$CI_REGISTRY_USER}" \
    --docker-password="${CI_DEPLOY_PASSWORD:-$CI_REGISTRY_PASSWORD}" \
    --docker-email="$GITLAB_USER_EMAIL" \
    -o yaml --dry-run | kubectl replace -n "$KUBE_NAMESPACE" --force -f -
}
# shellcheck disable=SC2086
function persist_environment_url() {
  echo $CI_ENVIRONMENT_URL >environment_url.txt
}
# shellcheck disable=SC2153 # warns that my_var vs MY_VAR is a possible misspelling
# shellcheck disable=SC2154 # env_ADDITIONAL_HOSTS eval assignment is not recognized
function deploy() {
  track="${1-stable}"
  percentage="${2:-100}"
  name=$(deploy_name "$track")
  if [[ -z "$CI_COMMIT_TAG" ]]; then
    image_repository=${CI_APPLICATION_REPOSITORY:-$CI_REGISTRY_IMAGE/$CI_COMMIT_REF_SLUG}
    image_tag=${CI_APPLICATION_TAG:-$CI_COMMIT_SHA}
  else
    image_repository=${CI_APPLICATION_REPOSITORY:-$CI_REGISTRY_IMAGE}
    image_tag=${CI_APPLICATION_TAG:-$CI_COMMIT_TAG}
  fi
  service_enabled="true"
  postgres_enabled="$POSTGRES_ENABLED"
  # if track is different than stable,
  # re-use all attached resources
  if [[ "$track" != "stable" ]]; then
    service_enabled="false"
    postgres_enabled="false"
  fi
  replicas=$(get_replicas "$track" "$percentage")
  if [[ "$CI_PROJECT_VISIBILITY" != "public" ]]; then
    secret_name='gitlab-registry'
  else
    secret_name=''
  fi
  if [[ -n "$AUTO_DEVOPS_MODSECURITY_SEC_RULE_ENGINE" ]]; then
    modsecurity_enabled="true"
  fi
  create_application_secret "$track"
  # shellcheck disable=SC2086 # double quote variables to prevent globbing
  env_slug=$(echo ${CI_ENVIRONMENT_SLUG//-/_} | tr -s '[:lower:]' '[:upper:]')
  # shellcheck disable=SC2086 # double quote variables to prevent globbing
  eval env_ADDITIONAL_HOSTS=\$${env_slug}_ADDITIONAL_HOSTS
  if [ -n "$env_ADDITIONAL_HOSTS" ]; then
    additional_hosts="{$env_ADDITIONAL_HOSTS}"
  elif [ -n "$ADDITIONAL_HOSTS" ]; then
    additional_hosts="{$ADDITIONAL_HOSTS}"
  fi
  # shellcheck disable=SC2086 # HELM_UPGRADE_EXTRA_ARGS -- double quote variables to prevent globbing
  if [[ -n "$DB_INITIALIZE" && -z "$(helm ls -q "^$name$")" ]]; then
    echo "Deploying first release with database initialization..."
    helm upgrade --install \
      --wait \
      --set service.enabled="$service_enabled" \
      --set gitlab.app="$CI_PROJECT_PATH_SLUG" \
      --set gitlab.env="$CI_ENVIRONMENT_SLUG" \
      --set gitlab.envName="$CI_ENVIRONMENT_NAME" \
      --set gitlab.envURL="$CI_ENVIRONMENT_URL" \
      --set releaseOverride="$RELEASE_NAME" \
      --set image.repository="$image_repository" \
      --set image.tag="$image_tag" \
      --set image.pullPolicy=IfNotPresent \
      --set image.secrets[0].name="$secret_name" \
      --set application.track="$track" \
      --set application.database_url="$DATABASE_URL" \
      --set application.secretName="$APPLICATION_SECRET_NAME" \
      --set application.secretChecksum="$APPLICATION_SECRET_CHECKSUM" \
      --set service.commonName="le-$CI_PROJECT_ID.$KUBE_INGRESS_BASE_DOMAIN" \
      --set service.url="$CI_ENVIRONMENT_URL" \
      --set service.additionalHosts="$additional_hosts" \
      --set replicaCount="$replicas" \
      --set postgresql.enabled="$postgres_enabled" \
      --set postgresql.nameOverride="postgres" \
      --set postgresql.postgresUser="$POSTGRES_USER" \
      --set postgresql.postgresPassword="$POSTGRES_PASSWORD" \
      --set postgresql.postgresDatabase="$POSTGRES_DB" \
      --set postgresql.imageTag="$POSTGRES_VERSION" \
      --set application.initializeCommand="$DB_INITIALIZE" \
      --set ingress.modSecurity.enabled="$modsecurity_enabled" \
      --set ingress.modSecurity.secRuleEngine="$AUTO_DEVOPS_MODSECURITY_SEC_RULE_ENGINE" \
      $HELM_UPGRADE_EXTRA_ARGS \
      --namespace="$KUBE_NAMESPACE" \
      "$name" \
      chart/
    echo "Deploying second release..."
    helm upgrade --reuse-values \
      --wait \
      --set application.initializeCommand="" \
      --set application.migrateCommand="$DB_MIGRATE" \
      $HELM_UPGRADE_EXTRA_ARGS \
      --namespace="$KUBE_NAMESPACE" \
      "$name" \
      chart/
  else
    echo "Deploying new release..."
    helm upgrade --install \
      --wait \
      --set service.enabled="$service_enabled" \
      --set gitlab.app="$CI_PROJECT_PATH_SLUG" \
      --set gitlab.env="$CI_ENVIRONMENT_SLUG" \
      --set gitlab.envName="$CI_ENVIRONMENT_NAME" \
      --set gitlab.envURL="$CI_ENVIRONMENT_URL" \
      --set releaseOverride="$RELEASE_NAME" \
      --set image.repository="$image_repository" \
      --set image.tag="$image_tag" \
      --set image.pullPolicy=IfNotPresent \
      --set image.secrets[0].name="$secret_name" \
      --set application.track="$track" \
      --set application.database_url="$DATABASE_URL" \
      --set application.secretName="$APPLICATION_SECRET_NAME" \
      --set application.secretChecksum="$APPLICATION_SECRET_CHECKSUM" \
      --set service.commonName="le-$CI_PROJECT_ID.$KUBE_INGRESS_BASE_DOMAIN" \
      --set service.url="$CI_ENVIRONMENT_URL" \
      --set service.additionalHosts="$additional_hosts" \
      --set replicaCount="$replicas" \
      --set postgresql.enabled="$postgres_enabled" \
      --set postgresql.nameOverride="postgres" \
      --set postgresql.postgresUser="$POSTGRES_USER" \
      --set postgresql.postgresPassword="$POSTGRES_PASSWORD" \
      --set postgresql.postgresDatabase="$POSTGRES_DB" \
      --set postgresql.imageTag="$POSTGRES_VERSION" \
      --set application.migrateCommand="$DB_MIGRATE" \
      --set ingress.modSecurity.enabled="$modsecurity_enabled" \
      --set ingress.modSecurity.secRuleEngine="$AUTO_DEVOPS_MODSECURITY_SEC_RULE_ENGINE" \
      $HELM_UPGRADE_EXTRA_ARGS \
      --namespace="$KUBE_NAMESPACE" \
      "$name" \
      chart/
  fi
  if [[ -z "$ROLLOUT_STATUS_DISABLED" ]]; then
    kubectl rollout status -n "$KUBE_NAMESPACE" -w "$ROLLOUT_RESOURCE_TYPE/$name"
  fi
}
function scale() {
  track="${1-stable}"
  percentage="${2-100}"
  name=$(deploy_name "$track")
  replicas=$(get_replicas "$track" "$percentage")
  if [[ -n "$(helm ls -q "^$name$")" ]]; then
    helm upgrade --reuse-values \
      --wait \
      --set replicaCount="$replicas" \
      --namespace="$KUBE_NAMESPACE" \
      "$name" \
      chart/
  fi
}
function delete() {
  track="${1-stable}"
  name=$(deploy_name "$track")
  if [[ -n "$(helm ls -q "^$name$")" ]]; then
    helm delete --purge "$name"
  fi
  secret_name=$(application_secret_name "$track")
  kubectl delete secret --ignore-not-found -n "$KUBE_NAMESPACE" "$secret_name"
}
## Helper functions
##
# Extracts variables prefixed with K8S_SECRET_
# and creates a Kubernetes secret.
#
# e.g. If we have the following environment variables:
#   K8S_SECRET_A=value1
#   K8S_SECRET_B=multi\ word\ value
#
# Then we will create a secret with the following key-value pairs:
#   data:
#     A: dmFsdWUxCg==
#     B: bXVsdGkgd29yZCB2YWx1ZQo=
#
function create_application_secret() {
  track="${1-stable}"
  # shellcheck disable=SC2155 # declare and assign separately to avoid masking return values.
  export APPLICATION_SECRET_NAME=$(application_secret_name "$track")
  env | sed -n "s/^K8S_SECRET_\(.*\)$/\1/p" >k8s_prefixed_variables
  kubectl create secret \
    -n "$KUBE_NAMESPACE" generic "$APPLICATION_SECRET_NAME" \
    --from-env-file k8s_prefixed_variables -o yaml --dry-run |
    kubectl replace -n "$KUBE_NAMESPACE" --force -f -
  # shellcheck disable=SC2002 # useless cat, prefer cmd < file
  # shellcheck disable=SC2155 # declare and assign separately to avoid masking return values.
  export APPLICATION_SECRET_CHECKSUM=$(cat k8s_prefixed_variables | sha256sum | cut -d ' ' -f 1)
  rm k8s_prefixed_variables
}
function application_secret_name() {
  track="${1-stable}"
  name=$(deploy_name "$track")
  echo "${name}-secret"
}
# shellcheck disable=SC2086
function deploy_name() {
  name="$RELEASE_NAME"
  track="${1-stable}"
  if [[ "$track" != "stable" ]]; then
    name="$name-$track"
  fi
  echo $name
}
# shellcheck disable=SC2004 # $/${} is unnecessary on arithmetic variables.
# shellcheck disable=SC2086 # double quote to prevent globbing
# shellcheck disable=SC2153 # incorrectly thinks replicas vs REPLICAS is a misspelling
function get_replicas() {
  track="${1:-stable}"
  percentage="${2:-100}"
  env_track=$(echo $track | tr -s '[:lower:]' '[:upper:]')
  env_slug=$(echo ${CI_ENVIRONMENT_SLUG//-/_} | tr -s '[:lower:]' '[:upper:]')
  if [[ "$track" == "stable" ]] || [[ "$track" == "rollout" ]]; then
    # for stable track get number of replicas from `PRODUCTION_REPLICAS`
    eval new_replicas=\$${env_slug}_REPLICAS
    if [[ -z "$new_replicas" ]]; then
      new_replicas=$REPLICAS
    fi
  else
    # for all tracks get number of replicas from `CANARY_PRODUCTION_REPLICAS`
    eval new_replicas=\$${env_track}_${env_slug}_REPLICAS
    if [[ -z "$new_replicas" ]]; then
      eval new_replicas=\$${env_track}_REPLICAS
    fi
  fi
  replicas="${new_replicas:-1}"
  replicas="$(($replicas * $percentage / 100))"
  # always return at least one replicas
  if [[ $replicas -gt 0 ]]; then
    echo "$replicas"
  else
    echo 1
  fi
}
##
## End Helper functions
option=$1
case $option in
  check_kube_domain) check_kube_domain ;;
  download_chart) download_chart ;;
  ensure_namespace) ensure_namespace ;;
  initialize_tiller) initialize_tiller ;;
  create_secret) create_secret ;;
  persist_environment_url) persist_environment_url ;;
  deploy) deploy "${@:2}" ;;
  scale) scale "${@:2}" ;;
  delete) delete "${@:2}" ;;
  create_application_secret) create_application_secret "${@:2}" ;;
  deploy_name) deploy_name "${@:2}" ;;
  get_replicas) get_replicas "${@:2}" ;;
  *) exit 1 ;;
esac
```
