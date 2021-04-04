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

    CREATE_TABLE_QUERY = f"""
      CREATE TABLE IF NOT EXISTS {TABLE_NAME} (
        first_operand INT NOT NULL,
        second_operand INT NOT NULL,
        answer INT NOT NULL,
        PRIMARY KEY(first_operand, second_operand)
      )"""

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
    table = "calculations"

    try:
        engine = create_engine(URI, echo=True)
        my_connection = engine.connect()

        my_connection.execute(f"""
            'INSERT INTO {table} VALUES ({firstNum}, {secondNum}, {answer})
        """
        return "Insertion successful"

    except exc.SQLAlchemyError as err:
        return f'Error occured inserting into table {table}. Exception: {err}'

    finally:
        my_connection.close()
        engine.dispose()
```

- Create the file `.\flask_app\app.py`

```python
from flask import Flask, request, jsonify
from flask_app.storage import insert_calculation, get_calculations

app = Flask(__name__)


@app.route('/')
def index():
    return "My Addition App", 200


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

- Create the file `requirements.txt`

```

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