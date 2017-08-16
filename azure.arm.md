
> 本内容は、作成時点での下記ページを元に作成したものです。　　
> https://docs.pivotal.io/pivotalcf/1-11/customizing/azure.html  
>  PCF動作を保証するものではありませんので予めご了承下さい。  
> また、全ての内容はdocs.pivotal.ioに記載されているものに従うものとして、
> 下記内容は、それらを補完するものとしてご活用ください。

# Preparing to Deploy PCF on Azure

## Step 1: Install and Configure the Azure CLI

1. azure cliのセットアップ

  - PCF 1.10 Mac http://aka.ms/mac-azure-cli
  - PCF 1.11 Mac https://aka.ms/InstallAzureCliWindows
  - PCF 1.11 Windows MSI https://aka.ms/InstallAzureCliWindows

      (Option). azure versionのcheck

      ``
$ azure --version
      ``

2. Cloudのセット

  ``
az cloud set --name AzureCloud
  ``

3. ログイン

  ``
  az login
  ``

## Step 2: Set Your Default Subscription

  1. Azureサブスクリプションの確認

  ``
  az account list
  ``

  "isDefault", "id", "tenantID"があるか確認

  2. (Option) サブスクリプションの変更
　
　"isDefault"が"true"でない場合は、下記で然るべきサブスクリプションに変更する

  ``
  az account set --subscription SUBSCRIPTION_ID
  ``

## Step 3: Create an Azure Active Directory (AAD) Application

  1. Azure Active Directory Applicationの作成

  ```
  az ad app create --display-name "Service Principal for BOSH" \
--password "PASSWORD" --homepage "http://BOSHAzureCPI" \
--identifier-uris "http://BOSHAzureCPI"
  ```
homepageとidentifier-urisについては任意の情報で構いません。

  > ドキュメント上は、上記を推奨します。とのこと。

　上記コマンドのアウトプットで出てくる"appId"は、次のService Principalに
利用します。

## Step 4: Create and Configure a Service Principal
　サービスプリンシパルの作成と構成


　1. Service Principalの作成

　``
az ad sp create --id YOUR-APPLICATION-ID
　``

  > 'YOUR-APPLICATION-ID'はStep3で取得したものです。

　2.  Contributor roleが必要のため、下記コマンドを実行します。

　``
az role assignment create --assignee "SERVICE-PRINCIPAL-NAME" \
--role "Contributor" --scope /subscriptions/SUBSCRIPTION-ID
　``

  > 'SERVICE-PRINCIPAL-NAME'は、1の出力から確認出来ます。
  > 'SUBSCRIPTION-ID'は、Step2の結果から確認出来ます。(Step2における'id')

　3. 出力内容の確認

　``
　az role assignment list --assignee "SERVICE-PRINCIPAL-NAME"
　``

## Step 5: Verify Your Service Principal

  サービスプリンシパルの確認

  ``
  az login --username APPLICATION_ID --password CLIENT_SECRET \
--service-principal --tenant TENANT_ID
  ``
　
  > 'APPLICATION_ID': Step3で作成済み（表示されている)
  > 'CLIENT_SECRET': Step3で作成済み(コマンドの引数において、ご自身で指定してます)
  > 'TENANT_ID': Step2で確認済み


## Step 6: Perform Registrations

　1. Microsoft.Storageの登録

　``
az provider register --namespace Microsoft.Storage
　``

　2. Microsoft.Networkの登録

　``
az provider register --namespace Microsoft.Network
　``

　3. Microsoft.Computeの登録

　``
az provider register --namespace Microsoft.Compute
　``

# Launching an Ops Manager Director Instance with an ARM Template
https://docs.pivotal.io/pivotalcf/1-11/customizing/azure-arm-template.html


## Step 1: Create BOSH Storage Account

1. リソースグループを選んで、$RESOURCE_GROUPとして環境変数を設定（リソースグループはこのあと作成)

``
$ export RESOURCE_GROUP="YOUR-RESOURCE-GROUP-NAME"
``

2. locationを指定

``
$ export LOCATION="YOUR-LOCATION"
``

``
az account list-locations
``

3. リソースグループの作成

``
az group create --name $RESOURCE_GROUP --location $LOCATION
``

4. ストレージアカウントの指定。Azure全体でもユニークである必要があるので、3-24文字で適当な名前を指定。

``
export STORAGE_NAME="YOUR-BOSH-STORAGE-ACCOUNT-NAME"
``

5. ストレージアカウントの作成。これまで指定した環境変数を利用

```
$ az storage account create --name $STORAGE_NAME \
--resource-group $RESOURCE_GROUP --sku Standard_LRS \
--kind Storage --location $LOCATION
```

6. ストレージアカウントのコネクション情報(connection string)の指定

```
$ az storage account show-connection-string \
--name $STORAGE_NAME --resource-group $RESOURCE_GROUP
```

7. 上記のconnectionStringの情報を確認。"DefaultEndpointsProtocol="からの文字列を確認

8. connectionStringを指定

``
$ export CONNECTION_STRING="YOUR-CONNECTION-STRING"
``

9. Ops Managerのイメージ用のコンテナを作成:

```
$ az storage container create --name opsman-image \
--connection-string $CONNECTION_STRING
```

10. Ops Manager VM用のコンテナを作成:

```
$ az storage container create --name vhds \
--connection-string $CONNECTION_STRING
```

11. Ops Manager用のコンテナを作成:

```
$ az storage container create --name opsmanager \
--connection-string $CONNECTION_STRING
```

12. BOSH用のコンテナを作成:

```
$ az storage container create --name bosh \
--connection-string $CONNECTION_STRING
```

13. Stemcell用のコンテナを作成:

```
az storage container create --name stemcell \
--public-access blob \
--connection-string $CONNECTION_STRING
```

14. Stemcellデータ用のテーブルを作成:

```
$ az storage table create --name stemcells \
--connection-string $CONNECTION_STRING
```

## Step 2: Copy Ops Manager Image

1. Pivotal NetworkでPivotal Cloud Foundry Ops Manager for Azureの最新版をダウンロード
2. ファイルを開いて適切なリージョンを選択
3. URLを指定

``
$ export OPS_MAN_IMAGE_URL="YOUR-OPS-MAN-IMAGE-URL"
``

4. Storage Accountにイメージを指定

```
$ az storage blob copy start --source-uri $OPS_MAN_IMAGE_URL \
--connection-string $CONNECTION_STRING \
--destination-container opsman-image \
--destination-blob image.vhd
```

5. 4の進捗確認(status"が"success"になっていれば良い)

```
$ az storage blob show --name image.vhd \
--container-name opsman-image \
--account-name $STORAGE_NAME
...
"copy": {
  "completionTime": "2017-06-26T22:24:11+00:00",
  "id": "b9c8b272-a562-4574-baa6-f1a04afcefdf",
  "progress": "53687091712/53687091712",
  "source": "https://opsmanagerwestus.blob.core.windows.net/images/ops-manager-1.11.3.vhd",
  "status": "success",
  "statusDescription": null
},
```

## Step 3: Configure the ARM Template

1. ubuntuと呼ばれるkeypairを作成

```
$ ssh-keygen -t rsa -f opsman -C ubuntu
```

passphaseを求められたら、Enterキーを押す。（入力せずに空のまま)

2. PCF Azure ARM TemplatesをGithubからコピー

https://github.com/pivotal-cf/pcf-azure-arm-templates.git

- ARM Template: azure-deploy.json
- パラメータファイル: azure-deploy-parameters.json

3. ファイルを開けて、以下の内容を設定

- storageAccountName: Step 1で作成したもの
- storageEndpoint: そのままにする(Azure China, Azure Government Cloud, or Azure Germanyを利用しない限り)
- adminSSHKey: Step 3-1で作成したpublic key
- tenantID: "Preparing to Deploy PCF on Azure"にて作成
- clientID: "Preparing to Deploy PCF on Azure"にて作成
- clientSecret: "Preparing to Deploy PCF on Azure"にて作成
- vmSize: Ops Manager VMのサイズを指定。Standard_DS2_v2.
- location: Ops Managerをインストールするローケーション(Step1と同じ）

## Step 4: Deploy the ARM Template and Deployment Storage Accounts

1. テンプレートのデプロイ

```
az group deployment create --template-file azure-deploy.json \
--parameters azure-deploy-parameters.json \
--resource-group $RESOURCE_GROUP --name cfdeploy
```

2. 実行後、下記を確認

- opsMan-FQDN
```
"opsMan-FQDN": {
    "type": "String",
    "value": "pcf-opsman-efecrvhdf7w7g.westus.cloudapp.azure.com"
  }
```
- extra Storage Account Prefix
```
"extra Storage Account Prefix": {
  "type": "String",
  "value": "xtrastrgefecrvhdf7w7g"
  },
```

3. 5つのストレージアカウントを作成するので、上記のextra Storage Account Prefixに
1-5の数字を追加する。

  - xtrastrgefecrvhdf7w7g1
  - xtrastrgefecrvhdf7w7g2
  - xtrastrgefecrvhdf7w7g3
  - xtrastrgefecrvhdf7w7g4
  - xtrastrgefecrvhdf7w7g5

それぞれのアカウントに対して、下記を実行

3-1. アカウントに対して、YOUR-DEPLOYMENT-STORAGE-ACCOUNT-NAMEを置き換える
```
$ az storage account show-connection-string \
--name YOUR-DEPLOYMENT-STORAGE-ACCOUNT-NAME --resource-group $RESOURCE_GROUP
```
以下のような出力になる
```
{
  "connectionString": "DefaultEndpointsProtocol=https;EndpointSuffix=core.windows.net;AccountName=cfdocsdeploystorage1;AccountKey=oa/QiSAmqj1OocsGhKBwn/Mf8wEwdeJMvvonrbmNk27bfkSL8ZFzAhs3Kb78si5CTPHhjHHiK4qPcYzn/8OmFg=="
}
```

3-2. 上記の出力から、connectionstringを確認


3-3. CONNECTION_STRING_Nを指定(Nは1-5の数字)
``
$ export CONNECTION_STRING_N="YOUR-CONNECTION-STRING"
``

3-4. BOSH用のコンテナを作成

```
$ az storage container create --name bosh \
--connection-string $CONNECTION_STRING_N
```

3-5. Stemcell用のコンテナを作成

```
$ az storage container create --name stemcell \
--connection-string $CONNECTION_STRING_N
```

4. pcf-nsgというネットワークセキュリティグループを作成

```
$ az network nsg create --name pcf-nsg \
--resource-group $RESOURCE_GROUP \
--location $LOCATION
```

5. ネットワークセキュリティグループロールをpcf-nsgグループに付与して、public networkからのアクセスを許可

```
$ az network nsg rule create --name internet-to-lb \
--nsg-name pcf-nsg --resource-group $RESOURCE_GROUP \
--protocol Tcp --priority 100 \
--destination-port-range '*'
```

## Step 5: Complete Ops Manager Director Configuration

1. Ops Managerにアクセス

opsMan-FQDNで指定されたドメインにアクセスするように、自身のドメインを登録

2. "Configuring Ops Manager Director on Azure"に進む
https://docs.pivotal.io/pivotalcf/1-11/customizing/azure-om-config.html
