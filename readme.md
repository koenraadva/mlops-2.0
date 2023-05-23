# MLOps - Automating AI projects

:::info
You can find the questions to this assignment in **[here](/MyiK4mWyRYauLyaDokvlfg)**. Use that page if you want to hand in this assignment.
:::

:::warning
This assignment doesn't contain a separate recorded theoretical session in PowerPoint format. All the required information has been written down in this document.
So it is important to also study this for the theoretical exam.
:::


## Introduction

In the previous assignment we have learned how to work with Azure in order to create AI projects in a cloud environment.
This offered us the benefit of ordering AI training machines for a short period of time to train an AI model with stronger machines.
Azure Machine Learning services also offered us integrations for data storage and model versioning. It was possible to deploy an AI model automatically from the command line or the UI, and get an Endpoint from Azure (which has been included in the code, but not executed due to the high costs), or include it in a FastAPI project like in the first assignment.

In this assignment, we will try to set up an automation pipeline which will be triggered on code changes or on manual starts. We can configure our automation pipeline to run on our own (virtual) machine, or a cloud machine from GitHub, Microsoft ...

It will be a little bit different than executing Jupyter Notebook cells, as our code will not allow any manual input during the execution.

This assignment has been split up into three stages.

In the first stage, you will explore the use-case of MLOps by using the GitHub Actions way and by re-using the codebase of the previous assignment.

In the second stage, we will explore a newly introduced way of using Azure Machine Learning pipelines, reusable components and pipeline triggers all from the Azure Machine Learning portal.

Then in the final stage, it's your time to connect the Azure Machine Learning pipeline into the GitHub Actions using the Azure CLI.

All the steps will be explained in depth in the following topics, but first of all, we need to dive into Stage 0, a little bit of theory.

# Stage 0, the theory behind MLOps

## CI/CD background information

In our third assignment, regarding the Kubernetes Automation, we learned about CI/CD pipelines, which we had then set up using GitHub Actions and our own custom Action Runners.

CI means **Continuous Integration** which means that our code will be continuously checked for errors and mistakes, and will always be **built** to be up-to-date and ready for further checks. These build steps will usually mean that code will be compiled into executables. **Docker images** are very comparable to executables as we can run them to create a Docker **Container**.

> **Thus, Docker building is also part of our CI pipeline.  
> This will be important to set up in that way later on.**

After the Docker building, we should perform some further checks including **Integration testing**. Essentially, this will check if the code inside your Docker container still functions together in a real environment. Is everything still compatible with other code? Examples include:
- A check to see if you can still upload a picture to your FastAPI code, and that the image can be processed by your AI model.
- A check to see if your code can read out information from a database (only seen in the extended versions)

*We will not be using these Integration tests in our MLOps pipeline, for more information you can always search for other tutorials about that if you want to.*

### CI/CD for MLOps ...

After we have done some extensive testing, we can proceed with the further steps of our MLOps pipeline. We have already entered our CD pipeline with the integration tests. But there is some more to do now. In a next part of the pipeline, we are going to deploy our AI model into a **staging** environment. Essentially, this means that we are going to deploy our code into an environment **very similar** to the actual production environment. In other words, we are going to install the same dependencies on the staging environment, as well as having the same hardware specs as our production environment.
This way, we are quite sure that our project will also run well in the production environment, where the actual users are going to use this.
We can use the staging setup for **alpha/beta/preview** tests as well as insider previews. Usually, the customer you are creating the project for will want to test this as well.
We're almost there, but for the final step comes the difference between the two meanings for CD.

1) Continuous **Delivery** will require manual approval to go to the actual production stage.
2) Continuous **Deployment** will automatically go to the production stage according to a fixed set of checks it goes through.

In our implementation, we will make sure to deploy our AI model as a FastAPI into a production environment and update it with every new run of our CI/CD pipeline. We will use **Kubernetes** as our production environment, which is something we have already seen in the earlier assignments.

What I have currently described is nothing more than a *classic* **DevOps** pipeline. We don't really have much MLOps so far, let's change that now!

### Finally, the actual MLOps!
There are little small changes to make to a DevOps pipeline to convert it to MLOps.
Most of those differences exist in the early stages of the CI part.
You might remember in the 3<sup>rd</sup> Notebook from the previous assignment, that we have packaged our AI model in a Docker Image. Well, that part was already the end of the CI pipeline.

This means that our first two notebooks, the Preprocessing and AI training should happen before the Docker building.

What we should keep in mind, is that in these pipelines all our code will be executed **automatically**. With that in mind, we have to set up our pipelines to read in some values and variables before running. This will be explained when the pipeline will be created, a little lower in this document.

The good thing about these pipelines is that Python is perfectly supported. We can set up a pipeline to run Python code without any problem.

Let's get started now that we know all of this.

# Stage 1: Setting up GitHub Actions and Azure Machine Learning studio

This stage of the assignment will explore the way to set up a GitHub Actions pipeline to connect to our Azure Machine Learning workspace using the Python SDK 1.0 (Which you used in the previous assignment).

It might feel a little rusty here and there, but it works nonetheless.

## Setting up Azure

The first thing we'll be doing is configure something on our Azure portal. Like I said, our pipelines will be running automatically, which also means that we have no manual way to log in to Azure.  
In order to do that, Azure uses **Service Principals** which you can create on your Azure Portal, and through the Azure CLI.  
This Service Principle will be used to authorize our CI/CD pipeline to work with our Azure account, and create or use Azure resources and services on our behalf. Think of the Service Principle (SP) as a seperate account that we introduce in Azure. It has it's seperate rights and access methods.

We have a few steps to perform in the Azure Portal to do this. I will explain the steps for the Azure CLI as well, that way you can choose.

### Azure Portal

:::warning
In your Azure for Students subscription, you cannot use the Azure Portal to create this App Registration. I have written this documentation in case you are ever going to use this in your own Azure setup.
Use the Azure CLI for the Azure for Students setup,
:::

- Create a new App Registration and call it `MLOps-SP-NS`. **Where `NS` stands for my initials. **If you get an error during creation, the App Registration might already be in use, then you should create a new name.
- Go to the Resource Group you used in the previous assignment and navigate to IAM (Access Control).
- Add a new Role Assignment.
- Choose the Contributor role. This allows your MLOps pipeline to access any resource into your account. <span style='font-size:0.75em'>**NOTE: If you want to customize the access, you can search for the different resources that you should allow. But as this is just a demonstration, I will not dive deeper into all the different resource roles.**</span>
- Add the `MLOps-SP-NS` App Registration as a member. This is called a Service Principal.

As you are not using your own Azure account, this does not compromise your passwords and is thus very secure. But don't forget you've given all the rights to create and delete Azure resources into your resource group.

### Azure CLI
If you do not want to perform these actions in the Azure Portal, and just want to execute one command in the Azure CLI, you could follow these instructions.

:::success
Note that you can also execute this script using the Azure CLI on your own laptop if you configured that.
:::

These commands can be executed directly in a Cloud Shell by Azure. Go to the Azure Portal once more, and press this button in the top navigation bar.
![Azure Cloud Shell](https://i.imgur.com/OYeIrsd.jpg)
Follow the instructions to activate your Cloud Shell when it's the first time you're using this.
The instructions will ask you to create a new Storage account. This doesn't cost that much, so you don't have to worry about the price.

Enter the following command which will give you a result like below.

`az ad sp create-for-rbac -n "MLOps-SP-NS" --role Contributor --scopes /subscriptions/<subID>/resourceGroups/<resourceGroupName>`
Fill in the resourceGroup name, which should be something like `mlops` like we used in the last assignment. The SubscriptionID can be found when navigating to that resource Group.

'MLOps-SP-NS' has NS which stands for my initials. If you get an error during creation, the App Registration might already be in use, then you should create a new name.

As a result we get this:
```text
{
    "appId": "<appId>",
    "displayName": "MLOps-SP-NS",
    "name": "<name>",
    "password": "<password>",
    "tenant": "<subID>"
}
```

You can use this information in the GitHub Secret now.

### Getting all the information

To use this Service Principal, we need a few details:
- Subscription ID: Navigate to your [Subscription List](https://portal.azure.com/#blade/Microsoft_Azure_Billing/SubscriptionsBlade), and copy the ID of the right Subscription
- Tenant ID: Click on your Subscription and find the `Parent Management Group`
- Service Principal name: The name you entered earlier: `MLOps-SP-NS`
- Service Principle App ID: The app ID from the result of the CLI / The Azure Portal
- Service Principle Secret: The password from the result of the CLI / The Azure Portal



## Setting up GitHub

The next step we should do is set some things up in GitHub.
We are already familiar with this, but it's a good way to re-use it once again to store not only our code, but also the pipelines we will use.

GitHub Actions, the CI/CD tool that's built-in into GitHub, works by defining a workflow in a YAML syntax in a `.github/workflows` folder onto your repository.
This way, you can define your pipelines into your code, without having to leave your favourite IDE (which, by now, should be Visual Studio Code ...)

Any file in the workflows directory will be executed depending on the defined **triggers**. There are triggers for **push, pull request, manual, workflow trigger and so much more**.
Read more on these triggers in [here](https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions).

:::success
**QUESTION**
Which trigger do we need for the manual start?
```
Answer here ...
```
:::

The actions we can use are very extensive. There are pre-made actions that were made by other users. There are even actions specfic for Azure, which can execute Azure CLI commands without our manual interference.

However, in some cases, it is best to use our own commands, that way we have all the control over what is getting executed and what happens behind the scenes.

In this case, we will also do this, by just executing Python scripts which will be executed in one go.

**Where will these commands be executed then?**
Well, if we are using GitHub Actions, we can define the machine that we want our pipeline to run on. Or, actually, we define the specs of the machine and what it should contain.

Some examples include `windows-latest`, `ubuntu-latest`, `ubuntu-2004` ... This simply states the operating system and will give us any machine from GitHub itself, which contains this requirement. We don't have to pay for it, unless we want more concurrent or longer pipeline runs.
We can also use our own laptop as a GitHub Actions runner, which is something you already have seen in the previous assignments.

### Cloning the repository

Use and clone this GitHub template into your own account: https://github.com/nathansegers/mlops-2.0

Clone your own repository onto laptop so it's ready for some local development and debugging.
The directory structure is as follows:

```yaml!
- azureml-1.0-automation # This contains the files that were used to automate the GitHub Actions pipelines using the Azure ML SDK v1
- azureml-1.0-sdk # This is the repository you used in the previous assignment
- azureml-2.0-sdk # This represents the code to use in Stage 2
- azureml-2.0-cli # This represents the code to use in Stage 3.
```
<!--
If you want to do some local debugging, follow the steps [at the bottom of this page.](#Debugging-without-GitHub-Actions)
-->

In this repo you can find a complete example on how to Automate your Azure Machine Learning pipeline.
I have made sure you can manually trigger the workflow and tick of a few boxes to redirect the flow of the pipeline. More on that will be explained later on.

![Manually triggering the workflow](https://i.imgur.com/viOdVf3.png)


### Filling in Secret and Environment Values

Remember that the GitHub Actions workflows can access Secrets, like AZURE_CREDENTIALS? Well, we can fill in these details under the GitHub Repository Settings. Scroll down to the Secrets and fill in the secrets you would like.

A few examples. Fill these in in your project

```
AZURE_SERVICE_PRINCIPAL_SECRET (named CLIENT_SECRET on GitHub)
```


Any values that don't really need to be secret, and may be read out by others, you can fill them in inside the GitHub Workflow.

In the Repository, we have already set up a few of the most important environment values in the `env` part. Notice that this is information that is visible to other using your project.
This can be interesting for things like:
- Azure Resource Group
- Azure Subscription
- Azure Tenant ID
- Batch Size in training
- Experiment name
- Model name
- ...

Change the values to your own setup accordingly.

The `env` variables in GitHub Actions are split up depending on their 'context'.
You will notice there is a `env` object on the root level (no indentation). These environment variables are shared by all jobs.
Underneath each job, there is also a `env` object. I have used this to set environment variables who are specific to each job. This can help in case you are ever going to override specific variables. The environment values in a job have a higher priority.


### Remarks on the GitHub Code

I have copied most of the code from the previous assignment which was still in notebooks, and converted it to allow for automation in executable Python scripts.

Below you can find some notes on things I had to change.

#### General remarks

I have used more environment values instead of variables that we kept in memory in the previous assignment. We use a lot of `os.environ.get()` to capture these environments.

As we can't control execution of specific code blocks / cells like we did in Notebooks I have provided things like this:

```python
# Yes, I'm afraid that environment variables are always strings, so I can't do a boolean check to True, but I need to use "true" instead...
if os.environ.get('PROCESS_IMAGES') == 'true':
    print('Processing the images')
    prepareDataset(ws)
else:
    print('Skipping the process part')
```

These allow us to skip parts of the code if we want to. Make sure to select the right values in the environment variables to allow your whole pipeline to run.

In the Azure Notebooks, we could easily connect to the Azure ML Workspace because we were already authenticated to Azure. But now we are using Service Principals for our connection, this means we have a new way of connecting.
As each script needs to access the workspace, I have created a `utils.py` script with a connection method.

#### Data Preparing script

As we are working on our own local machine to run the Data Preparing, we couldn't use the `mount()` option of a Dataset. This was converted to `download()` in order to work.

:::danger
Note that I have used the code that worked on my system from the previous assignment. Students who had changed something in their code should also change it in this version!
:::

#### AI Training script

The AI Training script is adapted to allow training on the runner itself, instead of a Cloud machine from Azure. Note that you do need to have enough memory / power to train the AI model. Our laptop's generally have enough power, so don't worry about it for this assignment.
In case you need GPU's, you can try using other custom machines.

The Conda Environment can be version controlled on GitHub as well. This is found under the `conda_dependencies.yml` file. This will configure the Azure Training Environment.

I have given the solutions to the previous assignment to pass all the input parameters. `train.py` also has some solutions how to log the Training information, thanks to the solutions of some students.


#### API Creation / Deployment

The API deployment was previously seperated into two notebooks. In here, we have combined these two notebooks into one script called `03_Deployment.py` which can either download the model locally, or otherwise deploy it on Azure and return the service.

If you are deploying locally, we can proceed to deploy to our Kubernetes cluster. The commands that were executed in the CustomDeployment notebook of the previous assignment were now added to the GitHub Actions Workflow file itself. You can find all the details in there.


#### Extra

You might notice there is a folder called `.devcontainer`. This is something new from Visual Studio Code that allows you to quickly spin up a Docker container that is configured with VSCode to develop apps easily.
I have included this in case you want to use it, but I haven't tested this that much yet.

### Configuring our own GitHub Actions Runner

As I have mentioned earlier on, we need to make sure to run our GitHub Actions pipeline on our own custom GitHub Actions Runner, to access our Kubernetes cluster.

Use the steps from the previous HackMD page's in order to get your local runner connected to this repository again.


#### Benefits
We are using the GitHub Actions Runner to access our local Kubernetes cluster.

:::success
**QUESTION**
Can you give another benefit for using a self-hosted GitHub Actions Runner?
```
Answer here
```
:::

## Stage 1b: More theory about MLops pipelines: Pipeline controlling
As we mentioned in the theoretical session, it is not always useful to run our pipeline from top to bottom.
Sometimes we want to skip a few steps in order to speed up the pipeline and we don't have to run the earlier steps of the pipeline.

When we only want to execute a new training job, we want to skip our Data processing pipeline.

In this case, we have enabled a the Pipeline Workflow **Inputs** in the GitHub Actions flow. These are called `data_prep`, `ai_training`, `api_creation`. You can enable these by setting the values to `True` or `False` to disable them. (Note: The values must be a string to be identified correctly for GitHub)

The first time you're running the pipeline, you will need to run everything starting from the beginning of the pipeline, otherwise we haven't done any Data Processing.
All of this information **can** be stored in Artifacts, but they have a Retention Period of 90 days, when you're using GitHub.
We can also store it in the Azure Storage accounts, because that's what is staying for a much longer time.

The only difficulty in here would be: **What version of the data are we going to use whenever we are going to rerun the pipeline and skipping a few steps?**.
In this case, I have implemented a `DATASET_VERSION` environment variable, which defaults to `latest`. This will always take the latest version that is available on Azure. Don't forget to fill in the name `DATASET_NAME` of course, otherwise it doesn't know what name it should get. In case you want to use a specific older version, than you can specify that as well.

:::info
Feel free to play around with the code to make it even more dynamic. Pull Requests / Issues to the main repository are always welcome!
:::


## Stage 1c: Theory about Version controlling in MLOps pipelines
In Figure 1 below we can see some difficulties regarding Version Control throughout the MLOps Pipeline. There are lots of different Version Controlling options for each different step of the project.
What we want to do is find out all the information regarding different versions used in our AI Project.

Imagine the scenario where a user is complaining about a wrongly classified image. The incorrect classification will be logged together with as much information about the deployed project as possible. It is not that hard to find out what version of the API the user was working on, as a developer could've linked it in the Footer somewhere. Let's say he used **Version 1.2.3** of the API.

Now, the difficulties are encountered when we want to find out **which AI model** was deployed into API version 1.2.3. We will go **from right to left** to find out *what change was made that broke this API.* This is not something we have already seen in the previous topics, but it is very important to think about.

![Figure 1: Version Controlling MLOps Pipelines](https://i.imgur.com/829TvHD.png)
<span style='font-size:0.90em;'>**Figure 1:** Version Controlling MLOps Pipelines. Own illustration.</span>

One important thing we can use during a lot of these steps is working with **GitHub**. GitHub allows us to easily link changes to a user, as a User's commits are always generating a SHA-1 ID. [How is the GitHub SHA generated? -- Stackoverflow.com](https://stackoverflow.com/questions/29106996/what-is-a-git-commit-id).
This Commit ID, often referred to as the SHA, is an ID that is unique to the repository, and allows you to identify **who** made **what** change and **when** ?
Using this information, we can easily **blame** users when something went wrong at a certain point. cf. [Git Blame -- Git SCM](https://git-scm.com/docs/git-blame) && [Git Blame -- Bitbucket](https://www.atlassian.com/git/tutorials/inspecting-a-repository/git-blame)

### Let's try to implement the GitHub SHA!

Let's try to implement this GitHub SHA in many of the different places of our pipeline.

**Data**

If we start on the left side of the pipeline in Figure 1, we can see our AI Data. This has been labelled with an **Azure Dataset**. In the previous assignment, we saw that it was possible to attach Tags or Versions to our Azure Datasets. When we are going to label our data in Azure, we can give it specific  versions.
We don't have to link it to our GitHub SHA, as the raw data is not always going to change by **code**.
You could link it to specific SQL queries if you are creating a Database export and uploading that, if you want to. But that's outside of the scope of this assignment.

**Data preparing**

Going on to the next step, our Data preparing code, we are going to link our GitHub SHA to the **prepped** data. We do this because our Data preparing script is included into GitHub and can be a reason why the AI model gives wrong results. E.g.: The images are not correctly resized. OpenCV (or other imaging processing libraries) are given the wrong results after an update ...

:::success
**QUESTION** -- TASK
Search for the lines in the code where we have used the GitHub SHA to label something on Azure. We can tag Datasets, Models and Endpoints!
```python
# Paste your code here
```
:::

**AI Training**

The following step, the AI training, can be Version Controlled simply by using GitHub and the default Azure Machine Learning tools.
We want to keep track of the structure of our AI model, and the code to recreate that. In our previous assignment, all of this information resided in the `train.py` script. In our MLOps project, this will be a file that exists in GitHub, and is thus linked to our GitHub SHA.
Remember that Azure automatically keeps a Snapshot of the Training Job's scripts? Well, this comes in play now! **Use the GitHub SHA** in the Training Job tags and you'll have your GitHub commit linked as well. The Snapshot information is already been stored.

**Model saving**

Next comes the saving of our actual trained AI model. Azure links this to our Training Job as this is a direct result of such a training job. This AI model can also be organized with Tags such as the GitHub SHA. 

**Model packaging (Docker Image)**

Finally, our AI model will be combined and packaged with our API code into a Docker Image. This is by far the easiest way of version controlling everything. Simply by adding a new **Docker Tag**, we can specify the GitHub SHA and organizing the Docker Tags in that way. [Tagging Docker image the right way -- container-solutions.com](https://blog.container-solutions.com/tagging-docker-images-the-right-way).
This doesn't stop you from using the Semantic Versioning (`v1.2.3`) in your Docker image as well... Docker Images are also hashed in a similar way as GitHub Commits. When you label a Docker image with multiple labels, they refer to the **exact** same version. In the table below, you can see that there are bunch of Docker images with different tags, each having the same Image ID and are thus exactly equal. 

```
REPOSITORY                             TAG                                        IMAGE ID       CREATED        SIZE
ghcr.io/nathansegers/azure-mlops-api   05ee055698af2171294615a1e423f71f4098628e   3f49a154a09c   5 days ago     998MB
ghcr.io/nathansegers/azure-mlops-api   05ee055                                    3f49a154a09c   5 days ago     998MB
ghcr.io/nathansegers/azure-mlops-api   customer-release                           3f49a154a09c   5 days ago     998MB
ghcr.io/nathansegers/azure-mlops-api   latest                                     3f49a154a09c   5 days ago     998MB
ghcr.io/nathansegers/azure-mlops-api   v1.2.3                                     3f49a154a09c   5 days ago     998MB

## These images are uploaded to Docker Hub instead of GitHub Container Registry
nathansegers/azure-mlops-api           v1.2.3                                     3f49a154a09c   5 days ago     998MB
nathansegers/azure-mlops-api           05ee055698af2171294615a1e423f71f4098628e   3f49a154a09c   5 days ago     998MB
```

### What if we skipped a few steps in our MLOps run?

Sometimes it is possible that earlier steps of your MLOps pipeline do not need to be ran again. In that case, we need to find out which earlier steps were executed for our API deployment.

Azure will help us for this, with all the information that gets logged into the Azure Machine Learning Service.



### Summarize how to find information

Let's summarize how we can get all the information now.

:::warning
This is quite experimental to write down. I will try to visualise it eventually, in case nobody understands this ...
:::

**Find out who changed the API Code**
Imagine you have your API deployed on Kubernetes, and you want to find out who did the latest change.
Most likely, you have kept some kind of metadata in the API. This can be the version you are using, like `v1.2.3`.
```
API Metadata --> ENVIRONMENT VARIABLE (API VERSION) --> v1.2.3
v1.2.3 --> DOCKER IMAGE ID --> 05ee055698af2171294615a1e423f71f4098628e
05ee055698af2171294615a1e423f71f4098628e --> GitHub SHA --> User Information + Code
```

**Find out which AI model was used in the API Code**

I think it's best to link a reference to the AI model version in an environment variable of the API code. This way you can always see which AI model was deployed in your API.

```
API CODE --> ENVIRONMENT VARIABLE (AI MODEL) --> v1.0.0
v1.0.0 --> AZURE MACHINE LEARNING SERVICE --> Training Job information
Training Job Information --> TAGS --> GitHub SHA
GitHub SHA --> User Information + Code
```

**Find out which AI data was used in the API Code**

You can delve deeper into the Azure Machine Learning Service to find information regarding the data that has been used to train our AI model.

```
AI MODEL v1.0.0 --> AZURE MACHINE LEARNING SERVICE --> Training Set and Testing Set
Training Set and Testing Set --> TAGS --> GitHub SHA
GitHub SHA --> User Information + Code
```

What other information would you like to find?


## Stage 1d: Debugging the pipeline code without GitHub Actions

As GitHub Actions is just about executing commands in the right order, we can try to debug the steps locally at first.

Here are a few suggestions how you can debug this:
- Clone the repository onto your Virtual Machine
- Either use the Docker Dev Containers, like in this screenshot. Press 'Reopen in Container'. This is only available on VSCode.
![](https://i.imgur.com/ZesxAJ4.png)
- Or else, create a new, fresh Virtual Environment for Python. [Creating a Venv for Python -- Python.org](https://docs.python.org/3/library/venv.html).
- Create a `.env` file which will contain all the environment variables that you normally fill in into the GitHub Actions secrets. Copy the `.env.example` to get a starter of the values to fill in.
- Now you can execute all the files one by one, and fix files if necessary.
- Don't forget to Commit your changes if you want them to get executed through the GitHub Actions now! In case you have changed some environment variables, make sure to reflect those changes into the `azure-ai.yml` file too.

## Stage 1e: Deploying to Kubernetes

The part we have done now is actually only our Continuous Integration part. We ended up getting a Docker Image pushed to our GitHub Container Registry, but we haven't deployed anything yet.

Try to add a new pipeline job to deploy to Kubernetes using the Kubectl and your GitHub Actions Runner.

Use your creativity in either creating a new pipeline job, a new pipeline step, or even a new pipeline workflow. It's all your choice.

:::success
**QUESTION**
Setup the Kubernetes deployment pipeline. Explain your solution below.
```text
Explanation here
```
Paste your YAML file / adaptions here
```yaml
# Yaml file here
```
:::

## Stage 2: Using the Azure Machine Learning Pipelines

:::warning

Detailed description coming soon

:::

## Stage 3: Combining the Azure ML Pipelines with GitHub Actions

:::warning

Detailed description coming soon

:::

## Stage X: Own suggestions regarding MLOps

:::success
**QUESTION**
Do you have more suggestions how we can add more functionalities into the GitHub Actions pipeline to create a better MLOps flow?
***This is a free question, there are no good or wrong answers. Use your own knowledge or things we learned in the previous sessions / assignments. It might be good to capture your interests to include it in the assignments later on...***
(Answer by adding more items to the list)
1. E.g.: My first suggestions would be to use a seperate Secret managing system like Azure Key Vault instead of the GitHub Secrets.
2. ...

:::
{%hackmd MyiK4mWyRYauLyaDokvlfg %}

###### tags: `MLOps@Home` `MCT` `MCT@Home` `AzureMLOps` `Azure`

