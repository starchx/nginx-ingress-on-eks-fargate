# nginx-ingress-on-eks-fargate
The guide demonstrates how to setup Nginx Ingress Controller running on EKS Fargate.

The idea is to have ALB in front of Nginx Ingress Controller running on EKS Fargate. Nginx ingress controller uses CLB or NLB by default, however, EKS Fargate does not support CLB and NLB, because it is not possible to register cross account Fargate instances as load balancer targets. While NLB support IP as target, this is [yet to be supported](https://github.com/kubernetes/kubernetes/issues/65613) via K8s service.

The following is only for how to run nginx ingress controller on EKS Fargate.

Nginx ingress controller runs the nginx service as `nginx` user, and the nginx service listens http on port 80, https on port 443. Besides that, the nginx service runs with user `nginx`. To allow nginx service to bind port 80, 443, this securityContext has been added to nginx ingress controller deployment:

```
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            runAsUser: 101
            allowPrivilegeEscalation: true
```

On the other end, EKS Fargate currently [does not support privileged containers](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html#fargate-considerations) and if we simply deploy nginx ingress controller on EKS Fargate, we will see the pod failed to deploy:

```
  Warning  FailedScheduling  <unknown>  fargate-scheduler  Pod not supported on Fargate: invalid SecurityContext fields: AllowPrivilegeEscalation
```

To workaround this issue, we can change nginx to listen http on port 8080 and https on port 8081, and this avoid the requirement of NET_BIND_SERVICE capability and privilege escalation. The details steps are follow:

- Set `allowPrivilegeEscalation` to false in nginx ingress deployment resource:

```
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            runAsUser: 101
            allowPrivilegeEscalation: false    <<=== change this from true to false
```

- Change nginx-ingress-controller listening ports for http and https in nginx ingress deployment resource: [list of arguments for nginx-ingress-controller](https://kubernetes.github.io/ingress-nginx/user-guide/cli-arguments/)

```
          args:
            - /nginx-ingress-controller
            - --publish-service=ingress-nginx/ingress-nginx-controller
            - --election-id=ingress-controller-leader
            - --ingress-class=nginx
            - --configmap=ingress-nginx/ingress-nginx-controller
            - --validating-webhook=:8443
            - --validating-webhook-certificate=/usr/local/certificates/cert
            - --validating-webhook-key=/usr/local/certificates/key
            - --http-port=8080        <<============ add this
            - --https-port=8081       <<============ add this
```

- Change container port exposed in nginx ingress deployment resource:

```
          ports:
            - name: http
              containerPort: 8080     <<=========== change this from 80 to 8080
              protocol: TCP
            - name: https
              containerPort: 8081     <<=========== change this from 443 to 8081
              protocol: TCP
            - name: webhook
              containerPort: 8443
              protocol: TCP
```
