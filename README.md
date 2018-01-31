## Day Extra - Let's Encrypt 自動套用 SSL！保證你會愛上它！

我們在 [Day 25 - 保護你的系統：套用 SSL](https://ithelp.ithome.com.tw/articles/10193959) 曾經提到過如何替網站套用 SSL。但難道需要每次都手動增加嗎？在這自動化盛行的時代，這種麻煩的事情當然要讓它自動完成。不過跟之前不同的是，我們這次把 SSL 直接套用在 Ingress 上，並透過 [kube-lego](https://github.com/jetstack/kube-lego) 來幫助我們自動檢查並更新憑證。

### Let's Encrypt

[Let's Encrypt](https://letsencrypt.org/)，是一個提供免費 SSL 憑證的網站，除了手動申請之外，透過它提供的 APIs 介接，我們可以讓申請 SSL 的流程程式化、自動化。如果你想知道完整的過程，可以參考 [官方說明](https://letsencrypt.org/how-it-works/)。不會太複雜，基本上就是證明你擁有 Domain，並要求簽發憑證。另外，官方也推薦了很多不同語言版本的 [Client Plugins](https://letsencrypt.org/docs/client-options/)，如果有需要，也可以挑選適合自己環境的工具使用。

今天，我們要示範的是 [kube-lego](https://github.com/jetstack/kube-lego) 這支工具來幫助我們在 k8s 上完成自動套用 SSL 的步驟。

### kube-lego

[kube-lego](https://github.com/jetstack/kube-lego) 這隻工具，會主動幫我們定期檢查憑證，如果發現憑證即將到期，則自動連線到 [Let's Encrypt](https://letsencrypt.org/)，並重新簽署憑證

> kube-lego 預設每八個小時 (可變更) 會去掃描一次是否有憑證需要更新，另外，預設當憑證有效期間小於三十天時 (可變更)，就視為需要更新。


### 自動套用 SSL

接下來，我們來示範如何在 [GKE](https://cloud.google.com/kubernetes-engine/) 的 k8s 叢集中透過 [Let's Encrypt](https://letsencrypt.org/) 提供的 APIs 自動套用 SSL

#### 第一步：申請 Domain

首先，你當然要先有一個有效的網域名稱 (Domain Name)，目前網路上有很多地方可以提供[免費申請網域](https://www.google.com.tw/search?ei=nM1uWpjPK4LK0gSVupOgDg&q=free+domain&oq=free+domain&gs_l=psy-ab.3..35i39k1j0l9.8598.11239.0.11842.14.14.0.0.0.0.72.640.13.14.0....0...1c.1.64.psy-ab..0.14.672.6..0i67k1j0i131k1j0i10k1.34.9NKhup7pnPQ)的網站，或者你可以[付費購買](https://www.google.com.tw/search?ei=5M1uWuaDLcKl0gSbgaqACQ&q=purchase+domain&oq=purchase+domain&gs_l=psy-ab.3..0i7i30k1l9j0.39766.41819.0.42099.8.8.0.0.0.0.57.398.8.8.0....0...1c.1.64.psy-ab..0.8.398...0i7i30i19k1.0.J54iHfIrTi0)你需要的網域。這裡先假設你已經有了自己的網域可以使用。如果你需要教學文章，可以參考[如何申請 freenom免費網域](http://jmiscellaneous.blogspot.tw/2017/11/freenom.html)。之後會需要把網域的 IP 位置，指到 k8s 的 Ingress IP 位址。

#### 第二步：部署應用程式

首先，我們先利用 [Day 27 - 漫步在雲端：在 GCP 中建立 k8s 叢集](https://ithelp.ithome.com.tw/articles/10193961) 的方式，建立一個全新的 `ssl` 叢集

> 你也可以直接在原來的 `ithome` 叢集上建立，但為了方便講解，我們這裡重新建立一個新的叢集。透過 [Release Notes](https://cloud.google.com/kubernetes-engine/release-notes) 你會發現 `1.8.4-gke.1` 已經不能用了！更新也太快了吧！

```bash
gcloud container clusters create ssl \
> --cluster-version=1.8.6-gke.0 \   <=== 原本的 1.8.4-gke.1 已經不支援了
> --machine-type=n1-standard-1 \
> --num-nodes=2 \
> --disk-size=50 \
> --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

接著，我們拿 [Day 13 - 別耍自閉開放你的應用吧：Ingress](https://ithelp.ithome.com.tw/articles/10193528) 分享的範例裡面的 `green.myweb.com` 做修改，修正後的內容如下：

```

# ingress.deployment.yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: green-nginx
spec:
    replicas: 1
    template:
      metadata:
        labels: 
          app: green-nginx
      spec:
        containers:
        - name: nginx
          image: jlptf/green
          ports:
          - containerPort: 80

# ingress.service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: green-service
spec:
  type: NodePort
  selector:
    app: green-nginx
  ports:
    - protocol: TCP
      port: 80
      

# ingress.yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web
spec:
  tls:
  - hosts:    <=== 套用 TLS 到 ithomeironman.ddns.net
    - ithomeironman.ddns.net       <=== 這裡修改成你的網域名稱
    secretName: green-certs
  backend:    <=== 預設的服務
    serviceName: green-service
    servicePort: 80
  rules:
  - host: ithomeironman.ddns.net   <=== 這裡修改成你的網域名稱
    http:
      paths:
      - path: /*    <=== 將 ithomeironman.ddns.net 的請求都導向 green-service 
        backend:
          serviceName: green-service
          servicePort: 80

```

這裡我們做了幾個修改：

* 套用一個 TLS (Transport Layer Security) 即 SSL 在 Ingress 物件上
* 在 Ingress 裡面，我們把網域 `ithomeironman.ddns.net` 與 `green-service` 做一個綁定，即 `ithomeironman.ddns.net` 的請求會導向 `green-service` 內。然後部署上 k8s
* 指定 `green-service` 為預設的服務

> `ithomeironman.ddns.net` 是在 [noip](https://www.noip.com/) 上申請的一個免費的網域，不過它每三十天要 renew 一次

所以，我們需要先產生一個名為 `green-certs` Secret 物件先部署到 k8s 上，這個 Secret 等下就會被 [kube-lego](https://github.com/jetstack/kube-lego) 替換成正式的證書。因此，我們先做一個 10 天內會過期的證書：

```bash
$ openssl req -x509 -nodes -days 10 -newkey rsa:2048 -keyout tls.key -out tls.crt
...
Common Name (eg, fully qualified host name) []:ithomeironman.ddns.net   <=== 填入你的網域名稱
...
```

> 請務必填寫 Common Name，不然 GKE 會報錯

上述指令已經在 [Day 25 - 保護你的系統：套用 SSL](https://ithelp.ithome.com.tw/articles/10193959) 說明過，這裡就不再說明，主要的區別是 `-days 10`，因為 [kube-lego](https://github.com/jetstack/kube-lego) 預設會更新三十天內過期的憑證，所以只要證書三十天內過期即可。

因為要放到 Secret 內，必須先以 base64 加密，所以請利用下列指令取得編碼後的內容

```bash
$ cat ./tls.key | base64
<編碼後內容>   

$ cat ./tls.crt | base64
<編碼後內容>
```

然後把它放到 yaml 設定裡面

```
# secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: green-certs
type: kubernetes.io/tls   <=== kube-lego 需要使用這種 type 才能更新
data:
  tls.crt: "<tls.crt base64 編碼後內容>"
  tls.kye: "<tls.key base64 編碼後內容>"
```

然後就可以把上述內容全部部署到叢集中

```bash
$ kubectl apply -f ./   <=== 一次部署在當前目錄下的 yaml
secret "green-certs" created
deployment "green-nginx" created
service "green-service" created
ingress "web" created
```

稍等一下後，Ingress 就會被分配一個 IP 位置，請把你的網域新增加一筆 `TYPE A` 的資源紀錄 (Resource Record) 指到這個 Ingress 的 IP 位置。

```bash
$ kubectl get ingress
NAME      HOSTS              ADDRESS          PORTS     AGE
web       ithomeironman.ddns.net   35.227.193.1   80        2m
<=== 請將 ithomeironman.ddns.net 網域指到 35.227.193.1 這個位置
```

> 可能會需要一段時間，網域才會生效

沒意外的話，你應該可以透過瀏覽器存取 `https://ithomeironman.ddns.net` 並看到畫面如下：

![https://ithelp.ithome.com.tw/upload/images/20180131/20107062N4fFrCHGhI.png](https://ithelp.ithome.com.tw/upload/images/20180131/20107062N4fFrCHGhI.png)

#### 第三步：部署 kube-lego

確保 `https://ithomeironman.ddns.net` 已經可以被正確存取之後，接下來我們要部署 [kube-lego](https://github.com/jetstack/kube-lego)，底下的內容是參考 [範例](https://github.com/jetstack/kube-lego/blob/master/examples/gce/lego/deployment.yaml)後修改的：

```
# lego.yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kube-lego
  namespace: default
spec:
  replicas: 1
  template:
    metadata:
      labels:
        # Required for the auto-create kube-lego-nginx service to work.
        app: kube-lego
    spec:
      serviceAccountName: kube-lego   <=== 這裡順便測試一下套用 service account
      containers:
      - name: kube-lego
        image: jetstack/kube-lego:0.1.5
        imagePullPolicy: Always
        ports:
        - containerPort: 8080    <=== 預設 port 請填 8080
        env:
        - name: LEGO_LOG_LEVEL
          value: debug
        - name: LEGO_EMAIL
          value: "your email address"   <=== 請換成你的 email
        - name: LEGO_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: LEGO_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 1
```

* `serviceAccountName: kube-lego`：這裡我們套用一個 Service Account 來順便測試一下，如果沒指定，則會使用命名空間內預設的 `default` Service Account。
* `env: LEGO_LOG_LEVEL`：可以看到 log
* `env: LEGO_EMAIL`：這裡需要換成你的 email

> 請注意，雖然 [Let's Encrypt](https://letsencrypt.org/) 是免費的，但是不表示你可以無限制的申請，目前一週的限制是 20次/週，即你每週可以申請 20 次，超過了就可能會被擋掉！

* `env: LEGO_NAMESPACE`：[kube-lego](https://github.com/jetstack/kube-lego) 運行的命名空間
* `env: LEGO_POD_IP`：[kube-lego](https://github.com/jetstack/kube-lego) 運行的 Pod IP，很重要，`status.podIP` 即為目前 Pod 的運行 IP

> 這裡為了方便起見直接填值，一般來說會把這些設定放在 ConfigMap 物件中再套用

其他部分，我們在先前的文章都有說明過，所以就不再說明。接著我們先產生一個 Service Account，然後再賦予它 `admin` 的權限，然後再把 Pod 部署上去。

> 這裡賦予 `admin` 的權限是方便操作，在實務上請視需求分配適當的權限。因為 [kube-lego](https://github.com/jetstack/kube-lego) 與 [Let's Encrypt](https://letsencrypt.org/) 溝通後，會取得新的證書，然後需要更新 Sercet 物件，因此需要分配足夠的權限給它。

```bash
$ kubectl create serviceaccount kube-lego   <=== 建立一個新的 Service Account
serviceaccount "kube-lego" created

$ kubectl create clusterrolebinding kube-lego \
--clusterrole=admin \
--serviceaccount=default:kube-lego   <=== default 底下的 kube-lego
clusterrolebinding "kube-lego" created

$ kubectl apply -f lego.yaml
deployment "kube-lego" created
```

你需要修改下面的設定，以配合 [kube-lego](https://github.com/jetstack/kube-lego)

```
# ingress.yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web
  annotations:
    kubernetes.io/tls-acme: "true"       <=== 有這個，kube-lego 才會處理
    kubernetes.io/ingress.class: "gce"   <=== 在 GCP 請填 gce
spec:
  tls:
  - hosts:
    - ithomeironman.ddns.net
    secretName: green-certs
  backend:
    serviceName: green-service
    servicePort: 80
  rules:
  - host: ithomeironman.ddns.net
    http:
      paths:
      - path: /*
        backend:
          serviceName: green-service
          servicePort: 80
  - host: ithomeironman.ddns.net
    http:
      paths:   <=== 遇到 /.well-known/acme-challenge 導向 kube-lego-gce
      - path: /.well-known/acme-challenge   
        backend:
          serviceName: kube-lego-gce   <=== kube-lego 建立的 Service
          servicePort: 8080
```

* `kubernetes.io/tls-acme: "true"`：[kube-lego](https://github.com/jetstack/kube-lego) 會定期搜尋這個，如果沒有設定的話，就不會處理。
* `kubernetes.io/ingress.class: "gce"`：目前支援 `gce` 與 `nginx`，由於我們使用 GCP 故填入 `gce`，然後 [kube-lego](https://github.com/jetstack/kube-lego) 會建立一個名為 `kube-lego-gce` 的 Service 物件。
* `path: /.well-known/acme-challenge`：這個位置是 [kube-lego](https://github.com/jetstack/kube-lego) 與 [Let's Encrypt](https://letsencrypt.org/) 溝通驗證的路徑，因此，當收到這個存取位址的要求時，我們直接把它導向 `kube-lego-gce` 即 `kube-lego` 處理。

然後就部署更新後的內容：

```bash
$ kubectl apply -f ingress.yaml
ingress "web" configured
```

然後可以觀察一下 log

```bash
$ kubectl logs kube-lego-777bbb747-gmqrt
...
level=debug msg="testing reachability of http://ithomeironman.ddns.net/.well-known/acme-challenge/_selftest" context=acme domain=ithomeironman.ddns.net 
level=debug msg="error while authorizing: reachability test failed: wrong status code '404'"
...
```

你會發現 `404` 告訴你找不到網址，為什麼會這樣？不用緊張，因為 GKE 的 Load Balancer 需要一段時間生效，因為它需要去檢查是否能夠存取相關的路徑，等到生效之後，你應該可以看到：

```bash
$ kubectl logs kube-lego-777bbb747-gmqrt
...
<=== 這裡告訴你已經成功拿到證書
level=info msg="successfully got certificate: domains=[ithomeironman.ddns.net] url=https://acme-staging.api.letsencrypt.org/acme/cert/fa92c8898d40348e297f9552d76ff0552739" context=acme 
...
<=== Let's Encrypt 的證書是九十天過期，這邊可以看到還剩多久需要更新
level=info msg="cert expires in 90.0 days, no renewal needed"
level=debug msg="worker: done processing true" context=kubelego 
...
```

到這裡，自動更新就算是成功了！這時候當然要去看一下畫面，不過你會發現跟上面一樣還是不安全！這是為什麼呢？不是已經成功了嗎？如果你有仔細看上面的 log 你會發現驗證的路徑是 `https://acme-staging.api.letsencrypt.org`，看到 `staging` 這個關鍵字了嗎？[kube-lego](https://github.com/jetstack/kube-lego) 預設會使用 [Let's Encrypt](https://letsencrypt.org/) 提供的測試環境，即 `staging` ，所以 `staging` 所拿到的證書當然是假的，就跟我們自己手動產證書的意思是一樣的。別擔心，我們只剩下最後一哩路！


#### 第四步：使用正式的 SSL 服務

如果要使用 [Let's Encrypt](https://letsencrypt.org/) 提供的正式的證書，只需要把環境從 `https://acme-staging.api.letsencrypt.org/directory` 換成 `https://acme-v01.api.letsencrypt.org/directory` 即可。

我們只要修改一下 lego.yaml 的內容如下

```
# lego.yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kube-lego
  namespace: default
spec:
  replicas: 1
  template:
    metadata:
      labels:
        # Required for the auto-create kube-lego-nginx service to work.
        app: kube-lego
    spec:
      serviceAccountName: kube-lego
      containers:
      - name: kube-lego
        ...
        - name: LEGO_URL   <=== 加這個環境變數指定正式環境的 url
          value: "https://acme-v01.api.letsencrypt.org/directory"
        ...

```

上面的內容有部份省略，其實只要在環境變數中增加一個 `LEGO_URL` 並把正式環境的 url 填入即可。接下來，只要帥氣的部署上去：

```bash
$ kubectl apply -f lego.yaml
deployment "kube-lego" configured
```

然後看一下 log

```bash
$ kubectl logs kube-lego-8cf4df799-cqkf5
...
level=info msg="cert expires in 89.9 days, no renewal needed" 
...
```

這裡告訴你，證書還沒過期！因為他參考的是剛剛透過 `staging` 所建立的內容，這時候你只要再把剛剛的 secret.yaml 再部署一次，並把 kube-lego Pod 刪掉讓它重新執行一次

```bash
$ kubectl apply -f secret.yaml 
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply  <=== 這個警告不用管他，因為現在的物件是 kube-lego 產生的
secret "green-certs" configured

$ kubectl delete pod kube-lego-8cf4df799-cqkf5
pod "kube-lego-8cf4df799-cqkf5" deleted
```

接著新的 kube-lego Pod 就會被建立，你可以查看一下 log

```bash
$ kubectl logs kube-lego-8cf4df799-nd647
...
<=== 準備建立新的 Secret 物件
level=info msg="Attempting to create new secret"
...
<=== 這邊告訴你通過認證了
level=info msg="authorization successful" context=acme domain=ithomeironman.ddns.net
level=info msg="successfully got certificate: domains=[ithomeironman.ddns.net] url=https://acme-v01.api.letsencrypt.org/acme/cert/03f93a03bdf4b22e25fbccc3b9693b144b6a" context=acme 
```

最後，我們還需要 Ingress 重新載入一下，你可以在 Ingress 內加入 `last_updated` 的 `labels` 讓它重新載入如下：

```
# ingress.yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web
  labels:
    last_updated: "1497599933"   <=== 這裡指定最後更新日期，修改日期可以讓 Ingress 重新載入內容
  annotations:
    kubernetes.io/tls-acme: "true"
    kubernetes.io/ingress.class: "gce"
spec:
...
```

> 你不強制重新載入 Ingress 也可以，但是需要等一段時間才會更新

最後，你就可以透過 `https://ithomeironman.ddns.net` 存取網頁了！有沒有覺得看到 `安全` 心裡就踏實點了！

![https://ithelp.ithome.com.tw/upload/images/20180131/20107062JTY505RuHv.png](https://ithelp.ithome.com.tw/upload/images/20180131/20107062JTY505RuHv.png)
