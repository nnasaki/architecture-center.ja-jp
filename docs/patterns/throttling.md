---
title: "調整"
description: "アプリケーションのインスタンス、個々のテナント、またはサービス全体によって使用されるリソースの使用量を制御します。"
keywords: "設計パターン"
author: dragon119
ms.date: 06/23/2017
pnp.series.title: Cloud Design Patterns
pnp.pattern.categories:
- availability
- performance-scalability
ms.openlocfilehash: 29156fc72f40a952dd53adcb20ffa7c3d0af79b4
ms.sourcegitcommit: b0482d49aab0526be386837702e7724c61232c60
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 11/14/2017
---
# <a name="throttling-pattern"></a><span data-ttu-id="18920-104">調整パターン</span><span class="sxs-lookup"><span data-stu-id="18920-104">Throttling pattern</span></span>

[!INCLUDE [header](../_includes/header.md)]

<span data-ttu-id="18920-105">アプリケーションのインスタンス、個々のテナント、またはサービス全体によって使用されるリソースの使用量を制御します。</span><span class="sxs-lookup"><span data-stu-id="18920-105">Control the consumption of resources used by an instance of an application, an individual tenant, or an entire service.</span></span> <span data-ttu-id="18920-106">これによりシステムは、需要の増加に伴ってリソースに過度な負荷がかけられた場合でも正常な動作を継続し、サービス レベル アグリーメントに準拠することができます。</span><span class="sxs-lookup"><span data-stu-id="18920-106">This can allow the system to continue to function and meet service level agreements, even when an increase in demand places an extreme load on resources.</span></span>

## <a name="context-and-problem"></a><span data-ttu-id="18920-107">コンテキストと問題</span><span class="sxs-lookup"><span data-stu-id="18920-107">Context and problem</span></span>

<span data-ttu-id="18920-108">通常、クラウド アプリケーションにかかる負荷は、アクティブ ユーザー数、またはアクティブ ユーザーが実行しているアクティビティの種類に応じて、時間の経過とともに変化します。</span><span class="sxs-lookup"><span data-stu-id="18920-108">The load on a cloud application typically varies over time based on the number of active users or the types of activities they are performing.</span></span> <span data-ttu-id="18920-109">たとえば、より多くのユーザーが業務時間中にアクティブになる可能性がある、あるいはシステム上で計算コストの高い分析を毎月月末に実行しなければならない可能性がある、などの場合です。</span><span class="sxs-lookup"><span data-stu-id="18920-109">For example, more users are likely to be active during business hours, or the system might be required to perform computationally expensive analytics at the end of each month.</span></span> <span data-ttu-id="18920-110">突然、予期せずアクティビティが急増することもあります。</span><span class="sxs-lookup"><span data-stu-id="18920-110">There might also be sudden and unanticipated bursts in activity.</span></span> <span data-ttu-id="18920-111">システムの処理要件が使用可能なリソースの容量を超えると、パフォーマンスが低下することや、障害が発生することさえあり得ます。</span><span class="sxs-lookup"><span data-stu-id="18920-111">If the processing requirements of the system exceed the capacity of the resources that are available, it'll suffer from poor performance and can even fail.</span></span> <span data-ttu-id="18920-112">そのシステムにおいて、契約したサービス レベルに準拠する必要がある場合は、そのような障害は許容できません。</span><span class="sxs-lookup"><span data-stu-id="18920-112">If the system has to meet an agreed level of service, such failure could be unacceptable.</span></span>

<span data-ttu-id="18920-113">クラウドには、アプリケーションのビジネス目標に応じて、変化する負荷に対応するための戦略が数多くあります。</span><span class="sxs-lookup"><span data-stu-id="18920-113">There're many strategies available for handling varying load in the cloud, depending on the business goals for the application.</span></span> <span data-ttu-id="18920-114">戦略の 1 つは、自動スケールを使用して、プロビジョニング済みのリソースを特定の時点のユーザーのニーズに合わせることです。</span><span class="sxs-lookup"><span data-stu-id="18920-114">One strategy is to use autoscaling to match the provisioned resources to the user needs at any given time.</span></span> <span data-ttu-id="18920-115">これにより、ランニング コストを最適化しつつ、一貫してユーザーの要求に応えていくことが可能になります。</span><span class="sxs-lookup"><span data-stu-id="18920-115">This has the potential to consistently meet user demand, while optimizing running costs.</span></span> <span data-ttu-id="18920-116">ただし、自動スケールはその他のリソースのプロビジョニングをトリガーできますが、このプロビジョニングは即時ではありません。</span><span class="sxs-lookup"><span data-stu-id="18920-116">However, while autoscaling can trigger the provisioning of additional resources, this provisioning isn't immediate.</span></span> <span data-ttu-id="18920-117">要求が急激に増加した場合、リソースが不足する期間が発生する可能性があります。</span><span class="sxs-lookup"><span data-stu-id="18920-117">If demand grows quickly, there can be a window of time where there's a resource deficit.</span></span>

## <a name="solution"></a><span data-ttu-id="18920-118">解決策</span><span class="sxs-lookup"><span data-stu-id="18920-118">Solution</span></span>

<span data-ttu-id="18920-119">自動スケールに代わる戦略として、アプリケーションに上限までのリソースの使用を許可し、この上限に達したときに調整を行う方法があります。</span><span class="sxs-lookup"><span data-stu-id="18920-119">An alternative strategy to autoscaling is to allow applications to use resources only up to a limit, and then throttle them when this limit is reached.</span></span> <span data-ttu-id="18920-120">使用量がしきい値を超えたときに 1 人または複数のユーザーの要求を制限できるよう、システムがリソースの使用状況を監視します。</span><span class="sxs-lookup"><span data-stu-id="18920-120">The system should monitor how it's using resources so that, when usage exceeds the threshold, it can throttle requests from one or more users.</span></span> <span data-ttu-id="18920-121">これにより、システムは正常な動作を継続し、適用されるすべてのサービス レベル アグリーメント (SLA) に準拠することができます。</span><span class="sxs-lookup"><span data-stu-id="18920-121">This will enable the system to continue functioning and meet any service level agreements (SLAs) that are in place.</span></span> <span data-ttu-id="18920-122">リソース使用状況の監視の詳細については、「[Instrumentation and Telemetry Guidance (インストルメンテーションとテレメトリに関するガイダンス)](https://msdn.microsoft.com/library/dn589775.aspx)」をご覧ください。</span><span class="sxs-lookup"><span data-stu-id="18920-122">For more information on monitoring resource usage, see the [Instrumentation and Telemetry Guidance](https://msdn.microsoft.com/library/dn589775.aspx).</span></span>

<span data-ttu-id="18920-123">システムには次のような複数の調整戦略を実装できます。</span><span class="sxs-lookup"><span data-stu-id="18920-123">The system could implement several throttling strategies, including:</span></span>

- <span data-ttu-id="18920-124">システムの API に指定した期間に 1 秒あたり n 回よりも多くアクセスした個人ユーザーからの要求を拒絶する。</span><span class="sxs-lookup"><span data-stu-id="18920-124">Rejecting requests from an individual user who's already accessed system APIs more than n times per second over a given period of time.</span></span> <span data-ttu-id="18920-125">これには、システムで各テナントのリソース使用量、またはアプリケーションを実行するユーザー数を測定することが必要です。</span><span class="sxs-lookup"><span data-stu-id="18920-125">This requires the system to meter the use of resources for each tenant or user running an application.</span></span> <span data-ttu-id="18920-126">詳細については、「[Service Metering Guidance (サービスの測定に関するガイダンス)](https://msdn.microsoft.com/library/dn589796.aspx)」をご覧ください。</span><span class="sxs-lookup"><span data-stu-id="18920-126">For more information, see the [Service Metering Guidance](https://msdn.microsoft.com/library/dn589796.aspx).</span></span>

- <span data-ttu-id="18920-127">重要なサービスを十分なリソースで滞りなく実行できるよう、選択した重要でないサービスの機能を無効にする、または品質を下げる。</span><span class="sxs-lookup"><span data-stu-id="18920-127">Disabling or degrading the functionality of selected nonessential services so that essential services can run unimpeded with sufficient resources.</span></span> <span data-ttu-id="18920-128">たとえば、アプリケーションでビデオ出力をストリーミングしている場合、低解像度に切り替えることができます。</span><span class="sxs-lookup"><span data-stu-id="18920-128">For example, if the application is streaming video output, it could switch to a lower resolution.</span></span>

- <span data-ttu-id="18920-129">負荷平準化によってアクティビティのボリュームを平準化する (このアプローチの詳細については、「[Queue-based Load Leveling pattern (キュー ベースの負荷平準化パターン)](queue-based-load-leveling.md)」をご覧ください)。</span><span class="sxs-lookup"><span data-stu-id="18920-129">Using load leveling to smooth the volume of activity (this approach is covered in more detail by the [Queue-based Load Leveling pattern](queue-based-load-leveling.md)).</span></span> <span data-ttu-id="18920-130">マルチテナント環境では、このアプローチですべてのテナントのパフォーマンスを縮小します。</span><span class="sxs-lookup"><span data-stu-id="18920-130">In a multi-tenant environment, this approach will reduce the performance for every tenant.</span></span> <span data-ttu-id="18920-131">SLA の異なるテナントが混在している状態がシステムによってサポートされている必要があれば、高価値のテナントの作業を即時に実行することがあります。</span><span class="sxs-lookup"><span data-stu-id="18920-131">If the system must support a mix of tenants with different SLAs, the work for high-value tenants might be performed immediately.</span></span> <span data-ttu-id="18920-132">その他のテナントの要求処理は差し止められ、バックログが収まった時点で処理されます。</span><span class="sxs-lookup"><span data-stu-id="18920-132">Requests for other tenants can be held back, and handled when the backlog has eased.</span></span> <span data-ttu-id="18920-133">このアプローチを導入しやすくするために、[優先順位キュー パターン][]を使用できます。</span><span class="sxs-lookup"><span data-stu-id="18920-133">The [Priority Queue pattern][] could be used to help implement this approach.</span></span>

- <span data-ttu-id="18920-134">優先度の低いアプリケーションまたはテナントの代わりとして実行される操作を延期する。</span><span class="sxs-lookup"><span data-stu-id="18920-134">Deferring operations being performed on behalf of lower priority applications or tenants.</span></span> <span data-ttu-id="18920-135">システムがビジー状態であることと、その操作をあとで再試行すべきであることをテナントに知らせるために例外を生成して、操作を一時停止または制限できます。</span><span class="sxs-lookup"><span data-stu-id="18920-135">These operations can be suspended or limited, with an exception generated to inform the tenant that the system is busy and that the operation should be retried later.</span></span>

<span data-ttu-id="18920-136">次の図では、3 つの機能を使用しているアプリケーションの、リソース使用量 (メモリ、CPU、帯域幅などの要素の組み合わせ) の領域グラフを時系列で示しています。</span><span class="sxs-lookup"><span data-stu-id="18920-136">The figure shows an area graph for resource use (a combination of memory, CPU, bandwidth, and other factors) against time for applications that are making use of three features.</span></span> <span data-ttu-id="18920-137">1 つの機能は、特定のタスク セットを実行するコンポーネント、複雑な計算を実行する 1 つのコード、またはメモリ内キャッシュなどのサービスを提供する要素など、1 つの機能領域を示しています。</span><span class="sxs-lookup"><span data-stu-id="18920-137">A feature is an area of functionality, such as a component that performs a specific set of tasks, a piece of code that performs a complex calculation, or an element that provides a service such as an in-memory cache.</span></span> <span data-ttu-id="18920-138">これらの機能に A、B、C という名前が付けられています。</span><span class="sxs-lookup"><span data-stu-id="18920-138">These features are labeled A, B, and C.</span></span>

![図 1 - 3 人のユーザーの代わりとして実行されているアプリケーションのリソース使用量を時系列で示すグラフ](./_images/throttling-resource-utilization.png)


> <span data-ttu-id="18920-140">機能の線のすぐ下の領域は、その機能が呼び出されたときにアプリケーションが使用するリソースを示しています。</span><span class="sxs-lookup"><span data-stu-id="18920-140">The area immediately below the line for a feature indicates the resources that are used by applications when they invoke this feature.</span></span> <span data-ttu-id="18920-141">たとえば、機能 A の線の下の領域は、機能 A を使用しているアプリケーションが使用するリソースを示しており、機能 A と機能 B の線の間にある領域は、機能 B を呼び出すアプリケーションが使用するリソースを示しています。各機能の領域の合計が、そのシステムが使用するリソースの総計です。</span><span class="sxs-lookup"><span data-stu-id="18920-141">For example, the area below the line for Feature A shows the resources used by applications that are making use of Feature A, and the area between the lines for Feature A and Feature B indicates the resources used by applications invoking Feature B. Aggregating the areas for each feature shows the total resource use of the system.</span></span>

<span data-ttu-id="18920-142">上の図では、操作を延期することによる効果を示しています。</span><span class="sxs-lookup"><span data-stu-id="18920-142">The previous figure illustrates the effects of deferring operations.</span></span> <span data-ttu-id="18920-143">時間 T1 の直前で、これらの機能を使用するすべてのアプリケーションに割り当てられたリソースの総計がしきい値 (リソース使用上限) に達しています。</span><span class="sxs-lookup"><span data-stu-id="18920-143">Just prior to time T1, the total resources allocated to all applications using these features reach a threshold (the limit of resource use).</span></span> <span data-ttu-id="18920-144">この時点で、アプリケーションが使用可能なリソースを使い果たす危険性が生じます。</span><span class="sxs-lookup"><span data-stu-id="18920-144">At this point, the applications are in danger of exhausting the resources available.</span></span> <span data-ttu-id="18920-145">このシステムでは、機能 B は 機能 A や機能 C よりも重要ではないため一時的に無効化され、機能 B が使用していたリソースが解放されます。</span><span class="sxs-lookup"><span data-stu-id="18920-145">In this system, Feature B is less critical than Feature A or Feature C, so it's temporarily disabled and the resources that it was using are released.</span></span> <span data-ttu-id="18920-146">時間 T1 と T2 の間では、機能 A と機能 C を使用しているアプリケーションが引き続き通常どおり実行されています。</span><span class="sxs-lookup"><span data-stu-id="18920-146">Between times T1 and T2, the applications using Feature A and Feature C continue running as normal.</span></span> <span data-ttu-id="18920-147">最終的には、これら 2 つの機能のリソース使用量が機能 B をもう一度有効にするための十分な容量が確保される時点 (時間 T2) まで減少します。</span><span class="sxs-lookup"><span data-stu-id="18920-147">Eventually, the resource use of these two features diminishes to the point when, at time T2, there is sufficient capacity to enable Feature B again.</span></span>

<span data-ttu-id="18920-148">自動スケールと調整のアプローチは、アプリケーションの応答性を保ち SLA に準拠させるために、組み合わせることも可能です。</span><span class="sxs-lookup"><span data-stu-id="18920-148">The autoscaling and throttling approaches can also be combined to help keep the applications responsive and within SLAs.</span></span> <span data-ttu-id="18920-149">需要が高いままになると予想される場合は、システムがスケールアウトしている間に、調整によって一時的な解決がなされます。この時点では、システムのすべての機能を復元できます。</span><span class="sxs-lookup"><span data-stu-id="18920-149">If the demand is expected to remain high, throttling provides a temporary solution while the system scales out. At this point, the full functionality of the system can be restored.</span></span>

<span data-ttu-id="18920-150">次の図は、システムで実行しているすべてのアプリケーションが使用する全体的なリソース使用量の領域グラフを時系列で示しており、調整と自動スケールをどのように組み合わせることができるかを説明しています。</span><span class="sxs-lookup"><span data-stu-id="18920-150">The next figure shows an area graph of the overall resource use by all applications running in a system against time, and illustrates how throttling can be combined with autoscaling.</span></span>

![図 2 - 調整と自動スケールを組み合わせることによる効果を示すグラフ](./_images/throttling-autoscaling.png)


<span data-ttu-id="18920-152">時間 T1 の時点で、リソース使用のソフト制限を指定するしきい値に達しています。</span><span class="sxs-lookup"><span data-stu-id="18920-152">At time T1, the threshold specifying the soft limit of resource use is reached.</span></span> <span data-ttu-id="18920-153">この時点で、システムはスケールアウトを開始できます。ただし、新しいリソースが使用できる状態になるのが間に合わなければ、既存のリソースを使い果たすか、システムに障害が発生する可能性があります。</span><span class="sxs-lookup"><span data-stu-id="18920-153">At this point, the system can start to scale out. However, if the new resources don't become available quickly enough, then the existing resources might be exhausted and the system could fail.</span></span> <span data-ttu-id="18920-154">この問題の発生を防ぐために、前述のとおりにシステムは一時的に調整されます。</span><span class="sxs-lookup"><span data-stu-id="18920-154">To prevent this from occurring, the system is temporarily throttled, as described earlier.</span></span> <span data-ttu-id="18920-155">自動スケールが完了してその他のリソースが使用可能になると、調整が緩和されます。</span><span class="sxs-lookup"><span data-stu-id="18920-155">When autoscaling has completed and the additional resources are available, throttling can be relaxed.</span></span>

## <a name="issues-and-considerations"></a><span data-ttu-id="18920-156">問題と注意事項</span><span class="sxs-lookup"><span data-stu-id="18920-156">Issues and considerations</span></span>

<span data-ttu-id="18920-157">このパターンの実装方法を決めるときには、以下の点に注意してください。</span><span class="sxs-lookup"><span data-stu-id="18920-157">You should consider the following points when deciding how to implement this pattern:</span></span>

- <span data-ttu-id="18920-158">アプリケーションの調整 (およびその使用の戦略) は、システム設計の全体に影響を与えるアーキテクチャの決定です。</span><span class="sxs-lookup"><span data-stu-id="18920-158">Throttling an application, and the strategy to use, is an architectural decision that impacts the entire design of a system.</span></span> <span data-ttu-id="18920-159">システムが実装されてしまうとあとから追加することが難しいため、アプリケーション設計の早い段階で調整について検討する必要があります。</span><span class="sxs-lookup"><span data-stu-id="18920-159">Throttling should be considered early in the application design process because it isn't easy to add once a system has been implemented.</span></span>

- <span data-ttu-id="18920-160">調整は迅速に実行する必要があります。</span><span class="sxs-lookup"><span data-stu-id="18920-160">Throttling must be performed quickly.</span></span> <span data-ttu-id="18920-161">システムは、アクティビティの増加を検出して、それに対応できる必要があります。</span><span class="sxs-lookup"><span data-stu-id="18920-161">The system must be capable of detecting an increase in activity and react accordingly.</span></span> <span data-ttu-id="18920-162">システムは、負荷が緩和されたらすぐ元の状態に戻ることができる必要もあります。</span><span class="sxs-lookup"><span data-stu-id="18920-162">The system must also be able to revert to its original state quickly after the load has eased.</span></span> <span data-ttu-id="18920-163">そのためには、適切なパフォーマンス データが継続的にキャプチャされ、監視されていることが必要です。</span><span class="sxs-lookup"><span data-stu-id="18920-163">This requires that the appropriate performance data is continually captured and monitored.</span></span>

- <span data-ttu-id="18920-164">サービスが一時的にユーザーの要求を拒否する必要がある場合、特定のエラー コードを返して、操作の実行が拒否されたのは調整のためだということを、クライアント アプリケーションが理解できるようにする必要があります。</span><span class="sxs-lookup"><span data-stu-id="18920-164">If a service needs to temporarily deny a user request, it should return a specific error code so the client application understands that the reason for the refusal to perform an operation is due to throttling.</span></span> <span data-ttu-id="18920-165">クライアント アプリケーションは、要求を再試行するまでの間、待機することができます。</span><span class="sxs-lookup"><span data-stu-id="18920-165">The client application can wait for a period before retrying the request.</span></span>

- <span data-ttu-id="18920-166">システムが自動スケールしている間、一時的な措置として調整を使用することができます。</span><span class="sxs-lookup"><span data-stu-id="18920-166">Throttling can be used as a temporary measure while a system autoscales.</span></span> <span data-ttu-id="18920-167">アクティビティの急増が突然で長くは続かないと予想される場合は、スケールにはかなりのランニング コストがかかるため、スケールするよりも単純に調整する方が良い場合があります。</span><span class="sxs-lookup"><span data-stu-id="18920-167">In some cases it's better to simply throttle, rather than to scale, if a burst in activity is sudden and isn't expected to be long lived because scaling can add considerably to running costs.</span></span>

- <span data-ttu-id="18920-168">システムが自動スケールしている間に一時的な措置として調整を使用する場合、また、リソースの需要が急激に増加する場合は、調整モードで運用されていたとしても、システムの正常な動作を継続できない場合があります。</span><span class="sxs-lookup"><span data-stu-id="18920-168">If throttling is being used as a temporary measure while a system autoscales, and if resource demands grow very quickly, the system might not be able to continue functioning&mdash;even when operating in a throttled mode.</span></span> <span data-ttu-id="18920-169">これを許容できない場合は、より大きな予備容量を維持することと、よりアグレッシブな自動スケールを構成することを検討してください。</span><span class="sxs-lookup"><span data-stu-id="18920-169">If this isn't acceptable, consider maintaining larger capacity reserves and configuring more aggressive autoscaling.</span></span>

## <a name="when-to-use-this-pattern"></a><span data-ttu-id="18920-170">このパターンを使用する状況</span><span class="sxs-lookup"><span data-stu-id="18920-170">When to use this pattern</span></span>

<span data-ttu-id="18920-171">このパターンは次の目的で使用します。</span><span class="sxs-lookup"><span data-stu-id="18920-171">Use this pattern:</span></span>

- <span data-ttu-id="18920-172">システムが継続してサービス レベル アグリーメントに準拠するようにする。</span><span class="sxs-lookup"><span data-stu-id="18920-172">To ensure that a system continues to meet service level agreements.</span></span>

- <span data-ttu-id="18920-173">単一のテナントが、アプリケーションによって提供されるリソースを占有することを防止する。</span><span class="sxs-lookup"><span data-stu-id="18920-173">To prevent a single tenant from monopolizing the resources provided by an application.</span></span>

- <span data-ttu-id="18920-174">アクティビティの急増に対応する。</span><span class="sxs-lookup"><span data-stu-id="18920-174">To handle bursts in activity.</span></span>

- <span data-ttu-id="18920-175">システムの正常な動作を継続するのに必要な、リソースの最大レベルを制限することで、システムのコスト最適化に役立てる。</span><span class="sxs-lookup"><span data-stu-id="18920-175">To help cost-optimize a system by limiting the maximum resource levels needed to keep it functioning.</span></span>

## <a name="example"></a><span data-ttu-id="18920-176">例</span><span class="sxs-lookup"><span data-stu-id="18920-176">Example</span></span>

<span data-ttu-id="18920-177">最後の図は、マルチテナント システムで調整を実装する方法を示しています。</span><span class="sxs-lookup"><span data-stu-id="18920-177">The final figure illustrates how throttling can be implemented in a multi-tenant system.</span></span> <span data-ttu-id="18920-178">各テナント組織のユーザーが、クラウドでホストされているアプリケーションにアクセスし、アンケートに記入して送信します。</span><span class="sxs-lookup"><span data-stu-id="18920-178">Users from each of the tenant organizations access a cloud-hosted application where they fill out and submit surveys.</span></span> <span data-ttu-id="18920-179">このアプリケーションには、これらのユーザーがアプリケーションに要求を送信する頻度を監視するインストルメンテーションが含まれています。</span><span class="sxs-lookup"><span data-stu-id="18920-179">The application contains instrumentation that monitors the rate at which these users are submitting requests to the application.</span></span>

<span data-ttu-id="18920-180">1 つのテナントのユーザーが、他のすべてのユーザーに対するアプリケーションの応答性と可用性に影響を与えることを防止するため、ユーザーが 1 つのテナントから送信できる 1 秒あたりの要求数を制限します。</span><span class="sxs-lookup"><span data-stu-id="18920-180">In order to prevent the users from one tenant affecting the responsiveness and availability of the application for all other users, a limit is applied to the number of requests per second the users from any one tenant can submit.</span></span> <span data-ttu-id="18920-181">このアプリケーションは、制限を超える要求をブロックします。</span><span class="sxs-lookup"><span data-stu-id="18920-181">The application blocks requests that exceed this limit.</span></span>

![図 3 - マルチテナント アプリケーションでの調整の実装](./_images/throttling-multi-tenant.png)


## <a name="related-patterns-and-guidance"></a><span data-ttu-id="18920-183">関連のあるパターンとガイダンス</span><span class="sxs-lookup"><span data-stu-id="18920-183">Related patterns and guidance</span></span>

<span data-ttu-id="18920-184">このパターンを実装する場合は、次のパターンとガイダンスも関連している可能性があります。</span><span class="sxs-lookup"><span data-stu-id="18920-184">The following patterns and guidance may also be relevant when implementing this pattern:</span></span>
- <span data-ttu-id="18920-185">[Instrumentation and Telemetry Guidance (インストルメンテーションと製品利用統計情報のガイダンス)](https://msdn.microsoft.com/library/dn589775.aspx)。</span><span class="sxs-lookup"><span data-stu-id="18920-185">[Instrumentation and Telemetry Guidance](https://msdn.microsoft.com/library/dn589775.aspx).</span></span> <span data-ttu-id="18920-186">調整は、サービスの使用頻度に関する情報をどれだけ収集できるかにかかっています。</span><span class="sxs-lookup"><span data-stu-id="18920-186">Throttling depends on gathering information about how heavily a service is being used.</span></span> <span data-ttu-id="18920-187">カスタムの監視情報を生成してキャプチャする方法を説明します。</span><span class="sxs-lookup"><span data-stu-id="18920-187">Describes how to generate and capture custom monitoring information.</span></span>
- <span data-ttu-id="18920-188">[Service Metering Guidance (サービスの測定に関するガイダンス)](https://msdn.microsoft.com/library/dn589796.aspx)。</span><span class="sxs-lookup"><span data-stu-id="18920-188">[Service Metering Guidance](https://msdn.microsoft.com/library/dn589796.aspx).</span></span> <span data-ttu-id="18920-189">サービスの使用状況を理解するために、サービスの使用量を測定する方法を説明します。</span><span class="sxs-lookup"><span data-stu-id="18920-189">Describes how to meter the use of services in order to gain an understanding of how they are used.</span></span> <span data-ttu-id="18920-190">この情報は、サービスを調整する方法を決めるのに役立ちます。</span><span class="sxs-lookup"><span data-stu-id="18920-190">This information can be useful in determining how to throttle a service.</span></span>
- <span data-ttu-id="18920-191">[自動スケール ガイダンス](https://msdn.microsoft.com/library/dn589774.aspx)。</span><span class="sxs-lookup"><span data-stu-id="18920-191">[Autoscaling Guidance](https://msdn.microsoft.com/library/dn589774.aspx).</span></span> <span data-ttu-id="18920-192">調整は、システムが自動スケールしている間に、またはシステムが自動スケールする必要をなくすために、暫定的な措置として使用されます。</span><span class="sxs-lookup"><span data-stu-id="18920-192">Throttling can be used as an interim measure while a system autoscales, or to remove the need for a system to autoscale.</span></span> <span data-ttu-id="18920-193">自動スケール戦略に関する情報が含まれています。</span><span class="sxs-lookup"><span data-stu-id="18920-193">Contains information on autoscaling strategies.</span></span>
- <span data-ttu-id="18920-194">[キュー ベースの負荷平準化パターン](queue-based-load-leveling.md)。</span><span class="sxs-lookup"><span data-stu-id="18920-194">[Queue-based Load Leveling pattern](queue-based-load-leveling.md).</span></span> <span data-ttu-id="18920-195">キュー ベースの負荷平準化は、調整を実装するためによく使用されるメカニズムです。</span><span class="sxs-lookup"><span data-stu-id="18920-195">Queue-based load leveling is a commonly used mechanism for implementing throttling.</span></span> <span data-ttu-id="18920-196">キューは、アプリケーションから送信された要求がサービスまで届く頻度を平準化するのに役立つバッファーとして機能します。</span><span class="sxs-lookup"><span data-stu-id="18920-196">A queue can act as a buffer that helps to even out the rate at which requests sent by an application are delivered to a service.</span></span>
- <span data-ttu-id="18920-197">[優先順位キュー パターン][]。</span><span class="sxs-lookup"><span data-stu-id="18920-197">[Priority Queue pattern][].</span></span> <span data-ttu-id="18920-198">システムは、調整戦略の一部として優先順位キューを使用して、比較的重要ではないアプリケーションのパフォーマンスを低下させつつ、非常に重要な、または比較的高価値のアプリケーションのパフォーマンスを維持することができます。</span><span class="sxs-lookup"><span data-stu-id="18920-198">A system can use priority queuing as part of its throttling strategy to maintain performance for critical or higher value applications, while reducing the performance of less important applications.</span></span>

[優先順位キュー パターン]: priority-queue.md