---
title: SQL Server を使用した n 層アプリケーション
description: 可用性、セキュリティ、スケーラビリティ、および管理容易性のために Azure で多層アーキテクチャを実装する方法について説明します。
author: MikeWasson
ms.date: 05/03/2018
pnp.series.title: Windows VM workloads
pnp.series.next: multi-region-application
pnp.series.prev: multi-vm
ms.openlocfilehash: 0f170f2fbcbbfeace53db199cb5d3949415b5546
ms.sourcegitcommit: a5e549c15a948f6fb5cec786dbddc8578af3be66
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 05/06/2018
---
# <a name="n-tier-application-with-sql-server"></a>SQL Server を使用した n 層アプリケーション

この参照アーキテクチャでは、Windows 上の SQL Server をデータ層に使用して、N 層アプリケーション用に構成された VM と仮想ネットワークをデプロイする方法を示します。 [**こちらのソリューションをデプロイしてください**。](#deploy-the-solution) 

![[0]][0]

"*このアーキテクチャの [Visio ファイル][visio-download]をダウンロードします。*"

## <a name="architecture"></a>アーキテクチャ 

アーキテクチャには次のコンポーネントがあります。

* **リソース グループ。** [リソース グループ][resource-manager-overview]は、リソースをグループ化して、有効期間、所有者、またはその他の条件別に管理できるようにするために使用されます。

* **仮想ネットワーク (VNet) とサブネット。** どの Azure VM も、複数のサブネットにセグメント化できる VNet にデプロイされます。 階層ごとに個別のサブネットを作成します。 

* **NSG。** [ネットワーク セキュリティ グループ][nsg] (NSG) を使用して、VNet 内のネットワーク トラフィックを制限します。 たとえば、ここに示されている 3 層アーキテクチャでは、データベース層は Web フロントエンドからのトラフィックを受信せず、ビジネス層と管理サブネットからのトラフィックのみ受信します。

* **仮想マシン**。 VM の構成に関する推奨事項については、「[Run a Windows VM on Azure (Azure での Windows VM の実行)](./windows-vm.md)」および「[Run a Linux VM on Azure (Azure での Linux VM の実行)](./linux-vm.md)」をご覧ください。

* **可用性セット。** 階層ごとに[可用性セット][azure-availability-sets]を作成し、各階層に少なくとも 2 つの VM をプロビジョニングします。 こうすると VM がより高度な VM の[サービス レベル アグリーメント (SLA)][vm-sla] に対応できるようになります。 

* **VM スケール セット** (ここでは示されていません)。 [VM スケール セット][vmss]は可用性セットの代わりに使用します。 スケール セットを使用すると、層内の VM を簡単にスケールアウトできます。スケールアウトは、手動で実行することも、定義済みの規則に基づいて自動的に実行することもできます。

* **Azure Load Balancer。** [ロード バランサー][load-balancer]は、受信インターネット要求を各 VM インスタンスに分散します。 [パブリック ロード バランサー][load-balancer-external]を使用して受信インターネット トラフィックを Web 層に分散し、[内部ロード バランサー][load-balancer-internal]を使用して Web 層からのネットワーク トラフィックをビジネス層に分散します。

* **パブリック IP アドレス**。 パブリック IP アドレスは、パブリック ロード バランサーがインターネット トラフィックを受信するために必要です。

* **ジャンプボックス。** [要塞ホスト]とも呼ばれます。 管理者が他の VM に接続するために使用するネットワーク上のセキュアな VM です。 ジャンプボックスの NSG は、セーフ リストにあるパブリック IP アドレスからのリモート トラフィックのみを許可します。 NSG は、リモート デスクトップ (RDP) トラフィックを許可する必要があります。

* **SQL Server Always On 可用性グループ。** レプリケーションとフェールオーバーを有効にすることで、データ層で高い可用性を提供します。

* **Active Directory Domain Services (AD DS) サーバー。** SQL Server Always On 可用性グループは、Windows Server フェールオーバー クラスター (WSFC) テクノロジでフェールオーバーできるように、ドメインに参加しています。 

* **Azure DNS**。 [Azure DNS][azure-dns] は、DNS ドメインのホスティング サービスであり、Microsoft Azure インフラストラクチャを使用した名前解決を提供します。 Azure でドメインをホストすることで、その他の Azure サービスと同じ資格情報、API、ツール、課金情報を使用して DNS レコードを管理できます。

## <a name="recommendations"></a>Recommendations

実際の要件は、ここで説明するアーキテクチャとは異なる場合があります。 これらの推奨事項は原案として使用してください。 

### <a name="vnet--subnets"></a>VNet/サブネット

VNet を作成するときに、各サブネット内のリソースが要求する IP アドレスの数を決定します。 [CIDR] 表記を使用して、必要な IP アドレスにとって十分な規模のサブネット マスクと VNet アドレス範囲を指定します。 標準的な[プライベート IP アドレス ブロック][private-ip-space] (10.0.0.0/8、172.16.0.0/12、192.168.0.0/16) の範囲内にあるアドレス空間を使用します。

後で VNet とオンプレミスのネットワークとの間にゲートウェイを設定する必要がある場合は、オンプレミスのネットワークと重複しないアドレス範囲を選択します。 VNet を作成した後は、アドレス範囲を変更できません。

機能とセキュリティの要件を念頭に置いてサブネットを設計します。 同じ層または同じロール内のすべての VM は、同じサブネットに入れる必要があります。これがセキュリティ境界になります。 VNet とサブネットの設計に関する詳細については、「[Azure Virtual Network の計画と設計][plan-network]」を参照してください。

### <a name="load-balancers"></a>ロード バランサー

VM を直接インターネットに公開せず、代わりに各 VM にプライベート IP アドレスを付与します。 クライアントは、パブリック ロード バランサーの IP アドレスを使用して接続します。

ロード バランサー規則を定義して、ネットワーク トラフィックを VM に転送します。 たとえば、HTTP トラフィックを有効にするには、フロントエンド構成からのポート 80 をバックエンド アドレス プールのポート 80 にマッピングする規則を作成します。 クライアントがポート 80 に HTTP 要求を送信するときに、ロード バランサーは、発信元 IP アドレスを含む[ハッシュ アルゴリズム][load-balancer-hashing]を使用して、バックエンド IP アドレスを選択します。 この方法で、クライアント要求がすべての VM に配布されます。

### <a name="network-security-groups"></a>ネットワーク セキュリティ グループ

NSG ルールを使用して階層間のトラフィックを制限します。 たとえば、上の 3 層アーキテクチャでは、Web 層はデータベース層と直接通信しません。 これを強制するには、データベース層が Web 層のサブネットからの着信トラフィックをブロックする必要があります。  

1. VNet からのすべての受信トラフィックを拒否します。 (ルール内で `VIRTUAL_NETWORK` タグを使用します。) 
2. ビジネス層のサブネットからの受信トラフィックを許可します。  
3. データベース層のサブネットからの受信トラフィックを許可します。 このルールにより、データベースのレプリケーションやフェールオーバーに必要な、データベース VM 間の通信が可能になります。
4. ジャンプボックスのサブネットからの RDP トラフィック (ポート 3389) を許可します。 このルールによって、管理者がジャンプボックスからデータベース層に接続できるようにします。

最初のルールよりも高い優先順位を設定してルール 2 から 4 を作成し、最初のルールをオーバーライドします。


### <a name="sql-server-always-on-availability-groups"></a>SQL Server Always On 可用性グループ

SQL Server の高可用性のために [Always On 可用性グループ][sql-alwayson]の使用をお勧めします。 Windows Server 2016 に先立って、Always On 可用性グループはドメイン コントローラーを必要とし、可用性グループ内のすべてのノードが同じ AD ドメイン内にある必要があります。

他の層は[可用性グループ リスナー][sql-alwayson-listeners]を使用してデータベースに接続します。 リスナーを使用することで、SQL クライアントは SQL Server の物理インスタンスの名前を知らなくても接続できます。 データベースにアクセスする VM はドメインに参加している必要があります。 クライアント (ここでは、別の層) は、DNS を使用してリスナーの仮想ネットワーク名を IP アドレスに解決します。

SQL Server Always On 可用性グループを構成する手順は、次のとおりです。

1. Windows Server フェールオーバー クラスタリング (WSFC) クラスター、SQL Server Always On 可用性グループ、プライマリ レプリカを作成します。 詳細については、「[AlwaysOn 可用性グループの概要][sql-alwayson-getting-started]」を参照してください。 
2. 静的プライベート IP アドレスを持つ内部ロード バランサーを作成します。
3. 可用性グループ リスナーを作成し、リスナーの DNS 名を内部ロード バランサーの IP アドレスにマッピングします。 
4. SQL Server リスニング ポート (既定ではTCP ポート 1433) に対するロード バランサーのルールを作成します。 ロード バランサーのルールでは *Floating IP* (Direct Server Return とも呼ばれます) を有効にする必要があります。 これにより VM が直接クライアントに応答でき、プライマリ レプリカへの直接接続が可能になります。
  
   > [!NOTE]
   > Floating IP が有効になっている場合は、フロントエンド ポート番号を、ロード バランサーのルール内のバックエンド ポート番号と同じにする必要があります。
   > 
   > 

SQL クライアントが接続を試みると、ロード バランサーがプライマリ レプリカに接続要求をルーティングします。 別のレプリカへのフェールオーバーが発生した場合は、ロード バランサーはその後の要求を自動的に新しいプライマリ レプリカにルーティングします。 詳細については、[SQL Server Always On 可用性グループの ILB リスナーの構成][sql-alwayson-ilb]に関する記事を参照してください。

フェールオーバー中は、既存のクライアント接続は閉じられます。 フェールオーバーが完了すると、新しい接続は新しいプライマリ レプリカにルーティングされます。

アプリケーションが書き込みよりもはるかに多くの読み取りを行う場合は、読み取り専用クエリの一部をセカンダリ レプリカにオフロードできます。 「[Using a Listener to Connect to a Read-Only Secondary Replica (Read-Only Routing)][sql-alwayson-read-only-routing]」(リスナーを使用した読み取り専用セカンダリ レプリカへの接続 (読み取り専用ルーティング)) を参照してください。

可用性グループの[手動フェールオーバーの強制][sql-alwayson-force-failover]によってデプロイをテストします。

### <a name="jumpbox"></a>Jumpbox

アプリケーション ワークロードを実行する VM へのパブリック インターネットからの RDP アクセスを許可しないでください。 代わりに、これらの VM へのすべての RDP アクセスは、ジャンプボックスを経由する必要があります。 管理者はジャンプボックスにログインし、次にジャンプボックスから他の VM にログインします。 ジャンプボックスは、既知の安全な IP アドレスからのみ、インターネットからの RDP トラフィックを許可します。

ジャンプボックスのパフォーマンス要件は最小限に抑えられているので、小さな VM サイズを選択します。 ジャンプボックス用に[パブリック IP アドレス]を作成します。 ジャンプボックスを、他の VM と同じ VNet 内の、個別の管理サブネット内に配置します。

ジャンプボックスをセキュリティで保護するには、安全な一連のパブリック IP アドレスからのみ RDP 接続を許可する NSG ルールを追加します。 他のサブネットに対しても NSG を構成して、管理サブネットからの RDP トラフィックを許可します。

## <a name="scalability-considerations"></a>スケーラビリティに関する考慮事項

[VM スケール セット][vmss]を使用して、同一の VM のセットをデプロイして管理できます。 スケール セットでは、パフォーマンス メトリックに基づく自動スケールがサポートされます。 VM の負荷が増えると、追加の VM が自動的にロード バランサーに追加されます。 VM をすばやくスケール アウトしたり、自動スケールしたりする必要がある場合は、スケール セットを検討してください。

スケール セットにデプロイされる VM を構成するには、2 つの基本的な方法があります。

- 拡張機能を使用してプロビジョニングされた後に VM を構成します。 この方法では、新しい VM インスタンスは、拡張機能なしの VM よりも起動に時間がかかる場合があります。

- カスタム ディスク イメージと共に [Managed Disk](/azure/storage/storage-managed-disks-overview) をデプロイします。 このオプションの方が早くデプロイできる場合があります。 ただし、イメージを最新の状態に保つ必要があります。

詳しい考慮事項については、「[スケール セットの設計上の考慮事項][vmss-design]」を参照してください。

> [!TIP]
> 自動スケールのソリューションを使用する場合は、十分前もって実稼働レベルのワークロードでテストしてください。

各 Azure サブスクリプションには、リージョンごとの VM の最大数などの、既定の制限があります。 サポート リクエストを提出することで、制限値を上げることができます。 詳細については、「[Azure サブスクリプションとサービスの制限、クォータ、制約][subscription-limits]」をご覧ください。

## <a name="availability-considerations"></a>可用性に関する考慮事項

VM スケール セットを使用していない場合は、同じ層の VM を可用性セットに配置します。 [Azure VM の可用性 SLA][vm-sla] をサポートするために、可用性セット内に少なくとも 2 つの VM を作成します。 詳細については、「 [Virtual Machines の可用性管理][availability-set]」を参照してください。 

ロード バランサーは、[正常性プローブ][health-probes]を使用して VM インスタンスの可用性を監視します。 タイムアウト期間内にプローブがインスタンスに到達できなかった場合、ロード バランサーはその VM へのトラフィックの送信を停止します。 ただし、ロード バランサーは引き続きプローブを行い、VM が再び使用可能になると、ロード バランサーはその VM へのトラフィックの送信を再開します。

ロード バランサーの正常性プローブには、次のような推奨事項があります。

* プローブでは、HTTP または TCP のいずれかをテストできます。 VM で HTTP サーバーを実行している場合、HTTP プローブを作成します。 それ以外の場合は、TCP プローブを作成します。
* HTTP プローブでは、HTTP エンドポイントへのパスを指定します。 プローブでは、このパスからの HTTP 200 の応答をチェックします。 これには、ルート パス ("/")、またはアプリケーションの正常性をチェックするためのカスタム ロジックを実装した正常性監視エンドポイントを指定できます。 エンドポイントでは、匿名の HTTP 要求を許可する必要があります。
* このプローブは、[既知の IP アドレス][health-probe-ip]である 168.63.129.16 から送信されます。 この IP アドレスとの間のトラフィックを、どのファイアウォール ポリシーまたはネットワーク セキュリティ グループ (NSG) 規則でもブロックしていないことを確認してください。
* [正常性プローブ ログ][health-probe-log]を使用して、正常性プローブの状態を表示します。 各ロード バランサーに対して Azure Portal のログ記録を有効にします。 ログは Azure Blob Storage に書き込まれます。 ログには、プローブの応答に失敗したことが原因で、ネットワーク トラフィックを受信していない バック エンドの VM 数が表示されます。

[VM の Azure SLA][vm-sla] によって提供されるものよりも高い可用性が必要な場合は、フェールオーバーに Azure Traffic Manager を使用して、2 つのリージョンでのアプリケーションのレプリケーションを検討します。 詳細については、「[高可用性のためのマルチリージョン n 層アプリケーション][multi-dc]」を参照してください。  

## <a name="security-considerations"></a>セキュリティに関する考慮事項

仮想ネットワークは、Azure のトラフィックの分離境界です。 ある VNet 内の VM が別の VNet 内の VM と直接通信することはできません。 トラフィックを制限する[ネットワーク セキュリティ グループ][nsg] (NSG) を作成していないかぎり、同じ VNet 内の VM 同士は通信可能です。 詳細については、「[Microsoft クラウド サービスとネットワーク セキュリティ][network-security]」をご覧ください。

インターネット トラフィックを受信する場合、ロード バランサーの規則でバック エンドに到達できるトラフィックを定義しています。 しかし、ロード バランサーの規則では IP の安全な一覧をサポートしていないため、特定のパブリック IP アドレスを安全な一覧に追加したい場合は、NSG をサブネットに追加してください。

ネットワーク仮想アプライアンス (NVA) を追加してインターネットと Azure Virtual Network の間の DMZ を作成することを検討してください。 NVA とは、ネットワーク関連のタスク (ファイアウォール、パケット インスペクション、監査、カスタム ルーティングなど) を実行できる仮想アプライアンスの総称です。 詳細については、[Azure とインターネットの間の DMZ の実装][dmz]に関する記事を参照してください。

機密の保存データを暗号化し、[Azure Key Vault][azure-key-vault] を使用してデータベース暗号化キーを管理します。 Key Vault では、ハードウェア セキュリティ モジュール (HSM) に暗号化キーを格納することができます。 詳細については、[Azure VM 上の SQL Server 向け Azure Key Vault 統合の構成][sql-keyvault]に関するページを参照してください。 データベース接続文字列などのアプリケーション シークレットも Key Vault に格納することをお勧めします。

## <a name="deploy-the-solution"></a>ソリューションのデプロイ方法

このリファレンス アーキテクチャのデプロイについては、[GitHub][github-folder] を参照してください。 

### <a name="prerequisites"></a>前提条件

1. [参照アーキテクチャ][ref-arch-repo] GitHub リポジトリに ZIP ファイルを複製、フォーク、またはダウンロードします。

2. Azure CLI 2.0 がコンピューターにインストールされていることを確認してください。 CLI をインストールするには、「[Azure CLI 2.0 のインストール][azure-cli-2]」の手順に従ってください。

3. [Azure の構成要素][azbb] npm パッケージをインストールします。

   ```bash
   npm install -g @mspnp/azure-building-blocks
   ```

4. コマンド プロンプト、bash プロンプト、または PowerShell プロンプトから、以下のコマンドの 1 つを使用して Azure アカウントにログインし、プロンプトに従います。

   ```bash
   az login
   ```

### <a name="deploy-the-solution-using-azbb"></a>azbb を使用したソリューションのデプロイ

N 層アプリケーションの参照アーキテクチャで Windows VM をデプロイするには、次の手順に従います。

1. 上の前提条件の手順 1 で複製したリポジトリの `virtual-machines\n-tier-windows` フォルダーに移動します。

2. このパラメーター ファイルは、デプロイ内の各 VM の既定の管理者ユーザー名とパスワードを指定します。 参照アーキテクチャをデプロイする前に、これらを変更する必要があります。 `n-tier-windows.json` ファイルを開き、各 **adminUsername** および **adminPassword** フィールドを新しい設定に置き換えます。
  
   > [!NOTE]
   > このデプロイ中に実行される複数のスクリプトが **VirtualMachineExtension** オブジェクトと、一部の **VirtualMachine** オブジェクトの **extensions** 設定の両方に存在します。 これらのスクリプトのいくつかには、今変更した管理者ユーザー名とパスワードが必要です。 これらのスクリプトをレビューして、正しい資格情報を指定したことを確認することをお勧めします。 正しい資格情報を指定していない場合は、デプロイが失敗する可能性があります。
   > 
   > 

ファイルを保存します。

3. 次に示すように、**azbb** コマンド ライン ツールを使用して参照アーキテクチャをデプロイします。

   ```bash
   azbb -s <your subscription_id> -g <your resource_group_name> -l <azure region> -p n-tier-windows.json --deploy
   ```

Azure の構成要素を使用してこのサンプルの参照アーキテクチャをデプロイする方法の詳細については、「[GitHub リポジトリ][git]」を参照してください。


<!-- links -->
[dmz]: ../dmz/secure-vnet-dmz.md
[multi-dc]: multi-region-sql-server.md
[n-tier]: n-tier.md
[azbb]: https://github.com/mspnp/template-building-blocks/wiki/Install-Azure-Building-Blocks
[azure-administration]: /azure/automation/automation-intro
[azure-availability-sets]: /azure/virtual-machines/virtual-machines-windows-manage-availability#configure-each-application-tier-into-separate-availability-sets
[azure-cli]: /azure/virtual-machines-command-line-tools
[azure-cli-2]: https://docs.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest
[azure-dns]: /azure/dns/dns-overview
[azure-key-vault]: https://azure.microsoft.com/services/key-vault
[要塞ホスト]: https://en.wikipedia.org/wiki/Bastion_host
[cidr]: https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing
[chef]: https://www.chef.io/solutions/azure/
[git]: https://github.com/mspnp/template-building-blocks
[github-folder]: https://github.com/mspnp/reference-architectures/tree/master/virtual-machines/n-tier-windows
[lb-external-create]: /azure/load-balancer/load-balancer-get-started-internet-portal
[lb-internal-create]: /azure/load-balancer/load-balancer-get-started-ilb-arm-portal
[load-balancer-external]: /azure/load-balancer/load-balancer-internet-overview
[load-balancer-internal]: /azure/load-balancer/load-balancer-internal-overview
[nsg]: /azure/virtual-network/virtual-networks-nsg
[operations-management-suite]: https://www.microsoft.com/server-cloud/operations-management-suite/overview.aspx
[plan-network]: /azure/virtual-network/virtual-network-vnet-plan-design-arm
[private-ip-space]: https://en.wikipedia.org/wiki/Private_network#Private_IPv4_address_spaces
[パブリック IP アドレス]: /azure/virtual-network/virtual-network-ip-addresses-overview-arm
[puppet]: https://puppetlabs.com/blog/managing-azure-virtual-machines-puppet
[ref-arch-repo]: https://github.com/mspnp/reference-architectures
[sql-alwayson]: https://msdn.microsoft.com/library/hh510230.aspx
[sql-alwayson-force-failover]: https://msdn.microsoft.com/library/ff877957.aspx
[sql-alwayson-getting-started]: https://msdn.microsoft.com/library/gg509118.aspx
[sql-alwayson-ilb]: /azure/virtual-machines/windows/sql/virtual-machines-windows-portal-sql-alwayson-int-listener
[sql-alwayson-listeners]: https://msdn.microsoft.com/library/hh213417.aspx
[sql-alwayson-read-only-routing]: https://technet.microsoft.com/library/hh213417.aspx#ConnectToSecondary
[sql-keyvault]: /azure/virtual-machines/virtual-machines-windows-ps-sql-keyvault
[vm-sla]: https://azure.microsoft.com/support/legal/sla/virtual-machines
[vnet faq]: /azure/virtual-network/virtual-networks-faq
[wsfc-whats-new]: https://technet.microsoft.com/windows-server-docs/failover-clustering/whats-new-in-failover-clustering
[Nagios]: https://www.nagios.org/
[Zabbix]: http://www.zabbix.com/
[Icinga]: http://www.icinga.org/
[visio-download]: https://archcenter.blob.core.windows.net/cdn/vm-reference-architectures.vsdx
[0]: ./images/n-tier-sql-server.png "Microsoft Azure を使用した N 層アーキテクチャ"
[resource-manager-overview]: /azure/azure-resource-manager/resource-group-overview 
[vmss]: /azure/virtual-machine-scale-sets/virtual-machine-scale-sets-overview
[load-balancer]: /azure/load-balancer/load-balancer-get-started-internet-arm-cli
[load-balancer-hashing]: /azure/load-balancer/load-balancer-overview#load-balancer-features
[vmss-design]: /azure/virtual-machine-scale-sets/virtual-machine-scale-sets-design-overview
[subscription-limits]: /azure/azure-subscription-service-limits
[availability-set]: /azure/virtual-machines/virtual-machines-windows-manage-availability
[health-probes]: /azure/load-balancer/load-balancer-overview#load-balancer-features
[health-probe-log]: /azure/load-balancer/load-balancer-monitor-log
[health-probe-ip]: /azure/virtual-network/virtual-networks-nsg#special-rules
[network-security]: /azure/best-practices-network-security