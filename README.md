# react_flask_pg_google
A template project with React frontent and Flask+PostgreSQL backend, hosted on Google Kubernetes
#### Source Document

Built using [Abhi Chakraborty's guide](https://mickeyabhi1999.medium.com/build-and-deploy-a-web-app-with-react-flask-nginx-postgresql-docker-and-google-kubernetes-e586de159a4d) but porting shell commands to Powershell

# Geting Started

- Install [Git for Windows](https://gitforwindows.org/)

 ```shell
 git clone https://github.com/bradAnders/react_flask_pg_google.git
cd react_flask_pg_google
```

## Cloud Setup

- Create a project `rfp-google-cloud-project`

- Create a **PostgreSQL** database instance `rfp-database` with the default username `postgres` and password `postgres`, enabling the **Compute Engine API** if necesscary

- Create a databse `rfp-database-db` in the newly created `rfp-database`

- Create a local `config.ini` file

```ini
[history_database]
user = postgres
password = postgres
dbname = rfp-database
host = 0.0.0.0
```

## Frontend

- Create the directory `.\flask_app\`

- Create the file `.\flask_app\storage.py`

```python
from configparser import ConfigParser
from sqlalchemy import create_engine, exc
import os

def read_config():
    '''
        Instantiating a config file reader for our use case.
        To be used to read credentials for connecting to PSQL
        DB hosted in GCP SQL.
    :return: ConfigParser
    '''

    current_dir = os.path.dirname(__file__)
    file_path = os.path.join(current_dir, '../config.ini')

    config = ConfigParser()
    config.read(file_path)

    return config

def connection_uri():
    '''
        Creating connection URI for connecting to PSQL DB. URI will
        be used to create SQLAlchemy engine for executing queries.
        :return: URI for our PSQL DB hosted in GCP SQL
    '''
    config = read_config()

    URI = 'postgresql+psycopg2://{}:{}@/{}?host={}'.format(
        config['history_database']['user'],
        config['history_database']['password'],
        config['history_database']['dbname'],
        config['history_database']['host']
    )

    return URI

def create_history_table():
    '''
        Function used to create an SQL table for inserting our
        calculations into. We are using SQLAlchemy's DB engine
        for executing our created query.
    :return:
    '''

    URI = connection_uri()
    my_connection = None
    TABLE_NAME = "calculations"

    CREATE_TABLE_QUERY = """
                    CREATE TABLE IF NOT EXISTS {} (
                        first_operand INT NOT NULL,
                        second_operand INT NOT NULL,
                        answer INT NOT NULL,
                        PRIMARY KEY(first_operand, second_operand)
                    )""".format(TABLE_NAME)
    
    try:
        engine = create_engine(URI, echo=False)
        my_connection = engine.connect()
        my_connection.execute(CREATE_TABLE_QUERY)

        return "Table created successfully"

    except exc.SQLAlchemyError as error:
        return 'Error trying to create table: {}'.format(error)

    finally:
        my_connection.close()
        engine.dispose()

def insert_calculation(firstNum, secondNum, answer):
    '''
    Function used to insert our calculation into the DB.
    :param firstNum: first operand of calculation
    :param secondNum: second operand of calculation
    :param answer: calculation answer
    :return: error or success strings for inserting into DB.
    '''
    URI = connection_uri()
    my_connection = None

    try:
        engine = create_engine(URI, echo=True)
        my_connection = engine.connect()

        my_connection.execute('INSERT INTO calculations VALUES (%s, %s, %s)', (firstNum, secondNum, answer))
        return "Insertion successful"

    except exc.SQLAlchemyError as err:
        return 'Error occured inserting into table {}. Exception: {}'.format("calculations", err)

    finally:
        my_connection.close()
        engine.dispose()

def get_calculations():
    '''
    Function used to fetch calculations history from PSQL DB.
    :return: hashmap with DB calculations values
    '''

    URI = connection_uri()
    my_connection = None
    
    GET_CALCULATIONS_QUERY = """
                                SELECT * FROM calculations
                             """

    try:
        engine = create_engine(URI, echo=False)
        my_connection = engine.connect()

        calculations = my_connection.execute(GET_CALCULATIONS_QUERY)

        calculations_history = {}

        i = 1
        for row in calculations:
            calculations_history[i] = (row['first_operand'], row['second_operand'], row['answer'])
            i += 1

        return calculations_history

    except exc.SQLAlchemyError as err:
        return 'Error fetching from table {}. Exception: {}'.format("calculations", err)

    finally:
        my_connection.close()
        engine.dispose()


def delete_calculation(firstNum, secondNum):
    '''
    Function used to delete calculations from calculations DB, based on operands.
    :param firstNum: first operand
    :param secondNum: second operand
    :return: string message on whether deleted successfully or not
    '''

    URI = connection_uri()
    my_connection = None

    try:
        engine = create_engine(URI, echo=False)
        my_connection = engine.connect()
        my_connection.execute('DELETE from calculations WHERE first_operand = {} AND second_operand = {}'.format(firstNum, secondNum))
        return "Deletion successful"
    except exc.SQLAlchemyError as err:
        return 'Error deleting data from table {}. Exception: {}'.format("calculations", err)

    finally:
        my_connection.close()
        engine.dispose()

if __name__=="__main__":
    delete_calculation(12345, 12345)
```

- Create the file `.\flask_app\app.py`

```python
from flask import Flask, request, jsonify
from flask_app.storage import insert_calculation, get_calculations

app = Flask(__name__)

@app.route('/')
def index():
    return "My Addition App", 200

@app.route('/health')
def health():
    return '', 200

@app.route('/ready')
def ready():
    return '', 200

@app.route('/data', methods=['GET'])
def data():
    '''
        Function used to get calculations history
        from Postgres database and return to fetch call in frontend.
    :return: Json format of either collected calculations or error message
    '''

    calculations_history = []

    try:
        calculations = get_calculations()
        for key, value in calculations.items():
            calculations_history.append(value)
    
        return jsonify({'calculations': calculations_history}), 200
    except:
        return jsonify({'error': 'error fetching calculations history'}), 500

@app.route('/insert_nums', methods=['POST'])
def insert_nums():
    '''
        Function used to insert a calculation into our postgres
        DB. Operands of operation received from frontend.
    :return: Json format of either success or failure response.
    '''

    insert_nums = request.get_json()
    firstNum, secondNum, answer = insert_nums['firstNum'], insert_nums['secondNum'], insert_nums['answer']

    try:
        insert_calculation(firstNum, secondNum, answer)
        return jsonify({'Response': 'Successfully inserted into DB'}), 200
    except:
        return jsonify({'Response': 'Unable to insert into DB'}), 500
```

- Create the file `.\run.py`

```python
from gevent.pywsgi import WSGIServer
from flask_app.app import app

# As flask is not a production suitable server, we use will
# a WSGIServer instance to serve our flask application.
if __name__ == '__main__':
    WSGIServer(('0.0.0.0', 8000), app).serve_forever()
```

- Setup and activate python environment

```shell
python -m pip install --upgrade pip
python -m venv venv
.\venv\Scripts\activate
pip install flask
pip install sqlalchemy
pip install gevent
pip install psycopg2
pip freeze > requirements.txt
```

- Try out the app at `http://localhost:8000/`

```shell
python run.py
```