## Securing Azure
### Keys
- Regenerate key command: ` az cognitiveservices account keys regenerate`
- Access to Key Vault works on "Security Principals"
  - Acounts use Microsoft Entra ID
  - Applications get get assigned a "managed identity"
- Token based auth
  - A subs key is presented first, and then a token is generated and granted to the app. 
  - Tokens last for 10 min

### MS Entra ID
- Ways to authenticate using MS Entra ID
  - Service principals:
    - Create custom subdomain
      - Select your subscription as a variable, and then create a service using a subdomain  
        ```
        Set-AzContext -SubscriptionName <Your-Subscription-Name>
        $account = New-AzCognitiveServicesAccount -ResourceGroupName <your-resource-group-name> -name <your-account-name> -Type <your-account-type> -SkuName <your-sku-type> -Location <your-region> -CustomSubdomainName <your-unique-subdomain-name>
        ```
    - Assign role to a service principal  
      - Register an application
        ```
        $SecureStringPassword = ConvertTo-SecureString -String <your-password> -AsPlainText -Force
        $app = New-AzureADApplication -DisplayName <your-app-display-name> -IdentifierUris <your-app-uris> -PasswordCredentials $SecureStringPassword
        ```
      - Create new service principal with `New-AzADServicePrincipal` and provide the app ID. 
        ```
        New-AzADServicePrincipal -ApplicationId <app-id>
        ```
      - Assign `Cognitive Services Users` role to the principal
        ```
        New-AzRoleAssignment -ObjectId <your-service-principal-object-id> -Scope <account-id> -RoleDefinitionName "Cognitive Services User"
        ```
  - Authenticate using managed identities
    - System assigned: linked to one resource. When the resource is deleted, the identity is too. 
    - User assigne: linked to multiple resources. Exists independently from its resources. 
    - A resource should have `Cognitive Services Contributor` role to be able to use the AI resource. 

## Monitoring
### Costs
- Use Azure Pricing Calculator
  - Results can be exported to excel
### Alerts
- Like Zabbix triggers.
- To create an alert for a resource:
  - resource >> Alerts tab >> add new alert.
  - Params:
    - Scope: the resource to be monitored
    - Condition: based on signal type (log or metric)
    - Actions: logic app, mail, etc.
### Metrics
- Can be checke on **Azure Monitor**
- Go to: resource >> metrics tab
- You can add dashboards with data from different resources
### Logs
[Documentation](https://learn.microsoft.com/en-us/azure/ai-services/diagnostic-logging)
- Azure Event Hubs: log data can be directed here. Then it is redirected to a third party service
- Two main services needed to use logs:
  - Azure Log Analytics: visualize and query logs
  - Azure Storage: store log archives (can be exported)
- On the resource side, you have to configure its **Diagnostic Setting**
  - Set the type of log that will be sent and the outputs

## Containers
[Documentation: Image list](https://learn.microsoft.com/en-us/azure/ai-services/cognitive-services-container-support#containers-in-azure-ai-services)
- When you deploy a container in host you must provide these 3 params:
  - ApiKey: from you AI service resource. Used for billing
  - Billing: Enpoint URI for the deployed AI service: Used for billing
  - Eula: `Accept` to say that you accept the license. 
- Authentication
  - Apps using a containerized AI service don't need to provide a subscription key. 
  - You can have your own authentification method.
- Examples of images:  
  > mcr.microsoft.com/product/azure-cognitive-services/vision
  > mcr.microsoft.com/product/azure-cognitive-services/speechservices
  > mcr.microsoft.com/azure-cognitive-services/textanalytics

## Content Safety
- It is an Azure resource. 
- Can be explored in the Foundry
- 対象：text , images and AI-generated content
- Available in major languages
- 4 Categories: hate, sexual, self-harm, violence
### Text
- Moderate text: rate the text for each category 0-6
- Prompt shields: detect prompts used to bypass LLM safety
- Protected material detection: AI generated text shouldn't contain copyrighted material.
- Groundedness detection: protects against inaccurated AI-generated text
  - Reasoning: the AI tells where it is bacing its response on
### Images
- Moderate: safe, low or high
- Multimodal Content: checks the images and any text in it with OCR. 
### Custome
- Custom categories: can be trained
- Safety system message: guides your AI's behaviour

# Generative AI
## Basics
- To create a service, you need **Azure OpenAI resource**
  ```
  az cognitiveservices account create \
  -n MyOpenAIResource \
  -g OAIResourceGroup \
  -l eastus \
  --kind OpenAI \
  --sku s0 \
  --subscription subscriptionID
  ```
- Roles needed in the Foundry
  - User: can view resources and use the chat playground
  - Contributor: create new deployments
- Some models you may not know:
  - Embedding models: convert text into vertors. Used for language analysis, like comparing text similarities
  - Whisper models: speech to text
  - Text to speech: 自明
- Pricing: tokens and model type
### Deployment
- Any number of deployments is okay. Just keep **tokens per minute** under control.
- Deployment methods:
  - Foundry
  - CLI:
    ```
    az cognitiveservices account deployment create \
      -g OAIResourceGroup \
      -n MyOpenAIResource \
      --deployment-name MyModel \
      --model-name gpt-35-turbo \
      --model-version "0125"  \
      --model-format OpenAI \
      --sku-name "Standard" \
      --sku-capacity 1
    ```
  - REST API
    - Define base model inside the body
    - [Documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/)
- Prompt types
  - classify content
  - Generate new content
  - Hold conversation
  - Transformation (translation and symbol conversion)
  - Summarize content
  - Factual responses
  - Utterance completion
- Factors that influence completion quality
  - The way the prompt is written
  - The model params
  - The data the model is trained on (!! This is the most important)
- API calls to GPT: [Azure Open AI >> How-to >> Completions & Chat Completions >> GPT-3.5-Turbo, GPT-4](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/chatgpt?tabs=python-new)
- "Few-shot": Provide a few examples to teach the model what it needs to do. 
### App Integration
- Available endpoints:
  - Completion
  - Chatcompletion: model takes input in the form of chat conversation
    - for each message the role is clearly defined ("system", "user")
    - You can send the message history
  - Embedding: model returns a vector representation of the input
- Availability per model 
  - model =< gpt3 : Completion
  - model = gpt 35 turbo : Chatcompletion is prefered
  - model => gpt 4: Chatcompletion
- API required params
  - endpoint, key, deployment name
    ```powershell
    curl https://YOUR_ENDPOINT_NAME.openai.azure.com/openai/deployments/YOUR_DEPLOYMENT_NAME/chat/completions?api-version=2023-03-15-preview \
      -H "Content-Type: application/json" \
      -H "api-key: YOUR_API_KEY" \
      -d '{"messages":[{"role": "system", "content": "You are a helpful assistant, teaching people about AI."},
    {"role": "user", "content": "Does Azure OpenAI support multiple languages?"},
    {"role": "assistant", "content": "Yes, Azure OpenAI supports several languages, and can translate between them."},
    {"role": "user", "content": "Do other Azure AI Services support translation too?"}]}'
    ```
  - Embeddings: curl https://YOUR_ENDPOINT_NAME.openai.azure.com/openai/deployments/YOUR_DEPLOYMENT_NAME/<b>embeddings</b>?api-version=2022-12-01 \
  - Completion: curl https://YOUR_ENDPOINT_NAME.openai.azure.com/openai/deployments/YOUR_DEPLOYMENT_NAME/chat/<b>completions</b>?api-version=2023-03-15-preview \
- **Using SDKs**
  - You also the need the **api version** to create a client
    ```py
    client = AzureOpenAI(
        azure_endpoint = '<YOUR_ENDPOINT_NAME>', 
        api_key='<YOUR_API_KEY>',  
        api_version="20xx-xx-xx" #  Target version of the API, such as 2024-02-15-preview
        )
    ```
  - The deployment name is defined in the request 
      ```py
      response = client.chat.completions.create(
          model=deployment_name,
          messages=[
              {"role": "system", "content": "You are a helpful assistant."},
              {"role": "user", "content": "What is Azure OpenAI?"}
          ]
      )
      ```
### Prompt Engineering
- Adjusting parameters
  - Increase temp: there will be more randomness in the model. More variation in sentence structure. 
  - top_p: more variation in word usage. 
- Guidelines for prompt engineering
  - Clear instructions: be descriptive
  - Instruction format: there is recency bias, so you might want to repeat the instructions at the end
  - Content: 
    - Primary: the subject of the query
    - Supporting: not the focus but may alter the response
    - Grounding: helps create more reliable responses
  - Cues: leading words for the model to build upon on
  - Request output format: Tell the model what kind of output to include
  - System message: may include tone, personality, topics to avoid, formating, ect. Use the role `system` to set the message. 
  - Message history
  - Few shot learning
    - Sent in the same way as history
    - Sets and example of the way the conversation should flow
  - Break down complex tasks
  - Chain of thought: Tell the model to explain how it arrived a that answer

## Rag: Retrieval Augmented Generation
### Intro
- RAG provides a knowledge base to a model, so that it can use the knowledge to generate answers. 
- You can update an LLM with new data without having to retrain it.
- You don't neet to fine-tune the LLM
- Used for: question answering, summarization, text generation
- Azure OpenAI makes use of AI Search to include the info in the response
- 流れ：
  1. Receive prompt
  2. Determine content and intent
  3. Query the index using content and intent
  4. Insert result chunk into the AI prompt, along with system message and user prompt
  5. Send 4. to AI as a whole prompt

- Fine Tuning:
  - Retrain an exisiting LLM model with extra data
  - More effective than prompt engineering
  - Time consuming and costly
### Add Data Source
[Documentation: How-to >> Use Your Data ](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/use-your-data?tabs=ai-search%2Ccopilot#ingesting-your-data-into-azure-cognitive-search?azure-portal=true)
- Methods:
  - Azure AI Studio
  - Chat Playground
  - Including it in the API call
- Where can the data be kept?
  - Upload data files
  - Use data in blob storage
  - Connect to an existing AI Index
- Suported data file types: 
  - md
  - html
  - txt
  - pdf
  - word
  - powerpoint
- ※Enabling **Semantic Seach** yields better responses but it's more costly. 
### Other
- It uses many tokens, so consider decreasing the token number in history, system message and user prompt
- When using RAG, the models responese is limited to 1500 tokens
- **API**
  - You need endpoint , key and indexName to use AI Search
    ```json
    {
    "dataSources": [
        {
            "type": "AzureCognitiveSearch",
            "parameters": {
                "endpoint": "<your_search_endpoint>",
                "key": "<your_search_endpoint>",
                "indexName": "<your_search_index>"
            }
        }
    ],
    "messages":[
        {
            "role": "system", 
            "content": "You are a helpful assistant assisting users with travel recommendations."
        },
    ...
    ```
  - The endpoint is different to the one used when simply calling a base model. It must have <code><b>extensions</b></code>
    ```
    <your_azure_openai_resource>/openai/deployments/<deployment_name>/chat/completions?api-version=<version>
    ```
  Include content type and api key as headers
## Dall-E
- Sizes : 1024 x 1024, 1792 x 1024, 1024 x 1792
- Vivid or natural
- Standard or HD
### Rest API
- You need: key (header), endopoint
- body: 
  - prompt
  - n: number of images (only 1 atm)
  - size: resolution 
  - quality: 
  - style: natural and vivid
- Response: 
  - DALL-E 2: gives you a call-back URL that the app can poll until image is ready
  - DALL-E 3: request is processed synchronously and returns a URL with the image
  - Include a **revised prompt** that is optimized

# AI Search
## Create AI Search solution
### Manage Capacity
- Tiers:
  - Free: to explore
  - Basic: small solutions. 15 indexes, 5 GB
  - Standart: more indeces and storage, optimized for better fast read performance
  - Storage optimized: Large indeces with more query latency
  - !! Tiers can't be changed later
- Replicas: create them to handle more requests
- Partitions: divide the index into multiple storage locations
- Search units: Replica number x partition number
### Components
- Data source:
  - Azure blob storage
  - Azure SQL data base
  - Documents in Cosmos DB
- Skillset
  - examples of data that can be added to enrich:
    - document lang
    - key phrases to determine the topics of doc
    - sentiment score
    - locations and people mentioned in the document
- Indexer
  - Inputs
    - data from skillsets
    - meta data
    - actual doc data
  - Outputs
    - Data maped to fields
- Index
  - Collection of JSON docs that can be queried
### Indexing process
- Creates a JSON doc with this structure:
  ```
    document
        metadata_storage_name ##規定
        metadata_author　＃＃規定
        content　##規定
        normalized_images　##規定
            image0
                Text ## from skillset
            image1
                Text
        language ## from skillset
        merged_content  ## from skillset

  ```
- 2 ways to index fields
  - Direct: fields extracted from data source are ALL directly mapped to index fields
    - implicit: same name
    - explicit: change name
  - Hierachichal: data from skills is located in fields that are 該当フィールドの直下
### Search an Index
- Full Text:
  - Lucene query syntax
    - simple:  match literal query
    - full: complex filters, regex, etc. 
- Params included in search through your app
  - search
  - queryType
  - searchFields
  - select
  - searchMode: "any", "all"
- Steps for querying
  1. Query parsing
  2. Lexical Analysis : 言葉の既約分数
  3. Document retrieval
  4. scoring 
### Filtering Results
- Methods
  - adding filters in Simple queryType
  - Odata filter expression with Full queryType
- Sorting
  - Default: sorted by relevance  
  - Use Odata $orderby to override
### Enhance searching
- Suggester
  - 2 options
    - Suggestions: suggested results
    - Autocomplete
  - Must be added to each field you it to be used for
  - Use suggestion and autocomplete API endpoints of the equivalent SDK methods for it to be usable on your app
### Custom scoring
- You can boos the relevance of certain fields
- The default is: term-frequency/inverse-document-frequency (TF/IDF) algorithm
- The custom scoring can be used for individual searches of as default
### Synonym
- Synonym maps
  - Create them for specific field so that users can use them for that field

## Create Custom Skill
- There are input and output schemas
### Add custom skill
- skill type: `Custom.WepApiSkill`
- specify:
  - URI to endpoing
  - context: where in the doc hierarchy to apply
  - inputs: from existing fields
  - output: target field name, etc. 
    ```json
    {
    "skills": [
          ...,
          {
            "@odata.type": "#Microsoft.Skills.Custom.WebApiSkill",
            "description": "<custom skill description>",
            "uri": "https://<web_api_endpoint>?<params>",
            "httpHeaders": {
                "<header_name>": "<header_value>"
            },
            "context": "/document/<where_to_apply_skill>",
            "inputs": [
              {
                "name": "<input1_name>",
                "source": "/document/<path_to_input_field>"
              }
            ],
            "outputs": [
              {
                "name": "<output1_name>",
                "targetName": "<optional_field_name>"
              }
            ]
          }
      ]
    }
    ```
- 流れ：
  1. Add new field to index
  2. Update skillset
  3. Change indexer to map the output of skill to newly created index field
  4. Rerun indexer

### Custom skill: text classification
### Custom Skill: machine learning
- Skill: `AmlSkill`
  - AML must be deployed as a web service endpoint
  - Must be Azure Kubernetes Service cluster. No Container Instances
  - Must be **HTTPS** endpoint

   ```json
    {
      "@odata.type": "#Microsoft.Skills.Custom.AmlSkill",
      "name": "AML name",
      "description": "AML description",
      "context": "/document",
      "uri": "https://[Your AML endpoint]",
      "key": "Your AML endpoint key",
      "resourceId": null,
      "region": null,
      "timeout": "PT30S",
      "degreeOfParallelism": 1,
      "inputs": [
        {
          "name": "field name in the AML model",
          "source": "field from the document in the index"
        },
        {
          "name": "field name in the AML model",
          "source": "field from the document in the index"
        },

      ],
      "outputs": [
        {
          "name": "result field from the AML model",
          "targetName": "result field in the document"
        }
      ]
    }
   ```

## Knowledge Store
### Intro
- Enrich data and populate index
- Pipeline of AI skills
- An index is a collection of JSON docs, but the data might be exported to other places
- Knowledge store is defined in the skillset
- Knowledge store is projections of enriched data
### Define Projections
- Shaper skill
  - Used to simplify complex fields so that they can be used in JSON projections
  - :thinking_face: fields in different hierarchies are put under the same field called `projection`
  - Defined like this:
    ```json
    {
      "@odata.type": "#Microsoft.Skills.Util.ShaperSkill",
      "name": "define-projection",
      "description": "Prepare projection fields",
      "context": "/document",
      "inputs": [
        {
          "name": "file_name",
          "source": "/document/metadata_content_name"
        },
        {
          "name": "url",
          "source": "/document/url"
        },
        {
          "name": "sentiment",
          "source": "/document/sentimentScore"
        },
        {
          "name": "key_phrases",
          "source": null,
          "sourceContext": "/document/merged_content/keyphrases/*",
          "inputs": [
            {
              "name": "phrase",
              "source": "/document/merged_content/keyphrases/*"
            }
          ]
        }
      ],
      "outputs": [
        {
          "name": "output",
          "targetName": "projection"
        }
      ]
    }

    ```
  - Creates this output:
    ```json
    {
        "file_name": "file_name.pdf",
        "url": "https://<storage_path>/file_name.pdf",
        "sentiment": 1.0,
        "key_phrases": [
            {
                "phrase": "first key phrase"
            },
            {
                "phrase": "second key phrase"
            },
            {
                "phrase": "third key phrase"
            },
            ...
        ]
    }
    ```
### Define Knowledge Store
- Must create `knowledgeStore` object in skillset
  - Must contain connection string to storage account
  - Must contain definition of projections
    - Can define these projections: object, table, file
    - !! Only one projection per type
    ```json
    "knowledgeStore": { 
          "storageConnectionString": "<storage_connection_string>", 
          "projections": [
            {
                "objects": [
                    {
                    "storageContainer": "<container>",
                    "source": "/projection"
                    }
                ],
                "tables": [],
                "files": []
            },
            {
                "objects": [],
                "tables": [
                    {
                    "tableName": "KeyPhrases",
                    "generatedKeyName": "keyphrase_id", ## field where the key of each record will be created
                    "source": "projection/key_phrases/*", ## the fields to populate in the table
                    },
                    {
                    "tableName": "docs",
                    "generatedKeyName": "document_id", 
                    "source": "/projection" 
                    }
                ],
                "files": []
            },
            {
                "objects": [],
                "tables": [],
                "files": [
                    {
                    "storageContainer": "<container>",
                    "source": "/document/normalized_images/*"
                    }
                ]
            }
        ]
    }
    ```

## Advanced Search
- Complex Lucene searchs with **full text search**
### Term boosting
- Example of simple search : `search=luxury&$select=HotelId, HotelName, Category, Tags, Description&$count=true`
  - There is no lexical analysis
  - The search service thinks that all terms in the search *must* be included in the result
- Query Parser
  - To use: add `&queryType=full`
    - Features
      - boolean operators
      - fielded search: search in a specific field
      - Fuzzy search
      - term proximity
      - regex
      - wildcards `*`, `?`
      - precedence grouping
      - term boosting
### Scoring profiles
- Azure ranks result docs using the BM25 algorithm
  - Factors:
    - number of times the term appears in the doc
    - doc size
    - rarity of each term
- You can boost specific fields with weights and functions
  - Functions:
    - Magnitude
    - Freshness
    - Distance
    - Tag
- How to create
  - Indexes >> scoring profile >> Add scoring profile
  - add a weight to specific field name
- You can set a scoring profile as the default
  - Otherwise you can also define which profile to use in the search with `&scoringProfile=PROFILE NAME`
### Analyzers and Tokenized Items
- Text from docs need processing when making the index. Ex:
  - text should be broken into words
  - function words should be removed
  - words reduced to root form 
- The processing is done by **analyzers**
  - 2 built-in types
    - Language: 
    - Specialized: language-agnostic. Used for eg.: zip codes or product IDs
      - `PatternAnalyzer` to use regexp
- Custom analyzers
  - consists of:
    - character filters: process a string
      - html_strip
      - mapping
      - pattern_replace
    - tokenizers: divide the text into tokens
    - token filter: remove or modify tokens
  - List of tokenizers and token filters
    - [AI Search >> How-to >> Keyword search >> Analyzers >> Add a custom analyzier](https://learn.microsoft.com/en-us/azure/search/index-add-custom-analyzers)  
- **Create Custom Analizer**
  - odata.type: `Microsoft.Azure.Search.CustomAnalyzer`
  - can only set 1 tokenizer, but various character and token filters
  - Testing: use REST API `Analyze Text` function
- Using the Analyzer
  - Define `indexAnalyzer` of `searchAnalyzer` in each field object.

### Include multiple languages in Index
- Add lang specific fields
  - Duplicate any fields that you want to offer in other languages
  - Add the appropriate Analyzer to that field for the specific lang
- You can limit the languages using in the rearch like so
  - use `$searchFields`
  - <code>search='parfait pour se divertir'&$select=listingId, description_fr, city, region, tags&<b>$searchFields=tags, description_fr</b>&queryType=full</code>
- Enrich with new translations
  - Add new translations with skills: `#Microsoft.Skills.Text.TranslationSkill`
  - Add field → add skill → update indexer to map output into index
    ```json
    "outputFieldMappings": [
      {
        "sourceFieldName": "/document/description/description_jp",
        "targetFieldName": "description_jp"
      },
      {
        "sourceFieldName": "/document/description/description_uk",
        "targetFieldName": "description_uk"
      }
    ]
    ```
### Order results by distance
- Geospatial funcs: 
  - `geo.distance`: returns distance in a straight line
  - `geo.intersects`: `true` if location is inside defined polygon
 - !! Locationf fields need be data type `Edm.GeographyPoint`

### Extra Info
- Character used to boost search term: `^`
- `Tag` is a func that can be used in a scoring profile

## Azure Data Factory
- ADF is zero-code
- ADF has connectors to 100 data stores
- Use REST or HTTP
- data stores can be used as sources or data sinks
- Index can be used as sink
  - source → ADF pipeline → index
- Index as a sink only supports these fields
  - str
  - int32
  - int64
  - double
  - boolean
  - DataTimeOffset
### Push API
- You can push data to your index 
- Features that can be manipulated with get, put, post, delete
  - index
  - doc
  - indexer
  - skillset
  - synonym map
- API requirements
  - API version in URL
  - key in header
  - your endpoint
- Eg. add data to index
  ```console
  POST https://[service name].search.windows.net/indexes/[index name]/docs/index?api-version=[api-version]

  ```
  - The body should include the action, which doc, and the data
  ```json
    {  
      "value": [  
        {  
          "@search.action": "upload (default) | merge | mergeOrUpload | delete",  
          "key_field_name": "unique_key_of_document", (key/value pair for key field from index schema)  
          "field_name": field_value (key/value pairs matching index schema)  
            ...  
        },  
        ...  
      ]  
    }
  ```
- Pushing too much data at the same time can yield a 503 error. Here are some ways to handle that:
  - Backoff
  - Threads

## Mantain AI Search
### Security
- 3 areas
  - Inbound search requests 
  - Outbound requests from solution to servers to index docs
  - Restrict access to docs
- Encription
  - Intransit data is encrypted with HTTPS TLS 1.3 encryption over port 443.
  - You can choose to use your own encryption keys with Key Vault
    - This enables **double encryption**
- Inbound traffic
  - Use **firewall** so that only your app can access the Search solution
  - **Most effective**: use private endpoint
- Authenticate requests
  - Use keys
    - Admin keys: write permissions and query sys info
    - Query: read permissions and query indeces
- RBAC
  - owner: all
  - contributor: all but cant change roles or assign
  - reader
- Outbound traffic
  - When Search solution inports data from data sources
  - Methods:
    - Key based auth
    - database logins
    - MS Entra logins
    - Firewall
    - Shared private link between services
      - Basic tier or S2
- Secure documents
  - Add security field 
   - use `search.in` filter
   ```json
    {
      "filter":"security_field/any(g:search.in(g, 'user_id1, group_id1, group_id2'))"  
    }
   ```
### Performance
- Measure performance
  - Use log analytics
  - capture diagnostics at search service level 
- Check for throttling
  - users will see a 503
  - indeces will see a 207
- Check specific queries
  - With postman or something
  - See the **elapsed-time** in the response
- Optimize index
  - Get rid of docs
  - review schema
    - delete skills
    - make some fields not searchable
  - scale up or scale out
- Optimize queries
  - Specify search fields
  - specify return fields
  - avoid partial search (regex etc.)
  - avoid high skip values
  - use search funcs and not individual terms
- Choosing tiers
  - [Documentation](https://learn.microsoft.com/en-us/training/modules/maintain-azure-cognitive-search-solution/03-optimize-performance-of-azure-cognitive-search-solution)
### Costs
- Tips
  - Minimize bandwith by using a few regions as possible
  - Scale up for indexing then down for querying
  - Keep search requests within datacenter using web apps
  - enable enrichment catching
### Reliability
- Make it highly available
  - Make replicas
    - two replicas gives you 99.9 for query
    - three reps gives you 99.9 for query and indexing
- Use different availability zone
- Distribute globally
- Back up indexes as json files

### Extra info
- Default metrics: Search latency, queries per second, and the percentage of throttled queries.
- Best way to manage service costs: Monitor and set budget alerts for all your search resources.

## Semantic Ranking
- Capability to use Language Understanding to get the context and improve ranking
- Default search ranking is BM25
- Semantic ranking does two things
  - Improves ranking through LU
  - Adds captions and answers
- How it works
  1. Take top 50 results
  2. Results are split into fields as per config
  3. Fields converted to tokens of 256
  4. Token strings are passed to a LU process that ranks them 
  5. Output the semantic relevance
- Semantic ranking can only ranked documents originally outputted by BM25
- Only uses top 50
- Up to 1000 queries are free. Otherwise choose standard pricing

- Where do I find list of AI Search Skills? 

### Set it up
- Need to have at least one index
- Not available in all regions
- Resources >> Search Service >> Semantic ranker
- Set per-index basis 
- Can have multiple configs on an index

### Extrainfo
- Prerequisites for using semantic ranking: Azure AI Search service with a billable tier.

## Vector search
- index, store and retrieve vercotr embedding from index
- Used to power:
  - Rag
  - Similarity searches
  - multi-modal searches
  - recommendation engines
- Usage scenarios
  - Use LLM models to encode text and use queries encoded as vectors to retrieve docs
  - Similarity search across different types of media
  - Represent docs in diff languages with multi-lingual embedded model
  - Hybrid text and vector search
  - Apply filters to text and num fields
  - Create a vector database to use as knowledge base etc. 
- Limitations:
  - Embeddings must be provided by **Azure OpenAI**, because AI Search does not generate them
  - Customer Managed Keys are not supported
  - There are storage limitations
    - If docs are too large, consider chunking. 

### Prepare the search
1. Send AI Search query to an embedded model
  - the model's response is then sent to a search engine
2. Check if index has vector field
  - run empty search 
  - check field `vectorSearch` with type `Collection(Edm.single)`
3. turn text query into vector

### Extra info
- Some embedding model types
  - Similarity: semantic similarity between texts
  - Text: relevance of docs
  - Code
- Embedding space: where vectors live.
- Q&A
  - What do you need to run a successful vector query? 
    - a search service URL and admin key
    - 

# Document Intelligence
## 101
- DocIntel outputs data in JSON
- 3 prebuilt models for general analysis:
  - Read
  - General doc
  - Layout
- 6 prebuilt models for spec type of doc
  - Invoice
  - Receipt
  - US tax declaration
  - ID doc
    - only bio pages of passports are supported
  - Business card
  - Health insurance card
- Otherwise use custom models
- Composed model: use multiple custome models in one solution
- DocIntel is more sophisticated than Azure AI Vision
  - AI Vision OCR will still need that you code some type of analysis process
### Resources
- Add Azure AI DocIntell resource
  - Free
    - not available for multi-service resource
  - Standard S0
### Choosing models
- General prebuild
  - Read
    - extract words and lines
    - printed or handwritten
    - detects lang
    - use `pages` to set the range of a pdf to be analyzed
  - General doc
    - Extract **key-value pairs**
    - **Entity extraction**. The only one that provides this!!!
  - Layout
    - Extract text, tables, and structure information
    - Good for getting info about the structure 
    - get selection marks to with bounding boxes
- Custom models
  - must provide **at least 5** examples
  - 2 types
    - Template models:
      - forms have a consistent visual template
      - 9 langs
      - also handwritten support
    - Neural:
      - More varied structures
      - Only major langs
- Composed model
  - DocIntell will automatically classify the form and apply the appropriate model
  - results of composed model include `docType` property, which tells you what model was used

- NO WORD Docs!!

## Extracting Data
- Requirements
  - JPG, PNG, BMP, PDF or TIFF
  - [Documentation](https://learn.microsoft.com/en-us/training/modules/work-form-recognizer/3-get-started)
### Train custom models
- you need the form docs and JSON docs that contain the labeled fields
- You can store in blob containers
- Docs you need
  - original form
  - .ocr.json: from *analyze doc* function
  - .labels.json: labeled fields with mapping
  - .fields.json: describing fields you want to extract
- Things to do
  - Put docs in container
  - Generate shared access security URL for container
  - Build model REST API func
  - Get model REST API to get id
- Alternatively, you can use the Azure Document Intelligence studio
  - 2 types
    - **custom template models**
      - When your forms are consisten!!
      - very fast, just a few min
      - accurately extract various types of data, text, tables, key-value pairs, etc.
      - available for **+100 lang**
    - **custom neural models** 
      - deep learned models
      - good for semi or unstructured data!!
### Use API
- to use API: use the `analyze document` function of either REST or SDK  
  ```py
  endpoint = "YOUR_DOC_INTELLIGENCE_ENDPOINT"
  key = "YOUR_DOC_INTELLIGENCE_KEY"

  model_id = "YOUR_CUSTOM_BUILT_MODEL_ID"
  formUrl = "YOUR_DOCUMENT"

  document_analysis_client = DocumentAnalysisClient(
      endpoint=endpoint, credential=AzureKeyCredential(key)
  )

  # Make sure your document's type is included in the list of document types the custom model can analyze
  task = document_analysis_client.begin_analyze_document_from_url(model_id, formUrl)
  result = task.result()
  ```
- succesful response contains `analyzeResult`
  - is an array of pages and their conten
- **improve confidence scores**
  - use better quality docs
  - if form appearance varies from doc to doc, consider making different models for each appearance. 
  - for low-risk apps, maybe 80% is okay. 
  - high risk, please use 100%
### Document Intelligence Studio
- Available projects
  - Document Analysis Models
    - Read: extract print or hand text. detect lang from text and img.
    - Layout: extract tables, text, and **structure information**
    - General Docs: key-value pairs, selection marks, and entities
  - Prebuilt models
  - Custom models 
- **How to create custom model**
  - Create Doc Intel resource or AI Services Resource
  - Use at least **5 or 6** samples for training
  - Confire **cross-domain resources** to store labeled data
  - Create project
    - provide storage container
  - Apply labels
  - Train model
  - Get Model ID
  - Test
## Create Composed Model
- Combine you costum models and publish as single service
- Used when there are multiple versions of a form
- Send documents to a single location, without you or the user having to sort what type it is. 
### How to
- Use `StartCreateComposedModelAsync()` in **custom code**
- Provide the Model ID of the Composed model
- In result, `docType` tells you what model was used for analysis
### Limits
- Free tier
  - Custom Template: 500
  - Custom Neural: 100
  - Composed: 5
- Standard s0
  - Custom Template: 5000
  - Custom Neural: 500
  - Composed: 200
- Neural and Custom Template cannot be mixed
- Custom Templat with Custom Template can be composed acrouss API 3.0 and 2.1
### Assemble a composed model
- You need
  - Azure AI Document Intelligence resource
  - Set of custom models
- With SDK
  - create object `DocumentModelAdministrationClient`
    - Needs endpoint and key
  - Create list with all the model IDs
  - Pas list to `StartCreateComposedModelAsync()`
### More Info
- You're trying to create a composed model but you're receiving an error. Which of the following should you check?
  - That the custom models were trained with labels.


# AI Language
- Service list
  - lang detec
  - key phrase extract
  - sentiment
  - named entity recog
  - entitity linking
- Basic JSON request format
  ```json
        {
        "kind": "EntityRecognition",
        "parameters": {
            "modelVersion": "latest"
        },
        "analysisInput": {
            "documents": [
            {
                ...
            }
            ]
        }
        }
  ```
### Detect Language
- **!! Docs must be under 5120 charac**
- **!! collection must be under 1000 items**
- Lang detected response.
  ```json
    {   "kind": "LanguageDetectionResults",
        "results": {
            "documents": [
            {
                "detectedLanguage": {
                "confidenceScore": 1,
                "iso6391Name": "en",
                "name": "English"
                },
                "id": "1",
                "warnings": []
            },
  ```
- If you pass a doc with multiple langs, the model returns the **prominent lang**
- If the parses can't understand the language both ISO and name are `unknown`. Confidence is `0`
### Extract Key Phrases
- **!! Docs must be under 5120 charac**
- Example response
  ```json
    "documents": [  
            {
            "id": "1",
            "keyPhrases": [
            "change",
            "world"
            ],
            "warnings": []
        },
  ```
### Sentiment
- Includes:
  - overall assesment of the doc
  - assesment for each sentence
- Confidence scores for three categories: positive, negative and neutral.
- How to calculate overall assesment
  - neutral = neutral
  - positive + neutral = positive
  - negative + neutral = negative
  - negative + positive = mixed
### Entities (NER)
- List of available types
  - Person
  - Location
  - DateTime
  - Organization
  - Address
  - Email
  - URL
  - ※List documntation [Ai Lang >> Azure AI Lang capabilities >> NER >> Prebuilt >> concepts >> recognized entitiy categories](https://learn.microsoft.com/en-us/azure/ai-services/language-service/named-entity-recognition/concepts/named-entity-categories?tabs=ga-api)
- **!! Must provide language in the request**
- Response
  ```json
    {
    "kind": "EntityRecognition",
    "parameters": {
        "modelVersion": "latest"
    },
    "analysisInput": {
        "documents": [
        {
            "id": "1",
            "language": "en",
            "text": "Joe went to London on Saturday"
        }
        ]
    }
    }
  ```
## Question Answering
- Azure AI Language has a "QA" capability
- You define a **knowledge base**
  - Can be queried by language
  - Published to REST enpoint
  - bots and apps use the endpoint
  - Examples of sources
    - Website containing FAQs
    - Files containing structured text (brochures)
    - built-in **question answer pairs**
- New versionn of QnA maker

### Compare QA and LU
||QA|LU|
|--|--|--|
|場面|user ask question, wants answer|submit utterance, want appropriate response|
|query processing|Use NLU to match q with a|Use NLU to interpret intent and match entity|
|Response|static|indicates most likely intent and entitiy|
|client logic|client logic present a|client app takes action based on the intent presented|

### Create Knowledge Base
- Turn on **question answering** in your Language Service resource
- Create or select Azure AI Search resource
- Lang Studio: create Custom Question Answering project

### Multi-turn conversation
- You can do this
- Define followup questions

### Train and Test
- Train the language model
- Test and check the
  - answers
  - confidence scores
  - other potential answers

### Consume Knowlege Base with API
- Use this request body
  ```json
    {
        "question": "What do I need to do to cancel a reservation?",
        "top": 2, ## max number of answers
        "scoreThreshold": 20, ## return answers >= to this num
        "strictFilters": [  ## limit to anwers that contain this metadata
            {
            "name": "category",
            "value": "api"
            }
        ]
    }
  ```
### Improve QA performance
- **Active learning**
  - Enabled by default
  - Offers alternate qs for your qa pairs ("Review suggestions" pane)
  - You can reject or accept suggestions
- Synonyms
  - Define them with rest
  ```json
  {
        "synonyms": [
            {
                "alterations": [
                    "reservation",
                    "booking"
                    ]
            }
        ]
    }
  ```
## Conversational Language Model
### Intro
- Natural Language Processing ＞＞　Natural Language Understanding
- Common flow
  - app takes Natlang
  - lang model determines intent
  - app uses intent to perform action
### Prebuilt Capabilities
- 2 categories
  - Pre-config
    - Summarization
      - for docs
      - for conversations
    - Named Entity Recognition
      - extract proper nouns
    - PII detection
      - identify, categorize and redact 
    - Key Phrase Extract
      - main concepts from a text
    - Sentiment Analysis
    - Lang Detection 
  - Learned
    - CLU
      - predict intent and extract important entities
      - data must be tagged
    - Custom named entity recog
      - extract specified entities from unstructured text
    - Custom text classi
      - Classify you text into groups you define
    - Question Answering
### Resources to build CLU model
- Create CLU resource to build model and use model after deploy
  - Create under **Language Service**
- You can use **language studio**
- Use REST
  - use this header for key：`Ocp-Apim-Subscription-Key`
  - Request deploy: `{ENDPOINT}/language/authoring/analyze-conversations/projects/{PROJECT-NAME}/deployments/{DEPLOYMENT-NAME}?api-version={API-VERSION}`
    - as a body it nees the `trainedModelLabel` param. This will be the name of the deployment
  - With `GET` you can get the deployment status. Use `/jobs/` and provide the job id: `{ENDPOINT}/language/authoring/analyze-conversations/projects/{PROJECT-NAME}/deployments/{DEPLOYMENT-NAME}/jobs/{JOB-ID}?api-version={API-VERSION}`
- **!! for buitl in features use the `analyze-text` enpoint**
  - then in the body, define the feature you want to use in the `kind` param
  ```json
  {
      "kind": "LanguageDetection",
      "parameters": {
          "modelVersion": "latest"
      },
      "analysisInput":{
          "documents":[
              {
                  "id":"1",
                  "text": "This is a document written in English."
              }
          ]
      }
  }
  ```
### Define intents, utterances, and entities
- By defining these 3 you create a model
- Identify all the intents your users might have
- Identify all the utterances and their synonymic expression that the users might use
- Labeling best practice
  - label precisely
  - label consintently 
  - label completely
- Entity types
  - Learned
  - List
  - Prebuilt
- For each intent provide different example of what might be said, so that the model can understand a variaty of stuff
- There are prebuilt components for entities. 
  - Eg., nums, emails, URL, choices
## Text classification solutions
- It is **custom**
- 2 types
  - single label classification
  - multiple label classification 
    - when labelling, data must remain clear and a good distribution must be provided 
- Metrics used
  - Recall: of all labels, how many were identified
  - precision: how many of the predicted labels are correct
  - f1 score: a function of recall and precision
- API
### How to build
- Build in Lang Studio or through api 
- Life cycle
  1. Define labels
  2. Tag data
  3. train model
  4. view model: model is scored between 0 and 1
  5. improve model 
  6. deploy
  7. classify
- Data split
  - automatic or manual
  - automatic: for large datasets, consistent data or well distributed data for all classes
- Deployment options
  - in one project you can have multiple models and deployments
  - each has its own name 
  - **!! each project has a limit of 10 names**
- Choose deployment model like so: 
  ```json
  <...>
    "tasks": [
      {
        "kind": "CustomSingleLabelClassification",
        "taskName": "MyTaskName",
        "parameters": {
          "projectName": "MyProject",
          "deploymentName": "MyDeployment"
        }
      }
    ]
  <...>
  ```
- REST API
  - key: `Ocp-Apim-Subscription-Key` param
  - train: `<YOUR-ENDPOINT>/language/analyze-text/projects/<PROJECT-NAME>/:train?api-version=<API-VERSION>`
    ```json
    {
        "modelLabel": "<MODEL-NAME>",
        "trainingConfigVersion": "<CONFIG-VERSION>", ##model version you wanna train
        "evaluationOptions": {
            "kind": "percentage",
            "trainingSplitPercentage": 80,
            "testingSplitPercentage": 20
        }
    }
    ```
  - consume model
  ```json
  {
    "displayName": "Classifying documents",
    "analysisInput": {
      "documents": [
        {
          "id": "1",
          "language": "<LANGUAGE-CODE>",
          "text": "Text1"
        },
        {
          "id": "2",
          "language": "<LANGUAGE-CODE>",
          "text": "Text2"
        }
      ]
    },
    "tasks": [
      {
        "kind": "<TASK-REQUIRED>", ##This is the part to remember
        "taskName": "<TASK-NAME>",
        "parameters": {
          "projectName": "<PROJECT-NAME>",
          "deploymentName": "<DEPLOYMENT-NAME>"
        }
      }
    ]
  }
  ```
- kind is `CustomMultiLabelClassification` or single

- How to see results of classification task: **Call the URL provided in the header of the qureqest response   **
 ## Custom Named Entity Recognition (Custom NER)
### Intro
- Custom vs Builtin NER
  - Builtin
    - person, location, organization, URL
    - Documentation for all types: [Azure AI Language >> Azure AI Language Capabilities >> Name entitity recognition >> Prebuilt >> Concepts >> Recognized entitity categories](https://learn.microsoft.com/en-us/azure/ai-services/language-service/named-entity-recognition/concepts/named-entity-categories?azure-portal=true&tabs=ga-api)
    - API: `<YOUR-ENDPOINT>/language/analyze-text/jobs?api-version=<API-VERSION>`
- Lifecycle
  - Define entities
  - Tag data
  - Train model
  - View moel
  - Improve model
  - Deploy
  - Extract entities
- How to get high quality data
  - Diversity
  - Distribution
  - Accuracy: data that is close to the real world  
- Avoid ambiguous entities
- Keep entities distinct and avoid 
- To request a task, use: ` "kind": "CustomEntityRecognition",`
### Project Limits
[Documentation: Azure AI Language >> Overview >> Quotas and Limits](https://learn.microsoft.com/en-us/azure/ai-services/language-service/concepts/data-limits)
- Restrictions:
  - Training - at least 10 files, and not more than 100,000
  - Deployments - 10 deployment names per project
  - APIs
    - Authoring - this API creates a project, trains, and deploys your model. Limited to 10 POST and 100 GET per minute
    - Analyze - this API does the work of actually extracting the entities; it requests a task and retrieves the results. Limited to 20 GET or POST
  - Projects - only 1 storage account per project, 500 projects per resource, and 50 trained models per project
  - Entities - each entity can be up to 500 characters. You can have up to 200 entity types.
### Labeling Data
- Three factors
  - Consistency
  - Precision
  - Completeness
- You can also import labeled data with the API
  - **!! Info in the MS Learn path**
### Train and evaluate
- Use precision, recall and F1
- Interpretation
  - Recall↑ + Precision↓ = model correctly identifies entities but can't label them correctly. 
  - Recall↓ + Precision↑ = model correctly labels entities but is not good at finding them.
- Confusion matrix
  - A detailed view of how each labeled performed

### Extra Info
- Where is labeled data stored??
  - In a JSON filed kept in the Storage account for the project
  - filedn JSON 'k'eft Accountn storage projenkt

## Azure AI Translation Resource
- Three main features
  1. Language detection
  2. One-to-many translation
  3. Script transliteration
- 専用resouce or multi resource
  - **use Text Analytics API**
### Examples of the services
- **Detect Lang**
  - Bash example: `curl -X POST "https://api.cognitive.microsofttranslator.com/detect?api-version=3.0" -H "Ocp-Apim-Subscription-Region: <your-service-region>" -H "Ocp-Apim-Subscription-Key: <your-key>" -H "Content-Type: application/json" -d "[{ 'Text' : 'こんにちは' }]`
  - The service returns the language in ISO format  
    ```json
    [
      {
        "language": "ja",
        "score": 1.0,
        "isTranslationSupported": true,
        "isTransliterationSupported": true
        
        
      }
    ]
    ```
- **Translate**
  - In the URL, specify `to` and `from` langs. 
  - There can be multiple `to` langs  
  - `curl -X POST "https://api.cognitive.microsofttranslator.com/translate?api-version=3.0&from=ja&to=fr&to=en" -H "Ocp-Apim-Subscription-Key: <your-key>" -H "Ocp-Apim-Subscription-Region: <your-service-region>" -H "Content-Type: application/json; charset=UTF-8" -d "[{ 'Text' : 'こんにちは' }]"`
  
- **Transliteration**
  - In the URL, specify `toScript` and `fromScript` langs. 
  - There can be multiple `toScript` langs  
  - `curl -X POST "https://api.cognitive.microsofttranslator.com/transliterate?api-version=3.0&fromScript=Jpan&toScript=Latn" -H "Ocp-Apim-Subscription-Key: <your-key>" -H "Ocp-Apim-Subscription-Region: <your-service-region>" -H "Content-Type: application/json" -d "[{ 'Text' : 'こんにちは' }]"`
  ### Translation Option
  - Word alignment: what source words correspond to what translated words
    - To use: set **includeAlignment** to `true`
    - You get the alignment like so:
    ```json
    [
      {
          "translations":[
            {
                "text":"智能服务",
                "to":"zh-Hans",
                "alignment":{
                  "proj":"0:4-0:1 6:13-2:3"
                }
            }
          ]
      }
    ]
    ```
- Sentence length
  - set **includeSentenceLength** parameter to `true`
  - You get:
  ```json
      "translations":[
         {
            "text":"Salut tout le monde",
            "to":"fr",
            "sentLen":{"srcSentLen":[12],"transSentLen":[20]}
         }
      ]
  ```
- Profanity filter
  - param is **profanityAction**
  - Set to:
    - `noAction`
    - `deleted`
    - `marked`: asterisks

- **Options Documentation:** [Azure AI Translator >> Text Translation >> Reference >> Translate](https://learn.microsoft.com/en-us/azure/ai-services/translator/text-translation/reference/v3/translate)

### Costum Translation
- For orgs with specific vocab
- Use Custom Translator portal
  1. Create workspace
  2. Create project
  3. Upload training data files
  4. Train model
  5. Test
  6. Publish
- Request a translation: `https://api.cognitive.microsofttranslator.com/translate?api-version=3.0`

## Azure AI Speech
- Services
  1. Text ⇔ Speech
  2. Text ⇔ Speech
  3. Speech translation
  4. Speaker Recognition
  5. Intent Recognition
### Speech to Text API
- 2 APIs
  1. Speech to Text API
  2. Speech to Text Short Audio: up to **60** secs
- SDK configs
  - "SpeechConfig": Info to connect to resource
  - "AudioConfig": Source to audio. Mic or file
  - "SpeechRecognizer" object: Created from the two above
  - Use method `RecognizeOnceAsync`
  - The result object: `SpeechRecognitionResult`
    - duration
    - OffsetInTicks
    - properties
    - Reason 
    - ResultID
    - Text
  - Examples fo Result: `NoMatch`, `Canceled` 
    - Should check `CancellationReason`
### Text to Speech API
- 2 APIs
  - Text to Speech: the normal one
  - Batch Synthesis: large volumes of audio
- SDK configs
  - "SpeechConfig": Info to connect to resource
  - "AudioConfig": Source to audio. Mic or file
  - "SpeechSynthesizer" object: Created from the two above
  - Use method `SpeechTextAsync`
  - The result object: `SpeechSynthesisResult`
    - Audiodata
    - properties
    - Reason 
    - ResultID
  - When success: Reason is `SynthesizingAudioCompleted `
### Other info
- Audio format can be set by `SpeechSynthesisOutputFormat `
  - Eg: `speechConfig.SetSpeechSynthesisOutputFormat(SpeechSynthesisOutputFormat.Riff24Khz16BitMonoPcm);`
- Voices can be set by `SpeechSynthesisVoiceName`
  - Eg: `speechConfig.SpeechSynthesisVoiceName = "en-GB-George";`
- How to use SSML : `speechSynthesizer.SpeakSsmlAsync(ssml_string);`

## Azure AI Speech service: Translate speech
### Provision the service
- Use AI Speech Resource or Multi-service Resource
- What you need for SDK or API
  - location (eg: eastus)
  - key
### Seting up object to use SDK
- SpeechTranslationConfig: info about connection
  - location
  - key
  - source lang
  - target lang
- AudioConfig: audio source
  - microphone or audiophile
- TranslationRecognizer: created with SpeechTranslationConfig and AudioConfig
  - Methods
    - RecognizeOnceAsync: translate **single** utterance
- Response keys:
  - Duration
  - OffsetInTicks
  - Properties
  - Reason: `RecognizedSpeech`
  - ResultId
  - Text: transcript of source lang
  - Translation: dic of translations

### Synthesize Translations
2 Types
- Event-based
  - 1:1 translation
  - speech is captured as audio stream
  - Config
    - TranslationConfig: here set voice
    - TranslationRecognizer: get event handler for **"Synthesizing"** event.
    - Result: use method GetAudio() to retreave byte stream of audio
- Manual
  - Iterate through translations with **SpeechSynthsizer**　object

# Azure AI Vision
### Analyze Images
Provision AI Vision Resource
- Features
  - Description and tag generation
  - Object detection
  - People detection
  - Image analysis: metadata, color, type
  - Category identification: also "does it contain any landmarks?"
  - Background removal
  - Moderation rating
  - OCR
  - Thumbnail generation
### How to Analyze and Image
- !! Detecting celebrities needs a **limited access policy**
- API
  - Create `ImageAnalysisClient` object: give credentials and endpoint
  - request analysis through the object's `VisualFeatures` property
  ```py
  result = client.analyze(
      image_url="<url>",
      visual_features=[VisualFeatures.CAPTION, VisualFeatures.READ],
      gender_neutral_caption=True,
      language="en",
  )
  ```
  - list of `visual_features`
    1. Tags
    2. objects
    3. captions
    4. dense_captions: more detailed
    5. people
    6. smart_crops: bounding box
    7. read: ocr
  - the result usually contains a confidence score and bounding box
### Thumnail creation and background removal
- Azure can determine the **region of interest** of an image
- This is used for thumnbnails
  - You can choose cropping rations from **0.75 ～　1.80**
- Removing background
  - Azure creates an **alpha matte**
## Classify images: AI Custom Vision
- Build your own models!!!
- 2 tasks involved
  1. Use existing labeled images to train
  2. Create client app to submit img to be classified
- You need 2 resources
  1. Training resouce
    - Either Cusom Vision resource (**training**) or multi-service resource
  2. Prediction resource (request to the model)
    - Either Cusom Vision resource (**prediction**) or multi-service resource
### About image classification
- Img classification is giving a **label** to an image
  - Multiclass: model contains many classes, but each img can only have one label
  - Multilabel: an img can have many labels
### How to train model
- Use REST, SDK, 0r Azure AI Custom Vision Portal
## Object Detection
- Model is trained to detect **one or more clasess** of objects in an img
- Two components in a prediction
  1. object class
  2. bounding box
- You need 2 resources
  1. Training resouce
    - Either Cusom Vision resource (**training**) or multi-service resource
  2. Prediction resource (request to the model)
    - Either Cusom Vision resource (**prediction**) or multi-service resource
### Options for labeling images (for training)
- In the Custom Vision Portal, Azure suggests bounding boxes
- **smart labeler**: after first training batch, it will suggest the labels for new training data
- Labeling tools
  - Azure Machine Learning Studio
  - Microsoft Visual Object Tagging Tool (VOTT)
- Specifieng bounding boxes numerically
  - It need 4 params
    1. Left
    2. Top
    3. Width
    4. Height
    - The units of measurements are **proportion** to the entire image

## Faces
- There are 2 service that have to do with faces
  1. Azure AI Vision service
    - detect people
    - return bounding box
  2. **Face** service
    - face detection (bounding box)
    - Facial feature analysis
    - Face comparison and verification **!Limited Access Policy**
    - Facal recognition **!Limited Access Policy**
### Face detection
- With such a request, you'll get:
  - Bounding box
  - Confidence score
  ```json
  {
    "modelVersion": "2023-10-01",
    "metadata": {
      "width": 400,
      "height": 600
    },
    "peopleResult": {
      "values": [
        {
          "boundingBox": {
            "x": 0,
            "y": 56,
            "w": 101,
            "h": 189
          },
  ```
- **!! Age and gender prediction have been removed**
### Feautures of Face service
1. Face detection: see section above
2. Attribute Analysis
  - Head pose
  - glasses
  - blur
  - exposure
  - noise
  - occlusion
  - accesories
  - quality for recognition
3. facial landmark location
4. Face comparison: are two faces the same?
5. Facial recognition: predict people in new images
6. Facial liveness: real or deepfake?

- Provisioning
  - AI Face resource
  - multi-service resource
### comparing faces
- When a face is capture, a GUID is generated for it. It lasts **24 hours**
### Create a facial recognition service
how to:
1. Create person group
2. Add person to group











