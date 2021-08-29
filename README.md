# Cloud Native Toolkit - GitOps Production Deployment Guide

The GitOps concept origifnated from [Weaveworks](https://www.weave.works/) back in 2017 and the goal was to automate the operations of a Kubernetes (K8s) system using a model external to the system as the source of truth ([History of GitOps](https://www.weave.works/blog/the-history-of-gitops)).   

This repository provides our opinionated point of view on how `GitOps` can be used to manage the infrastructure, services and application layers of K8s based systems.  It takes into account the various personas interacting with the system and accounts for separation of duties.  The instructions and examples are focused around the [Red Hat OpenShift](https://cloud.redhat.com/learn/what-is-openshift) platform and [IBM Cloud Paks](https://www.ibm.com/cloud/paks).

The reference architecture for this GitOps workflow can be found [here](https://cloudnativetoolkit.dev/adopting/use-cases/gitops/gitops-toolkit/).  


## Table of contents
- [Pre-requisites](#pre-requisites)
    - [Red Hat OpenShift cluster](#red-hat-openshift-cluster)
    - [CLI tools](#cli-tools)
    - [IBM Entitlement Key](#ibm-entitlement-key)    
- [Setup git repositories](#setup-git-repositories)
    - [Custom GitHub Orgnaization](#custom-github-organization)
    - [Create base gitops repositories](#create-base-gitops-repositories)
- [Select resources to deploy](#select-resources-to-deploy)
    - [Infrastructure layer](#infrastructure-layer)
    - [Services layer](#services-layer)
    - [Application layer](#application-layer)
- [Deploy the OpenShift GitOps Operator](#install-openshift-gitops-operator)
- [Bootstrap the OpenShift cluster](#bootstrap-the-openshift-cluster)
- [Troubleshooting](doc/troubleshooting.md)
- [FAQ](doc/faq.md)


## Pre-requisites

### Red Hat OpenShift cluster 
- An OpenShift v4.7+ cluster is required.  

### CLI tools
- Install the OpenShift CLI oc (version 4.7+) .  The binary can be downloaded from the Help menu from the OpenShift Console. 
    <details>
    <summary>Download oc cli</summary>

    ![oc cli](doc/images/oc-cli.png)
    </details>
- Log in from a terminal window.
    ```bash
    oc login --token=<token> --server=<server>
    ```

### IBM Entitlement Key
- The `IBM Entitlement Key` is required to pull IBM Cloud Pak specific container images from the IBM Entitled Registry.  To get an entitlement key, 

    1. Log in to [MyIBM Container Software Library](https://myibm.ibm.com/products-services/containerlibrary) with an IBMid and password associated with the entitled software.  
    2. Select the **View library** option to verify your entitlement(s). 
    3. Select the **Get entitlement key** to retrieve the key.

- A **Secret** containing the entitlement key is created in the `tools` namespace. 

    ```bash
    oc new-project tools || true
    oc create secret docker-registry ibm-entitlement-key -n tools \
    --docker-username=cp \
    --docker-password="<entitlement_key>" \
    --docker-server=cp.icr.io
    ```

## Setup git repositories
- The following set of Git repositories will be used for our GitOps workflow.  
    - Main GitOps repository ([https://github.com/cloud-native-toolkit/multi-tenancy-gitops](https://github.com/cloud-native-toolkit/multi-tenancy-gitops)): This repository contains all the ArgoCD Applications for  the `infrastructure`, `services` and `application` layers.  Each ArgoCD Application will reference a specific K8s resource (yaml resides in a separate git repository), contain the configuration of the K8s resource, and determine where it will be deployed into the cluster.  
    - Infrastructure GitOps repository ([https://github.com/cloud-native-toolkit/multi-tenancy-gitops-infra](https://github.com/cloud-native-toolkit/multi-tenancy-gitops-infra)): Contains the YAMLs for cluster-wide and/or infrastructure related K8s resources managed by a cluster administrator.  This would include `namespaces`, `clusterroles`, `clusterrolebindings`, `machinesets` to name a few.
    - Services GitOps repostiory ([https://github.com/cloud-native-toolkit/multi-tenancy-gitops-services](https://github.com/cloud-native-toolkit/multi-tenancy-gitops-services)): Contains the YAMLs for K8s resources which will be used by the `application` layer.  This could include `subscriptions` for Operators, YAMLs of custom resources provided, or Helm Charts for tools provided by a third party.  These resource would usually be managed by the Administrator(s) and/or a DevOps team supporting application developers.  

### Tasks: 
1. Create a new GitHub Organization using instructions from this [GitHub documentation](https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/creating-a-new-organization-from-scratch). 
2. From each template repository, click the `Use this template` button and create a copy of the repository in the new GitHub Organization.
    ![Create repository from a template](doc/images/git-repo-template-button.png)
3. Run script to replace the git url and branch to your git organization where you created the git repositories
    ```bash
    GIT_ORG=<GitHub Organization> GIT_BRANCH=master ./scripts/set-git-source.sh
    ```


## Select resources to deploy
- Select the K8s resources to deploy in the [infrastructure](0-bootstrap/single-cluster/1-infra/kustomization.yaml) and [services](0-bootstrap/single-cluster/2-services/kustomization.yaml) layers.
- Recipes are available and additional ones will be made available in the **doc** directory. 
    - [ACE recipe](doc/ace-recipe.md)
    - [MQ recipe](doc/mq-recipe.md)
    - [Process Mining recipe](doc/process-mining-recipe.md)

### Tasks: 
1. Select a profile and delete the others from the `0-bootstrap` directory.  If this is your first usage of the gitops workflow, Use the `single-cluster`.
    ```bash
    GITOPS_PROFILE="0-bootstrap/single-cluster"
    ```
2. Review the `Infrastructure` layer [kustomization.yaml](0-bootstrap/single-cluster/1-infra/kustomization.yaml) and un-comment the resources to deploy.  
3. Review the `Services` layer [kustomization.yaml](0-bootstrap/single-cluster/2-services/kustomization.yaml) and un-comment the resources to deploy.  
4. Commit and push changes to your git repository
    ```bash
    git add .
    git commit -m "intial boostrap setup"
    git push origin
    ```


## Install and configure OpenShift GitOps 
- [Red Hat OpenShift GitOps](https://docs.openshift.com/container-platform/4.7/cicd/gitops/understanding-openshift-gitops.html) uses [Argo CD](https://argoproj.github.io/argo-cd/), an open-source declarative tool, to maintain and reconcile cluster resources. 

### Tasks: 
1. Install the OpenShift GitOps Operator, create a `ClusterRole` and deploy a default instance of ArgoCD.  
    ```bash
    oc apply -f setup/ocp47/
    while ! oc wait crd applications.argoproj.io --timeout=-1s --for=condition=Established  2>/dev/null; do sleep 30; done
    while ! oc wait pod --timeout=-1s --for=condition=Ready -l '!job-name' -n openshift-gitops > /dev/null; do sleep 30; done
    ```
1. Delete the default ArgoCD instance
    ```bash
    oc delete gitopsservice cluster -n openshift-gitops || true
    oc delete argocd openshift-gitops -n openshift-gitops || true
    ```
1. Create a custom ArgoCD instance with custom checks
    ```bash
    oc apply -f gitops-0-bootstrap/setup/ocp47/argocd-instance/ -n openshift-gitops
    while ! oc wait pod --timeout=-1s --for=condition=ContainersReady -l app.kubernetes.io/name=openshift-gitops-cntk-server -n openshift-gitops > /dev/null; do sleep 30; done
    ```


## Bootstrap the OpenShift cluster 
- The bootstrap YAML follows the [app of apps pattern](https://argoproj.github.io/argo-cd/operator-manual/cluster-bootstrapping/#app-of-apps-pattern) to deploy the `infrastructure` and `services` resources declared in the kustomization YAMLs. 

### Tasks: 
1. Apply ArgoCD Bootstrap Application
    ```bash
    oc apply -f ${GITOPS_PROFILE}/bootstrap.yaml
    ```
2. Retrieve the ArgoCD/GitOps URL and admin password:
    ```bash
    oc get route -n openshift-gitops openshift-gitops-cntk-server -o template --template='https://{{.spec.host}}'
    oc extract secrets/openshift-gitops-cntk-cluster --keys=admin.password -n openshift-gitops --to=-
    ```
3. If an IBM Cloud Pak is installed, retrieve the console URL and admin password.
    ```bash
    oc get route -n tools integration-navigator-pn -o template --template='https://{{.spec.host}}'
    oc extract -n ibm-common-services secrets/platform-auth-idp-credentials --keys=admin_username,admin_password --to=-
    ```