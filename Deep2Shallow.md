Role & Objective:
You are an expert Senior AI Systems Engineer specializing in LangChain, LangGraph, and Deep Agent architectures. 
We need to refactor our existing Deep Agent application. Currently, all sub-agents were incorrectly set up as recursive deep agents, causing recursion depth errors (>25). 

We need to review our existing flow and convert each component into its correct design pattern (Compiled ReAct State Graph vs. Simple Tool Call) as outlined in the table below.

---

### Target Component Architecture Table

| Component Name | Implementation Type | Technical & Functional Specifications |
| :--- | :--- | :--- |
| **1. InfoMax Search**<br>(Vector DB / Multi-Channel) | **Compiled State Graph**<br>*(Strict max_depth = 1)* | - It queries multiple channels in parallel within a single tool execution pass.<br>- Formats the response into a clean, well-structured string.<br>- Must **not** trigger secondary deep planning loops or recursive child calls. |
| **2. Client Search**<br>(Disambiguation Handler) | **Compiled State Graph**<br>(or ReAct Graph) | - Handles queries containing client name, account number, or household ID.<br>- Fetches the client/household data and saves the resolved household context into the graph state.<br>- **Ambiguity Rule:** If a search yields multiple matching clients, households, or account numbers, it must return an interactive choice/disambiguation payload for the user instead of failing. |
| **3. Client Account / Account 360**<br>(Multi-API Portfolio Engine) | **Compiled State Graph**<br>(or ReAct Graph) | - Executes multiple sequential or parallel API calls behind the scenes based on the resolved household state.<br>- Calculates portfolio details: account summaries, stock investments, maximum amounts, current values, and loss metrics. |
| **4. CRM Notes**<br>(Topmost Notes Lookup) | **Simple Tool Call**<br>*(Atomic Function)* | - A single, direct API call requiring only `first_name` and `last_name` to fetch the topmost/latest CRM notes.<br>- Does **not** require a sub-agent or graph wrapper; registered directly as a tool in the supervisor/lead agent. |

---

### Implementation Instructions for the Assistant:
1. **Inspect Existing Flow:** Review our current implementation where all sub-agents are incorrectly acting as recursive deep agents.
2. **Refactor Sub-Agents:** Convert InfoMax Search, Client Search, and Account 360 into standalone **Compiled State Graphs** (`CompiledStateGraph` via `create_react_agent` or custom state graphs) with bounded execution depths.
3. **Convert CRM Notes:** Strip the deep agent wrapper from the CRM notes lookup and convert it into a standard, clean Python tool function that can be bound directly to the supervisor.
4. **Maintain State & Telemetry:** Ensure state persistence (like storing the resolved household ID in state) works seamlessly across handoffs, and ensure all changes preserve our existing **Arize Phoenix / OTel** telemetry tracing.
5. **Provide Output:** Show the updated code structure, focusing on how the Supervisor delegates to these compiled graphs versus invoking the simple CRM tool.
