---
title: "AWSネットワーク構築を CloudFormation → CDKで“再現”する手順【初心者向け・IaC入門】"
emoji: "🌎"
type: "tech"
topics: ["aws", "cloudformation", "iac", "CDK", "初心者"]
published: true
published_at: 2025-09-08 05:00
publication_name: "secondselection"
---


## はじめに

私自身約1か月前からAWSを触り始め、GUIからはじまり様々苦戦しながらシンプルなAWS Iacでの構築の基礎ができる現在に至っています。

今回AWS CDK(Cloud Development Kit)を実際に利用し、特徴・作成過程で得た知見をアウトプットします。

以前にCloudFormationで同様の内容の構築をしたので、両者を比較した意見も交えながら、**CDKを使用する利点・再現前にまずきっちり要点を理解しておくべきだったと感じた内容**もご紹介していきます。

:::message

**本記事を通して得られること**
・AWS CDKをする利用する前に抑えておきたいポイントを知ることができる。
・初心者でもAWS CDKを使って、AWSの基礎的な環境を構築できる。

:::

### 今回CDKで作成した構成図

![画像](/images/begginer-cdk/cdk_architecture.drawio.png)

前回の記事の内容をCDKで再現してみました。
【memoです】あとからリンクを張り付けしたいと思っています。

## CloudFormationとCDKの違いは？

[【初心者向け】AWS CDK 入門！完全ガイド](https://zenn.dev/issy/articles/zenn-cdk-overview)によると、両者の特徴は下記が挙げられています。

| 特徴 | CloudFormation | AWS CDK |
| --- | --- | --- |
| 記述形式 | JSONやYAMLのみ | TypeScriptなどプログラミング言語が利用可能 |
| 定義方法 | すべて明示的に記述する必要がある | CDKライブラリによる自動生成でコード量を最小限に抑えられる |
| コード量 | 構成が複雑になると増大する | 自動生成や抽象化により抑えられる |
| 可読性 | コード量増大で低下しやすい | コメントやIDE機能により影響は限定的 |
| 開発支援 | 基本的になし | IDEでのコード補完などが利用可能 |

タイトルの通り前回CloudFormationで作成したymlファイルを、今回CDKでtypescriptを使いコーディングをして再現したので、VPCスタックを例としてコード量を比較します。

※memoです。ここに比較画像貼りたいと思っています。

### AWS CDKとは

AWS CDKは、TypeScript、Pythonなどのプログラミング言語で、AWSリソースを定義できるオープンソースフレームワークです。

>![画像](/images/begginer-cdk/cdk_graphicRecording.drawio.png)*[参照元:https://aws.amazon.com/jp/builders-flash/202309/awsgeek-aws-cdk/](https://aws.amazon.com/jp/builders-flash/202309/awsgeek-aws-cdk/)*
:::details 📝上記参照元のAWS builders.flash「使い慣れたプログラミング言語でクラウド環境を構築 ! AWS CDKをグラレコで解説」のCDK説明内容(一部抜粋)。

[AWS builders.flash 使い慣れたプログラミング言語でクラウド環境を構築 ! AWS CDK をグラレコで解説](https://aws.amazon.com/jp/builders-flash/202309/awsgeek-aws-cdk/)では下記の説明があります。

> - AWS CDKとは、開発者が自分の得意なプログラミング言語を使用してインフラを定義できるフレームワークです。
> - AWS CDK の構成要素は主に「アプリケーション」「スタック」「コンストラクト」の 3つのレイヤーに分けられます。
> - AWS CDK の中心的な概念は「コンストラクト」です。コンストラクトには 1 つまたは複数の AWS リソースを含めることができます。また、コンストラクトには別のコンストラクトを自由に含めることができるため、任意の構成をパッケージングして再利用できます。コンストラクトの使用により、共通の設定やベストプラクティスを簡単に共有でき、複雑なクラウド環境であっても効率的に構築を行えます。
>

:::

### AWS CDKの構成要素について

> - AWS CDK の構成要素は主に「アプリケーション」「スタック」「コンストラクト」の 3つのレイヤーに分けられます。 (参照元:[AWS builders.flash](https://aws.amazon.com/jp/builders-flash/202309/awsgeek-aws-cdk/))

AWS CDKでは**アプリケーションがスタックを内包し、スタックの中でコンストラクトを組み合わせてインフラを構築していく**という階層構造を持っています。

AWS CDKの核である「コンストラクト」は、1つあるいは複数のAWSリソースをまとめた部品です。このコンストラクトのまとまりが「スタック」で、デプロイ可能な最小単位です。そして、複数スタックの依存関係を定義する要素が「アプリケーション」です。

## 手順概要

今回行った手順とともに、AWS CDKで使用する主なコマンドについても説明していきます。

AWS CDKの環境構築が完了していることを前提に、作業の流れを下記に記載します。

### 1. CDKプロジェクトの初期化

任意のフォルダ内で、必要なファイルをセットアップします。

- `cdk init app --language=[language]`
- languageの部分は利用するプログラミング言語を指定 (例：`cdk init app --language typescript`)
- サンプルコード付を希望する場合のコマンド `cdk init sample-app --language=typescript`

※ 対象アカウント・リージョンにつき、初回1回のみ`cdk bootstrap`コマンドを実行してCDK環境を初期化が必要。

### 2. CDKアプリケーションの作成

初期化したプロジェクトでアプリケーションをコーディングします。

- `cdk diff {スタック名}`で差分を確認し、必要に応じて修正していきます。

![画像](/images/begginer-cdk/cdk_diff.drawio.png)*[参照元:https://speakerdeck.com/konokenj/cdk-best-practice-2024?slide=18](https://speakerdeck.com/konokenj/cdk-best-practice-2024?slide=18)*

:::message alert

#### 論理IDの変更にはご注意ください

コンストラクタの第2引数に指定する文字列をconstructIDといいます。
このconstructIDを変更したり、Constructを別の階層に移動すると論理IDが変わり、置き換えが発生します。

:::

### 3. AWSへのデプロイ

- スタックが完成すれば`cdk deploy {スタック名}`コマンドでデプロイします。

CloudFormationテンプレートが生成されたあと、デプロイ時にCloudFormationサービスへ渡されることで、AWSリソースが構築されます。

![画像](/images/begginer-cdk/cdk_deploy.drawio.png)*[参照元:https://speakerdeck.com/konokenj/cdk-best-practice-2024?slide=8](https://speakerdeck.com/konokenj/cdk-best-practice-2024?slide=8)*

## CDKアプリケーションの作成例

後述で紹介するファイルのコードは、あくまでサンプルとして作成したものであり、運用上の命名規則や各種ベストプラクティスには沿っておりません。またCloudformationのハンズオン動画のサンプルファイルを模して、データベースのパスワードもハードコーディングを行った点など、実運用での検討が必須となる項目も含まれております。そのため**学習や確認を目的とした使用に限って**ご参照ください。

### 1. アプリケーションの使用を宣言する

まずはじめにbinディレクトリのtmp.tsで「アプリケーション」の使用を宣言します。

```ts:bin/tmp.ts
import * as cdk from 'aws-cdk-lib';

const app = new cdk.App();

```

### 2. VPCスタックを作成する

- 次にVPCスタックの使用を宣言します。

![画像](/images/begginer-cdk/cdk_architecture_vpc.drawio.png)

```ts:bin/tmp.ts
import * as cdk from 'aws-cdk-lib';
import { VpcStack } from "../lib/vpc-stack"; // 追加

const app = new cdk.App();

// 追加:VPC
const vpcStack = new VpcStack(app, "VpcStack3");
```

vpc-stack.tsにスタックの詳細を記載していきます。

```ts:lib/vpc-stack.ts
import * as cdk from "aws-cdk-lib";
import { Construct } from "constructs";
import * as ec2 from "aws-cdk-lib/aws-ec2";

export class VpcStack extends cdk.Stack {
    public readonly vpc: ec2.Vpc; 

    constructor(scope: Construct, id: string, props?: cdk.StackProps) {
        super(scope, id, props); 

        // VPC
        this.vpc = new ec2.Vpc(this, "VpcStack3", {
            ipAddresses: ec2.IpAddresses.cidr("13.0.0.0/16"),
            maxAzs: 2,
            natGateways: 0, 
            subnetConfiguration: [
                {
                    name: "VpcStack3-PublicSubnet",
                    subnetType: ec2.SubnetType.PUBLIC,
                    cidrMask: 24,
                },
                {
                    name: "VpcStack3-PrivateSubnet",
                    subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
                    cidrMask: 24,
                },
            ], 
        });
    }
}

```

### 3. RDSスタックを作成する

![画像](/images/begginer-cdk/cdk_architecture_rds.drawio.png)

```ts:bin/tmp.ts
import * as cdk from 'aws-cdk-lib';

import { VpcStack } from "../lib/vpc-stack";
import { RdsStack } from "../lib/rds-stack";// 追記

const app = new cdk.App();

// VPC
const vpcStack = new VpcStack(app, "VpcStack3");

// 追記:RDS
const rdsStack = new RdsStack(app, "RdsStack3", {
    vpc: vpcStack.vpc,
});

```

rds-stack.tsにスタックの詳細を追記します。

```ts:lib/rds-stack.ts

import * as cdk from "aws-cdk-lib";
import { Construct } from "constructs";
import * as ec2 from "aws-cdk-lib/aws-ec2";
import * as rds from "aws-cdk-lib/aws-rds";

interface RdsStackProps extends cdk.StackProps {
    vpc: ec2.Vpc;
}

export class RdsStack extends cdk.Stack {
    constructor(scope: Construct, id: string, props: RdsStackProps) {
        super(scope, id, props);

        const sg = new ec2.SecurityGroup(this, "RdsSg", {
            vpc: props.vpc,
            description: 'Allow MySQL access from within VPC',
            allowAllOutbound: true,
        });
        sg.addIngressRule(ec2.Peer.ipv4("13.0.0.0/16"), ec2.Port.tcp(3306), "MySQL access");

        new rds.DatabaseInstance(this, "AppRds", {
            vpc: props.vpc,
            engine: rds.DatabaseInstanceEngine.mysql({
                version: rds.MysqlEngineVersion.VER_8_0,
            }),
            instanceType: ec2.InstanceType.of(ec2.InstanceClass.T4G, ec2.InstanceSize.MICRO),
            credentials: rds.Credentials.fromPassword(
                "dbmaster", 
                cdk.SecretValue.unsafePlainText("H&ppyHands0n") 
            ),
            databaseName: "wordpress",
            allocatedStorage: 20,
            securityGroups: [sg],
            vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
            multiAz: false,
            removalPolicy: cdk.RemovalPolicy.DESTROY,
        });
    }
}

```

### 4. EC2スタックを作成する

![画像](/images/begginer-cdk/cdk_architecture_ec2.drawio.png)

tmp.tsにEC2スタックを追記します。

```ts:bin/tmp.ts
import * as cdk from 'aws-cdk-lib';

import { VpcStack } from "../lib/vpc-stack";
import { RdsStack } from "../lib/rds-stack";
import { Ec2Stack } from "../lib/ec2-stack";// 追記

const app = new cdk.App();

// VPC
const vpcStack = new VpcStack(app, "VpcStack3");

// RDS
const rdsStack = new RdsStack(app, "RdsStack3", {
    vpc: vpcStack.vpc,
});

// 追記:EC2
const ec2Stack = new Ec2Stack(app, "Ec2Stack3", {
    vpc: vpcStack.vpc,
});


```

ec2-stack.tsにスタックの詳細を追記します。

```ts:lib/ec2-stack.ts
import * as cdk from "aws-cdk-lib";
import { Construct } from "constructs";
import * as ec2 from "aws-cdk-lib/aws-ec2";
import * as iam from "aws-cdk-lib/aws-iam";

interface Ec2StackProps extends cdk.StackProps {
    vpc: ec2.Vpc;
}

export class Ec2Stack extends cdk.Stack {
    public readonly instance: ec2.Instance;

    constructor(scope: Construct, id: string, props: Ec2StackProps) {
        super(scope, id, props);

        const role = new iam.Role(this, "EC2Role", {
            assumedBy: new iam.ServicePrincipal("ec2.amazonaws.com"),
            managedPolicies: [
                iam.ManagedPolicy.fromAwsManagedPolicyName("AmazonSSMManagedInstanceCore")
            ],
        });

        const sg = new ec2.SecurityGroup(this, "Ec2Sg", {
            vpc: props.vpc,
            allowAllOutbound: true,
        });

        sg.addIngressRule(ec2.Peer.anyIpv4(), ec2.Port.tcp(80), "HTTP access");

        const publicSubnet = props.vpc.publicSubnets[0];

        this.instance = new ec2.Instance(this, "Ec2Stack3", {
            vpc: props.vpc,
            vpcSubnets: { subnets: [publicSubnet] }, 
            instanceType: ec2.InstanceType.of(ec2.InstanceClass.T2, ec2.InstanceSize.MICRO),
            machineImage: ec2.MachineImage.genericLinux({
                'ap-northeast-1': 'ami-09ed31f8f34719e20'
            }),
            securityGroup: sg,
            role,
        });

        this.instance.userData.addCommands(
            '#! /bin/bash',
            'yum update -y',
            'amazon-linux-extras install php7.2 -y',
            'yum -y install mysql httpd php-mbstring php-xml',
            'wget http://ja.wordpress.org/latest-ja.tar.gz -P /tmp/',
            'tar zxvf /tmp/latest-ja.tar.gz -C /tmp',
            'cp -r /tmp/wordpress/* /var/www/html/',
            'touch /var/www/html/.check_alive',
            'chown apache:apache -R /var/www/html',
            'systemctl enable httpd.service',
            'systemctl start httpd.service'
        );
    }
}
```

### 5. ELBスタックを作成する

![画像](/images/begginer-cdk/cdk_architecture_elb.drawio.png)

tmp.tsにELBスタックを追記します。

```ts:bin/tmp.ts
import * as cdk from 'aws-cdk-lib';

import { VpcStack } from "../lib/vpc-stack";
import { RdsStack } from "../lib/rds-stack";
import { Ec2Stack } from "../lib/ec2-stack";
import { ElbStack } from "../lib/elb-stack";// 追記:ELB

const app = new cdk.App();

// VPC
const vpcStack = new VpcStack(app, "VpcStack3");

// RDS
const rdsStack = new RdsStack(app, "RdsStack3", {
    vpc: vpcStack.vpc,
});

// EC2
const ec2Stack = new Ec2Stack(app, "Ec2Stack3", {
    vpc: vpcStack.vpc,
});

// 追記:ELB 
const elbStack = new ElbStack(app, "ElbStack3", {
    vpc: vpcStack.vpc,
    ec2Instance: ec2Stack.instance,
});

```

elb-stack.tsをスタックの詳細を記載します。

```ts:lib/elb-stack.ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as elbv2Targtes from 'aws-cdk-lib/aws-elasticloadbalancingv2-targets'

interface ElbStackProps extends cdk.StackProps {
    vpc: ec2.Vpc;
    ec2Instance: ec2.Instance;
}

export class ElbStack extends cdk.Stack {
    constructor(scope: Construct, id: string, props: ElbStackProps) {
        super(scope, id, props);

        const lbSecurityGroup = new ec2.SecurityGroup(this, 'elbSg', {
            vpc: props.vpc,
            description: 'Allow HTTP access for elb',
            allowAllOutbound: true,
        });

        lbSecurityGroup.addIngressRule(
            ec2.Peer.anyIpv4(),
            ec2.Port.tcp(80),
            'Allow elb HTTP from anywhere for elb'
        );

        const alb = new elbv2.ApplicationLoadBalancer(this, 'FrontLB', {
            vpc: props.vpc,
            internetFacing: true,
            securityGroup: lbSecurityGroup,
            vpcSubnets: { subnetType: ec2.SubnetType.PUBLIC }, 
        });

        const targetGroup = new elbv2.ApplicationTargetGroup(this, 'FrontLBTargetGroup', {
            vpc: props.vpc,
            port: 80,
            protocol: elbv2.ApplicationProtocol.HTTP,
            healthCheck: {
                path: '/.check_alive',
                interval: cdk.Duration.seconds(30),
            },
            targetGroupName: `${id}-tg`,
        });

        targetGroup.addTarget(new elbv2Targtes.InstanceTarget(props.ec2Instance, 80));

        alb.addListener('FrontLBListener', {
            port: 80,
            defaultTargetGroups: [targetGroup],
        });

        new cdk.CfnOutput(this, "ElbStack3-FrontLBEndpoint", {
            value: alb.loadBalancerDnsName,
            exportName: "ElbStack3-elb-Endpoint", 
        });
    }
}

```

## さいごに

あくまで個人の意見ですが、今回AWS CDKで環境構築をしてみて、CloudFormationより難しいけど楽しいなあと感じました。

CloudFormationは必要な項目をymlファイルで必要なパラメータを設定していく要素が多かったです。
一方でCDKはコンストラクトの機能を選び、テンプレートを自動生成する機能の恩恵を使うことが本当に正しいか見極めたり確認しながら、組み上げていく感覚を持てる点に面白さがありました。
（AWSにほんの少し慣れてきて、小さな成功体験が増えたことで目を向けられることの幅が少しずつ広がっていることも、楽しさに比例している可能性も否めません）

とはいえ、AWSさんが出しているベストプラクティスにはまだまだ程遠いですし、前述した要点も抑えられていなかったので、これからもっと精進していきます。

今回の記事が似たような立場の人が良い経験を積むことに繋がっていれば幸いです。
最後までご覧いただき、ありがとうございました。

## 参考

@[card](https://speakerdeck.com/konokenj/cdk-best-practice-2024)

@[card](https://pages.awscloud.com/rs/112-TZM-766/images/AWS-Black-Belt_2023_AWS-CDK-Basic-1-Overview_0731_v1.pdf)
