# OpenEBS Helm Operator

## Introduction

- This is  a helm operator for openebs
- The content of values.yaml will be placed as spec items in `OpenEBSInstallTemplate` CR 
- The operator watches all namespaces. There is no need for helm client/tillerto be installed.
  (the helm operator imports code from helm project)
- Changes to spec of the CR will behave similar to values override via helm
- All OpenEBS resources have ownerReference of `OpenEBSInstallTemplate` & are deleted upon removal of CR

Example of OpenEBSInstallTemplate: 

  ```yaml
  apiVersion: openebs.io/v1alpha1
kind: OpenEBSInstallTemplate
metadata:
  name: oebs 
  namespace: openebs
spec:
  # Default values copied from <project_dir>/helm-charts/openebs/values.yaml
  
  # Default values for openebs.
  # This is a YAML-formatted file.
  # Declare variables to be passed into your templates.
  
  rbac:
    # Specifies whether RBAC resources should be created
    create: true
  
  serviceAccount:
    create: true
    name:

  release:
    # "openebs.io/version" label for control plane components
    version: "1.0.0"
  
  image:
    pullPolicy: IfNotPresent
  
  apiserver:
    image: "quay.io/openebs/m-apiserver"
    imageTag: "1.0.0"
    replicas: 1
    ports:
      externalPort: 5656
      internalPort: 5656
    sparse:
      enabled: "false"
    nodeSelector: {}
    tolerations: []
    affinity: {}
    healthCheck:
      initialDelaySeconds: 50
      periodSeconds: 60
  
  provisioner:
    image: "quay.io/openebs/openebs-k8s-provisioner"
    imageTag: "1.0.0"
    replicas: 1
    nodeSelector: {}
    tolerations: []
    affinity: {}
    healthCheck:
      initialDelaySeconds: 30
      periodSeconds: 60

  localprovisioner:
    image: "quay.io/openebs/provisioner-localpv"
    imageTag: "1.0.0"
    helperImage: "quay.io/openebs/openebs-tools"
    helperImageTag: "3.8"
    replicas: 1
    basePath: "/var/openebs/local"
    nodeSelector: {}
    tolerations: []
    affinity: {}
    healthCheck:
      initialDelaySeconds: 30
      periodSeconds: 60
  
  snapshotOperator:
    controller:
      image: "quay.io/openebs/snapshot-controller"
      imageTag: "1.0.0"
    provisioner:
      image: "quay.io/openebs/snapshot-provisioner"
      imageTag: "1.0.0"
    replicas: 1
    upgradeStrategy: "Recreate"
    nodeSelector: {}
    tolerations: []
    affinity: {}
    healthCheck:
      initialDelaySeconds: 30
      periodSeconds: 60
  
  ndm:
    image: "quay.io/openebs/node-disk-manager-amd64"
    imageTag: "v0.4.0"
    sparse:
      path: "/var/openebs/sparse"
      size: "10737418240"
      count: "1"
    filters:
      excludeVendors: "CLOUDBYT,OpenEBS"
      includePaths: ""
      excludePaths: "loop,fd0,sr0,/dev/ram,/dev/dm-,/dev/md"
    nodeSelector: {}
    healthCheck:
      initialDelaySeconds: 30
      periodSeconds: 60

  ndmOperator:
    image: "quay.io/openebs/node-disk-operator-amd64"
    imageTag: "v0.4.0"
    replicas: 1
    upgradeStrategy: Recreate
    nodeSelector: {}
    tolerations: []
    readinessCheck:
      initialDelaySeconds: 4
      periodSeconds: 10
      failureThreshold: 1
    cleanupImage: "quay.io/openebs/linux-utils"
    cleanupImageTag: "3.9"


  webhook:
    image: "quay.io/openebs/admission-server"
    imageTag: "1.0.0"
    generateTLS: true
    replicas: 1
    nodeSelector: {}
    tolerations: []
    affinity: {}
  
  jiva:
    image: "quay.io/openebs/jiva"
    imageTag: "1.0.0"
    replicas: 3
  
  cstor:
    pool:
      image: "quay.io/openebs/cstor-pool"
      imageTag: "1.0.0"
    poolMgmt:
      image: "quay.io/openebs/cstor-pool-mgmt"
      imageTag: "1.0.0"
    target:
      image: "quay.io/openebs/cstor-istgt"
      imageTag: "1.0.0"
    volumeMgmt:
      image: "quay.io/openebs/cstor-volume-mgmt"
      imageTag: "1.0.0"
  
  policies:
    monitoring:
      enabled: true
      image: "quay.io/openebs/m-exporter"
      imageTag: "1.0.0"
  
  analytics:
    enabled: true
    # Specify in hours the duration after which a ping event needs to be sent.
    pingInterval: "24h"
  ```

## Limitations 

- Release names are auto-generated based on CR UID. Impact is character-length will exceed limit if CR name
  is longer. 
  - Workaround: Give shorter nams: 
  - ER: https://github.com/operator-framework/operator-sdk/issues/1094

- Release namespace doesn't have a field in the spec. (need more exploration) 
  - Workaround: Create ns "openebs" or any desired before applying the CR

- Uninstall of a failed helm release (say, wrong spec) will be stuck. 
  - Workaround: finalizer has to be removed manually
  - Snippet: 
  ```
    status:
    conditions:
    - lastTransitionTime: "2019-06-28T02:12:46Z"
      status: "True"
      type: Initialized
    - lastTransitionTime: "2019-06-28T02:12:46Z"
      message: 'failed to get release history: release: "example-openebsinstalltemplate-3c4h3yqweylajqktnjs04e135"
        not found'
      reason: UninstallError
      status: "True"
      type: ReleaseFailed
   ```

