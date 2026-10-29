# 化工园区信息化平台 · 分布式事务（Seata Saga）实现示例

> **版本**：V1.0 | **日期**：2026-08-22
> **配套文档**：《化工园区信息化平台-微服务架构设计方案.md》《化工园区信息化平台-服务接口契约与事件设计.md》
> **场景**：特殊作业票审批长流程（提交→气体检测→视频抽查→园区审核→批准作业），跨 `safety-svc` / `device-svc` / `monitor-svc` / 视频AI网关。

---

## 目录

1. [为什么用 Saga](#一为什么用-saga)
2. [作业票审批流程定义](#二作业票审批流程定义)
3. [Saga 状态机 JSON 定义](#三saga-状态机-json-定义)
4. [Java 实现示例](#四java-实现示例)
5. [补偿（回滚）逻辑](#五补偿回滚逻辑)
6. [与业务状态机的配合](#六与业务状态机的配合)
7. [异常处理与可观测](#七异常处理与可观测)

---

## 一、为什么用 Saga

特殊作业票审批是**跨多服务、长周期、可补偿**的业务：

- 提交作业票（`safety-svc` 本地事务）
- 联动查询气体检测实时值（`device-svc` + `monitor-svc`）
- 联动视频 AI 抽查（视频网关）
- 园区审核（人工，可能耗时数小时）
- 批准作业（状态变更）

若用 2PC（XA）会长时间锁资源、拖垮高频监测库。**Saga** 以"本地事务 + 前向/补偿"方式，保证最终一致且不长期持锁，符合化工园区"审批可驳回/可退回修改"的业务现实。

---

## 二、作业票审批流程定义

```
[Submit] ──→ [GasCheck] ──→ [VideoCheck] ──→ [Review] ──→ [Approve]
   │             │                │               │            │
   └─补偿:        └─补偿:           └─补偿:          └─补偿:       └─补偿:
   rollbackDraft  clearGasHold     clearVideoHold   rejectToDraft revertApprove
```

| 步骤 | 服务 | 本地事务 | 补偿动作 |
|------|------|----------|----------|
| Submit | safety-svc | 插入作业票 `SUBMITTED` | 置 `DRAFT` / 逻辑删除 |
| GasCheck | device/monitor | 记录气体检测快照、占用传感器查询会话 | 释放查询会话、标记快照失效 |
| VideoCheck | 视频网关 | 创建抽查任务、截图 | 取消任务、删除临时截图 |
| Review | safety-svc | 状态 `REVIEWED`/`REJECTED` | 回退 `SUBMITTED` |
| Approve | safety-svc | 状态 `APPROVED`，作业生效 | 回退 `REVIEWED`，作业失效 |

> 任一前向步骤失败，Saga 逆序执行已完成步骤的补偿，使整体回到一致态（作业票回到 `DRAFT` 或保留在失败前状态并标记 `ABNORMAL`）。

---

## 三、Saga 状态机 JSON 定义

Seata Saga 使用状态机 JSON（`seata-saga-statemachine-designer` 可可视化编辑）。

```json
{
  "Name": "hotWorkTicketApproval",
  "Comment": "特殊作业票(动火)审批 Saga",
  "StartState": "Submit",
  "Version": "1.0.0",
  "States": {
    "Submit": {
      "Type": "ServiceTask",
      "ServiceName": "safetyTicketService",
      "ServiceMethod": "submit",
      "CompensateState": "SubmitCompensate",
      "Next": "GasCheck",
      "Input": ["$.ticketId", "$.enterpriseId"]
    },
    "GasCheck": {
      "Type": "ServiceTask",
      "ServiceName": "gasCheckService",
      "ServiceMethod": "check",
      "CompensateState": "GasCheckCompensate",
      "Next": "VideoCheck",
      "Input": ["$.ticketId", "$.sensors"]
    },
    "VideoCheck": {
      "Type": "ServiceTask",
      "ServiceName": "videoCheckService",
      "ServiceMethod": "spotCheck",
      "CompensateState": "VideoCheckCompensate",
      "Next": "Review",
      "Input": ["$.ticketId", "$.cameraIds"]
    },
    "Review": {
      "Type": "ServiceTask",
      "ServiceName": "safetyTicketService",
      "ServiceMethod": "review",
      "CompensateState": "ReviewCompensate",
      "Next": "Approve",
      "Input": ["$.ticketId", "$.reviewerId"]
    },
    "Approve": {
      "Type": "ServiceTask",
      "ServiceName": "safetyTicketService",
      "ServiceMethod": "approve",
      "CompensateState": "ApproveCompensate",
      "Next": "Success",
      "Input": ["$.ticketId", "$.approverId"]
    },
    "SubmitCompensate": { "Type": "ServiceTask", "ServiceName": "safetyTicketService", "ServiceMethod": "compensateSubmit", "Next": "Fail" },
    "GasCheckCompensate": { "Type": "ServiceTask", "ServiceName": "gasCheckService", "ServiceMethod": "compensateCheck", "Next": "Fail" },
    "VideoCheckCompensate": { "Type": "ServiceTask", "ServiceName": "videoCheckService", "ServiceMethod": "compensateSpotCheck", "Next": "Fail" },
    "ReviewCompensate": { "Type": "ServiceTask", "ServiceName": "safetyTicketService", "ServiceMethod": "compensateReview", "Next": "Fail" },
    "ApproveCompensate": { "Type": "ServiceTask", "ServiceName": "safetyTicketService", "ServiceMethod": "compensateApprove", "Next": "Fail" },
    "Success": { "Type": "Succeed" },
    "Fail": { "Type": "Fail" }
  }
}
```

> 说明：`Input` 使用 JsonPath 从上下文取参；`CompensateState` 声明失败时逆序回滚入口。

---

## 四、Java 实现示例

### 4.1 配置（Seata Saga）

```yaml
# application.yml (safety-svc)
seata:
  enabled: true
  tx-service-group: safety_tx_group
  saga:
    state-machine-mode: true
  config:
    type: nacos
  registry:
    type: nacos
```

### 4.2 发起 Saga（safety-svc）

```java
@Service
@RequiredArgsConstructor
public class TicketApprovalAppService {

    private final StateMachineEngine stateMachineEngine;
    private final TicketRepository ticketRepository;

    public String startApproval(HotWorkTicketCmd cmd) {
        // 1. 先落本地草稿（独立本地事务，保证 ticketId 存在）
        Ticket ticket = ticketRepository.save(cmd.toDraft());

        // 2. 构造 Saga 上下文
        Map<String, Object> ctx = new HashMap<>();
        ctx.put("ticketId", ticket.getId());
        ctx.put("enterpriseId", cmd.getEnterpriseId());
        ctx.put("sensors", cmd.getGasSensorNos());   // 如 ["TANK-01-GT","TANK-02-GT"]
        ctx.put("cameraIds", cmd.getCameraIds());    // 如 ["cam_07"]
        ctx.put("reviewerId", cmd.getReviewerId());
        ctx.put("approverId", cmd.getApproverId());

        // 3. 启动 Saga（异步，状态变更由各服务本地事务提交）
        String businessKey = "TICKET_" + ticket.getId();
        stateMachineEngine.startWithBusinessKey("hotWorkTicketApproval",
                null, businessKey, ctx);
        return businessKey;
    }
}
```

### 4.3 各服务 Task 实现（示例：GasCheck）

```java
// gasCheckService 由 device-svc / monitor-svc 通过 Feign 暴露，Saga 引擎经注册中心发现
@Component("gasCheckService")
public class GasCheckService {

    @Autowired
    private DeviceClient deviceClient;     // Feign → device-svc
    @Autowired
    private MonitorClient monitorClient;   // Feign → monitor-svc
    @Autowired
    private GasSnapshotRepository repo;

    public boolean check(Map<String, Object> ctx) {
        String ticketId = (String) ctx.get("ticketId");
        List<String> sensors = (List<String>) ctx.get("sensors");

        // 取实时气体浓度（可燃/有毒），并做合格判定
        for (String s : sensors) {
            RealtimeVO vo = monitorClient.getRealtime(s);
            if (vo.getValue() > vo.getThreshold()) {
                throw new BizException("GAS_OVER_LIMIT", s + " 浓度超标，禁止作业");
            }
            repo.save(new GasSnapshot(ticketId, s, vo.getValue(), vo.getTs()));
        }
        return true;
    }

    public boolean compensateCheck(Map<String, Object> ctx) {
        // 释放查询会话、标记快照失效（幂等）
        repo.invalidateByTicket((String) ctx.get("ticketId"));
        return true;
    }
}
```

> `videoCheckService` / `safetyTicketService` 结构类似，分别对接视频网关与本地作业票库。

---

## 五、补偿（回滚）逻辑

补偿遵循三条原则：

1. **幂等**：补偿可因重试多次执行，须以业务键（ticketId）去重，不产生副作用。
2. **可逆**：前向动作必须有对应补偿；如"批准作业"补偿为"回退到 REVIEWED 且作业失效"。
3. **不破坏审计**：补偿仅改作业票**状态**与**占用资源释放**，所有操作仍入审计日志（WORM），保留完整链路。

```java
// safetyTicketService 补偿示例
@Component("safetyTicketService")
public class SafetyTicketService {

    public boolean compensateSubmit(Map<String, Object> ctx) {
        String id = (String) ctx.get("ticketId");
        Ticket t = repo.findById(id).orElse(null);
        if (t == null) return true;                  // 已删除，幂等返回
        if (t.getStatus() == TicketStatus.SUBMITTED) {
            t.setStatus(TicketStatus.DRAFT);         // 回到草稿
            repo.save(t);
        }
        return true;
    }

    public boolean compensateApprove(Map<String, Object> ctx) {
        String id = (String) ctx.get("ticketId");
        Ticket t = repo.findById(id).orElse(null);
        if (t != null && t.getStatus() == TicketStatus.APPROVED) {
            t.setStatus(TicketStatus.REVIEWED);      // 作业失效，回退审核态
            t.setEffective(false);
            repo.save(t);
        }
        return true;
    }
}
```

---

## 六、与业务状态机的配合

Saga 负责**跨服务最终一致**，业务状态机（`Spring Statemachine`）负责**单服务内状态合法跃迁**，两者职责分离：

```
Saga 步骤成功 → 调用 safety-svc 内部状态机执行 TRANSITION
                (如 SUBMITTED → REVIEWED，由状态机守卫校验)
```

- Saga 不感知业务细节，只触发"服务方法"；
- 状态机拦截非法跃迁（如 `DRAFT → APPROVED` 跳过审核），抛出 `40901`，Saga 触发补偿；
- 人工审核/批准由用户操作触发对应 Saga 步骤（流程可暂停等待人工，Seata Saga 支持 `Wait` 状态 + 唤醒）。

```java
// 审核步骤（人工唤醒）
public void review(String ticketId, String reviewerId, boolean pass) {
    // 状态机守卫：仅 SUBMITTED 可转 REVIEWED/REJECTED
    stateMachine.sendEvent(new ReviewEvent(ticketId, reviewerId, pass));
    // 通知 Saga 引擎继续/补偿
    sagaEngine.wakeUp(businessKeyOf(ticketId), Map.of("reviewPass", pass));
}
```

---

## 七、异常处理与可观测

| 场景 | 处理 |
|------|------|
| 前向步骤超时（如视频网关不可用） | Saga 标记 `ABNORMAL`，逆序补偿；作业票回到 `DRAFT` 并推送提醒 |
| 人工审核驳回 | 直接走 `ReviewCompensate` + `SubmitCompensate` 回到 `DRAFT`，企业可修改重报 |
| 补偿失败 | 进入 Saga 死信，告警运维；支持人工重试补偿 |
| 可观测 | Saga 实例状态、步骤耗时经 Prometheus 暴露；关键节点入 `audit.log` Topic |

**监控指标建议**：`saga_instances_total{state}`、`saga_step_duration`、`saga_compensate_total`，对接 Grafana 大屏。

---

*文档结束 · 版本 V1.0*
