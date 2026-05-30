# Pull request открыл — стенд появился. Закрыл — исчез. Эфемерные окружения в kubernetes через FluxCD

Разработчики в основном новые фичи и фиксы пилят локально. Но как быть, если хочется показать, проверить изменения еще кому-то? Можно, конечно, публиковать ветку на kubernetes dev кластер и наблюдать за разворачиванием этой красоты. А как быть, если несколько разработчиков хотят это сделать одновременно, может вообще нужна отдельная БД, или же просто хочется сломать приложение и всей командой смотреть на это, попивая кофе.

> 💡 *Статья на habr: [ephemeral-envs-fluxcd](https://habr.com/ru/articles/1042866/)*

Для реализации задуманного хотелось шаблонизировать разворачиваемую инфраструктуру с подстановкой переменных для эфимерных окружений. Из подходящих инструментов можно рассмотреть [kluctl](https://kluctl.io/), [FluxCD](https://fluxcd.io/). Так как в инфраструктуре уже использовался FluxCD, то долго выбирать не пришлось. Всего лишь добавлением директивы `postBuild` во flux kustomization можно будет шаблонизировать манифесты.

Через CI/CD создается директория эфимерного окружения. Новый Flux kustomization подхватит все манифесты, подставит переменные и развернет окружение с автообновлением образов приложения.

Теперь сложим всю картину вместе. У нас есть 2 git репозитория: [приложение](app-repo) и [инфраструктура](gitops-repo). В репозитории приложения разработчик вешает лейбл "deploy-dev" на Pull Request. На него запускается CI/CD со сборкой образа текущей ветки. Все манифесты пушатся уже в репозиторий инфраструктуры. Выпускается tls сертификат от letsencrypt. И наконец, запускается БД и другие сервисы. Приложение будет доступно по уникальному url.

## Подготовка CI/CD в репозитории приложения
Добавляем в репозиторий приложения лейбл "deploy-dev". Только ветки PR с таким лейблом будут деплоится в dev кластер, а потом удаляться при закрытии или снятии лейбла. Триггером для запуска CI/CD будет любые изменения в PR, включая добавление и удаление лейблов. Как только будет добавлен лейбл к PR, запустится provision, он подтянет build. Затем каждый раз при пуше ветку будет собираться образ до тех пор пока открыт PR с лейблом.

Пайплайн "deploy-dev" состоит из 3х шагов: запуск сборки образа, разворачивание, удаление окружения. Для каждого шага важно верно выставить условия срабатывания.

```yaml
# deploy-dev.yaml
on:
  pull_request:
    types: [opened, closed, synchronize, labeled, unlabeled]
jobs:
  # пересобирается образ при каждом пуше в ветку PR
  build:
    if: |
      github.event.action != 'closed' &&
      github.event.action != 'unlabeled' &&
      contains(github.event.pull_request.labels.*.name, 'deploy-dev')
  # создается окружение по лейблу deploy-dev
  provision:
    needs: [build]
    if: github.event.action == 'labeled'
  # Удаляется эфимерное окружение после закрытия PR или удаления лейбла
  teardown:
    if: |
      (github.event.action == 'unlabeled' && github.event.label.name == 'deploy-dev') ||
      (github.event.action == 'closed' && contains(github.event.pull_request.labels.*.name, 'deploy-dev'))
```

В джобу provision мы поместим только те манифесты, которые будут уникальны для выбранного окружения. Вопрос - а какой минимальный набор манифестов нам нужен, чтобы не перегружать CI/CD. Их будет всего 6:
- flux kustomization для развертывания postgres, redis и rabbitmq
- flux kustomization для развертывания приложения
- ConfigMap для передачи всех переменных из CI/CD
- namespace
- kustomization
- values приложения для добавления дополнительных переменных и автоматического обновления образов

```yaml
# ConfigMap example
apiVersion: v1
kind: ConfigMap
metadata:
  name: "tenant-settings"
  namespace: "pr-1234"
data:
  author: "mishikama"
  environment: "dev"
  db_name: "app_dev"
  namespace: "pr-1234"
  host: "pr-1234.dev.example.com"
  branch: "feature_60530225_ai-integration"
  image_prefix: "feature_60530225_ai_integration"
```

После успешного CI/CD манифесты будут лежать в репозитории инфраструктуры. Values helm чарта для удобства был отделен в другую директорию. В него в дальнейшем будут пушаться новые тэги и в дальнейшем можно будет внести какие-то изменения поверх базовых манифестов. Но при желании можно вообще все держать в одном файле.

## Шаблонизация в репозитории инфраструктуры
В корневой FluxCD директории для dev кластера нужно создать flux kustomization, который будет смотреть в директорию ephemeral. Базовый минимум для template директории приложения, все манифесты универсальные для создаваемых окружений.

```bash
helmrelease-patch.yaml
image-automation.yaml
image-policy.yaml
kustomization.yaml
values.yaml
```

В любой из манифестов можно подставить переменные, достаточно для ключа прописать значение `${value}` из ConfigMap. ImagePolicy позволяет получать новейшие образы из container registry, а ImageAutomation делает пуш этого тэга в `values.yaml`.

```yaml
# values.yaml
envs:
  APP_URL: https://${host}
  APP_ENV: ${environment}
  POSTGRES_DB: ${db_name}
ingresses:
  ${host}:
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt
    hosts:
      - paths:
          - path: /
            serviceName: main
            servicePort: 80
    extraTls:
      - hosts:
          - ${host}
        secretName: ${host}-tls
```

Инфраструктуру кластера удобно описывать слоями [flux2-kustomize-helm-example](https://github.com/fluxcd/flux2-kustomize-helm-example). Например, есть base слой с общими ресурсами для всех окружений, затем идет overlays слои с специфичными для окружения ресурсами. В нашем случае dev слой - это шаблон для эфимерных окружений.

Проследим за магией FluxCD, как он по порядку подтягивает все манифесты. FluxCD сначала следит за своим корневым каталогом, там он находит flux kustomization, который указывает на директорию ephemeral, затем каждый flux kustomization смотрит уже на самый верхний слой приложения previews своего окружения, и уже самый верхний слой k8s kustomization подтягивает все нижестоящие слои — template и base. Цепочка длинная, но при этом все вышестоящие слои дополняют, переопределяют все нижестоящие.

```bash
fluxcd
  -> flux ks ephemeral (pr-1234.yaml)
    -> flux ks services (services/_pr-template)
    -> flux ks app (apps/previews/pr-1234)
      -> k8s ks preview (pr-1234) -> k8s ks templates (_pr-template) -> k8s ks base
```

Порядок может показаться странным. Почему начинается все с верхних слоев? А мы в базовом слое не сможем ссылаться на overlays слои, тогда все остальные окружения подтянут это. Конфликтов никаких не будет. Kustomization patch в overlays слоях будет перезаписывать базовые значения. А в helmrelease важно соблюдать очередность подтягивания ConfigMaps.

```yaml
# helmrelease.yaml
valuesFrom:
  - kind: ConfigMap
    name: app-values
  - kind: ConfigMap
    name: app-values-overrides
  - kind: ConfigMap
    name: app-values-ephemeral
```

В зависимости от создания БД, ее инициализация может занимать некоторое время, например, 5 минут. Для корректного первого запуска и автоматических последующих миграций можно добавить инит контейнер. И тогда основной контейнер будет запущен только после завершения успешной миграции.

```yaml
# values.yaml
initContainers:
  - name: migration
    args: ["rake", "db:migrate"]
    envSecrets:
      - db-credentials
```

## Возможные проблемы
Каждое разворачивание новых окружений съедает немало ресурсов и свободного пространства на жестком диске. Можно прописать ограничения ресурсов и размера pv на уровне namespace через `ResourceQuota`. А для установки лимитов на количество создаваемых окружений можно посмотреть в сторону Kyverno ClusterPolicy и выставить ограничения на создания новых namespace по шаблону `^pr-[0-9]+$`.

## Итог
Теперь вы знаете, как автоматизировать создание эфемерных окружений с помощью FluxCD одной кнопкой в GitHub.

После добавления этой автоматизации dev кластер упал. Пошли выяснять, что случилось. Открываем список PR и видим 5 PR с этим новым лейблом "deploy-dev". Изначально ресурсы кластера были рассчитаны всего лишь на 3 одновременных окружения и никаких ограничений не было.

Когда я искал готовые решения для быстрого разворачивания окружений, то не нашел ничего подходящего. Если вам приходилось автоматизировать и/или использовать инструменты поднятия временных окружений, то расскажите об этом. Также интересно узнать, как вы в команде тестируете приложения с доступом извне.

А вообще, Flux или Argo? Кто что использует?

## Ссылки
- [ephemeral-envs-fluxcd](https://github.com/mishikama/it-articles/tree/ephemeral-envs/ephemeral-envs-fluxcd) — github репозиторий с примерами из статьи
- [flux2-kustomize-helm-example](https://github.com/fluxcd/flux2-kustomize-helm-example) — пример многослойной структуры репозитория
- [nxs-universal-chart](https://github.com/nixys/nxs-universal-chart) — универсальный Helm чарт
