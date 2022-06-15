# CKA notes
### Commends
#### Typical
```
kubectl run <pod-name> --image ...
kubectl run <pod-name> --image ... --dry-run=client -o yaml > <filename>.yaml
kubectl get all
kubectl get <resource>
kubectl get <resource> -A
kubectl get <resource> -n <namespace>
kubectl get <resource> -w wide
kubectl get <resource> -o yaml
kubectl get <resource> --as some-user
kubectl describe <resource>
kubectl create -f <filename-to-create-from>
kubectl create <resource> --help
kubectl replace -f <filename-to-create-from> --force
kubectl edit <object>
kubectl apply -f <filename-to-create-from> ??
kubectl scale
```
#### Useful
```
ps -aux | grep kubelet
kubectl auth can-i get pods --as nazwa-usera
kubectl api-resources
```
### Services
#### NodePort
Komunikacja note - swiat zewnetrzny.
#### LoadBalancer
To samo co NodePort tylko bardziej zaawansowany. Zazwyczaj w cloudach.
#### ClusterIP
Komunikacja w nodzie miedzy podami/deployami.
### Node affinity
Nodey moga miec labelki. Przy ich wykorzystaniu mozna sprecyzowac jakie nody powinien pod wybrac.
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: color
            operator: In
            values:
            - blue
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpression:
            - key: shape
              operator: In
              values:
              - circles
```
### Taints and Tolerations
Taints - mark na nodzie mowiacy o tym ze tylko pody z tolerancja na ten mark moga byc schedulowane. Mozna nadac 3 efekty:
* NoSchedule
* PreferNoSchedule
* NoExecute
To taint node
```
kubectl taint node <node-name> key1=value1:<effect>
```
To remove taint
```
kubectl taint node <node-name> key1=value1:<effect>-
```
Tolerations - tolerancja poda na tainty. W definicji poda/deployu jako element spec.
```yaml
tolerations:
- key: "color"
  operator: "Equal"
  value: "blue"
  effect: "NoSchedule"
- key: "shape"
  operator: "Exists"
  effect: "NoExecute"
```
### DaemonSet
Jest to cos jak ReplicaSet podow ktore wystepuja w 1 i tylko 1 sztuce na KAZDYM nodzie. Tworzy pody automatycznie - nie podlega pod scheduler. Uzywane np do loggowanie, monitorowania. NIE da sie stworzyc przez `kubectl create ds` !!. Trzeba trickiem `kubectl create deployment ... --dry-run=client -o yaml > daemon-set-file.yaml` potem trzeba zmienic kind na DaemonSet. Za pomoca tolerations mozna sprawic zeby omijal niektore nodey.

### StaticPods
Pody nie podlegajace pod scheduler. Dzieki czemu moga byc tworzone na node workerze bez istnienia node master. Dba o nie kubelet. Tworzy je na podstawie yml znajdujacego sie w konkretnej lokalizacji. Mozna ja znalezc robiac:
1. `ssh nazwa-nodea`
1. `ps -aux | grep kubelet` - pelna komenda odpalajaca kubelet, trzeba znalezc sciezke z flagi `--config` - pod ta sciezka znajduje sie config kubeleta.
1. `cat konfigu kubeleta` - szukamy pola `staticPod` - to jest szukana sciezka.

Ta sciezka to defaultowo
```
/etc/kubernetes/manifests/
```

### Node operations
- `kubectl drain node01` - usuwa pody z node01 i recreating je na innych nodeach - jezeli sa czescia ReplicaSet/Deployment
- `kubectl uncordon node01` - pozwala na ponowne schedulowanie podow na nodzie
- `kubectl cordon node01` - zabrania scedulowania podow na nodzie
#### Upgrade Nodea
#TBD
#### etcd backup
#TBD
### Certificates
To jest trudny temat. Ogolnie to wszystko musi miec ze wszystkim certyfikat.

Sposob na podejzenie certyfikatu jako plaintext
```
openssl x509 -in file-path.crt -text -noout
```

Jezeli sa jakies problemy z certyfikatami, to najczesciej jest to problem z api-serverem. Jego lokalizacja:
```
/etc/kubernetes/manifests/kube-apiserver.yaml
```

#### CertificateSigningRequest
jak wygenerowac taki certyfikat ??!! #TBD
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: nazwa-csr
spec:
  groups:
  - system:authenticated
  request: LS0tL-dluugie-f-huj-S1CRU
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```
```
kubectl get csr
kubectl certificate approve/deny nazwa-csr
```
### Role, RoleBinding
Role - zestaw uprawnien do konkrenych resoursow
RoleBinding - przypisanie Role do usera.
Wszystko dzieje sie w ramach konkretnego namespaceu.
```
kubectl create role
kubectl create role-binding
kubectl get pods --as nazwa-usera
kubectl auth can-i get pods --as nazwa-usera
```
### ClusterRole, ClusterRoleBinding
`kubectl api-resources` - kozak sprawa, pozwala na dowiedzenie sie co ma jakie api i skrot

Z tym jest tak samo jak ze zwyklymi rolami i bindami, z ta rownica ze uprawnienia sa wazne na calym klastrze
, nie w konkretnym namespace.

### ServiceAccount
sa jest dla botow
```
kubectl create sa
```
zawiera token, info o tokenie, mozna go wyswietlic: 
```
kubectl describe secret nazwa-tokenu
```
przy definicji poda w polu serviceAccountName mozna wpisac takie customowo utworzone i wtedy pod automatycznie uzywa tokuenu sa, a nie defaultowego.
```yaml
spec:
  serviceAccountName: token-name
```

### ImageSecurity
Obrazy dockerowe wykorzystywane w podach moga wymagac credentiali do pulla.
Docker registry zawiera sie w nazwie image'a.
```
kubectl create secret docker-registry <name> \
--docker-server= \
--docker-username= \
--docker-password= \
--docker-email= \
```
potem mozna taki secret uzywc w konfiguracji poda w sekcji spec, na rowni z containers,
```yaml
spec:
  imagePullSecrets:
  - name: myregistrykey
```
### NetworkPolicies
okresla kanaly dostepu do obiektu sprecyzowanego przez spec.podSelector:
* Ingres - kanal do poda
* Egress - kanal wychodzacy z poda
  
Wazne zeby sie nie popierdolic w yamlu... co jest kurwa niewykonalne. Udanej zabawy.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: 
  name: internal-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
  - Egress
  - Ingress
  ingress:
  - {}
  egress:
  - to:
    - podSelector:
        matchLabels:
          name: mysql
    ports:
    - protocol: TCP
      port: 3306
  - to:
    - podSelector:
        matchLabels:
          name: payroll
    ports:
    - protocol: TCP
      port: 8080
```