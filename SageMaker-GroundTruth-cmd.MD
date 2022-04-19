#  GroundTruth 命令行模式完成标注任务

## 背景

数据集标柱是机器学习中不可缺少的一环，分为图像标注，文本标注，3D点云标注，视频标注。GroundTruth 是AWS的数据标注服务，支持以上标注类型，并且可以对标注团队进行端到端的全流程管理，您可以将标注任务分发给公共团队Amazon Mechanical Turk，也可以分发给自有团队。您可以方便的通过AWS Console端管理团队，管理标注任务。但在安全管理要求高的公司或组织，AWS Console端也严格受控，数据标注的管理员是没有Console端控制权限的。本文基于此背景，只需要一台EC2实例，仅通过命令行方式完成标注任务的分发和管理工作。

## PoC 任务目标

通过命令行方式完成图像bounding-box任务，标注火车与汽车。

* 创建标注团队
* 加入成员。
* 创建标注任务
* 标注成员登陆系统完成标注。


https://github.com/awsdocs/amazon-sagemaker-developer-guide/blob/master/doc_source/sms-create-labeling-job-api.md

## 准备工作

* 通过管理台创建sns topic “AmazonSageMaker-label-notify” 准备用来邮件通知使用。

将如下Access Policy 更新到sns topic “AmazonSageMaker-label-notify”

```
{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Sid": "AmazonSageMakerConfigPolicy",
      "Effect": "Allow",
      "Principal": {
        "Service": "sagemaker.amazonaws.com"
      },
      "Action": "SNS:Publish",
      "Resource": "arn:aws:sns:*:*:*AmazonSageMaker*"
    }
  ]
}
```

[Image: 截屏2022-03-22 21.15.02.png]
* 创建EC2 角色 DataLabelRole ，基于最小权限分配原则，创建GroundTruthPolicy 如下。将GroundTruthPolicy，[AmazonSageMakerFullAccess](https://us-east-1.console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/AmazonSageMakerFullAccess) 添加到DataLablelRole上

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:ListBucket"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::SageMaker"
            ]
        },
        {
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::SageMaker/*",
                "arn:aws:s3:::static-handson/*" ## 存放需要标注的数据及配置文件的bucket。
            ]
        },
        {
            "Action": [
                "sns:*"
            ],
            "Effect": "Allow",
            "Resource": "*"
        }
       
            
        
        
    ]
}
```

将DataLabelRole 绑定到某个EC2实例

```
aws ec2 associate-iam-instance-profile --instance-id i-xxxx5af07 --iam-instance-profile Name=groundTruthRole --region=us-east-1
```

## 1.创建标注团队

使用cognito服务创建user pool ， 用户组。然后创建sagemaker workforce，workteam将标注组织与cognito 服务关联。采用用户密码方式登陆。

### 创建 User Pool

创建cognito 用户池，定义用户加入时的重置密码邮件格式。


```

aws cognito-idp create-user-pool --pool-name LabelUserPool \
--username-attributes "email" `--``username``-``configuration ``CaseSensitive``=``true`` \`
`--``email``-``configuration``=``EmailSendingAccount``=``"COGNITO_DEFAULT"`


```

```
## 返回信息
{
    "UserPool": {
        "Id": "us-east-1_A9BuYVC0l",
        "Name": "LabelUserPool2",
        "Policies": {
            "PasswordPolicy": {
                "MinimumLength": 8,
                "RequireUppercase": true,
                "RequireLowercase": true,
                "RequireNumbers": true,
                "RequireSymbols": true,
                "TemporaryPasswordValidityDays": 7
            }
        },
        ... ... 
}
```

```
# 定义user-pool 环境变量
export USER_POOL=us-east-1_A9BuYVC0l
```

### 创建Group

不同的标注任务，方便管理，可以分配给不同的一组人员完成，

```
aws cognito-idp create-group --group-name "Label-group" --user-pool-id ${USER_POOL}
# 定义group_name 环境变量
export GROUP_NAME="Label-group"
```



### 创建User Pool Client

创建使用user pool的client配置，采用用户名/密码的登陆方式

```

` aws cognito``-``idp create``-``user``-``pool``-``client ``--``client``-``name ``"labelclient"``  \`
` ``--``user``-``pool``-``id ${USER_POOL}`` ``--``generate``-``secret ``--``enable``-``token``-``revocation  \`
` ``--``explicit``-``auth``-``flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH`
` `
```

```
export CLIENT_ID=3harthams75mapq0hnb3rl1gsk
```

### 创建Domain

```
aws cognito-idp create-user-pool-domain --user-pool-id ${USER_POOL}  --domain label
-workteam
```

### 创建Team

```
# 将前面创建的topic ARN copy到此处
export SNS_TOPIC="arn:aws:sns:us-east-1:845524701534:AmazonSageMaker-label-notify2"
aws sagemaker create-workteam --workteam-name "test2" --workforce-name "testforce" \
`--``notification``-``configuration ``NotificationTopicArn``=``$``{``SNS_TOPIC``}`` \`
--member-definitions=CognitoMemberDefinition="{UserPool=${USER_POOL},UserGroup=${GROUP_NAME},ClientId=${CLIENT_ID}}"  --description "labelTeam" 


```

```
# 返回信息
{
    "WorkteamArn": "arn:aws:sagemaker:us-east-1:845524701534:workteam/private-crowd/test2"
}
```

```
export WORK_TEAM="arn:aws:sagemaker:us-east-1:845524701534:workteam/private-crowd/test2"
```

### 查看获得teams的网址

```
aws sagemaker list-workteams
# resp
{
    "Workteams": [
        {
            "WorkteamName": "test",
            "MemberDefinitions": [
                {
                    "CognitoMemberDefinition": {
                        "UserPool": "us-east-1_A9BuYVC0l",
                        "UserGroup": "Label-group",
                        "ClientId": "4n2odc4odgqk01f69rcjpd71ou"
                    }
                }
            ],
            "WorkteamArn": "arn:aws:sagemaker:us-east-1:845524701534:workteam/private-crowd/test",
            "Description": "labelTeam",
            "SubDomain": "h2vaydps6l.labeling.us-east-1.sagemaker.aws",
            "CreateDate": "2022-03-16T06:21:41.374000+00:00",
            "LastUpdatedDate": "2022-03-16T06:21:41.374000+00:00",
            "NotificationConfiguration": {}
        }
    ]
}
```

```
export LOGIN_CALLBACK_URL="https://h2vaydps6l.labeling.us-east-1.sagemaker.aws/oauth2/idpresponse"
export LOGOUT_CALLBACK_URL="https://h2vaydps6l.labeling.us-east-1.sagemaker.aws/logout"
# 更新邀请邮件模版，包含用户名，初始密码，访问链接
aws cognito-idp update-user-pool --user-pool-id ${USER_POOL} --email-configuration=EmailSendingAccount="COGNITO_DEFAULT" \
--admin-create-user-config '{"AllowAdminCreateUserOnly":true,"InviteMessageTemplate":{"EmailMessage":"Your username is {username} and temporary password is {####} \n Please visit https://h2vaydps6l.labeling.us-east-1.sagemaker.aws ","EmailSubject":"You are invited to Label Project."}}'

```

### 更新cognito user client 的callback url

```
aws cognito-idp update-user-pool-client \
--user-pool-id ${USER_POOL} --client-id ${CLIENT_ID} \
--supported-identity-providers COGNITO \
--callback-urls ${LOGIN_CALLBACK_URL} --logout-urls ${LOGOUT_CALLBACK_URL} \
--refresh-token-validity 30 \
 --enable-token-revocation --explicit-auth-flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH \
 --allowed-o-auth-flows-user-pool-client --allowed-o-auth-flows code implicit \
 --allowed-o-auth-scopes email openid profile 
```

以上步骤执行完成后，不会在经常变化，除非您需要更改登录规则，登录失效时间等配置。



## 2.增加用户

```
#  2.1创建用户
aws cognito-idp admin-create-user     --user-pool-id ${USER_POOL}  \
   --username oyanli@amazon.com     --user-attributes Name=email,Value=oyanli@amazon.com Name=phone_number,Value="+13544199856"  \
   --message-action SUPPRESS `--``desired``-``delivery``-``mediums EMAIL`
# 2.2发送初始密码邮件
aws cognito-idp admin-create-user     --user-pool-id  ${USER_POOL} \
   --username oyanli@amazon.com     --user-attributes Name=email,Value=oyanli@amazon.com Name=phone_number,Value="+13544199856"  \
   --message-action RESEND --desired-delivery-mediums EMAIL
   
# 2.3查看用户id
aws cognito-idp list-users 
# 2.4将用户加到组Label-group


aws cognito-idp admin-add-user-to-group --user-pool-id ${USER_POOL}  --username 3fa0ac23-e1b8-42fa-8e3c-b685bea54bb5 --group-name Label-group
     
# 删除用户 用户管理步骤，非必须！！！
aws cognito-idp admin-delete-user --user-pool-id us-east-1_A9BuYVC0l --username xxx   
```

2.2步骤完成后，用户会收到初始密码邮件,首次登陆必须重新修改密码。
[Image: 截屏2022-03-20 下午4.07.47.png]
## 3.创建标注任务

### 数据准备

* 将需要进行标注的数据上传到s3。此案例中将待标注图片上传到S3，并将对应的manifest说明文件上传

待标注图片路径：s3://static-handson/labeling-image/
manifest说明文件路径：s3://static-handson/manifest/manifest-with-input-data.json

manifest-with-input-data.json内容示例：

```
{"source-ref": "s3://static-handson/labeling-image/train1.jpeg"}
{"source-ref": "s3://static-handson/labeling-image/train2.jpeg"}
{"source-ref": "s3://static-handson/labeling-image/crossroad-traffic.jpeg"}
```



* 将标注的UI 模版上传到S3。[参考标注UI模版](https://github.com/aws-samples/amazon-sagemaker-ground-truth-task-uis/tree/master/images)。

此demo选择 预定义label的bounding-box页面模版。并上传到`s3://static-handson/ui-template/bounding-box.liquid.html`

## 生成labels 目录

格式示例如下，把label_1,label_2... 替换为你需要的label，保存后上传到S3.

```
`{`` `
`"document-version"``:`` ``"2018-11-28"``,
"labels": [{"label": "label_1"},{"label": "label_2"},...{"label": "label_n"}]
}`
```




## 创建labeling job



```


`aws ``--``region us``-``east``-``1`` sagemaker create``-``labeling``-``job \`
`--``labeling``-``job``-``name ``"image-labeling-job20"`` \`
`--``label``-``attribute``-``name ``"label"`` \`
`--``role``-``arn ``"arn:aws:iam::845524701534:role/service-role/AmazonSageMaker-ExecutionRole-20220218T150659"`` \`
`--``input``-``config ``'{`
`        "DataAttributes": {`
`            "ContentClassifiers": [`
`                "FreeOfPersonallyIdentifiableInformation",`
`                "FreeOfAdultContent"`
`            ]`
`        },`
`        "DataSource": {`
`            "S3DataSource": {`
`                "ManifestS3Uri": "s3://static-handson/manifest/manifest-with-input-data.json"`
`            }`
`        }`
`    }'`` \`
`--``output``-``config ``'{`
`        "KmsKeyId": "",`
`        "S3OutputPath": "s3://static-handson/output-label/"`
`    }'`` \`
`--``human``-``task``-``config ``'{`
`        "AnnotationConsolidationConfig": {`
`            "AnnotationConsolidationLambdaArn": "arn:aws:lambda:us-east-1:432418664414:function:ACS-BoundingBox"`
`        },`
`        "PreHumanTaskLambdaArn": "arn:aws:lambda:us-east-1:432418664414:function:PRE-BoundingBox",`
`        "TaskAvailabilityLifetimeInSeconds": 21600,`
`        "TaskTimeLimitInSeconds": 3600,`
`        "NumberOfHumanWorkersPerDataObject": 3,`
`        "WorkteamArn": "arn:aws:sagemaker:us-east-1:845524701534:workteam/private-crowd/test2",`
`        "TaskDescription": "Bounding-box trains",`
`        "MaxConcurrentTaskCount": 10,`
`        "TaskTitle": "Bounding-box-custom-lablel image classification task",`
`        "UiConfig": {`
`            "UiTemplateS3Uri": "``s3://static-handson/ui-template/``bounding-box.liquid.html``"`
`        }`
`    }'`` \`
`--``label``-``category``-``config``-``s3``-``uri ``"s3://static-handson/ui-template/labels.json"`
```

## 查看邮件启动标注任务




