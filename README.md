# ECStest

```bash
myip=$(curl https://checkip.amazonaws.com)
aws cloudformation deploy --template-file ./ecs01.yaml --stack-name ecs01 \
--capabilities CAPABILITY_NAMED_IAM --no-execute-changeset \
--parameter-overrides MyIP=$myip

```

# Memo
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

+ ECSService の TaskDefinition は必須ではない。TaskDefは Cfn対象外として運用範疇とする

[register-task-definition](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/register-task-definition.html)
[describe-task-definitionで取得したJSONはそのままではregister-task-definitionで登録できないお話](https://dev.classmethod.jp/articles/describe-task-definition-to-register-task-definition/)

-> WebアプリだとNG: Serviceからどのリビジョンを指定するかわからないので


## STep Functions のASLファイルはS3格納できる

⭐️更新タイミングは？？







## CloudFormationのロール
ユーザに対するロールがcfn deployのみで済む
⭐️方法を探す（あっったはず）
 



