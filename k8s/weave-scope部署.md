# weave scope部署

[weaveworks Github](https://github.com/weaveworks/scope)

部署参考[官网](https://www.weave.works/docs/scope/latest/installing/#k8s)

```bash
kubectl apply -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

### ingress

```bash
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: weave-ingress
  namespace: weave
spec:
  rules:
  - host: weave.jrtzcloud.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: weave-scope-app
          servicePort: 80
```

