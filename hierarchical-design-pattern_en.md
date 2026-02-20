---
layout: default
title: "Refactoring Complex Flows with a Hierarchical Design Pattern"
description: "Behavior trees, Pub/Sub, adapters, and observers for scalable device control and workflow orchestration."
date: 2025-02-19
---

# Refactoring Complex Flows with a Hierarchical Design Pattern: Behavior Trees, Pub/Sub, Adapters, and Observers

**How to make “device control + workflow orchestration” systems more extensible and maintainable using layered architecture and classic design patterns—without removing existing functionality.**

---

Complex hardware/software control systems often end up with flow logic, hardware control, and external API calls tangled together: changing one part affects everything, and new developers struggle to onboard. This post outlines a **practical refactoring plan based on a multi-level, heterogeneous design pattern**: use **behavior trees** to orchestrate high-level flow, **publish/subscribe** to decouple communication, **adapters** to unify hardware interfaces, and **observers** to handle results and errors; **strategy pattern** is optional for rule extension. Terminology is kept generic so readers with basic technical background can map the ideas to their own projects.

---

## 1. What Problem Are We Solving?

### Typical Starting Point

Assume the project has already had an initial refactor and has:

- A single entry point and clear layering: e.g. `flow/` (workflow), `hardware/` (devices), `control_api/` (backend control API);
- A “flow controller” that orchestrates one cycle, a state machine for flow states, and a “hardware provider” that wraps various devices;
- Failover switching, WebSocket for control commands, CI/CD, and tests in place.

So: **all features exist, but high-level flow, mid-level communication, and hardware interfaces can be abstracted one more layer** to make extension and maintenance easier.

### Goals of This Article

- **Keep existing behavior**: main flow, failover, error signaling, and deployment stay the same.
- **Introduce design patterns by layer**:
  - **System (high level)**: Use a **behavior tree (BT)** to replace or wrap “state machine + flow orchestration,” and handle flow control, decisions, and recovery (retry / failover / terminate) in one place.
  - **Mid-level communication**: Abstract WebSocket/MQTT events into **publish/subscribe (Pub/Sub)** so modules interact asynchronously by topic/event and are less tightly coupled.
  - **Hardware layer**: Define the existing “hardware provider” clearly as an **adapter**: different devices (e.g. vibration module, sensor module, future serial devices) expose a **unified interface**.
  - **Results and events**: When “result ready” or errors are detected, use **observers** to notify upper layers instead of calling “submit result” or send-error in many places.
- **Optional**: Use **strategy pattern** for different rules or cycle policies so that changing scoring or flow variants only swaps strategies, not the flow skeleton.

---

## 2. Pattern and Architecture Mapping

| Layer | Pattern | Before Refactor | After Refactor |
|-------|---------|-----------------|----------------|
| **System (high)** | Behavior Tree (BT) | Flow controller + state machine manually orchestrate one cycle (start → execute → detect → submit result → error handling). | Use **py_trees** (or similar) to define “main cycle loop” and recovery (retry → failover → terminate). State machine can remain as BT leaf nodes or subtrees to avoid state explosion at the top. |
| **Mid-level** | Publish/Subscribe | WebSocket callbacks call the flow controller directly; MQTT used inside hardware. | Introduce an **event bus** and **topics**: cycle start, control command, failover, result ready, error, etc. are **subscribed/published**. Flow controller (or BT) subscribes and reacts; decoupled from backend API and MQTT. |
| **Hardware** | Adapter | “Hardware provider” abstraction + one concrete impl wrapping several sub-modules. | Define a **MachineAdapter** interface. Main devices as **MachineAdapter A/B**; future serial devices as **SerialDeviceAdapter**, etc. Upper layer only sees `trigger()` / `is_ready()` / `get_result()`. |
| **Results/events** | Observer | On “detect complete,” code directly calls “submit result”; on error, directly calls send-error / failover. | **Result observer**: on detect complete, publish “result ready”; subscribers (flow or BT nodes) submit. **Error observer**: on error, publish “error”; subscribers send error signal and/or trigger failover. |
| **Optional** | Strategy | Rules and flow live inside state machine / flow controller. | Extract “cycle policy” (e.g. retry count, whether to continue after certain errors) or “scoring rule” into **CyclePolicy** / **ScoringStrategy**, injected by flow controller or BT. Change environment or rules by swapping strategy, not flow. |

**Hardware layer (manual mode)**  
If the system must support **operator-triggered actions** in addition to the automatic flow, implement an **InputAdapter** for the **desk-side control unit**: use a light HID input library to read the device and publish `OPERATOR_INPUT` on the event bus; the behavior tree or flow layer subscribes and drives manual flow.

### Layers and Event Flow (Concept)

- **Entry**: Single `main` assembles and starts the flow controller.
- **System (high)**: **Behavior tree**; flow controller ticks the BT; TreeBuilder builds the tree; BT nodes call adapters and publish/subscribe.
- **Mid-level**: **EventBus** as Pub/Sub hub; topics (cycle start, error, result ready, etc.) published by WebSocket/MQTT and subscribed by flow controller / BT nodes / observers.
- **Hardware**: **MachineAdapter** interface; each device is an adapter exposing `trigger()` / `is_ready()` / `get_result()`, using MQTT/Serial internally. **InputAdapter** (manual): only polls HID → publishes events; does not implement trigger/get_result; publishes `OPERATOR_INPUT` to EventBus for BT or subscribers to handle.
- **Observers**: e.g. “submit result” subscribes to “result ready” and calls backend API; “error recovery” subscribes to “error” and triggers error signal / failover.
- **Connections**: Solid lines = direct call/dependency; event flow = “publish → EventBus → subscriber.” Operator control device is just an input source, decoupled from the flow layer via EventBus.

---

## 3. Target Architecture (Conceptual Directory Layout)

After refactoring, the directory can look like this (names can be adjusted per project):

```
project_root/
├── main.py
├── flow/                          # Flow and behavior
│   ├── flow_controller.py         # Assembles and ticks BT; subscribes to mid-level events
│   ├── flow_state.py              # Optional: state definitions / leaf logic for BT
│   └── behavior/                  # Behavior tree
│       ├── tree_builder.py        # Builds main cycle BT (start → execute → detect → submit + recovery subtree)
│       ├── nodes/                 # Custom BT nodes (call adapter, submit result, trigger failover, etc.)
│       └── blackboard.py          # (Optional) BT shared data: cycle_id, result, error_type
├── middleware/                    # Mid-level Pub/Sub
│   ├── event_bus.py               # subscribe(topic, callback) / publish(topic, data)
│   └── topics.py                  # Topic constants: LAUNCH, CONTROL_COMMAND, SWITCH, RESULT_READY, ERROR, OPERATOR_INPUT
├── hardware/                      # Adapter layer
│   ├── base_provider.py           # MachineAdapter: trigger(), is_ready(), get_result()
│   ├── hardware_factory.py        # Returns adapter instances (main / serial / optional input)
│   ├── primary_provider.py        # Main MachineAdapter: wraps vibration + sensor, etc.
│   ├── serial_provider.py         # (Optional) Serial device adapter
│   └── input_adapter.py           # (Manual mode) InputAdapter: read desk HID, publish OPERATOR_INPUT
├── control_api/                   # Backend control API
│   ├── control_command_client.py
│   ├── switch_flow.py             # Failover
│   ├── pending_result_store.py
│   └── ...
└── requirements.txt               # Add py_trees if using it
```

**Dependency direction**:  
`main` → `flow_controller` → `behavior` (BT) → `middleware/event_bus`, `hardware/*` (adapters).  
Backend API is only called by BT nodes or event_bus subscribers; it does not depend on flow/behavior.

---

## 3.1 Operator Manual Mode and Input Adapter (Optional Reading)

If the system is currently **automatic** (backend API sends “cycle start,” devices run → detect → submit), and you want to support **operator-triggered actions** at the desk, add the **desk-side control unit** to the hardware layer and use a **lightweight HID input library** (e.g. one focused on USB gamepad/buttons) as the input source.

### Why a Light HID Library Instead of a Large Robot/ROS Stack?

- **Large meta-frameworks**: standardized, distributed, industrial-grade, but heavy for “read desk buttons only.”
- **Light HID library**: focused on HID devices, small, easy to integrate, good as a **single input-adapter node** alongside the existing behavior tree and EventBus.

So: **do not** make the input library the center of the system; use it only in the hardware layer as the implementation of an **input adapter**, coexisting with “behavior tree + EventBus.”

### Place in the Architecture

- **Layer**: Still **hardware · adapter**. The operator control unit is “input” hardware, distinct from “execution” devices (vibration/sensor). Thus:
  - **MachineAdapter**: trigger / is_ready / get_result (execution and detection devices).
  - **InputAdapter**: only **poll HID → publish events**; does not implement trigger/get_result; its contract to upper/mid layer is “publish `OPERATOR_INPUT` to EventBus.”
- **Wiring**: InputAdapter uses HID library to poll; on button press, `event_bus.publish(OPERATOR_INPUT, { button_id, action, ... })`; BT or flow controller subscribes to `OPERATOR_INPUT` and drives manual flow (e.g. operator “confirm result” → submit; “retry” → trigger retry). Like WebSocket/MQTT, operator input does not call the flow layer directly; it is decoupled via EventBus.

Manual vs automatic can be switched via `--mode auto | manual` or config; in manual mode, create and start InputAdapter, and nodes/BT subtrees that subscribe to `OPERATOR_INPUT` participate (e.g. WaitOperatorConfirm node instead of WaitLaunch).

---

## 4. Phased Refactoring (Conceptual Steps)

### Phase 0: Preparation and Dependencies

- Add behavior tree library (e.g. `py_trees`), create `middleware` and `behavior` directories; **do not change current flow behavior**.
- Implement a simple subscribe/publish in `event_bus`; define topic constants in `topics`.
- Done when: project runs, existing tests pass, only new dependencies and empty modules added.

### Phase 1: Mid-Level Pub/Sub and Event Wiring

- Have WebSocket and internal flow interact via “publish event” and “subscribe to event.”
- E.g. on “cycle start” / “control command” / “failover,” use `event_bus.publish(...)` instead of direct callbacks to the flow controller; flow controller subscribes to these topics and runs the same logic.
- Define and publish “result ready,” “error,” etc.; publish on submit complete or on error; subscribers use them for logging, counting, or future BT nodes.
- Done when: main flow is event-driven, behavior matches pre-refactor, and there is no “WebSocket callback directly calling flow controller” coupling.

### Phase 2: Hardware Layer as Adapter

- In base layer define **MachineAdapter**: at least `trigger()`, `is_ready()`, `get_result()`.
- Rename/document existing main hardware wrapper as a concrete Adapter (e.g. PrimaryMachineAdapter), noting “adapts multiple sub-modules to one interface.”
- Factory return type annotated as MachineAdapter for future device adapters.
- Done when: hardware layer naming and docs match adapter pattern, existing tests pass.

### Phase 3: Behavior Tree Replaces High-Level Flow

- Use BT library to define “one cycle main flow” and “recovery”; high-level decision logic lives in the BT.
- Implement custom nodes: e.g. WaitLaunchNode, ExecuteAndDetectNode (calls Adapter.trigger/get_result), SubmitResultNode, ErrorRecoveryNode (retry or publish ERROR to trigger failover).
- In tree_builder build main tree: Sequence/Selector, e.g. “WaitLaunch → ExecuteAndDetect → SubmitResult → (on failure, ErrorRecovery subtree)”; ErrorRecovery subtree can be “retry N times → if still failed, run_switch_sequence / request_terminate.”
- Change flow controller “run one cycle” to: on cycle-start event set blackboard, then **tick the behavior tree** until that cycle is SUCCESS/FAILURE; on FAILURE when failover is needed, ErrorRecovery node calls failover and terminate.
- Done when: one cycle is driven by the BT, recovery is expressed inside the BT, logic is clear and extensible.

### Phase 4: Observer Pattern for Results and Errors

- On detect complete use `event_bus.publish(RESULT_READY, result)` instead of returning directly; SubmitResultNode or a dedicated subscriber subscribes to RESULT_READY and calls backend API to submit.
- On error (hardware or BT node) use `event_bus.publish(ERROR, ...)`; subscribers to ERROR send error signal and trigger failover when appropriate.
- Done when: results and errors go through events; subscribers handle submit result / error signal / failover; behavior matches pre-refactor.

### Phase 5: (Optional) Strategy Pattern and Documentation

- If you have “cycle policy” or “scoring rules,” extract interfaces like CyclePolicy / ScoringStrategy, injected when building the flow controller or BT; default impl keeps current behavior.
- Update README / architecture doc: describe “high-level BT, mid-level Pub/Sub, hardware adapters, result/error observers,” and link to internal docs if needed.
- Done when: optional extension points are in place, docs match implementation, tests pass.

### Phase 6: (Optional) Operator Input Adapter (Manual Mode)

- Add `OPERATOR_INPUT` to topics; add HID input library to requirements.
- Add `input_adapter.py`: InputAdapter receives event_bus; polls HID and on button press calls `event_bus.publish(OPERATOR_INPUT, payload)`.
- In main or flow controller support `--mode manual`: create and start InputAdapter; BT or subscribers subscribe to OPERATOR_INPUT and drive manual flow by button_id.
- Done when: manual mode is optional, operator input is decoupled from flow via EventBus, dependencies stay light.

---

## 5. Pattern-to-Change Summary

| Pattern | Corresponding idea |
|---------|--------------------|
| **Behavior tree** | Flow controller ticks BT; tree_builder + nodes define main flow and recovery. |
| **Pub/Sub** | event_bus + topics; WebSocket/MQTT publish; flow controller/BT subscribe. |
| **Adapter** | MachineAdapter interface + concrete device adapters; upper layer sees trigger/is_ready/get_result. |
| **Adapter (input)** | InputAdapter: read desk HID, publish OPERATOR_INPUT to EventBus; BT or subscribers drive manual flow. |
| **Observer** | RESULT_READY / ERROR published by hardware or BT; SubmitResult / ErrorRecovery etc. subscribe and submit / signal / failover. |
| **Strategy (optional)** | CyclePolicy / ScoringStrategy interface and default impl, injected by flow controller or BT. |

---

## 6. Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| BT library and existing asyncio | Wrap synchronous tick in asyncio.run or run BT tick in an executor; or use an async-capable BT extension. |
| Event order and race conditions | Define topic and payload format clearly; use a single consumer or queue for ordering if needed; avoid heavy logic inside publish. |
| BT debugging | Use tree visualization or dot export; log SUCCESS/FAILURE/RUNNING at key nodes. |
| Over-abstracting | Implement event_bus as “minimum viable” first (single process, synchronous callbacks); confirm behavior, then consider cross-process or async. |

---

## 7. Summary and Takeaways

- **High level**: Use a behavior tree for “one cycle + recovery,” to avoid state explosion and scattered flow logic.
- **Mid level**: Use an event bus and topics for Pub/Sub so WebSocket, MQTT, and hardware callbacks become “publish event”; flow layer only subscribes and does not depend on concrete transport.
- **Hardware**: Use adapters (trigger / is_ready / get_result) so new devices mean new adapters; manual mode uses InputAdapter publishing OPERATOR_INPUT, decoupled from flow.
- **Results and errors**: Use observers (subscribe to RESULT_READY / ERROR) for “submit result” and “error signal / failover”—single responsibility, easy to test.
- **Optional**: Strategy pattern lets you swap “cycle policy” and “scoring rules” without changing the main flow skeleton.

This layering and pattern set fits “device control + flow orchestration + backend integration” systems. If you maintain a similar architecture, you can start with mid-level Pub/Sub and hardware adapters, then add the behavior tree and observers step by step—keeping existing behavior while leaving room for future extension.

---
