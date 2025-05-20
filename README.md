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
- End node: A node that indicates the conversation is over
- Jump node: A node that indicates a jump to a different node

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

You can modify these definitions if you prefer.

Example:

Let's say we have the Based code:

```python
say("Hello how are you today?")
user_info = <api request>
loop:
  res = talk("Talk to the user about their day.", True)
until "user says they're bored":
  say("I'm sorry to hear that")

  if user_info["married"]:
    loop:
      res_counsel = talk("Recommend the user talk to their spouse", True)
    until "user says okay":
      <api post request to update user state>
      say("Amazing, talk to you later!")
      end()
  
until "user says they're excited about their day":
  <api post request to update user state>
  say("That's awesome, let's keep talking then!")
  return "Keep talking to the user"  # this goes back to the high level loop-until node
```

This should give the following nodes and edges:

```yaml
nodes:
  - id: start
    type: based
    name: Start
    inputs: []
    outputs:
      - id: out_start
        name: next
    data:
      code: |
        say("Hello how are you today?")
        user_info = <api request>

  - id: loop_1
    type: loop-until
    name: Talk about their day
    inputs:
      - id: in_loop_1
        name: input
    outputs:
      - id: out_bored
        name: "user says they're bored"
      - id: out_excited
        name: "user says they're excited about their day"
    data:
      prompt: "Talk to the user about their day."
      until: 
        - "user says they're bored"
        - "user says they're excited about their day"

  - id: bored_reaction
    type: based
    name: User is bored
    inputs:
      - id: in_bored_reaction
        name: input
    outputs:
      - id: out_if_married
        name: check marriage
    data:
      code: say("I'm sorry to hear that")

  - id: if_married
    type: if
    name: Check if user is married
    inputs:
      - id: in_if_married
        name: input
    outputs:
      - id: out_married
        name: "True"
      - id: out_not_married
        name: "False"
    data:
      condition: "user_info['married']"

  - id: talk_to_spouse
    type: loop-until
    name: Suggest spouse talk
    inputs:
      - id: in_talk_to_spouse
        name: input
    outputs:
      - id: out_user_says_okay
        name: "user says okay"
    data:
      prompt: "Recommend the user talk to their spouse"
      until:
        - "user says okay"

  - id: married_action
    type: based
    name: Married - Final Action
    inputs:
      - id: in_married_action
        name: input
    outputs:
      - id: out_married_end
        name: next
    data:
      code: |
        <api post request to update user state>
        say("Amazing, talk to you later!")

  - id: married_end
    type: end
    name: End (Married Path)
    inputs:
      - id: in_married_end
        name: input
    outputs: []
    data: {}

  - id: excited_action
    type: based
    name: Excited - Keep Talking
    inputs:
      - id: in_excited_action
        name: input
    outputs:
      - id: out_jump_back_to_loop
        name: next
    data:
      code: |
        <api post request to update user state>
        say("That's awesome, let's keep talking then!")
        return "Keep talking to the user"

  - id: jump_back_to_loop
    type: jump
    name: Jump back to outer loop
    inputs:
      - id: in_jump_back
        name: input
    outputs:
      - id: out_jump_loop
        name: jump
    data:
      targetNodeId: loop_1

edges:
  - id: edge_start_to_loop
    source:
      nodeId: start
      portId: out_start
    target:
      nodeId: loop_1
      portId: in_loop_1

  - id: edge_loop_to_bored
    source:
      nodeId: loop_1
      portId: out_bored
    target:
      nodeId: bored_reaction
      portId: in_bored_reaction
    condition:
      type: ai
      prompt: "user says they're bored"

  - id: edge_bored_to_if
    source:
      nodeId: bored_reaction
      portId: out_if_married
    target:
      nodeId: if_married
      portId: in_if_married

  - id: edge_if_true
    source:
      nodeId: if_married
      portId: out_married
    target:
      nodeId: talk_to_spouse
      portId: in_talk_to_spouse
    condition:
      type: logic
      expression: "user_info['married'] == True"

  - id: edge_talk_to_spouse_to_action
    source:
      nodeId: talk_to_spouse
      portId: out_user_says_okay
    target:
      nodeId: married_action
      portId: in_married_action
    condition:
      type: ai
      prompt: "user says okay"

  - id: edge_married_action_to_end
    source:
      nodeId: married_action
      portId: out_married_end
    target:
      nodeId: married_end
      portId: in_married_end

  - id: edge_loop_to_excited
    source:
      nodeId: loop_1
      portId: out_excited
    target:
      nodeId: excited_action
      portId: in_excited_action
    condition:
      type: ai
      prompt: "user says they're excited about their day"

  - id: edge_excited_action_to_jump
    source:
      nodeId: excited_action
      portId: out_jump_back_to_loop
    target:
      nodeId: jump_back_to_loop
      portId: in_jump_back

  - id: edge_jump_back_to_loop
    source:
      nodeId: jump_back_to_loop
      portId: out_jump_loop
    target:
      nodeId: loop_1
      portId: in_loop_1
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
