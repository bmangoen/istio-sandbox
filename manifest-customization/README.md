# Sail Operator Manifest Customization

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