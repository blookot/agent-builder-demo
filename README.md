# Elastic AI Agent Builder Demo

This repo will help you build a local demo of **Elastic AI Agent Builder** using a local Elastic stack and a local LLM (with multilingual support!), play with MCP clients and A2A (Agent-to-Agent). Ambitous, right!?

Note: this is for demo purpose only! This deployment is not secure, so do not use it in production or with confidential data!


## Table of contents

- [Intro](#elastic-ai-agent-builder-demo)
  - [Requirements](#requirements)
  - [References](#refs)
- [Setup](#setup-pre-requisites)
  - [Clone this repo](#clone-this-repo)
  - [Start a local Elastic stack](#start-a-local-elastic-stack)
  - [Setup a local LLM](#setup-a-local-llm)
  - [Create the LLM connector](#create-the-llm-connector)
- [Get data in](#get-data-in)
- [Configure the Agent Builder](#configure-the-agent-builder)
  - [Activate AB](#activate-the-agent-builder-tech-preview)
  - [Setup the tools](#setup-the-tools)
  - [Configure the agent](#configure-the-agent)
- [Chat time!](#chat-time)
  - [Using Kibana UI](#using-kibana-ui)
  - [Calling Kibana API](#calling-kibana-api)
  - [MCP with Claude Desktop](#using-a-first-mcp-client-claude-desktop)
  - [MCP with Cherry Studio](#using-a-second-mcp-client-cherry-studio)
  - [Agent to Agent (A2A)](#agent-to-agent-a2a)



## Requirements

This demo requires a strong Linux instance with (I would say) 12+ GB RAM. I personnaly run it on a MacBook Pro (48GB RAM)...

This demo has been tested on Elastic v9.2.1


## Refs

This demo was inspired by:

* Elastic start-local ([ref](https://github.com/elastic/start-local)) that will spawn Elastic+Kibana locally,
* A local LLM demo ([ref](https://github.com/fred-maussion/demo_local_ia_assistant)) based on ollama from my colleague Frédéric Maussion,
* The MCP server for Agent Builder [blog post](https://www.elastic.co/search-labs/blog/elastic-mcp-server-agent-builder-tools).

Other references are provided in each section.


## Setup pre-requisites

We will start a local Elasticsearch + Kibana stack and setup a local LLM with ollama.

### Clone this repo

Open your terminal and clone this repo first:

```sh
git clone https://github.com/blookot/agent-builder-demo
cd agent-builder-demo
```

### Start a local Elastic stack

We will first deploy an Elastic stack using start-local.
Run this in your terminal:

```sh
curl -fsSL https://elastic.co/start-local | sh -s -- -v 9.2.1
```

Do capture the output of the script, specially the elastic password that you will use to login to Kibana, and also the API key that you may use to also connect to Elasticsearch.
Let's record your API key (from the start-local output):

```sh
export ES_API_KEY="dE9NM3Rab0Jyd3dRa3FnRzJzZUg6TmdhdGJmRWZoMHJSekVKR3hleWdFUQ=="
```

### Setup a local LLM

Download & install [ollama](https://github.com/ollama/ollama) for your platform.

Get the model you want, as long as it has the 'tools' and 'thinking' tags (see the available models from the [ollama library](https://ollama.com/library?sort=popular)).

I personnaly used [qwen3](https://ollama.com/library/qwen3) that worked great, even in French! The default 8b model is about 5GB big to download.

I also tested llama3.2 and mistral that sometimes triggered errors. Be aware that, at this stage (v9.2), we recommend [specific models](https://www.elastic.co/docs/solutions/search/agent-builder/models#recommended-models), and the `Error executing agent: No tool calls found in the response.` I was getting with mistral illustrates [the issues you may face](https://www.elastic.co/docs/solutions/search/agent-builder/limitations-known-issues#incompatible-llms) with incompatible LLMs...

Besides, these models do not have the reasoning ability (see the 'thinking' tag in ollama models library!) so they struggle to leverage our tools.

Here is how to setup qwen3:

```sh
ollama pull qwen3
ollama create gpt-4o -f Modelfile-qwen3
```

Test your new LLM (setting the model to the model you chose of course, qwen3 in my example):

```sh
curl http://localhost:11434/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "qwen3",
        "messages": [
            {
                "role": "system",
                "content": "Tu es un assistant français qui donne des réponses concises."
            },
            {
                "role": "user",
                "content": "Bonjour ! Comment vas-tu ?"
            }
        ]
    }'
```

### Create the LLM connector

Connectors are handled by Kibana, which in our case, is executed inside a container and needs to interact with ollama, executed on the host. For Kibana to reach ollama, we will need to use the following host: `host.docker.internal`

Note: the connector is expecting an API Key configured to work. Ollama doesn't provide this feature so you can enter a random string and save the connector.

Go to Kibana > Stack Management > Connectors > Create connector > OpenAI

(Do not go for the "AI Connector"! Scroll down to select the "OpenAI" tile)

Configure the connector with this setup:

* Connector name: `Qwen3`
* OpenAI provider: `OpenAI`
* URL: `http://host.docker.internal:11434/v1/chat/completions`
* Default model: `qwen3`
* API key: `whatever` (not used, but mandatory)

Which should look like this:

![Qwen3 config](./img/qwen3-connector.png)

Click the 'Save & Test' button, and on the 'Test' tab, click the 'Run' button to validate the connector is well configured.

Then 'Close' at the buttom of the pane.

## Get data in!

Let's [open Kibana](http://localhost:5601/) and connect with the elastic account (and the password captured earlier).

We will first load a sample dataset of web logs.

When [opening Kibana](http://localhost:5601/), you will see a Search solutions view. We first need to switch to the classic view to load the dataset. To do so, click on the bottom left icon (gear), then down the menu to "Spaces", and edit the "Default" line to select the Solutions view "Classic". Finally, "Apply changes" at the bottom and update the space.

_Tip_: instead, you can open the dev tools and run:

```json
PUT kbn://api/spaces/space/default
{
    "id": "default",
    "name": "Default",
    "solution": "classic" 
}
```

And reload the page. Voilà ;-)

Now, click the Elastic logo at the top left of the screen, "Try sample data", expand "Other sample datasets" and click the "Add data" for the `Sample web logs` tile and the `Sample eCommerce orders` . Wait 5-10 seconds. Then "View data" and switch do "Discover" to explore this dataset.

## Configure the Agent Builder

We will create an agent to explore and analyze the content of these web logs.

### Activate the Agent Builder (tech preview)

As the AI Agent Builder is in tech preview in v9.2, we need to enable it first.

Go to the [Kibana advanced settings](http://localhost:5601/app/management/kibana/settings?query=agent), toggle the "Elastic Agent Builder" setting on to enable it, click "Save changed" in the bottom bar, then reload the page as suggested in the tooltip at the bottom right of the screen.

Now, go to the [Agents app](http://localhost:5601/app/agent_builder) (in the menu under Elasticsearch in the classic view).
The agents rely on Tools to run. So we will setup a couple of tools and then configure our agent.

_Tip_: I heard there was a secret Kibana API call to modify advanced settings. Anyone having it, please open an issue!

### Optional : Activate Workflow (tech preview)

Elastic has introduce workflow as part of the [keephq acquisition](https://www.elastic.co/blog/elastic-and-keep-join-forces) starting as tech preview as part of the >9.3 release.
To activate it you need to go to `dev tools` and enter the following command:

```sh
POST kbn://internal/kibana/settings
{
  "changes": {
    "workflows:ui:enabled": true
  }
}
```

And after a Kibana refresh, you should see the workflow menu appear on the left menu.

### Use case 1 : Logs Analyst

We are going to create an SRE Analyst agent which is dedicated to analyze the Web Logs we've ingested earlier where we provide dedicated tools that will enhance his capabilities.

#### Setup the tools

Here, we will setup 2 tools as an example: one to list the client IPs that generated the largest amount of logs, and one to analyze the requests targetting specific URLs that generated the biggest responses (volume wise).

##### List client IPs

Click "Tools" at the bottom left. You will see a predefined list of tools that let you list indices, search and retrieve docs. We will add our own tools.

Click "New tool" and enter:

* Tool ID: `logs-count-per-clientip`
* Description: `Count number of logs per client IP to find the biggest requesters`
* Labels: `logs`

In the ES|QL query editor, paste the following:

```sql
FROM kibana_sample_data_logs
| STATS count=COUNT(*) BY clientip
| KEEP clientip,count
| SORT count DESC
| LIMIT ?nb
```

Scroll down and click "Infer parameters". The variable `nb` is automatically filled. You can add the description `Number of client IPs to return` and change the type to `integer`.

Finally click "Save" at the bottom right of the page.

##### Analyze large responses

Same, click "New tool", enter:

* Tool ID: `logs-large-requests`
* Description: `List largest requests (in number of bytes returned) on a URL matching a specific word.`
* Labels: `logs`

In the ES|QL query editor, paste the following:

```sql
FROM kibana_sample_data_logs
| WHERE MATCH(request, ?request_keyword)
| SORT bytes DESC
| LIMIT ?nb
```

Scroll down and click "Infer parameters". The variables `request_keyword` and `nb` are automatically filled. You can add (respectively) the descriptions `Word that is searched in the request URL` and `Number of logs to retrieve`, then change the types to `keyword` and `integer`. See illustration below:

![ESQL params](./img/esql-params.png)

Finally click "Save" at the bottom right of the page.

##### Error logs

**TODO**

You can add a third tool to analyze errors. Here for inspiration:

```sql
FROM kibana_sample_data_logs
| WHERE TO_INTEGER(response)>=400
| STATS count=COUNT(*) BY geo.dest
| KEEP geo.dest,count
| SORT count DESC
| LIMIT ?nb
```

##### Analyze IP reputation & Create Case

_Analyze IP Reputation Workflow (tech preview)_

Note : For this you will to create an API Key on [AbsueIPDB](https://www.abuseipdb.com/) website and copy paste it in the yml provided below.

If you've activated workflow, you can create a tool that will be able to analyze the reputation of the IP and attach it to your agent. For this, go to `Workflow` -> `Create a new workflow` and paste the content [provided here](./logs-agent-workflow-1.yml)

And save it.

Now go back to agent, click "Tools" at the bottom left. You will see a predefined list of tools that let you list indices, search and retrieve docs. We will add our own tools.

Click "New tool" and enter:

* Type: `workflow`
* Workflow : `IP Reputation Check`
* Tool ID: `default.ip_reputation`
* Description: `This workflow helps to provide and IP reputation from AbuseIPDB. The input are the following :
  - name: ip_address
    type: string
    description:
      The IP address to check for malicious activity and threat intelligence
      (e.g., 8.8.8.8)
    required: true`
* Labels: `logs`

Finally click "Save" at the bottom right of the page.

_Create Case Workflow (tech preview)_

If you've activated workflow, you can create a tool that will be able to create a case and attach it to your agent. For this, go to `Workflow` -> `Create a new workflow` and paste the content [provided here](./logs-agent-workflow-2.yml)

And save it.

Now go back to agent, click "Tools" at the bottom left. You will see a predefined list of tools that let you list indices, search and retrieve docs. We will add our own tools.

Click "New tool" and enter:

* Type: `workflow`
* Workflow : `createCaseTool`
* Tool ID: `default.create_case`
* Description: `This workflow helps to create a case in elastic. The input are the following :
  - name: caseDescription
    required: true
    type: string
  - name: caseTitle
    required: true
    type: string`
* Labels: `logs`

Finally click "Save" at the bottom right of the page.

#### Configure the SRE Analyst agent

Now that we have our tools, we will configure our agent that will rely on these tools.

Go back to the [Agents app](http://localhost:5601/app/agent_builder), click "Agents" at the bottow left and click "New agent".

Enter:

* Agent ID: `logs-agent`
* Custom instructions: copy the instructions [provided here](./logs-agent-instructions.txt) to have it run in French
* Labels: `logs`
* Display name: something creative like `Logs investigator`
* Display description: `Investigating in the logs sample dataset.`

Then scroll back up and click on the second tab "Tools".

Here, you may leave the 4 pre-selected tools that will let the agent query Elasticsearch, and select the 2 tools or 4 tools (optional) we created. Finally, click "Save" in the bottom bar.

### Use case 2 : Business Analyst

We are going to create a Business Analyst agent which is dedicated to analyze the eCommerce orders to business audience using out of the box tools provided by Elastic.

#### Setup the tools

No Tools needed, elastic already provided the necessary tools :-)

_Send email Workflow (tech preview)_

If you've activated workflow, you can create a tool that will be able to send email and attach it to your agent. For this, go to `Workflow` -> `Create a new workflow` and paste the content [provided here](./business-agent-workflow-1.yml)

And save it.

Now go back to agent, click "Tools" at the bottom left. You will see a predefined list of tools that let you list indices, search and retrieve docs. We will add our own tools.

Click "New tool" and enter:

* Type: `workflow`
* Tool ID: `default.sent_email`
* Description: `This workflow helps to send an email. the input are the following, if no input for user_name take your_email@elastic.co
  - name: user_email
    type: string
    description: Email Recipient Arrary
  - name: subject
    type: string
    description: Subject of the message
  - name: body
    type: string
    description: Body of the message`
* Labels: `business`
* Configuration: `Send email`

Finally click "Save" at the bottom right of the page.

#### Configure the Business Analyst agent

Now that we have our tools, we will configure our agent that will rely on these tools.

Go back to the [Agents app](http://localhost:5601/app/agent_builder), click "Agents" at the bottow left and click "New agent".

Enter:

* Agent ID: `business_analytics`
* Custom instructions: copy the instructions [provided here](./business-agent-instructions.txt) to have it run in French
* Labels: `business`
* Display name: something creative like `Business Analyst`
* Display description: `Hello !
I'm here to help you understand and analyze the data from your eCommerce data. 
Ask me any type of business questions, I will answer them `

Then scroll back up and click on the second tab "Tools".

Here, in addition to the 4 pre-selected tools that will let the agent query Elasticsearch, select the following ootb tools provided by elastic 
* `platform.core.generate_esql` 
* `platform.core.execute_esql`  
* `platform.core.index_explorer`
* `platform.core.create_visualization`
* `default.sent_email` (optional)

Finally, click "Save" in the bottom bar.

## Chat time!

### Using Kibana UI

#### Use case 1 : SRE Analyst

When you hover the "Logs investigator" line, you can click the little "chat" icon to start a conversation. Alternatively, you may go back to [Agent Builder](http://localhost:5601/app/agent_builder), make sure the "Logs investigator" agent is selected and start a conversation.

I first suggest to try this question in French: `Peux tu lister les IP de mes 3 plus gros clients de mon site Web (qui ont fait le plus de requêtes) ?`

You should get something like this:

![First request on logs](./img/logs-req1.png)

Now let's try the second tool:
`Et peux tu me donner plus de détails sur les requêtes qui ont ciblé les pages 'apm' qui ont généré les réponses les plus volumineuses ?`

You should get something like this:

![Second request on logs](./img/logs-req2.png)

Finally ask the agent to bring additional IP context on the gathered information, resume and create the a case containing the information.
`Vérifie la reputation de ces IP et resume moi cette conversation dans un case afin de commencer une investigation`

![Third request on logs](./img/logs-req3.png)

#### Use case 2 : Business Analyst

When you hover the "Business Analyst" line, you can click the little "chat" icon to start a conversation. Alternatively, you may go back to [Agent Builder](http://localhost:5601/app/agent_builder), make sure the "Business Analyst" agent is selected and start a conversation.

I first suggest to try this question in French: `Quel est l'article le plus vendu au cours des six derniers mois en Europe ? ?`

You should get something like this:

![First request on logs](./img/business-1.png)

Now let's try a second search: `Quel est notre revenu annuel pour la France sur le secteur global feminin ?`

You should get something like this where a graph is available as we asked to generate visualization if this is relevant:

![Second request on logs](./img/business-2.png)

Now let's try a last search: `Donne moi le top 10 des articles masculin les plus vendus pour les pays suivant : France, Grande-Bretagne, États-Unis.`

You should get something like this:

![Second request on logs](./img/business-3.png)

(Optional) if you activated workflow, you can repeat the last search and send the summary per email : `Donne moi le top 10 des articles masculin les plus vendus pour les pays suivant : France, Grande-Bretagne, États-Unis. Résume notre conversation et envoi le résumé par email`

You should check out for your inbox right away ;-)

### Calling Kibana API

We will play with [Kibana API](https://www.elastic.co/docs/api/doc/kibana/group/endpoint-agent-builder) to be able to build up on top of our agent with an external chat (out of Kibana).

First, we can try to query our tools, with the following requests (here to be input in Kibana dev tools, but you have the equivalent curl commands).

List tools:

```sh
GET kbn://api/agent_builder/tools
```

Get the description of our first tool:

```sh
GET kbn://api/agent_builder/tools/logs-count-per-clientip
```

Execute this tool with the ES|QL parameter (number of client IPs to return):

```sh
POST kbn://api/agent_builder/tools/_execute
{
  "tool_id": "logs-count-per-clientip",
  "tool_params": {
    "nb": 3
  }
}
```

We see that this basically calls the ES|QL query with the parameters manually set.

Let's go one step further by querying agents!

List agents:

```sh
GET kbn://api/agent_builder/agents
```

Describe our agent:

```sh
GET kbn://api/agent_builder/agents/logs-agent
```

Finally, let's start a conversation!

```sh
POST kbn://api/agent_builder/converse
{
  "agent_id": "logs-agent",
  "input": "Peux tu lister les IP de mes 3 plus gros clients de mon site Web (qui ont fait le plus de requêtes) ?"
}
```

And you get your response!

![API response](./img/api-response.png)

_Note_: Keeping the "conversation_id" from the response will let you continue the conversation further!

_Note 2_: You see the two steps of "reasoning" (aka thinking tag of our qwen3 model) and "tool" call.

![API steps](./img/api-steps.png)

### Using a first MCP client: Claude desktop

Kibana ships with an integrated MCP server (see [doc](https://www.elastic.co/docs/solutions/search/agent-builder/mcp-server)) that exposes the tools we created earlier!

At the top right of the [Tools page](http://localhost:5601/app/agent_builder/tools), you will be able to copy the MCP server URL. If you used the default space, here it is: http://localhost:5601/api/agent_builder/mcp

Let's start with Claude Desktop.

Download it and install it on your computer. A free account will let you test a few requests using Sonnet LLM.

Next, open the settings, go to the "developer" settings. Click "Modify configuration", which will open the folder where the `claude_desktop_config.json` file is located. Edit this file and add the MCP server (with your own API key of course! If you forgot it you can still `echo $ES_API_KEY`):

```json
{
  "mcpServers": {
    "elastic-agent-builder": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "http://localhost:5601/api/agent_builder/mcp",
        "--header",
        "Authorization:ApiKey dE9NM3Rab0Jyd3dRa3FnRzJzZUg6TmdhdGJmRWZoMHJSekVKR3hleWdFUQ=="
      ]
    }
  }
}
```

_Note_: the `npx` only sometimes doesn't work. If you have issues, try adding the full path (check with a `whereis npx`). On my MacOS it is `/opt/homebrew/bin/npx`.

Save the file and close it. Possibly need to restart Claude.

Back to Claude settings, in the "Connectors" menu, you should see the "elastic-agent-builder" connector. Click on "Configure" to view the available tools, including the ones we created. Close the settings page.

Now open a new conversation and type the same question we asked earlier: `Peux tu lister les IP de mes 3 plus gros clients de mon site Web (qui ont fait le plus de requêtes) ?`

First time, you will be asked to approve the use of the tool. You may "Always approve", as illustrated below:

![Approve on Claude](./img/claude-approve.png)

You should get something like this, which is the same answer we got earlier in the Agent Builder chat app in Kibana:

![Claude desktop](./img/claude.png)

### Using a second MCP client: Cherry Studio

Because Claude desktop only integrates with Sonnet and is limited in its free version (and doesn't leverage our qwen3 LLM!!), let's try another MCP client: [Cherry Studio](https://www.cherry-ai.com/)!

First, [download](https://www.cherry-ai.com/download) and install Cherry Studio on your computer.

Next, open the settings (gear icon at the bottom left), go to "MCP" in the left menu. Click "Add" and "Quick create" and enter `MCP Elastic local` as Name, leave the `stdio` type, enter `/opt/homebrew/bin/npx` as command and the 4 following lines as arguments:

```sh
mcp-remote
http://localhost:5601/api/agent_builder/mcp
--header
Authorization:ApiKey dE9NM3Rab0Jyd3dRa3FnRzJzZUg6TmdhdGJmRWZoMHJSekVKR3hleWdFUQ==
```

You should have this MCP configuration:

![MCP config on Cherry Studio](./img/cherry-mcp.png)

Save and toggle on. If it works, nothing happens!

Then click "Model Provider" in the left menu, search "ollama", select it and configure whatever API key, leave `http://localhost:11434` as host, then click "Manage" and the + on the qwen3 group.

For the LLM configuration, you should have this:

![LLM config on Cherry Studio](./img/cherry-llm.png)

You may go back to conversations (the top left icon). In the bottom line of icon, locate the "@" icon and select your "Ollama" LLM. Left of it, click the hammer icon and select your "MCP Elastic local" server, as illustrated below:

![Chat config in Cherry Studio](./img/cherry-chat.png)

Now you're ready to go!

Type the same question we asked earlier: `Peux tu lister les IP de mes 3 plus gros clients de mon site Web (qui ont fait le plus de requêtes) ?` and you should get:

![Request in Cherry Studio](./img/cherry-req.png)

### Agent to agent (A2A)

The Agent-to-Agent (A2A) server (included in Agent Builder) enables external A2A clients to communicate with Elastic Agent Builder agents.

However, the [documentation](https://www.elastic.co/docs/solutions/search/agent-builder/a2a-server) and the API calls ([get A2A card](https://www.elastic.co/docs/api/doc/kibana/operation/operation-get-agent-builder-a2a-agentid-json) or [send A2A task](https://www.elastic.co/docs/api/doc/kibana/operation/operation-post-agent-builder-a2a-agentid)) won't help much :-/

I rather recommend to read our search labs ([post 1](https://www.elastic.co/search-labs/blog/a2a-protocol-elastic-agent-builder-gemini-enterprise) and [post 2](https://www.elastic.co/search-labs/blog/a2a-protocol-mcp-llm-agent-newsroom-elasticsearch)) that explain how A2A can be achieved. 

Let's first try to fetch the A2A agent card, via Kibana dev tools:

```sh
GET kbn://api/agent_builder/a2a/logs-agent.json
```

or via terminal:

```sh
curl -X GET -H "Authorization: ApiKey $ES_API_KEY" "http://localhost:5601/api/agent_builder/a2a/logs-agent.json" | jq
```

Where you should see the agent metadata (including the list of tools).

Then, you may try the [a2a-inspector](https://www.elastic.co/search-labs/blog/a2a-protocol-elastic-agent-builder-gemini-enterprise#test-your-agent-with-the-a2a-inspector):

```sh
git clone https://github.com/a2aproject/a2a-inspector
cd a2a-inspector
pip install uv
uv sync
cd frontend
npm install
cd ..
chmod +x scripts/run.sh
scripts/run.sh
```

And open [the a2a inspector UI](http://127.0.0.1:5001).

To configure the connection, fill the URL `http://localhost:5601/api/agent_builder/a2a/logs-agent.json` and set the Authentication as "API Key" with the Header Name `Authorization` and the API Key `ApiKey dE9NM3Rab0Jyd3dRa3FnRzJzZUg6TmdhdGJmRWZoMHJSekVKR3hleWdFUQ==` (with your API key of course), as illustrated below:

![A2A inspector configuration](./img/a2a-config.png)

When clicking "Connect", you should see the agent card:

![A2A agent card](./img/a2a-agent-card.png)

At the very bottom of the page, you can enter our traditional `Peux tu lister les IP de mes 3 plus gros clients de mon site Web (qui ont fait le plus de requêtes) ?` question and read our answer:

![A2A chat](./img/a2a-chat.png)

Finally, you may start using A2A in your own agent development with the [official SDKs](https://github.com/a2aproject/A2A)!


---

## Authors

* **Vincent Maury** - *Initial commit* - [blookot](https://github.com/blookot)

## License

This project is licensed under the Apache 2.0 License - see the [LICENSE.md](LICENSE.md) file for details
