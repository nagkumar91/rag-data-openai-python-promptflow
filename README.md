# Copilot application that implements RAG

This is a sample copilot that application that implements RAG via custom Python code, and can be used with the Azure AI Studio. This sample aims to provide a starting point for an enterprise copilot grounded in custom data that you can further customize to add additional intelligence or capabilities.  

Following the below steps, you will: set up your development environment, create or reference your Azure AI resources, explore prompts, build an index containing product information, run your copilot, evaluate it, and deploy your copilot to a managed endpoint.

> [!IMPORTANT]
> We do not guarantee the quality of responses produced by these samples or their suitability for use in your scenarios, and responses will vary as development of the samples is ongoing. You must perform your own validation the outputs of the application and its suitability for use within your company.

## Step 1: Az login
If you haven't already done so, run `az login` to authenticate to Azure in your terminal.
    - Note: if you are running from within a Codespace or the curated VS Code cloud container, you will need to use `az login --use-device-code`


## Step 2: Reference Azure AI resources
Based on the instructions [here](https://microsoft-my.sharepoint.com/:w:/p/mesameki/Ed5UKepTDSpCpUCwigrxFrsBKMBZrEugqhSrosnz8jtdZQ?e=cudeiv), you already have everything you need. Navigate to your hub and project, click on "Settings" from the left menu, scroll down to "Connected Resource" and click on "View all". We need the information here to fill some of the details of our yaml file below. Open your ./provisioning/provision.yaml file and let's fill it together step by step:

### For the section under "ai":

Under your AI Studio project's "Settings" tab, there is a section called "Project properties". Copy paste all the info you need from there into this part of the yaml file. Note that:
- "hub_name": copy paste what you see under "hub resource name" in the UI 
- "project_name"= The string under field "Name" in the UI

### For the section under "aoai":
Click on "Settings" from the left menu of Azure AI Studio, scroll down to "Connected Resource" and click on "View all". Click on the table row whose type is "Azure OpenAI". Once opened:

- aoai_resource_name: What comes under "Resource" in your table
- kind: "OpenAI" (keep it as is)
- connection_name: Title of the page (written above "Connection Details")
### For the section under "deployments":

Click on the "Deployments" tab from the left menu of Azure AI Studio. If you followed all the steps in the workshop guide doc, you already have two deployments here. One embedding model and one GPT model. Insert information from that table here (the table has column headers name, model name, and version. Exactly what you will use here):

- name: from your Deployments table, copy what is under "name". Example: "gpt-4" 

  model: from your Deployments table, copy what is under "model name". Example: "gpt-4"

  version: from your Deployments table, copy what is under "Model version". Example: 1106.
  
  Repeat this for your embedding model:

- name: from your Deployments table, copy what is under "name"/ Example: "text-embedding-ada-002"
  
  model: from your Deployments table, copy what is under "model name". Example: "gpt-4""text-embedding-ada-002"

  version: from your Deployments table, copy what is under "Model version". Example: "2" # if you don't know, comment this line and we'll pick default
### For the section under "search":
Click on "Settings" from the left menu of Azure AI Studio, scroll down to "Connected Resource" and click on "View all". Click on the table row whose type is "Azure AI Search (Cognitive Search)". Once opened:

- search_resource_name: What comes under "Resource" in your table
- connection_name: Title of the page (written above "Connection Details")


Once you set up those parameters, run:

    ```bash
    # Note: make sure you run this command from the src/ directory so that your .env is written to the correct location (src/)
    cd src
    python provisioning/provision.py --export-env .env

    ```

The script will check whether the resources you specified exist, otherwise it will create them. It will then construct a .env for you that references the provisioned or referenced resources, including your keys. Once the provisioning is complete, you'll be ready to move to step 3.

## Step 3: Create an index

Our goal is to ground the LLM in our custom data (located in src > indexing > data > product-info). To do this, we will use promptflow to create a search index based on the specified product data.

### Step 3a: Create a new index

The following is a script to streamline index creation. It build the search index locally, and publishes it to your AI Studio project in the cloud.

``` bash
python -m indexing.build_index --index-name <desired_index_name> --path-to-data=indexing/data/product-info
```

You can view and use the index you just created on the **Indexes** page of your Azure AI Studio project.

### Step 3b: Set the index reference

NOTE: **Once you have the index you want to use, add the below entry to your .env file.** Note that the copilot code relies on this environment variable.

``` text
AZUREAI_SEARCH_INDEX_NAME=<index-name>
```


## Step 4: Use prompt flow to test copilot code

This sample includes custom code to add retrieval augmented generation (RAG) capabilities to a basic chat application.

The code implements the following general logic:

1. Generate a search query based on user query intent and any chat history
1. Use an embedding model to embed the query
1. Retrieve relevant documents from the search index, given the query
1. Pass the relevant context to the Azure Open AI chat completion model
1. Return the response from the Azure Open AI model

You can modify this logic as appropriate to fit your use case.

Use prompt flow's testing capability to validate how your copilot performs as expected on sample inputs.

``` bash
pf flow test --flow ./copilot_flow --inputs chat_input="how much do the Trailwalker shoes cost?"
```

You can use the `--ui` flag to test interactively with a sample chat experience. Prompt flow locally serves a front end integrated with your code.



## Step 5: Evaluate copilot performance

Evaluation is a key part of developing a copilot application. Once you have validated your logic on a sample set of inputs, its time to test it on a larger set of inputs.

Evaluation relies on an evaluation dataset. In this case, we have an evaluation dataset with chat_input, and then a target function that adds the LLM response and context to the evaluation dataset before running the evaluations.

Running evaluation logs traces to cloud. Make sure you have logged in Azure CLI (az login, refer to Azure CLI doc for more informations) before execute below CLI command:
``` bash
pf config set trace.destination=azureml://subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/Microsoft.MachineLearningServices/workspaces/<project-name>
```
The following script streamlines the evaluation process. Update the evaluation code to set your desired evaluation metrics, or optionally evaluate on custom metrics. You can also change where the evaluation results get written to.

``` bash
python -m evaluation.evaluate --evaluation-name <evaluation_name>
```
Examples:
This command generates evaluations on a much larger test set and generates some built-in quality metrics such as groundedness and relevance, as well as a custom evaluator called "friendliness". Learn more about our built-in quality metrics [here](https://learn.microsoft.com/en-us/azure/ai-studio/concepts/evaluation-metrics-built-in?tabs=warning#generation-quality-metrics).
``` bash
python -m evaluation.evaluate  --evaluation-name quality_evals_contoso_retail  --dataset-path=./evaluation/ContosoTestBuild.jsonl
```
This command generates one single custom evaluator called "Completeness" on a much larger test set.
``` bash
python -m evaluation.evaluate_completeness  --evaluation-name completeness_evals_contoso_retail  --dataset-path=./evaluation/evaluation_dataset.jsonl --cot
```
This commands generates evaluations on an adversarial dataset generated via our simulator. You can try the simulator by running evaluation/simulate_and_evaluate_online_endpoints.py. The command below uses the adversarial data to generate our four safety metrics. Learn more about our built-in safety metrics [here](https://learn.microsoft.com/en-us/azure/ai-studio/concepts/evaluation-metrics-built-in?tabs=warning#risk-and-safety-metrics).

``` bash
python -m evaluation.evaluatesafetyrisks --evaluation-name safety_evals_contoso_retail_jailbreak  --dataset-path=./evaluation/adversarial_questions_jailbreak.jsonl
```


We recommend viewing your evaluation results in the Azure AI Studio, to compare evaluation runs with different prompts, or even different models. The _evaluate.py_ script is set up to log your evaluation results to your AI Studio project. 

> [!NOTE]
> This will configure your project with a Cosmos DB account for logging. It may take several minutes the first time you run an evaluation.


If you do not want to log evaluation results to your AI Studio project, you can modify the _evaluation.py_ script to not pass the azure_ai_project parameter.

## Step 6: Deploy application to AI Studio

Use the deployment script to deploy your application to Azure AI Studio. This will deploy your app to a managed endpoint in Azure, that you can test, integrate into a front end application, or share with others.

You can make any changes to deployment specifications to fit your use case.
> [!NOTE]
> If you made any custom changes to your .env not covered in this README, make sure you reference them in the deploy.py script before you deploy so that they are available in the deployed environment.

``` bash
python -m deployment.deploy --endpoint-name <endpoint_name> --deployment-name <deployment_name>
```

If you get a quota error for the VM size during deployment, you can check VM availability on the Azure ML tab of the Quota page in the Azure AI Studio (the quota page is located on the home page left nav).

- You can then update the instance_type specified in the deploy.py script to match a size you have quota for. There is a comment with [helpful documentation for supported instance types for deployment](https://learn.microsoft.com/en-us/azure/machine-learning/reference-managed-online-endpoints-vm-sku-list?view=azureml-api-2).

Once you create an endpoint, you can re-deploy or update an existing deployment on that same endpoint.

## Step 7: Verify your deployment

We recommend you test your application in the Azure AI Studio. The previous step outputted a handy link to your deployment. If you don't use the link, simply navigate to the Deployments tab in your project, and select your new deployment.

Navigate to the Test tab, and try asking a question in the chat interface. You should see the response come back and you have verified your deployment!

If you prefer to test your deployed endpoint locally, you can invoke it with a default query.

``` bash
python -m deployment.invoke --endpoint-name <endpoint_name> --deployment-name <deployment_name>
```

The invoke script uses a default query, but you can also pass your own query with the `--query` argument in the command above.
