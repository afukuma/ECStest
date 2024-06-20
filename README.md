# ECStest

```bash
myip=$(curl https://checkip.amazonaws.com)
aws cloudformation deploy --template-file ./ecs01.yaml --stack-name ecs01 \
--capabilities CAPABILITY_NAMED_IAM --no-execute-changeset \
--parameter-overrides MyIP=$myip

```

# Memo
## CodeCommit
### CodeCommit x IIC (SSO)

[AWS CLIでAWS SSOを使用する](https://qiita.com/hapoon/items/ed48a99e2d6c8946756e)

```aws configure sso``` コマンドでブラウザ経由で認可させる

ウィザードを進めていけばconfigファイルにProfileが作成される 以下参照

```.aws/config
[profile skya]
sso_session = my-sso
sso_account_id = 905********
sso_role_name = AWSAdministratorAccess
region = ap-northeast-1
output = json
[sso-session my-sso]
sso_start_url = https://d-9*****.awsapps.com/start
sso_region = ap-northeast-1
sso_registration_scopes = sso:account:access
```

gitconfigに以下設定

```.gitconfig
[credential "https://git-codecommit.ap-northeast-1.amazonaws.com"]
    helper = !aws --profile skya codecommit credential-helper $@
    UseHttpPath = true
```

#### 使い方
aws sso login --profile <プロファイル名>

git clone https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/<リポジトリ名>
認証ウィンドウが表示されるが、✕で終了






## Dockerfile
https://blog.serverworks.co.jp/NewRelicAndDockerOnEC2_1#Dockerfileindexhtml%E3%81%AE%E4%BD%9C%E6%88%90

```Dockerfile
FROM amazonlinux:2023
RUN yum update -y \
    && yum install httpd -y
# ポートの開放
EXPOSE 80
# Docker上のディレクトリにローカルファイルをコピー
COPY index.html /var/www/html/index.html
# コンテナ起動時に実行されるコマンド
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

```html: index.html
<html>
    <body>
        <h1>RUNNING !!</h1>
    </body>
</html>
```




## ECS TaskDefinition 更新方法検討
### [x] Webアプリ CFN管理としてパラメータによりタスク定義を設定する
Webアプリは、ECSサービスでリビジョンを指定する

WebアプリはCFNで設定する

+ CFnのパラメータで、TaskDefに設定するコンテナタグなどを投入して最新化する
[CloudformationでFargateを構築する 2019.02](https://dev.classmethod.jp/articles/cloudformation-fargate/#toc-5)

- ECSTaskName
- ECSTaskCPUUnit
- ECSTaskMemory
- ECSImageName

+ AWS管理コンソールでタスク定義を追加してもCFN側で検知しない。
+ リビジョン追加してサービスで新しいリビジョンを指定して、からのCFN更新→　戻らない

タスク定義を直接修正（リビジョン追加）してもらってもデグレすることはない
よって、AWS管理コンソールを使ったバージョン変更を基本として、CFNパラメータで必要箇所のみ修正できるものとする

+ CFNでタスク定義修正 → ECSサービス参照先も変更される

#### CICD検討

ECRの指定リビジョンが更新された時、aws cfn deploy にパラメータにImageタグを設定して実行

を指定時刻（夜間）に実施する


### [ ] バッチ CFN対象外にする
バッチは、SFNからタスクリビジョンを指定してRunTaskを実行する

タスク定義は単独で作成する
1. JSONファイルを準備する
2. CLIでリビジョン作成する
3. SFNで指定する（LATESTとしておけば変更不要）

+ ECSService の TaskDefinition は必須。-> 必要なタスク定義はCFNで準備しておく必要がある
  しかも1：１orz

https://qiita.com/RyoMar/items/06e23d60d9df2d955221
サービスはいらない様子



[register-task-definition](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/register-task-definition.html)
[describe-task-definitionで取得したJSONはそのままではregister-task-definitionで登録できないお話](https://dev.classmethod.jp/articles/describe-task-definition-to-register-task-definition/)

-> WebアプリだとNG: Serviceからどのリビジョンを指定するかわからないので

#### TODO
+ １サービス内に複数タスクが起動できるか？？  -> サービス不要 Cluster＋TaskDefで構成
+ 処理終了後にタスクが終わるか？  -> する
 
AL2023 でコマンドをセットしたタスク定義を複数準備する（CLIで作成する）
aws s3 で標準出力をオブジェクト出力するコマンド
aws cliでruntaskを実行

```bash
CLUSTER_NAME=BatchCluster
SUBNET_ID=subnet-
SECURITY_GROUP_ID=sg- 
TASK_DEF_ARN=arn:aws:ecs:ap-northeast-1:463389754164:task-definition/task-definition-batch01:2

aws ecs run-task --cluster $CLUSTER_NAME --task-definition $TASK_DEF_ARN \
  --network-configuration awsvpcConfiguration="{subnets=[\"$SUBNET_ID\"],securityGroups=[\"$SECURITY_GROUP_ID\"],assignPublicIp=ENABLED}"  --launch-type FARGATE

```

[Step Functions で Amazon ECS または Fargate タスクを管理する](https://docs.aws.amazon.com/ja_jp/step-functions/latest/dg/connect-ecs.html#connect-ecs-pass-to)

Commandの上書きが可能な様子
```json
{
 "StartAt": "Run an ECS Task and wait for it to complete",
 "States": {
   "Run an ECS Task and wait for it to complete": {
     "Type": "Task",
     "Resource": "arn:aws:states:::ecs:runTask.sync",
     "Parameters": {
                "Cluster": "cluster-arn",
                "TaskDefinition": "job-id",
                "Overrides": {
                    "ContainerOverrides": [
                        {
                            "Name": "container-name",
                            "Command.$": "$.commands" 
                        }
                    ]
                }
            },
     "End": true
    }
  }
}
```

CloudTrailでのECR Pullイベントが非常におぽくなり、GuardDutyのコストが高くなった

https://speakerdeck.com/hisamouna/shu-bai-yi-shang-noyang-nahatutikadong-iteiruhatutiji-pan-woecs-on-fargatekaraeks-on-ec2heyi-xing-sitahua?slide=17


## STep Functions のASLファイルはS3格納できる

⭐️更新タイミングは？？







## CloudFormationのロール
ユーザに対するロールがcfn deployのみで済む
⭐️方法を探す（あっったはず）

[CloudFormationサービスロールの概要と注意点](https://go-to-k.hatenablog.com/entry/2021/08/09/000812)

## RDS

[RDSにIAMデータベース認証でログインする](https://blog.serverworks.co.jp/rds-iamdblogin)

```bash
mysql -u [IAMデータベース認証ログインユーザ名] -h [RDSエンドポイント] -p`aws rds generate-db-auth-token --hostname [RDSエンドポイント] --port 3306 --username [IAMデータベース認証ログインユーザ名] --region ap-northeast-1` --enable-cleartext-plugin
```


### RDS Snapchot
https://qiita.com/tmiki/items/19cc4fc71949eaafbc3a
