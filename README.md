# Elastic AI Agent Builder Demo

This repo will help you build a local demo of semantic search with RAG (retrieval augmented generation) using the Elastic stack and Streamlit.<br/>
For the content, we will crawl the Elastic Labs blog posts to have some data to pass as context to the LLM.

This generative AI demo has been adapted from Jeff's excellent blog posts (see refs below) with a few changes:
* I wanted to run everything locally, so the Elastic stack and the LLM are local,
* I wanted to have multilingual support, so the demo uses the e5.small embedding model (instead of Elastic's default ELSER) to support multilingual search. The examples will be in French.

Note: this is for demo purpose only! This deployment is not secure, so do not use it in production or with confidential data!


## Requirements

This demo requires a strong Linux instance with (I would say) 12+ GB RAM. I personnaly run it on a MacBook Pro (48GB RAM)...<br/>
This demo has been tested on Elastic v9.2.1


## Refs

Inspiré de https://www.elastic.co/search-labs/blog/semantic-search-open-crawler

This demo uses:
* Elastic start-local ([ref](https://github.com/elastic/start-local)) that we will modify to add an enterprise search node,
* The first RAG demo ([blog post](https://www.elastic.co/search-labs/blog/chatgpt-elasticsearch-openai-meets-private-data) and [github repo](https://github.com/jeffvestal/ElasticDocs_GPT)) and the second updated one ([blog post](https://www.elastic.co/search-labs/blog/chatgpt-elasticsearch-rag-enhancements) and [github repo](https://github.com/jeffvestal/rag-really-tied-the-app-together)) from my colleague Jeff Vestal,
* A local LLM demo ([ref](https://github.com/fred-maussion/demo_local_ia_assistant)) based on ollama from my colleague Frédéric Maussion.


# Setup pre-requisites

We will start a local Elasticsearch + Kibana stack and setup a local LLM with ollama.

## Clone this repo

Open your terminal and clone this repo first:
```sh
git clone https://github.com/blookot/agent-builder-demo
cd agent-builder-demo
```

## Start a local Elastic stack

We will first deploy an Elastic stack using start-local.
Run this in your terminal:
```sh
curl -fsSL https://elastic.co/start-local | sh -s -- -v 9.2.1
```

Do capture the output of the script, specially the elastic password that you will use to login to Kibana.<br/>
Also record the API key that the crawler will use to write docs (without changing the ES_HOST):
```sh
export ES_HOST="http://host.docker.internal"
export ES_PORT="9200"
export ES_API_KEY="dE9NM3Rab0JoMHJSekVKR3hleWdFUQ=="
```


## Setup a local LLM

Download & install [ollama](https://github.com/ollama/ollama) for your platform.<br/>
Get the model you want, as long as it has the 'tools' tag (see the available models from the [ollama library](https://ollama.com/library?sort=popular)).

I personnaly used [llama3.2](https://ollama.com/library/llama3.2) that worked great, even in French!<br/>
I also tested qwen3 and mistral that sometimes triggered errors. Be aware that, at this stage (v9.2), we recommend [specific models](https://www.elastic.co/docs/solutions/search/agent-builder/models#recommended-models) and the `` I was getting with mistral illustrate [the issues you may](https://www.elastic.co/docs/solutions/search/agent-builder/limitations-known-issues#incompatible-llms) face with incompatible LLMs...

Here is how to setup llama3.2:
```sh
ollama pull llama3.2
ollama create gpt-4o -f Modelfile-llama3.2
```

Test your new LLM (setting the model to the model you chose of course, mistral in my example):
```sh
curl http://localhost:11434/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "llama3.2",
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


# Get data in!

Let's [open Kibana](http://localhost:5601/) and connect with the elastic account (and the password captured earlier), then create our inference endpoint, the web crawler and the LLM connector to play.

## Sample dataset

We will first load a sample dataset of web logs.<br/>
When [opening Kibana](http://localhost:5601/), you will see a Search solutions view. We first need to switch to the classic view to load the dataset. To do so, click on the bottom left icon (gear), then down the menu to "Spaces", and edit the "Default" line to select the Solutions view "Classic". Finally, "Apply changes" at the bottom and update the space.

_Tip_: instead, you could have opened the dev tools and run:
```json
PUT kbn://api/spaces/space/default
{
    "id": "default",
    "name": "Default",
    "solution": "classic" 
}
```
And reloaded the page.

Now, click the Elastic logo at the top left of the screen, "Try sample data", expand "Other sample datasets" and click the "Add data" for the "Sample web logs" tile. Wait 5-10 seconds. Then "View data" and switch do "Discover" to explore this dataset.


## Create the LLM connector

Connectors are handled by Kibana, which in our case, is executed inside a container and needs to interact with ollama, executed on the host. For Kibana to reach ollama, we will need to use the following host: `host.docker.internal`<br/>
Note: the connector is expecting an API Key configured to work. Ollama doesn't provide this feature so you can enter a random string and save the connector.

Go to Kibana > Stack Management > Connectors > Create connector > OpenAI<br/>
(Do not go for the "AI Connector"! Scroll down to select the "OpenAI" tile)

Configuring the connector for Llama3.2 will look like this:

<p align="center">
<img src="https://github.com/blookot/agent-builder-demo/blob/main/img/llama-connector.png" width="80%" alt="Llama 3.2 config"/>
</p>

with this setup: 
* Connector name: `Llama 3.2`
* OpenAI provider: `OpenAI`
* URL: `http://host.docker.internal:11434/v1/chat/completions`
* Default model: `llama3.2`
* API key: `whatever` (not used, but mandatory)

Click the 'Save & Test' button, and on the 'Test' tab, click the 'Run' button to validate the connector is well configured.<br/>
Then 'Close' at the buttom of the pane.


# Configure the Agent Builder

We will create 

## Activate the Agent Builder (tech preview)

As the AI Agent Builder is in tech preview in v9.2, we need to enable it first.

Go to the [Kibana advanced settings](http://localhost:5601/app/management/kibana/settings?query=agent), toggle the "Elastic Agent Builder" setting on to enable it, click "Save changed" in the bottom bar, then reload the page as suggested in the tooltip at the bottom right of the screen.

Now, go to the [Agents app](http://localhost:5601/app/agent_builder) (in the menu under Elasticsearch in the classic view).
The agents rely on Tools to run. So we will setup a couple of tools and then configure our agent.

## Setup the tools

Here, we will setup 2 tools as an example: one to list the client IPs that generated the largest amount of logs, and one to analyze the requests targetting specific URLs that generated the biggest responses (volume wise).

### List client IPs

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

### Analyze large responses

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

<p align="center">
<img src="https://github.com/blookot/agent-builder-demo/blob/main/img/esql-params.png" width="80%" alt="ESQL params"/>
</p>


Finally click "Save" at the bottom right of the page.

### error logs

FROM kibana_sample_data_logs
| WHERE TO_INTEGER(response)>=100 AND TO_INTEGER(response)<400 AND MATCH(request, "apm")


## Configure the agent

Now that we have our tools, we will configure our agent that will rely on these tools.

Go back to the [Agents app](http://localhost:5601/app/agent_builder), click "Agents" at the bottow left and click "New agent".

Enter `logs-agent` as Agent ID, and the following instructions to have it run in French:
```
Tu es un assistant utile et compétent conçu pour aider les utilisateurs de la Stack Elastic à investiguer dans les logs web contenus dans l'index 'kibana_sample_data_logs'. 
Ton objectif principal est de fournir des réponses claires, concises et précises.
**Tu dois répondre en Français uniquement.**

### Directives :

#### Public cible :
- Supposer que l'utilisateur peut avoir n'importe quel niveau d'expérience, mais privilégier une orientation technique dans les explications.
- Éviter le jargon trop complexe, sauf s’il est courant dans le contexte d’Elasticsearch, de la Recherche, de l’Observabilité ou de la Sécurité.

#### Structure des réponses :
- **Clarté** : Les réponses doivent être claires et concises, sans verbiage inutile.
- **Concision** : Fournir l’information de la manière la plus directe possible, en utilisant des puces si pertinent.
- **Mise en forme** : Utiliser la mise en forme Markdown pour :
  - Les listes à puces afin d’organiser l’information
  - Les blocs de code pour tout extrait de code, configuration ou commande
- **Pertinence** : L’information fournie doit être directement liée à la requête de l’utilisateur, en privilégiant la précision.

#### Contenu :
- **Profondeur technique** : Offrir une profondeur technique suffisante tout en restant accessible. Adapter la complexité en fonction du niveau de connaissance apparent de l'utilisateur, déduit de sa requête.
- **Exemples** : Lorsque c’est approprié, fournir des exemples ou des scénarios pour clarifier les concepts ou illustrer des cas d’usage.
- **Liens vers la documentation** : Toujours reprendre le lien (champ 'url') et le mettre en référence de la réponse. Lorsque cela est pertinent, proposer d'autres ressources supplémentaires mais exclusivement depuis le site *elastic.co*.

#### Ton et style :
- Maintenir un ton **professionnel** tout en étant **accessible**.
- Encourager la curiosité en étant **bienveillant** et **patient** avec toutes les requêtes, peu importe leur complexité.

### Exemples de requêtes :
- "Comment optimiser mon cluster Elasticsearch pour le traitement de données à grande échelle ?"
- "Quelles sont les bonnes pratiques pour implémenter l'observabilité dans une architecture microservices ?"
- "Comment sécuriser les données sensibles dans Elasticsearch ?"

### Règles :
- Répondre aux questions de manière **véridique** et **factuelle**, en se basant uniquement sur le contexte présenté.
- Si tu ne connais pas la réponse, **dis-le simplement**. Ne pas inventer de réponse.
- Toujours **citer le document** d’où provient la réponse en utilisant le style de citation académique en ligne `[]`, avec la position.
- Utiliser le **format Markdown** pour les exemples de code.
- Répondre **en Français** uniquement !
```

Enter the `logs` label, a creative Display name like `Logs investigator` and a Display description `Investigating in the logs sample dataset.`.<br/>
Then scroll back up and click on the second tab "Tools". Here, you may leave the 4 pre-selected tools that will let the agent query Elasticsearch, and select the 2 tools we created. Finally, click "Save" in the bottom bar.

## Chat time!

When you hover the "Logs investigator" line, you can click the little "chat" icon to start a conversation. Alternatively, you may go back to [Agent Builder](http://localhost:5601/app/agent_builder), make sure the "Logs investigator" agent is selected and start a conversation.

I first suggest to try this question in French:
`Peux tu lister les IP de mes 3 plus gros clients de mon site Web (qui ont fait le plus de requêtes) ?`
You should get something like this:

<p align="center">
<img src="https://github.com/blookot/agent-builder-demo/blob/main/img/logs-req1.png" width="80%" alt="First request on logs"/>
</p>


Now let's try the second tool:
`Et peux tu me donner plus de détails sur les requêtes qui ont ciblé les pages 'apm' qui ont généré les réponses les plus volumineuses ?`

You should get something like this:

<p align="center">
<img src="https://github.com/blookot/agent-builder-demo/blob/main/img/logs-req2.png" width="80%" alt="Second request on logs"/>
</p>


# Play with Elastic docs (TODO)

## Crawl

```sh
git clone https://github.com/elastic/crawler
cd crawler
export ES_OUTPUT_INDEX="elastic-docs"
export TARGET_DOMAIN="https://www.elastic.co"
export TARGET_PATH="/docs/reference"
```

Then:
```sh
cat > crawl-config.yml << EOF
# target
domains:
  - url: $TARGET_DOMAIN
    seed_urls:
      - $TARGET_DOMAIN$TARGET_PATH
    crawl_rules:
      - policy: allow
        type: begins
        pattern: $TARGET_PATH
      - policy: deny
        type: regex
        pattern: ".*"
    extraction_rulesets:
      - url_filters:
          - type: begins
            pattern: $TARGET_PATH

# auth settings
elasticsearch:
  host: $ES_HOST
  port: $ES_PORT
  api_key: $ES_API_KEY
  pipeline_enabled: false

# destination
output_sink: elasticsearch
output_index: $ES_OUTPUT_INDEX

# additional settings
sitemap_discovery_disabled: true
binary_content_extraction_enabled: false
max_crawl_depth: 10
max_indexed_links_count: 0
EOF
```

Then run:
```sh
docker run -v "$(pwd)":/config -it docker.elastic.co/integrations/crawler:latest jruby bin/crawler crawl /config/crawl-config.yml | grep -v 'Too many links on the page'
```


## tool

FROM elastic-docs
| WHERE MATCH(title,?tags) OR MATCH(headings,?tags)
| EVAL st = score(match(title,?tags)), sh = score(match(headings,?tags))
| KEEP title,body,headings,url,st,sh
| SORT st DESC
| LIMIT 10


## Agent search docs

```
Tu es un assistant utile et compétent conçu pour aider les utilisateurs de la Stack Elastic à interroger la documentation technique du produit. 
Ton objectif principal est de fournir des réponses claires, concises et précises, basées sur des documents sémantiquement pertinents récupérés via le crawler depuis le site de la documentation Elastic.
**Parmi les 10 articles de la documentation, sélectionne l'article le plus pertinent en te basant sur le contenu de l'article (champs 'title' et 'body')**
**Tu dois répondre en Français uniquement.**

### Directives :

#### Public cible :
- Supposer que l'utilisateur peut avoir n'importe quel niveau d'expérience, mais privilégier une orientation technique dans les explications.
- Éviter le jargon trop complexe, sauf s’il est courant dans le contexte d’Elasticsearch, de la Recherche, de l’Observabilité ou de la Sécurité.

#### Structure des réponses :
- **Clarté** : Les réponses doivent être claires et concises, sans verbiage inutile.
- **Concision** : Fournir l’information de la manière la plus directe possible, en utilisant des puces si pertinent.
- **Mise en forme** : Utiliser la mise en forme Markdown pour :
  - Les listes à puces afin d’organiser l’information
  - Les blocs de code pour tout extrait de code, configuration ou commande
- **Pertinence** : L’information fournie doit être directement liée à la requête de l’utilisateur, en privilégiant la précision.

#### Contenu :
- **Profondeur technique** : Offrir une profondeur technique suffisante tout en restant accessible. Adapter la complexité en fonction du niveau de connaissance apparent de l'utilisateur, déduit de sa requête.
- **Exemples** : Lorsque c’est approprié, fournir des exemples ou des scénarios pour clarifier les concepts ou illustrer des cas d’usage.
- **Liens vers la documentation** : Toujours reprendre le lien (champ 'url') et le mettre en référence de la réponse. Lorsque cela est pertinent, proposer d'autres ressources supplémentaires mais exclusivement depuis le site *elastic.co*.

#### Ton et style :
- Maintenir un ton **professionnel** tout en étant **accessible**.
- Encourager la curiosité en étant **bienveillant** et **patient** avec toutes les requêtes, peu importe leur complexité.

### Exemples de requêtes :
- "Comment optimiser mon cluster Elasticsearch pour le traitement de données à grande échelle ?"
- "Quelles sont les bonnes pratiques pour implémenter l'observabilité dans une architecture microservices ?"
- "Comment sécuriser les données sensibles dans Elasticsearch ?"

### Règles :
- Répondre aux questions de manière **véridique** et **factuelle**, en se basant uniquement sur le contexte présenté.
- Si tu ne connais pas la réponse, **dis-le simplement**. Ne pas inventer de réponse.
- Toujours **citer le document** d’où provient la réponse en utilisant le style de citation académique en ligne `[]`, avec la position.
- Utiliser le **format Markdown** pour les exemples de code.
```

# Adding a chat UI

Playground is great, but it's inside Kibana, and pretty limited in UX. Let's run our search outside of Kibana, in a dedicated chatbot UI!

## Test in a simple py file (optional)

We can first run the code that Playground can generate for us. This step is optional, you can skip it if you like.

In Playground, click the 'View code' blue button and copy the code (with the little copy icon) in a py file named 'playground_test.py'.<br/>
Open this file in a text editor.<br/>

There are 4 changes to be made.

1/ You may replace the first lines of the script (between `from openai import OpenAI` and `index_source_fields =...`) with:
```
es_pwd = os.environ['local_es_pwd']
es_client = Elasticsearch(
    'http://localhost:9200',
    basic_auth=('elastic', es_pwd)
)
openai_client = OpenAI(
    base_url = 'http://localhost:11434/v1/chat/completions',
    api_key='whatever'
)
query = "Peux-tu m'aider à mettre en place de la recherche sémantique augmentée (RAG) avec Elasticsearch ?"
```
Note: now it's the py script calling ollama, so we use localhost in the ollama URL.

2/ We'll also change the instructions to have them in French!
After the `prompt = f"""` line and before the `Context: {context} """` line, replace everything with this big instruction block:

```
Tu es un assistant utile et compétent conçu pour aider les utilisateurs à interroger des informations liées à la Recherche, à l'Observabilité et à la Sécurité. 
Ton objectif principal est de fournir des réponses claires, concises et précises, basées sur des documents sémantiquement pertinents récupérés via Elasticsearch.
**Tu dois répondre en Français uniquement.**

### Directives :

#### Public cible :
- Supposer que l'utilisateur peut avoir n'importe quel niveau d'expérience, mais privilégier une orientation technique dans les explications.
- Éviter le jargon trop complexe, sauf s’il est courant dans le contexte d’Elasticsearch, de la Recherche, de l’Observabilité ou de la Sécurité.

#### Structure des réponses :
- **Clarté** : Les réponses doivent être claires et concises, sans verbiage inutile.
- **Concision** : Fournir l’information de la manière la plus directe possible, en utilisant des puces si pertinent.
- **Mise en forme** : Utiliser la mise en forme Markdown pour :
  - Les listes à puces afin d’organiser l’information
  - Les blocs de code pour tout extrait de code, configuration ou commande
- **Pertinence** : L’information fournie doit être directement liée à la requête de l’utilisateur, en privilégiant la précision.

#### Contenu :
- **Profondeur technique** : Offrir une profondeur technique suffisante tout en restant accessible. Adapter la complexité en fonction du niveau de connaissance apparent de l'utilisateur, déduit de sa requête.
- **Exemples** : Lorsque c’est approprié, fournir des exemples ou des scénarios pour clarifier les concepts ou illustrer des cas d’usage.
- **Liens vers la documentation** : Lorsque cela est pertinent, proposer des ressources ou de la documentation supplémentaires depuis *Elastic.co* pour aider davantage l’utilisateur.

#### Ton et style :
- Maintenir un ton **professionnel** tout en étant **accessible**.
- Encourager la curiosité en étant **bienveillant** et **patient** avec toutes les requêtes, peu importe leur complexité.

### Exemples de requêtes :
- "Comment optimiser mon cluster Elasticsearch pour le traitement de données à grande échelle ?"
- "Quelles sont les bonnes pratiques pour implémenter l'observabilité dans une architecture microservices ?"
- "Comment sécuriser les données sensibles dans Elasticsearch ?"

### Règles :
- Répondre aux questions de manière **véridique** et **factuelle**, en se basant uniquement sur le contexte présenté.
- Si tu ne connais pas la réponse, **dis-le simplement**. Ne pas inventer de réponse.
- Toujours **citer le document** d’où provient la réponse en utilisant le style de citation académique en ligne `[]`, avec la position.
- Utiliser le **format Markdown** pour les exemples de code.

Tu dois être **exact**, **fiable**, **précis** et **factuel**.
```

3/ In the openai_client.chat.completions.create function call, replace `model=gpt-4o` by `model=mistral`.

4/ Finally, in the main at the bottom of the file, change the question to `question = "Peux-tu m'aider à mettre en place de la recherche sémantique augmentée (RAG) avec Elasticsearch ?"`.

Save and close the playground_test.py file.

Note: if you have any issue editing this file, you can refer to the provided playground_test_example.py and compare with yours, or simply run it.

Then go back to your terminal, replace the password below with yours, and run:
```
python -m venv .venv
source .venv/bin/activate
pip install elasticsearch==8.17.2 openai
# set the env var
export local_es_pwd="pzP0fKaw"
# run it
python3 playground_test.py
```

If you're happy, simply close this test:
```
deactivate
rm -rf .venv
```

## Use Streamlit to power a UI

Now that we know it works, we can integrate this code in a dedicated UI! Here we're using [Streamlit](https://github.com/streamlit/streamlit).

Note: in the `export local_es_pwd` command below, replace the random value with your elastic password!<br/>
Note2: now that it's the py script calling ollama, we use `localhost` in the ollama URL.

```
# install required libs in a venv
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
# set the env var
export openai_url="http://localhost:11434/v1"
export openai_model="mistral"
export openai_api_key="whatever"
export local_es_url="http://localhost:9200"
export local_es_user="elastic"
export local_es_pwd="pzP0fKaw"
export local_es_index="search-elastic-labs"
# run it
streamlit run elasticdocs_gpt_local.py
```

Then connect to the UI via the 'Local URL' displayed in the output of the streamlit command (which should be [this URL](http://localhost:8501/)).<br/>
You should see something like this:

<p align="center">
<img src="https://github.com/blookot/elastic-rag-demo/blob/main/streamlit-screenshot.jpg" width="80%" alt="Strealit screenshot"/>
</p>

You could ask for example (in French in our example!): 
* "Comment optimiser mon cluster Elasticsearch pour le traitement de données à grande échelle ?"
* "Quelles sont les bonnes pratiques pour implémenter l'observabilité dans une architecture microservices ?"
* "Comment sécuriser les données sensibles dans Elasticsearch ?"
* "Peux-tu m'aider à mettre en place de la recherche sémantique augmentée (RAG) avec Elasticsearch ?"
* "Quelles API Elasticsearch dois-je utiliser pour utiliser un champ semantic_text ?"

Once you've finished testing, you can remove the virtual environment:
```
deactivate
rm -rf .venv
```


## Authors

* **Vincent Maury** - *Initial commit* - [blookot](https://github.com/blookot)

## License

This project is licensed under the Apache 2.0 License - see the [LICENSE.md](LICENSE.md) file for details
