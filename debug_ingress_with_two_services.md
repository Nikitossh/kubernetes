h1. debugging psy-2101  

h2. schema:

<pre>
ingress:               /              /api
services:            frontend	     backend
pods:                 vue 	         node
</pre>

h2. check ingress logs  


<pre><code class="shell">
$ kubectl -n sandbox logs -f web-sites-nginx-ingress-597dd756f7-w6qf4  
</code></pre>

h2. Set logs to debug.

<pre><code class="shell">
$ kubectl edit deploy -n sandbox web-sites-nginx-ingress
</code></pre>

In opened file find `args` and set it to --v=5

But there were no logs... Checked default namespace and found all logs there  

<pre><code class="shell">
$ kubectl get pods
</code></pre>

Found ingress-controller's Pod and show logs.

<pre><code class="shell">
$ kubectl logs -f nginx-ingress-ingress-nginx-controller-56b98c9dc6-c4j5d 
</code></pre>

h2. Checking pods logs 

<pre><code class="shell">
$ kubectl -n sandbox logs psy-2101-0 psy-2101-vue
172.31.5.1 - - [06/Feb/2021:05:36:15 +0000] "GET / HTTP/1.1" 200 1877 "-" "kube-probe/1.17+" "-"
172.31.5.1 - - [06/Feb/2021:05:36:25 +0000] "GET / HTTP/1.1" 200 1877 "-" "kube-probe/1.17+" "-"
</code></pre>


<pre><code class="shell">
$ kubectl -n sandbox logs psy-2101-0 psy-2101-node
Server running on port 4600
Connected to the SQLite database.
</code></pre>

h2. Pods are OK. Moving to services 

<pre>
$ kubectl -n sandbox describe service psy-2101-frontend-service
Name:                     psy-2101-frontend-service
Namespace:                sandbox
Labels:                   <none>
Annotations:              Selector:  app=psy-2101-frontend-app		<----- There was a wrong value. Changed it to app=psy-2101-app
Type:                     NodePort
IP:                       10.100.178.163
Port:                     psy-2101-frontend-service  80/TCP
TargetPort:               80/TCP
NodePort:                 psy-2101-frontend-service  31863/TCP
Endpoints:                <none>                                        <------------------------- PROBLEM HERE! There is not endpoint. Fixed by changing value above.
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
</pre>

<pre>
$ kubectl -n sandbox describe service psy-2101-backend-service
Name:                     psy-2101-backend-service
Namespace:                sandbox
Labels:                   <none>
Annotations:              Selector:  app=psy-2101-backend-app		<----- There was a wrong value. Changed it to app=psy-2101-app
Type:                     NodePort
IP:                       10.100.158.97
Port:                     psy-2101-backend-service  4600/TCP
TargetPort:               4600/TCP
NodePort:                 psy-2101-backend-service  31193/TCP
Endpoints:                <none>					<------ PROBLEM HERE! There is not endpoint. Fixed above.
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
</pre>

h2. After fixing:

<pre>
$ kubectl -n sandbox describe service psy-2101-backend-service
Name:                     psy-2101-backend-service
Namespace:                sandbox
Labels:                   <none>
Annotations:              Selector:  app=psy-2101-app
Type:                     NodePort
IP:                       10.100.158.97
Port:                     psy-2101-backend-service  4600/TCP
TargetPort:               4600/TCP
NodePort:                 psy-2101-backend-service  31193/TCP
Endpoints:                172.31.14.140:4600
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
</pre>

h2. Another problem is 404 when request /api/heartbeat  

researched this:
[[https://kubernetes.github.io/ingress-nginx/examples/rewrite/]]


h2. Updated nginx-ingress to latest stable version with helm. 

<pre>
helm upgrade --reuse-values nginx-ingress ingress-nginx/ingress-nginx
</pre>

Tried deploy changes again and have new ERROR:

<pre>
Error from server (InternalError): error when creating "sandbox-psy-2101-statefulset.yml": Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": Post https://nginx-ingress-ingress-nginx-controller-admission.default.svc:443/networking/v1beta1/ingresses?timeout=10s: remote error: tls: internal error
</pre>

h2. Found solution there

[[https://github.com/kubernetes/ingress-nginx/issues/5401]]
Issue is still open, but has one workaround

<pre><code class="shell">
$ kubectl delete -A ValidatingWebhookConfiguration nginx-ingress-ingress-nginx-admission
</code></pre>


Here I have read about ValidatingAdmissionWebhook and decided to delete it untill this issue is closed.

Seems like it works! There are valid Ingress resouce definition:  
<pre>
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/enable-rewrite-log: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
  name: psy-2101-ingress
  namespace: sandbox
spec:
  rules:
  - host: sandbox-psy-2101.dotin.us
    http:
      paths:
      - backend:
          serviceName: psy-2101-backend-service
          servicePort: 4600
        path: /api(/|$)(.*)
      - backend:
          serviceName: psy-2101-frontend-service
          servicePort: 80
        path: /
  tls:
  - hosts:
    - sandbox-psy-2101.dotin.us
    secretName: sandbox-psy-2101
</pre>
