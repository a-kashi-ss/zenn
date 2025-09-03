---
title: "はじめての AWSネットワーク構築を GUI → CloudFormation で“再現”する手順【初心者向け・IaC入門】"
emoji: "🏗"
type: "tech"
topics: ["aws","cloudformation","iac","vpc","初心者"]
published: true
published_at: 2025-09-08 05:00
publication_name: "secondselection"
---


## はじめに

今回、AWS上でのネットワーク環境構築をまずGUI上で行い、コードベースの構成管理(CloudFormation)に挑戦しました。
本記事では環境構築用の設定方法を理解するまでの道筋についてまとめました。
入門者の視点から、実際のハンズオンで出会った混乱や躓きも正直に共有していきます。

**対象読者**
・Webアプリケーションの開発経験はあるが、AWSを触ったことがない方。
・AWSの基礎を理解したうえで、コードベースの構成管理に挑戦したい方。

**この記事を通して得られること**
・IaCを使って、AWSの基礎的な環境を構築できる。

:::details  📝 IaCとは？

IaCとは**Infrastructure as Code**の略称です。具体的にはインフラ構築の内容をコード化して環境構築を自動化する手法で、主なメリットは以下の通りです。

- 手作業による人為的なミスを削減できる。
- 同じ環境を複製する際の作業時間を短縮できる。
- Webアプリケーション開発と同様にコードのバージョン管理ができる。

:::

## 今回構築した構成図

AWSハンズオンを参考にして、下記2つを作成しました。
![画像](/images/begginer-aws-cfn/handson_gui.drawio.png)

![画像](/images/begginer-aws-cfn/handson_cfn.drawio.png)

:::details  GUIでの構築編 (上図)  -参考ハンズオン
[Network編#1 AWS上にセキュアなプライベートネットワーク空間を作成する](https://pages.awscloud.com/JAPAN-event-OE-Hands-on-for-Beginners-Network1-2022-reg-event.html?trk=aws_introduction_page)

> まずAWSマネジメントコンソールを使用し、視覚的に理解しながら作成しました。
> VPCやサブネット、セキュリティグループなどの基礎を洗い出し理解を深めました。

:::

:::details  IaCでの構築編 (下図)  -参考ハンズオン
[AWS環境のコード管理 AWS CloudFormationでWebシステムを構築する](https://pages.awscloud.com/JAPAN-event-OE-Hands-on-for-Beginners-cfn-2022-reg-event.html?trk=aws_introduction_page)

> CloudFormationを活用して、インフラ構築のコード化と環境の自動構築ができることを確認しました。
> GUI操作で学んだことを実践でき、IaCの第一歩としては踏み出しやすかったです。

:::

:::details  Amazon EC2/Amazon S3/Amazon RDS/Amazon Systems Managerの概要。

今回のハンズオンで4サービスの概要について少しだけご紹介します。

![画像](/images/begginer-aws-cfn/category_other.drawio.png)

- **Amazon EC2**（サーバーサービス）
  - AWSで代表的な仮想サーバー。
        必要に応じてコンピューティングリソース（CPU・メモリなど）を柔軟に利用可能。

- **Amazon S3**（ストレージサービス）
  - ファイルそのものと、属性情報をセットで「オブジェクト」として保存可能。
        バックアップや静的Webサイトのホスティングなど幅広く利用されています。

- **Amazon RDS**（データベースサービス）
  - AWSが提供するフルマネージド型のリレーショナルデータベースサービス。
      バックアップ、ソフトウェア更新、スケーリングなどの運用作業を自動化可能。
      MySQL・PostgreSQL・MariaDB・Oracle・SQL Server・Auroraに対応する。

- **Amazon Systems Manager**（運用管理）
  - AWSが提供するネットワークやシステムを構成する個々の要素を監視、制御、管理が可能。

:::

## 1. GUIでの構築編

### ハンズオンの概要(Network編#1 AWS上にセキュアなプライベートネットワーク空間を作成する)

GUIでネットワークを構築し、各リソースの設定方法とVPC内外への通信方法を確認しました。

![画像](/images/begginer-aws-cfn/handson_gui.drawio.png)

#### 【GUIでの構築編の手順】

1. VPCでネットワーク作成
2. VPC内にサブネット4つ作成
3. IGWを設定
4. ルートテーブルでIGWとサブネットを接続し、パブリックサブネットを定義。
5. パブリックサブネットにリソース(EC2)を配置
  ・`パターンa` 外部から接続できるように設定する。
6. プライベートサブネットにリソース(EC2)を配置
  ・`パターンb` NATゲートウェイ経由で外部に接続できるように設定する。
  ・`パターンc` Interface型VPCエンドポイント経由でSystems Managerに接続できるように設定する。
  ・`パターンd` Gateway型VPCエンドポイント経由でS3に接続できるように設定する。

### ネットワーク系サービスについて

それぞれのサービスの役割・つながりについて触れながら、どのような手順でネットワークが安全に作成できるかについて説明していきます。

![画像](/images/begginer-aws-cfn/category_network.drawio.png)

#### 【1. ネットワーク内に区画を設ける】

AWSでネットワークを構築する際の基本となるのが **『VPC』** です。VPCはクラウド上に作られる、自分専用で他と隔離されたネットワーク空間を作ります。

VPCの中をさらに分けて使う仕組みが **『サブネット』** です。例えば、外部から通信できる領域を「パブリックサブネット」、外部から直接通信できない領域を「プライベートサブネット」として区別します。

:::message

#### セキュリティリスク対策

内部にデータベースを配置することで、インターネット経由で直接アクセスを不可とし、セキュリティリスクの回避策を取る。
（例として、IaC編の構成図のようにRDSを配置するなど）

:::

:::details  📝 クラウド環境のデータセンターの体制は...？

- **リージョン (Region)**  
  - 国や地域単位の概念です。日本では **東京 (ap-northeast-1)** や **大阪 (ap-northeast-3)** が存在します。  
  - 地理的に離れた場所にデータセンターを配置することで、災害や障害に強い設計が可能になります。  
  - 仮想マシンやストレージなどのリソースを作成する際に、どのリージョンで動かすかを決めます。  

- **アベイラビリティゾーン (Availability Zone／AZ)**  
  - リージョン内にある **1つ以上のデータセンターのグループ** です。  
  - AZは独立した電源・ネットワーク・冷却設備を持つため、1つのAZで障害が発生しても他のAZに影響が及びにくい設計になっています。  
  - 複数AZにまたがってリソースを配置することで、高可用性のシステム構築が可能です。  

:::

---

#### 【2. インターネットに接続する】

インターネットとやり取りするためには、**『インターネットゲートウェイ』**(以下**IGW**と記載) をVPCに接続します。  
IGWと接続することによって、VPC内のリソースがインターネットと通信できるようになります。  

もしプライベートサブネットから外部への通信だけを行いたい場合には、 **『NATゲートウェイ』** を利用します。
IGWとは異なり外部から直接アクセスはできません。
（例：ソフトウェアのアップデートをダウンロードのみ必要な場合など、内部から外部への通信のみに制限したいときに使用）

| 項目 | IGW | NATゲートウェイ |
| ---- | ---------------------- | ------------ |
| 通信 | 双方向（外→内、内→外） | 片方向（内→外のみの一方通行） |
| 用途 | 外部公開用のWebサーバー（EC2にパブリックIPを付与して世界中からアクセス可能） | プライベートサブネットのEC2やDBのソフト更新など |

---

#### 【3. 通信の経路を管理する】

通信をどこに送るかを決めるのが **『ルートテーブル』** です。「この宛先へ行くときはIGWを通る」「この宛先へ行くときはNATゲートウェイを通る」といったルールを、サブネットごとの通信設定として定義します。  

:::message  

サブネットに関連付けることができるルートテーブルは1つです。

一方でルートテーブル側からサブネットを複数関連付けることは可能です。

:::

:::details  📝サブネットの初期設定は...？

#### サブネットの初期設定は...？

サブネット作成直後はまず自動でメインルートテーブルに紐づけが行われます。

メインルートテーブルは、IGWへのデフォルトルートがありません。
IGWへの接続が不要なサブネット、つまりプライベートサブネットはこのままメインルートテーブルに配置します。

IGWへの接続が必要なサブネットをそれぞれパブリックルートテーブルに配置していく流れとなります。

:::

##### (1) パブリックルートテーブル

| 送信先      | ターゲット          |
| ----------- | ------------------- |
| 11.0.0.0/16 | local               |
| 0.0.0.0/0   | Internet Gateway    |

※ 構成図の`パターンa`です。

##### (2) メインルートテーブル（=プライベートルートテーブル NATゲートウェイ利用時）

| 送信先      | ターゲット          |
| ----------- | ------------------- |
| 11.0.0.0/16 | local               |
| 0.0.0.0/0   | NAT Gateway         |

※ 構成図の`パターンb`です。

![画像](/images/begginer-aws-cfn/handson_gui_gateway.drawio.png)

---

#### 【4. VPC外のAWSのサービスへ接続する】

インターネットを経由せずに、VPC内からAWSのサービスへ安全にアクセスするには **『VPCエンドポイント』** を利用して接続します。

| 種類         | 特徴                             | 構成図         |
| ------------ | -------------------------------- | ------------ |
| **Interface型VPCエンドポイント** | S3やDynamoDB以外の多くのサービスで利用可能 |`パターンc`         |
| **Gateway型VPCエンドポイント**   | S3やDynamoDB専用                  | `パターンd`       |

![画像](/images/begginer-aws-cfn/handson_gui_endpoint.drawio.png)

### リソースの設定項目について

GUIでのリソースの設定項目について、EC2の設定画面・設定項目を例としてご紹介します。

![画像](/images/begginer-aws-cfn/handson_ec2.drawio.png)

## 2. IaCでの構築編

### ハンズオンの概要(AWS環境のコード管理 AWS CloudFormationでWebシステムを構築する)

VPC内にパブリックサブネットとプライベートサブネットを配置し、パブリックサブネットには外部公開用のEC2、プライベートサブネットにはDBを設置します。外部からのアクセスはELBを経由してEC2へ振り分けられる構成です。

![画像](/images/begginer-aws-cfn/handson_cfn.drawio.png)

### CloudFormationについて

CloudFormationとは、コードからAWSサービスのリソース群を作成できるIaCのツールです。

利用するためには、リソースごとの起動方法やリソース同士の関連付け、ユーザーが入力できるパラメータなどを **『テンプレート』**（YAML/JSONファイル）の準備が必要です。  

作成したテンプレートをCloudFormationに読み込むことで、一連のリソース群が作成されます。この作成されたリソース群を **『スタック』** と呼びます。  
スタック作成と同時にテンプレートファイルで指定した環境が自動で構築されます。

![画像](/images/begginer-aws-cfn/cfn.drawio.png)

::: details  📝 テンプレート：テンプレートには9つの要素があります。

### テンプレートの要素(セクション)

次に、テンプレートに含まれる要素である「セクション」について紹介します。
セクションは大きく分けて9つあります。今回使用した3つのセクションについての説明は後述します。

```yml

AWSTemplateFormatVersion: "version date"     # テンプレートバージョン
Description:         # テンプレートの説明文
  String
Metadata:            # テンプレートに関する付加情報
  template metadata
Parameters:          # 実行時にユーザー入力を求めるパラメータを定義
  set of parameters
Mappings:            # キーと値のマッピング（条件やパラメータ値の指定に使う）
  set of mappings
Conditions:          # 条件と条件制御内容を管理
  set of conditions
Transform:           # サーバレスアプリケーションに使用。変換や SAM バージョン指定
  set of transforms
Resources:           # Amazon EC2 や Amazon RDS など、スタック構成のためのリソース定義（必須）
  set of resources
Outputs:             # スタック構築後に出力する値（例：DNS 名、IP 値など）
  set of outputs

```

:::  

::: details  📝 テンプレート：今回ハンズオンで使用した3つのセクションの概要を確認できます。

- **Resources**
  - テンプレートで定義されたリソースを記述する唯一の必須セクションです。
    EC2、RDS、S3など、スタックとして構成される各種リソースを指定し、リソースごとに決められたテンプレートを設定します。

  ```yml
  Resources:
    LogicalResourceName1:     # テンプレートで区別が可能な一意な論理IDを指定
      Type: AWS::ServiceName::ResourceType     # リソースタイプ
      Properties:     # リソースごとのプロパティ
        PropertyName1: PropertyValue1
        ...
  ```

- **Parameters**
  - スタック生成時にユーザーから入力を受け取るためのセクションです。
    使用することで、スタックを作成または更新するたびにカスタム値の入力が可能です。
    環境に応じた設定の変更が可能となり、テンプレートの柔軟性が高まります。

  ```yml
  Parameters:
    ParameterLogicalID:
      Description: Information about the parameter
      Type: DataType
      Default: value
      AllowedValues:
        - value1
        - value2
  ```

- **Outputs**
  - テンプレートをカスタマイズして、スタック作成後に返される値を定義します。
    1つのスタックで出力した値を他のスタックでの再利用できます。

  ```yml
  Outputs:
    OutputLogicalID:     # 出力データの名称
      Description: Information about the value
      Value: Value to return     # 出力する値
      Export:
        Name: Name of resource to export     # 組み込み関数を使い文字列を加工
  ```

:::

::: details  📝 テンプレート：記載方法の参照先はこちらをご確認ください。

使用するリソースタイプが決まれば、各項目は[テンプレートリファレンス](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/TemplateReference/aws-template-resource-type-ref.html)を確認していきます。

テンプレートリファレンスには**構文／プロパティ／戻り値／記載例／参考資料**が記載されていますので、参照しながら記載を進めます。

:::

::: details  📝 CloudFormation：3つの基本機能(作成・変更・削除)の補足説明です。

**基本機能**
CloudFormationには、作成を含め3つの機能があります。

- **作成**
  - テンプレートに定義された構成でスタックを自動作成
  - 並列でリソースを作成し、依存関係がある場合は自動的に解決
- **変更**
  - スタックに前回のテンプレートとの差分を適用
  - リソース変更時には、無停止変更/再起動/再作成のいずれかが発生
- **削除**
  - 依存関係を解決しつつリソースを全て削除

:::

#### 【IaCでの構築編の手順】

今回はスタックを分割し、テンプレートを4種使用しました(vpcスタック/ec2スタック/rdsスタック/elbスタック)。1～4の作業をスタックごとに繰り返し行いました。

1. 「テンプレート」ファイルをYAMLやJSONのフォーマットで作成する。
2. AWSのマネジメントコンソールのCloudFormationの管理画面に移動し、「スタックの作成」ボタンをクリックして、テンプレートをアップロードする。
3. アップロードを行ったスタック名のステータスが完了することを確認する。(CREATE_COMPLETE, UPDATE_COMPLETEと表示されます)

※ ファイルのアップロードの際にエラーが発生すれば、スタックごとの「イベント」タブから詳細を確認します。失敗したリソースやエラーメッセージが表示されるため、スタックのテンプレートやパラメータを修正するなど行い、再作成または更新します。

※後述のサンプルコードは、VpcIDとSubnetIdとEC2のインスタンスIDはXXXXXXXXXXXXXXXXXでハードコーディングしています。もしお試しされる場合は各環境に合わせた情報を記載してください。

:::details  💪 スタックの作成・更新方法についての補足。

GUI上ではスタックの作成・更新を下記のボタンから実行できます。

![画像](/images/begginer-aws-cfn/stack.drawio.png)

※ハンズオン内では、Cloud9(コードを記述・実行・デバッグできるクラウドベースの統合開発環境)を使用していました。
[2024年にCloud9の新規受付終了](https://repost.aws/ja/questions/QU0Y4weDCNQrObtlqybyYM8Q/cloud-9-%E3%81%AE%E6%96%B0%E8%A6%8F%E5%8F%97%E4%BB%98%E7%B5%82%E4%BA%86%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6)しているため、今回はGUIで行いました。

:::

:::details  💪 スタックを分割するメリットは...？

スタックを分割する理由について、変更や運用時のリスクを抑えつつ、柔軟性・再利用性・保守性を高まることがメリットのひとつかと考察しています。

ひとつの大きなスタックにすべてのリソースをまとめると、小さな変更でも全体を更新する必要があり、関係のないリソースまで影響を受ける可能性が高まります。例を挙げると、EC2インスタンスを含む場合、更新の際に再起動や再作成が行われ、サービス停止のリスクが生じます。

これに対し、スタックを役割ごとに分割し疎結合化することで、例えばネットワークやセキュリティのように安定して利用する部分は固定したまま、アプリケーション層だけを更新することが可能になります。このように分割管理を行うことで、影響範囲を限定しながら運用でき、再利用性や保守性の向上にもつながります。

:::

### テンプレート別の作業内容

:::message

テンプレートをアップロードする度に、スタックごとにGUIで正しく設定されているかどうかを洗い出して確認することをおすすめします。

:::

#### ✅ vpcスタックを作成

VPC内にパブリックサブネットとプライベートサブネットを配置します。

![画像](/images/begginer-aws-cfn/handson_cfn1_vpc.drawio.png)

:::details  📝vpc.yml

```yml:vpc.yml
AWSTemplateFormatVersion: 2010-09-09
Description: Hands-on template for VPC

Resources:
  CFnVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 11.0.0.0/16
      InstanceTenancy: default
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: handson-cfn

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock: 11.0.0.0/24
      VpcId: !Ref CFnVPC
      AvailabilityZone: !Select [ 0, !GetAZs ]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: PublicSubnet1

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock: 11.0.1.0/24
      VpcId: !Ref CFnVPC
      AvailabilityZone: !Select [ 1, !GetAZs ]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: PublicSubnet2

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock: 11.0.2.0/24
      VpcId: !Ref CFnVPC
      AvailabilityZone: !Select [ 0, !GetAZs ]
      Tags:
        - Key: Name
          Value: PrivateSubnet1

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock: 11.0.3.0/24
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

```text
☑ デプロイ後チェックリスト
・ VPC
    VPC CIDR が正しく登録されている
・ サブネット
    VPC CIDR・AZが正しく登録されている
・ ルートテーブル
    パブリックルートテーブルに 0.0.0.0/0 → IGW のルートがある
    プライベートサブネットが誤ってパブリックルートテーブルに関連付けされていない
```

#### ✅ ec2スタックを作成

パブリックサブネットに外部公開用のEC2を配置します。

![画像](/images/begginer-aws-cfn/handson_cfn2_ec2.drawio.png)

:::details  📝ec2.yml

```yml:ec2.yml
AWSTemplateFormatVersion: 2010-09-09
Description: Hands-on template for EC2

Parameters:
  VPCStack:
    Type: String
    Default: handson-cfn
  EC2AMI:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2

Resources:
  EC2WebServer01:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref EC2AMI
      InstanceType: t2.micro
      SubnetId: subnet-XXXXXXXXXXXXXXXXX
      UserData: !Base64 |
        #! /bin/bash
        yum update -y
        amazon-linux-extras install php7.2 -y
        yum -y install mysql httpd php-mbstring php-xml

        wget http://ja.wordpress.org/latest-ja.tar.gz -P /tmp/
        tar zxvf /tmp/latest-ja.tar.gz -C /tmp
        cp -r /tmp/wordpress/* /var/www/html/
        touch /var/www/html/.check_alive
        chown apache:apache -R /var/www/html

        systemctl enable httpd.service
        systemctl start httpd.service

  EC2SG:
    Type: AWS::EC2::SecurityGroup
    Properties: 
      GroupDescription: sg for web server
      VpcId: vpc-XXXXXXXXXXXXXXXXX
      SecurityGroupIngress:
        - IpProtocol: tcp
          CidrIp: 11.0.0.0/16
          FromPort: 80
          ToPort: 80

Outputs:
  EC2WebServer01:
    Value: !Ref EC2WebServer01
    Export:
      Name: !Sub ${AWS::StackName}-EC2WebServer01
```

:::

```text
☑ デプロイ後チェックリスト例
・ EC2インスタンス
    インスタンスが正しくで作成されている
    AMI ID が指定通りになっている
・ セキュリティグループ
    インバウンドルール：TCP 80番 (HTTP) → 11.0.0.0/16 から許可されている
    アウトバウンドルールがデフォルト（全許可）になっている
```

#### ✅ rdsスタックを作成

プライベートサブネットにはデータベースを設置します。

![画像](/images/begginer-aws-cfn/handson_cfn3_rds.drawio.png)

:::details  📝rds.yml

```yml:rds.yml
AWSTemplateFormatVersion: 2010-09-09
Description: Hands-on template for RDS

Parameters:
  VPCStack:
    Type: String
    Default: handson-cfn
  DBUser:
    Type: String
    Default: dbmaster
  DBPassword:
    Type: String
    Default: H&ppyHands0n
    NoEcho: true

Resources:
  DBInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Delete
    Properties:
      DBInstanceClass: db.t4g.micro
      AllocatedStorage: "20"
      StorageType: gp2
      Engine: MySQL
      MasterUsername: !Ref DBUser
      MasterUserPassword: !Ref DBPassword  # 実際にはNoEchoパラメータやSecrets Managerから取得する方式に置き換える
      DBName: wordpress
      BackupRetentionPeriod: 0
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups:
        - !Ref DBSecurityGroup

  DBSubnetGroup: 
    Type: AWS::RDS::DBSubnetGroup
    Properties: 
      DBSubnetGroupDescription: DB Subnet Group for Private Subnet
      SubnetIds: 
        - subnet-XXXXXXXXXXXXXXXXX
        - subnet-XXXXXXXXXXXXXXXXX
  DBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties: 
      GroupDescription: !Sub ${AWS::StackName}-MySQL
      VpcId: vpc-XXXXXXXXXXXXXXXXX
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          CidrIp: 11.0.0.0/16

Outputs:
  DBEndpoint:
    Value: !GetAtt DBInstance.Endpoint.Address
    Export:
      Name: !Sub ${AWS::StackName}-DBEndpoint
```

:::

```text
☑ デプロイ後チェックリスト例
・ RDSインスタンス
    インスタンス・エンジンが正しく作成されている
・ サブネットグループ
    プライベートサブネット2つが正しく関連付けられている
    サブネットが異なるAZに配置されているか
・ セキュリティグループ
    インバウンドルール：TCP 3306番 (MySQL) → 11.0.0.0/16 から許可されている
    アウトバウンドルールがデフォルト（全許可）になっている
```

:::details  💪 データベースのインスタンスタイプの選定方法についての補足。

RDSスタック作成時にハンズオンのサンプルファイルの内容が非対応とのエラーが出たので、データベースのインスタンスタイプを選択する機会を得ることができました。
下記のサイトにて料金を比較し選択したのですが、データベースのインスタンスがあるのか知る機会にもなったので参考として記載しておきます。

@[card](https://aws.amazon.com/jp/rds/mysql/pricing/)

:::

#### ✅ elbスタックを作成

外部からのアクセスはELBを経由してEC2へ振り分けられる構成にします。

![画像](/images/begginer-aws-cfn/handson_cfn4_elb.drawio.png)

:::details  📝elb.yml

```yml:elb.yml
AWSTemplateFormatVersion: 2010-09-09
Description: Hands-on template for ALB

Parameters:
  VPCStack:
    Type: String
    Default: handson-cfn
  EC2Stack:
    Type: String
    Default: handson-cfn-ec2

Resources:
  FrontLB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Ref AWS::StackName
      Subnets:
        - subnet-XXXXXXXXXXXXXXXXX
        - subnet-XXXXXXXXXXXXXXXXX
      SecurityGroups: 
        - !Ref SecurityGroupLB

  FrontLBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref FrontLB
      Port: 80
      Protocol: HTTP 
      DefaultActions: 
        - Type: forward
          TargetGroupArn: !Ref FrontLBTargetGroup

  FrontLBTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub ${AWS::StackName}-tg
      VpcId: vpc-XXXXXXXXXXXXXXXXX
      Port: 80
      Protocol: HTTP
      HealthCheckPath: /.check_alive
      Targets:
        - Id: i-XXXXXXXXXXXXXXXXX

  SecurityGroupLB:
    Type: AWS::EC2::SecurityGroup
    Properties: 
      GroupDescription: !Ref AWS::StackName
      VpcId: vpc-XXXXXXXXXXXXXXXXX
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

Outputs:
  FrontLBEndpoint:
    Value: !GetAtt FrontLB.DNSName
    Export:
      Name: handson-cfn-elb-Endpoint
```

:::

```text
☑ デプロイ後チェックリスト例
・ インスタンス
    指定した2つのサブネットに配置されている
    セキュリティグループが正しく関連付けられている
・ ターゲットグループ
    プロトコルがHTTP (80番)に設定されている
    VPC・ヘルスチェックパスが正しく指定されている
    ヘルスチェックの結果がhealthyになっているか
・ セキュリティグループ
    インバウンドルール：TCP 80番 (HTTP)  → 11.0.0.0/16 から許可されている
    アウトバウンドルールがデフォルト（全許可）になっている
```

#### ✅ 動作確認

- CloudFormationの画面で、ELBスタックのDNS名(出力欄の値)を立ち上げたブラウザに入力して、ELB経由でWordPressにアクセスします
- WordPress (EC2) の画面から設定と動作確認を行います。手順は[こちら](https://catalog.us-east-1.prod.workshops.aws/workshops/47782ec0-8e8c-41e8-b873-9da91e822b36/ja-JP/simple-mode/40-wp#)をご確認ください。

:::details  💪 Systems Managerの活用方法。

#### Systems Managerの活用方法

今回EC2が想定と異なる挙動があったため、私はSystems Managerを活用しました。
IAMロールを割り当てて接続する方法として、こちらの記事を参考にさせていただきました。

@[card](https://techblog.techfirm.co.jp/entry/aws-ssm-ec2-access-from-aws-console)

:::

:::message alert

#### 費用面での注意事項

**不要な費用発生を避けるために、使用しない場合には即時スタックを削除しましょう。**
(一例をあげると、NATゲートウェイは固定費が発生します。)

削除以外にも、EC2やRDSなどのリソースによっては**起動を一時停止**することで、費用発生の対策を取ることが可能です。

:::

## まとめ

GUIとIaCの両方を試してみた結果、今後の環境構築には**IaCを中心に活用したい**という結論に至っています。

手段は異なりますが、「**各項目を適切に設定することが重要**」という点は共通点です。
GUIに比べ、IaCではリファレンスを1つずつ確認する必要があり、習得まで時間がかかることもあります。
しかし、リファレンスを確認し設定内容の詳細を把握しながら取捨選択できる点においては、IaCはアドバンテージがあると考えています。

以下にGUIとCloudFormationのメリット・デメリットをそれぞれまとめておきますので、比較検討の参考にしてください。

### GUIでの構築

直感的な操作で概要をつかみやすく、初心者にとっては理解の助けになりました。
ただし、同じ構成を再現するには毎回同じ作業を繰り返す必要があり、リソースが増えるほど時間と手間もかかりそうです。
さらに、UIは随時アップデートされていることもあり、画面操作をベースとした情報の習得・把握へ依存することは不安が残ります。

### IaCでの構築

今回IaCを使用してみて、設定内容や変更履歴をコードで一元管理できることで、**再現性と透明性の高さ**を特に実感できました。
テンプレートの文法やエラー調査といった学習面での課題はありますが、メリットが大きいため、実運用でも積極的に活用できるようIaCの習得を進めていきたいと考えています。

## さいごに

今回、初心者でもIaCでの環境構築ができた内容を、失敗談も交えた体験とともにまとめました。
この記事が同じような初学者の方の学習の助けになりましたら幸いです。

## 参照

> @[card](https://pages.awscloud.com/JAPAN-event-OE-Hands-on-for-Beginners-Network1-2022-reg-event.html?trk=aws_introduction_page)
> @[card](https://pages.awscloud.com/JAPAN-event-OE-Hands-on-for-Beginners-Scalable-2022-reg-event.html?trk=aws_introduction_page)
> @[card](https://catalog.us-east-1.prod.workshops.aws/workshops/47782ec0-8e8c-41e8-b873-9da91e822b36/ja-JP)
> @[card](https://pages.awscloud.com/JAPAN-event-OE-Hands-on-for-Beginners-cfn-2022-reg-event.html?trk=aws_introduction_page)
