

# Play with Elastic docs (TODO)


## Crawl

Get the crawler:
```sh
git clone https://github.com/elastic/crawler
cd crawler
```

Create crawler config:
```sh
export ES_HOST="http://host.docker.internal"
export ES_PORT="9200"
export ES_API_KEY="dE9NM3Rab0Jyd3dRa3FnRzJzZUg6TmdhdGJmRWZoMHJSekVKR3hleWdFUQ=="
export ES_OUTPUT_INDEX="elastic-docs"
export TARGET_DOMAIN="https://www.elastic.co"
export TARGET_PATH="/docs/reference"
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
| KEEP title,body,headings,url
| LIMIT 10

(sorting by match score is coming, not yet available in 9.2)

## Agent search docs

[ref](https://github.com/blookot/agent-builder-demo/blob/main/docs-agent-instructions.txt)


