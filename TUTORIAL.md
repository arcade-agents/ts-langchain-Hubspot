---
title: "Build a Hubspot agent with LangChain (TypeScript) and Arcade"
slug: "ts-langchain-Hubspot"
framework: "langchain-ts"
language: "typescript"
toolkits: ["Hubspot"]
tools: []
difficulty: "beginner"
generated_at: "2026-03-12T01:34:36Z"
source_template: "ts_langchain"
agent_repo: ""
tags:
  - "langchain"
  - "typescript"
  - "hubspot"
---

# Build a Hubspot agent with LangChain (TypeScript) and Arcade

In this tutorial you'll build an AI agent using [LangChain](https://js.langchain.com/) with [LangGraph](https://langchain-ai.github.io/langgraphjs/) in TypeScript and [Arcade](https://arcade.dev) that can interact with Hubspot tools — with built-in authorization and human-in-the-loop support.

## Prerequisites

- The [Bun](https://bun.com) runtime
- An [Arcade](https://arcade.dev) account and API key
- An OpenAI API key

## Project Setup

First, create a directory for this project, and install all the required dependencies:

````bash
mkdir hubspot-agent && cd hubspot-agent
bun install @arcadeai/arcadejs @langchain/langgraph @langchain/core langchain chalk
````

## Start the agent script

Create a `main.ts` script, and import all the packages and libraries. Imports from 
the `"./tools"` package may give errors in your IDE now, but don't worry about those
for now, you will write that helper package later.

````typescript
"use strict";
import { getTools, confirm, arcade } from "./tools";
import { createAgent } from "langchain";
import {
  Command,
  MemorySaver,
  type Interrupt,
} from "@langchain/langgraph";
import chalk from "chalk";
import * as readline from "node:readline/promises";
````

## Configuration

In `main.ts`, configure your agent's toolkits, system prompt, and model. Notice
how the system prompt tells the agent how to navigate different scenarios and
how to combine tool usage in specific ways. This prompt engineering is important
to build effective agents. In fact, the more agentic your application, the more
relevant the system prompt to truly make the agent useful and effective at
using the tools at its disposal.

````typescript
// configure your own values to customize your agent

// The Arcade User ID identifies who is authorizing each service.
const arcadeUserID = process.env.ARCADE_USER_ID;
if (!arcadeUserID) {
  throw new Error("Missing ARCADE_USER_ID. Add it to your .env file.");
}
// This determines which MCP server is providing the tools, you can customize this to make a Slack agent, or Notion agent, etc.
// all tools from each of these MCP servers will be retrieved from arcade
const toolkits=['Hubspot'];
// This determines isolated tools that will be
const isolatedTools=[];
// This determines the maximum number of tool definitions Arcade will return
const toolLimit = 100;
// This prompt defines the behavior of the agent.
const systemPrompt = "# HubSpot ReAct Agent \u2014 Prompt\n\n## Introduction\nYou are a ReAct-style AI agent that helps users interact with a HubSpot instance using a set of provided tools. Your job is to plan and execute multi-step CRM tasks (search, create, update, associate, log activities) by calling the appropriate HubSpot tools in sequence, interpreting tool results, and asking the user for clarification when needed.\n\nUse the ReAct pattern: interleave \"Thought\" (reasoning), \"Action\" (a tool call with parameters), and \"Observation\" (tool results). Finish with a concise natural-language response to the user. Do not fabricate data\u2014always surface tool observations and ask follow-ups when necessary.\n\n---\n\n## Instructions\n\n- ReAct message format:\n  - Thought: short internal reasoning about next step.\n  - Action: call a tool (name and JSON parameters).\n  - Observation: paste the tool output.\n  - Repeat until task complete.\n  - Final: a clear, user-facing summary and next steps or questions.\n\n- Always validate required fields before calling a tool. If required parameters are missing, ask the user for them before acting.\n\n- When an action can accept IDs or keywords:\n  - If the user gave an explicit ID, use it.\n  - If not, search first (GetContactDataByKeywords / GetCompanyDataByKeywords / GetDealDataByKeywords / corresponding activity search tools). When multiple matches exist, present choices to the user and ask which to use before proceeding.\n\n- Activity creation rules:\n  - Activities (call, email, meeting, note, communication) must be associated with at least one of: contact, company, or deal. Ensure you include the appropriate associate_to_* parameter.\n  - Include required fields (e.g., subject/when_occurred for emails, title/start_date/start_time for meetings).\n  - Use ISO timestamps when required: YYYY-MM-DDTHH:MM:SS for when_occurred; YYYY-MM-DD for dates, HH:MM or HH:MM:SS for times.\n\n- Deal creation/update rules:\n  - If creating a deal and you need a non-default pipeline/stage, call Hubspot_GetDealPipelines and/or Hubspot_GetDealPipelineStages first to get valid stage IDs.\n  - If pipeline_id is omitted, the default pipeline is used.\n  - Use Hubspot_UpdateDealStage for stage-only updates (preferred); use Hubspot_UpdateDealCloseDate for expected close date updates.\n\n- Company creation:\n  - If the user specifies an industry type, call Hubspot_GetAvailableIndustryTypes to validate and choose a valid value.\n\n- User and owner operations:\n  - Call Hubspot_WhoAmI early when you need context about the current user.\n  - Use Hubspot_GetAllUsers and Hubspot_GetUserById when changing or showing owners.\n\n- Pagination and limits:\n  - Many list/search tools accept limit, associations_limit, next_page_token. Use reasonable defaults (limit up to tool max) and ask the user if they want more pages.\n  - Set truncate_big_strings=true if large string properties should be summarized.\n\n- Error handling:\n  - If a tool returns an error, include the error in your Observation and propose next steps (retry with corrected parameters, search again, or ask user).\n  - Do not proceed with dependent steps if a required tool call failed.\n\n- Confirmation \u0026 destructive actions:\n  - For actions that may modify or overwrite important records (e.g., updates to deals/contacts/companies), confirm with the user if they did not explicitly authorize the change.\n\n- Respect data fidelity:\n  - Always expose IDs, timestamps, and relevant fields from tool responses back to the user.\n  - Never invent IDs, statuses, or associations.\n\n---\n\n## Workflows\n\nBelow are common workflows and the recommended tool sequences. For each workflow, follow the ReAct loop: Thought \u2192 Action (tool call) \u2192 Observation \u2192 Thought \u2192 Action \u2192 ...\n\n1) Onboard / Get current user context\n- Sequence:\n  1. Hubspot_WhoAmI\n  2. (Optional) Hubspot_GetAllUsers (if you need user list)\n\n- Example:\n```\nThought: Need user context to assign default owner.\nAction: Hubspot_WhoAmI\nObservation: { ...user info... }\nFinal: \"I am operating as {user.name} (owner id {user.id}).\"\n```\n\n2) Create a new company, contact, and deal (lead capture)\n- Sequence:\n  1. Hubspot_GetAvailableIndustryTypes (if user provided industry or you want to validate)\n  2. Hubspot_CreateCompany\n  3. Hubspot_CreateContact (company_id from step 2)\n  4. Hubspot_GetDealPipelines (if user requests a specific pipeline/stage)\n  5. Hubspot_CreateDeal (optionally set deal_owner and associate after)\n  6. Hubspot_AssociateContactToDeal (if the deal was not automatically associated)\n  7. Optionally log initial activity (Hubspot_CreateNoteActivity / Hubspot_CreateCallActivity / Hubspot_CreateEmailActivity) with associate_to_deal_id and/or associate_to_contact_id\n\n- Example action calls:\n```\nAction: Hubspot_GetAvailableIndustryTypes\nAction: Hubspot_CreateCompany\n{\n  \"company_name\": \"Acme Corp\",\n  \"web_domain\": \"acme.com\",\n  \"company_city\": \"Austin\",\n  \"company_country\": \"US\"\n}\nAction: Hubspot_CreateContact\n{\n  \"company_id\": 123,\n  \"first_name\": \"Jane\",\n  \"last_name\": \"Doe\",\n  \"email\": \"jane@acme.com\",\n  \"job_title\": \"VP Sales\"\n}\nAction: Hubspot_CreateDeal\n{\n  \"deal_name\": \"Acme - Initial Opportunity\",\n  \"deal_amount\": 15000,\n  \"pipeline_id\": \"default\"\n}\nAction: Hubspot_AssociateContactToDeal\n{\n  \"deal_id\": 456,\n  \"contact_id\": 789\n}\n```\n\n3) Log a call / email / meeting / note for an existing contact/company/deal\n- Sequence:\n  - If ID is known: call the appropriate create activity tool with associate_to_contact_id / associate_to_company_id / associate_to_deal_id.\n  - If ID unknown: Hubspot_GetContactDataByKeywords or Hubspot_GetCompanyDataByKeywords or Hubspot_GetDealDataByKeywords \u2192 present choices \u2192 then create activity.\n\n- Tool mapping:\n  - Call: Hubspot_CreateCallActivity\n  - Email: Hubspot_CreateEmailActivity\n  - Meeting: Hubspot_CreateMeetingActivity\n  - Note: Hubspot_CreateNoteActivity\n  - Other comms: Hubspot_CreateCommunicationActivity\n\n- Example:\n```\nAction: Hubspot_CreateCallActivity\n{\n  \"title\": \"Intro call with Jane\",\n  \"when_occurred\": \"2026-01-20T14:00:00\",\n  \"direction\": \"OUTBOUND\",\n  \"summary\": \"Discussed needs and next steps\",\n  \"duration\": 900,\n  \"associate_to_contact_id\": 789,\n  \"associate_to_deal_id\": 456\n}\n```\n\n4) Search for contacts/companies/deals/activities by keywords\n- Sequence:\n  - Contacts: Hubspot_GetContactDataByKeywords\n  - Companies: Hubspot_GetCompanyDataByKeywords\n  - Deals: Hubspot_GetDealDataByKeywords\n  - Activities: use corresponding Get*DataByKeywords (Call, Email, Meeting, Note, Communication, Task)\n\n- Guidance:\n  - Use associations_limit to fetch associated objects (contacts, deals) when needed.\n  - If results \u003e 1, present a short list with key fields (id, name, email, company, last modified) and ask user to pick one.\n\n5) Update a contact/company/deal when ID is unknown\n- Sequence:\n  1. Search by Hubspot_GetContactDataByKeywords / Hubspot_GetCompanyDataByKeywords / Hubspot_GetDealDataByKeywords.\n  2. If the user selects an item (or there is a single match), call Hubspot_UpdateContact / Hubspot_UpdateCompany / Hubspot_UpdateDeal with the ID and updated fields or, if you want to surface matches for confirmation, call with keywords first (tool supports keywords param).\n  3. For stage-only updates: Hubspot_UpdateDealStage (prefer this if only changing stage).\n  4. For expected close date changes: Hubspot_UpdateDealCloseDate (use this when specifically changing close date).\n\n- Example:\n```\nAction: Hubspot_GetContactDataByKeywords\n{ \"keywords\": \"jane@acme.com\", \"limit\": 5 }\nObservation: { ... }\nThought: User selected contact id 789.\nAction: Hubspot_UpdateContact\n{ \"contact_id\": 789, \"job_title\": \"Head of Partnerships\", \"phone\": \"+1-512-555-1212\" }\n```\n\n6) Update an activity when ID is unknown\n- Sequence:\n  - Use the appropriate Hubspot_Update*Activity tool with keywords to surface matches, then update by ID or pass updated fields.\n\n- Example:\n```\nAction: Hubspot_UpdateEmailActivity\n{ \"keywords\": \"Follow up on pricing\", \"matches_limit\": 5 }\nObservation: [list of matching email activities]\nThought: Select email_id 2345 to update.\nAction: Hubspot_UpdateEmailActivity\n{ \"email_id\": 2345, \"body_text\": \"Updated body: agreed to send quote by Fri.\" }\n```\n\n7) Change deal owner or reassign objects\n- Sequence:\n  1. Hubspot_GetAllUsers (find new owner by name/email)\n  2. Hubspot_UpdateDeal (set deal_owner)\n  3. Or Hubspot_UpdateContact/Hubspot_UpdateCompany for contact or company owner updates\n\n- Example:\n```\nAction: Hubspot_GetAllUsers\nObservation: { ... }\nAction: Hubspot_UpdateDeal\n{ \"deal_id\": 456, \"deal_owner\": \"987\" }\n```\n\n8) Associate an existing activity to a deal (or other object)\n- Sequence:\n  - Use Hubspot_AssociateActivityToDeal to associate an activity to a deal.\n  - Provide activity_type (e.g., \"EMAIL\", \"CALL\", \"MEETING\", \"NOTE\", \"COMMUNICATION\") and activity_id and deal_id.\n\n- Example:\n```\nAction: Hubspot_AssociateActivityToDeal\n{ \"activity_type\": \"NOTE\", \"activity_id\": 3210, \"deal_id\": 456 }\n```\n\n9) List and browse objects\n- Tools:\n  - Hubspot_ListContacts\n  - Hubspot_ListCompanies\n  - Hubspot_ListDeals\n\n- Guidance:\n  - Use limit, associations_limit, sort_order, next_page_token for paging and sorting. Ask user if they want more results.\n\n---\n\n## Practical examples of ReAct-style exchanges\n\nExample 1 \u2014 Create company + contact + deal:\n```\nThought: User wants to create a new lead with company \u0026 contact.\nAction: Hubspot_CreateCompany\n{\"company_name\":\"Acme Corp\",\"web_domain\":\"acme.com\",\"company_city\":\"Austin\",\"company_country\":\"US\"}\nObservation: {\"company_id\":123, ...}\nThought: Company created; now create contact associated with company.\nAction: Hubspot_CreateContact\n{\"company_id\":123,\"first_name\":\"Jane\",\"last_name\":\"Doe\",\"email\":\"jane@acme.com\",\"job_title\":\"VP Sales\"}\nObservation: {\"contact_id\":789, ...}\nThought: Create deal and associate contact.\nAction: Hubspot_CreateDeal\n{\"deal_name\":\"Acme - Initial Opportunity\",\"deal_amount\":15000}\nObservation: {\"deal_id\":456, ...}\nAction: Hubspot_AssociateContactToDeal\n{\"deal_id\":456,\"contact_id\":789}\nObservation: { ... association success ... }\nFinal: \"Created company Acme Corp (id 123), contact Jane Doe (id 789), and deal \u0027Acme - Initial Opportunity\u0027 (id 456). Contact associated to the deal. Do you want me to log an initial note or schedule a follow-up meeting?\"\n```\n\nExample 2 \u2014 Update deal stage (safe path):\n```\nThought: Need to change deal 456 to a new stage; validate stage id.\nAction: Hubspot_GetDealPipelines\n{\"search\":\"default\"}\nObservation: {... pipelines and stages ...}\nThought: Found stage id \u0027stage_789\u0027 for desired stage.\nAction: Hubspot_UpdateDealStage\n{\"deal_id\":456,\"deal_stage\":\"stage_789\"}\nObservation: {...}\nFinal: \"Deal 456 moved to stage \u0027Negotiation\u0027 (stage_789).\"\n```\n\n---\n\n## Additional guidance \u0026 best practices\n\n- Always ask clarifying questions if the user\u2019s request lacks required fields (e.g., timestamps, IDs, company name).\n- Use search tools when IDs are not provided; never assume which record to update when multiple matches exist.\n- When presenting search results, show key details (ID, name, email, company, last activity) so the user can pick.\n- Respect the tools\u2019 limits (max limits described in tool docs). Offer pagination if results are truncated.\n- Use the toolkit guidance (Hubspot_ToolkitEnviromentGuidance) if you need broader best practices about the tools.\n\n---\n\nIf you understand these rules, respond with a short acknowledgement thought and wait for the user\u0027s instruction.";
// This determines which LLM will be used inside the agent
const agentModel = process.env.OPENAI_MODEL;
if (!agentModel) {
  throw new Error("Missing OPENAI_MODEL. Add it to your .env file.");
}
// This allows LangChain to retain the context of the session
const threadID = "1";
````

Set the following environment variables in a `.env` file:

````bash
ARCADE_API_KEY=your-arcade-api-key
ARCADE_USER_ID=your-arcade-user-id
OPENAI_API_KEY=your-openai-api-key
OPENAI_MODEL=gpt-5-mini
````

## Implementing the `tools.ts` module

The `tools.ts` module fetches Arcade tool definitions and converts them to LangChain-compatible tools using Arcade's Zod schema conversion:

### Create the file and import the dependencies

Create a `tools.ts` file, and add import the following. These will allow you to build the helper functions needed to convert Arcade tool definitions into a format that LangChain can execute. Here, you also define which tools will require human-in-the-loop confirmation. This is very useful for tools that may have dangerous or undesired side-effects if the LLM hallucinates the values in the parameters. You will implement the helper functions to require human approval in this module.

````typescript
import { Arcade } from "@arcadeai/arcadejs";
import {
  type ToolExecuteFunctionFactoryInput,
  type ZodTool,
  executeZodTool,
  isAuthorizationRequiredError,
  toZod,
} from "@arcadeai/arcadejs/lib/index";
import { type ToolExecuteFunction } from "@arcadeai/arcadejs/lib/zod/types";
import { tool } from "langchain";
import {
  interrupt,
} from "@langchain/langgraph";
import readline from "node:readline/promises";

// This determines which tools require human in the loop approval to run
const TOOLS_WITH_APPROVAL = ['Hubspot_AssociateActivityToDeal', 'Hubspot_AssociateContactToDeal', 'Hubspot_CreateCallActivity', 'Hubspot_CreateCommunicationActivity', 'Hubspot_CreateCompany', 'Hubspot_CreateContact', 'Hubspot_CreateDeal', 'Hubspot_CreateEmailActivity', 'Hubspot_CreateMeetingActivity', 'Hubspot_CreateNoteActivity', 'Hubspot_UpdateCallActivity', 'Hubspot_UpdateCommunicationActivity', 'Hubspot_UpdateCompany', 'Hubspot_UpdateContact', 'Hubspot_UpdateDeal', 'Hubspot_UpdateDealCloseDate', 'Hubspot_UpdateDealStage', 'Hubspot_UpdateEmailActivity', 'Hubspot_UpdateMeetingActivity', 'Hubspot_UpdateNoteActivity'];
````

### Create a confirmation helper for human in the loop

The first helper that you will write is the `confirm` function, which asks a yes or no question to the user, and returns `true` if theuser replied with `"yes"` and `false` otherwise.

````typescript
// Prompt user for yes/no confirmation
export async function confirm(question: string, rl?: readline.Interface): Promise<boolean> {
  let shouldClose = false;
  let interface_ = rl;

  if (!interface_) {
      interface_ = readline.createInterface({
          input: process.stdin,
          output: process.stdout,
      });
      shouldClose = true;
  }

  const answer = await interface_.question(`${question} (y/n): `);

  if (shouldClose) {
      interface_.close();
  }

  return ["y", "yes"].includes(answer.trim().toLowerCase());
}
````

Tools that require authorization trigger a LangGraph interrupt, which pauses execution until the user completes authorization in their browser.

### Create the execution helper

This is a wrapper around the `executeZodTool` function. Before you execute the tool, however, there are two logical checks to be made:

1. First, if the tool the agent wants to invoke is included in the `TOOLS_WITH_APPROVAL` variable, human-in-the-loop is enforced by calling `interrupt` and passing the necessary data to call the `confirm` helper. LangChain will surface that `interrupt` to the agentic loop, and you will be required to "resolve" the interrupt later on. For now, you can assume that the reponse of the `interrupt` will have enough information to decide whether to execute the tool or not, depending on the human's reponse.
2. Second, if the tool was approved by the human, but it doesn't have the authorization of the integration to run, then you need to present an URL to the user so they can authorize the OAuth flow for this operation. For this, an execution is attempted, that may fail to run if the user is not authorized. When it fails, you interrupt the flow and send the authorization request for the harness to handle. If the user authorizes the tool, the harness will reply with an `{authorized: true}` object, and the system will retry the tool call without interrupting the flow.

````typescript
export function executeOrInterruptTool({
  zodToolSchema,
  toolDefinition,
  client,
  userId,
}: ToolExecuteFunctionFactoryInput): ToolExecuteFunction<any> {
  const { name: toolName } = zodToolSchema;

  return async (input: unknown) => {
    try {

      // If the tool is on the list that enforces human in the loop, we interrupt the flow and ask the user to authorize the tool

      if (TOOLS_WITH_APPROVAL.includes(toolName)) {
        const hitl_response = interrupt({
          authorization_required: false,
          hitl_required: true,
          tool_name: toolName,
          input: input,
        });

        if (!hitl_response.authorized) {
          // If the user didn't approve the tool call, we throw an error, which will be handled by LangChain
          throw new Error(
            `Human in the loop required for tool call ${toolName}, but user didn't approve.`
          );
        }
      }

      // Try to execute the tool
      const result = await executeZodTool({
        zodToolSchema,
        toolDefinition,
        client,
        userId,
      })(input);
      return result;
    } catch (error) {
      // If the tool requires authorization, we interrupt the flow and ask the user to authorize the tool
      if (error instanceof Error && isAuthorizationRequiredError(error)) {
        const response = await client.tools.authorize({
          tool_name: toolName,
          user_id: userId,
        });

        // We interrupt the flow here, and pass everything the handler needs to get the user's authorization
        const interrupt_response = interrupt({
          authorization_required: true,
          authorization_response: response,
          tool_name: toolName,
          url: response.url ?? "",
        });

        // If the user authorized the tool, we retry the tool call without interrupting the flow
        if (interrupt_response.authorized) {
          const result = await executeZodTool({
            zodToolSchema,
            toolDefinition,
            client,
            userId,
          })(input);
          return result;
        } else {
          // If the user didn't authorize the tool, we throw an error, which will be handled by LangChain
          throw new Error(
            `Authorization required for tool call ${toolName}, but user didn't authorize.`
          );
        }
      }
      throw error;
    }
  };
}
````

### Create the tool retrieval helper

The last helper function of this module is the `getTools` helper. This function will take the configurations you defined in the `main.ts` file, and retrieve all of the configured tool definitions from Arcade. Those definitions will then be converted to LangGraph `Function` tools, and will be returned in a format that LangChain can present to the LLM so it can use the tools and pass the arguments correctly. You will pass the `executeOrInterruptTool` helper you wrote in the previous section so all the bindings to the human-in-the-loop and auth handling are programmed when LancChain invokes a tool.


````typescript
// Initialize the Arcade client
export const arcade = new Arcade();

export type GetToolsProps = {
  arcade: Arcade;
  toolkits?: string[];
  tools?: string[];
  userId: string;
  limit?: number;
}


export async function getTools({
  arcade,
  toolkits = [],
  tools = [],
  userId,
  limit = 100,
}: GetToolsProps) {

  if (toolkits.length === 0 && tools.length === 0) {
      throw new Error("At least one tool or toolkit must be provided");
  }

  // Todo(Mateo): Add pagination support
  const from_toolkits = await Promise.all(toolkits.map(async (tkitName) => {
      const definitions = await arcade.tools.list({
          toolkit: tkitName,
          limit: limit
      });
      return definitions.items;
  }));

  const from_tools = await Promise.all(tools.map(async (toolName) => {
      return await arcade.tools.get(toolName);
  }));

  const all_tools = [...from_toolkits.flat(), ...from_tools];
  const unique_tools = Array.from(
      new Map(all_tools.map(tool => [tool.qualified_name, tool])).values()
  );

  const arcadeTools = toZod({
    tools: unique_tools,
    client: arcade,
    executeFactory: executeOrInterruptTool,
    userId: userId,
  });

  // Convert Arcade tools to LangGraph tools
  const langchainTools = arcadeTools.map(({ name, description, execute, parameters }) =>
    (tool as Function)(execute, {
      name,
      description,
      schema: parameters,
    })
  );

  return langchainTools;
}
````

## Building the Agent

Back on the `main.ts` file, you can now call the helper functions you wrote to build the agent.

### Retrieve the configured tools

Use the `getTools` helper you wrote to retrieve the tools from Arcade in LangChain format:

````typescript
const tools = await getTools({
  arcade,
  toolkits: toolkits,
  tools: isolatedTools,
  userId: arcadeUserID,
  limit: toolLimit,
});
````

### Write an interrupt handler

When LangChain is interrupted, it will emit an event in the stream that you will need to handle and resolve based on the user's behavior. For a human-in-the-loop interrupt, you will call the `confirm` helper you wrote earlier, and indicate to the harness whether the human approved the specific tool call or not. For an auth interrupt, you will present the OAuth URL to the user, and wait for them to finishe the OAuth dance before resolving the interrupt with `{authorized: true}` or `{authorized: false}` if an error occurred:

````typescript
async function handleInterrupt(
  interrupt: Interrupt,
  rl: readline.Interface
): Promise<{ authorized: boolean }> {
  const value = interrupt.value;
  const authorization_required = value.authorization_required;
  const hitl_required = value.hitl_required;
  if (authorization_required) {
    const tool_name = value.tool_name;
    const authorization_response = value.authorization_response;
    console.log("⚙️: Authorization required for tool call", tool_name);
    console.log(
      "⚙️: Please authorize in your browser",
      authorization_response.url
    );
    console.log("⚙️: Waiting for you to complete authorization...");
    try {
      await arcade.auth.waitForCompletion(authorization_response.id);
      console.log("⚙️: Authorization granted. Resuming execution...");
      return { authorized: true };
    } catch (error) {
      console.error("⚙️: Error waiting for authorization to complete:", error);
      return { authorized: false };
    }
  } else if (hitl_required) {
    console.log("⚙️: Human in the loop required for tool call", value.tool_name);
    console.log("⚙️: Please approve the tool call", value.input);
    const approved = await confirm("Do you approve this tool call?", rl);
    return { authorized: approved };
  }
  return { authorized: false };
}
````

### Create an Agent instance

Here you create the agent using the `createAgent` function. You pass the system prompt, the model, the tools, and the checkpointer. When the agent runs, it will automatically use the helper function you wrote earlier to handle tool calls and authorization requests.

````typescript
const agent = createAgent({
  systemPrompt: systemPrompt,
  model: agentModel,
  tools: tools,
  checkpointer: new MemorySaver(),
});
````

### Write the invoke helper

This last helper function handles the streaming of the agent’s response, and captures the interrupts. When the system detects an interrupt, it adds the interrupt to the `interrupts` array, and the flow interrupts. If there are no interrupts, it will just stream the agent’s to your console.

````typescript
async function streamAgent(
  agent: any,
  input: any,
  config: any
): Promise<Interrupt[]> {
  const stream = await agent.stream(input, {
    ...config,
    streamMode: "updates",
  });
  const interrupts: Interrupt[] = [];

  for await (const chunk of stream) {
    if (chunk.__interrupt__) {
      interrupts.push(...(chunk.__interrupt__ as Interrupt[]));
      continue;
    }
    for (const update of Object.values(chunk)) {
      for (const msg of (update as any)?.messages ?? []) {
        console.log("🤖: ", msg.toFormattedString());
      }
    }
  }

  return interrupts;
}
````

### Write the main function

Finally, write the main function that will call the agent and handle the user input.

Here the `config` object configures the `thread_id`, which tells the agent to store the state of the conversation into that specific thread. Like any typical agent loop, you:

1. Capture the user input
2. Stream the agent's response
3. Handle any authorization interrupts
4. Resume the agent after authorization
5. Handle any errors
6. Exit the loop if the user wants to quit

````typescript
async function main() {
  const config = { configurable: { thread_id: threadID } };
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });

  console.log(chalk.green("Welcome to the chatbot! Type 'exit' to quit."));
  while (true) {
    const input = await rl.question("> ");
    if (input.toLowerCase() === "exit") {
      break;
    }
    rl.pause();

    try {
      let agentInput: any = {
        messages: [{ role: "user", content: input }],
      };

      // Loop until no more interrupts
      while (true) {
        const interrupts = await streamAgent(agent, agentInput, config);

        if (interrupts.length === 0) {
          break; // No more interrupts, we're done
        }

        // Handle all interrupts
        const decisions: any[] = [];
        for (const interrupt of interrupts) {
          decisions.push(await handleInterrupt(interrupt, rl));
        }

        // Resume with decisions, then loop to check for more interrupts
        // Pass single decision directly, or array for multiple interrupts
        agentInput = new Command({ resume: decisions.length === 1 ? decisions[0] : decisions });
      }
    } catch (error) {
      console.error(error);
    }

    rl.resume();
  }
  console.log(chalk.red("👋 Bye..."));
  process.exit(0);
}

// Run the main function
main().catch((err) => console.error(err));
````

## Running the Agent

### Run the agent

```bash
bun run main.ts
```

You should see the agent responding to your prompts like any model, as well as handling any tool calls and authorization requests.

## Next Steps

- Clone the [repository](https://github.com/arcade-agents/ts-langchain-Hubspot) and run it
- Add more toolkits to the `toolkits` array to expand capabilities
- Customize the `systemPrompt` to specialize the agent's behavior
- Explore the [Arcade documentation](https://docs.arcade.dev) for available toolkits

