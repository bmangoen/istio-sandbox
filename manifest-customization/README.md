# Sail Operator Manifest Customization

## Overview

Today, users/personas can not customize any field - which is not a helm variable - of resources managed by sail-operator.  

### Annotation sailoperator.io/ignore

Setting the annotation `sailoperator.io/ignore: "true"` to resource skips the reconciliation when there is any modification to its `spec`.  

> [!NOTE]  
> This method allows modifying all the fields of the resource and users may want to specify an individual field to be ignored.


1. Clone the Manifest Customization sail-operator POC branch  

    ```bash
    git clone -b feat/ignore-update-when-annotation https://github.com/bmangoen/sail-operator.git
    ```

1. Install the feature specific sail-operator build

    ```bash
    make -e HUB=quay.io/bmangoen TAG=ignore-predicate deploy
    ```

1. Create `istio-system` namespace  

    ```bash
    kubectl create ns istio-system
    ```

1. Create `Istio` resource  

    ```bash
    kubectl apply -f chart/samples/istio-sample.yaml -n istio-system
    ```

1. Save the original MutatingWebhookConfiguration spec

    ```bash
    kubectl -n istio-system describe mutatingwebhookconfigurations.admissionregistration.k8s.io istio-sidecar-injector > /tmp/istio-mutatingwebhook-ori.txt
    ```


1. Add `sailoperator.io/ignore: "true"` annotation to disable the reconciliation to the resource  

    ```bash
    kubectl -n istio-system patch mutatingwebhookconfigurations.admissionregistration.k8s.io istio-sidecar-injector --patch '{"metadata": {"annotations": {"sailoperator.io/ignore": "true"}}}'
    ```

1. Simulate a change to this resource  

    ```bash
    kubectl -n istio-system patch mutatingwebhookconfigurations.admissionregistration.k8s.io istio-sidecar-injector --patch '{"metadata": {"labels": {"app": "sidecar-injector-test"}}}'
    ```

1. Save the current MutatingWebhookConfiguration spec output

    ```bash
    kubectl -n istio-system describe mutatingwebhookconfigurations.admissionregistration.k8s.io istio-sidecar-injector > /tmp/istio-mutatingwebhook-customized.txt
    ```

1. Diff the two files result  

    ```bash
    diff /tmp/istio-mutatingwebhook-ori.txt /tmp/istio-mutatingwebhook-customized.txt 
    
    3c3
    < Labels:       app=sidecar-injector
    ---
    > Labels:       app=sidecar-injector-test
    15a16
    >               sailoperator.io/ignore: true
    32d32
    ```

## Usage

1. Clone the Manifest Customization sail-operator POC branch  

    ```bash
    git clone -b poc/manifest-custumization https://github.com/bmangoen/sail-operator.git
    ```

1. Install sail-operator

    ```bash
    make -e HUB=quay.io/bmangoen deploy
    ```

1. Create `istio-system` namespace  

    ```bash
    kubectl create ns istio-system
    ```

1. Create `Istio` and `IstioCNI` resources  

    ```bash
    kubectl apply -f chart/samples/istio-sample.yaml -n istio-system
    kubectl apply -f chart/samples/istiocni-sample.yaml -n istio-system
    ```

1. Create a `ManifestCustomization` CR  

    ```bash
cat <<EOF | kubectl apply -f -
---
apiVersion: sailoperator.io/v1alpha1
kind: ManifestCustomization
metadata:
  name: ignore-mutating-webhook-configuration
  namespace: istio-system
spec:
  targetRefs:
    - kind: Istio
  rules:
    - applyTo:
        - group: admissionregistration.k8s.io/v1
          kind: MutatingWebhookConfiguration
          name: "*"
      actions:
        - operation: IGNORE
          field: metadata.labels
EOF
    ```

1. Test to modify `metadata.labels` value  

    ```bash
    kubectl -n istio-system patch mutatingwebhookconfigurations.admissionregistration.k8s.io istio-sidecar-injector --patch '{"metadata": {"labels": {"app": "sidecar-injector-test"}}}'
    ```

1. Verify that `metadata.labels.app` has been changed successfully  

    ```bash
    kubectl -n istio-system get mutatingwebhookconfigurations.admissionregistration.k8s.io istio-sidecar-injector -oyaml
    ```