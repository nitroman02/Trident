# Trident導入・検証 - ONTAP NAS/iSCSI編 -
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Trident導入・検証 - ONTAP NAS/iSCSI編 -](#trident導入検証---ontap-nasiscsi編--)
  - [環境](#環境)
  - [Tridentセットアップ](#tridentセットアップ)
    - [パッケージDL・展開](#パッケージdl展開)
    - [HelmでTridentをインストール](#helmでtridentをインストール)
    - [参考](#参考)
      - [Namespace作成](#namespace作成)
      - [ツールのコンポーネント一式をデプロイ](#ツールのコンポーネント一式をデプロイ)
    - [Backend設定](#backend設定)
    - [ONTAP側設定](#ontap側設定)
  - [NAS編](#nas編)
    - [WorkerNode準備](#workernode準備)
    - [Backend登録](#backend登録)
    - [StorageClass作成](#storageclass作成)
  - [SAN編](#san編)
    - [WorkerNode準備](#workernode準備-1)
    - [ONTAP SAN(iSCSI)設定](#ontap-saniscsi設定)
    - [Backend登録](#backend登録-1)
    - [StorageClass作成](#storageclass作成-1)
  - [Tips/その他](#tipsその他)
        - [インストール周り](#インストール周り)
        - [トラブルシューティング方法](#トラブルシューティング方法)
        - [PVがDeleteされるとONTAP上でも削除される](#pvがdeleteされるとontap上でも削除される)
        - [リソース要件は？](#リソース要件は)
        - [CSIでのNetApp利用](#csiでのnetapp利用)

<!-- /code_chunk_output -->


## 環境
* Linux 3.10.0-1160.24.1.el7.x86_64 (CentOS 7.9.2009)
* Kubernetes ver. 1.20.2-0
* Trident ver. 21.01.2
<br>

## Tridentセットアップ
- 参考サイト（公式）

導入にあたって以下を参照
  
https://netapp-trident.readthedocs.io/en/latest/kubernetes/deploying/operator-deploy.html
（Webサイト左下のプルダウンから、Ver.指定可能）
<br>


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
<br>

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
  <summary> 手動インストールの場合（クリックで展開）</summary>

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

### Backend設定
- ONTAPの場合

ONTAP用のドライバだけで5種類くらいある。
-economyとついてるやつは、Qtreeとしてデプロイされるっぽい

https://netapp-trident.readthedocs.io/en/stable-v21.01/kubernetes/operations/tasks/backends/ontap/drivers.html#choosing-a-driver

連携ストレージの認証、Credential-based, Certificate-basedの2種類がある
<br>

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


<br>

## NAS編
### WorkerNode準備
・NFSモジュールインストール
```
yum install -y nfs-utils
```
<br>

### Backend登録
tridentctlコマンドでバックエンドとなるストレージ情報を登録
　※ trident-installer/sample-inputにバックエンド登録用のサンプルファイルが置いてあるので、登録したい環境に合ったファイルを編集してください。
（登録名、IP、クレデンシャル等を記載）
```
# ./tridentctl create backend -f ../backend-ontap-nas_AFF8040.json
+---------+----------------+--------------------------------------+--------+---------+
|  NAME   | STORAGE DRIVER |                 UUID                 | STATE  | VOLUMES |
+---------+----------------+--------------------------------------+--------+---------+
| AFF8040 | ontap-nas      | bf432fe1-9a9d-46cc-b9d5-30eaf7c26f35 | online |       0 |
+---------+----------------+--------------------------------------+--------+---------+
```
<br>

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



<br>

## SAN編
### WorkerNode準備
<参考> https://netapp-trident.readthedocs.io/en/stable-v21.01/kubernetes/operations/tasks/worker.html?highlight=iscsi-utils

・iSCSIモジュールインストール
```
sudo yum install -y lsscsi iscsi-initiator-utils sg3_utils device-mapper-multipath
rpm -q iscsi-initiator-utils
```
iscsi-initiator-utils version が 6.2.0.874-2.el7 以上であることを確認。

・iSCSIセットアップ
```
sudo sed -i 's/^\(node.session.scan\).*/\1 = manual/' /etc/iscsi/iscsid.conf
sudo mpathconf --enable --with_multipathd y
sudo systemctl enable --now iscsid multipathd
sudo systemctl enable --now iscsi
```
<br>

### ONTAP SAN(iSCSI)設定
・portset作成
```
AFF8040::*> portset create -vserver ti_Trident_svm -portset portset1 -protocol iscsi -port-name trident-iscsi-lif

AFF8040::*> portset show
Vserver   Portset      Protocol Port Names              Igroups
--------- ------------ -------- ----------------------- ------------
ti_Trident_svm
          portset1     iscsi    trident-iscsi-lif       -
```
・igroup作成、portsetを紐づけ
※ backendの設定で指定できるが、何もなければigroup名は"trident"にすること
```          
AFF8040::*> igroup create -vserver ti_Trident_svm -igroup trident -protocol iscsi -ostype linux -portset portset1

AFF8040::*> igroup show
Vserver   Igroup       Protocol OS Type  Initiators
--------- ------------ -------- -------- ------------------------------------
ti_Trident_svm
          trident      iscsi    linux    -

AFF8040::*> portset show
Vserver   Portset      Protocol Port Names              Igroups
--------- ------------ -------- ----------------------- ------------
ti_Trident_svm
          portset1     iscsi    trident-iscsi-lif       trident
```          
・iSCSI有効化
```          
AFF8040::> iscsi create -vserver ti_Trident_svm

AFF8040::> iscsi show
           Target                           Target                       Status
Vserver    Name                             Alias                        Admin
---------- -------------------------------- ---------------------------- ------
ti_Trident_svm
           iqn.1992-08.com.netapp:sn.61b09dafa0d611ebbf7600a098a02fe9:vs.11
                                            ti_Trident_svm               up
```
<br>

### Backend登録
```
[root@trident-001 trident-installer]# ./tridentctl create backend -f ../backend-ontap-san_AFF8040.json
+---------------+----------------+--------------------------------------+--------+---------+
|     NAME      | STORAGE DRIVER |                 UUID                 | STATE  | VOLUMES |
+---------------+----------------+--------------------------------------+--------+---------+
| AFF8040-iSCSI | ontap-san      | f24d9355-c42e-49de-8bb9-94cf7475507e | online |       0 |
+---------------+----------------+--------------------------------------+--------+---------+
[root@trident-001 trident-installer]#
[root@trident-001 trident-installer]# ./tridentctl get backend
+---------------+----------------+--------------------------------------+--------+---------+
|     NAME      | STORAGE DRIVER |                 UUID                 | STATE  | VOLUMES |
+---------------+----------------+--------------------------------------+--------+---------+
| AFF8040       | ontap-nas      | bf432fe1-9a9d-46cc-b9d5-30eaf7c26f35 | online |      10 |
| AFF8040-iSCSI | ontap-san      | f24d9355-c42e-49de-8bb9-94cf7475507e | online |       0 |
+---------------+----------------+--------------------------------------+--------+---------+
```

★ ONTAP側で iscsi create を実施せずにtridentctl cerate backendを実行しまった際のエラー

>Error: could not create backend: problem initializing storage driver 'ontap-san': error initializing ontap-san driver: error checking default initiator's auth type: API status: failed, Reason: entry doesn't exist, Code: 15661 (400 Bad Request)

<br>

### StorageClass作成
StorageClass(ontap-iscsi)定義は以下を利用
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ontap-iscsi
provisioner: csi.trident.netapp.io
parameters:
  backendType: "ontap-san"
  fsType: "ext4"
```
```
# kubectl create -f storage-class_ontap-san.yaml
storageclass.storage.k8s.io/ontap-iscsi created

# kubectl get sc
NAME          PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
ontap-gold    csi.trident.netapp.io   Delete          Immediate           false                  64d
ontap-iscsi   csi.trident.netapp.io   Delete          Immediate           false                  2m22s
```

※ ちなみに、ontap-sanの場合、RWX（readWriteMany）は利用不可
<br>

<details>
  <summary> 色々エラーで試行錯誤 編（クリックで展開）</summary>
<br>
★エラーが出たので、LIFを追加
(LIFがVolume/LUNが自動作成される側にいなかった模様)
>  Warning  ProvisioningFailed  2m50s (x3 over 3m)  csi.trident.netapp.io_trident-csi-6649bc485c-lt2m7_275175c3-243e-47f2-aa95-d4934fe548f3  failed to provision volume with StorageClass "ontap-iscsi": rpc error: code = Unknown desc = encountered error(s) in creating the volume: [Failed to create volume pvc-459f109b-6682-4ea5-93a5-b44ffddb63b5 on storage pool aggr1_n1 from backend AFF8040-iSCSI: problem mapping LUN /vol/trident_pvc_459f109b_6682_4ea5_93a5_b44ffddb63b5/lun0: results: {http://www.netapp.com/filer/admin results}
status,attr: failed
reason,attr: The node "AFF8040-02" has no LIFs configured with the iSCSI or FCP protocol for Vserver "ti_Trident_svm".
errno,attr: 18619
lun-id-assigned: nil
]

```
AFF8040::> portset add -vserver ti_Trident_svm -portset portset1 -port-name trident-iscsi-lif2

AFF8040::> portset show
Vserver   Portset      Protocol Port Names              Igroups
--------- ------------ -------- ----------------------- ------------
ti_Trident_svm
          portset1     iscsi    trident-iscsi-lif, trident-iscsi-lif2
                                                        trident
```

★さらにエラー。
PVCがBoundになっても、initiator云々でPodにmountされない

>  Warning  FailedAttachVolume  20s (x7 over 52s)  attachdetach-controller  AttachVolume.Attach failed for volume "pvc-6a982bd5-3fd7-43ee-ac64-6d7d93ea6a29" : rpc error: code = Internal desc = error publishing ontap-san driver: unknown initiator for node trident-004

以下の記載があるので今回Helmでインストールした新しいVer.なので大丈夫という認識だが、ONTAP側でiqnが登録されていない。
https://netapp-trident.readthedocs.io/en/stable-v21.01/kubernetes/operations/tasks/backends/ontap/ontap-san/preparing.html

>If Trident is configured to function as a CSI Provisioner, Trident manages the addition of IQNs from worker nodes when mounting PVCs. As and when PVCs are attached to pods running on a given node, Trident adds the node’s IQN to the igroup configured in your backend definition.
>If Trident does not run as a CSI Provisioner, the igroup must be manually updated to contain the iSCSI IQNs from every worker node in the Kubernetes cluster. The igroup needs to be updated when new nodes are added to the cluster, and they should be removed when nodes are removed as well.


とりあえず手で追加
```
AFF8040::> igroup add -vserver ti_Trident_svm -igroup trident -initiator iqn.1994-05.com.redhat:131af07c20d

AFF8040::> igroup add -vserver ti_Trident_svm -igroup trident -initiator iqn.1994-05.com.redhat:56b8196a93f9

AFF8040::> igroup add -vserver ti_Trident_svm -igroup trident -initiator iqn.1994-05.com.redhat:ccb2a658634

AFF8040::>
AFF8040::> igroup show
Vserver   Igroup       Protocol OS Type  Initiators
--------- ------------ -------- -------- ------------------------------------
ti_Trident_svm
          trident      iscsi    linux    iqn.1994-05.com.redhat:131af07c20d
                                         iqn.1994-05.com.redhat:56b8196a93f9
                                         iqn.1994-05.com.redhat:ccb2a658634
```

事象変わらずなので、trident側の設定が間違っていそうな？
エラーメッセージの一部"unknown initiator for node"とかでググるとGitHubで公開されているソースコードが引っ掛かったので、git cloneしてソースコードを読んでいると、NodePrepとかいう名前がちょいちょい関連してくるのでどうもAutomatic worker node preparationというBeta機能が怪しいんじゃ？ということで、Tridentをインストールしなおし。
この際、--set enable-node-prep=false を付けることで、その機能を明示的に無効化。
（Beta機能がデフォルト有効なの！？）
```
# helm install trident -n trident trident-operator-21.01.2.tgz --set enable-node-prep=false
NAME: trident
LAST DEPLOYED: Thu Jun 24 17:06:53 2021
NAMESPACE: trident
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing trident-operator, which will deploy and manage NetApp's Trident CSI
storage provisioner for Kubernetes.

Your release is named 'trident' and is installed into the 'trident' namespace.
Please note that there must be only one instance of Trident (and trident-operator) in a Kubernetes cluster.

To configure Trident to manage storage resources, you will need a copy of tridentctl, which is
available in pre-packaged Trident releases.  You may find all Trident releases and source code
online at https://github.com/NetApp/trident.

To learn more about the release, try:

  $ helm status trident
  $ helm get all trident
```
これにて問題なくマウントされるようになった。
</details>
<br>

PVCがデプロイされてBoundになると、ONTAP側は自動でigroupにinitiatorが登録されている。当然、Volume, LUNも自動作成される。
```
AFF8040::> igroup show
Vserver   Igroup       Protocol OS Type  Initiators
--------- ------------ -------- -------- ------------------------------------
ti_Trident_svm trident iscsi    linux    iqn.1994-05.com.redhat:131af07c20d
                                         iqn.1994-05.com.redhat:53a588121c67
                                         iqn.1994-05.com.redhat:56b8196a93f9
                                         iqn.1994-05.com.redhat:91b8fe84d7a7
                                         iqn.1994-05.com.redhat:99d25bf8406a
                                         iqn.1994-05.com.redhat:ccb2a658634
```
initiatorを確認するとWorkerNodeだけでなくMasterNodeの分も登録されている。
<br>
```
AFF8040::> volume show -vserver ti_Trident_svm
Vserver   Volume       Aggregate    State      Type       Size  Available Used%
--------- ------------ ------------ ---------- ---- ---------- ---------- -----
ti_Trident_svm nfs_test aggr1_n1    online     RW         10GB     9.50GB    0%
ti_Trident_svm ti_Trident_svm_root aggr1_n2 online RW      1GB    972.0MB    0%
（省略）
ti_Trident_svm trident_pvc_3926023c_2e5e_4150_9cb7_7d6025432567 aggr1_n1 online RW 20GB 19.99GB  0%
ti_Trident_svm trident_pvc_96ac2fb3_85dd_4587_935b_d1b518d6ee80 aggr1_n1 online RW 20GB 20.00GB  0%
ti_Trident_svm trident_pvc_c858f867_a3cb_4e83_9213_a76946caafac aggr1_n1 online RW 20GB 19.98GB  0%
（省略）
15 entries were displayed.
```
```
AFF8040::> lun show -vserver ti_Trident_svm
Vserver   Path                            State   Mapped   Type        Size
--------- ------------------------------- ------- -------- -------- --------
ti_Trident_svm /vol/trident_pvc_3926023c_2e5e_4150_9cb7_7d6025432567/lun0 online mapped linux 20GB
ti_Trident_svm /vol/trident_pvc_96ac2fb3_85dd_4587_935b_d1b518d6ee80/lun0 online mapped linux 20GB
ti_Trident_svm /vol/trident_pvc_c858f867_a3cb_4e83_9213_a76946caafac/lun0 online mapped linux 20GB
3 entries were displayed.
```

<br>

## Tips/その他
##### インストール周り
・Trident v21.04は  K8s v1.21.1 に対応してないのか、Helmでのインストールで、はじかれはしないがOperator以外のPodとか色々必要なものがデプロイされない
・helm upgrade tridentとやると、Chartにあるからかパッケージを指定しなくても勝手にUpgradeされたが、Operatorが「ImagePullBackOff」なってたので、やらない方が良い。
<br>

##### トラブルシューティング方法
以下を参照
・NetApp KB
・GitHub Tridentのレポジトリ
　　　issue, コード自体
意外とエラーメッセージでコードを読むと、はまってる事象にあたりがついたりした
・関連オブジェクトのdescribe, logs等を確認する
・https://netapp-trident.readthedocs.io/en/latest/kubernetes/troubleshooting.html
（公式サイト/Troubleshooting）
<br>

##### PVがDeleteされるとONTAP上でも削除される
⇒ *Storage Class*の設定がデフォルトで*RECLAIMPOLICY*が*DELETE*になっているかと思われます
<br>

##### リソース要件は？
⇒ 明示的な記載なし…？要確認
<br>

##### CSIでのNetApp利用
PVCの定義内で、上記で作成した*Storage Class*(この例だとontap-gold/ontap-iscsi)を指定することで、バックエンドとして登録したONTAP/SVM上にVolumeやLUNが作成されるようになります。
