---
title: Helm グラフの保存
description: Azure Container Registry のリポジトリを使用して Kubernetes アプリケーションの Helm グラフを保存する方法について説明します
ms.topic: article
ms.date: 03/20/2020
ms.openlocfilehash: 3f1a68258b758380a66b63e3c3137f1d460d288c
ms.sourcegitcommit: 849bb1729b89d075eed579aa36395bf4d29f3bd9
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 04/28/2020
ms.locfileid: "81399390"
---
# <a name="push-and-pull-helm-charts-to-an-azure-container-registry"></a>Azure コンテナー レジストリに対する Helm グラフのプッシュおよびプル

アプリケーションを Kubernetes 用に簡単に管理し、デプロイするには、[オープン ソースの Helm パッケージ マネージャー][helm]を使用します。 Helm を使用すると、アプリケーション パッケージは[グラフ](https://helm.sh/docs/topics/charts/)として定義されます。これは収集され、[Helm グラフ リポジトリ](https://helm.sh/docs/topics/chart_repository/)に格納されます。

この記事では、Helm 3 または Helm 2 のインストールを使用して、Azure コンテナー レジストリで Helm グラフ リポジトリをホストする方法について説明します。 多くのシナリオでは、開発したアプリケーション用の独自のグラフを構築し、アップロードすることになります。 独自の Helm グラフを作成する方法の詳細については、「[Chart Template Developer's Guide (グラフ テンプレート開発者ガイド)][develop-helm-charts]」を参照してください。 また、別の Helm リポジトリから既存の Helm グラフを格納することもできます。

> [!IMPORTANT]
> Azure Container Registry での Helm グラフのサポートは現在プレビュー段階です。 プレビュー版は、追加の[使用条件][terms-of-use]に同意することを条件に使用できます。 この機能の一部の側面は、一般公開 (GA) 前に変更される可能性があります。

## <a name="helm-3-or-helm-2"></a>Helm 3 か Helm 2 か

Helm グラフの格納、管理、およびインストールには、Helm クライアントと Helm CLI を使用します。 Helm クライアントのメジャー リリースには、Helm 3 と Helm 2 があります。 Helm 3 では新しいグラフ形式がサポートされており、Tiller サーバー側コンポーネントはインストールされなくなりました。 バージョン間の違いの詳細については、[バージョンに関する FAQ](https://helm.sh/docs/faq/) を参照してください。 以前に "Helm 2" グラフをデプロイした場合は、「[Migrating Helm v2 to v3 (Helm v2 から v3 への移行)](https://helm.sh/docs/topics/v2_v3_migration/)」を参照してください。

Helm 3 または Helm 2 を使用して Azure Container Registry で Helm グラフをホストし、各バージョンに固有のワークフローを使用できます。

* [Helm 3 クライアント](#use-the-helm-3-client) - Helm CLI で `helm chart` コマンドを使用して、レジストリ内のグラフを [OCI 成果物](container-registry-image-formats.md#oci-artifacts)として管理します
* [Helm 2 クライアント](#use-the-helm-2-client) - Azure CLI の [az acr helm][az-acr-helm] コマンドを使用して、コンテナー レジストリを Helm グラフ リポジトリとして追加および管理します

### <a name="additional-information"></a>関連情報

* ほとんどの場合、グラフを OCI 成果物として管理するために、Helm 3 ワークフローをネイティブの `helm chart` コマンドと共に使用することをお勧めします。
* Helm 3 クライアントおよびグラフでは、従来の [az acr helm][az-acr-helm] Azure CLI コマンドとワークフローを使用できます。 ただし、`az acr helm list` などの特定のコマンドは Helm 3 グラフと互換性がありません。
* Helm 3 では、[az acr helm][az-acr-helm] コマンドは、主に Helm 2 クライアントおよびグラフ形式との互換性のためにサポートされています。 今後、これらのコマンドの開発は計画されていません。

## <a name="use-the-helm-3-client"></a>Helm 3 クライアントを使用する

### <a name="prerequisites"></a>前提条件

- Azure サブスクリプションの **Azure コンテナー レジストリ**。 必要に応じて、[Azure portal](container-registry-get-started-portal.md) または [Azure CLI](container-registry-get-started-azure-cli.md) を使用してレジストリを作成します。
- **Helm クライアント バージョン 3.1.0 以降** - 現在のバージョンを確認するには、`helm version` を実行します。 Helm のインストール方法とアップグレード方法について詳しくは、「[Installing Helm (Helm のインストール)][helm-install]」をご覧ください。
- Helm グラフをインストールする **Kubernetes クラスター**。 必要に応じて、[Azure Kubernetes Service クラスター][aks-quickstart]を作成します。 
- **Azure CLI バージョン 2.0.71 以降** - バージョンを確認するには `az --version` を実行します。 インストールまたはアップグレードする必要がある場合は、[Azure CLI のインストール][azure-cli-install]に関するページを参照してください。

### <a name="high-level-workflow"></a>ワークフローの概要

**Helm 3** の場合:

* Azure コンテナー レジストリに 1 つ以上の Helm リポジトリを作成できます
* Helm 3 グラフをレジストリに [OCI 成果物](container-registry-image-formats.md#oci-artifacts)として格納します。 現在、OCI の Helm 3 のサポートは "*試験段階*" です。
* `helm registry login` コマンドを使用して、レジストリで認証します。
* Helm CLI で `helm chart` コマンドを使用して、レジストリ内の Helm グラフをプッシュ、プル、および管理します
* `helm install` を使用して、ローカル リポジトリ キャッシュから Kubernetes クラスターにグラフをインストールします。

例については、次のセクションを参照してください。

### <a name="enable-oci-support"></a>OCI のサポートを有効にする

次の環境変数を設定して、Helm 3 クライアントで OCI のサポートを有効にします。 現時点では、このサポートは試験段階です。 

```console
export HELM_EXPERIMENTAL_OCI=1
```

### <a name="create-a-sample-chart"></a>サンプル グラフを作成する

次のコマンドを使用して、テスト グラフを作成します。

```console
mkdir helmtest

cd helmtest
helm create hello-world
```

基本的な例として、ディレクトリを `templates` フォルダーに変更し、まずその内容を削除します。

```console
cd hello-world/templates
rm -rf *
```

`configmap.yaml` フォルダー内に `templates` というファイルを作成し、内容を次のようにします。

```console
cat <<EOF > configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-world-configmap
data:
  myvalue: "Hello World"
EOF
```

この例の作成と実行の詳細については、Helm ドキュメントの「[Getting Started](https://helm.sh/docs/chart_template_guide/getting_started/)」を参照してください。

### <a name="save-chart-to-local-registry-cache"></a>ローカル レジストリ キャッシュにグラフを保存する

ディレクトリを `hello-world` サブディレクトリに変更します。 次に、`helm chart save` を実行してグラフのコピーをローカルに保存し、レジストリの完全修飾名 (すべて小文字) とターゲット リポジトリおよびタグを使用して別名を作成します。 

次の例では、レジストリ名は *mycontainerregistry*、ターゲット リポジトリは *hello-world*、ターゲット グラフ タグは *v1* ですが、実際の環境に合わせて値を置き換えてください。

```console
cd ..
helm chart save . hello-world:v1
helm chart save . mycontainerregistry.azurecr.io/helm/hello-world:v1
```

`helm chart list` を実行して、ローカル レジストリ キャッシュにグラフが保存されたことを確認します。 出力は次のようになります。

```console
REF                                                      NAME            VERSION DIGEST  SIZE            CREATED
hello-world:v1                                           hello-world       0.1.0   5899db0 3.2 KiB        2 minutes 
mycontainerregistry.azurecr.io/helm/hello-world:v1       hello-world       0.1.0   5899db0 3.2 KiB        2 minutes
```

### <a name="authenticate-with-the-registry"></a>レジストリで認証する

Helm 3 CLI で `helm registry login` コマンドを実行して、実際のシナリオに合った適切な資格情報を使用して[レジストリで認証](container-registry-authentication.md)を行います。

たとえば、レジストリに対する[プルおよびプッシュ権限 (AcrPush ロール) がある Azure Active Directory サービス プリンシパル](container-registry-auth-service-principal.md#create-a-service-principal)を作成します。 次に、サービス プリンシパルの資格情報を `helm registry login` に対して指定します。 次の例では、環境変数を使用してパスワードを指定しています。

```console
echo $spPassword | helm registry login mycontainerregistry.azurecr.io \
  --username <service-principal-id> \
  --password-stdin
```

### <a name="push-chart-to-azure-container-registry"></a>Azure Container Registry にグラフをプッシュする

Helm 3 CLI で `helm chart push` コマンドを実行して、完全修飾ターゲット リポジトリにグラフをプッシュします。

```console
helm chart push mycontainerregistry.azurecr.io/helm/hello-world:v1
```

プッシュが成功すると、出力は次のようになります。

```output
The push refers to repository [mycontainerregistry.azurecr.io/helm/hello-world]
ref:     mycontainerregistry.azurecr.io/helm/hello-world:v1
digest:  5899db028dcf96aeaabdadfa5899db025899db025899db025899db025899db02
size:    3.2 KiB
name:    hello-world
version: 0.1.0
```

### <a name="list-charts-in-the-repository"></a>リポジトリのグラフ一覧

Azure コンテナー レジストリに格納されているイメージと同様に、[az acr repository][az-acr-repository] コマンドを使用して、グラフをホストしているリポジトリと、グラフのタグとマニフェストを表示できます。 

たとえば、[az acr repository show][az-acr-repository-show] を実行して、前の手順で作成したリポジトリのプロパティを表示します。

```azurecli
az acr repository show \
  --name mycontainerregistry \
  --repository helm/hello-world
```

出力は次のようになります。

```output
{
  "changeableAttributes": {
    "deleteEnabled": true,
    "listEnabled": true,
    "readEnabled": true,
    "writeEnabled": true
  },
  "createdTime": "2020-03-20T18:11:37.6701689Z",
  "imageName": "helm/hello-world",
  "lastUpdateTime": "2020-03-20T18:11:37.7637082Z",
  "manifestCount": 1,
  "registry": "mycontainerregistry.azurecr.io",
  "tagCount": 1
}
```

[az acr repository show-manifests][az-acr-repository-show-manifests] コマンドを実行して、リポジトリに格納されているグラフの詳細を表示します。 次に例を示します。

```azurecli
az acr repository show-manifests \
  --name mycontainerregistry \
  --repository helm/hello-world --detail
```

出力 (この例では省略されています) には、`application/vnd.cncf.helm.config.v1+json` の `configMediaType` が表示されます。

```output
[
  {
    [...]
    "configMediaType": "application/vnd.cncf.helm.config.v1+json",
    "createdTime": "2020-03-20T18:11:37.7167893Z",
    "digest": "sha256:0c03b71c225c3ddff53660258ea16ca7412b53b1f6811bf769d8c85a1f0663ee",
    "imageSize": 3301,
    "lastUpdateTime": "2020-03-20T18:11:37.7167893Z",
    "mediaType": "application/vnd.oci.image.manifest.v1+json",
    "tags": [
      "v1"
    ]
```

### <a name="pull-chart-to-local-cache"></a>ローカル キャッシュにグラフをプルする

Helm グラフを Kubernetes にインストールするには、グラフがローカル キャッシュにある必要があります。 この例では、最初に `helm chart remove` を実行して、`mycontainerregistry.azurecr.io/helm/hello-world:v1` という名前の既存のローカル グラフを削除します。

```console
helm chart remove mycontainerregistry.azurecr.io/helm/hello-world:v1
```

`helm chart pull` を実行して、Azure コンテナー レジストリからローカル キャッシュにグラフをダウンロードします。

```console
helm chart pull mycontainerregistry.azurecr.io/helm/hello-world:v1
```

### <a name="export-helm-chart"></a>Helm グラフをエクスポートする

グラフをさらに操作するには、`helm chart export` を使用してローカル ディレクトリにエクスポートします。 たとえば、プルしたグラフを `install` ディレクトリにエクスポートします。

```console
helm chart export mycontainerregistry.azurecr.io/helm/hello-world:v1 \
  --destination ./install
```

リポジトリにエクスポートされたグラフの情報を表示するには、グラフをエクスポートしたディレクトリで `helm show chart` コマンドを実行します。

```console
cd install
helm show chart hello-world
```

次の出力例に示されているように、グラフの最新バージョンに関する詳細情報が Helm によって返されます。

```output
apiVersion: v2
appVersion: 1.16.0
description: A Helm chart for Kubernetes
name: hello-world
type: application
version: 0.1.0    
```

### <a name="install-helm-chart"></a>Helm グラフをインストールする

ローカル キャッシュにプルしてエクスポートした Helm グラフをインストールするには、`helm install` を実行します。 *myhelmtest* などのリリース名を指定するか、`--generate-name` パラメーターを渡します。 次に例を示します。

```console
helm install myhelmtest ./hello-world
```

グラフのインストールが成功した後の出力は次のようになります。

```console
NAME: myhelmtest
LAST DEPLOYED: Fri Mar 20 14:14:42 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

インストールされていることを確認するには、`helm get manifest` コマンドを実行します。 このコマンドは、`configmap.yaml` テンプレート ファイル内の YAML データを返します。

クラスター上のグラフのリリースをアンインストールするには、`helm uninstall` を実行します。

```console
helm uninstall myhelmtest
```

### <a name="delete-a-helm-chart-from-the-repository"></a>Helm グラフをリポジトリから削除する

リポジトリからグラフを削除するには、[az acr repository delete][az-acr-repository-delete] コマンドを使用します。 次のコマンドを実行し、プロンプトが表示されたら操作を確認します。

```azurecli
az acr repository delete --name mycontainerregistry --image helm/hello-world:v1
```

## <a name="use-the-helm-2-client"></a>Helm 2 クライアントを使用する

### <a name="prerequisites"></a>前提条件

- Azure サブスクリプションの **Azure コンテナー レジストリ**。 必要に応じて、[Azure portal](container-registry-get-started-portal.md) または [Azure CLI](container-registry-get-started-azure-cli.md) を使用してレジストリを作成します。
- **Helm クライアント バージョン 2.11.0 (RC バージョンではない) 以降** - 現行バージョンを調べるには、`helm version` を実行してください。 また、Kubernetes クラスター内で開始される Helm サーバー (Tiller) も必要です。 必要に応じて、[Azure Kubernetes Service クラスター][aks-quickstart]を作成します。 Helm のインストール方法とアップグレード方法について詳しくは、「[Installing Helm (Helm のインストール)][helm-install-v2]」をご覧ください。
- **Azure CLI バージョン 2.0.46 以降** - バージョンを見つけるには `az --version` を実行してください。 インストールまたはアップグレードする必要がある場合は、[Azure CLI のインストール][azure-cli-install]に関するページを参照してください。

### <a name="high-level-workflow"></a>ワークフローの概要

**Helm 2** の場合:

* Azure コンテナー レジストリを "*単一*" の Helm グラフ リポジトリとして構成します。 グラフをリポジトリに追加および削除すると、Azure Container Registry によってインデックス定義が管理されます。
* Azure CLI 経由で Azure コンテナー レジストリを使用して認証します。これにより、レジストリの URI と資格情報を使用して Helm クライアントが自動的に更新されます。 このリポジトリ情報を手動で指定する必要はないため、資格情報はコマンドの履歴で公開されません。
* Azure CLI で [az acr helm][az-acr-helm] コマンドを使用して、Azure コンテナー レジストリを Helm グラフ リポジトリとして追加し、グラフをプッシュおよび管理します。 これらの Azure CLI コマンドを実行すると、Helm 2 クライアント コマンドがラップされます。
* Azure コンテナー レジストリのグラフ リポジトリをローカルの Helm リポジトリ インデックスに追加し、グラフの検索をサポートします。
* `helm install` を使用して、ローカル リポジトリ キャッシュから Kubernetes クラスターにグラフをインストールします。

例については、次のセクションを参照してください。

### <a name="add-repository-to-helm-client"></a>Helm クライアントにリポジトリを追加する

[az acr helm repo add][az-acr-helm-repo-add] コマンドを使用して Helm クライアントに Azure Container Registry Helm グラフのリポジトリを追加します。 このコマンドは、Helm クライアントによって使用される Azure コンテナー レジストリの認証トークンを取得します。 認証トークンは 3 時間有効です。 `docker login` と同様に、このコマンドを将来の CLI セッションで実行し、Azure Container Registry Helm のグラフレジストリで Helm クライアントを認証することができます。

```azurecli
az acr helm repo add --name mycontainerregistry
```

### <a name="add-a-sample-chart-to-the-repository"></a>サンプル グラフをリポジトリに追加する

最初に、ローカル ディレクトリを *~/acr-helm* に作成し、既存の *stable/wordpress* グラフ をダウンロードします。

```console
mkdir ~/acr-helm && cd ~/acr-helm
helm repo update
helm fetch stable/wordpress
```

「`ls`」と入力してダウンロードしたグラフを一覧表示し、ファイル名に含まれている Wordpress バージョンを確認します。 `helm fetch stable/wordpress` コマンドでは特定のバージョンを指定しなかったため、*最新*バージョンがフェッチされました。 次の出力例では、Wordpress グラフのバージョンは *8.1.0* です。

```output
wordpress-8.1.0.tgz
```

Azure CLI で [az acr helm push][az-acr-helm-push] コマンドを使用して Azure Container Registry の Helm グラフ リポジトリにグラフをプッシュします。 前の手順でダウンロードした Helm グラフの名前を *wordpress-8.1.0.tgz* などに指定します。

```azurecli
az acr helm push --name mycontainerregistry wordpress-8.1.0.tgz
```

しばらくすると、次の出力例に示すように Azure CLI によって、グラフが保存されていることが報告されます。

```output
{
  "saved": true
}
```

### <a name="list-charts-in-the-repository"></a>リポジトリのグラフ一覧

前の手順でアップロードしたグラフを使用するには、ローカルの Helm リポジトリのインデックスを更新する必要があります。 Helm クライアントでリポジトリのインデックスを更新したり、Azure CLI を使用してリポジトリのインデックスを更新することができます。 グラフを独自のリポジトリに追加するたびに、次の手順を完了する必要があります。

```azurecli
az acr helm repo add --name mycontainerregistry
```

リポジトリに保存されているグラフと、ローカルに使用できる更新されたインデックスを使用して、通常の Helm クライアント コマンドを使用して検索またはインストールを実行できます。 リポジトリ内のすべてのグラフを表示するには、独自の Azure Container Registry 名を指定して `helm search` コマンドを使用します。

```console
helm search mycontainerregistry
```

次の出力例に示すように、前の手順でプッシュした Wordpress グラフが一覧されます。

```output
NAME                  CHART VERSION    APP VERSION    DESCRIPTION
helmdocs/wordpress    8.1.0           5.3.2          Web publishing platform for building blogs and websites.
```

Azure CLI の [az acr helm list][az-acr-helm-list] を使用してグラフを一覧することもできます。

```azurecli
az acr helm list --name mycontainerregistry
```

### <a name="show-information-for-a-helm-chart"></a>Helm グラフの情報を表示する

リポジトリ内の特定のグラフの情報を表示するには、`helm inspect` コマンドを使用できます。

```console
helm inspect mycontainerregistry/wordpress
```

バージョン番号を指定しない場合、*最新*バージョンが使用されます。 次の縮約された出力例が示すように、グラフに関する詳細情報が Helm によって返されます。

```output
apiVersion: v1
appVersion: 5.3.2
description: Web publishing platform for building blogs and websites.
engine: gotpl
home: http://www.wordpress.com/
icon: https://bitnami.com/assets/stacks/wordpress/img/wordpress-stack-220x234.png
keywords:
- wordpress
- cms
- blog
- http
- web
- application
- php
maintainers:
- email: containers@bitnami.com
  name: Bitnami
name: wordpress
sources:
- https://github.com/bitnami/bitnami-docker-wordpress
version: 8.1.0
[...]
```

Azure CLI [az acr helm show][az-acr-helm-show] コマンドを使用してグラフの情報を表示することもできます。 ここでも*最新*バージョンのグラフが既定で返されます。 `--version` をアペンドすれば、*8.1.0* など、特定バージョンのグラフを一覧することができます。

```azurecli
az acr helm show --name mycontainerregistry wordpress
```

### <a name="install-a-helm-chart-from-the-repository"></a>Helm グラフをリポジトリからインストールする

リポジトリ内の Helm グラフは、リポジトリ名を指定して、グラフ名を指定することでインストールされます。 Wordpress のグラフをインストールするには、次のように Helm クライアントを使用します。

```console
helm install mycontainerregistry/wordpress
```

> [!TIP]
> Azure Container Registry の Helm グラフ リポジトリにプッシュして、後で新しい CLI セッションに戻る場合、ローカルの Helm クライアントは更新された認証トークンを必要とします。 新しい認証トークンを取得するには、[az acr helm repo add][az-acr-helm-repo-add] コマンドを使用します。

インストール プロセス中には、次の手順が完了します。

- Helm クライアントは、ローカル リポジトリのインデックスを検索します。
- 対応するグラフが Azure Container Registry リポジトリからダウンロードされます。
- グラフは、Tiller を使用して Kubernetes クラスターにデプロイされます。

インストールが進んだら、コマンド出力の指示に従って WorPress の URL と資格情報を確認します。 また、`kubectl get pods` コマンドを実行して、Helm グラフを介してデプロイされた Kubernetes リソースを確認することもできます。

```output
NAME                                    READY   STATUS    RESTARTS   AGE
wordpress-1598530621-67c77b6d86-7ldv4   1/1     Running   0          2m48s
wordpress-1598530621-mariadb-0          1/1     Running   0          2m48s
[...]
```

### <a name="delete-a-helm-chart-from-the-repository"></a>Helm グラフをリポジトリから削除する

リポジトリからグラフを削除するには、[az acr helm delete][az-acr-helm-delete] コマンドを使用します。 グラフの名前 (*wordpress* など) および削除するバージョン (*8.1.0* など) を指定します。

```azurecli
az acr helm delete --name mycontainerregistry wordpress --version 8.1.0
```

名前付きのグラフのすべてのバージョンを削除する場合は、`--version` パラメーターを除外します。

`helm search` を実行すると、引き続きグラフが返されます。 ここでも、Helm クライアントは、リポジトリで使用可能なグラフの一覧を自動的に更新することはありません。 Helm クライアントのリポジトリ インデックスを更新するには、同じく [az acr helm repo add][az-acr-helm-repo-add] コマンドを使用します。

```azurecli
az acr helm repo add --name mycontainerregistry
```

## <a name="next-steps"></a>次のステップ

* Helm グラフを作成してデプロイする方法の詳細については、[Helm グラフの開発][develop-helm-charts]に関するページを参照してください。
* [Azure Kubernetes Service (AKS)](../aks/kubernetes-helm.md) での Helm を使用したアプリケーションのインストールについて詳細を確認します。
* Helm グラフは、コンテナーのビルド プロセスの一部として使用できます。 詳細については、[Azure Container Registry タスクの使用][acr-tasks]に関するページを参照してください。

<!-- LINKS - external -->
[helm]: https://helm.sh/
[helm-install]: https://helm.sh/docs/intro/install/
[helm-install-v2]: https://v2.helm.sh/docs/using_helm/#installing-helm
[develop-helm-charts]: https://helm.sh/docs/chart_template_guide/
[semver2]: https://semver.org/
[terms-of-use]: https://azure.microsoft.com/support/legal/preview-supplemental-terms/

<!-- LINKS - internal -->
[azure-cli-install]: /cli/azure/install-azure-cli
[aks-quickstart]: ../aks/kubernetes-walkthrough.md
[acr-bestpractices]: container-registry-best-practices.md
[az-configure]: /cli/azure/reference-index#az-configure
[az-acr-login]: /cli/azure/acr#az-acr-login
[az-acr-helm]: /cli/azure/acr/helm
[az-acr-repository]: /cli/azure/acr/repository
[az-acr-repository-show]: /cli/azure/acr/repository#az-acr-repository-show
[az-acr-repository-delete]: /cli/azure/acr/repository#az-acr-repository-delete
[az-acr-repository-show-tags]: /cli/azure/acr/repository#az-acr-repository-show-tags
[az-acr-repository-show-manifests]: /cli/azure/acr/repository#az-acr-repository-show-manifests
[az-acr-helm-repo-add]: /cli/azure/acr/helm/repo#az-acr-helm-repo-add
[az-acr-helm-push]: /cli/azure/acr/helm#az-acr-helm-push
[az-acr-helm-list]: /cli/azure/acr/helm#az-acr-helm-list
[az-acr-helm-show]: /cli/azure/acr/helm#az-acr-helm-show
[az-acr-helm-delete]: /cli/azure/acr/helm#az-acr-helm-delete
[acr-tasks]: container-registry-tasks-overview.md
