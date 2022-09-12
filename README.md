# azure-spring-apps-handson-city-service
Azure spring apps hands on project

ポータルサイトの代わりにAzure CLIを使用
# Bashコンソールを開き、本ページ作業完了後閉じず継続使用
az login # Sign into an azure account
az account show # See the currently signed-in account.

# 環境変数を設定 
AZ_RESOURCE_GROUP=SPRING_APPS_HANDSON_GRP
AZ_SPRING_APPS_NAME=azure-spring-apps-lab001
AZ_SPRING_APPS_SERVICE_NAME=city-service
# リソースグループを作成

az group create --resource-group $AZ_RESOURCE_GROUP --location eastus

# インスタンスを作成
az spring create -g "$AZ_RESOURCE_GROUP" \
    -n "$AZ_SPRING_APPS_NAME" --sku standard

# azコマンドのデフォルト値を設定、後のコマンドは短縮可能になる
az configure --defaults group=$AZ_RESOURCE_GROUP
az configure --defaults spring=$AZ_SPRING_APPS_NAME

# 最初開いたコンソールを継続で利用
# 環境変数はspringappshandsonlab & azure-spring-apps-labであることを確認
echo $AZ_RESOURCE_GROUP
echo $AZ_SPRING_APPS_NAME

# city-serviceというアプリを作成
az spring app create -n $AZ_SPRING_APPS_SERVICE_NAME --runtime-version Java_17

# CosmosDB作成 Create a SQL API database and container
# Variable block
location="East US"
failoverLocation="South Central US"
tag="create-sql-cosmosdb"
account="account-cosmosdb001" #needs to be lower case
database="spring-apps-cosmosdb001"
container="City"
partitionKey="/name"

# Create a resource group: skip, be with Spring Apps together

# Create a Cosmos account for SQL API
echo "Creating $account"
az cosmosdb create --name $account --resource-group $AZ_RESOURCE_GROUP --default-consistency-level Eventual --locations regionName="$location" failoverPriority=0 isZoneRedundant=False --locations regionName="$failoverLocation" failoverPriority=1 isZoneRedundant=False

# Create a SQL API database
echo "Creating $database"
az cosmosdb sql database create --account-name $account --resource-group $AZ_RESOURCE_GROUP --name $database

# Create a SQL API container
echo "Creating $container with $maxThroughput"
az cosmosdb sql container create --account-name $account --resource-group $AZ_RESOURCE_GROUP --database-name $database --name $container --partition-key-path $partitionKey --throughput 4000 

# Insert data into DB
# 以下のJsonデーターを2回分けてインサート
# 入力欄に貼り付けて、Saveボタンを押す
{
    "name": "Paris, France"
}

2. 同様データーを貼り付けて、Saveボタンを押す
{
    "name": "London, UK"
}


# Spring Apps サービスのバインドを設定
AZ_RESOURCE_ID=`az resource list --query "[? contains(name, 'account-cosmosdb001')].id" -o tsv`

az spring app binding cosmos add --api-type sql \
                                 --app  $AZ_SPRING_APPS_SERVICE_NAME \
                                 --name "cosmodb-binding" \
                                 --resource-group $AZ_RESOURCE_GROUP \
                                 --resource-id  \
                                 --service $AZ_SPRING_APPS_NAME \
                                 --database-name $database

# ソースコードをダウンロード
git clone https://github.com/joyoji/azure-spring-apps-handson-city-service.git

# ターゲットプロジェクトフォルダへ移動
cd azure-spring-apps-handson-city-service

# ローカル実行を確認
mvn clean package -DskipTests

# jarファイルをcity-serviceアプリへデプロィ
az spring app deploy -n AZ_SPRING_APPS_SERVICE_NAME --artifact-path target/demo-0.0.1-SNAPSHOT.jar

# 任意コンソールを開く、動作確認を実施
# URLに cities は city-service のpathとなる
# 期待値 Http status = 200, レスポンス : [[{"name":"Paris, France"},{"name":"London, UK"}]]
curl <コピーしたエンドポイント>/cities![image](https://user-images.githubusercontent.com/87787209/189562098-bf4bb61b-a291-4979-ab28-263628e1f446.png)
