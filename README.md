# tutorial-flask-api
Practising API creation by following tutorial at: https://github.com/Azure-Samples/azure-sql-db-python-rest-api.
There will be some differences to the original.

For the moment, you can find the API here: 
https://tutorial-flask-api.azurewebsites.net/customers
https://tutorial-flask-api.azurewebsites.net/customer/123

## Create resource group and SQL server
- az group create -n rg-wwimporters -l westeurope 
- az sql server create -g rg-wwimporters -n sql-wwimporters -u adm-sql-wwimporters -p SetYourPwHere

## Import Database
https://github.com/yorek/azure-sql-db-samples#restore-wideworldimporters-database

There was maybe something wrong with yorek's import script doctypes, as I did not manage to use Azure CLI or other scripted versions. Instead I did the import via Azure Portal. Note that I chose the standard version and not the full example database, so that a cheaper (Standard, not the BC_Gen5_2 like in the original tutorial) service plan is enough. 

- Create a Block Blob Storage and upload the bacpac there as block blob:
  - https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-portal
  - direct link to the Standard version of the bacpac: https://github.com/Microsoft/sql-server-samples/releases/download/wide-world-importers-v1.0/WideWorldImporters-Standard.bacpac
- Import DB via Azure portal
  - Video at: https://docs.microsoft.com/en-us/azure/azure-sql/database/database-import?view=azuresql&tabs=azure-cli
  - Server -> Networking -> Exceptions: allow Azure services and resources to access this server
  - Start with Create database. Remember to set Standard service plan, so it won't get _too_ costly.
  - Follow up how the import is going: Import/Export history. Only remove the bacpac blop once you have verified your db works.

## Create web schema in the database for the app

- Connect with e.g. Azure Data Studio
  - How to use Data Studio: https://docs.microsoft.com/en-us/sql/azure-data-studio/quickstart-sql-database
  - Connection details
    - Connection type: Microsoft SQL Server
    - Server: sql-wwimporters.database.windows.net
    - Authentication type: SQL Login
    - the user name & pw you gave earlier
- Use the query at https://github.com/Azure-Samples/azure-sql-db-python-rest-api/blob/master/sql/WideWorldImportersUpdates.sql to create 
  - web-schema
  - PythonWebApp -user
  - Procedures the API will use (web.get_customers, web.get_customer, web.delete_customer, web.patch_customer, web.put_customer)

## Run the app locally

### Clone the example repo to local computer
- Clone the example repo to local computer e.g. via GitDesktop: File -> Clone reposity: paste url (from repo: Code-button -> copy https-url) and check directory.
- I ended up creating own repo instead and copying the parts I used there.

### Setting up the virtual environment in Anaconda

Create new python environment (e.g. tutorial-flask-api) via Anaconda Navigator with Python 3.9
- Open cmd from the environment
- Go to folder where the requirements.txt is
- Anaconda does not have opencensus-ext-flask, so installed it with pip:
  - pip install opencensus-ext-flask
  - Remove it from the requirements.txt and install rest:
  - conda install -c conda-forge --file requirements.txt (installs all packages in requirements.txt with an additional channel to search for packages)

Add environmental variables
- conda activate tutorial-flask-api
- conda env config vars set FLASK_ENV=development
- conda env config vars set SQLAZURECONNSTR_WWIF="DRIVER={ODBC Driver 17 for SQL Server};SERVER=sql-wwimporters.database.windows.net;DATABASE=WideWorldImporters-Standard;UID=PythonWebApp;PWD="AddYourPasswordHere""
- conda activate tutorial-flask-api (for the changes to take effect)
- conda env config vars list (to check them out)
- You can remove vars, e.g.: conda env config vars unset FLASK_ENV

Visual Code
To run the app: go to the folder where the app.py is, enter
- flask run

Test with another cmd: curl -X GET http://localhost:5000/customer/123 or set up Postman. See the original tutorial for delete, put and patch.

### Debugging with VS Code
Only the get and delete worked without changes (though the database integrity constraints prevent deleting customers with e.g. invoices and all existing customers have invoices, so if you only want to operate via this API, you need to create a new customer to be able to test delete). 

Set up debugging: https://towardsdatascience.com/how-to-debug-flask-applications-in-vs-code-c65c9bdbef21

Edited put and patch in app.py, as flask-restful requestparser caused errors that I was not able to workaround. Finally ended up removing requestparser and instead got the json from Flask's request-object.

Test all request with e.g. Postman. For me the cUrl samples for put and patch (that included 'customer= ) did not work. With Postman all was ok and I could validate the json body with https://jsonlint.com/.

## Deploy the App to Azure
I followed the tutorial and deployed from local git repo instead of doing the deployment directly from Github.

I switched to following a Medium article (https://medium.com/@GeekTrainer/deploying-python-web-apps-to-azure-app-services-413cc16d4d68) and Microsoft's documentation, like instructed on the tutorial.

### Deployment account
Used for authentication when pushing the code to Azure

- Azure Portal -> Cloud Shell (from the top, right side of the search field)
  - First time when using Cloud Shell, you need to create a file share. It will get prompted. Choose an existing storage accoun or create a new one (I used the same I created for the blob when importing the database) and created a new fileshare fs-st-wwimporters-cli under it.
- az webapp deployment user set --user-name depuser-tutorial-flask-api (you will get prompted for setting up a password)

### Resource group
- Let's use the same we created for the db: rg-www-importers

### App service plan
App service plan, which will act as our server farm and control the operating system and execution resources. We will be creating an App Service plan using the Basic SKU. This will incur a charge.

- az appservice plan create --name plan-tutorial-flask-api --resource-group rg-wwimporters --sku B1 --is-linux

### Web app
Will be used to configure our runtime provide the endpoint for the end users and for the deployment.

- I had developed this with python 3.9.12, the tutorial has older version 3.7. You can check the available runtime versions with on Azure CLI: az webapp list-runtimes. 3.9. was available.
- az webapp create --name tutorial-flask-api --plan plan-tutorial-flask-api --resource-group rg-wwimporters --runtime "Python:3.9" --deployment-local-git
  - note down the url local git is configured at
- az webapp config connection-string set -g rg-wwimporters -n tutorial-flask-api --settings WWIF="DRIVER={ODBC Driver 17 for SQL Server};SERVER=sql-wwimporters.database.windows.net;DATABASE=WideWorldImporters-Standard;UID=PythonWebApp;PWD="AddYourPasswordHere"" --connection-string-type=SQLAzure

If you want to check 
- connection string: az webapp config connection-string list --name tutorial-flask-api --resource-group rg-wwimporters
- appsettings: az webapp config appsettings list --name tutorial-flask-api --resource-group rg-wwimporters

### Push from local repository to Azure
- In VS Code terminal, go to the folder where your local repo for this project is
- git remote add azure insert-the-url-you-noted-down-earlier-when-creating-the-webapp
- git push azure main:master (my git repo's main branch is called main. If yours is called master: git push azure master)


