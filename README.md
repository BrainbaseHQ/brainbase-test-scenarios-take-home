# Brainbase – Automated Testing
Brainbase is a powerful tool for provisioning complex AI agents. It uses a language we developed called Based to provisiton AI agents.

These agents are able to use both deterministic, Python-like code as well as fuzzy LLM logic together which makes them very powerful.

We want you to build a new version of our automated test suite that will allows our users to automatically test their Based flows across all possible scenarios.

## Specifications
The Automated Testing Suite will work as follows:

1. The user creates a Based flow
2. The Automated Testing suite takes this piece of Based code and analyzes all possible scenarios in a recursive fashion
3. Based on each identified potential path, the Testing Suite makes a simulated scenario with dummy information
4. Testing Suite connects to Brainbase's Engine and tests these scenarios in a back-and-forth text conversation
5. Testing Suite reports the transcripts and the outcomes to the user

## Stack

Brainbase has a frontend (client) and a backend (AI server).

Frontend: NextJS, React, TypeScript, Tailwind CSS, Shadcn
Backend: Python or TypeScript

### Frontend
The UI should look similar to the following screenshot from our main platform client:

<img width="1512" alt="image" src="https://github.com/user-attachments/assets/b76fea94-a16d-4ea9-b17b-f4687877d2f1" />

The testing suite will be included as a new tab on the left.

### Backend
Automated Testing's backend needs to be written in Python or TypeScript and must be an API server. This API will need to have endpoints for:

- `/based/node`: Endpoint for analyzing incoming Based code snippets and convert them to node format (more details below).
- `/based/paths`: Endpoint for creating all the possible paths the conversation can take recursively.
- `/based/scenario`: Endpoint for creating a simulated scenario given a path.
- `/based/test`: Endpoint for initializing a test given a scenario (note that running a conversation will be too long for a single API request, so this request would take the test request, add it to a queue and run it in a worker)
  
## Milestones
We expect you to approach this task in steps which have associated milestones:

### Milestone 1: Convert Based into nodes
Create a function that will take in a piece of Based code as `string` and output a `nodes` list.

A Based flow can only have the following node types:
- Based node: A multiline code node with no `if`s or `case`s
- Loop-until node: A node that runs a `talk` function
- Switch node: A node that has one edge coming in and N edges going out
- If node: A specialized switch node

A node should have the approximate format: 

```javascript
type Node = {
  id: string; // Unique identifier
  type: string; // e.g., "loop-until", "based", "if", "switch"
  name: string; // Display name

  inputs: Port[]; // Optional, for custom port behavior
  outputs: Port[];

  data: Record<string, any>; // Config data specific to the node type, e.g. based nodes would keep a `code` variable, loop-until nodes would keep a `prompt` variable
};

type Port = {
  id: string;
  name: string;
  type?: "input" | "output";
};

type Edge = {
  id: string;
  source: {
    nodeId: string;
    portId?: string;
  };
  target: {
    nodeId: string;
    portId?: string;
  };
  condition?: EdgeCondition;
};

type EdgeCondition =
  | {
      type: "ai";
      prompt: string; // natural language condition, e.g. "if user sounds angry"
    }
  | {
      type: "logic";
      expression: string; // Python-style, e.g. "input.value > 10"
    };
```



### Milestone 2: Agent that can generate Based diffs and apply
Modify your first agent to output not full Based code but unified diffs on the code written so far and then apply them. Should still be able to iterate. When the diff format is wrong, there should be a checker that notices the diff format is wrong and has the LLM redo it. It runs the Based code (python example:
https://github.com/BrainbaseHQ/guides/blob/main/hello_world/brainbase_runner.py). You will need to use the Braibase API/SDK (or our web client) to create a worker with the generated flow so that you can run it. You can get an API key here - https://beta.usebrainbase.com/dashboard/settings.

Exit criteria for milestone: Agent can write Based code diffs to modify the existing code that runs without errors.

### Milestone 3: Websocket agent
The agent from Milestone 2 can be connected via websocket. In each websocket session, keep the messages so far in the websocket session in an array (you can also keep the code so far here), instead of relying on the client to keep passing in past messages in it's API requests.

Exit criteria for milestone: Agent can now run on a websocket

### Final Milestone: Client that can connect to this websocket
The client should be able to connect to the websocket agent from Milestone 3 and send messages. These messages should be handled in the backend server, and then the results should be returned.

Exit criteria for milestone: Client that can interact with the agent over websocket

## Rules and Guidelines
- Using coding assistants such as ChatGPT, Claude, Cursor and other are absolutely allowed and strongly encouraged. If you can build this entire project through vibe coding we have no problem with it :)
- Getting 3/4 milestones completely is better than getting 75% on all four milestones. Please follow the progression of the flows.
- Keep your code clean so we can go through it. We know code hygiene is hard to maintain when you're shipping fast, but it's important that we understand what you did. You can use Cursor before each commit to automatically go and comment out your code.

## Based Guidelines (feel free to feed these entire pages into the AI)
- https://docs.usebrainbase.com/based-crash-course
- https://docs.usebrainbase.com/based-language-fundamentals
