# GitOps Kafka Topics

Este repositorio contiene un Helm chart para gestionar topics de Kafka de forma declarativa usando GitOps con ArgoCD y el operador Strimzi.

## ¿Qué hace este repositorio?

Este repositorio permite gestionar múltiples topics de Kafka de manera centralizada y versionada mediante:

- **Helm Chart**: Chart de Helm que genera recursos `KafkaTopic` de Strimzi
- **GitOps**: Gestión declarativa de la configuración de topics mediante control de versiones
- **ArgoCD**: Sincronización automática entre el repositorio Git y el cluster de Kubernetes

### Características

- Gestión centralizada de múltiples topics de Kafka
- Configuración declarativa mediante valores de Helm (`values.yaml`)
- Soporte para configuraciones personalizadas por topic (particiones, réplicas, retención, etc.)
- Etiquetas globales y por topic
- Integración con el operador Strimzi de Kafka

## Estructura del Repositorio

```
gitops-kafka/
├── Chart.yaml              # Metadatos del Helm chart
├── values.yaml             # Configuración de los topics de Kafka
├── templates/
│   ├── _helpers.tpl        # Plantillas auxiliares de Helm
│   └── kafkatopics.yaml    # Plantilla que genera los recursos KafkaTopic
└── README.md               # Esta documentación
```

## Configuración de Topics

Los topics se definen en el archivo `values.yaml`. Cada topic puede tener las siguientes propiedades:

- `name`: Nombre del topic
- `partitions`: Número de particiones
- `replicas`: Número de réplicas (puede ser `null` para usar el valor por defecto del cluster)
- `config`: Configuraciones adicionales del topic (retention.ms, segment.bytes, etc.)
- `kafkaCluster`: (Opcional) Nombre del cluster Kafka (por defecto usa `defaultKafkaCluster`)
- `labels`: (Opcional) Etiquetas específicas del topic
- `annotations`: (Opcional) Anotaciones específicas del topic

### Ejemplo de configuración

```yaml
topics:
  - name: "mi-topic-dev"
    partitions: 10
    replicas: 3
    config:
      retention.ms: 604800000
      segment.bytes: 1073741824
```

## Crear la Aplicación ArgoCD

Para desplegar este Helm chart en tu cluster de Kubernetes usando ArgoCD, necesitas crear un recurso `Application` de ArgoCD.

### YAML de la Aplicación ArgoCD

Crea un archivo `argocd-application.yaml` con el siguiente contenido:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    argocd.argoproj.io/sync-options: 'Prune=false,ApplyOutOfSyncOnly=true'
  name: gitops-kafka
  namespace: openshift-gitops
  labels:
    app: gitops-kafka
    app.kubernetes.io/component: gitops-kafka
    app.kubernetes.io/instance: collections-apps-migration
    app.kubernetes.io/name: gitops-kafka
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  destination:
    namespace: prod-broker
    server: https://kubernetes.default.svc
  source:
    path: .
    repoURL: 'https://10.164.30.14:38001/tmve/devops/openshift/gitops-kafka.git'
    targetRevision: production
    helm:
      valueFiles:
        - values.yaml
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ApplyOutOfSyncOnly=true
      - Prune=true
      - SkipDryRunOnMissingResource=true
```

### Parámetros importantes a ajustar

- `metadata.name`: Nombre de la aplicación en ArgoCD
- `metadata.namespace`: Namespace donde está instalado ArgoCD (openshift-gitops)
- `spec.source.repoURL`: URL del repositorio Git
- `spec.source.targetRevision`: Rama o tag del repositorio a usar (production)
- `spec.destination.namespace`: Namespace donde se crearán los recursos KafkaTopic
- `spec.destination.server`: URL del servidor del cluster de Kubernetes (https://kubernetes.default.svc para el cluster local)
- `spec.syncPolicy.automated`: Configuración de sincronización automática

### Aplicar la Aplicación ArgoCD

Una vez que tengas el YAML configurado, puedes aplicarlo usando la CLI de OpenShift:

```bash
oc create -f argocd-application.yaml -n openshift-gitops
```


### Verificar el despliegue

Después de crear la aplicación, puedes verificar su estado:

```bash
# Ver el estado de la aplicación
argocd app get gitops-kafka

# Ver los recursos desplegados
kubectl get kafkatopics -n prod-broker

# Ver los logs de sincronización
argocd app logs gitops-kafka
```

## Modificar Topics

Para agregar, modificar o eliminar topics:

1. Edita el archivo `values.yaml` en el repositorio
2. Haz commit y push de los cambios a la rama `production`
3. ArgoCD detectará los cambios automáticamente y los aplicará según las opciones de sincronización configuradas
4. O sincroniza manualmente desde la UI de ArgoCD o CLI:

```bash
argocd app sync gitops-kafka
```

## Requisitos Previos

- Cluster de Kubernetes con el operador Strimzi instalado
- ArgoCD instalado y configurado en el cluster
- Acceso al repositorio Git configurado en ArgoCD
- Permisos para crear recursos `KafkaTopic` en el namespace destino (`prod-broker`)

## Soporte

Para más información sobre:
- **Strimzi Kafka Operator**: https://strimzi.io/
- **ArgoCD**: https://argo-cd.readthedocs.io/
- **Helm**: https://helm.sh/docs/
