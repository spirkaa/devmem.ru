---
title: "Увеличение тома Longhorn при использовании операторов Kubernetes"
date: 2022-03-31T16:42:00+03:00
toc: true
categories:
  - Kubernetes
tags:
  - Longhorn
---

## Проблемы

* Longhorn не поддерживает изменение размера томов на лету, поэтому том должен быть отключен (Detached), а следовательно под, использующий том, должен быть удален. А перед этим нужно удалить StatefulSet или Deployment.
* Операторы Kubernetes постоянно отслеживают и исправляют состояние, например, создают удаленные вручную StatefulSet или Deployment.
* StatefulSet не поддерживает изменение volumeClaimTemplates.

## Решение для StatefulSet на примере kube-prometheus-stack (Prometheus Operator) и ArgoCD

1. Отключить или настроить все операторы, выполняющие согласование (reconciliation) объектов Kubernetes, с которыми связан целевой том:
    1. Отключить автоматическую синхронизацию и исправление приложения в [ArgoCD](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/): в коде удалить блок `Application.spec.syncPolicy` и отправить в репозиторий, дождаться синхронизации
    1. Уменьшить реплики Prometheus Operator до 0

        `kubectl -n observability scale deployment kps-operator --replicas 0`

1. Удалить StatefulSet, но оставить зависимые объекты (Pod, PersistentVolumeClaim)

    `kubectl -n observability delete statefulsets.apps prometheus-kps-prometheus --cascade=orphan`

1. Удалить Pod, тем самым переведя том Longhorn в состояние Detached

    `kubectl -n observability delete pod prometheus-kps-prometheus-0`

1. Отредактировать PersistentVolumeClaim, указав новый размер в `spec.resources.requests.storage`

    `kubectl -n observability edit pvc prometheus-kps-prometheus-db-prometheus-kps-prometheus-0`

1. Дождаться применения изменений драйвером Longhorn

    `kubectl -n observability describe pvc prometheus-kps-prometheus-db-prometheus-kps-prometheus-0`

    ```console
      Events:
        Type     Reason                  Age   From                                 Message
        ----     ------                  ----  ----                                 -------
        Warning  ExternalExpanding       34s   volume_expand                        Ignoring the PVC: didn't find a plugin capable of expanding the volume; waiting for an external controller to process this PVC.
        Normal   Resizing                34s   external-resizer driver.longhorn.io  External resizer is resizing volume pvc-a4de8e0e-c664-4f60-be90-ddb0207eae7f
        Normal   VolumeResizeSuccessful  22s   external-resizer driver.longhorn.io  Resize volume succeeded
    ```

1. Задать в коде новый размер PersistentVolumeClaim в `prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage`

1. Вернуть в коде `Application.spec.syncPolicy`, отправить в репозиторий, запустить синхронизацию в ArgoCD

1. Вернуть исходное количество реплик оператора из п.1, в результате чего оператор пересоздаст удаленный ранее StatefulSet

    `kubectl -n observability scale deployment kps-operator --replicas 1`

## Решение для Deployment

1. Удалить в коде `Application.spec.syncPolicy`, отправить в репозиторий, дождаться синхронизации ArgoCD
1. Удалить Deployment
1. Задать в коде новый размер `PersistentVolumeClaim`, отправить в репозиторий, запустить синхронизацию только ресурса PersistentVolumeClaim в ArgoCD, а не всего приложения
1. Дождаться применения изменений драйвером Longhorn
1. Вернуть в коде `Application.spec.syncPolicy`, отправить в репозиторий, запустить синхронизацию в ArgoCD

## Ссылки по теме

* <https://longhorn.io/docs/1.3.0/volumes-and-nodes/expansion/>
* <https://itnext.io/resizing-statefulset-persistent-volumes-with-zero-downtime-916ebc65b1d4>
