---
title: "使用 ONOS 來控制 P4 Switch"
date: 2020-12-14T14:59:13+08:00
draft: false
tags: [P4, ONOS, SDN]
categories: ["P4 Language"]
---

# ONOS 與 P4

![](https://i.imgur.com/HPk0acL.png)

如果要在 ONOS 上使用 P4 主要是用以下這些步驟進行開發

1. 撰寫 P4 程式
2. 編譯後得到 P4Info 文件
3. 撰寫與編譯 Pipeconf 應用程式，此時會將 P4Info、BMv2 JSON、Tofino Binary 等相關文件打包成 oar 檔
4. 撰寫控制應用程式，可以是 Pipeline-agnostic 或是 Pipeline-aware

而其代表的意義如下

* Pipeline-agnostic application: 
    這類型的應用程式並不會知道即將面對的裝置所用的 Pipeline 長怎麼樣，僅會專心去做要做的事情，舉例來說 ProxyArp 或是 LLDP provider 等等。
    
* Pipeline-aware application: 
    這類型的則是針對特定 Pipeline 所設計的，會直接產生出特定的 Flow 以及 Group，並直接安裝到特定的 Table 中。

# ONOS 與 PI

講到 P4 的控制層面一定會想到 PI(protocol independent) 架構
雖然也是有人使用 OpenFlow 來當作控制協議
但其在彈性上並不是很好
故目前大多數人會傾向使用 P4Runtime

![](https://i.imgur.com/GrBL5wX.png)


上圖是 PI 在 ONOS 架構中的設計
層級可分為三層分別是 Protocol、Driver 與 Core

在 Protocol 層有 P4Runtime，其是透過 gRPC 的方式進行溝通
往上一層是 Driver 層，依照所使用的設備選擇驅動
目前支援 Barefoot Tofino, Mellanox Spectrum 與 BMv2 ... 等
在往上一層是 Core 層，也就是 PI 架構所在的一層
其包含三大模組 PI module、Flow Rule Translation 以及 Pipeconf
若 ONOS 要控制底層 P4 交換器則需撰寫 Pipeconf 應用

# 透過 ONOS 控制 bmv2 switch
![](https://i.imgur.com/VAvgwfy.png)

當寫好一個 P4 檔如果需要控制器去控制的話
ONOS 控制器是一個很好的選擇 其提供 Pipeconf 的方式
讓 ONOS 知道底層 P4 交換器有哪些 pipeline 與 table
ONOS 目前有提供一些 default pipeconf 給使用者選擇
也就是 basic 與 fabric

一個完整的 pipeconf 包含以下元素
* pipeline config ( bmv2 json, Tofino binary… )
* p4info
* PipeconfLoader （進入點）
* Interpreter ( 可有可無 )
* Pipeliner ( 可有可無 )

這邊主要會以 PipeconfLoader、Interpreter、Pipeliner 的撰寫來講解

### 撰寫 PipeconfLoader
當 ONOS 啟動 PipeconfLoader 時，會透過 PiPipeconfService 去註冊 Pipeconf
此程式為 Pipeconf 的核心以及程式的進入點
舉例來說如果希望 Pipeconf 擁有 Interpreter 以及 Pipeliner 兩種 Driver Behavior
以及使用 bmv2 json + p4info，可以這樣寫

```java
PiPipeconfId id = new PiPipeconfId("my-test-pipeconf");
URL bmv2ConfigPath = TestPipeconfLoader.class.getResource("/test-pipeline.json");
URL p4infoPath = TestPipeconfLoader.class.getResource("/test-pipeline.p4info");

pipeconf = DefaultPiPipeconf.builder()
    .withId(id)
    .addBehaviour(Pipeliner.class, TestPipeliner.class)
    .addBehaviour(PiPipelineInterpreter.class, TestInterpreter.class)
    .addExtension(PiPipeconf.ExtensionType.BMV2_JSON, bmv2ConfigPath)
    .addExtension(PiPipeconf.ExtensionType.P4_INFO_TEXT, p4infoPath)
    .build();

piPipeconfService.register(pipeconf);
```

上述的程式可以放在 activate function 中
ONOS 安裝這個 App（oar）時就會將 Pipeconf 載入

### 撰寫 Interpreter

Interpreter 主要是處理 ONOS API 轉換至 PI API 的工作

* mapCriterionType: 將 ONOS Criterion 轉換成 PI match 欄位
* mapPiMatchFieldId: 將 PI match 欄位轉換成普通的 Criterion

```java
@Override
public Optional<PiMatchFieldId> mapCriterionType(Criterion.Type type) {
    if (CRITERION_MAP.containsKey(type)) {
        return Optional.of(PiMatchFieldId.of(CRITERION_MAP.get(type)));
    } else {
        return Optional.empty();
    }
}
```
* mapFlowRuleTableId: 將數字 Id 轉換成像是 P4 一樣使用字串的 Id
* mapPiTableId: 將字串 Id 轉換回數字 Id
```java
@Override
public Optional<PiTableId> mapFlowRuleTableId(int flowRuleTableId) {
    return Optional.empty();
}
```
* mapTreatment: 將 ONOS 的 TrafficTreatment 加上 TableId 轉換成 PiAction，這主要是要解決多個 Action 對到單一個 Action 的問題
```java
@Override
public PiAction mapTreatment(TrafficTreatment treatment, PiTableId piTableId)
        throws PiInterpreterException {
    throw new PiInterpreterException("Treatment mapping not supported");
}
```
* mapOutboundPacket: 將 ONOS PacketOut 轉換成 PI 格式，即 metadata + paylad
```java
@Override
public Collection<PiPacketOperation> mapOutboundPacket(OutboundPacket packet)
        throws PiInterpreterException {
    TrafficTreatment treatment = packet.treatment();

    // Packet-out in main.p4 supports only setting the output port,
    // i.e. we only understand OUTPUT instructions.
    List<OutputInstruction> outInstructions = treatment
            .allInstructions()
            .stream()
            .filter(i -> i.type().equals(OUTPUT))
            .map(i -> (OutputInstruction) i)
            .collect(toList());

    if (treatment.allInstructions().size() != outInstructions.size()) {
        // There are other instructions that are not of type OUTPUT.
        throw new PiInterpreterException("Treatment not supported: " + treatment);
    }

    ImmutableList.Builder<PiPacketOperation> builder = ImmutableList.builder();
    for (OutputInstruction outInst : outInstructions) {
        if (outInst.port().isLogical() && !outInst.port().equals(FLOOD)) {
            throw new PiInterpreterException(format(
                    "Packet-out on logical port '%s' not supported",
                    outInst.port()));
        } else if (outInst.port().equals(FLOOD)) {
            // To emulate flooding, we create a packet-out operation for
            // each switch port.
            final DeviceService deviceService = handler().get(DeviceService.class);
            for (Port port : deviceService.getPorts(packet.sendThrough())) {
                builder.add(buildPacketOut(packet.data(), port.number().toLong()));
            }
        } else {
            // Create only one packet-out for the given OUTPUT instruction.
            builder.add(buildPacketOut(packet.data(), outInst.port().toLong()));
        }
    }
    return builder.build();
}
```
* mapInboundPacket: 將 ONOS PacketIn 轉換成 PI 格式
```java
@Override
public InboundPacket mapInboundPacket(PiPacketOperation packetIn, DeviceId deviceId)
        throws PiInterpreterException {

    // Find the ingress_port metadata.
    final String inportMetadataName = "ingress_port";
    Optional<PiPacketMetadata> inportMetadata = packetIn.metadatas()
            .stream()
            .filter(meta -> meta.id().id().equals(inportMetadataName))
            .findFirst();

    if (!inportMetadata.isPresent()) {
        throw new PiInterpreterException(format(
                "Missing metadata '%s' in packet-in received from '%s': %s",
                inportMetadataName, deviceId, packetIn));
    }

    // Build ONOS InboundPacket instance with the given ingress port.

    // 1. Parse packet-in object into Ethernet packet instance.
    final byte[] payloadBytes = packetIn.data().asArray();
    final ByteBuffer rawData = ByteBuffer.wrap(payloadBytes);
    final Ethernet ethPkt;
    try {
        ethPkt = Ethernet.deserializer().deserialize(
                payloadBytes, 0, packetIn.data().size());
    } catch (DeserializationException dex) {
        throw new PiInterpreterException(dex.getMessage());
    }

    // 2. Get ingress port
    final ImmutableByteSequence portBytes = inportMetadata.get().value();
    final short portNum = portBytes.asReadOnlyBuffer().getShort();
    final ConnectPoint receivedFrom = new ConnectPoint(
            deviceId, PortNumber.portNumber(portNum));

    return new DefaultInboundPacket(receivedFrom, ethPkt, rawData);
}
```

### 撰寫 Pipeliner
Pipeliner 是專門將 FlowObjective 轉換成 Flow + Group 的一個組件

* FilteringObjective：用來表示允許或是擋掉封包進入 Pipeliner 的規則
```java
@Override
public void filter(FilteringObjective obj) {
    obj.context().ifPresent(c -> c.onError(obj, ObjectiveError.UNSUPPORTED));
}
```

* ForwardingObjective：用來描述封包在 Pipeliner 中需要如何去處理

```java
@Override
public void forward(ForwardingObjective obj) {
    if (obj.treatment() == null) {
        obj.context().ifPresent(c -> c.onError(obj, ObjectiveError.UNSUPPORTED));
    }

    // Whether this objective specifies an OUTPUT:CONTROLLER instruction.
    final boolean hasCloneToCpuAction = obj.treatment()
            .allInstructions().stream()
            .filter(i -> i.type().equals(OUTPUT))
            .map(i -> (Instructions.OutputInstruction) i)
            .anyMatch(i -> i.port().equals(PortNumber.CONTROLLER));

    if (!hasCloneToCpuAction) {
        // We support only objectives for clone to CPU behaviours (e.g. for
        // host and link discovery)
        obj.context().ifPresent(c -> c.onError(obj, ObjectiveError.UNSUPPORTED));
    }

    // Create an equivalent FlowRule with same selector and clone_to_cpu action.
    final PiAction cloneToCpuAction = PiAction.builder()
            .withId(PiActionId.of(CLONE_TO_CPU))
            .build();

    final FlowRule.Builder ruleBuilder = DefaultFlowRule.builder()
            .forTable(PiTableId.of(ACL_TABLE))
            .forDevice(deviceId)
            .withSelector(obj.selector())
            .fromApp(obj.appId())
            .withPriority(obj.priority())
            .withTreatment(DefaultTrafficTreatment.builder()
                                   .piTableAction(cloneToCpuAction).build());

    if (obj.permanent()) {
       ruleBuilder.makePermanent();
    } else {
       ruleBuilder.makeTemporary(obj.timeout());
    }

    final GroupDescription cloneGroup = Utils.buildCloneGroup(
            obj.appId(),
            deviceId,
            CPU_CLONE_SESSION_ID,
            // Ports where to clone the packet.
            // Just controller in this case.
            Collections.singleton(PortNumber.CONTROLLER));

    switch (obj.op()) {
        case ADD:
            flowRuleService.applyFlowRules(ruleBuilder.build());
            groupService.addGroup(cloneGroup);
            break;
        case REMOVE:
            flowRuleService.removeFlowRules(ruleBuilder.build());
            groupService.removeGroup(deviceId, cloneGroup.appCookie(), obj.appId());
            break;
        default:
            log.warn("Unknown operation {}", obj.op());
    }

    obj.context().ifPresent(c -> c.onSuccess(obj));
}
```
* NextObjective：用來描述 Egress table 裡面需要放置什麼樣的東西
```java
@Override
public void next(NextObjective obj) {
    obj.context().ifPresent(c -> c.onError(obj, ObjectiveError.UNSUPPORTED));
}

@Override
public List<String> getNextMappings(NextGroup nextGroup) {
    // We do not use nextObjectives or groups.
    return Collections.emptyList();
}
```

# 參考文獻
* [P4 台灣社群](https://p4tw.org/)
* [ONOS+P4 Tutorial for Beginners](https://wiki.onosproject.org/pages/viewpage.action?pageId=16122675)