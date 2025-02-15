---
title: Desplegar a Google Kubernetes Engine
intro: Puedes desplegar hacia Google Kubernetes Engine como parte de tus flujos de trabajo de despliegue continuo (DC).
redirect_from:
  - /actions/guides/deploying-to-google-kubernetes-engine
  - /actions/deployment/deploying-to-google-kubernetes-engine
versions:
  fpt: '*'
  ghes: '*'
  ghae: '*'
  ghec: '*'
type: tutorial
topics:
  - CD
  - Containers
  - Google Kubernetes Engine
shortTitle: Deploy to Google Kubernetes Engine
ms.openlocfilehash: 0572a326d52654b256e0e1ad7fe9c9c4e9d547ac
ms.sourcegitcommit: fcf3546b7cc208155fb8acdf68b81be28afc3d2d
ms.translationtype: HT
ms.contentlocale: es-ES
ms.lasthandoff: 09/11/2022
ms.locfileid: '147409551'
---
{% data reusables.actions.enterprise-beta %} {% data reusables.actions.enterprise-github-hosted-runners %}

## Introducción

Esta guía te explica cómo utilizar las {% data variables.product.prodname_actions %} para crear una aplicación contenedorizada, subirla al Registro de Contenedores de Google (GCR) y desplegarla en Google Kubernetes Engine (GKE) cuando haya una subida a la rama `main`.

GKE es un agrupamiento administrado de Kubernetes de Google Cloud que puede hospedar tus cargas de trabajo contenerizadas en la nube o en tu propio centro de datos. Para obtener más información, vea [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine).

{% ifversion fpt or ghec or ghae-issue-4856 or ghes > 3.4 %}

{% note %}

**Nota**: {% data reusables.actions.about-oidc-short-overview %}

{% endnote %}

{% endif %}

## Prerrequisitos

Antes de proceder con la creación del flujo de trabajo, necesitarás completar los siguientes pasos par tu proyecto de Kubernetes. Esta guía asume que la raíz de su proyecto ya tiene un elemento `Dockerfile` y un archivo de configuración para la implementación de Kubernetes. Para obtener un ejemplo, vea [google-github-actions](https://github.com/google-github-actions/setup-gcloud/tree/master/example-workflows/gke).

### Crear un agrupamiento de GKE

Para crear un clúster de GKE, primero necesitará autenticarse mediante la CLI de `gcloud`. Para obtener más información sobre este paso, consulta los siguientes artículos:
- [`gcloud auth login`](https://cloud.google.com/sdk/gcloud/reference/auth/login)
- [CLI de `gcloud`](https://cloud.google.com/sdk/gcloud/reference)
- [CLI de `gcloud` y SDK de Cloud](https://cloud.google.com/sdk/gcloud#the_gcloud_cli_and_cloud_sdk)

Por ejemplo:

{% raw %}
```bash{:copy}
$ gcloud container clusters create $GKE_CLUSTER \
    --project=$GKE_PROJECT \
    --zone=$GKE_ZONE
```
{% endraw %}

### Habilitar las API

Habilita las API de Kubernetes Engine y del Registro de Contenedor. Por ejemplo:

{% raw %}
```bash{:copy}
$ gcloud services enable \
    containerregistry.googleapis.com \
    container.googleapis.com
```
{% endraw %}

### Configurar una cuenta de servicio y almacenar sus crendenciales

Este procedimiento demuestra cómo crear la cuenta de servicio para tu integración con GKE. Explica cómo crear la cuenta, agregarle roles, recuperar sus llaves y almacenarlas como un secreto de repositorio cifrado y codificado en base 64 denominado `GKE_SA_KEY`.

1. Cree una cuenta de servicio: {% raw %}
  ```
  $ gcloud iam service-accounts create $SA_NAME
  ```
  {% endraw %}
1. Recupere la dirección de correo electrónico en la cuenta de servicio que acaba de crear: {% raw %}
  ```
  $ gcloud iam service-accounts list
  ```
  {% endraw %}
1. Agrega roles a la cuenta de servicio. Nota: Aplica roles más restrictivos para que se acoplen a tus requisitos.
  {% raw %}
  ```
  $ gcloud projects add-iam-policy-binding $GKE_PROJECT \
    --member=serviceAccount:$SA_EMAIL \
    --role=roles/container.admin
  $ gcloud projects add-iam-policy-binding $GKE_PROJECT \
    --member=serviceAccount:$SA_EMAIL \
    --role=roles/storage.admin
  $ gcloud projects add-iam-policy-binding $GKE_PROJECT \
    --member=serviceAccount:$SA_EMAIL \
    --role=roles/container.clusterViewer
  ```
  {% endraw %}
1. Descargue el archivo de clave JSON para la cuenta de servicio: {% raw %}
  ```
  $ gcloud iam service-accounts keys create key.json --iam-account=$SA_EMAIL
  ```
  {% endraw %}
1. Almacene la clave de la cuenta de servicio como un secreto llamado `GKE_SA_KEY`: {% raw %}
  ```
  $ export GKE_SA_KEY=$(cat key.json | base64)
  ```
  {% endraw %} Para obtener más información sobre cómo almacenar un secreto, vea "[Secretos cifrados](/actions/security-guides/encrypted-secrets)".

### Almacenar el nombre de tu proyecto

Almacene el nombre del proyecto como un secreto denominado `GKE_PROJECT`. Para obtener más información sobre cómo almacenar un secreto, vea "[Secretos cifrados](/actions/security-guides/encrypted-secrets)".

### (Opcional) Configurar kustomize
Kustomize es una herramietna opcional que se utiliza para administrar las especificaciones YAML. Después de crear un archivo `kustomization`, el siguiente flujo de trabajo puede utilizarse para configurar campos de la imagen dinámicamente y agregar el resultado en `kubectl`. Para obtener más información, vea [Uso de Kustomize](https://github.com/kubernetes-sigs/kustomize#usage).

### (Opcional) Configurar un ambiente de despliegue

{% data reusables.actions.about-environments %}

## Crear un flujo de trabajo

Una vez que hayas completado los prerequisitos, puedes proceder con la creación del flujo de trabajo.

El siguiente flujo de trabajo demuestra cómo crear una imagen de contenedor y cómo subirla a GCR. Después, usa las herramientas de Kubernetes (como `kubectl` y `kustomize`) para extraer la imagen en la implementación del clúster.

En la clave `env`, cambie el valor de `GKE_CLUSTER` por el nombre del clúster, `GKE_ZONE` por la zona del clúster, `DEPLOYMENT_NAME` por el nombre de la implementación y `IMAGE` por el nombre de la imagen.

{% data reusables.actions.delete-env-key %}

```yaml{:copy}
{% data reusables.actions.actions-not-certified-by-github-comment %}

{% data reusables.actions.actions-use-sha-pinning-comment %}

name: Build and Deploy to GKE

on:
  push:
    branches:
      - main

env:
  PROJECT_ID: {% raw %}${{ secrets.GKE_PROJECT }}{% endraw %}
  GKE_CLUSTER: cluster-1    # Add your cluster name here.
  GKE_ZONE: us-central1-c   # Add your cluster zone here.
  DEPLOYMENT_NAME: gke-test # Add your deployment name here.
  IMAGE: static-site

jobs:
  setup-build-publish-deploy:
    name: Setup, Build, Publish, and Deploy
    runs-on: ubuntu-latest
    environment: production

    steps:
    - name: Checkout
      uses: {% data reusables.actions.action-checkout %}

    # Setup gcloud CLI
    - uses: google-github-actions/setup-gcloud@94337306dda8180d967a56932ceb4ddcf01edae7
      with:
        service_account_key: {% raw %}${{ secrets.GKE_SA_KEY }}{% endraw %}
        project_id: {% raw %}${{ secrets.GKE_PROJECT }}{% endraw %}

    # Configure Docker to use the gcloud command-line tool as a credential
    # helper for authentication
    - run: |-
        gcloud --quiet auth configure-docker

    # Get the GKE credentials so we can deploy to the cluster
    - uses: google-github-actions/get-gke-credentials@fb08709ba27618c31c09e014e1d8364b02e5042e
      with:
        cluster_name: {% raw %}${{ env.GKE_CLUSTER }}{% endraw %}
        location: {% raw %}${{ env.GKE_ZONE }}{% endraw %}
        credentials: {% raw %}${{ secrets.GKE_SA_KEY }}{% endraw %}

    # Build the Docker image
    - name: Build
      run: |-
        docker build \
          --tag "gcr.io/$PROJECT_ID/$IMAGE:$GITHUB_SHA" \
          --build-arg GITHUB_SHA="$GITHUB_SHA" \
          --build-arg GITHUB_REF="$GITHUB_REF" \
          .

    # Push the Docker image to Google Container Registry
    - name: Publish
      run: |-
        docker push "gcr.io/$PROJECT_ID/$IMAGE:$GITHUB_SHA"

    # Set up kustomize
    - name: Set up Kustomize
      run: |-
        curl -sfLo kustomize https://github.com/kubernetes-sigs/kustomize/releases/download/v3.1.0/kustomize_3.1.0_linux_amd64
        chmod u+x ./kustomize

    # Deploy the Docker image to the GKE cluster
    - name: Deploy
      run: |-
        ./kustomize edit set image gcr.io/PROJECT_ID/IMAGE:TAG=gcr.io/$PROJECT_ID/$IMAGE:$GITHUB_SHA
        ./kustomize build . | kubectl apply -f -
        kubectl rollout status deployment/$DEPLOYMENT_NAME
        kubectl get services -o wide
```

## Recursos adicionales

Para obtener más información sobre las herramientas que se utilizan en estos ejemplos, consulta la siguiente documentación:

* Para obtener el flujo de trabajo de inicio completo, vea el [flujo de trabajo "Compilar e implementar en GKE](https://github.com/actions/starter-workflows/blob/main/deployments/google.yml)".
* Para obtener más flujos de trabajo de inicio y código complementario, vea los [flujos de trabajo de ejemplo {% data variables.product.prodname_actions %}](https://github.com/google-github-actions/setup-gcloud/tree/master/example-workflows/) de Google.
* Motor de personalización YAML de Kubernetes: [Kustomize](https://kustomize.io/).
* "[Implementación de una aplicación web contenedorizada](https://cloud.google.com/kubernetes-engine/docs/tutorials/hello-app)" en la documentación de Google Kubernetes Engine.
