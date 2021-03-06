---
title: 回復性に優れた Azure 用アプリケーションの設計
description: 高可用性とディザスター リカバリーのために Azure で回復性に優れたアプリケーションを構築する方法。
author: MikeWasson
ms.date: 07/29/2018
ms.custom: resiliency
ms.openlocfilehash: b925748e1d3d4a8d490bbd5d7cb76f3961ffcfb2
ms.sourcegitcommit: dbbf914757b03cdee7a274204f9579fa63d7eed2
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 11/02/2018
ms.locfileid: "50916602"
---
# <a name="designing-resilient-applications-for-azure"></a>回復性に優れた Azure 用アプリケーションの設計

分散システムでは、障害が発生します。 ハードウェアの障害が発生することがあります。 ネットワークに一時的な障害が起きることがあります。 まれに、サービス全体またはリージョン全体で中断が発生する可能性がありますが、こうしたことについても計画しておく必要があります。 

クラウドで信頼性の高いアプリケーションの構築は、エンタープライズ設定で信頼性の高いアプリケーションの構築とは異なります。 従来は、スケールアップのためにハイエンドなハードウェアを購入していましたが、クラウド環境では、スケールアップではなくスケールアウトする必要があります。 クラウド環境は、汎用的なハードウェアを使用することで低コストを維持できます。 この新しい環境では、エラーを回避し、"平均故障間隔" を最適化することではなく、"復元までの平均時間" に集中できます。 目標は、障害の影響を最小限に抑えることです。

この記事では、Microsoft Azure で回復性に優れたアプリケーションを構築する方法の概要について説明します。 まず、*回復性*という用語の定義と関連する概念から説明します。 次に、設計、実装からデプロイ、運用までというアプリケーションの有効期間全体に体系的な方法を使用することで、回復性を実現するプロセスについて説明します。

## <a name="what-is-resiliency"></a>回復性とは
**回復性**とは、障害から回復して動作を続行する、システムの能力です。 障害を*回避*することではなく、ダウンタイムまたはデータの損失を回避するように障害に*対応*することです。 回復性の目的は、障害後にアプリケーションを完全に機能している状態に戻すことです。

回復性には、高可用性とディザスター リカバリーという 2 つの重要な側面があります。

* **高可用性** (HA) は、大幅なダウンタイムなしに、正常な状態で実行を継続するアプリケーションの能力です。 "正常な状態" とは、アプリケーションが応答し、ユーザーがアプリケーションに接続して対話できるという意味です。  
* **ディザスター リカバリー** (DR) は、まれに発生する大きなインシデント、すなわち地域全体に影響するサービス中断のように、一時的なものではない大規模な障害から回復する能力です。 ディザスター リカバリーには、データ バックアップやアーカイブの他に、バックアップからのデータベースの復元などの手動操作も含まれる場合があります。

HA と DR のとらえ方として、障害の影響が HA 設計で対応できる限度を超えたときに DR が始まると考えることができます。  

回復性を設計するときは、ご自分の可用性要件を理解する必要があります。 どの程度のダウンタイムを許容できるのでしょうか。 これは、ある程度はコストによって決まります。 起こり得るダウンタイムによって貴社のビジネスが被るコストはいくらになるでしょうか。 アプリケーションの可用性を高めることにいくら投資したいですか。 また、何をもってアプリケーションが使用可能であるとするかについても定義する必要があります。 たとえば、ユーザーが注文を送信しても、通常の時間枠内にシステムで処理できない場合、アプリケーションは "停止" 状態でしょうか。 また、特定の種類の障害が発生する可能性と、リスク軽減戦略のコスト効果が高いかどうかについても考慮します。

もう 1 つの一般的な用語に**ビジネス継続性** (BC) があります。BC とは、自然災害やサービスの停止などの悪条件の発生時と発生後に、重要なビジネス機能を実行できることです。 BC は、物理的な設備、人、コミュニティ、運輸、IT を含め、業務全体が対象です。 この記事ではクラウド アプリケーションを中心に扱いますが、回復性の計画は BC 要件全体のコンテキストで行う必要があります。 

**データ バックアップ**は、DR の重要な部分です。 アプリケーションのステートレスなコンポーネントが失敗した場合、そのコンポーネントはいつでも再展開できます。 ただし、データが失われると、システムは安定した状態に戻ることはできません。 リージョン全体の障害に備えて、理想的には異なるリージョンにデータをバックアップする必要があります。 

バックアップは、**データ レプリケーション**とは異なります。 データ レプリケーションでは、システムが迅速にレプリカにフェールオーバーできるように、ほぼリアルタイムでデータがコピーされます。 多くのデータベース システムがレプリケーションをサポートしています。たとえば、SQL Server では、SQL Server Always On 可用性グループがサポートされます。 データ レプリケーションでは、データのレプリカを確実かつ常に待機させることで、障害からの復旧時間を短縮できます。 しかし、データ レプリケーションで人的ミスを防ぐことはできません。 人的ミスによりデータが破損すると、その破損したデータはレプリカにコピーされます。 このため、DR 戦略には引き続き長期的なバックアップを含める必要があります。 

## <a name="process-to-achieve-resiliency"></a>回復性を実現するプロセス
回復性はアドオンではありません。 システムの設計に追加し、運用方法に組み込む必要があります。 一般的なモデルは次のとおりです。

1. **定義**: ビジネスのニーズに基づいて可用性の要件を定義します。
2. **設計**: 回復性を考慮してアプリケーションを設計します。 実証済みの手法に従ったアーキテクチャから始めて、そのアーキテクチャで考えられる障害点を特定します。
3. **実装**: 障害を検出し、回復する戦略を実装します。 
4. **テスト**: 障害をシミュレートし、強制的なフェールオーバーをトリガーして実装をテストします。 
5. **デプロイ**: 信頼性が高い反復可能なプロセスを使用して、アプリケーションを運用環境にデプロイします。 
6. **監視**: アプリケーションを監視して障害を検出します。 システムを監視することで、アプリケーションの正常性を評価し、必要に応じてインシデントに対応できます。 
7. **対応**: 手動の介入が必要な障害が発生した場合に対応します。

以降、これらの各手順について詳しく説明します。

## <a name="define-your-availability-requirements"></a>可用性の要件を定義する
回復性の計画は、ビジネス要件から始まります。 ビジネス要件という点で回復性について考える場合の方法をいくつか挙げます。

### <a name="decompose-by-workload"></a>ワークロードごとに分解する
多くのクラウド ソリューションは、複数のアプリケーション ワークロードで構成されます。 このコンテキストで "ワークロード" という用語は、ビジネス ロジックとデータ ストレージ要件の点で、他のタスクとは論理的に区別できる個別の機能またはコンピューティング タスクを意味します。 たとえば、eコマース アプリには次のようなワークロードがあります。

* 製品カタログの閲覧と検索。
* 注文の作成と追跡。
* 推奨事項の表示。

これらのワークロードは、可用性、スケーラビリティ、データの一貫性、ディザスター リカバリーなどの要件が異なる可能性があります。 これらもビジネス上の決定事項です。

また、使用パターンについても考慮します。 システムが使用可能である必要がある特定の重要な期間はありますか。 たとえば、税金申告書サービスが申告期限の直前に停止してはならない、大規模なスポーツ イベント中にビデオ ストリーミング サービスは稼働している必要がある、などです。 重要な期間中は、複数のリージョンにまたがる冗長的なデプロイにして、1 つのリージョンで障害が発生してもアプリケーションがフェールオーバーできるようにすることができます。 ただし、複数リージョンのデプロイはコストが高くなるため、重要な期間以外は、1 つのリージョンでアプリケーションを実行することができます。

### <a name="rto-and-rpo"></a>RTO と RPO
考慮すべき 2 つの重要なメトリックとして、目標復旧時間と回復ポイントの目標があります。

* **目標復旧時間** (RTO) は、インシデントが発生した後に、アプリケーションに許容する使用不能状態の最大時間です。 RTO が 90 分の場合、災害発生から 90 分以内にアプリケーションを実行状態に復元できる必要があります。 RTO を非常に低い値にする場合、リージョン全体の停止から保護するために、スタンバイ状態で継続的に動作する第 2 のデプロイを維持する必要があります。

* **回復ポイントの目標** (RPO) は、災害発生時に許容できるデータ損失の最大期間です。 たとえば、単一のデータベースにデータを格納し、他のデータベースへのレプリケーションなしで、毎時のバックアップを実行する場合、最大 1 時間のデータを失う可能性があります。 

RTO と RPO はビジネス要件です。 リスク評価の実施は、アプリケーションの RTO と RPO の定義に役立ちます。 もう 1 つの一般的なメトリックは、**平均復旧時間** (MTTR) です。MTTR は、障害発生後にアプリケーションの復元にかかる平均時間です。 MTTR は、システムに関する経験的事実です。 MTTR が RTO を超えると、システム障害の結果、定義されている RTO 以内にシステムを復元できなくなるため、許容できないビジネスの中断が発生します。 

### <a name="slas"></a>SLA
Azure の[サービス レベル アグリーメント][sla] (SLA) では、稼働時間と接続性に関する Microsoft のコミットメントが示されています。 特定のサービスの SLA が 99.9% の場合は、そのサービスを 99.9% の確率で使用可能だと予想できることを意味します。

> [!NOTE]
> また、Azure SLA には、SLA が満たされない場合にサービス クレジットを取得するための規定と、各サービスの "可用性" の詳細な定義も含まれています。 SLA のこの側面は、強制ポリシーとして機能します。 
> 
> 

お客様のソリューションの各ワークロードには、独自の目標 SLA を定義することをお勧めします。 SLA があると、アーキテクチャがビジネス要件を満たしているかどうかを評価できるようになります。 たとえば、ワークロードのアップタイム要件が 99.99% で、SLA が 99.9% のサービスに依存している場合、そのサービスをシステムの単一障害点にしてはなりません。 救済策として、サービスで障害が発生した場合に備えてフォールバック パスを用意するか、そのサービスの障害から復旧する他の措置を実行する方法があります。 

次の表は、さまざまな SLA レベルで考えられる累積的なダウンタイムの一覧です。 

| SLA | 週あたりのダウンタイム | 月あたりのダウンタイム | 年あたりのダウンタイム |
| --- | --- | --- | --- |
| 99% |1.68 時間 |7.2 時間 |3.65 日 |
| 99.9% |10.1 分 |43.2 分 |8.76 時間 |
| 99.95% |5 分 |21.6 分 |4.38 時間 |
| 99.99% |1.01 分 |4.32 分 |52.56 分 |
| 99.999% |6 秒 |25.9 秒 |5.26 分 |

当然ながら高可用性が望ましく、他の項目は同じくらいです。 ただし、99.99...% の 9 の桁数を増やすにつれて、そのレベルの可用性を実現するためのコストと複雑さは増大します。 99.99% のアップタイムは、月あたりの合計ダウンタイムの約 5 分に換算されます。 5 つの 9 (99.999%) を実現するために、複雑さとコストが増大する価値はありますか。 その答えはビジネス要件によって変わります。 

SLA を定義する場合の他の考慮事項を次に示します。

* 4 つの 9 (99.99%) を実現するには、手動操作による障害からの復旧には依存できません。 アプリケーションには自己診断機能と自己復旧機能が必要です。 
* 4 つの 9 を超えると、SLA を満たすことができるほど迅速に障害を検出することは困難です。
* SLA を測定する時間枠について考慮します。 この時間枠を短くするほど、許容度は厳格になります。 時間単位または日単位のアップタイムという観点で SLA を定義することはおそらく意味がありません。 

### <a name="composite-slas"></a>複合 SLA
Azure SQL Database に書き込む App Service Web アプリについて検討します。 この記事の執筆時点で、このような Azure サービスには次の SLA があります。

* App Service Web Apps = 99.95%
* SQL Database = 99.99%

![複合 SLA](./images/sla1.png)

このアプリケーションで予想される最大ダウンタイムはどのくらいですか。 いずれかのサービスで障害が発生すると、アプリケーション全体で障害が発生します。 一般的に、各サービスで障害が発生する可能性は独立しているため、このアプリケーションの複合 SLA は 99.95% &times; 99.99% = 99.94% です。 これは個々の SLA よりも低い値です。複数のサービスに依存するアプリケーションは潜在的な障害点が多くなるため、これは驚くことではありません。 

一方、複合 SLA を改善するには、独立したフォールバック パスを作成する方法があります。 たとえば、SQL Database が使用できない場合は、後で処理できるようにトランザクションをキューに格納します。

![複合 SLA](./images/sla2.png)

この設計の場合、データベースに接続できない場合でも、アプリケーションは使用可能な状態です。 ただし、データベースとキューの両方で同時に障害が発生すると、アプリケーションは使用できなくなります。 同時に障害が発生する時間の予測される割合は 0.0001 &times; 0.001 なので、この組み合わせのパスに関する複合 SLA は次のとおりです。  

* データベース OR キュー = 1.0 &minus; (0.0001 &times; 0.001) = 99.99999%

合計複合 SLA は次のとおりです。

* Web アプリ AND (データベース OR キュー) = 99.95% &times; 99.99999% = 99.95% 以下

ただし、この方法にはトレードオフがあります。 アプリケーション ロジックは複雑になり、キューのコストがかかり、データの一貫性に関する問題も考慮が必要になる可能性があります。

**複数リージョン デプロイの SLA**。 もう 1 つの HA 手法は、アプリケーションを複数のリージョンにデプロイし、1 つのリージョンのアプリケーションで障害が発生した場合にフェールオーバーするように Azure Traffic Manager を使用する方法です。 2 リージョンのデプロイの場合、複合 SLA は次のように計算されます。 

*N* は、1 つのリージョンにアプリケーションをデプロイした場合の複合 SLA であるとします。 同時に両方のリージョンのアプリケーションで障害が発生する可能性は (1 &minus; N) &times; (1 &minus; N) と予測されます。 そのため、次のようになります。

* 両リージョンの複合 SLA = 1 &minus; (1 &minus; N)(1 &minus; N) = N + (1 &minus; N)N

最後に、[Traffic Manager の SLA][tm-sla] を考慮する必要があります。 この記事の執筆時点で Traffic Manager の SLA は 99.99% です。

* 複合 SLA = 99.99% &times; (両リージョンの複合 SLA)

また、フェールオーバーは瞬時に完了しないので、フェールオーバー時にはある程度のダウンタイムが発生する可能性があります。 [Traffic Manager エンドポイントの監視とフェールオーバー][tm-failover]に関するページを参照してください。

計算された SLA 値は便利なベースラインですが、可用性のすべてを把握できる訳ではありません。 重要ではないパスで障害が発生した場合、アプリケーションの機能を適切に低下できることがよくあります。 書籍のカタログを表示するアプリケーションを例に説明します。 そのアプリケーションが表紙のサムネイル画像を取得できない場合は、プレース ホルダー イメージを表示することができます。 この場合、画像を取得できないと、アプリケーションの稼働時間は減りませんが、ユーザー エクスペリエンスには影響が出ます。  

## <a name="design-for-resiliency"></a>回復性の設計

設計フェーズで、障害モードの分析 (FMA) を実行することをお勧めします。 FMA の目標は、潜在的な障害点を特定し、アプリケーションがその障害に対して対応する方法を定義することです。

* アプリケーションはこのような障害をどのように検出するでしょうか。
* アプリケーションはこのような障害にどのように対応するでしょうか。
* このような障害のログ記録と監視はどのように行うでしょうか。 

FMA プロセスの詳細と、Azure 固有の推奨事項については、[Azure の回復性ガイダンスの障害モードの分析][fma]に関するページを参照してください。

### <a name="example-of-identifying-failure-modes-and-detection-strategy"></a>障害モードと検出戦略の特定例
**障害点:** 外部 Web サービス/API の呼び出し。

| 障害モード | 検出戦略 |
| --- | --- |
| サービスを使用できない |HTTP 5xx |
| 調整 |HTTP 429 (要求が多すぎます) |
| Authentication |HTTP 401 (権限がありません) |
| 遅い応答 |要求のタイムアウト |


### <a name="redundancy-and-designing-for-failure"></a>障害のための冗長性と設計

障害が及ぼす影響の範囲はさまざまです。 たとえば、ディスク障害などの一部のハードウェア障害が、1 台のホスト コンピューターに影響を及ぼすことがあります。 障害が発生したネットワーク スイッチが、サーバー ラック全体に影響する場合もあります。 データ センターの停電など、データ センター全体を中断させる障害はまれです。 ほとんど発生しませんが、リージョン全体が利用できなくなることもあります。

アプリケーションの回復性を実現するための 1 つの手段として、冗長性があります。 ただし、この冗長性は、アプリケーションの設計時に計画しなければなりません。 また、必要な冗長性のレベルはビジネス要件によって異なります。リージョン障害に備えるために、すべてのアプリケーションにリージョン間での冗長性が必要だとは限りません。 一般的に、冗長性と信頼性を高めると、コストと複雑さが増すというトレードオフがあります。  

Azure には、個別の VM からリージョン全体まで、あらゆる障害レベルでアプリケーションの冗長性を確保するための機能が複数用意されています。 

![](./images/redundancy.svg)

**単一の VM**。 Azure では、単一の VM に対してはアップタイム SLA が用意されています。 2 つ以上の VM を実行すると高い SLA を得られますが、ワークロードによっては単一の VM で十分な信頼性が得られる場合もあります。 運用環境のワークロードの場合は、2 つ以上の VM を使用して冗長性を確保することをお勧めします。 

**可用性セット**。 ディスクやネットワーク スイッチの障害など、ローカライズされたハードウェアの障害から保護するには、可用性セットに 2 つ以上の VM をデプロイします。 可用性セットは、電源とネットワーク スイッチを共有する 2 つ以上の "*障害ドメイン*" で構成されます。 可用性セットの VM は複数の障害ドメインに分散されるため、ある障害ドメインがハードウェア障害の影響を受けた場合は、ネットワーク トラフィックを他の障害ドメインの VM にルーティングできます。 可用性セットの詳細については、「[Azure での Windows 仮想マシンの可用性の管理](/azure/virtual-machines/windows/manage-availability)」を参照してください。

**可用性ゾーン**。  可用性ゾーンとは、Azure リージョンの物理的に独立したゾーンのことです。 可用性ゾーンはそれぞれ異なる電源、ネットワーク、および冷却装置を持ちます。 可用性ゾーンに VM をデプロイすると、データセンター全体の障害からアプリケーションを保護するのに役立ちます。 

**ペアになっているリージョン**。 リージョンの障害からアプリケーションを保護するために、複数のリージョンにアプリケーションをデプロイし、Azure Traffic Manager を使用してインターネット トラフィックを異なるリージョンに分散できます。 各 Azure リージョンは別のリージョンとペアになります。 これが[リージョン ペア](/azure/best-practices-availability-paired-regions)になります。 ブラジル南部を除き、リージョン ペアは、税および法の執行を目的としたデータ常駐要件を満たすために同じ地理的場所に配置されます。

複数リージョンのアプリケーションを設計する場合、リージョン間のネットワーク待機時間は、リージョン内の待機時間よりも長くなることを考慮してください。 たとえば、データベースをレプリケートしてフェールオーバーを有効にするときは、リージョン間の非同期データ レプリケーションではなく、リージョン内で同期データ レプリケーションを使用します。

| &nbsp; | 可用性セット | 可用性ゾーン | ペアのリージョン |
|--------|------------------|-------------------|---------------|
| 障害の範囲 | ラック | データセンター | リージョン |
| 要求のルーティング | Load Balancer | クロスゾーン ロード バランサー | Traffic Manager |
| ネットワーク待ち時間 | 非常に低い | 低 | 中～高 |
| 仮想ネットワーク  | VNet | VNet | リージョン間 VNet ピアリング |

## <a name="implement-resiliency-strategies"></a>回復性戦略を実装する
ここでは、一般的な回復性戦略の調査について説明します。 ほとんどの項目は特定のテクノロジに限定されていません。 このセクションの説明では、各手法の背後にある一般的な考えをまとめ、詳細なドキュメントのリンクを紹介しています。

**一時的な障害を再試行する**。 一時的な障害は、ネットワーク接続の一時的な喪失、データベース接続の欠落、またはサービスがビジー状態のときのタイムアウトが原因で発生します。 多くの場合、一時的な障害は、要求を再試行するだけで解決できます。 多くの Azure サービスでは、クライアント SDK が呼び出し元には認識されない方法の自動再試行を実装しています。詳細については、[サービス固有の再試行ガイダンス][retry-service-specific guidance]に関するページを参照してください。

再試行ごとに、合計待機時間が追加されます。 また、失敗した要求が多すぎると、保留中の要求がキューに蓄積されるため、ボトルネックの原因になる可能性があります。 これらのブロックされた要求が重要なシステム リソース (メモリ、しきい値、データベース接続など) を保持することで、障害が連鎖する可能性があります。 この問題を回避するために、再試行間隔を長くして、失敗した要求の合計数を制限します。 

![](./images/retry.png)

**インスタンス間で負荷分散する**。 スケーラビリティのために、インスタンス数を増やしてクラウド アプリケーションをスケールアウトすることをお勧めします。 この方法では、正常ではないインスタンスをローテーションから除去できるため、回復性も改善されます。 例: 

* 複数の VM をロード バランサーの内側に配置します。 ロード バランサーはトラフィックをすべての VM に分散します。 「[Run load-balanced VMs for scalability and availability][ra-multi-vm]」(スケーラビリティと可用性のために負荷分散された VM を実行する) を参照してください。
* Azure App Service アプリを複数インスタンスにスケールアウトします。 App Service は自動的にインスタンス全体に負荷を分散します。 「[Basic web application][ra-basic-web]」(基本的な Web アプリケーション) を参照してください。
* [Azure Traffic Manager][tm] を使用して、一連のエンドポイント全体でトラフィックを分散します。

**データをレプリケートする**。 データのレプリケートは、データ ストアの一時的ではない障害を処理する場合の一般的な戦略です。 Azure SQL Database、Cosmos DB、Apache Cassandra など、多くのストレージ テクノロジにはレプリケーション機能が組み込まれています。 読み取りパスと書き込みパスの両方を考慮することが重要です。 戦略テクノロジに応じて、複数の書き込み可能なレプリカ、または単一の書き込み可能なレプリカと複数の読み取り専用レプリカを用意する必要があります。 

可用性を最大限にするために、レプリカを複数のリージョンに配置する方法もあります。 ただし、データをレプリケートすると待機時間が長くなります。 通常、複数リージョンをまたがるレプリケートは非同期に行われます。これは最終的な整合性モデルであり、レプリカで障害が発生した場合にはデータ損失が発生する可能性があります。 

**潔く機能を減らす**。 サービスで障害が発生し、フェールオーバー パスがない場合、許容範囲のユーザー エクスペリエンスを提供した状態で、アプリケーションの機能を適切に低下させることができます。 例: 

* 作業項目を後で処理できるようにキューに格納します。 
* 予測値を返します。
* ローカルにキャッシュされたデータを使用します。 
* ユーザーにエラー メッセージを表示します (この選択肢は、アプリケーションが要求への応答を停止するよりも優れています)。

**高負荷のユーザーを調整する**。 少数のユーザーが過剰な負荷を発生させることがあります。 その結果、他のユーザーに影響が及び、アプリケーションの全体的な可用性が低くなる可能性があります。

単一のクライアントが大量の要求を行った場合、一定期間、アプリケーションはクライアントを調整することができます。 この調整期間、(詳細な調整戦略に基づいて) アプリケーションはそのクライアントからの要求の一部またはすべてを拒否します。 調整のしきい値は、ユーザーのサービス レベルによって変わる可能性があります。 

調整は、クライアントが悪意を持って実行している場合に限定されません。サービス クォータを超えたというだけの理由で行われます。 場合によっては、コンシューマーが常にクォータを超えているか、動作が正しくない可能性があります。 この場合は、さらにユーザーをブロックすることがあります。 通常、このブロックを行うには、API キーまたは IP アドレス範囲をブロックします。 詳細については、「[Throttling pattern][throttling-pattern]」(調整パターン) を参照してください。

**サーキット ブレーカーを使用する**。 [サーキット ブレーカー][circuit-breaker-pattern] パターンを使用すると、失敗する可能性のある操作がアプリケーションで繰り返し再試行されるのを回避できます。 サーキット ブレーカーはサービスの呼び出しをラップし、最近発生したエラーの数を追跡します。 エラーの数がしきい値を超えると、サーキット ブレーカーはサービスを呼び出さずに、エラー コードを返し始めます。 これにより、サービスが復旧するための時間が確保されます。 

**負荷平準化を使用してトラフィックの急増をなくす**。 アプリケーションではトラフィックの急増が起きることがあり、これがバックエンドのサービスに大きな負荷をかけることがあります。 バックエンド サービスが要求にすばやく応答できない場合、要求がキューに格納されることや (バックアップ)、サービスによってアプリケーションが調整されることがあります。 この状況を回避するために、バッファーとしてキューを使用することができます。 新しい作業項目がある場合、バックエンド サービスを即時に呼び出すのではなく、アプリケーションで作業項目をキューに格納して非同期に実行します。 キューは、負荷のピークを平準化するバッファーとして機能します。 詳細については、[Queue-Based Load Leveling パターン][load-leveling-pattern]に関するページを参照してください。

**重要なリソースを分離する**。 1 つのサブシステムで複数の障害が連鎖し、アプリケーションの他の部分で障害が発生する可能性があります。 これが起きるのは、障害によってスレッドやソケットなどのリソースが適切なタイミングで解放されない場合で、リソースの消費につながります。 

これを回避するには、システムを分離グループにパーティション分割して、1 つのパーティションでの障害がシステム全体をダウンさせないようにします。 この手法は、バルクヘッド パターンとも呼ばれます。

次に例を示します。

* データベースをパーティション分割し (テナント別など)、各パーティションに Web サーバー インスタンスの別プールを割り当てます。  
* 個別のスレッド プールを使用して、異なるサービスの呼び出しを分離します。 こうすることで、いずれかのサービスで障害が発生した場合の障害の連鎖を回避できます。 例については、Netflix の [Hystrix ライブラリ][hystrix]を参照してください。
* [コンテナー][containers]を使用して、特定のサブシステムに使用できるリソースを制限します。 

![](./images/bulkhead.png)

**補正トランザクションを適用する**。 [補正トランザクション][compensating-transaction-pattern]は、別の完了したトランザクションの影響を元に戻すトランザクションです。 分散システムでは、厳密なトランザクションの一貫性を実現することが非常に困難な可能性があります。 補正トランザクションは、各手順で元に戻すことができる一連の小規模な個別トランザクションを使用することで、一貫性を実現する方法です。

たとえば、旅行を予約する場合、ユーザーは車、ホテルの客室、航空便を予約する可能性があります。 これらいずれかの手順に失敗した場合、操作全体が失敗します。 操作全体に単一の分散トランザクションを使用するのではなく、各手順に補正トランザクションを定義することができます。 たとえば、車の予約を元に戻すには、予約を取り消します。 操作全体を完了するために、コーディネーターは各手順を実行します。 いずれかの手順が失敗すると、コーディネーターは補正トランザクションを適用し、完了したすべての手順を元に戻します。 

## <a name="test-for-resiliency"></a>回復性のテスト
一般的に、アプリケーション機能のテストと同じ方法 (単体テストの実行など) で回復性をテストすることはできません。 代わりに、断続的にのみ発生する障害条件下で、エンドツーエンドのワークロードが動作する方法をテストする必要があります。

テストは反復的なプロセスです。 アプリケーションをテストし、結果を測定し、結果のすべてのエラーを分析して対応し、プロセスを繰り返します。

**フォールト挿入テスト**。 実際のエラーをトリガーするまたはシミュレートすることによって、システムの障害への回復性をテストすることができます。 テストされる一般的な障害シナリオを次に示します。

* VM インスタンスのシャットダウン。
* プロセスのクラッシュ。
* 証明書の期限切れ。
* アクセス キーの変更。
* ドメイン コントローラー上の DNS サービスのシャットダウン。
* RAM やスレッド数など、使用可能なシステム リソースの制限。
* ディスクのマウント解除。
* VM の再デプロイ。

回復時間を測定し、ビジネス要件を満たしていることを確認します。 障害モードの組み合わせもテストします。 障害が連鎖しないこと、分離された方法で処理されることを確認します。

これが、設計フェーズで潜在的な障害点を分析することが重要なもう 1 つの理由です。 この分析結果をテスト計画に反映させることをお勧めします。

**ロード テスト**。 ロード テストは、過負荷状態またはサービスの調整が行われているバックエンド データベースなど、負荷がかかった状態でのみ発生する障害を特定するために重要です。 運用データまたは運用データに可能な限り近い総合的なデータを使用して、ピーク負荷の場合をテストします。 目標は、実際の条件下でアプリケーションがどのように動作するかを確認することです。   

## <a name="deploy-using-reliable-processes"></a>信頼性の高いプロセスを使用してデプロイする
アプリケーションを運用環境にデプロイした場合、更新プログラムがエラーの原因になることがあります。 最悪の場合には、更新プログラムの不良でダウンタイムが発生することがあります。 この問題を回避するために、デプロイ プロセスを予測可能で反復的にする必要があります。 デプロイには、Azure リソースのプロビジョニング、アプリケーション コードのデプロイ、構成設定の適用が含まれます。 1 つの更新プログラムで、この 3 つすべて、または一部が行われることがあります。 

手動のデプロイはエラーが起きやすいという点が重要です。 そのため、必要に応じて実行可能で、何かが失敗した場合に再実行できる自動的なべき等プロセスを用意することをお勧めします。 

* Azure Resource Manager テンプレートを使用して、Azure リソースのプロビジョニングを自動化します。
* [Azure Automation Desired State Configuration][dsc] (DSC) を使用して VM を構成します。
* アプリケーション コードに自動デプロイ プロセスを使用します。

*コードとしてのインフラストラクチャ*と*イミュータブル インフラストラクチャ*という回復性のデプロイに関連する 2 つの概念があります。

* **コードとしてのインフラストラクチャ**は、コードを使用してインフラストラクチャのプロビジョニングと構成を行う手法です。 コードとしてのインフラストラクチャには、宣言型の方法、命令型の方法 (またはその両方の組み合わせ) を使用することができます。 宣言型の方法の例として、Resource Manager テンプレートがあります。 PowerShell スクリプトは、命令型の方法の一例です。
* **イミュータブル インフラストラクチャ**は、運用環境にデプロイした後にインフラストラクチャを変更すべきではないという原則です。 そうしないと、場当たりの変更が適用され、変更内容を正確に把握しづらくなり、システムについて論理的に判断しづらくなる状態に陥る可能性があります。 

もう 1 つの問題は、アプリケーションの更新プログラムを展開する方法です。 ブルーグリーン デプロイまたはカナリア リリースなどの手法を採用して高度に制御された方法で更新プログラムをプッシュし、不適切なデプロイの考えられる影響を最小限に抑えることをお勧めします。

* [ブルーグリーン デプロイ][blue-green]は、ライブ アプリケーションとは別の運用環境に更新プログラムをデプロイする手法です。 デプロイの検証が完了したら、トラフィック ルーティングを更新されたバージョンに切り替えます。 たとえば、Azure App Service Web Apps を使用すると、このステージング スロットが可能になります。
* [カナリア リリース][canary-release]は、ブルーグリーン デプロイと似ています。 すべてのトラフィックを更新されたバージョンに切り替えるのではなく、トラフィックの一部を新しいデプロイにルーティングすることで更新プログラムを少数のユーザーに展開します。 問題が発生した場合は、以前のデプロイに戻します。 問題が発生しない場合は、より多くのトラフィックを新しいバージョンにルーティングします。トラフィックの 100% になるまでこの手順を実行します。

どの方法を採用する場合でも、新しいバージョンが機能しなかった場合に最後の正常なデプロイに戻すことができるようにします。 また、エラーが発生した場合、アプリケーション ログでエラーの原因となったバージョンを特定できるようにします。 

## <a name="monitor-to-detect-failures"></a>監視によって障害を検出する
監視と診断は回復性にとって非常に重要です。 何かが失敗した場合、失敗したことを把握し、障害の原因を分析する必要があります。 

大規模な分散システムの監視は、大きな課題です。 数十単位の VM で実行されるアプリケーションを例にして説明します。&mdash; 各 VM 1 つずつにログを記録し、ログ ファイルを確認し、問題を解決することは実用的ではありません。 さらに、VM インスタンス数はおそらく静的ではありません。 アプリケーションのスケールインとスケールアウトで VM 数は増減し、インスタンスが失敗して再プロビジョニングが必要になることがあります。 さらに、一般的なクラウド アプリケーションは複数のデータ ストア (App Storage、SQL Database、Cosmos DB、Redis Cache) を使用することや、単一のユーザー アクションが複数のサブシステムに広がることがあります。 

監視と診断プロセスは、いくつかの異なる段階があるパイプラインと考えることができます。

![複合 SLA](./images/monitoring.png)

* **インストルメンテーション**。 監視と診断の生データは、多様なソースに由来します。たとえば、アプリケーション ログ、Web サーバー ログ、OS のパフォーマンス カウンター、データベース ログ、Azure プラットフォームに組み込まれている診断などです。 Most Azure サービスには、問題の原因特定に使用できる診断機能があります。
* **収集と保存**。 生インストルメンテーション データは多様な場所に多様な形式で保持される可能性があります (たとえば、アプリケーション トレース ログ、IIS ログ、パフォーマンス カウンターなど)。 このようなさまざまなソースは収集され、統合されて、信頼性の高いストレージに格納されます。
* **分析と診断**。 データの統合後は、分析して問題を解決し、アプリケーションの正常性の全体を把握することができます。
* **視覚化とアラート**。 この段階では、オペレーターが問題や傾向にすばやく気づくことができる方法で利用統計情報が表示されます。 たとえば、ダッシュボードや電子メールのアラートがあります。  

監視は障害の検出と同じではありません。 たとえば、アプリケーションが一時的なエラーを検出して再試行し、ダウンタイムが発生しないことがあります。 それでも再試行操作はログに記録されるので、エラー率を監視して、アプリケーション正常性の全体像を把握することができます。 

アプリケーション ログは、診断データの重要なソースです。 アプリケーションのログ記録には、次のようなベスト プラクティスがあります。

* 運用環境でログを記録します。 そうしないと、最も必要な場合に洞察できません。
* サービスの境界でイベントのログを記録します。 フローがサービス境界をまたがる関連付け ID を含めます。 トランザクション フローが複数のサービスを経由し、いずれかが失敗した場合、関連付け ID でトランザクションが失敗した理由を特定できます。
* セマンティック ログ (構造化ログとも呼ばれます) を使用します。 構造化されていないログの場合、ログ データの使用と分析の自動化が困難です。こうした自動化はクラウド規模で必要になります。
* 非同期のログ記録を使用します。 そうしないと、ログ アプリケーションによって、ログ記録イベントの書き込みを待機中に要求がブロックされ、要求がバックアップされ、アプリケーションで障害が発生する原因になる可能性があります。
* アプリケーションのログ記録は、監査と同じではありません。 監査は、コンプライアンスまたは法規制上の理由で行われることがあります。 そのため、監査レコードは完全である必要があります。トランザクションの処理中に削除が発生することは許容されません。 アプリケーションに監査が必要な場合、診断ログとは別に維持する必要があります。 

監視と診断の詳細については、「[Monitoring and diagnostics guidance][monitoring-guidance]」(監査と診断のガイダンス) を参照してください。

## <a name="respond-to-failures"></a>障害に対応する
これまでのセクションでは、高可用性に重要な自動的な復旧戦略を中心に説明してきましたが、 手動操作が必要な場合もあります。

* **[Alerts]** (アラート)。 アプリケーションで、積極的な介入が必要な可能性がある警告の兆候を監視します。 たとえば、SQL Database または Cosmos DB がアプリケーションを常に調整している場合は、データベース容量の増加や、クエリの最適化が必要な可能性があります。 この例では、アプリケーションが調整エラーを透過的に処理している場合でも、利用統計情報でアラートが報告されるので、対応することができます。  
* **手動フェールオーバー**。 一部のシステムは自動的にフェールオーバーせず、手動フェールオーバーが必要です。 
* **運用準備状況のテスト**。 アプリケーションがセカンダリ リージョンにフェールオーバーした場合、運用準備状況のテストを実行してから、プライマリ リージョンにフェールバックすることをお勧めします。 テストでは、プライマリ リージョンが正常で、トラフィックを再び受け取る準備が整っていることを確認します。
* **データ整合性チェック**。 データ ストアで障害が発生した場合、特にデータがレプリケートされた場合には、ストアを再び使用可能な状態にするときにデータに整合性がある必要があります。 
* **バックアップからの復元**。 たとえば、SQL Database で地域的な停電が発生した場合、最新のバックアップからデータベースの geo リストアを実行する方法があります。

ディザスター リカバリー計画を文書化し、テストします。 アプリケーションの障害によるビジネスの影響を評価します。 できるだけ多くのプロセスを自動化し、手動の手順を文書化します。たとえば、バックアップからの手動のフェールオーバーやデータの復元などです。 ディザスター リカバリー プロセスは定期的にテストし、計画の検証と改善を行います。 

## <a name="summary"></a>まとめ
この記事では、クラウド固有の一部の課題を特に取り上げながら、包括的な観点から回復性について説明しました。 たとえば、クラウド コンピューティングの分散の特性、汎用的なハードウェアの使用、一時的なネットワーク障害の存在などです。

この記事の主な点を次に示します。

* 回復性によって可用性が高くなり、障害からの平均復旧時間が短くなります。 
* クラウドで回復性を実現するには、従来のオンプレミス ソリューションのさまざまな手法が必要です。 
* 偶発的に回復性が実現することはありません。 初期段階から設計し、構築する必要があります。
* 回復性は、計画、コーディングから運用まで、アプリケーションのライフサイクルのあらゆる部分に関係します。
* テストと監視をお勧めします。


<!-- links -->

[blue-green]: https://martinfowler.com/bliki/BlueGreenDeployment.html
[canary-release]: https://martinfowler.com/bliki/CanaryRelease.html
[circuit-breaker-pattern]: https://msdn.microsoft.com/library/dn589784.aspx
[compensating-transaction-pattern]: https://msdn.microsoft.com/library/dn589804.aspx
[containers]: https://en.wikipedia.org/wiki/Operating-system-level_virtualization
[dsc]: /azure/automation/automation-dsc-overview
[contingency-planning-guide]: https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-34r1.pdf
[fma]: failure-mode-analysis.md
[hystrix]: https://medium.com/netflix-techblog/introducing-hystrix-for-resilience-engineering-13531c1ab362
[jmeter]: https://jmeter.apache.org/
[load-leveling-pattern]: ../patterns/queue-based-load-leveling.md
[monitoring-guidance]: ../best-practices/monitoring.md
[ra-basic-web]: ../reference-architectures/app-service-web-app/basic-web-app.md
[ra-multi-vm]: ../reference-architectures/virtual-machines-windows/multi-vm.md
[checklist]: ../checklist/resiliency.md
[retry-pattern]: ../patterns/retry.md
[retry-service-specific guidance]: ../best-practices/retry-service-specific.md
[sla]: https://azure.microsoft.com/support/legal/sla/
[throttling-pattern]: ../patterns/throttling.md
[tm]: https://azure.microsoft.com/services/traffic-manager/
[tm-failover]: /azure/traffic-manager/traffic-manager-monitoring
[tm-sla]: https://azure.microsoft.com/support/legal/sla/traffic-manager
