
涉及的知识点：
- Service
- Endpoints


## 外部服务 Service
需注意 Service 是一个不带 selector 的，否则就会自动创建 Endpoint，name 保持一致即可。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: to-external
spec:
  type: NodePort
  ports:
    - port: 80
      nodePort: 31081
      protocol: TCP
---
apiVersion: v1
kind: Endpoints
metadata:
  name: to-external
subsets:
- addresses:
  - ip: 11.50.24.32
  ports:
  - port: 80
```

这样就可以通过 http://127.0.0.1:31081 访问了。

