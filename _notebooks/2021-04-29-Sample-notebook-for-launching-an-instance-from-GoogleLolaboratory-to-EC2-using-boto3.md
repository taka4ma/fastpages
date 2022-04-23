{
  "nbformat": 4,
  "nbformat_minor": 0,
  "metadata": {
    "colab": {
      "name": "Boto3を使ってEC2インスタンスを起動するサンプル",
      "provenance": [],
      "collapsed_sections": [],
      "include_colab_link": true
    },
    "kernelspec": {
      "name": "python3",
      "display_name": "Python 3"
    }
  },
  "cells": [
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "view-in-github",
        "colab_type": "text"
      },
      "source": [
        "<a href=\"https://colab.research.google.com/gist/taka4ma/39a4889869f029f1f82536ff3155c3a6/boto3-ec2.ipynb\" target=\"_parent\"><img src=\"https://colab.research.google.com/assets/colab-badge.svg\" alt=\"Open In Colab\"/></a>"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "XcPUuKYzmptV"
      },
      "source": [
        "# このnotebookは......\n",
        "\n",
        "Amazon Web Services (AWS)のPython向けSDKであるBotoをGoogleColaboratoryから使用し、EC2にネットワークを構築し、そこへインスタンスを起動するサンプルです。\n",
        "\n",
        "起動したインスタンスへはGoogleColaboratoryから **ではなく** PCからSSHでアクセスできるように設定します。\n",
        "\n",
        "Botoの詳細は、 [公式ドキュメント](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) を参照してください。特にEC2Clientについてはhttps://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html を参照してください。\n"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "IWW9VF51nmCY"
      },
      "source": [
        "# 準備\n",
        "\n",
        "## AWSアクセスキーを作成しておく\n",
        "\n",
        "Botoを使用するには、事前にAWSのアクセスキーを用意する必要があります。\n",
        "\n",
        "アクセスキー管理方法は様々ありますが、AWS IAMのユーザー https://console.aws.amazon.com/iam/home?region=us-west-2#/users からNotebook用のユーザーを作成する方法があります。万が一アクセスキーが漏れた場合に備えて、権限を最小限に、いつでも無効化できるように設定する必要があります。\n"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "DvMa59rgpOfi"
      },
      "source": [
        "## botoをインストールする\n",
        "\n",
        "Boto公式ドキュメントの [Installation](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/quickstart.html#installation) を参考に、GoogleColaboratoryのラインタイム環境へ、boto3をインストールします。"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "K-lW5kZQnsSd"
      },
      "source": [
        "!pip install boto3"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "7M3PLFMztDyc"
      },
      "source": [
        "## botoのための認証資格情報を環境変数に設定する\n",
        "\n",
        "Botoを使用するためには、認証資格情報をセットアップする必要があります。\n",
        "このnotebookでは以下３つの環境変数を設定することにします。\n",
        "\n",
        "\n",
        "1. AWS_ACCESS_KEY_ID\n",
        "1. AWS_SECRET_ACCESS_KEY\n",
        "1. AWS_DEFAULT_REGION\n",
        "\n",
        "AWS_ACCESS_KEY_IDとAWS_SECRET_ACCESS_KEYには、AWSアクセスキーのアクセスキー IDとシークレットアクセスキーを設定します。\n",
        "\n",
        "AWS_DEFAULT_REGIONには使用するリージョンを設定します。このnotebookでは東京リージョン( ap-northeast-1 )を使用します。\n"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "yDgAaiUeDph6"
      },
      "source": [
        "### アクセスキーIDを設定する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "vhNAmlbOwzB9"
      },
      "source": [
        "import os\n",
        "import getpass\n",
        "\n",
        "os.environ['AWS_ACCESS_KEY_ID'] = getpass.getpass()"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "7QZ1Z2Q4D5HG"
      },
      "source": [
        "### シークレットアクセスキーを設定する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "mBB8bNtkzZZJ"
      },
      "source": [
        "os.environ['AWS_SECRET_ACCESS_KEY'] = getpass.getpass()"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "sSKV4818D8mK"
      },
      "source": [
        "### デフォルトリージョンを設定する\n",
        "\n"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "zLLGpa-fyxqF"
      },
      "source": [
        "os.environ['AWS_DEFAULT_REGION'] = 'ap-northeast-1'"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "fO0F4MJtwsZo"
      },
      "source": [
        "## pandasをインポートしておく\n",
        "\n",
        "pandasを多用することになるので、インポートしておく"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "na9NhHvfwqWp"
      },
      "source": [
        "import pandas as pd"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "4g_erxy9xEx0"
      },
      "source": [
        "## ec2のclientを作っておく"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "VjCjprTbbhxs"
      },
      "source": [
        "import boto3\n",
        "ec2_client = boto3.client('ec2')"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "iXcOwpb357Jd"
      },
      "source": [
        "# ネットワークを構築する\n",
        "\n",
        "EC2インスタンスを起動するネットワークを構築します"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "rLbJ2xd1birK"
      },
      "source": [
        "## 新しいVPCを作る"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "2fpIuxGRxJ7O"
      },
      "source": [
        "### 現在のVPCのリストを確認する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "VmG31jMmxM3K"
      },
      "source": [
        "pd.DataFrame(ec2_client.describe_vpcs()['Vpcs'])"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "eMTTrBPYzjsH"
      },
      "source": [
        "### 新しいVPCを作成する\n",
        "\n",
        "今回は、`10.0.0.0/16`のCIDRブロックを持つVPCを作成する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "dqh-1DO8evK6"
      },
      "source": [
        "result = ec2_client.create_vpc(CidrBlock = '10.0.0.0/16',)"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "i5zk7xA2fFIn"
      },
      "source": [
        "result"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "8jDq5bHi3RrP"
      },
      "source": [
        "作成したvpcの情報を`NEW_VPC`に入れておく"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "Fk7FMWDY2NQc"
      },
      "source": [
        "NEW_VPC = result['Vpc']\n",
        "NEW_VPC"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "RdLQzs510U56"
      },
      "source": [
        "### VPCのリストを確認する\n",
        "\n",
        "新しく作成したVPCも見えるはず"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "FrgEC0ePuty5"
      },
      "source": [
        "pd.DataFrame(ec2_client.describe_vpcs()['Vpcs'])"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "Rtv18oGW0ecl"
      },
      "source": [
        "見えた"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "WqpVxwO50rsu"
      },
      "source": [
        "## サブネットを作成する\n",
        "\n",
        "新しく作成したVPCにサブネットを作成する"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "f6ghfBJb0wzi"
      },
      "source": [
        "### 現在のサブネットのリストを確認する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "Q1ZseiW5wRKI"
      },
      "source": [
        "subnets = ec2_client.describe_subnets()['Subnets']\n",
        "pd.DataFrame(subnets)"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "_yEMM0Cw28eD"
      },
      "source": [
        "新しく再生したVPCのサブネットは、"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "bKLukB0M1RPY"
      },
      "source": [
        "list(filter(lambda x:x['VpcId'] == NEW_VPC['VpcId'], subnets))"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "UmElcZMo3BX2"
      },
      "source": [
        "当然、ない。"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "XZEVBtLv3GN5"
      },
      "source": [
        "### 新しいVPCにサブネットを作成する\n",
        "\n",
        "availability zoneは`ap=northeast-1a`, CIDRブロックは`10.0.0.0/24`にする\n"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "4AReO8v-3AYM"
      },
      "source": [
        "result = ec2_client.create_subnet(\n",
        "    AvailabilityZone = 'ap-northeast-1a',\n",
        "    CidrBlock = '10.0.0.0/24',\n",
        "    VpcId = NEW_VPC['VpcId']\n",
        ")\n",
        "result"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "gsPH2d-A27GH"
      },
      "source": [
        "NEW_SUBNET = result['Subnet']\n",
        "NEW_SUBNET"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "7H4IX7lQ4qEn"
      },
      "source": [
        "### サブネットのリストを確認する\n",
        "\n",
        "新しく作成したサブネットも見えるはず"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "k74yev4v4m4o"
      },
      "source": [
        "subnets = ec2_client.describe_subnets()['Subnets']\n",
        "pd.DataFrame(subnets)"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "KoKs7HxA4151"
      },
      "source": [
        "list(filter(lambda x:x['VpcId'] == NEW_VPC['VpcId'], subnets))"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "Skl6KV5wg213"
      },
      "source": [
        "## サブネットにインスタンス起動時にパブリックIPを付与する設定を追加する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "kHJGbJB_g3pt"
      },
      "source": [
        "response = ec2_client.modify_subnet_attribute(\n",
        "    MapPublicIpOnLaunch={'Value': True},\n",
        "    SubnetId=NEW_SUBNET['SubnetId']\n",
        ")\n",
        "response"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "rjhT4210hlzm"
      },
      "source": [
        "list(filter(lambda x:x['VpcId'] == NEW_VPC['VpcId'], ec2_client.describe_subnets()['Subnets']))"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "0mxWFiwMhukV"
      },
      "source": [
        "`MapPublicIpOnLaunch`の値がTrueになっている"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "A7p3Am7R6Mek"
      },
      "source": [
        "## インターネットゲートウェイを作成する"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "Hi4jZ5Fw6vkz"
      },
      "source": [
        "### 現在のインターネットゲートウェイのリストを確認する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "vKiiB51x44gQ"
      },
      "source": [
        "internet_gateways = ec2_client.describe_internet_gateways()['InternetGateways']\n",
        "pd.DataFrame(internet_gateways)"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "uev8Duf39YlE"
      },
      "source": [
        "### 新しいインターネットゲートウェイを作成する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "yzChjmzZ6mFq"
      },
      "source": [
        "result = ec2_client.create_internet_gateway()\n",
        "result"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "BbZfrEmI9jlH"
      },
      "source": [
        "NEW_IG = result['InternetGateway']\n",
        "NEW_IG"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "K22YiaEb9vsy"
      },
      "source": [
        "### インターネットゲートウェイのリストを確認する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "HQD0wRlX9q01"
      },
      "source": [
        "internet_gateways = ec2_client.describe_internet_gateways()['InternetGateways']\n",
        "pd.DataFrame(internet_gateways)"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "N8PzgXpQ_TtB"
      },
      "source": [
        "## VPCにインターネットゲートウェイをアタッチする\n"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "FdrR3PoQA0Wx"
      },
      "source": [
        "### アタッチする"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "WbsZPqCu90pZ"
      },
      "source": [
        "result = ec2_client.attach_internet_gateway(\n",
        "    InternetGatewayId = NEW_IG['InternetGatewayId'],\n",
        "    VpcId = NEW_VPC['VpcId']\n",
        ")\n",
        "\n",
        "result"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "5QYlFWgRAdRs"
      },
      "source": [
        "### インターネットゲートウェイの設定を確認する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "4d94sfZx_4eS"
      },
      "source": [
        "list(filter(lambda x:x['InternetGatewayId'] == NEW_IG['InternetGatewayId'], ec2_client.describe_internet_gateways()['InternetGateways']))"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "pEzw29kGAqpj"
      },
      "source": [
        "`Attachments`にアタッチしたVPCのIDが設定されている"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "wF5tcqAWOzRK"
      },
      "source": [
        "## サブネットにインターネットゲートウェイをデフォルトゲートウェイにするルートを追加する\n",
        "\n",
        "以下の順番で行います\n",
        "\n",
        "1. 新しいルートテーブルを作成する\n",
        "1. 作成したルートテーブルにインターネットゲートウェイをデフォルトゲートウェイにするルートを追加する\n",
        "1. 作成したルートテーブルを作成したサブネットに関連づける"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "nTWkbFcNPQwV"
      },
      "source": [
        "### 現在のルートテーブルのリストを確認する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "ZxGacSzSO4Kx"
      },
      "source": [
        "route_tables = ec2_client.describe_route_tables()['RouteTables']\n",
        "pd.DataFrame(route_tables)"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "Ji_EhudWZeOH"
      },
      "source": [
        "新しいVPCのルートテーブルを確認する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "5LzBZ3LyZeZ9"
      },
      "source": [
        "route_table = list(filter(lambda x:x['VpcId'] == NEW_VPC['VpcId'], ec2_client.describe_route_tables()['RouteTables']))[0]\n",
        "route_table"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "zefqVk2DZ_tX"
      },
      "source": [
        "`Routes`に VPCのCidrブロック(10.0.0.0/16)宛のパケットを`local`、つまりVPC領域のルータへ転送する設定がされていることがわかる"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "zxyy0cISCE6m"
      },
      "source": [
        "### ルートテーブルを作成する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "BqtUbUJ5VKjs"
      },
      "source": [
        "response = ec2_client.create_route_table(\n",
        "    VpcId = NEW_VPC['VpcId']\n",
        ")\n",
        "response"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "LuhCiIyCVk-B"
      },
      "source": [
        "NEW＿ROUTE_TABLE = response['RouteTable']\n",
        "NEW＿ROUTE_TABLE"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "6ORp7brZbGf-"
      },
      "source": [
        "この時点で、`Routes`は既存のルートテーブルと同じになっている。また、`Associations`が空なので何にも関連づけられていない。"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "7PuFMKvXVZFG"
      },
      "source": [
        "### 作成したルートテーブルに、全てのパケットをインターネットゲートウェイに転送するルートを作成する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "7f1XKYd1RlDV"
      },
      "source": [
        "response = ec2_client.create_route(\n",
        "    DestinationCidrBlock = '0.0.0.0/0',\n",
        "    GatewayId = NEW_IG['InternetGatewayId'],\n",
        "    RouteTableId = NEW＿ROUTE_TABLE['RouteTableId']\n",
        ")\n",
        "response"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "1zD-IkwxSV2C"
      },
      "source": [
        "新しいルートテーブルの設定を確認する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "z5X9gNNqR7Um"
      },
      "source": [
        "route_table = list(filter(lambda x:x['RouteTableId'] == NEW＿ROUTE_TABLE['RouteTableId'], ec2_client.describe_route_tables()['RouteTables']))[0]\n",
        "route_table"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "QhT_99rXSakN"
      },
      "source": [
        "`Routes`にDestinationCidrBlockが0.0.0.0/0の設定が追加されている"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "xHh0K7I3Smou"
      },
      "source": [
        "### 新しいルートテーブルをサブネットに関連づける"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "3ia_6FpsSZ0k"
      },
      "source": [
        "response = ec2_client.associate_route_table(\n",
        "    RouteTableId = NEW＿ROUTE_TABLE['RouteTableId'],\n",
        "    SubnetId = NEW_SUBNET['SubnetId']\n",
        ")\n",
        "response"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "e_a5LebIXcpv"
      },
      "source": [
        "新しいルートテーブルの設定を確認する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "Vtgsi3hqTnwV"
      },
      "source": [
        "route_table = list(filter(lambda x:x['RouteTableId'] == NEW＿ROUTE_TABLE['RouteTableId'], ec2_client.describe_route_tables()['RouteTables']))[0]\n",
        "route_table"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "wS6v7l_JXebV"
      },
      "source": [
        "`Associations`が設定されていることがわかる"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "ZfEMtHMNh4h2"
      },
      "source": [
        "# サーバを構築する"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "GhzX2dEtiHKN"
      },
      "source": [
        "## セキュリティグループを作成する"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "O59TvgJ0iLBV"
      },
      "source": [
        "### 現在のセキュリティグループのリストを確認する\n",
        "\n",
        "新しくVPCを作成するとデフォルトのセキュリティグループが作成されるので確認する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "0IjiBrAuiWr_"
      },
      "source": [
        "list(filter(lambda x:x['VpcId'] == NEW_VPC['VpcId'], ec2_client.describe_security_groups()['SecurityGroups']))"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "7Xypw3ldjhS9"
      },
      "source": [
        "### 新しいセキュリティグループを作成する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "97w7IWDpi4vL"
      },
      "source": [
        "response = ec2_client.create_security_group(\n",
        "    Description = 'from-colab',\n",
        "    GroupName = 'from-colab',\n",
        "    VpcId = NEW_VPC['VpcId']\n",
        ")\n",
        "response"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "kzXHeSSvkA0Z"
      },
      "source": [
        "NEW_SECURITY_GROUP_ID = response['GroupId']\n",
        "NEW_SECURITY_GROUP_ID"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "MOpfIKwYm_6Y"
      },
      "source": [
        "### 作成したセキュリティグループの設定を確認しておく"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "dord6Rfhj6LH"
      },
      "source": [
        "list(filter(lambda x:x['GroupId'] == NEW_SECURITY_GROUP_ID, ec2_client.describe_security_groups()['SecurityGroups']))"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "VfKnzL1gkpLK"
      },
      "source": [
        "## セキュリティグループにsshアクセス用の設定を追加する"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "eHRnp0qZmEFs"
      },
      "source": [
        "### インスタンスにSSHでアクセスするPCのIPアドレスを設定する\n",
        "\n",
        "AWSコンソールから設定する場合は 'マイ IP' を選択すれば自分のPCのIPアドレスを設定できましたが、boto3から設定する場合は自分のIPアドレスを値として設定する必要があります"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "zzrxsAsalrNh"
      },
      "source": [
        "myip = 'xxx.xxx.xxx.xxx/xx'"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "m7g5wVIJmw0n"
      },
      "source": [
        "### セキュリティグループに設定を追加する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "qUec_Hs3kOl-"
      },
      "source": [
        "response = ec2_client.authorize_security_group_ingress(\n",
        "    GroupId = NEW_SECURITY_GROUP_ID,\n",
        "    IpPermissions = [{\n",
        "        'FromPort': 22,\n",
        "        'IpProtocol': 'tcp',\n",
        "        'IpRanges': [{\n",
        "            'CidrIp': myip,\n",
        "            'Description': 'SSH access from my ip'\n",
        "        }],\n",
        "        'ToPort': 22\n",
        "    }]\n",
        ")\n",
        "response"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "zJjfxJDenHiJ"
      },
      "source": [
        "### セキュリティグループの設定を確認しておく"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "J33VvAw7m1OL"
      },
      "source": [
        "list(filter(lambda x:x['GroupId'] == NEW_SECURITY_GROUP_ID, ec2_client.describe_security_groups()['SecurityGroups']))"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "uL8XJYbTnPpM"
      },
      "source": [
        "`IpPermissions`に設定が追加されている"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "O9Hb-_l-oQ7d"
      },
      "source": [
        "## キーペアを作成する"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "VDDLyJE6oWMk"
      },
      "source": [
        "### 秘密鍵の保存先としてGoogleDriveをマウントする\n",
        "\n",
        "Google ドライブをマウントするには、次のセルを実行してください。  \n",
        "実行すると、ランタイム環境がGoogleDriveへアクセスするための権限を要求されます。\n",
        "\n",
        "1. Go to this URL in a browser: https://accounts.google.com/o/oauth2/<以下略> という形式のリンクが表示されるので、マウスでクリックして開いてください\n",
        "2. (必要に応じてGoogleアカウントへのログインと)開いた先でアクセス権限を要求されるので、承認してください  承認すると認証コードが表示されるのでコピーします。\n",
        "3. Enter your authorization code: と表示されているテキストボックスへ認証コードを入力しEnterを押下してください.  Mounted at /content/drive と表示されれば成功です\n"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "iJsH2bP0namm"
      },
      "source": [
        "from google.colab import drive\n",
        "drive.mount('/content/drive')"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "Q6vZ8OgXo0OB"
      },
      "source": [
        "### キーペアを作成する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "l5BD9r6FonvP"
      },
      "source": [
        "response = ec2_client.create_key_pair(\n",
        "   KeyName = 'from-colab'\n",
        ")\n",
        "response"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "ZoUGGNp5pkNS"
      },
      "source": [
        "NEW_KEY_NAME = response['KeyName']\n",
        "NEW_KEY_NAME"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "ml3T5NY7p4SO"
      },
      "source": [
        "### 秘密鍵を保存する\n",
        "\n",
        "GoogleDriveの\"マイドライブ\"保存します。\n",
        "保存後、SSHアクセスするPCへ移動させてください。"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "DSeMc1IqpzxU"
      },
      "source": [
        "file_path = '/content/drive/MyDrive/{}.pem'.format(NEW_KEY_NAME)\n",
        "with open(file_path, mode='w') as f:\n",
        "  f.write(response['KeyMaterial'])"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "CqeuRqN0wlUD"
      },
      "source": [
        "## インスタンスを起動する"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "x9jnU3oHzDV_"
      },
      "source": [
        "### インスタンス名を設定する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "dxfA37GPzF07"
      },
      "source": [
        "instance_name = 'from-colab'"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "fY8ZPjxLworY"
      },
      "source": [
        "### イメージidを設定する\n",
        "\n",
        "インスタンスにインストールするイメージのidを設定します。  \n",
        "今回は`amzn2-ami-hvm-2.0.20210427.0-x86_64-gp2`を使います。\n",
        "\n",
        "// 本当は`describe_images`メソッドを使って探したいところですが、  \n",
        "// イメージの数が多いので単純に探すと時間がかかり、絞り込みも大変なので  \n",
        "// awsコンソールのec2からインスタンスの起動を選び、ステップ1: Amazon マシンイメージ(AMI)で確認するのが簡単です。"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "gdypG5q9woBt"
      },
      "source": [
        "image_id = 'ami-0ca38c7440de1749a'"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "kpQe_5Rty78x"
      },
      "source": [
        "image_idでimageが見つかることを確認しておく"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "jm_kDbjeyZ_f"
      },
      "source": [
        "ec2_client.describe_images(\n",
        "    ImageIds = [image_id]\n",
        ")"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "rK5jhPj3zLhG"
      },
      "source": [
        "### インスタンスを起動する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "CbnAcA_zypXq"
      },
      "source": [
        "responce = ec2_client.run_instances(\n",
        "    ImageId = image_id,\n",
        "    MinCount = 1,\n",
        "    MaxCount=1,\n",
        "    InstanceType='t2.micro',\n",
        "    KeyName = NEW_KEY_NAME,\n",
        "    SecurityGroupIds = [NEW_SECURITY_GROUP_ID],\n",
        "    SubnetId = NEW_SUBNET['SubnetId'],\n",
        "    TagSpecifications = [{\n",
        "        'ResourceType': 'instance',\n",
        "        'Tags': [\n",
        "            {'Key': 'Name', 'Value': instance_name}\n",
        "        ]\n",
        "    }]\n",
        ")"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "elbHI48V0Qyr"
      },
      "source": [
        "NEW_INSTANCES = responce['Instances']\n",
        "NEW_INSTANCES"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "wFDpwFB91Qa0"
      },
      "source": [
        "### 起動したインスタンスのパブリックIPを確認する\n",
        "\n",
        "`run_instances`のタイミングではパブリックIPは設定されていないので、起動が終わるのを待ってから詳細情報を取得し、そこからパブリックIPを得る"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "aafPHfmH0Y9X"
      },
      "source": [
        "response = ec2_client.describe_instances(\n",
        "    InstanceIds = list(map(lambda x:x['InstanceId'], NEW_INSTANCES))\n",
        ")\n",
        "response"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "URcqypl93tUA"
      },
      "source": [
        "pd.DataFrame([\n",
        "  {\n",
        "        'InstanceId': insntance['InstanceId'],\n",
        "        'InsntanceNmae': list(map(lambda x:x['Value'], filter(lambda x:x['Key'] == 'Name', insntance['Tags'])))[0],\n",
        "        'KeyName': insntance['KeyName'],\n",
        "        'PublicIpAddress': insntance['PublicIpAddress'],\n",
        "  } for reservation in response['Reservations'] for insntance in reservation['Instances'] \n",
        "])"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "zJTMYpHo57d_"
      },
      "source": [
        "### ssh接続確認\n",
        "\n",
        "上で出力した情報を参考PCから起動したインスタンスへsshで接続できることを確認します。\n",
        "\n",
        "確認できたら、boto3を使ったEC2インスタンスの起動は完了です。\n"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "9zeY0ZAG6tYK"
      },
      "source": [
        "# 後片付け\n",
        "\n",
        "このnotebookはサンプルなので、作成したリソースを削除します。  \n",
        "もし、作成したリソースを使い続けたい場合はこのセクションはスキップしてください"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "l3sU6m5x7A2G"
      },
      "source": [
        "## インスタンスを削除する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "Lm0TmTHW5soK"
      },
      "source": [
        "response = ec2_client.terminate_instances(\n",
        "    InstanceIds = list(map(lambda x:x['InstanceId'], NEW_INSTANCES))\n",
        ")\n",
        "response"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "rMk7Th407Pi8"
      },
      "source": [
        "## キーペアを削除する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "I8B-QU8O7JQe"
      },
      "source": [
        "response = ec2_client.delete_key_pair(KeyName=NEW_KEY_NAME)\n",
        "response"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "0eP7BlNO7c3l"
      },
      "source": [
        "## セキュリティグループを削除する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "NM2yWnQW7YcM"
      },
      "source": [
        "response = ec2_client.delete_security_group(GroupId=NEW_SECURITY_GROUP_ID)\n",
        "response"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "Rlvbw0oQ8dHO"
      },
      "source": [
        "## ルートテーブルを削除する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "QeznWYpi7o3M"
      },
      "source": [
        "route_table = list(filter(lambda x:x['RouteTableId'] == NEW＿ROUTE_TABLE['RouteTableId'], ec2_client.describe_route_tables()['RouteTables']))[0]\n",
        "route_table"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "SjY3Ilxz8sUD"
      },
      "source": [
        "### ルートテーブルの関連付けを解除する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "Cei1Kx_N8vZ5"
      },
      "source": [
        "for association in route_table['Associations']:\n",
        "  response = ec2_client.disassociate_route_table(\n",
        "      AssociationId = association['RouteTableAssociationId']\n",
        "  )\n",
        "  print(response)"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "mj_QmCSs9Eis"
      },
      "source": [
        "### ルートテーブルを削除する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "ogL4LCkm8UGY"
      },
      "source": [
        "responce = ec2_client.delete_route_table(\n",
        "    RouteTableId = NEW＿ROUTE_TABLE['RouteTableId']\n",
        ")\n",
        "responce"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "qoZKi5jZ9MTz"
      },
      "source": [
        "## インターネットゲートウェイを削除する"
      ]
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "Qim4mH9T9fP0"
      },
      "source": [
        "## インターネットゲートウェイをVPCからデタッチする"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "sENU4k2b9iqe"
      },
      "source": [
        "responce = ec2_client.detach_internet_gateway(\n",
        "    InternetGatewayId = NEW_IG['InternetGatewayId'],\n",
        "    VpcId = NEW_VPC['VpcId']\n",
        ")\n",
        "responce"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "eX4-OE9M9tqi"
      },
      "source": [
        "### インターネットゲートウェイを削除する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "qzdJrO6K8ocw"
      },
      "source": [
        "responce = ec2_client.delete_internet_gateway(\n",
        "    InternetGatewayId = NEW_IG['InternetGatewayId']\n",
        ")\n",
        "responce"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "fsj53u1c944E"
      },
      "source": [
        "## サブネットを削除する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "-Qg62a0h9d7Z"
      },
      "source": [
        "response = ec2_client.delete_subnet(\n",
        "    SubnetId = NEW_SUBNET['SubnetId']\n",
        ")\n",
        "response"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "TGPd9jbw-Fsh"
      },
      "source": [
        "## VPCを削除する"
      ]
    },
    {
      "cell_type": "code",
      "metadata": {
        "id": "F3_GO1Kx-BCB"
      },
      "source": [
        "response = ec2_client.delete_vpc(\n",
        "    VpcId = NEW_VPC['VpcId']\n",
        ")\n",
        "response"
      ],
      "execution_count": null,
      "outputs": []
    },
    {
      "cell_type": "markdown",
      "metadata": {
        "id": "MqXwFMhP-YO2"
      },
      "source": [
        "以上"
      ]
    }
  ]
}