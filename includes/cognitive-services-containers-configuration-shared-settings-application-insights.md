---
author: IEvangelist
ms.author: dapine
ms.date: 06/25/2019
ms.service: cognitive-services
ms.topic: include
ms.openlocfilehash: 046c5c5e32c71364e1bf54551989e19eec4937e4
ms.sourcegitcommit: 2ec4b3d0bad7dc0071400c2a2264399e4fe34897
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 03/28/2020
ms.locfileid: "67712609"
---
`ApplicationInsights` 設定を使用すると、[Azure Application Insights](https://docs.microsoft.com/azure/application-insights) テレメトリのサポートをお客様のコンテナーに追加できます。 Application Insights によってお客様のコンテナーを詳細に監視できます。 コンテナーの可用性、パフォーマンス、利用状況を簡単に監視できます。 さらに、お客様のコンテナーのエラーを迅速に特定して診断することもできます。

次の表に、`ApplicationInsights` セクションでサポートされている構成設定について説明します。

|必須| Name | データ型 | 説明 |
|--|------|-----------|-------------|
|いいえ| `InstrumentationKey` | String | コンテナーのテレメトリ データの送信先の Application Insights インスタンスのインストルメンテーション キー。 詳細については、「[Application Insights for ASP.NET Core](https://docs.microsoft.com/azure/application-insights/app-insights-asp-net-core)」を参照してください。 <br><br>例:<br>`InstrumentationKey=123456789`|

