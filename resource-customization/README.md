# Sail Operator Resource Customization

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

### Manifest Customization

#### Prerequisite

Install sail-operator which includes the manifest customization feature:  

1. Clone the Manifest Customization sail-operator POC branch  

    ```bash
    git clone -b poc/manifest-customization https://github.com/bmangoen/sail-operator.git
    ```

1. Install sail-operator

    ```bash
    cd sail-operator
    make -e HUB=quay.io/bmangoen TAG=manifest-customization deploy
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

#### Examples

##### IGNORE action on mutating webhook configuration

1. Create the `ManifestCustomization` CR sample `chart/samples/manifestcustomization/manifestcustomization-ignore-webhook.yaml`

    ```bash
    kubectl apply -n istio-system -f chart/samples/manifestcustomization/manifestcustomization-ignore-webhook.yaml
    ```

    This Manifest Customization prevents the reconcialition of the `MutatingWebhookConfiguration` `istio-sidecar-injector`'s following fields:
    * `metadata.labels.app`: user is able to modify the value of the `app` label
    * `webhooks[name:rev.namespace.sidecar-injector.istio.io].namespaceSelector.matchExpressions`: user can add a new entry of `matchExpressions` (or overwrite its content)

1. Modify `metadata.labels.app` value  

    ```bash
    # Check the value of the field `metadata.labels.app`
    kubectl -n istio-system get mutatingwebhookconfigurations.admissionregistration.k8s.io istio-sidecar-injector -ojsonpath='{.metadata.labels.app}'
    sidecar-injector
    
    # Modify the value to "sidecar-injector-test"
    kubectl -n istio-system patch mutatingwebhookconfigurations.admissionregistration.k8s.io istio-sidecar-injector --patch '{"metadata": {"labels": {"app": "sidecar-injector-test"}}}'

    # Verify the value has been changed successfully
    kubectl -n istio-system get mutatingwebhookconfigurations.admissionregistration.k8s.io istio-sidecar-injector -ojsonpath='{.metadata.labels.app}'
    sidecar-injector-test
    ```

1. Let's now simulate the [issue](https://github.com/istio-ecosystem/sail-operator/issues/1148) where an external controller adds a new entry to `webhooks[name:rev.namespace.sidecar-injector.istio.io].namespaceSelector.matchExpressions` which is currently reconciled and overwritten by Sail Operator.  

    ```bash
    # Check the content of `namespaceSelector.matchExpressions`
    kubectl -n istio-system get mutatingwebhookconfigurations.admissionregistration.k8s.io istio-sidecar-injector -ojsonpath='{.webhooks[0].namespaceSelector}' | jq .
    {
    "matchExpressions": [
        {
        "key": "istio.io/rev",
        "operator": "In",
        "values": [
            "default"
        ]
        },
        {
        "key": "istio-injection",
        "operator": "DoesNotExist"
        }
    ]
    }

    # Edit in live the MutatingWebhookConfiguration and add the new `namespaceSelector.matchExpressions` entry
    kubectl -n istio-system edit mutatingwebhookconfigurations.admissionregistration.k8s.io istio-sidecar-injector
    
    ## BEGIN - output truncated
    name: rev.namespace.sidecar-injector.istio.io
    namespaceSelector:
        matchExpressions:
        ## output truncated
        - key: kubernetes.azure.com/managedby
        operator: NotIn
        values:
        - aks
    ## END - output truncated

    # Verify this entry is not reconciled and still remains
    kubectl -n istio-system get mutatingwebhookconfigurations.admissionregistration.k8s.io istio-sidecar-injector -ojsonpath='{.webhooks[0].namespaceSelector}' | jq .
    {
    "matchExpressions": [
        {
        "key": "istio.io/rev",
        "operator": "In",
        "values": [
            "default"
        ]
        },
        {
        "key": "istio-injection",
        "operator": "DoesNotExist"
        },
        {
        "key": "kubernetes.azure.com/managedby",
        "operator": "NotIn",
        "values": [
            "aks"
        ]
        }
    ]
    }
    ```