---
title: AutoML でのオーバーフィット データと不均衡データの回避
titleSuffix: Azure Machine Learning
description: Azure Machine Learning の自動機械学習ソリューションを使用して、ML モデルの一般的な落とし穴を特定し、管理します。
services: machine-learning
ms.service: machine-learning
ms.subservice: core
ms.topic: conceptual
ms.reviewer: nibaccam
author: nibaccam
ms.author: nibaccam
ms.date: 04/09/2020
ms.openlocfilehash: 76f920ad6aae68defb567a7a6623d1ffd488af5f
ms.sourcegitcommit: 2d7910337e66bbf4bd8ad47390c625f13551510b
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 04/08/2020
ms.locfileid: "80874859"
---
# <a name="prevent-overfitting-and-imbalanced-data-with-automated-machine-learning"></a>自動機械学習でのオーバーフィット データと不均衡データを防止する

機械学習モデルを構築する際に陥りやすい落とし穴が、オーバーフィットや偏ったデータです。 既定では、Azure Machine Learning の自動機械学習は、これらのリスクを特定するのに役立つグラフとメトリックを提供し、リスクを軽減するためのベスト プラクティスを実装します。 

## <a name="identify-over-fitting"></a>オーバーフィットを特定する

機械学習のオーバーフィットは、モデルがトレーニング データに過剰にフィットする場合に発生します。結果として、未見のテスト データで正確な予測を実行できなくなります。 言い換えると、モデルは、トレーニング データ内の特定のパターンや不要な情報を単に記憶しただけで、実際のデータで予測を行うだけの十分な柔軟性がありません。

次のトレーニング済みモデルと、それに対応するトレーニングとテストの精度について検討します。

| モデル | トレーニングの精度 | テストの精度 |
|-------|----------------|---------------|
| A | 99.9% | 95% |
| B | 87% | 87% |
| C | 99.9% | 45% |

モデル **A** では、未見のデータに対するテストの精度が、トレーニングの精度よりも低くなっています。この場合に、モデルがオーバーフィットしていると認識するのはよくある誤認です。 ただし、テストの精度は常にトレーニングの精度よりも低くなるべきであり、*どの程度*精度が低いかによって、オーバーフィットしているか、適切にフィットしているか見分けます。 

モデル **A** とモデル **B** を比較した場合、モデル **A** の方が適切なモデルです。これは、テストの精度がより高く、またテストの精度が 95% と少し低めですが、オーバーフィットが発生していることを示す大きな差がないためです。 トレーニングの精度とテストの精度が非常に近いという単純な理由で、モデル **B** は選択しません。

モデル **C** は、オーバーフィットが発生していることを明確に表しています。トレーニングの精度が非常に高い一方で、テストの精度はまったく高くありません。 この見分け方は主観的ですが、問題とデータの知識、および許容できる誤差の大きさから得られるものです。

## <a name="prevent-over-fitting"></a>オーバーフィットを防ぐ

最も極端なケースでは、オーバーフィットしたモデルは、トレーニング中に検出された特徴値の組み合わせが、ターゲットのまったく同じ出力に常に生成されると見なします。

オーバーフィットを防ぐ最も適切な方法は、次のような ML のベスト プラクティスに従うことです。

* より多くのトレーニング データを使用し、統計的偏りを排除する
* ターゲット漏えいを防ぐ
* 使用する機能の削減
* **正則化とハイパーパラメーターの最適化**
* **モデルの複雑さの制限**
* **クロス検証**

自動 ML のコンテキストでは、上記の最初の 3 つの項目は、**ユーザーが実装するベスト プラクティス**です。 最後の 3 つの太字の項目は、オーバーフィットを防ぐために既定で**自動 ML により実装されるベスト プラクティス**です。 自動 ML 以外の設定でも、モデルのオーバーフィットを回避するために、6 つのベスト プラクティスすべてに従うことをお勧めします。

### <a name="best-practices-you-implement"></a>ユーザーが実装するベスト プラクティス

**より多くのデータ**を使用することは、オーバーフィットを防ぐための最も簡単で最も効果的な方法です。また、追加の利点として、通常は精度が向上します。 より多くのデータを使用すると、モデルは正確なパターンを記憶することが困難になり、より多くの条件に対応するより柔軟な解を導き出さざるを得なくなります。 また、**統計的偏り**を認識して、実際の予測データに存在しないような分離パターンがトレーニング データに含まれないようにすることも重要です。 このシナリオは解決するのが困難な場合があります。これは、トレーニング セットとテスト セットの間でオーバーフィットが発生しない可能性がある一方で、実際のテスト データと比較した場合に、オーバーフィットが発生する可能性があるためです。

ターゲット漏えいは同様の問題であり、トレーニング セットとテスト セットの間でオーバーフィットが確認されない一方で、予測時にこれが現れる可能性があります。 ターゲット漏えいは、モデルが予測時には通常使用しないデータにトレーニング中にアクセスすることで、"ずる" を行うと発生します。 たとえば、問題が、金曜日に商品価格がどのようになるかを月曜日に予測することであるときに、いずれかの特徴に木曜日のデータが誤って含まれている場合、それは予測時にモデルが使用しないデータです。未来は調べることができないためです。 ターゲット漏えいは、見逃しやすいミスですが、問題に対して異常に精度が高いという特色が多くの場合発生します。 株価を予測し、95% の精度でモデルをトレーニングした場合、特徴のどこかにターゲット漏えいがある可能性が高くなります。

特徴を削除することが、オーバーフィットで役立つ場合もあります。これは、モデルで特定のパターンを記憶するために使用するフィールドが多くなり過ぎるのを防ぎ、柔軟性を向上させます。 定量的な測定は困難な場合がありますが、特徴を削除しても同じ精度を維持できる場合、モデルの柔軟性が向上し、オーバーフィットのリスクが軽減された可能性が高いと言えます。

### <a name="best-practices-automated-ml-implements"></a>自動 ML により実装されるベスト プラクティス

正則化は、オーバーフィットした複雑なモデルにペナルティーを科すためのコスト関数を最小化するプロセスです。 さまざまな種類の正則化関数がありますが、一般に、これらはすべて、モデルの係数サイズ、分散、および複雑さに対してペナルティーを科します。 自動 ML では、オーバーフィットを制御するさまざまなモデル ハイパーパラメーター設定と共に、L1 (Lasso)、L2 (Ridge)、および ElasticNet (L1 と L2 を同時使用) がさまざまな組み合わせで使用されます。 簡単に言うと、自動 ML は、モデルをどの程度規制するかを変えて、最適な結果を選択します。

自動 ML では、オーバーフィットを防ぐために、明示的なモデルの複雑さの制限も実装されます。 ほとんどの場合、この実装は、特にデシジョン ツリーまたはフォレスト アルゴリズム用です。ここでは、個々のツリーの最大深度が制限され、さらにフォレストまたはアンサンブル手法で使用されるツリーの合計数が制限されます。

クロス検証 (CV) は、全トレーニング データのサブセットを多く取得し、各サブセットでモデルをトレーニングするプロセスです。 これは、モデルは、"運がよい" 場合に、1 つのサブセットで高い精度を実現できる可能性があるが、多くのサブセットを使用することで、モデルはこの高い精度を毎回は達成できないという考え方です。 CV を実行する場合、検証の予約データセットを指定し、CV フォールド (サブセットの数) を指定します。自動 ML は、モデルをトレーニングし、ハイパーパラメーターを調整して検証セットのエラーを最小限に抑えます。 CV フォールドが 1 つでは、オーバーフィットとなる可能性がありますが、それらを多数使用することで、最終的なモデルがオーバーフィットする確率は低下します。 このトレードオフとして、CV ではトレーニング時間が長くなり、コストが高くなります。これは、モデルを一度にトレーニングするのではなく、*n* 個の各 CV サブセットで 1 回ずつトレーニングするためです。 

> [!NOTE]
> クロス検証は既定では有効になっていません。自動 ML 設定で構成する必要があります。 ただし、クロス検証を構成し、検証データセットを用意した後は、プロセスは自動化されます。 参照先 

<a name="imbalance"></a>

## <a name="identify-models-with-imbalanced-data"></a>偏ったデータを含むモデルを特定する

偏ったデータは、機械学習の分類シナリオのデータでよく見られるもので、各クラスに不適切な観測比率を含むデータを意味します。 この不均衡によって、入力データが 1 つのクラスに偏り、トレーニングされたモデルがその偏りを再現するため、モデルの精度に擬陽性の影響が生じる可能性があります。 

分類アルゴリズムは通常、精度によって評価されるため、モデルの精度スコアを確認することは、偏ったデータの影響を受けたかどうかを特定するのに適した方法です。 特定のクラスはとても高精度またはとても低精度でしたか?

さらに、自動 ML を実行すると、次のグラフが自動的に生成されます。これは、モデルの分類の正確性を把握し、偏ったデータの影響を受ける可能性のあるモデルを特定するのに役立ちます。

グラフ| 説明
---|---
[混同行列](how-to-understand-automated-ml.md#confusion-matrix)| 適切に分類されたラベルをデータの実際のラベルと比較して評価します。 
[精度/再現率](how-to-understand-automated-ml.md#precision-recall-chart)| 適切なラベルの比率を、検出されたデータのラベル インスタンスの比率と比較して評価します 
[ROC 曲線](how-to-understand-automated-ml.md#roc)| 適切なラベルの比率を、擬陽性ラベルの比率と比較して評価します。

## <a name="handle-imbalanced-data"></a>偏ったデータの処理 

機械学習ワークフローを簡略化する目標の一環として、自動 ML には、偏ったデータを処理するための以下のような機能が組み込まれています。 

- **重み列**: 自動 ML では、重み列が入力としてサポートされているため、データ内の行が重み付けされ、クラスの "重要度" を増減できます。

- 自動 ML によって使用されるアルゴリズムでは、最大 20:1 の不均衡を適切に処理できます。つまり、最も一般的なクラスは、最も一般的でないクラスの 20 倍の行をデータ内に持つことができます。

次の手法は、自動 ML の外部で偏ったデータを処理するための追加のオプションです。 

- より小さいクラスをアップサンプリングするか、より大きいクラスをダウンサンプリングすることで、クラスの偏りを均等にするための再サンプリング。 これらの方法では、処理および分析するための専門知識が必要です。

- 偏ったデータをより適切に処理するパフォーマンス メトリックを使用します。 たとえば、F1 スコアは、精度とリコールの加重平均です。 精度では分類子の正確性が測定され、低精度は擬陽性の数が多いことを示します。リコールでは分類子の完全性が測定されて、低いリコールは擬陰性の数が多いことを示します。 

## <a name="next-steps"></a>次のステップ

例を参照して、自動化された機械学習を使用してモデルを構築する方法を学習してください。

+ 次のチュートリアルを修了してください。[チュートリアル:Azure Machine Learning で回帰モデルを自動的にトレーニングする](tutorial-auto-train-models.md)

+ 自動トレーニング実験の設定を構成してください。
  + Azure Machine Learning Studio で、[こちらの手順](how-to-use-automated-ml-for-ml-models.md)を使用します。
  + Python SDK で、[こちらの手順](how-to-configure-auto-train.md)を使用します。


