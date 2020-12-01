---
title: "Достать манифест из k8s определенной apiVersion"
date: 2020-11-23T19:28:10+03:00
draft: false
description: >-
  Учимся преобразовывать маинфесты из одной apiVersion в другую средствами куба.
categories: ["devops", "kubernetes", "K8S"]
toc: false
displayInMenu: false
displayInList: true
resources:
- name: featuredImage
  src: "Filename of the post's featured image, used as the card image and the image at the top of the article"
  params:
    description: "Get manifest from k8s post image"
---

## TL;DR
> Чтобы получить из куба манифест в любой поддерживаемой кубом версии, нужно использовать развернутую форму `kubectl get`: `kubectl get deployments.v1beta1.extensions mydeploy -o yaml`

## Кратко и с примерами
Все умеют получать манифест ресурса из куба. Однако мало кто знает, что в куб можно передать манифест с одной `apiVersion`, а получить - с другой.

### Практический кейс, синтетический пример:

1. Есть `deployment` в кластере, который деплоится манифестом с `api-version extensions/v1beta1` из репы.
1. Обновили кластер (1.15 => 1.16). Старый деплой в кластере и работает, но ничего нового задеплоить старым манифестом не получится, т.к. в 1.16 `api-version extensions/v1beta1` выпилили.
1. Есть два варианта: переписать все ручками, либо просто выдернуть манифест нужного формата из куба (`apps/v1`) и смержить с тем в репе, который хочется деплоить.

Чтоб выудить из куба манифест с определенной api-версией, делаем так:
> (на примере куба 1.15, где еще есть поддержка extensions/v1beta1)

```bash
# хотим из extensions
kubectl get deployments.extensions ext -o yaml
# хотим из extensions, да еще чтоб именно v1beta1
kubectl get deployments.v1beta1.extensions ext -o yaml
# хотим из apps
kubectl get deployments.apps ext -o yaml
# хотим из apps, именно v1
kubectl get deployments.v1.apps ext -o yaml
```

Примечательно, что если сделать просто `kubectl get deployments ext -o yaml`, то мы получим манифест из `extensions/v1beta1`, и пофиг, что изначально применяли `apps/v1`, например. [Тут](https://github.com/kubernetes/kubernetes/issues/58131#issuecomment-356823588) подробнее.

## Проверяем на практике

1. Накатываем куб 1.15

   ```bash
   minikube start --kubernetes-version=v1.15.12
   ```

1. Берем 2 манифеста для `deployments` разных версий. Обратите внимание на отсутствие `spec.selector` в `extensions/v1beta1`

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     creationTimestamp: null
     labels:
       run: apps
     name: apps
   spec:
     replicas: 1
     selector:
       matchLabels:
         run: apps
     template:
       metadata:
         creationTimestamp: null
         labels:
           run: apps
       spec:
         containers:
         - image: nginx
           name: apps
   ---
   apiVersion: extensions/v1beta1
   kind: Deployment
   metadata:
     labels:
       run: ext
     name: ext
   spec:
     replicas: 1
     template:
       metadata:
         creationTimestamp: null
         labels:
           run: ext
       spec:
         containers:
         - image: nginx
           name: ext
   ```

1. Применяем

   ```bash
   ❯ minikube kubectl -- apply -f .
   deployment.apps/apps created
   deployment.extensions/ext created
   ```

1. Убеждаемся, что ресурсы `apps` и `ext` есть и в `extensions/v1beta1`, и в `apps/v1`.

   * Сложный путь - curl:

   ```bash
   ❯ minikube kubectl -- proxy
   Starting to serve on 127.0.0.1:8001
   ❯ curl -s 127.0.0.1:8001/apis/extensions/v1beta1/namespaces/default/deployments/apps | yq . --yaml-output
   kind: Deployment
   apiVersion: extensions/v1beta1
   # ...
   ❯ curl -s 127.0.0.1:8001/apis/extensions/v1beta1/namespaces/default/deployments/ext | yq . --yaml-output
   kind: Deployment
   apiVersion: extensions/v1beta1
   # ...
   ❯ curl -s 127.0.0.1:8001/apis/apps/v1/namespaces/default/deployments/apps | yq . --yaml-output
   kind: Deployment
   apiVersion: apps/v1
   # ...
   ❯ curl -s 127.0.0.1:8001/apis/apps/v1/namespaces/default/deployments/ext | yq . --yaml-output
   kind: Deployment
   apiVersion: apps/v1
   # ...
   ```

   * Либо через kubectl:

   ```bash
   # хотим из extensions
   kubectl get deployments.extensions ext -o yaml
   # хотим из extensions, да еще чтоб именно v1beta1
   kubectl get deployments.v1beta1.extensions ext -o yaml
   # хотим из apps
   kubectl get deployments.apps ext -o yaml
   # хотим из apps, именно v1
   kubectl get deployments.v1.apps ext -o yaml
   ```

## Бонус: kubectl explain

Если сделать `kubectl explain deployment`, то (сюрприз!) мы получим описание из `extensions/v1beta1`. Т.к. `kubectl explain` [работает так](https://github.com/kubernetes/kubernetes/issues/73062) же, как `kubectl get`:

Если нужна конкретная версия, используем флаг:

```bash
minikube kubectl -- explain deployment.spec --api-version apps/v1
minikube kubectl -- explain deployment.spec --api-version extensions/v1beta1
```

## Обсудим?

{{< tweet 1330238026262458371 >}}
