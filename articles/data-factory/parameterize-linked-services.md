---
title: Azure Data Factory のリンクされたサービスのパラメーター化
description: Azure Data Factory のリンクされたサービスをパラメーター化し、実行時に動的な値を渡す方法について説明します。
services: data-factory
documentationcenter: ''
ms.service: data-factory
ms.workload: data-services
ms.topic: conceptual
ms.date: 12/18/2018
author: djpmsft
ms.author: daperlov
manager: anandsub
ms.openlocfilehash: 8fa8603f2bee7e42db0f085d78117d61bd14ce5c
ms.sourcegitcommit: 849bb1729b89d075eed579aa36395bf4d29f3bd9
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 04/28/2020
ms.locfileid: "81419393"
---
# <a name="parameterize-linked-services-in-azure-data-factory"></a>Azure Data Factory のリンクされたサービスのパラメーター化
[!INCLUDE[appliesto-adf-asa-md](includes/appliesto-adf-asa-md.md)]

リンクされたサービスをパラメーター化し、実行時に動的な値を渡せるようになりました。 たとえば、同じ Azure SQL Database サーバー上の異なるデータベースに接続する場合に、リンクされたサービスの定義内でデータベース名をパラメーター化することができます。 これにより、Azure SQL データベースサーバー上のデータベースごとに、リンクされたサービスを作成する必要がなくなります。 リンクされたサービスの定義内で、他のプロパティをパラメーター化することもできます (たとえば、*ユーザー名*など)。

リンクされたサービスをパラメーター化するには、Azure portal の Data Factory UI か、プログラミング インターフェイスを使用できます。

> [!TIP]
> パスワードやシークレットはパラメーター化しないようにすることをお勧めします。 接続文字列はすべて Azure Key Vault 内に格納し、*シークレット名*をパラメーター化するようにしてください。

この機能の概要とデモンストレーションについては、以下の 7 分間の動画を視聴してください。

> [!VIDEO https://channel9.msdn.com/shows/azure-friday/Parameterize-connections-to-your-data-stores-in-Azure-Data-Factory/player]

## <a name="supported-data-stores"></a>サポートされているデータ ストア

現在のところ、リンクされたサービスのパラメーター化は、次のデータ ストアを対象に、Azure portal の Data Factory UI でサポートされています。 その他のすべてのデータ ストアについては、 **[接続]** タブで **[コード]** アイコンを選択し、JSON エディターを使用することで、リンクされたサービスをパラメーター化できます。
- Azure SQL データベース
- Azure SQL Data Warehouse
- SQL Server
- Oracle
- Cosmos DB
- Amazon Redshift
- MySQL
- Azure Database for MySQL

## <a name="data-factory-ui"></a>Data Factory UI

![リンクされたサービスの定義に動的なコンテンツを追加する](media/parameterize-linked-services/parameterize-linked-services-image1.png)

![新しいパラメーターを作成する](media/parameterize-linked-services/parameterize-linked-services-image2.png)

## <a name="json"></a>JSON

```json
{
    "name": "AzureSqlDatabase",
    "properties": {
        "type": "AzureSqlDatabase",
        "typeProperties": {
            "connectionString": "Server=tcp:myserver.database.windows.net,1433;Database=@{linkedService().DBName};User ID=user;Password=fake;Trusted_Connection=False;Encrypt=True;Connection Timeout=30"
        },
        "connectVia": null,
        "parameters": {
            "DBName": {
                "type": "String"
            }
        }
    }
}
```
