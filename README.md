# Gitlab on small Kubernetes cluster

## Provision a Kubernetes cluster
Order and provision a cluster with ovh at this url:
https://labs.ovh.com/kubernetes-k8s

Configure kubectl to point to this cluster.
Configure your domain name to point to one of your nodes.

### Open these firewall ports

Priorité | Action | Protocole | IP | Port destination | Options
--- | --- | --- | --- | --- | --- | --- | --- | --- | --- | ---
0 | 	Autoriser | 	TCP | 	tous |  | 	established
1 | 	Autoriser | 	TCP | 	tous | 	22 |
2 | 	Autoriser | 	TCP | 	tous | 	80 |
3 | 	Autoriser | 	TCP | 	tous | 	443 |
4 | 	Autoriser | 	UDP | 	tous | 	10000 |
5 | 	Autoriser | 	ICMP | 	tous |  |
6 | 	Autoriser | 	TCP | 	tous | 	30260 |
7 | 	Autoriser | 	TCP | 	tous | 	31148 |
19 | 	Refuser | 	IPv4 | 	tous |  |

## Install Helm

Create the rbac rules for tiller:
```sh
kubectl apply -f rbac.yaml
```

Install tiller:
```sh
helm init --tiller-namespace kube-system --service-account tiller
```

## Install gitlab-helm chart

```sh
helm upgrade --install gitlab-small-team .
	--set gitlab.global.hosts.domain=<your-domain>
	--set gitlab.global.hosts.externalIP=<ip-address-your-domain-resolves-to>
	--set gitlab.certmanager-issuer.email=<email-address-for-certificates>
	--set gitlab.gitlab-runner.runners.cache.s3AccessKey=<some-access-key>
	--set gitlab.gitlab-runner.runners.cache.s3SecretKey=<some-secret-key>
	--set grafana.adminPassword=<initial-password-for-monitoring>
	--set grafana.ingress.hosts=monitoring.<your-domain>
	--set grafana.ingress.tls.hosts=monitoring.<your-domain>

```
