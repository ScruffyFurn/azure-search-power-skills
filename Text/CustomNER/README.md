# custom_ner_skill

---

Description:

- It is common to have custom entities along different texts that don't fit any of the predefined entities that can be extracted with the Named Entity Extraction service. Custom Named Entity Recognition provides the capability to ingest your training texts, label your set of custom entities and train a model to identify them. You can easily deploy the model in a secured fashion to later on run your inference along your texts. As an outcome you will get the detected custom entities, their position (inside the text) and the confidence level

- custom_ner_skill is an Azure Cognitive Search skill to integrate Azure Text Analytics Custom Named Entity Recognition within a Azure Cognitive Search skillset. This will enable the cracking of documents in a programmatic way to enrich your search with different custom entities. For example, show me the loan documents signed with the credit institution X between May and June 2021 with higher purchase amount than one million dollars. This filtering is possible because Text Analytics has identified all those fields along the skillset execution and exposes the ability to narrow the results within the ACS index.

Languages:

- ![python](https://img.shields.io/badge/language-python-orange)

Products:

- Azure Cognitive Search
- Azure Cognitive Services for Language (Text Analytics)
- Azure Functions

Table of Contents:

* [Steps](#steps)
  
  - [Create or reuse a Custom NER project](#create-or-reuse-a-custom-ner-project)
  - [Create a Function App resource and deploy the powerskill to Azure](#create-a-function-app-resource-and-deploy-the-powerskill-to-Azure)
    
    - [ARM Deployment](#arm-deployment)
    
    - [Manual Deployment](#manual-deployment)
      
      - [Create a Function App](#create-a-function-app)
      
      - [Deploy the powerskill](#deploy-the-powerskill)
    
    - [Additional details](#additional-details)
  
  - [Integrate with Azure Cognitive Search](integrate-with-azure-cognitive-search)
    
    - [Skillset](#skillset)
    
    - [Index](#index)
    
    - [Indexer](#indexer)
    
    - [Query the index](#query-the-index)

- [Testing](#testing)

---

# Steps

#### Create or reuse a Custom NER project

In order to use Custom NER, we need a language resource and a trained (and deployed) project to be used in recognizing the custom entities. If they aren't previously created, now is the time to do that. A good place to start is [Quickstart - Custom named entity recognition (NER) - Azure Cognitive Services | Microsoft Docs](https://docs.microsoft.com/en-us/azure/cognitive-services/language-service/custom-named-entity-recognition/quickstart?pivots=language-studio).

After this step you should have:

* A Language resource.

* A project

* A deployment

#### Create a Function App resource and deploy the powerskill to Azure

A powerskill is basically just an Azure Function written to be used as a custom skill in an Azure Cognitive Search pipeline. To deploy a function, an Azure App resource is needed. Notice the difference between a function (code that can be deployed, like the one in this subrepo) and an Azure Function App (the Azure resource that a function can be deployed to). An ARM template (see [Templates overview - Azure Resource Manager | Microsoft Docs](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview)) is provided to automate creating the Function App resource and deploying this powerskill on the resource. Alternatively, one may choose to manually create a Function App resource (for example, from the Azure portal), clone the repository, and deploy the code from VSCode. Both options are explored below.

##### ARM deployment

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure-Samples%2Fazure-search-power-skills%2Fmain%2FText%2FCustomNER%2Fazuredeploy.json)

This ARM template provides a streamlined way to both create a Function App resource and deploy the powerskill. The template takes the following parameters

| App setting          | Description                                                         | Details                                                                                                          |
| -------------------- | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Function App Name    | The name of the created Function App resource                       | A default value of `nerfunc<a-generated-unique-id>` is provided, but can be set to any other unique name.        |
| Storage Account Name | The name of the created Function App resource                       | A default value of `nerstor<the-same-generated-unique-id>` is provided, but can be set to any other unique name. |
| Storage Account Type | The type of the storage account                                     | `Standard_LRS`, `Standard_GRS` or `Standard_RAGRS`                                                               |
| Location             | The location of all created resources                               | `West US`, `Central US`, etc. Uses the resource group location by default.                                       |
| Package Uri          | A link to the zipped function                                       | See below. By default, uses the zip file in the repository.                                                      |
| Language Endpoint    | The endpoint of the Language resource created in the previous step. | Can be found under `Resource Management -> Keys and Endpoint` in the Language resource page in Azure Portal.     |
| Language Key         | The key of the Language resource.                                   | Can be found under `Resource Management -> Keys and Endpoint` in the Language resource page in Azure Portal.     |
| Project Name         | The name of the project in the Language resource.                   |                                                                                                                  |
| Deployment Name      | The name of the deployment in the Language resource.                |                                                                                                                  |

Due to limitations in ARM templates, this ARM template cannot grab the source code for the function project automatically from Github. Instead, the user needs to package the project in a Zip format, upload it somewhere accessible to Azure, and provide a link to the uploaded Zip file to the template. A ready-to-deploy Zip file is available inside the repository and is used by default. Alternatively, one way to create the Zip file is

* [Download](https://downgit.github.io/#/home?url=https:%2F%2Fgithub.com%2FAzure-Samples%2Fazure-search-power-skills%2Ftree%2Fmain%2FText%2FCustomNER) (and extract) the source code for this powerskill using the open-source utility [DownGit](https://downgit.github.io). More experienced users might prefer to clone the repository instead.
* Zip the necessary folders/files. These are `custom_ner`, `host.json` and `requirements.txt`. On Windows, you can select the folders/files in File Explorer, right-click, and select `Compress to ZIP file` (or `Send to -> Compressed (zipped) folder` on older versions). On Linux, running `zip -r customner-powerskill.zip custom_ner host.json requirements.txt` inside the project directory should yield the same result.
* Upload the created Zip file to any file hosting service. An obvious service is Azure Blob Storage. Instructions for Blob Storage can be found [here](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-portal). Make sure to make your file publicly accessible, so that the ARM template can access it.
* Use the URL of the uploaded file in the ARM template.

The template creates the following resources:

* A Function App with the powerskill deployed on it.
* A storage account to be used by the Function App.
* A serverless Consumption hosting plan for the function app.
* An Application Insights resource for the Function App. This is useful for monitoring the Function App.

Generally, the Function App is the main resource, while the other resources are only occasionally interacted with.

##### Manual Deployment

###### Create a Function App

From the Azure portal, select `Create a resource` and search for `Function App`. The creation page should appear, click `Create`. In the new page, in the `Basics` tab, select your subscription and resource group, give your Function App a globally unique name and select a suitable region. Fill the remaining parameters as follows:

| App setting      | Value                      |
| ---------------- | -------------------------- |
| Publish          | `Code`                     |
| Runtime Stack    | `Python`                   |
| Version          | `3.9`                      |
| Operating System | `Linux`                    |
| Plan type        | `Consumption (Serverless)` |

When done, click `Review + create` and wait a few seconds while the inputs are validated. If all goes well, a red warning telling you that `You will need to Agree to the terms of service below to create this resource successfully.` should appear. To accept the terms and deploy the template, click `Create`. You should now have a Function App resource.

After creating the resource, some app settings (visible inside Azure Functions as environment variables) need to be added for the powerskill to run correctly. here's how to [change app settings in the Azure portal](https://docs.microsoft.com/en-us/azure/app-service/configure-common?tabs=portal#configure-app-settings).

| App setting     | Description                                                         | Details                                                                                                      |
| --------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| LANG_ENDPOINT   | The endpoint of the Language resource created in the previous step. | E.g. https://<language-resource-name>.cognitiveservices.azure.com/                                           |
| LANG_KEY        | The access key to be able to access the endpoint.                   | Can be found under `Resource Management -> Keys and Endpoint` in the Language resource page in Azure Portal. |
| PROJECT_NAME    | The key of the Language resource.                                   |                                                                                                              |
| DEPLOYMENT_NAME | The name of the deployment in the Language resource.                |                                                                                                              |

###### Deploy the powerskill

In this set of instructions, VSCode is used to deploy the function following this [quickstart](https://docs.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python). For more information on deployment methods, see [Deployment technologies in Azure Functions | Microsoft Docs](https://docs.microsoft.com/en-us/azure/azure-functions/functions-deployment-technologies).

First, we need to download the source code for the powerskill. [Download](https://downgit.github.io/#/home?url=https:%2F%2Fgithub.com%2FAzure-Samples%2Fazure-search-power-skills%2Ftree%2Fmain%2FText%2FCustomNER) (and extract) the source code for this powerskill using the open-source utility [DownGit](https://downgit.github.io). More experienced users might prefer to clone the repository instead.

Afterwards, make sure you have [the following requirements](https://docs.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python#configure-your-environment). If so, open the downloaded directory in VSCode (or its equivalent location in a cloned repo). [Sign into Azure](https://docs.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python#sign-in-to-azure) and [deploy the project to Azure](https://docs.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python#deploy-the-project-to-azure). Select the previously created Function App when prompted. The powerskill project should now be deployed and running.

##### Additional details

The function adheres to the input/output format specified by Azure Cognitive Search for custom skills. More information about custom skills and format, see [Custom skill interface - Azure Cognitive Search | Microsoft Docs](https://docs.microsoft.com/en-us/azure/search/cognitive-search-custom-skill-interface). Sample inputs and outputs for this powerskill are shown below. Notice that specifying the language is optional, and defaults to English.

```json
{
    "values": [
      {
        "recordId": "0",
        "data":
           {
            "text":"Date 10/18/2019\n\nThis is a Loan agreement between the two individuals mentioned below in the parties section of the agreement.\n\nI. Parties of agreement\n\n- Casey Jensen with a mailing address of 2469 Pennsylvania Avenue, City of New Brunswick, State of New Jersey (the \"Borrower\")\n- Hollie Rees with a mailing address of 42 Gladwell Street, City of Memphis, State of Tennessee (the \"Lender\")\n\nII. Amount\nThe loan amount given by lender to borrower is one hundred ninety-two thousand nine hundred eighty-nine Dollars ($192,989.00) (\"The Note\")\n\nIII. Interest\nThe Note shall bear interest five percent (5%) compounded annually.\n\nIV. Payment\nThe amount mentioned in this agreement (the \"Note\"), including the principal and any accrued interest, is\n\nV. Payment Terms\nAny delay in payment is subject to a fine with a flat amount of $50 for every week the payment is delayed.\nAll payments made by the Borrower shall be go into settling the the accrued interest  and any late fess and then into the payment of the principal amount.\n\nVI. Prepayment\nThe borrower is able to pay back the Note in full at any time, thus terminating this agreement.\nThe borrower also can make additional payments at any time and this will take of from the amount of the latest installments. \n\nVII. Acceleration.\nIn case of Borrower's failure to pay any part of the principal or interest as and when due under this Note; or Borrower's becoming insolvent or not paying its debts as they become due. The lender has the right to declare an \"Event of Acceleration\" in which case the Lender has the right to to declare this Note immediately due and payable \n\nIX. Succession\nThis Note shall outlive the borrower and/or the lender in the even of their death. This note shall be binging to any of their successors.",
            "language":"en"
           }
      }
    ]
}
```

```json
{
    "values": [
        {
            "recordId": "0",
            "data": {
                "entities": [
                    {
                        "text": "10/18/2019",
                        "category": "Date",
                        "subcategory": null,
                        "length": 10,
                        "offset": 5,
                        "confidence_score": 1.0
                    },
                    {
                        "text": "Casey Jensen",
                        "category": "BorrowerName",
                        "subcategory": null,
                        "length": 12,
                        "offset": 155,
                        "confidence_score": 1.0
                    },
                    {
                        "text": "2469 Pennsylvania Avenue",
                        "category": "BorrowerAddress",
                        "subcategory": null,
                        "length": 24,
                        "offset": 194,
                        "confidence_score": 0.99
                    },
                    {
                        "text": "New Brunswick",
                        "category": "BorrowerCity",
                        "subcategory": null,
                        "length": 13,
                        "offset": 228,
                        "confidence_score": 0.95
                    },
                    {
                        "text": "New Jersey",
                        "category": "BorrowerState",
                        "subcategory": null,
                        "length": 10,
                        "offset": 252,
                        "confidence_score": 0.81
                    },
                    {
                        "text": "Hollie Rees",
                        "category": "LenderName",
                        "subcategory": null,
                        "length": 11,
                        "offset": 282,
                        "confidence_score": 1.0
                    },
                    {
                        "text": "42 Gladwell Street",
                        "category": "LenderAddress",
                        "subcategory": null,
                        "length": 18,
                        "offset": 320,
                        "confidence_score": 1.0
                    },
                    {
                        "text": "Memphis",
                        "category": "LenderCity",
                        "subcategory": null,
                        "length": 7,
                        "offset": 348,
                        "confidence_score": 1.0
                    },
                    {
                        "text": "Tennessee",
                        "category": "LenderState",
                        "subcategory": null,
                        "length": 9,
                        "offset": 366,
                        "confidence_score": 1.0
                    },
                    {
                        "text": "one hundred ninety-two thousand nine hundred eighty-nine Dollars",
                        "category": "LoanAmountWords",
                        "subcategory": null,
                        "length": 64,
                        "offset": 450,
                        "confidence_score": 1.0
                    },
                    {
                        "text": "$192,989.00",
                        "category": "LoanAmountNumbers",
                        "subcategory": null,
                        "length": 11,
                        "offset": 516,
                        "confidence_score": 1.0
                    },
                    {
                        "text": "5%",
                        "category": "Interest",
                        "subcategory": null,
                        "length": 2,
                        "offset": 600,
                        "confidence_score": 1.0
                    }
                ]
            },
            "warnings": []
        }
    ]
}
```

#### Integrate with Azure Cognitive Search

To finally be able to use the deployed powerskill, an Azure Cognitive Search resource should be available. If not, now is the time to create one. For instructions, see [Create a search service in the portal - Azure Cognitive Search | Microsoft Docs](https://docs.microsoft.com/en-us/azure/search/search-create-service-portal).

###### Skillset

An Azure Cognitive Search pipeline consists of an Index, an Indexer, a skillset and data source. This function can be used as a custom skill in a skillset (either as a singular custom skill in a skillset, or as a skill among many others in a skillset). To add this function as a custom skill, some parameters need to be specified, including the the endpoint URL of the Function App deployed in the previous steps, the `x-functions-key` header, what is needed as input, and what the output is named. An example is shown below. Here, the text of each document (`/document/content`) is sent to the api as a value for the key named `text` (as the function expects) and the output is the value of the key named `entities` in the response. An example of what a skillset may look like is shown below. For more information on skillsets, see [Skillset concepts - Azure Cognitive Search | Microsoft Docs](https://docs.microsoft.com/en-us/azure/search/cognitive-search-working-with-skillsets).

```json
{
    "skills": [
      "[... your existing skills remain here]",  
      {
        "@odata.type": "#Microsoft.Skills.Custom.WebApiSkill",
        "name": "customner-skill",
        "description": "",
        "context": "/document",
        "uri": "https://<azure-function-name>.azurewebsites.net/api/custom_ner",
        "httpMethod": "POST",
        "timeout": "PT30S",
        "batchSize": 1,
        "degreeOfParallelism": 1,
        "inputs": [
          {
            "name": "text",
            "source": "/document/content"
          }
        ],
        "outputs": [
          {
            "name": "entities",
            "targetName": "entities"
          }
        ],
        "httpHeaders": {
          "x-functions-key": "<azure-function-key>"
        }
      }
  ]
}
```

###### Index

Next, the output from the custom skill can be used as an input to yet another skill, or be part of the final output that is saved into the index. Assuming the latter, the index needs to have a field definition that matches the output it's given. An example for what the definition should look like is shown below. For more information on search indexes, see [Index overview - Azure Cognitive Search | Microsoft Docs](https://docs.microsoft.com/en-us/azure/search/search-what-is-an-index).

```json
{
      "name": "entities",
      "type": "Collection(Edm.ComplexType)",
      "fields": [
        {
          "name": "text",
          "type": "Edm.String",
          "searchable": true,
          "filterable": false,
          "retrievable": true,
          "sortable": false,
          "facetable": false,
          "key": false,
          "indexAnalyzer": null,
          "searchAnalyzer": null,
          "analyzer": "standard.lucene",
          "normalizer": null,
          "synonymMaps": []
        },
        {
          "name": "category",
          "type": "Edm.String",
          "searchable": true,
          "filterable": false,
          "retrievable": true,
          "sortable": false,
          "facetable": false,
          "key": false,
          "indexAnalyzer": null,
          "searchAnalyzer": null,
          "analyzer": "standard.lucene",
          "normalizer": null,
          "synonymMaps": []
        },
        {
          "name": "subcategory",
          "type": "Edm.String",
          "searchable": true,
          "filterable": false,
          "retrievable": true,
          "sortable": false,
          "facetable": false,
          "key": false,
          "indexAnalyzer": null,
          "searchAnalyzer": null,
          "analyzer": "standard.lucene",
          "normalizer": null,
          "synonymMaps": []
        },
        {
          "name": "length",
          "type": "Edm.Int64",
          "searchable": false,
          "filterable": true,
          "retrievable": true,
          "sortable": false,
          "facetable": false,
          "key": false,
          "indexAnalyzer": null,
          "searchAnalyzer": null,
          "analyzer": null,
          "normalizer": null,
          "synonymMaps": []
        },
        {
          "name": "offset",
          "type": "Edm.Int64",
          "searchable": false,
          "filterable": true,
          "retrievable": true,
          "sortable": false,
          "facetable": false,
          "key": false,
          "indexAnalyzer": null,
          "searchAnalyzer": null,
          "analyzer": null,
          "normalizer": null,
          "synonymMaps": []
        },
        {
          "name": "confidence_score",
          "type": "Edm.Double",
          "searchable": false,
          "filterable": true,
          "retrievable": true,
          "sortable": false,
          "facetable": false,
          "key": false,
          "indexAnalyzer": null,
          "searchAnalyzer": null,
          "analyzer": null,
          "normalizer": null,
          "synonymMaps": []
        }
      ]
    }
```

###### Indexer

Finally, the indexer ties everything together. The indexer needs to be setup up such that the outputs from the custom skill are mapped to the field that was just defined. Notice the `outputFieldMappings` key in the example shown below, the content of each document is mapped to the a field named `textBody`, and the output from the powerskill (that was named `entities` in the skillset) is mapped to a field named `entities` in the index. For more information on output mappings, see [Map skill output fields - Azure Cognitive Search | Microsoft Docs](https://docs.microsoft.com/en-us/azure/search/cognitive-search-output-field-mapping) in addition to the previously linked article about skillsets.

```json
{
  "@odata.context": "...",
  "@odata.etag": "...",
  "name": "<indexer-name>",
  "dataSourceName": "<datasource-name>",
  "skillsetName": "<skillset-name>",
  "targetIndexName": "<index-name>",
  ...
  "outputFieldMappings": [
    {
      "sourceFieldName": "/document/content",
      "targetFieldName": "textBody"
    },
    {
      "sourceFieldName": "/document/entities",
      "targetFieldName": "entities"
    }
  ]
}
```

#### Query the index

Now that we have a populated index, we can use the capabilities of the index to make useful queries. For example, using the sample data for loan agreements from the custom NER documentation (found [here](https://github.com/Azure-Samples/cognitive-services-sample-data-files/tree/master/language-service/Custom%20NER)), we can query the index to find all loan agreements on a specific date. For example,

```
$queryType=full&$searchMode=all&$count=true&$filter=entities/any(e: e/text eq '6/28/2019')&$select=content,entities/text,entities/category
```

will get all loan agreements on `6/28/2019` and only returns specific fields so as not to clutter the output. The response should look like the following.

```json
{
  "@odata.context": "...",
  "@odata.count": 3,
  "value": [
    {
      "@search.score": 1,
      "content": "Date 6/28/2019\r\n\r\nThis is a Loan agreement between the two individuals mentioned below in the parties section of the agreement.\r\n\r\nI. Parties of agreement\r\n\r\n- Jennifer Wilkins with a mailing address of 4759 Reel Avenue, City of Las Cruces, State of New Mexico (the \"Borrower\")\r\n- Libby Harrison with a mailing address of 3093 Keyser Ridge Road, City of Greensboro, State of North Carolina (the \"Lender\")\r\n\r\nII. Amount\r\nThe loan amount given by lender to borrower is two hundred thirty-eight thousand eight hundred sixty-nine Dollars ($238,869.00) (\"The Note\")\r\n\r\nIII. Interest\r\nThe Note shall bear interest five percent (3%) compounded annually.\r\n\r\nIV. Payment\r\nThe amount mentioned in this agreement (the \"Note\"), including the principal and any accrued interest, is\r\n\r\nV. Payment Terms\r\nAny delay in payment is subject to a fine with a flat amount of $50 for every week the payment is delayed.\r\nAll payments made by the Borrower shall be go into settling the the accrued interest  and any late fess and then into the payment of the principal amount.\r\n\r\nVI. Prepayment\r\nThe borrower is able to pay back the Note in full at any time, thus terminating this agreement.\r\nThe borrower also can make additional payments at any time and this will take of from the amount of the latest installments. \r\n\r\nVII. Acceleration.\r\nIn case of Borrower's failure to pay any part of the principal or interest as and when due under this Note; or Borrower's becoming insolvent or not paying its debts as they become due. The lender has the right to declare an \"Event of Acceleration\" in which case the Lender has the right to to declare this Note immediately due and payable \r\n\r\nIX. Succession\r\nThis Note shall outlive the borrower and/or the lender in the even of their death. This note shall be binging to any of their successors.\n",
      "entities": [
        {
          "text": "6/28/2019",
          "category": "Date"
        },
        {
          "text": "Jennifer Wilkins",
          "category": "BorrowerName"
        },
        {
          "text": "4759 Reel Avenue",
          "category": "BorrowerAddress"
        },
        {
          "text": "Las Cruces",
          "category": "BorrowerCity"
        },
        {
          "text": "New Mexico",
          "category": "BorrowerState"
        },
        {
          "text": "Libby Harrison",
          "category": "LenderName"
        },
        {
          "text": "3093 Keyser Ridge Road",
          "category": "LenderAddress"
        },
        {
          "text": "Greensboro",
          "category": "LenderCity"
        },
        {
          "text": "North Carolina",
          "category": "LenderState"
        },
        {
          "text": "two hundred thirty-eight thousand eight hundred sixty-nine Dollars",
          "category": "LoanAmountWords"
        },
        {
          "text": "$238,869.00",
          "category": "LoanAmountNumbers"
        },
        {
          "text": "(3%)",
          "category": "Interest"
        }
      ]
    },
    {
      "@search.score": 1,
      "content": "Date 6/28/2019\r\n\r\nThis is a Loan agreement between the two individuals mentioned below in the parties section of the agreement.\r\n\r\nI. Parties of agreement\r\n\r\n- William Kirby with a mailing address of 4433 Oakmound Road, City of Burr Ridge, State of Illinois (the \"Borrower\")\r\n- Quinn Anderson with a mailing address of 458 Frank Avenue, City of Crafton, State of Pennsylvania (the \"Lender\")\r\n\r\nII. Amount\r\nThe loan amount given by lender to borrower is seven hundred twenty thousand sixty-seven Dollars ($720,067.00) (\"The Note\")\r\n\r\nIII. Interest\r\nThe Note shall bear interest five percent (4%) compounded annually.\r\n\r\nIV. Payment\r\nThe amount mentioned in this agreement (the \"Note\"), including the principal and any accrued interest, is\r\n\r\nV. Payment Terms\r\nAny delay in payment is subject to a fine with a flat amount of $50 for every week the payment is delayed.\r\nAll payments made by the Borrower shall be go into settling the the accrued interest  and any late fess and then into the payment of the principal amount.\r\n\r\nVI. Prepayment\r\nThe borrower is able to pay back the Note in full at any time, thus terminating this agreement.\r\nThe borrower also can make additional payments at any time and this will take of from the amount of the latest installments. \r\n\r\nVII. Acceleration.\r\nIn case of Borrower's failure to pay any part of the principal or interest as and when due under this Note; or Borrower's becoming insolvent or not paying its debts as they become due. The lender has the right to declare an \"Event of Acceleration\" in which case the Lender has the right to to declare this Note immediately due and payable \r\n\r\nIX. Succession\r\nThis Note shall outlive the borrower and/or the lender in the even of their death. This note shall be binging to any of their successors.\n",
      "entities": [
        {
          "text": "6/28/2019",
          "category": "Date"
        },
        {
          "text": "William Kirby",
          "category": "BorrowerName"
        },
        {
          "text": "4433 Oakmound Road",
          "category": "BorrowerAddress"
        },
        {
          "text": "Burr Ridge",
          "category": "BorrowerCity"
        },
        {
          "text": "Illinois",
          "category": "BorrowerState"
        },
        {
          "text": "Quinn Anderson",
          "category": "LenderName"
        },
        {
          "text": "458 Frank Avenue",
          "category": "LenderAddress"
        },
        {
          "text": "Crafton",
          "category": "LenderCity"
        },
        {
          "text": "Pennsylvania",
          "category": "LenderState"
        },
        {
          "text": "seven hundred twenty thousand sixty-seven Dollars",
          "category": "LoanAmountWords"
        },
        {
          "text": "$720,067.00",
          "category": "LoanAmountNumbers"
        },
        {
          "text": "4%",
          "category": "Interest"
        }
      ]
    },
    {
      "@search.score": 1,
      "content": "Date 6/28/2019\r\n\r\nThis is a Loan agreement between the two individuals mentioned below in the parties section of the agreement.\r\n\r\nI. Parties of agreement\r\n\r\n- Andre Lawson with a mailing address of 4759 Reel Avenue, City of Las Cruces, State of New Mexico (the \"Borrower\")\r\n- Malik Barden with a mailing address of 3093 Keyser Ridge Road, City of Greensboro, State of North Carolina (the \"Lender\")\r\n\r\nII. Amount\r\nThe loan amount given by lender to borrower is eight two hundred thirty-eight thousand eight hundred sixty-nine Dollars ($238,869.00) (\"The Note\")\r\n\r\nIII. Interest\r\nThe Note shall bear interest five percent (7%) compounded annually.\r\n\r\nIV. Payment\r\nThe amount mentioned in this agreement (the \"Note\"), including the principal and any accrued interest, is\r\n\r\nV. Payment Terms\r\nAny delay in payment is subject to a fine with a flat amount of $50 for every week the payment is delayed.\r\nAll payments made by the Borrower shall be go into settling the the accrued interest  and any late fess and then into the payment of the principal amount.\r\n\r\nVI. Prepayment\r\nThe borrower is able to pay back the Note in full at any time, thus terminating this agreement.\r\nThe borrower also can make additional payments at any time and this will take of from the amount of the latest installments. \r\n\r\nVII. Acceleration.\r\nIn case of Borrower's failure to pay any part of the principal or interest as and when due under this Note; or Borrower's becoming insolvent or not paying its debts as they become due. The lender has the right to declare an \"Event of Acceleration\" in which case the Lender has the right to to declare this Note immediately due and payable \r\n\r\nIX. Succession\r\nThis Note shall outlive the borrower and/or the lender in the even of their death. This note shall be binging to any of their successors.\n",
      "entities": [
        {
          "text": "6/28/2019",
          "category": "Date"
        },
        {
          "text": "Andre Lawson",
          "category": "BorrowerName"
        },
        {
          "text": "4759 Reel Avenue",
          "category": "BorrowerAddress"
        },
        {
          "text": "Las Cruces",
          "category": "BorrowerCity"
        },
        {
          "text": "New Mexico",
          "category": "BorrowerState"
        },
        {
          "text": "Malik Barden",
          "category": "LenderName"
        },
        {
          "text": "3093 Keyser Ridge Road",
          "category": "LenderAddress"
        },
        {
          "text": "Greensboro",
          "category": "LenderCity"
        },
        {
          "text": "North Carolina",
          "category": "LenderState"
        },
        {
          "text": "eight two hundred thirty-eight thousand eight hundred sixty-nine Dollars",
          "category": "LoanAmountWords"
        },
        {
          "text": "$238,869.00",
          "category": "LoanAmountNumbers"
        },
        {
          "text": "7%",
          "category": "Interest"
        }
      ]
    }
  ]
}
```

For more details and examples, see [Use full Lucene query syntax - Azure Cognitive Search | Microsoft Docs](https://docs.microsoft.com/en-us/azure/search/search-query-lucene-examples) and [Filter on search results - Azure Cognitive Search | Microsoft Docs](https://docs.microsoft.com/en-us/azure/search/search-filters).

## Testing

A small test suite is provided in the `tests` directory. The tests assume a `.env` file that contains the parameters that the powerskill needs to run. The file uses a simple `KEY=VAL` syntax separated by newlines.

```
PROJECT_NAME=<project-name>
DEPLOYMENT_NAME=<deployment-name>
LANG_KEY=<language-resource-key>
LANG_ENDPOINT=https://<language-resource-name>.cognitiveservices.azure.com
```

The test cases assume that the model is trained to recognize load agreements like the example in [Quickstart - Custom named entity recognition (NER) - Azure Cognitive Services | Microsoft Docs](https://docs.microsoft.com/en-us/azure/cognitive-services/language-service/custom-named-entity-recognition/quickstart?pivots=language-studio). Training and test data can be found [here](https://github.com/Azure-Samples/cognitive-services-sample-data-files/tree/master/language-service/Custom%20NER).

To run the tests, `cd` into the root of the project and run `unittest`.

```bash
cd CustomNER
python -m unittest
```
