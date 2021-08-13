# ADR 5: No More Bootstrap

## Status

Accepted

## Context

### Problem 1

Connaisseur's installation order is fairly critical. The webhook responsible for intercepting all requests is dependent on the Connaisseur pods and can only work, if those pods are available and ready. If not and  `FailurePolicy` is set to `Fail`, the webhook will block anything and everything, including the Connaisseur pods themselves. This means, the webhook must be installed *after* the Connaisseur pods are ready. This was previously solved using the `post-install` helm hook, which installs the webhook configuration after all other resources have been applied and are considered ready. Just for installation purposes, this solution suffices. A downside of this is, every resource installed via a helm hook isn't natively considered to be part of the chart, meaning a `helm uninstall` would completely ignore those resources and leave the webhook configuration in place. Then the situation of everything an anything being blocked arises again. Additionally, upgrading won't be possible, since you can't tell helm to temporarily delete resources and then reapply them. That's why the `helm-hook` image and `bootstrap-sentinel` where introduced. They were used to temporarily delete the webhook and reapply it before and after installations, in order to beat the race conditions. Unfortunately this solution always felt a bit clunky and added a lot of complexity for a seemingly simple problem. This ADR explains a new simpler solution.

### Problem 2

All admission webhooks must use TLS for communication purposes or they won't be accepted by Kubernetes. That is why Connaisseur creates its own self signed certificate, which it uses for communication between the webhook and its pods. This certificate is created within the helm chart, using the native `genSelfSignedCert` function, which makes Connaisseur pipeline friendly as there is no need for additional package installation such as OpenSSL. Unfortunately this certificate gets created every time helm is used, whether that being a `helm install` or `helm upgrade`. Especially during an upgrade, the webhook will get a new certificate, while the pods will get their new one written into a secret. The problem is that the pods will only capitalize on the new certificate inside the secret once they are restarted. If no restart happens, the pods and webhook will have different certificates and any validation will fail.

## Solutions

### Solution 1

The `bootstrap sentinel` and `helm-hook` image won't be used anymore. Instead, an empty webhook configuration (a configuration without any rules) will be applied along all other resources during the normal helm installation phase. This way the webhook can be normally deleted with the `helm uninstall` command. Additionally, during the `post-install` (and `post-upgrade`/`post-rollback`) helm hook, the webhook will be updated so it actually can intercept incoming request. So in a sense an unloaded webhook gets installed, which then gets "armed" during `post-install`. This also works during an upgrade, since the now "armed" webhook will be overwritten by the empty one when trying to apply the chart again! This will obviously be reverted back again after upgrading, with a `post-upgrade` helm hook.

### Solution 2

Instead of always generating a new certificate, the `lookup` function for helm templates could be used, to see whether there already is a secret defined that contains a certificate and then use this one. This way the same certificate is reused the whole time so no pod restarts are necessary. Should there be no secret with certificate to begin with, a new one can be generate within the helm chart.

## Positive Outcomes

There is no more need for the complex `bootstrap-sentinel` + `helm-hook` image approach, as the same result can be achieved much simpler. Additionally with the reuse of the certificate, `helm upgrade` is reusable again.

## Negative Outcomes

--
