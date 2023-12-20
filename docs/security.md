# Security

In this lab we explore some of the security features of the Istio service mesh.

## Mutual TLS

By default, Istio is configured such that when a service is deployed onto the mesh, it will take advantage of mutual TLS:

- Workloads are given an identity as a function of their associated service account and namespace.
- An x.509 certificate is issued to the workload (and regularly rotated) and used to identify the workload in calls to other services.

In the [observability](dashboards.md#kiali) lab, we looked at the Kiali dashboard and noted the :material-lock: icons indicating that traffic was secured with mTLS.

### Can a workload receive plain-text requests?

We can test whether a mesh workload, such as the `customers` service, will allow a plain-text request as follows:

1. Create a separate namespace that is not configured with automatic injection.

    ```{.shell .language-shell}
    kubectl create ns other-ns
    ```

1. Deploy `sleep` to that namespace

    ```{.shell .language-shell}
    kubectl apply -f sleep.yaml -n other-ns
    ```

1. Verify that the sleep pod has no sidecars:

    ```{.shell .language-shell}
    kubectl get pod -n other-ns
    ```

1. Call the customer service from that pod:

    ```{.shell .language-shell}
    kubectl exec -n other-ns deploy/sleep -- curl -s customers.default
    ```

The output is a JSON-formatted list of customers.

We conclude that Istio is configured by default to allow plain-text request.
This is called _permissive mode_ and is specifically designed to allow services that have not yet fully on-boarded onto the mesh to participate.

### Enable strict mode

Istio provides the `PeerAuthentication` custom resource to specify peer authentication policy.

1. Review the following policy.

    !!! tldr "mtls-strict.yaml"
        ```yaml linenums="1"
        --8<-- "security/mtls-strict.yaml"
        ```

    !!! info

        Strict mtls can be enabled globally by setting the namespace to the name of the Istio root namespace, which by default is `istio-system`

1. Apply the `PeerAuthentication` resource to the cluster.

    ```{.shell .language-shell}
    kubectl apply -f mtls-strict.yaml
    ```

1. Verify that the peer authentication has been applied.

    ```{.shell .language-shell}
    kubectl get peerauthentication
    ```

### Verify that plain-text requests are no longer permitted

```{.shell .language-shell}
kubectl exec -n other-ns deploy/sleep -- curl customers.default
```

The console output should indicate that the _connection was reset by peer_.

## Inspecting a workload certificate

1. Capture the certificate returned by the `customers` workload:

    ```{.shell .language-shell}
    kubectl exec deploy/sleep -c istio-proxy -- \
    openssl s_client -showcerts -connect customers:80 > cert.txt
    ```

1. Inspect the certificate with:

    ```{.shell .language-shell}
    openssl x509 -in cert.txt -text -noout
    ```

1. Review the certificate fields:

    1. The certificate validity period should be 24 hrs.
    1. The _Subject Alternative Name_ field should contain the spiffe URI.



!!! question "How do I know that traffic is mTls-encrypted?"

    Here is a recipe that uses the [`tcpdump`](https://www.tcpdump.org/){target=_blank} utility to spy on traffic to a service to verify that it is indeed encrypted.

    1. Update the Istio installation with the configuration field `values.proxy.privileged` set to `true`:

        ```{.shell .language-shell}
        istioctl install --set values.global.proxy.privileged=true
        ```

        For a description of this configuration field, see the output of `helm show values istio/istiod | grep privileged`.

    1. Restart the `customers` deployment:

        ```{.shell .language-shell}
        kubectl rollout restart deploy customers-v1
        ```

    1. Grab the IP address of the customers Pod:

        ```{.shell .language-shell}
        IP_ADDRESS=$(kubectl get pod -l app=customers -o jsonpath='{.items[0].status.podIP}')
        ```

    1. Shell into the `customers` sidecar container:

        ```{.shell .language-shell}
        kubectl exec -it svc/customers -c istio-proxy -- env IP_ADDRESS=$IP_ADDRESS /bin/bash
        ```

    1. Start `tcpdump` on the port that the `customers` service is listening on:

        ```{.shell .language-shell}
        sudo tcpdump -vvvv -A -i eth0 "((dst port 3000) and (net ${IP_ADDRESS}))"
        ```

    1. In separate terminal make a call to the customers service:

        ```{.shell .language-shell}
        kubectl exec deploy/sleep -- curl customers
        ```

    You will see encrypted text in the `tcpdump` output.

## Security in depth

Another important layer of security is to define an authorization policy, in which we allow only specific services to communicate with other services.

At the moment, any container can, for example, call the customers service or the web-frontend service.

1. Call the `customers` service.

    ```{.shell .language-shell}
    kubectl exec deploy/sleep -- curl -s customers
    ```

1. Call the `web-frontend` service.

    ```{.shell .language-shell}
    kubectl exec deploy/sleep -- curl -s web-frontend | head
    ```

Both calls succeed.

We wish to apply a policy in which _only `web-frontend` is allowed to call `customers`_, and _only the ingress gateway can call `web-frontend`_.

Study the below authorization policy.

!!! tldr "authz-policy-customers.yaml"
    ```yaml linenums="1"
    --8<-- "security/authz-policy-customers.yaml"
    ```

- The `selector` section specifies that the policy applies to the `customers` service.
- Note how the rules have a "from: source: " section indicating who is allowed in.
- The nomenclature for the value of the `principals` field comes from the [spiffe](https://spiffe.io/docs/latest/spiffe-about/overview/){target=_blank} standard.  Note how it captures the service account name and namespace associated with the `web-frontend` service.  This identity is associated with the x.509 certificate used by each service when making secure mtls calls to one another.

Tasks:

- [ ] Apply the policy to your cluster.
- [ ] Verify that you are no longer able to reach the `customers` pod from the `sleep` pod

### Challenge

Can you come up with a similar authorization policy for `web-frontend`?

- Use a copy of the `customers` authorization policy as a starting point
- Give the resource an apt name
- Revise the selector to match the `web-frontend` service
- Revise the rule to match the principal of the ingress gateway

!!! hint

    The ingress gateway has its own identity.

    Here is a command which can help you find the name of the service account associated with its identity:

    ```{.shell .language-shell}
    kubectl get pod -n istio-system -l app=istio-ingressgateway -o yaml | grep serviceAccountName
    ```

    Use this service account name together with the namespace that the ingress gateway is running in to specify the value for the `principals` field.


### Test it

Don't forget to verify that the policy is enforced.

- Call both services again from the sleep pod and ensure communication is no longer allowed.
- The console output should contain the message _RBAC: access denied_.

## Next

In the next lab we show how to use Istio's traffic management features to upgrade the `customers` service with zero downtime.
