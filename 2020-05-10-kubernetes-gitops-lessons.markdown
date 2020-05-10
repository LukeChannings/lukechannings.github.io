---
Author: Luke Channings
Date: 2020-05-10
Tags: Kubernetes GitOps Debugging
---

# Mistakes made and lessons learned with Kubernetes and GitOps

The promise of Kubernetes is to manage your computing workloads in such a way that you don't need to care about which physical server any particular piece is running on.

The promise of GitOps is to define those workloads in a version-controlled, and possibly centralised repository. A slight error in that `nginx.conf` is much more likely to be noticed in a git repo than sitting in `available-site`s on some server, out of sight.

With this ideal in mind, I jumped into migrating my home services from disparate Docker Compose services. Here are some of the mistakes I made along the way, and problems that bit me.

I'm writing this article to hopefully spare you some pain and frustration, and perhaps give some insight into the problems you will face.

## Rule of thumb: Only change one thing at a time

It can be tempting when you're trying to get a complex system up and running to make lots of changes across many components, but then something breaks and you don't know specifically what caused the problem.

In my experience there are sometimes multiple things that could cause a problem, and if you've just added configuration that could be either one, you'll have a harder time debugging.

Small, incremental changes. Write, release, test, repeat.

## Keep an eye on versions

If you're using GitOps and need to manually specify versions of chart dependencies, make sure the version is what you think it is, and get the latest version with: `helm show chart repo/chart-name | grep version`. I also have a [script](https://github.com/LukeChannings/kube-config/tree/master/scripts/helm-tools#compare-helm-versions) to do this.

I have had this problem multiple times. I'd scratch my head for an hour when cross-referencing a helm chart's source with my values.yaml wondering why this thing that *definitely should work* wasn't working. It turns out the chart was out of date.

Before going into a deep debugging session, check the basics. Is the chart definitely the version you think it is?

## `echo`, `base64`, and the newline character

Secrets in Kubernetes must be encoded as base64, this is so that people looking over your shoulder don't get a peak at your password.

The problem: If you're using Terminal to generate your base64 value, you might, for example `echo 'MY SUPER SECRET PASSWORD' | base64 | pbcopy`. This looks ok on the face of it, it'll output a reasonable looking base64 string.

Because you can't decode base64 in your head, you might not notice though is the difference between `U0VDUkVUIFBBU1NXT1JECg==` and `U0VDUkVUIFBBU1NXT1JE`.

The difference is that echo adds a trailing newline character by default. This can cause havoc, because most software -- especially when comparing credentials -- will compare the exact value. It's not going to trim that newline for you, and so your password doesn't match.

You might even know this (like I did), and still make this mistake. Use `echo -n` to avoid printing the newline and you won't kick yourself. Or better yet, if you have an editor with a built-in "encode base64" and "decode base64" command, use that instead. This [VS Code extension](https://marketplace.visualstudio.com/items?itemName=ipedrazas.kubernetes-snippets) adds those commands.

## PVCs and PVs

I have had many issues with [PVCs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes) not binding to the correct [PV](), or not binding because of a mismatch.

Check your accessModes match between the PV and PVC, and use a volumeName, especially if you have multiple PVs that are unbound. You don't want the PVC to bind to the wrong volume.

If you're using a Helm chart and it doesn't let you specify a volumeName, it probably will let you use an existingClaim, so you can just define your own PVC with a volumeName.

## Ingress paths

If you're using an [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) with [ssl passthrough](https://github.com/haproxytech/kubernetes-ingress/blob/master/documentation/README.md#https), and the backend is different, you probably can't use paths.

I decided I would like WebSocket access to my MQTT broker as well as regular MQTT protocol, so I innocently added a line to my Ingress.yaml:

```yaml
  rules:
    - host: mqtt.private.channings.me
      http:
        paths:
          - path: /
            backend:
              serviceName: mosquitto
              servicePort: mqtts
          - path: /ws
            backend:
              serviceName: mosquitto
              servicePort: wsmqtts
```

This definition looks ok, and my [ingress controller](https://github.com/haproxytech/kubernetes-ingress) doesn't complain. However, it translates into a rule that looks like this:

```
use_backend mosquitto-mosquitto-wsmqtts if { req_ssl_sni -i mqtt.private.channings.me
use_backend mosquitto-mosquitto-mqtts if { req_ssl_sni -i mqtt.private.channings.me
```

As you can see, the mqtts backend will never get hit. Arguably this is a bug in the ingress controller, but it's likely a bug in other ingress controllers too. If you want to do this, a second subdomain is probably the way to go.

## Mitigating these issues

### Linting

Some of these issues are highly specific to my setup, sure. But you'll encounter highly specific issues of your own, and you'll need to be able to diagnose issues like this.

It helps if you're familiar with the software that's actually being run in the pods. Like with the haproxy issue, I spotted that because I have experience with haproxy config. Choose software you're likely to be able to debug.

Avoid stupid mistakes and save time by linting your repo. Check the YAML with [yamllint](https://yamllint.readthedocs.io/en/stable/), [kubeval](https://www.kubeval.com) can check your kubernetes definitions are correct, [helm lint](https://helm.sh/docs/helm/helm_lint/) can check your helm chart.

If you're writing shell scripts, you might also want to use [shellcheck](https://www.shellcheck.net).

Write tools to stop you from making mistakes, too. Some of these issues are quite niche, but there's nothing stopping you from writing a [small script](https://github.com/LukeChannings/kube-config/blob/master/scripts/lint.sh) to check these things before you commit. And now that [GitHub Actions](https://github.com/features/actions) is here, you can easily [run those linting tools on GitHub](https://github.com/LukeChannings/kube-config/blob/master/.github/workflows/lint.yaml).

If you're interested in a repo in which linting is set up as I've described, you can take a look at my [Kubeconfig](https://github.com/LukeChannings/kube-config) repo.

### Debugging

In my experience, using `kubectl exec` to get into a pod that isn't working correctly, to poke around, is extremely handy. But what happens if a livlinessProbe is failing and the pod is getting killed before you can find anything out? Use `kubectl edit` to just delete the livelinessProbe and stop it from being killed.

Whilst `kubectl logs` is perfectly good most of the time, but [kail](https://github.com/boz/kail) or [kubetail](https://github.com/johanhaleby/kubetail) are also good options for watching logs for a group of pods.

If you're looking at resources that are linked in a parent-child relationship (like `Certificate`, `CertificateRequest`, `Challange`, etc.), [kube tree](https://github.com/ahmetb/kubectl-tree) is useful for seeing the relationships and statues.

Here's some example output:

```bash
λ k tree certificate haproxy-internal-tls -n haproxy-ingress
NAMESPACE        NAME                                                  READY  REASON  AGE
haproxy-ingress  Certificate/haproxy-internal-tls                      True   Ready   100m
haproxy-ingress  └─CertificateRequest/haproxy-internal-tls-3343126354  True   Issued  95m
haproxy-ingress    └─Order/haproxy-internal-tls-3343126354-920263207   -              95m
```

If you're using VS Code, the [Kubernetes](https://marketplace.visualstudio.com/items?itemName=ms-kubernetes-tools.vscode-kubernetes-tools) and [Kubernetes Support](https://marketplace.visualstudio.com/items?itemName=ipedrazas.kubernetes-snippets) extensions are extremely useful to validate Kubernetes resource as you're writing them.
