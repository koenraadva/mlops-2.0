This directory will explore how to automate all the notebooks, which have now been converted into Python scripts. These Python scripts will be executed in order by chaining them in the GitHub Actions pipelines (.github/workflows/...)


This directory contains Python scripts, which can be found in the "steps" directory. The API to test our application is found under "API".
Any other scripts have been provided into the "scripts" directory.

We use the yaml files like "conda_dependencies.yml" and "deployment_environment.yml" to keep our configuration in a seperate file for ease of use.

The "requirements.txt" file is needed if you want to run these scripts on your own laptop, or on the GitHub Actions runner, which is what we will be doing.

The ".env.example" contains all the example variables that you need to fill in, in order to run the scripts. You can copy this file and rename it to ".env" and fill in the variables.

The reason we're not using an actual ".env" file is because that one will contain some secrets, and will not be copied into our repository. You can see that because it's added to the ".gitignore" file.

There are a few variables that do need to stay a secret, and those are the ones we need to enter in GitHub.

Let's have a look now ...

- Move over to the ".env.example" files and to the ".github/workflows" directory.

You might wonder why we need to fill it in twice then??
Well, because the .env file is not included into the Repository, we can use that one to test locally... If we want to run the Python scripts on our own laptop, we can test if they actually work or not ...

It's possible by installing all the dependencies and packages using the "requirements.txt" file, then just running the Python scripts in the right order. The "dotenv" package will then read the ".env" file and use those variables.


Because we are getting the variables using "os.environ.get()", we can also use that one with the variables in the GitHub Actions pipeline