## KUDO Instance parameterization repro

### Installing external operator directly (no issues)

```
kubectl kudo install external_operator
```
Once installed, check created `ConfigMap`s which use `range` and `toYaml` for generating a list from array paramter: 
[config-map-range.yaml](external_operator/templates/config-map-range.yaml) and [config-map-toyaml.yaml](external_operator/templates/config-map-toyaml.yaml)
```
kubectl get configmap example-config-range -o yaml

apiVersion: v1
data:
  config.yaml: |-
    images:
    - example/x:latest
    - example/y:latest
    - example/z:latest
...
<rest of the output>
```

```
kubectl get configmap example-config-toyaml -o yaml

apiVersion: v1
data:
  config.yaml: |-
    images:
      - example/x:latest
      - example/y:latest
      - example/z:latest
...
<rest of the output>
```

### Creating external operator version "manually" and parameterizing the instance from another operator

Uninstall previously installed operator and install operator version only. The instance will be created
by the "main" operator using [external.yaml](main_operator/templates/external.yaml) manifest. 

```
kubectl kudo uninstall --instance external-operator-instance
kubectl kudo install external_operator --skip-instance
kubectl kudo install main_operator
```

Once installed, check created `ConfigMap`s
```
kubectl get configmap example-config-toyaml -o yaml
apiVersion: v1
data:
  config.yaml: |-
    images:
      - example/x:test example/y:test example/z:test
...
<rest of the output>
```

```
kubectl get configmap example-config-range -o yaml
apiVersion: v1
data:
  config.yaml: |-
    images:
    - example/x:test example/y:test example/z:test
...
<rest of the output>
```

### Discussion
From the observed behavior it looks like that the array which is passed via instance parameter is transformed to a one-element
array which contains a string representation of the array passed as a parameter. To verify this theory, check 
[config-map-custom.yaml](external_operator/templates/config-map-custom.yaml) template where this situation addressed
explicitly.
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-config-custom
data:
  config.yaml: |-
    images:
    {{- if eq (len .Params.images) 1 }}
    {{- range $image := (splitList " " (index .Params.images 0))}}
    - {{$image -}}
    {{- end }}
    {{- else }}
    {{- range $image := .Params.images }}
    - {{$image -}}
    {{- end }}
    {{- end }}
```

Check the generated `ConfigMap`:
```
kubectl get configmap example-config-custom -o yaml
apiVersion: v1
data:
  config.yaml: |-
    images:
    - example/x:test
    - example/y:test
    - example/z:test
...
<rest of the output>
```
