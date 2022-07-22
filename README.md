# tutorial-flask-api
Practising API creation with https://github.com/Azure-Samples/azure-sql-db-python-rest-api

Setup done in Anaconda environment

Create new python environment (e.g. tutorial-flask-api) via Anaconda with Python 3.9.12
    - Open cmd from the environment
	- Go to folder where the requirements.txt is
    - Anaconda does not have opencensus-ext-flask, so installed it with pip:
	    - pip install opencensus-ext-flask
        - Remove it from the requirements.txt and install rest:
		    - conda install -c conda-forge --file requirements.txt (installs all packages in requirements.txt with an additional channel to search for packages)

conda activate tutorial-flask-api
conda env config vars set FLASK_ENV=development
conda env config vars set SQLAZURECONNSTR_WWIF="DRIVER={ODBC Driver 17 for SQL Server};SERVER=sql-test-importers.database.windows.net;DATABASE=WideWorldImporters-Standard;UID=PythonWebApp;PWD="a987REALLY#$%TRONGpa44w0rd""
conda activate tutorial-flask-api (for the changes to take effect)
conda env config vars list

You can remove vars, e.g.: conda env config vars unset FLASK_ENV

To run the app: go to the folder where the app.py is, enter
    - flask run

Test with e.g. Postman or another cmd: curl -X GET http://localhost:5000/customer/123