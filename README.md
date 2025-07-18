# How To: Configure Elastic Grounding in GCP Vertex and Elastic Serverless 

## Overview
  
- Set up Elasticsearch Project
- Deploy ELSER
- Download and index data in Elasticsearch
- Create and test ELSER/search template
- Gather configuration information
- Configure Vertex
- Configure grounding - endpoint, api, index name and search template
- System commands
- Prompt with and without grounding


## Set up Elasticsearch Project

Navigate to cloud.elastic.co and login
<img width="974" alt="Screenshot 2025-06-17 at 9 58 38 AM" src="https://github.com/user-attachments/assets/e8ca7cd7-2fd2-484b-8b15-a17befe1dec3" />

Create a new Elastic Project by clicking Create serverless project


In the next screen, select Elasticsearch by clicking Next
<img width="1078" alt="Screenshot 2025-06-17 at 9 58 46 AM" src="https://github.com/user-attachments/assets/e96e0d82-f787-496f-9fb5-0bfb7e0329eb" />


Configure the project name and hyperscaler in this screen and click Create serverless project to continue
<img width="1099" alt="Screenshot 2025-06-17 at 9 58 57 AM" src="https://github.com/user-attachments/assets/293a75e9-875e-4e7d-a0dd-2eb9913bcf4d" />


The project will take a few minutes to deploy
<img width="1107" alt="Screenshot 2025-06-17 at 9 59 10 AM" src="https://github.com/user-attachments/assets/70f5f1b1-6e5d-4c61-9bdf-1512c5cfcd08" />

## Deploy ELSER

Once the project has been deployed,  an update to the creating project page will occur. Click continue. This will bring you to Kibana with an add data page. In the lower left corner, expand project settings and select Trained Models.
<img width="1030" alt="Screenshot 2025-06-17 at 10 12 24 AM" src="https://github.com/user-attachments/assets/8925236c-8122-4c4e-97c9-63d5918f3eb2" />


Click Start Deployment on .elser_model_2_linux-x86_64
<img width="1075" alt="Screenshot 2025-06-17 at 10 14 38 AM" src="https://github.com/user-attachments/assets/e32672df-c47a-4a88-b7f2-4151f3d92733" />

Select the defaults by clicking start<br>
<img width="514" alt="Screenshot 2025-06-17 at 10 16 15 AM" src="https://github.com/user-attachments/assets/093e0698-d3e7-4c1d-a726-39ec5f7be20e" />


And the model will show as deploying
<img width="860" alt="Screenshot 2025-06-17 at 10 16 39 AM" src="https://github.com/user-attachments/assets/c4a7a2f6-24fc-4ab9-b0a3-2d54909383e0" />


This will take a few minutes and in the meantime we can index the sample data.

## Download and index data in Elasticsearch
This guide uses an abbreviated (1000 lines) msmarco-passagetest2019-top1000 data set called [msmarco-passagetest-2019-top-1000-unique-short.tsv](https://github.com/abeaudin/Grounding_VertexAI_With_Elastic_and_Elser/blob/main/msmarco-passagetest2019-unique-short.tsv), The original top 1000 version of this file is referenced [here](https://www.elastic.co/docs/solutions/search/semantic-search/semantic-search-elser-ingest-pipelines#load-data) in the Elastic Docs. 


From the menu on the left side, select Getting Started
<img width="758" alt="Screenshot 2025-06-17 at 10 22 20 AM" src="https://github.com/user-attachments/assets/994d05ff-36ab-4de8-af3e-eb8cebf77d5a" />


Then select Upload a file and in the following screen, select or drag and drop the tsv file 
<img width="1011" alt="Screenshot 2025-06-17 at 10 24 25 AM" src="https://github.com/user-attachments/assets/a53f170d-f8cd-4013-8bb9-0c411e51d55c" />

Click import and in the subsequent screen, add the index-name, leave Create data view checked and click import
<img width="846" alt="Screenshot 2025-06-17 at 10 47 52 AM" src="https://github.com/user-attachments/assets/a5e6f537-b08d-4c52-9c48-39d9f6f445bc" />


The importer will have imported the data and now we can switch to the dev console for the next steps.
<img width="1069" alt="Screenshot 2025-06-17 at 10 49 08 AM" src="https://github.com/user-attachments/assets/80c89768-11f1-4eb4-a3b2-9b10bf0300a1" />

## Create pipeline and reindex data

### ELSER Setup and Embedding Creation
Create embedding index
```
PUT my-index
{
 "mappings": {
   "properties": {
     "content_embedding": {
       "type": "sparse_vector"
     },
     "content": {
       "type": "text"
     }
   }
 }
}
```
Create Ingest Pipeline
```
PUT _ingest/pipeline/elser-v2-test
{
 "processors": [
   {
     "inference": {
       "model_id": ".elser_model_2_linux-x86_64",
       "input_output": [
         {
           "input_field": "content",
           "output_field": "content_embedding"
         }
       ]
     }
   }
 ]
}
```
Reindex the data
```
POST _reindex
{
 "source": {
   "index": "test-data",
   "size": 50
 },
 "dest": {
   "index": "my-index",
   "pipeline": "elser-v2-test"
 }
}
```

Test query
```
GET my-index/_search
{
 "_source": ["content"],
  "query":{
     "sparse_vector":{
        "field": "content_embedding",
        "inference_id": ".elser_model_2_linux-x86_64",
        "query": "What is Pontiac"
     }
  }
}
```

## Create Search Template and Test
Create the search template
```
PUT _scripts/google-template-elser
{
 "script": {
   "lang": "mustache",
   "source": {
     "_source": ["content"],
       "size": "{{num_hits}}",
       "query":{
         "sparse_vector":{
           "field": "content_embedding",
           "inference_id": ".elser_model_2_linux-x86_64",
           "query": "{{query}}"
         }
       }
   }
 }
} 
```

Test the search template
```
GET my-index/_search/template
{
 "id": "google-template-elser",
 "params": {
   "query": "What is Pontiac",
   "num_hits": 5
 }
}
```

Now that this is all tested, let’s gather up the elasticsearch endpoint and create an API key so that we can configure the grounding service in the Google Cloud Console.

## Gather configuration information

Return to the Getting Started section of Kibana by clicking on Getting Started on the lower left, scroll down and there are two sections we need to work with
API Key
Copy your connection details
<img width="1116" alt="Screenshot 2025-06-17 at 11 55 31 AM" src="https://github.com/user-attachments/assets/152cd12c-001f-4698-994e-0a5f0c98d676" />




Copy your connection details is straight forward, copy it as an Elasticsearch endpoint.

For the API Key, click New and in the flyover:
<img width="717" alt="Screenshot 2025-06-17 at 11 56 14 AM" src="https://github.com/user-attachments/assets/a6832d2e-fda0-4d6a-a62a-b06c4696aefd" />


Enter a Name
Set expiration/metadata/security privileges as needed. I left them as default.

Click Create API key at the lower right of the flyover

The flyover will disappear and a ‘Store this API Key’ section in green will appear.Copy the key.
<img width="1123" alt="Screenshot 2025-06-17 at 11 54 46 AM" src="https://github.com/user-attachments/assets/ac8dcbde-f760-4bef-8223-43e41e7822d2" />


Google Configuration
In the GCP Console, using the menu in the upper left, navigate to Vertex AI -> Dashboard
<img width="474" alt="Screenshot 2025-06-17 at 12 06 51 PM" src="https://github.com/user-attachments/assets/be5d1faa-1875-4049-bb8d-f6150ef4fc52" />


In the dashboard, select Vertex AI Studio
<img width="718" alt="Screenshot 2025-06-17 at 12 08 12 PM" src="https://github.com/user-attachments/assets/e4d08422-6c83-4dcd-9355-fae3dda5dcf8" />

Then select Create Prompt on the left, I right click it and select open link in new tab twice. One tab for grounding and one tab for not.

<img width="916" alt="Screenshot 2025-06-17 at 12 08 20 PM" src="https://github.com/user-attachments/assets/871a84c1-d406-4b84-84a4-e56d10599bbb" />

In the grounding tab, configure grounding Your Data  and select Elasticsearch. This will bring up a flyover where you add in

<img width="452" alt="Screenshot 2025-06-17 at 4 28 41 PM" src="https://github.com/user-attachments/assets/f397d8b1-ca0b-47c0-9285-00cccda8e4b8" />

Elasticsearch host
```
Api Key (prepend ApiKey in front of the key as follows ApiKey sadlk9w-2wjalksjd)
Index name: my-index
Search Template: google-template-elser
```
And click save
<br>

In the system prompt for grounded, enter something like this
```
“you are a helpful assistant who is grounded with data from Elasticsearch. If Elasticsearch doesn't know about it, neither do you. Just reply that you don't have enough information”
```
Then in both tabs (grounded and non grounded) ask
```
What is Pontiac
```
and
```
What is Buick
```
For questions above, in non grounded, the LLM will answer and with Pontiac in grounded, it will list it’s sources from Elastic and for Buick it will answer that it doesn’t know.


