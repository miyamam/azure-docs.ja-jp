---
title: チュートリアル - Azure Linux Virtual Machines のカナリア デプロイを構成する
description: このチュートリアルでは、ブルーグリーン デプロイ戦略を使用して、Azure 仮想マシンのグループを更新する継続的デプロイ (CD) パイプラインのセットアップ方法について説明します。
author: moala
manager: jpconnock
tags: azure-devops-pipelines
ms.assetid: ''
ms.service: virtual-machines-linux
ms.topic: tutorial
ms.tgt_pltfrm: azure-pipelines
ms.workload: infrastructure
ms.date: 4/10/2020
ms.author: moala
ms.custom: devops
ms.openlocfilehash: b1a57245434bb188ffaab56a8891b4b0ee27f044
ms.sourcegitcommit: 58faa9fcbd62f3ac37ff0a65ab9357a01051a64f
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 04/29/2020
ms.locfileid: "82120550"
---
# <a name="tutorial---configure-blue-green-deployment-strategy-for-azure-linux-virtual-machines"></a>チュートリアル - Azure Linux Virtual Machines のブルーグリーン デプロイ戦略を構成する


## <a name="iaas---configure-cicd"></a>IaaS - CI/CD の構成 
Azure Pipelines では、仮想マシンにデプロイするための機能が完備された CI/CD 自動化ツールのセットが提供されます。 Azure VM に対する継続的デリバリー パイプラインは、Azure portal から直接構成できます。 このドキュメントでは、ブルーグリーン戦略を使用して複数マシンをデプロイする CI/CD パイプラインの設定に関連した手順を説明します。 [ローリング](https://aka.ms/AA7jlh8)や[カナリア](https://aka.ms/AA7jdrz)など、特別な設定なしに Azure portal から使用できる他の戦略もご覧いただけます。 

 
 **仮想マシンで CI/CD を構成する**

仮想マシンは、[デプロイ グループ](https://docs.microsoft.com/azure/devops/pipelines/release/deployment-groups)にターゲットとして追加することができます。また、複数マシンの更新の対象にすることができます。 デプロイ後は、デプロイ グループ内の**デプロイ履歴**を見ることで、VM からパイプライン、さらにコミットまでの追跡が可能になります。 
 
  
**ブルーグリーン デプロイ**: ブルーグリーン デプロイでは、同一のスタンバイ環境を設けることでダウンタイムを短くします。 どの時点でも、いずれか 1 つの環境が有効になります。 新リリースに向けた準備の過程で、テストの最終ステージをグリーン環境で行います。 グリーン環境でソフトウェアが正常に動作していれば、トラフィックを切り替えて、すべて受信要求をグリーン環境に誘導します。この時点で、ブルー環境はアイドルとなります。
"**仮想マシン**" へのブルーグリーン デプロイは、Azure portal から継続的デリバリー オプションを使用して構成することができます。 

その詳細な手順は次のとおりです。

1. Azure portal にサインインして仮想マシンに移動します。 
2. VM の左ペインで、 **[継続的デリバリー]** に移動します。 **[構成]** をクリックします。 

   ![AzDevOps_configure](media/tutorial-devops-azure-pipelines-classic/azure-devops-configure.png) 
3. 構成パネルで、 **[Azure DevOps 組織]** をクリックして既存のアカウントを選択するか、新たに作成します。 次に、パイプラインを構成するプロジェクトを選択します。  


   ![AzDevOps_project](media/tutorial-devops-azure-pipelines-classic/azure-devops-rolling.png) 
4. デプロイ グループは、"開発"、"テスト"、"UAT"、"運用" などの物理環境を表す、デプロイ ターゲット マシンの論理的な集まりです。 新しいデプロイ グループを作成することも、既存のデプロイ グループを選択することもできます。 
5. 仮想マシンにデプロイするパッケージを発行するビルド パイプラインを選択します。 発行されたパッケージには、パッケージ ルートの `deployscripts` フォルダーに _deploy.ps1_ または _deploy.sh_ のデプロイ スクリプトが含まれている必要があることに注意してください。 このデプロイ スクリプトは、実行時に Azure DevOps パイプラインによって実行されます。
6. 任意のデプロイ方法を選択します。 **[ブルーグリーン]** を選択します。
7. ブルーグリーン デプロイに含める VM に "blue (ブルー)" または "green (グリーン)" タグを追加します。 VM がスタンバイロール用である場合は、"green (グリーン)" のタグを、それ以外の場合は "blue (ブルー)" 付ける必要があります。
![AzDevOps_bluegreen_configure](media/tutorial-devops-azure-pipelines-classic/azure-devops-blue-green-configure.png)

8. 継続的デリバリー パイプラインを構成するには、 **[OK]** をクリックします。 これで仮想マシンにデプロイするよう継続的デリバリー パイプラインが構成されます。
![AzDevOps_bluegreen_pipeline](media/tutorial-devops-azure-pipelines-classic/azure-devops-blue-green-pipeline.png)


9. Azure DevOps のリリース パイプラインの **[編集]** をクリックして、パイプラインの構成を確認します。 パイプラインは、3 つのフェーズで構成されています。 最初のフェーズはデプロイ グループ フェーズで、"_green (グリーン)_ " タグが付けられた VM (スタンバイ VM) にデプロイされます。 2 番目のフェーズでは、パイプラインを一時停止し、実行を再開するための手動介入を待機します。 ユーザーは、デプロイが安定していることを確認した後、"_green (グリーン)_ " VM にトラフィックをリダイレクトし、パイプラインの実行を再開できます。これにより、VM の "_blue (ブルー)_ " タグと "_green (グリーン)_ " タグが入れ替えられます。 これにより、アプリケーションのバージョンが古い VM に "_green (グリーン)_ " のタグが付けられ、次のパイプライン実行でのデプロイ先になります。
![AzDevOps_bluegreen_task](media/tutorial-devops-azure-pipelines-classic/azure-devops-blue-green-tasks.png)


10. デプロイ スクリプトの実行タスクでは、既定で、発行されたパッケージのルート ディレクトリの `deployscripts` フォルダー内の _deploy.ps1_ または _deploy.sh_ デプロイ スクリプトが実行されます。 選択したビルド パイプラインによって、パッケージのルート フォルダーにそれが発行されていることを確認してください。
![AzDevOps_publish_package](media/tutorial-deployment-strategy/package.png)




## <a name="other-deployment-strategies"></a>その他のデプロイ戦略
- [ローリング デプロイ戦略を構成する](https://aka.ms/AA7jlh8)
- [カナリア デプロイ戦略を構成する](https://aka.ms/AA7jdrz)

## <a name="azure-devops-project"></a>Azure DevOps プロジェクト 
Azure は、これまでよりも簡単に始められるようになりました。
 
DevOps Projects を使用すれば、3 つのステップ (アプリケーション言語、ランタイム、Azure サービスの選択) だけで、どの Azure サービスでもアプリケーションの実行を開始できます。
 
[詳細については、こちらを参照してください](https://azure.microsoft.com/features/devops-projects/ )。
 
## <a name="additional-resources"></a>その他のリソース 
- [DevOps プロジェクトを使用して Azure 仮想マシンにデプロイする](https://docs.microsoft.com/azure/devops-project/azure-devops-project-vms)
- [Azure 仮想マシン スケール セットへのアプリの継続的デプロイを導入する](https://docs.microsoft.com/azure/devops/pipelines/apps/cd/azure/deploy-azure-scaleset)
