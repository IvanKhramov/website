## Требования

- GitLab;

- Kubernetes для запуска GitLab Runner.

## Установка и регистрация GitLab Runner

Для установки GitLab Runner в Kubernetes и его регистрации в GitLab следуйте [официальным инструкциям](https://docs.gitlab.com/runner/register/index.html).

## Настройка GitLab Runner

Измените конфигурацию зарегистрированного GitLab Runner'а, добавив в его `config.toml` следующие параметры:

```toml
[[runners]]
  name = <имя зарегистрированного Runner'а>
  [runners.kubernetes]
    namespace = "gitlab-ci"
    [runners.kubernetes.pod_annotations]
      "container.apparmor.security.beta.kubernetes.io/build" = "unconfined"
    [runners.kubernetes.pod_security_context]
      run_as_non_root = true
      run_as_user = 1000
      run_as_group = 1000
      fs_group = 1000
```

Если нужно кеширование `.werf` и `/builds` (рекомендуется), то добавьте эти параметры:

```toml
[[runners]]
  name = <имя зарегистрированного Runner'а>
  [runners.kubernetes]
    [[runners.kubernetes.volumes.pvc]]
      name = "gitlab-ci-kubernetes-executor-werf-cache"
      mount_path = "/home/build/.werf"
    [[runners.kubernetes.volumes.pvc]]
      name = "gitlab-ci-kubernetes-executor-builds-cache"
      mount_path = "/builds"
```

... а если не нужно, то добавьте эти:

```toml
[[runners]]
  name = <имя зарегистрированного Runner'а>
  [runners.kubernetes]
    [[runners.kubernetes.volumes.empty_dir]]
      name = "gitlab-ci-kubernetes-executor-werf-cache"
      mount_path = "/home/build/.werf"
    [[runners.kubernetes.volumes.empty_dir]]
      name = "gitlab-ci-kubernetes-executor-builds-cache"
      mount_path = "/builds"
```

Если будете развертывать приложения с werf в тот же кластер, в котором запускается GitLab Runner, то добавьте ещё один параметр:

```toml
[[runners]]
  name = <имя зарегистрированного Runner'а>
  [runners.kubernetes]
    service_account = "gitlab-ci-kubernetes-executor"
```

При желании произведите [дополнительную конфигурацию](https://docs.gitlab.com/runner/configuration/advanced-configuration.html) GitLab Runner'а.

## Настройка Kubernetes

Если в конфигурации GitLab Runner вы включили кеширование `.werf` и `/builds`, то создайте в кластере соответствующие PersistentVolumeClaims, например:

```shell
$ kubectl create -f - <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-ci-kubernetes-executor-werf-cache
  namespace: gitlab-ci
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-ci-kubernetes-executor-builds-cache
  namespace: gitlab-ci
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 30Gi
EOF
```

Если будете развертывать приложения в тот же кластер, в котором запускается GitLab Runner, то в кластере для запуска GitLab Runner настройте RBAC следующей командой:

```shell
$ kubectl create -f - <<EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-ci-kubernetes-executor
  namespace: gitlab-ci
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitlab-ci-kubernetes-executor
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: gitlab-ci-kubernetes-executor
    namespace: gitlab-ci
EOF
```

> Для большей безопасности можете создать более ограниченную ClusterRole/Role и использовать её вместо кластерной роли `cluster-admin` выше.

Если Kubernetes-ноды для GitLab Runner имеют ядро Linux версии 5.12 или ниже, то разрешите в этом кластере использование FUSE для GitLab Runner следующей командой:

```shell
$ kubectl create -f - <<EOF
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fuse-device-plugin
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fuse-device-plugin
  template:
    metadata:
      labels:
        name: fuse-device-plugin
    spec:
      hostNetwork: true
      containers:
      - image: soolaugust/fuse-device-plugin:v1.0
        name: fuse-device-plugin-ctr
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        volumeMounts:
          - name: device-plugin
            mountPath: /var/lib/kubelet/device-plugins
      volumes:
        - name: device-plugin
          hostPath:
            path: /var/lib/kubelet/device-plugins
---
apiVersion: v1
kind: LimitRange
metadata:
  name: enable-fuse
  namespace: gitlab-ci
spec:
  limits:
  - type: "Container"
    default:
      github.com/fuse: 1
EOF
```

## Настройка проекта GitLab

- Включите [требование удачно выполненного pipeline для merge requests](https://docs.gitlab.com/ee/user/project/merge_requests/merge_when_pipeline_succeeds.html#require-a-successful-pipeline-for-merge).

- Включите [возможность автоматически отменять лишние pipelines](https://docs.gitlab.com/ee/ci/pipelines/settings.html#auto-cancel-redundant-pipelines).

- [Создайте и сохраните access token](https://docs.gitlab.com/ee/user/project/settings/project_access_tokens.html#create-a-project-access-token) для очистки ненужных образов из container registry со следующей конфигурацией:
  
  - **Token name:** `werf-images-cleanup`
  
  - **Role:** `developer`
  
  - **Scopes:** `api`

- [В переменных проекта](https://docs.gitlab.com/ee/ci/variables/#for-a-project) добавьте следующие переменные:
  
  - Версия werf:
    
    - **Key:** `WERF_VERSION`
    
    - **Value:** `{{ include.version }} {{ include.channel }}`
  
  - Access token для очистки ненужных образов:
    
    - **Key:** `WERF_IMAGES_CLEANUP_PASSWORD`
    
    - **Value:** `<сохранённый "werf-images-cleanup" access token>`
    
    - **Protect variable:** `yes`
    
    - **Mask variable:** `yes`

- [Добавьте плановое задание](https://docs.gitlab.com/ee/ci/pipelines/schedules.html#add-a-pipeline-schedule) на каждую ночь для очистки ненужных образов в container registry, указав ветку `main`/`master` в качестве **Target branch**.

## Конфигурация CI/CD проекта

TODO

## TODO

* change context in jobs in CI
* ```
  image:
      name: "registry.werf.io/werf/werf:1.2-stable"
      pull_policy: always
  before_script:
  - source "$(werf ci-env gitlab --as-file)"
  ```