---
title: "EVPN"
date: 2021-01-27T13:24:31+08:00
draft: false
tags: [evpn]
categories: ["protocol"]
---

# EVPN

EVPN 的全名為 Ethernet VPN，其最先被定義在 ```RFC 7432```，文件名稱為 BGP MPLS-Based Ethernet VPN。做為一個 MPLS 的 L2 VPN，之前就有相關的標準存在，像是：

* ```RFC4447``` 提出基於 LDP 的 VPWS
* ```RFC4762``` 提出基於 LDP 的 VPLS
* ```RFC4761``` 提出基於 BGP 的 VPLS

一台交換器最重要的元件為轉發表，其支撐了整個第二層網路的運作，包含 Learn, Flood, Filter, Forward 這幾個動作。轉發表記錄了 MAC 地址與其相對應的端口，與路由表不同，轉發表並沒有一個控制層存在，而是透過資料層的自我學習機制建立。VPLS 亦是採用同樣的方式建構轉發表，在資料層進行 Flood & Learn。為了避免轉發表無限膨脹，需在轉發表設置 aging，清除不常用的轉發項目。隨著網路規模的增加，會對 PE 和 WAN 網路造成很大的負擔，進而限制 VPLS 的網路規模。由於近期網路領域出現了 SDN 的概念，因此提出了 EVPN。EVPN 架構是在現有的 BGP VPLS（```RFC4761```）方案上，參考 BGP/MPLS L3 VPN（```RFC4364```）架構所提出的。對於 EVPN 來說，控制層是 MP-BGP，而資料層可為 MPLS、PBB、VXLAN。本篇文章會以  ```RFC 7432``` 為主，其擴展協議並不會包含在此文章中。

![](https://i.imgur.com/AkE8yhD.png "EVPN Standard")

# EVPN 名詞意義

本篇文章所使用到的網路技術用語會在此段落進行說明。

#### Normal
* CE：Customer Edge device，用戶端設備，可能是 host, router, switch 等。
* PE：Provider Edge device，網路服務供應商設備。
* BD：Broadcast Domain，一個廣播封包可送達的範圍就稱為 Broadcast Domain。

#### EVPN
* EVI：EVPN Instance，EVPN 是一個虛擬私有網路，可以在實體設備上實現一個或多個 EVPN 環境，每個環境是互相獨立存在。
* ET：Ethernet Tag，用來區分在同一個 EVI 上不同的 MAC-VRF。
* ES：Ethernet Segment，CE 做 multihomed 時所形成的區域。
* ESI：Ethernet Segment Identifier，用於標示 ES，且整個網路中不會重複。
* MAC-VRF：MAC Virtual Routing and Forwarding table，PE 上的虛擬轉發單元。
* BUM：Broadcast, Unknown unicast, Multicast，是廣播、未知單播、多播的總稱。
* DF：Designated Forwarder，在 Multihomed 情況下，指定處理 BUM 流量的 PE。

#### Other
* AFI：Address Family Identifier, 表示所傳遞訊息之網路型態或位址格式。
* SAFI：Subsequent Addrss Family Identifier, 表示提供哪些 NLRI 附加訊息。
* NLRI：Network Layer Reachability Information, 網絡層可達性信息。
* RD：Route Distinguisher，用於區分不同的 EVI 之間重複的訊息。
* P-tunnel：穿越一家或多家服務提供商的隧道。
* PMSI：Provider Multicast Service Interface，允許 PE 之間的多播。

# EVPN 服務模式

### VLAN-Based Service Interface

每個 EVI 只會有一個 BD，連接一組用戶網路，每個 MAC-VRF 中只有一張 MAC 轉發表。

![](https://i.imgur.com/FqiIjDi.png "VLAN-Based Service")

在這種模式下由於每組的網路都是獨立的，隔離性最好，不會互相影響，但缺點是太浪費資源了，假如一個用戶需要 10 組 L2 VPN 網路，那這一個用戶就會需要 10 個 EVI，然而通常能提供的 EVI 數量是有限的，故此種模式只能適用於小規模的部屬環境。

### VLAN Bundle Service Interface

一個 EVI 擁有多個 BD，連接多組用戶網路，且多組用戶網路共用一個 MAC 轉發表。

![](https://i.imgur.com/690L79z.png "VLAN Bundle Service")

在此種模式下，EVI 內部類似於 VLAN with SVL (Share VLAN Learning)，實際上 BD 還是只有一個，透過邏輯分組區分不同的 VLAN。而轉發表類似於下圖所示。由於是透過 MAC 地址來做轉發，因此在此模式下須要求所有網路中的 MAC 地址不可重複，並且 EVPN 連結的用戶網路 VLAN ID 必須一致。

![](https://i.imgur.com/dAhSAIF.png "VLAN Bundle Service Forwarding Table")

### VLAN-Aware Bundle Service Interface

一個 EVI 擁有多個 BD，連接多組用戶網路，且每組用戶網路皆有獨立的 MAC 轉發表。

![](https://i.imgur.com/Mu6HaEs.png "VLAN-Aware Bundle Service")

這種模式的 EVI 內部類似於 VLAN with IVL (Independent VLAN Learning)，每個 MAC-VRF 有多張 MAC 轉發表，每張 MAC 轉發表對應一個用戶網路。轉發決策同時使用 MAC 地址與 ET，首先透過 ET 來定位到 MAC 轉發表，之後利用 MAC 地址查詢轉發表。此模式允許多組用戶網路之間存在相同 MAC 地址，且每組網路的 VLAN ID 可以不一致。

# BGP EVPN Route Type

PE 之間主要還是透過 MP-BGP 來進行轉發訊息的傳遞，而這也是 EVPN 與 VPLS 最大的區別。MP-BGP 為 BGP-4 的擴展，最新版本為 ```RFC4760```。首先在 MP-BGP 中定義 EVPN 的地址族（AFI: 25, SAFI: 70），在此地址族下又定義了不同的 NLRI ，同時也定義了多個新的 BGP Extended Community (```RFC4360```)。

#### MP_REACH_NLRI

![](https://i.imgur.com/GoS548R.png "MP_REACH_NLRI Format")

#### EVPN NLRI

![](https://i.imgur.com/zzLPOL4.png "EVPN NLRI")


#### EVPN NLRI Route Type
| Route Type |        Route Description        |                  Route Usage                  |
|:----------:|:-------------------------------:|:---------------------------------------------:|
|    0x01    |  Ethernet Auto-Discovery Route  |  Endpoint Discovery, Aliasing, Mass Withdraw  |
|    0x02    |     Mac Advertisement Route     |             MAC/IP Advertisement              |
|    0x03    |    Inclusive Multicast Route    |               BUM Flooding Tree               |
|    0x04    |     Ethernet Segment Route      |           ES Discovery, DF Election           |

#### EVPN Extended Community

|    Type     |            Description             |           Usage            |
|:-----------:|:----------------------------------:|:--------------------------:|
|  0x06/0x01  |    ESI Label Extended Community    |    Split Horizon Label     |
|  0x06/0x02  |       ES-Import Route Target       | Redundancy Group Discovery |
|  0x06/0x00  |  MAC Mobility Extended Community   |        MAC Mobility        |
| 0x03/0x030d | Default Gateway Extended Community |      Default Gateway       |

### Ethernet Auto-Discovery Route (Type 1)

EVPN Type 1 主要用於實現 Leaf 之間的負載平衡，當在 EVPN 設備上配置 ES 後，會向所有的 EVPN 設備發送 Type 1 訊息，此外也可用於 Fast Convergence 以及 Aliasing。標頭格式如下圖：

![](https://i.imgur.com/3sztNJw.png "EVPN Type 1")

#### ESI Label Extended Community
![](https://i.imgur.com/TpSVRjx.png "ESI Label Extended Community")

### MAC/IP Advertisement Route (Type 2)

EVPN Type 2 用於在設備之間同步 MAC/IP 地址資訊，Host/VM 在上線時通常會發送 DHCP 和 ARP，連接的 EVPN 設備會依據這兩個封包學習其 MAC/IP 地址，建構出本地端的轉發表與 ARP 表，並透過 Type 2 通知其他的 EVPN 設備。標頭格式如下圖：

![](https://i.imgur.com/piYJtW5.png "EVPN Type 2")

只學習到 MAC 地址是不夠的，還需要維護 MAC 地址的狀態，才能夠防止出現路由黑洞。尤其是在雲端環境中，虛擬機的移動是常有的事，因此必須要處理好 MAC 地址移動問題，而 EVPN 透過 Type 2 附加 MAC Mobility Extended Community 來解決。

#### MAC Mobility Extended Community
![](https://i.imgur.com/QKJNs5z.png "MAC Mobility Extended Community")

#### Default Gateway Extended Community
![](https://i.imgur.com/f59bjHa.png)

### Inclusive Multicast Ethernet Tag Route (Type 3)

EVPN Type 3 是用於處理 BUM 流量，在 NLRI 後面會附帶一個 PMSI Attribute，而其中的 tunnel 類型指定了 BUM 流量的處理方式。EVPN 既能夠使用 Ingress Replication，亦可使用 Underlay Replication。PMSI Attribute 定義於 ```RFC6514```。標頭格式如下圖：

#### EVPN Type 3
![](https://i.imgur.com/qW6G1Hr.png "EVPN Type 3")

#### PMSI Attribute
![](https://i.imgur.com/KXypaqc.png "PMSI Attribute")

### Ethernet Segment Route (Type 4)

EVPN Type 4 是用於處理在 Multihomed 下重複收到相同封包問題，透過 PE 之間選出一個處理 BUM 流量的 DF，其他的 PE 會丟棄 BUM 封包。而另一個功用是自動發現相同 ES 的設備。標頭格式如下圖：

![](https://i.imgur.com/fCTZdT6.png "EVPN Type 4")

#### ES-Import Route Target
![](https://i.imgur.com/VU1WsEL.png "ES-Import Route Target")

# Multihoming Functions

EVPN 最大的特點在於支援 Multihomed 環境，在 ```RFC7432``` 中定義了各個應用場景下的行為，像是 ES Auto-discovery, Fast Convergence, Split Horizon, Aliasing 等。

### Multihomed Ethernet Segment Auto-discovery

透過 EVPN Type 4 與 ES-Import Route Target，擁有相同的 ESI 可以互相發現彼此且無須而外進行配置。以下圖舉例，PE2 與 PE1 擁有相同的 ESI 值（ES1），而 PE3 是 Single Homed，故其 ESI 值為 0。PE2 透過 EVPN Type 4 與 ES-Import Route Target 通知其他的 EVPN 設備其 ESI 值為 ES1。PE1 和 PE3 將會收到 PE2 送來的通知，由於 PE3 的 ESI 值與接收到的 ESI 值不同，所以會將其通知丟棄，反之 PE1 則是會接收這個通知並了解 PE2 與自己在同一 ES 內。

![](https://i.imgur.com/pFH7tnt.png "ES Auto-discovery")

### Fast Convergence

由於 EVPN 是使用 BGP 做路由協議，當 PE 與 CE 之間的連線中斷時，會發送一個 EVPN Type 2 撤銷通知給所有的 EVPN 設備。由於一條路由會發送一條撤銷通知，也就是說擁有 100 條路由的 PE 斷線時，會發送 100 條撤銷通知。若連接到 PE 的設備數量太多時，其收斂時間會很久。為了加快收斂時間，EVPN 透過 Type 1 設計了 Mass Withdraw 的機制，PE 發現與自己相連的 CE 連線中斷時，發送一個 Type 1 來向其他 EVPN 設備撤銷自己所屬的 ES。

![](https://i.imgur.com/otSBGOG.png "Fast Convergence")

以上圖舉例來說，PE1 與 CE1 的連線中斷時會發送一個 EVPN Type 1的撤銷通知，PE2 會被重新選為 DF，而 PE3 收到後會註銷所有送往 PE1 的路由。

### Split Horizon

CE 與兩個以上的 PE 以 All-Active 模式連接形成 ES，當 CE 發送 BUM 流量至非 DF 的 PE 設備後，會轉發至包含被選為 DF 的所有設備，DF 會將 BUM 流量傳回至 CE，這樣就會形成迴圈。EVPN 使用水平切割來解決這個問題。當 DF 收到由非 DF 轉送且為 CE 的 BUM 封包將會丟棄，這就是水平切割。

![](https://i.imgur.com/AbCyReH.png "Split Horizon")

為了做到水平切割機制，任何從非 DF 要轉發 BUM 封包都會插入 ESI 標籤，這個標籤是透過 EVPN Type 1 以及 ESI Label Extended Community 來建立的。PE1 會發送包含 ESI 標籤為 L1 的 EVPN Type 1 通知給 PE2 與 PE3，同樣的 PE2 也會發送 ESI 標籤為 L2 的通知給 PE1 與 PE3。這樣 PE1 與 PE2 就可互相知道各自的 ESI 標籤。

![](https://i.imgur.com/e6aekXU.png "Split Horizon")

當非 DF 的 PE1 轉發來自 CE1 的 BUM 給 DF 的 PE2 時，由於已事前知道 PE2 的 ESI 標籤為 L2，故在轉發給 PE2 前會在 BUM 封包內插入 L2。PE2 接收到 BUM 封包後發現此 BUM 封包來自同一個 ES 會將其丟棄。

### Aliasing

在 multihomed 的環境下，PE 之間可能因為某些原因造成只有其中一台學習到 MAC 地址，這導致遠端的 PE 只會收到其中一台的 EVPN Type 2 的通知，認為只有這條路徑可以到達 CE，影響到負載平衡的功能，而 Aliasing 機制就是為這個問題出現的。PE 在發送 EVPN Type 2 時，會對 NLRI 增加 ESI 字段，將 CE 的路由與 ES 關聯在一起。即使沒有收到其他 PE 的通知，遠端 PE 知道此流量可透過同 ES 的 PE 送達到目的地 CE，並進行負載平衡。

![](https://i.imgur.com/Xp9NWtg.png "Aliasing")

舉例來說，PE1 先被配置了 ES1 與 L1 標籤，並先傳送 EVPN Type 1 給其他的設備。在還沒完成配置好 PE2 的時候，CE1 就先將流量送往 ES1，故只有 PE1 學習到 CE1 的 MAC 地址並發送 EVPN Type 2 路由通知。當 PE2 配置完後(ES1 與 L20)發送 EVPN Type 1，當 PE3 收到後得知也可透過 PE2 送達到 CE1。因此，即使 PE3 沒有收到 PE2 的 EVPN Type 2 路由通知，PE3 在處理送往 CE1 的流量時也可在 PE1 與 PE2 之間進行負載平衡。

![](https://i.imgur.com/qINAvse.png "Aliasing")

### Designated Forwarder Election

經過 EVPN Type 4 的自動發現 ES，接著就是 DF 選舉。DF 是為了防止在處理 BUM 封包時造成的 Duplicate Packets，所以只有 DF 才能處理 BUM 流量，非 DF 收到封包後直接丟棄。DF 的選舉首先要建立基於 IP 地址的 Ordered List。接著透過公式 V mod N（V 是 ET 值，N 是 PE 的數量）來選擇誰是 DF。

舉例來說，ES1 上有 PE1 (IP: 1.1.1.1) 與 PE2 (IP: 2.2.2.2)，並且配置了兩個 MAC-VRF，分別是 ET 300 以及 ET 301。首先建立 Ordered List，PE1 為 0，PE2 為 1，接者透過計算可以得知 ET 300 的 DF 為 PE1，而 ET 301 的 DF 為 PE2。

#### Ordered List:
| Position |      PE       |
|:--------:|:-------------:|
|    0     | PE1 (1.1.1.1) |
|    1     | PE2 (2.2.2.2) |

#### Modulus Operation:
| Ethernet Tag ID | V mod N (2) |
|:---------------:|:-----------:|
|       300       |      0      |
|       301       |      1      |

# EVPN Packet Process

這邊主要是講述 EVPN 是如何處理封包，包含 MAC 學習、抑制 ARP Flooding、處理 BUM 流量、

### MAC Learning

CE 與 PE 之間通常害是透過資料層進行 MAC 學習，也就是說 PE 需要透過解析 CE 發送過來的封包，擷取出來源端 MAC 地址，並記錄在 MAC-VRF 中的 MAC 轉發表中。通常這些封包都是設備送出的第一個封包，像是 DHCP, ARP 等。使用傳統的方式主要是因為 CE 不需要了解 EVPN 網路的實現，且與 CE 設備有更好的互通性。

![](https://i.imgur.com/6Z2X48D.png "Mac Advertisement Route")

在 PE 與 PE 之間是透過 MP-BGP 傳遞路由訊息，上圖是 EVPN Type 2 的流程，當 PE1 學習到 CE1 的 MAC 地址（M1），PE1 會發送 EVPN Type 2 封包其他 EVPN 設備，此通知封包包含 RD、ESI (若非 multihomed 則其為 0)、MAC 地址、MPLS 標籤以及 IP 地址 (此為 optional)。

### ARP Suppression

為了避免廣播封包占用 EVPN 網路的頻寬，PE 會在本地端建立起自己的 ARP 表，紀錄 IP 地址與 MAC 地址的對應。後續 PE 收到 ARP 的請求封包會優先像本地的 ARP 表進行代理回應，若沒有對應的項目存在，才會將 ARP 封包進行 Flooding，這樣的機制可以大幅減少 ARP 封包 Flooding 的次數。

### BUM Packets

PE 處理 BUM 流量時可以使用 Ingress Replication、P2MP或是MP2MP 的 LSP。這邊先以 Ingress Replication 為主，處理 BUM 流量首先利用 EVPN Type 1 交換 ESI 標籤，透過 Split Horizon 機制避免 BUM 流量送回給來源端 CE。接著利用 EVPN Type 3 交換 Mcast 標籤後，PE 將會透過 Ingress Replication 來傳遞 BUM 流量。

![](https://i.imgur.com/UAnVYo8.png "Inclusive Multicast Route")

![](https://i.imgur.com/5vHSvWY.png "Process BUM Packets")

舉例來說，PE1 收到來自 PE2 的 EVPN Type 1 並知道 PE2 的 ESI 值為 L2。PE2 (16006) 與 PE3 (16001) 分別傳送 EVPN Type 3 告知 PE1 各自的 Mcast 標籤。當 PE1 收到從 CE1 送過來的廣播封包後，送往 PE3 的封包新增 Mcast 標籤 16001 並送往 PE3；送往 PE2 的封包新增 Mcast 標籤 16006 以及 ESI 標籤 L2 並送往 PE2。PE2 收到後由於 Split Horizon 機制並不會送回給 CE1。

### Ingress Replication

採用單播的方式，PE 將 BUM 流量傳送到透過 Tunnel 連接的其他 PE，收到 BUM 流量的 PE 會 Flooding 到與其連結的 CE，但並不會再送給其他 PE。如果有使用 Multihomed 的話要啟用 Split Horizon 確保 BUM 封包不會送回給 CE。

# MAC Mobility

在資料中心中 VM 的搬移是很常見的事情，因此要解決 MAC 地址移動所衍生的問題。MAC 地址的維護主要有兩種方式：(1) Host 移動後透過 ARP 封包對 PE 進行更新，(2) 若 Host 移動後等待轉發表超時移除後再來重新學習。第一種方式需要跨 Uderlay 進行 APR Flooding，而第二個收斂時間會相當慢。EVPN 使用了 MP-BGP 主動維護 MAC 地址，只要一個交換器學習到新的 MAC 地址，就同步到其他 EVPN 設備，這樣可以避免 ARP Flooding 與局部交換器無法收斂問題。

EVPN 利用 Type 2 + MAC Mobility Extended Community 來解決上述問題。當 Host 第一次與 EVPN 網路連接時，PE1 會學習到 MAC 地址並發送 EVPN Type 2 通知，若移動到到 PE2 時，PE2 會學習到 MAC 並更新本地路由資訊與發送 EVPN Type 2 + MAC Mobility Extended Community。為了同步訊息，EVPN 在 MAC Mobility Extended Community 引入了時序的概念，每移動一次其 seq 就會加一。

![](https://i.imgur.com/wCINCCR.png "MAC Mobility")

PE 在收到 EVPN Type 2 後會依據 MAC 地址與 seq 來判別此路由通知是否要更新路由表，在相同的 MAC 地址情況下，若 EVPN Type 2 的 seq 大於路由表中的 seq，則將此通知更新至路由表中並發送撤銷通告給其他的 PE；若 EVPN Type 2 的 seq 小於路由表中的 seq，則會丟棄該封包；若 EVPN Type 2 有相同的 seq，則選擇 IP 地址最小的當作最佳路由。

### MAC Duplication Issue

當 Host 從 CE1 複製到 CE2 時，若複製後的 Host 的 MAC 地址沒改與原本的 Host 一樣會發生以下情形：

1. H1 接上 PE1 並發送 EVPN Type 2 通知其他 PE
2. 將 H1 複製到 PE2 變成 H2，但與 H1 MAC 地址一樣
3. PE2 認為 H1 移動到 PE2 底下並發送 EVPN Type 2 更新通知
4. PE1 收到 EVPN Type 2 更新通知後更新路由表，當 H1 發送封包時又會認為 H1 移回來再發送 EVPN Type 2 更新通知
5. 當 H2 發送封包又會認為 H1 移回來 PE2，一直在 PE1、PE2 之間輪迴

為了防止上述情形發生，PE 可以設定在 M 秒內重複 N 次就認為 MAC Duplication 的情況發生，停止傳送 EVPN Type 2 並發出警告。

# Reference

* Bates, T., Chandra, R., Katz, D., &Rekhter, Y. (2007). Multiprotocol Extensions for BGP-4. Retrieved July2, 2020, from Ietf Rfc4760 website: https://tools.ietf.org/pdf/rfc4760.pdf
* Diptanshu Singh. (2014). EVPN: Intro to next gen L2VPN. Retrieved July2, 2020, from https://packetpushers.net/evpn-introduction-next-generation-l2vpn/
* R. Aggarwal N. Bitar, A. I. J. U. J. D. W. H. (2015). BGP MPLS-Based Ethernet VPN (IETF RFC 7432). Retrieved from https://tools.ietf.org/html/rfc7432
* 肖宏辉. (2017). EVPN简介及实现. Retrieved July2, 2020, from https://www.sdnlab.com/19650.html
* Juniper Networks. (2018). Overview of VLAN Services for EVPN. Retrieved July2, 2020, from https://www.juniper.net/documentation//en_US/junos/topics/concept/evpn-services-overview.html
* Juniper Networks. (2020). Overview of MAC Mobility.  Retrieved July7, 2020, from https://www.juniper.net/documentation//en_US/junos/topics/concept/mac-mobility.html