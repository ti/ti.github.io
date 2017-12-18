


helm init --upgrade -i gcr.io/kubernetes-helm/tiller:v2.7.2 



tiller-deploy-546cf9696c-p7fr6


这是因为我们使用的2.3.1版本的helm init的时候没有为tiller创建serviceaccount和clusterrolebiding的缘故导致的。


```yaml



kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
# helm init -i sz-pg-oam-docker-hub-001.tendcloud.com/library/kubernetes-helm-tiller:v2.3.1
kubectl -n kube-system  patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'



kubectl -n kube-system patch deploy/tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'



```



helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.5.1 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts



CGO_ENABLED=0  go build -a -installsuffix cgo


参考文档

https://www.ibm.com/support/knowledgecenter/en/SSBS6K_1.2.0/app_center/add_package.html