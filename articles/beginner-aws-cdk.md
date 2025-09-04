---
title: "AWSネットワーク構築を CloudFormation → CDKで“再現”する手順【初心者向け・IaC入門】"
emoji: "🌎"
type: "tech"
topics: ["aws", "CDK", "cloudformation", "初心者",  "zennfes2025infra"]
published: true
published_at: 2025-09-15 05:00
publication_name: "secondselection"
---
test

## はじめに

私自身約1か月前からAWSを触り始め、GUIからはじめAWS Iacでの構築の基礎はできるようになりました。
今回はAWS CDK(Cloud Development Kit)を実際に利用したので、構築する過程で得た知識をアウトプットします。

以前にCloudFormationで同様の内容の構築をしたため比較も交えながら、**AWS CDKを使用するメリット・使用前に抑えておきたかったポイント**もご紹介していきます。

:::message

**本記事を通して得られること**
・AWS CDKを使用するうえで抑えておきたいポイントを知ることができる。
・初心者でもAWS CDKを使って、AWSの基礎的な環境を構築できる。

:::

### 今回AWS CDKで作成した構成図

前回CloudFormationで作成した環境を、今回CDKで再現してみました。

VPC内にパブリックサブネットとプライベートサブネットを配置し、パブリックサブネットには外部公開用のEC2、プライベートサブネットにはDBを設置します。外部からのアクセスはELBを経由してEC2へ振り分けられる構成です。
![画像](/images/begginer-cdk/cdk_architecture.drawio.png)

CloudFormationについて確認したい方やネットワークの基礎について、前回作成した記事には記載してますので、必要に応じて下記はご参照ください。
@[card](https://zenn.dev/secondselection/articles/beginner-aws-cfn)

## CloudFormationとAWS CDKの違いは？

AWS CDKの詳しい説明の前に、まずはCloudFormationと比較しながら概要に触れていきます。

[【初心者向け】AWS CDK 入門！完全ガイド](https://zenn.dev/issy/articles/zenn-cdk-overview)によると、両者の特徴は下記が挙げられています。

| 特徴 | CloudFormation | AWS CDK |
| --- | --- | --- |
| 記述形式 | JSONやYAMLのみ | TypeScriptなどプログラミング言語が利用可能 |
| 定義方法 | すべて明示的に記述する必要がある | CDKライブラリによる自動生成でコード量を最小限に抑えられる |
| コード量 | 構成が複雑になると増大する | 自動生成や抽象化により抑えられる |
| 可読性 | コード量増大で低下しやすい | コメントやIDE機能により影響は限定的 |
| 開発支援 | 基本的になし | IDEでのコード補完などが利用可能 |

上記の**コード量**に関する記述は私も使用して実感できた点なので、具体例としてVPCスタックを作成するファイルの内容を確認します。

前回CloudFormationで作成したymlファイルと、今回AWS CDKで作成したコード量を比較しました。
**CloudFormationでは128行で構築した内容が、AWS CDKで書くと47行で済み、記載量を半分以上削減できました。**

:::details CloudFromationで作成したymlファイルの内容です。

```yml
AWSTemplateFormatVersion: 2010-09-09
Description: Hands-on template for VPC

Resources:
  CFnVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      InstanceTenancy: default
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: handson-cfn

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock: 10.0.0.0/24
      VpcId: !Ref CFnVPC
      AvailabilityZone: !Select [ 0, !GetAZs ]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: PublicSubnet1

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock: 10.0.1.0/24
      VpcId: !Ref CFnVPC
      AvailabilityZone: !Select [ 1, !GetAZs ]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: PublicSubnet2

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock: 10.0.2.0/24
      VpcId: !Ref CFnVPC
      AvailabilityZone: !Select [ 0, !GetAZs ]
      Tags:
        - Key: Name
          Value: PrivateSubnet1

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock: 10.0.3.0/24
      VpcId: !Ref CFnVPC
      AvailabilityZone: !Select [ 1, !GetAZs ]
      Tags:
        - Key: Name
          Value: PrivateSubnet2

  CFnVPCIGW:
    Type: AWS::EC2::InternetGateway
    Properties: 
      Tags:
        - Key: Name
          Value: handson-cfn

  CFnVPCIGWAttach:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties: 
      InternetGatewayId: !Ref CFnVPCIGW
      VpcId: !Ref CFnVPC

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref CFnVPC
      Tags:
        - Key: Name
          Value: Public Route

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: CFnVPCIGW
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref CFnVPCIGW

  PublicSubnet1Association:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable

  PublicSubnet2Association:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable

Outputs:
  VPCID:
    Description: VPC ID
    Value: !Ref CFnVPC
    Export:
      Name: !Sub ${AWS::StackName}-VPCID

  PublicSubnet1:
    Description: PublicSubnet1
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub ${AWS::StackName}-PublicSubnet1

  PublicSubnet2:
    Description: PublicSubnet2
    Value: !Ref PublicSubnet2
    Export:
      Name: !Sub ${AWS::StackName}-PublicSubnet2

  PrivateSubnet1:
    Description: PrivateSubnet1
    Value: !Ref PrivateSubnet1
    Export:
      Name: !Sub ${AWS::StackName}-PrivateSubnet1

  PrivateSubnet2:
    Description: PrivateSubnet2
    Value: !Ref PrivateSubnet2
    Export:
      Name: !Sub ${AWS::StackName}-PrivateSubnet2


```

:::

なぜコード量が削減できる理由も把握していただけるよう、このあとAWS CDKの特徴・仕組みをお伝えしていきます。

### AWS CDKとは

AWS CDKは、TypeScript、Pythonなどのプログラミング言語で、AWSリソースを定義できるオープンソースフレームワークです。

>![画像](/images/begginer-cdk/cdk_graphicRecording.drawio.png)*[参照元:https://aws.amazon.com/jp/builders-flash/202309/awsgeek-aws-cdk/](https://aws.amazon.com/jp/builders-flash/202309/awsgeek-aws-cdk/)*
> こちらの参照元のサイトが一番概要を掴みやすかったです。

### AWS CDKの構成要素について

> - AWS CDK の構成要素は主に「アプリケーション」「スタック」「コンストラクト」の 3つのレイヤーに分けられます。 (参照元:[AWS builders.flash](https://aws.amazon.com/jp/builders-flash/202309/awsgeek-aws-cdk/))

AWS CDKでは**アプリケーションがスタックを内包し、スタックの中でコンストラクトを組み合わせてインフラを構築していく**という階層構造を持っています。

AWS CDKの核である「コンストラクト」は、1つあるいは複数のAWSリソースをまとめた部品です。このコンストラクトのまとまりが「スタック」で、デプロイ可能な最小単位です。そして、複数スタックの依存関係を定義する要素が「アプリケーション」です。

コンストラクトにはレベルがあり、詳しい説明は下記のように書かれていました。
![画像](/images/begginer-cdk/cdk_constract.drawio.png)*[参照元:https://blog.usize-tech.com/aws-cdk-construct/](https://blog.usize-tech.com/aws-cdk-construct/)*

私は最初コンストラクトの優位性が理解しきれておらず、危うくAWS CDKの恩恵を享受しながらコードを書けていませんでした。コンストラクトの種類を理解したうえで、選定することでコード量の削減もできるようになったので、冒頭にお伝えしたポイントは**コンストラクトを使いこなすこと**ではないかと考えています。

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

※　以下に記載しているコードは、あくまでサンプルとしてテスト作成したものであり、運用上の命名規則や各種ベストプラクティスには沿っておりません。データベースのパスワードもハードコーディングを行っている点など、実運用での検討が必要な項目も含まれております。そのため**学習や確認を目的とした使用に限って**ご参照ください。

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
記載後`cdk diffコマンド`を実行し、内容を確認して問題なければ`cdk deployコマンド`を実行します。

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
            ipAddresses: ec2.IpAddresses.cidr("10.0.0.0/16"),
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

#### 💡コンストラクトがポイントということを前述したので、ここで少しだけ深掘りします

以下の画像はVpcStackのデプロイ前にcdk diffコマンドで内容を確認時のキャプチャーです。
![画像](/images/begginer-cdk/cdk_constract_diff.drawio.png)

抽象度の低いコンストラクトを使用した際は、黄色枠線のみが更新される形だったので、他のコンストラクターの追記をしなければスタックが構築できないという状況に遭遇しました。
そこで、コンストラクタを変更することで自動作成の恩恵を受けることができ、追記も不要となり、一気にリソースの設定も進み利便性を感じることができました。

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
記載後`cdk diffコマンド`を実行し、内容を確認して問題なければ`cdk deployコマンド`を実行します。

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
        sg.addIngressRule(ec2.Peer.ipv4("10.0.0.0/16"), ec2.Port.tcp(3306), "MySQL access");

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

tmp.tsにEC2スタックの詳細を追記します。

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
記載後`cdk diffコマンド`を実行し、内容を確認して問題なければ`cdk deployコマンド`を実行します。

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

elb-stack.tsにスタックの詳細を記載します。
記載後`cdk diffコマンド`を実行し、内容を確認して問題なければ`cdk deployコマンド`を実行します。

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

あくまで個人の意見ですが、今回AWS CDKで環境構築をしてみて、CloudFormationより難しいけど楽しいし便利だなあと感じています。

CloudFormationは必要な項目をymlファイルで必要なパラメータをすべて記載し、アップロードを行う流れでした。一方でAWS CDKはどのコンストラクトを使用するか選び、テンプレートが自動生成される恩恵を使いながらCLI上で確認を挟みながら組み立てていける点に面白さがありました。
（AWSにほんの少し慣れてきて、小さな成功体験が増えたことで目を向けられることの幅が少しずつ広がっていることも、楽しさに比例している可能性は否めません）

とはいえ、AWSさんが出しているベストプラクティスにはまだまだ程遠いですし、ポイントを抑えられていない状態で手を動かしてしまったり反省・課題は山積みなので、これからもっと精進していきます。今回の記事が似たような立場の人にとって有益な情報となっていれば幸いです。
最後までご覧いただき、ありがとうございました。

## 参考

@[card](https://speakerdeck.com/konokenj/cdk-best-practice-2024)

@[card](https://pages.awscloud.com/rs/112-TZM-766/images/AWS-Black-Belt_2023_AWS-CDK-Basic-1-Overview_0731_v1.pdf)
