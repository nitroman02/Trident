# Trident導入・検証

- 参考サイト（公式）
  
https://netapp-trident.readthedocs.io/en/stable-v21.01/kubernetes/deploying/operator-deploy.html#deploy-trident-operator-manually

<br>
以降、Operator版での導入を実施

###   パッケージDL・展開

```
wget https://github.com/NetApp/trident/releases/download/v21.01.2/trident-installer-21.01.2.tar.gz
tar -xf trident-installer-21.01.2.tar.gz
cd trident-installer
```
展開後は以下の感じ。helmディレクトリ内のtgzファイルを、helmでのインストールで指定する。

tridentctlはtrident操作用の（たぶん）バイナリ

Tridentは、CSI driver + tridentctl（連携操作用のツール）で構成されていて、連携するストレージの登録などはtridentctlを利用して実施する。
```
# ls -l
total 36884
drwxr-xr-x. 3 root root      231 Apr 19 10:08 deploy
drwxr-xr-x. 4 root root       30 Apr  9 05:09 extras
drwxr-xr-x. 2 root root       42 Apr  9 05:09 helm
drwxr-xr-x. 2 root root     4096 Apr  9 05:01 sample-input
-rwxr-xr-x. 1 root root 37765120 Apr  9 05:09 tridentctl
```

### HelmでTridentをインストール
helm install <名前> <packageパス>
<br>※<名前>は任意で指定

```
# helm install trident trident-installer/helm/trident-operator-21.01.2.tgz 
NAME: trident
LAST DEPLOYED: Mon Apr 19 11:59:05 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing trident-operator, which will deploy and manage NetApp's Trident CSI
storage provisioner for Kubernetes.

Your release is named 'trident' and is installed into the 'default' namespace.
Please note that there must be only one instance of Trident (and trident-operator) in a Kubernetes cluster.

To configure Trident to manage storage resources, you will need a copy of tridentctl, which is
available in pre-packaged Trident releases.  You may find all Trident releases and source code
online at https://github.com/NetApp/trident.

To learn more about the release, try:

  $ helm status trident
  $ helm get all trident
```

```
# helm list
NAME   	NAMESPACE	REVISION	UPDATED                               	STATUS  	CHART                   	APP VERSION
trident	default  	1       	2021-04-19 11:59:05.33153328 +0900 JST	deployed	trident-operator-21.01.2	21.01.2    

```
リソースはdefault Namespaceにデプロイされる模様。

インストール後Tridentの各種APIオブジェクトがデプロイされる。

```
# kubectl get all |grep trident
pod/trident-csi-5lwwt                   2/2     Running   0          83m
pod/trident-csi-6649bc485c-lt2m7        6/6     Running   0          83m
pod/trident-csi-j74ls                   2/2     Running   0          83m
pod/trident-csi-k4s8z                   2/2     Running   0          83m
pod/trident-csi-nz75v                   2/2     Running   0          83m
pod/trident-csi-tsvp2                   2/2     Running   0          83m
pod/trident-csi-w22t5                   2/2     Running   0          83m
pod/trident-operator-7c77dd5466-gbqw4   1/1     Running   0          84m
service/trident-csi   ClusterIP   10.101.56.46   <none>        34571/TCP,9220/TCP   83m
daemonset.apps/trident-csi   6         6         6       6            6           kubernetes.io/arch=amd64,kubernetes.io/os=linux   83m
deployment.apps/trident-csi        1/1     1            1           83m
deployment.apps/trident-operator   1/1     1            1           84m
replicaset.apps/trident-csi-6649bc485c        1         1         1       83m
replicaset.apps/trident-operator-7c77dd5466   1         1         1       84m
#
# kubectl get crd
NAME                                                  CREATED AT
bgpconfigurations.crd.projectcalico.org               2021-04-19T02:57:53Z
bgppeers.crd.projectcalico.org                        2021-04-19T02:57:53Z
blockaffinities.crd.projectcalico.org                 2021-04-19T02:57:53Z
clusterinformations.crd.projectcalico.org             2021-04-19T02:57:53Z
felixconfigurations.crd.projectcalico.org             2021-04-19T02:57:53Z
globalnetworkpolicies.crd.projectcalico.org           2021-04-19T02:57:53Z
globalnetworksets.crd.projectcalico.org               2021-04-19T02:57:53Z
hostendpoints.crd.projectcalico.org                   2021-04-19T02:57:53Z
ipamblocks.crd.projectcalico.org                      2021-04-19T02:57:53Z
ipamconfigs.crd.projectcalico.org                     2021-04-19T02:57:53Z
ipamhandles.crd.projectcalico.org                     2021-04-19T02:57:53Z
ippools.crd.projectcalico.org                         2021-04-19T02:57:53Z
kubecontrollersconfigurations.crd.projectcalico.org   2021-04-19T02:57:53Z
networkpolicies.crd.projectcalico.org                 2021-04-19T02:57:53Z
networksets.crd.projectcalico.org                     2021-04-19T02:57:53Z
tridentbackends.trident.netapp.io                     2021-04-19T02:59:17Z
tridentnodes.trident.netapp.io                        2021-04-19T02:59:19Z
tridentorchestrators.trident.netapp.io                2021-04-19T02:58:11Z
tridentsnapshots.trident.netapp.io                    2021-04-19T02:59:20Z
tridentstorageclasses.trident.netapp.io               2021-04-19T02:59:18Z
tridenttransactions.trident.netapp.io                 2021-04-19T02:59:19Z
tridentversions.trident.netapp.io                     2021-04-19T02:59:17Z
tridentvolumes.trident.netapp.io                      2021-04-19T02:59:18Z
```
<br>

### 参考

<details>
  <summary> 手動インストールの場合</summary>

#### Namespace作成
    $ kubectl apply -f deploy/namespace.yaml


#### ツールのコンポーネント一式をデプロイ
your Kubernetes version must be 1.16 and above

```
# kubectl create -f deploy/crds/trident.netapp.io_tridentorchestrators_crd_post1.16.yaml
customresourcedefinition.apiextensions.k8s.io/tridentorchestrators.trident.netapp.io created
```

```
# kubectl get customresourcedefinition tridentorchestrators.trident.netapp.io
NAME                                     CREATED AT
tridentorchestrators.trident.netapp.io   2021-04-19T00:48:16Z
```

```
# kubectl create -f deploy/bundle.yaml
serviceaccount/trident-operator created
clusterrole.rbac.authorization.k8s.io/trident-operator created
clusterrolebinding.rbac.authorization.k8s.io/trident-operator created
deployment.apps/trident-operator created
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy/tridentoperatorpods created
```
※
namespaceでデフォルトの「trident」以外を利用したい場合は、bundle.yaml内でのnamespace指定を編集する必要がある
- Have you updated the yaml manifests? Generate your bundle.yaml using the kustomization.yaml
```
kubectl kustomize deploy/ > deploy/bundle.yaml
```
</details>

<br>

### NFS/iSCSIなどのモジュールをインストール
    yum install -y nfs-utils


### Backend設定
- ONTAPの場合

ONTAP用のドライバだけで5種類くらいある。
-economyとついてるやつは、Qtreeとしてデプロイされるっぽい

https://netapp-trident.readthedocs.io/en/stable-v21.01/kubernetes/operations/tasks/backends/ontap/drivers.html#choosing-a-driver

連携ストレージの認証、Credential-based, Certificate-basedの2種類がある

### ONTAP側設定
- Trident用SVM作成
- nfs on 			※個人的に忘れがち
- Data LIF作成
- export-policy設定 ※ export-policyも設定されるパターンもある
- aggr-list登録
```
AFF8040::> vserver show -vserver ti_Trident_svm -fields aggr-list
vserver        aggr-list
-------------- ---------
ti_Trident_svm aggr1_n1
```

trident-installer/sample-inputにバックエンド登録用のサンプルファイルが置いてあるので、登録したい環境に合ったファイルを編集してください。
（登録名、IP、クレデンシャル等を記載）

### バックエンド登録
tridentctlコマンドでバックエンドとなるストレージ情報を登録
```
# ./tridentctl create backend -f ../backend-ontap-nas_AFF8040.json
+---------+----------------+--------------------------------------+--------+---------+
|  NAME   | STORAGE DRIVER |                 UUID                 | STATE  | VOLUMES |
+---------+----------------+--------------------------------------+--------+---------+
| AFF8040 | ontap-nas      | bf432fe1-9a9d-46cc-b9d5-30eaf7c26f35 | online |       0 |
+---------+----------------+--------------------------------------+--------+---------+
```

### StorageClass作成
trident-installer/sample-input にsample-input/storage-class-***というのがあるので、参考にして利用。
```
# cat sample-input/storage-class-ontapnas-gold.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ontap-gold
provisioner: netapp.io/trident
parameters:
  backendType: "ontap-nas"		★（おそらく）tridentctlで作成したBackendを指定
  media: "ssd"
  provisioningType: "thin"
  snapshots: "true"
```
```
# kubectl create -f sample-input/storage-class-ontapnas-gold.yaml
storageclass.storage.k8s.io/ontap-gold created
# kubectl get sc
NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
ontap-gold   csi.trident.netapp.io   Delete          Immediate           false                  17m
```

以下の感じで、StorageClassに紐づくBackendを確認できる
jqコマンドインストール
```
# yum -y install epel-release
# yum -y install jq
# ./tridentctl get storageclass -o json | jq  '[.items[] | {storageClass: .Config.name, backends: [.storage]|unique}]'
[
  {
    "storageClass": "ontap-gold",
    "backends": [
      {
        "AFF8040": [
          "aggr1_n1",
          "aggr1_n2"
        ]
      }
    ]
  }
]
```



### その他
★ PVがDeleteされるとONTAP上でも削除される
⇒ *Storage Class*の設定がデフォルトで*RECLAIMPOLICY*が*DELETE*になっているかと思われます

★ リソース要件は？
⇒ 明示的な記載なし…

### CSIでのNetApp利用
PVCの定義内で、上記で作成した*Storage Class*(この例だとontap-gold)をを指定することで、バックエンドとして登録したストレージ上にVolumeが作成されるようになります。
