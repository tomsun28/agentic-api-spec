<p align="center">
 English | <b><a href="README_CN.md">中文</a></b> 
</p>

## Agentic API Spec Design: When API Callers Change from Application to AI Agent

In current AI Agent development, we are used to feeding context to Agents using Skills, Tools, MCP, or Prompts.

As agent tools continue to evolve, the initial hype around MCP has cooled down. While it standardizes how APIs are called, it also makes things more complex. It requires clients that support its protocol, and the server needs a matching protocol implementation. If it's just wrapping a bunch of API endpoints, why not just write code to call those APIs directly? Skill is a better alternative for this. More importantly, it provides instruction manuals that models can easily understand.

For OpenClaw, besides its unified message gateway and cron job mechanism, its Skill capabilities and ClawHub are certainly big reasons for its popularity. The progressive disclosure of Skills helps improve the model's context. However, for complex systems, a huge `skill.md` file can still overload the model. Also, since it is a client-side solution, we have to manually install and maintain skills. Their versions must match the server APIs. If a backend endpoint slightly changes a path or a parameter, the Skill needs a full update. In the Agent's world, after it calls an API and gets data, it knows what it can do next—but it actually knew this way ahead of time, and it was forced to learn a lot of things it didn't need to know, rather than just learning what it needs right now.

Because of this, I wonder if we can do an even finer-grained progressive disclosure. It wouldn't have state sync issues, wouldn't need client installation, and would let the model know as little as possible while still getting the job done. Therefore, I designed the following API specification to see if it can work together with Skills on the server side, making agent planning and API calling smoother and smarter.

*Note: The discussion above is about service API-oriented Skills, ignoring local skills.*


## Agentic API Design

> This is an API design specification built natively for Agents. Its core logic is: don't make the Agent memorize the whole documentation; let the API speak for itself.


### 1. Core Response Structure

This requires all business API responses to include three core fields:

```json
{
  "data": {},    // Business data
  "error": {},   // Error control
  "relates": []  // Related actions (navigation info)
}
```

* **`data` represents the "current state"**: It returns the real business data, letting the Agent know the current status.
* **`error` provides a "feedback mechanism"**: By using stable error codes (like `TASK_LOCKED`), the Agent can run specific logic instead of guessing from vague error messages.
* **`relates` shows "future options"**: This is the key design. Instead of making the Agent guess "which API should I call after getting this ID?", the response directly tells it what "next step" API operations are available.

The full structure looks like this:

```json
{
  "data": object,
  "error": {
    "code": string,
    "message": string
  },
  "relates": [
    {
      "method": "GET|POST|PUT|DELETE",
      "path": "/path/to/resource",
      "desc": "A brief description of the API's intent and parameters",
      "schema": "typechat schema define = { param: string; }"
    }
  ]
}
```

---

### 2. Relates: Injecting "Documentation" into "Runtime"

Traditional APIs are isolated islands, but an Agent needs a map. The `relates` structure provides a complete action guide for other APIs related to the current one:

```json
{
  "data": { "id": "task_123", "title": "Fix bug" },
  "relates": [
    {
      "method": "GET",
      "path": "/tasks/{id}",
      "desc": "Retrieve task details",
      "schema": "type GetTask = { id: string };"
    },
    {
      "method": "PUT",
      "path": "/tasks/{id}",
      "desc": "Update task info",
      "schema": "interface UpdateTask { title?: string; priority?: 'low' | 'medium' | 'high'; completed?: boolean; }"
    },
    {
      "method": "DELETE",
      "path": "/tasks/{id}",
      "desc": "Delete task permanently",
      "schema": "type DeleteTask = { id: string };"
    }
  ]
}
```

### 3. API Discovery: The Entry Point to the Map

To solve the Agent's "cold start" problem, we need a unified entry point: `GET /api/llms.txt`.

This endpoint does not return business data. Instead, it returns the **top-level operation APIs** that the current user is allowed to use. This gives the Agent a full view of the system to plan its tasks (we also need to consider how much detail to expose here):

```
# Project Name

project description, usage guide, and more.

## Entry APIs

### POST /tasks
desc: Create a new task with specified priority  
schema: interface CreateTask { title: string; priority: 'low' | 'medium' | 'high'; description?: string; }


### GET /tasks
desc: List all tasks with pagination  
schema: interface ListTasks { page?: number; limit?: number; status?: 'todo' | 'done'; }

```

---

**Let's compare:**

* **Traditional approach**: The Agent reads a massive static document to find how to create a task.
* **This approach**: After the Agent requests the task list, the response directly includes the interface and parameters for "create task". The Agent just matches its goal with the `desc` (description) and makes the call.

This design essentially shifts the **API discovery logic from "static Prompts" to "dynamic responses"**.

### 4. Comparison

| Dimension | Traditional API | Skill Mode | Agentic API |
|-----------|----------------|------------|-------------|
| Awareness | Trial and error with prompt stuffing | Static awareness. Documentation tells all capabilities. | Dynamic awareness. Real-time capability delivery based on current data state. |
| Token Consumption | Repeated calls during trial and error | Linear growth with features, easily explodes. | Only one `llms.txt` endpoint, on-demand delivery afterwards. |
| Decoupling | Hard to use, no coupling | Backend code changes require prompt updates. | Backend code changes, Agent automatically adapts via `relates`. |

---

### 5. Summary

For Agents planning and calling server-side APIs, this specification does add some backend development complexity—we have to maintain the code that generates `relates`. However, considering a future where Agents are everywhere, this is an acceptable trade-off.
