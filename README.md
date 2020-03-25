# kubeflow-1.0-on-aws
kubeflow1.01をAWS上で構築します。
基本は[こちら](https://www.kubeflow.org/docs/aws/deploy/) に従うだけですが、コマンドの実行はdockerコンテナ上で行います。


# dockerイメージ・コンテナの用意
[こちら](https://github.com/asahi0301/eks-toolkit) を参考にして、dockerコンテナの中に入っている状態で始めます
クレデンシャルも設定しておく必要があります

# リージョンについて
kubeflowのサンプルコードが何故か オレゴンリージョンになっているので、ここでもオレゴンリージョン(us-west-2)を選択していますが、
EKSが使えるリージョンであれば、どこでも動くと思います

# 設定するパラメーターの設定
以下のコマンドを実行して、環境変数を設定しておきます。
```sh
export AWS_REGION=us-west-2
export AWS_DEFAULT_REGION=${AWS_REGION}
export AWS_CLUSTER_NAME=kubeflow
export BASE_DIR=/src

export KF_NAME=${AWS_CLUSTER_NAME}
export KF_DIR=${BASE_DIR}/${KF_NAME}
export CONFIG_FILE=${KF_DIR}/kfctl_aws.yaml
```


# eksctl で EKS クラスターの作成
以下を実行して、EKSクラスターを作成します。15分くらいかかるので気長に待ちます。
```sh
eksctl create cluster --name=${AWS_CLUSTER_NAME} --nodes=6 --managed --alb-ingress-access --region=${AWS_REGION}
```
何らかの理由で失敗した場合は、AWSの管理画面に入って、CloudFromationのstackを見ると、原因が書いてあります。
よくある原因としては、EIPやVPC数が上限に達している、設定したクレデンシャルが間違っている(権限が弱い)などです。
問題を修正後に、以下のコマンドを実行したのち

```sh
eksctl delete cluster kubeflow
```

次に、**失敗したCloudFromationのStackを削除**して、 eksctl create cluster ~~ コマンドを再実行してください。

# kubeflowのインストール(シンプル版)
cognitoを使って認証を設ける、IAM Role for Podsを使うなどありますが、ここではそれらを一切使用せずに
以下を実行してkubeflowをインストールします

```sh
# kfctlのインストール
wget https://github.com/kubeflow/kfctl/releases/download/v1.0.1/kfctl_v1.0.1-0-gf3edb9b_linux.tar.gz
tar zxvf kfctl_v1.0.1-0-gf3edb9b_linux.tar.gz
mv kfctl /usr/local/bin/

# kubeflow on aws の設定ファイル(congnito版と non congnito版の2つあり、ここでは no cognito版(無認証)を利用)
export CONFIG_URI="https://raw.githubusercontent.com/kubeflow/manifests/v1.0-branch/kfdef/kfctl_aws.v1.0.1.yaml"
export KF_NAME=${AWS_CLUSTER_NAME}

mkdir -p ${BASE_DIR}
export KF_DIR=${BASE_DIR}/${KF_NAME}

mkdir -p ${KF_DIR}
cd ${KF_DIR}

# kubeflow on aws の設定ファイルのカスタマイズ
wget -O kfctl_aws.yaml $CONFIG_URI
export CONFIG_FILE=${KF_DIR}/kfctl_aws.yaml

NodeInstanceRole=`aws iam list-roles \
    | jq -r ".Roles[] \
    | select(.RoleName \
    | startswith(\"eksctl-$AWS_CLUSTER_NAME\") and contains(\"NodeInstanceRole\")) \
    .RoleName"`

sed -i'.bak' -e 's/kubeflow-aws/'"$AWS_CLUSTER_NAME"'/' ${CONFIG_FILE}
sed -i -e 's/eksctl-kubeflow-nodegroup-ng-a2-NodeInstanceRole-xxxxxxx/'"$NodeInstanceRole"'/' ${CONFIG_FILE}
sed -i -e 's/us-west-2/'"$AWS_REGION"'/' ${CONFIG_FILE}


# kubeflowをデプロイ(デプロイ完了まで数分かかる)
cd ${KF_DIR}
kfctl apply -V -f ${CONFIG_FILE}
```


# 確認
## リソース一覧
全リソースが 正常に動いているか。なっていない場合は、時間をおいて再度確認する。
```sh
kubectl -n kubeflow get all
```
もし、時間をおいても、正常になってないものがあればログとかみてトラブルシュートが必要

# ingressの確認
Addressの部分に、ELBのDNS名が表示されるので、メモる
```sh
kubectl get ingress -n istio-system
NAME            HOSTS   ADDRESS                                                                  PORTS   AGE
istio-ingress   *       xxxxxx-istiosystem-istio-2af2-xxxxx.us-west-2.elb.amazonaws.com   80      5m46s
```

# Kubeflow UIへアクセス
WebブラウザからさきほどメモしたELBのDNS名を入力すると、KubeflowのUIにつながる
あとは遊ぶだけ。

# 注意点
デフォルトだと無認証かつHTTPかつIP制限は行っていません。
さすがに大事な情報はいれないと思うので、HTTPでもいいかと思いますが、
アドレスさえわかれば、誰でもアクセスできる状態なので、作成されたELB(ALB)に対してセキュリティグループでIP制限くらいは加えておきましょう。

# 削除
以下のコマンドで削除を行います

```sh
cd ${KF_DIR}
kfctl delete -f ${CONFIG_FILE}
eksctl delete cluster kubeflow
```
ekscltのdeleteが失敗した場合は、CloudFormationのStackを手動で消します。
IAM roleにポリシーを手動で足したり、セキュリティグループの変更を行った場合は
それすら失敗する可能性があるので、手動でもとに戻したり消したりする必要があるかもしれません。
消し忘れがあると料金がかかってしまうので、念の為、AWSの管理コンソールで削除確認をしたほうがいいかもしれません

# トラブルシューティング
## ingress に ELB の DNS名が表示されない
具体的にには以下のように、ADDRESS の部分が表示されない状態です。
AWSのコンソールを見ると、ALB自体は作成されているにも関わらず、Target Groupが作成されていない状態でした。
```sh
kubectl get ingress -n istio-system
NAME            HOSTS   ADDRESS   PORTS   AGE
istio-ingress   *                  80      
```
### 対処方法
ALBに関わる部分のyamlを delete/applyします。
```sh
cd ${KF_DIR}/kustomize/istio-ingress 
kubectl kustomize . | kubectl delete -f -
kubectl kustomize . | kubectl apply -f -
```
これで、ALBのDNS名が表示されるようになります

