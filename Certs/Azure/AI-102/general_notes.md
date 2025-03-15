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


- Where do I find list of AI Search Skills? 





