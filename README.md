# CrewAI DBOS durable agent

This repo demonstrates how to add durable execution support into CrewAI agents.
It integrates DBOS with `CrewAgentExecutor.invoke` and related methods to provide out-of-the-box durable execution and checkpointing.

This is based on PR: https://github.com/crewAIInc/crewAI/pull/3526


### Overview

* **File Structure:**
  * The `durable_agent/` folder contains all the relevant files.
    * `dbos_agent.py`: the main entrypoint for using DBOS agents.
    * `dbos_agent_executor.py`: executor for managing the agent main loop.
    * `dbos_llm.py`: wrapping llm calls as DBOS steps.
    * `dbos_util.py`: define `StepConfig` for configurable step retries.
  * Added a new `test_dbos_agent.py` test.

* **Workflows:**

  * `DBOSAgentExecutor.invoke` is automatically decorated a DBOS workflow.

* **Steps:**

  * LLM requests are automatically decorated as DBOS steps.
  * Outside DBOS, these remain ordinary functions.
  * Inside DBOS workflows, step outputs are automatically checkpointed in Postgres.

* **Tooling:**

  * Users may pass in DBOS-decorated functions (workflows, steps, transactions) as tools or event handlers.
  * The integration does *not* automatically wrap tools in DBOS decorators, so users retain full control. For example, they can pass in a workflow which will be invoked as a child workflow; or they can pass in a step.
  * Tools behave as normal functions when not used with DBOS.

### Example

To use the integration, users only need to add a few lines of DBOS code on top of their existing agent code. Here is the code from the test:

```python
@tool
@DBOS.step()  # Decorate this function as a DBOS step
def multiplier(first_number: int, second_number: int) -> float:
    """Useful for when you need to multiply two numbers together."""
    return first_number * second_number

# Agent declaration remains the same
orig_agent = Agent(
    role="test role",
    goal="test goal",
    backstory="test backstory",
    tools=[multiplier],
    allow_delegation=False,
)

# Wrap the original agent in a DBOS agent
dbos_agent = DBOSAgent(
    agent_name="test_agent_execution_with_tools",
    orig_agent=orig_agent,
)

task = Task(
    description="What is 3 times 4?",
    agent=dbos_agent,
    expected_output="The result of the multiplication.",
)
received_events = []

@crewai_event_bus.on(ToolUsageFinishedEvent)
@DBOS.step()  # Decorate this tool as a DBOS step
def handle_tool_end(source, event):
    received_events.append(event)

# Optionally set a workflow ID for tracking progress. If unspecified, a UUID will be generated as the ID.
with SetWorkflowID("test_execution"):
    # The main agent execution loop is automatically a DBOS workflow, and the LLM calls are DBOS steps. Tools that are annotated with DBOS.step() are also DBOS steps.
    output = dbos_agent.execute_task(task)
assert output == "The result of the multiplication is 12."
```
