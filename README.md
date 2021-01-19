# Knative Serving Serverles ( FaaS ) Demo

https://knative.dev/docs/serving/

```sh
### Depoyu çekelim
git clone https://github.com/dincerkurnaz/knative-demo.git
docker build -t dincerkurnaz/echo:v1 .
docker push dincerkurnaz/echo:v1

### kubernetes kontrol edelim
kubectl get nodes

NAME                          STATUS   ROLES    AGE    VERSION
lke16824-20536-5ffee265d590   Ready    <none>   105s   v1.18.8

### istio yu kuralım ve default namespace de aktif edelim.
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.8.1
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled

### istio kurulumunu kontrol edelim
kubectl get ns

NAME              STATUS   AGE
default           Active   4m18s
istio-system      Active   48s
kube-node-lease   Active   4m20s
kube-public       Active   4m20s
kube-system       Active   4m20s

### Knative kuralım
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.18.0/serving-crds.yaml
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.18.0/serving-core.yaml
kubectl apply --filename https://github.com/knative/net-istio/releases/download/v0.19.0/release.yaml
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.19.0/serving-default-domain.yaml
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.19.0/serving-hpa.yaml

### Knative kontrol edelim
kubectl get pods -n knative-serving

NAME                                READY   STATUS      RESTARTS   AGE
activator-7dc7bc9d8-gltpj           1/1     Running     0          54s
autoscaler-598bc966b4-rjkct         1/1     Running     0          53s
autoscaler-hpa-5cdd7c6b69-djkgt     1/1     Running     0          32s
controller-7859944464-t9dfx         1/1     Running     0          53s
default-domain-9sxrm                0/1     Completed   0          38s
istio-webhook-75cc84fbd4-b4rn5      1/1     Running     0          45s
networking-istio-6dcbd4b5f4-2k8kl   1/1     Running     0          45s
webhook-dfc6974b-cstbs              1/1     Running     0          53s

### https://github.com/knative/client
kn version

Version:      v20210111-c795f213
Build Date:   2021-01-11 10:43:07
Git Revision: c795f213
Supported APIs:
* Serving
  - serving.knative.dev/v1 (knative-serving v0.19.0)
* Eventing
  - sources.knative.dev/v1alpha2 (knative-eventing v0.19.0)
  - eventing.knative.dev/v1beta1 (knative-eventing v0.19.0)

### İlk serverless uygulamamızı deploy edelim
kn service create hello --image dincerkurnaz/echo:v1 # --namespace hello

Creating service 'hello' in namespace 'default':

  0.022s The Configuration is still working to reflect the latest desired specification.
  0.116s The Route is still working to reflect the latest desired specification.
  0.170s Configuration "hello" is waiting for a Revision to become ready.
 27.339s ...
 27.340s Ingress has not yet been reconciled.
 27.393s Waiting for load balancer to be ready
 27.575s Ready to serve.

Service 'hello' created to latest revision 'hello-grkcq-1' is available at URL:
http://hello.default.185.3.93.24.xip.io

kn service update hello --concurrency-limit=1
kn service update hello --revision-name hello-v1 --concurrency-limit=0
kn service update hello --revision-name hello-v2 --env TARGET="dincer"

kn revision list
NAME            SERVICE   TRAFFIC   TAGS   GENERATION   AGE     CONDITIONS   READY   REASON
hello-v2        hello     100%             3            24s     4 OK / 4     True
hello-v1        hello                      2            86s     3 OK / 4     True
hello-mphzp-1   hello                      1            2m35s   3 OK / 4     True

kn service update hello --traffic hello-v1=50,hello-v2=50

kn revision list
NAME            SERVICE   TRAFFIC   TAGS   GENERATION   AGE     CONDITIONS   READY   REASON
hello-v2        hello     50%              3            2m16s   4 OK / 4     True
hello-v1        hello     50%              2            3m18s   4 OK / 4     True

kn service update hello --revision-name hello-v3 --tag hello-v3=test --env TARGET="dincer test"

curl http://test-hello.default.185.3.93.24.xip.io/
Hello dincer test!

kn service describe hello

kn route describe hello

### Stress testi yapalım
hey -z 30s -c 50 "http://hello.default.185.3.93.24.xip.io/" 

kubectl get pods -w

### İstek yokken Pod ları kontrol edelim Pod zero yu gözlemleyelim. 
kubectl get pods

No resources found in default namespace.

### Kubernetes CRD sayesinde
kubectl get ksvc

NAME    URL                                       LATESTCREATED   LATESTREADY   READY   REASON
hello   http://hello.default.185.3.93.24.xip.io   hello-v3        hello-v3      True

### Kubernetes Yaml Objesi olarak
kubectl get ksvc hello -oyaml

### İşimiz bitti serverless uygulamamızı silelim.
kn service delete hello

### microservice image processing
kn service create image --image h2non/imaginary --arg="-enable-url-source" --concurrency-limit=10 # "--scale-min=1" Eger Zero olmasın cold-start biraz hızlı olsun isteniyorsa ekleyebiliriz. Disable Scale-to-Zero

http://image.default.185.3.93.24.xip.io/form
http://image.default.185.3.93.24.xip.io/crop?width=500&height=400&url=https://raw.githubusercontent.com/h2non/imaginary/master/testdata/large.jpg&type=webp&quality=90

kubectl apply -f image.yaml

kn service update image --scale-min=1

```
