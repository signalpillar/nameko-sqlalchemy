nameko-sqlalchemy
=================

A SQLAlchemy dependency for `nameko <http://nameko.readthedocs.org>`_, enabling services to interface with a relational database.

Usage
-----

.. code-block:: python

    from nameko_sqlalchemy import Session

    from .models import Model, DeclarativeBase

    class Service(object):

        session = Session(DeclarativeBase)

        @entrypoint
        def write_to_db(self):
            model = Model(...)
            self.session.add(model)
            self.session.commit()

        @entrypoint
        def query_db(self):
            queryset = self.session.query(Model).filter(...)
            ...


Database drivers
----------------

You may use any database `driver compatible with SQLAlchemy <http://docs.sqlalchemy.org/en/rel_0_9/dialects/index.html>`_ provided it is safe to use with `eventlet <http://eventlet.net>`_. This will include all pure-python drivers. Known safe drivers are:

* `pysqlite <http://docs.sqlalchemy.org/en/rel_0_9/dialects/sqlite.html#module-sqlalchemy.dialects.sqlite.pysqlite>`_
* `pymysql <http://docs.sqlalchemy.org/en/rel_0_9/dialects/mysql.html#module-sqlalchemy.dialects.mysql.pymysql>`_


Pytest fixtures
---------------

Pytest fixtures to allow for easy testing are available.

* ``db_session`` fixture (which depends on ``db_connection`` fixture) will instantiate test database and tear it down at the end of each test.
* ``model_base`` fixture can be overridden to provide custom ``declarative_base``.
* ``db_engine_options`` fixture can be overriden to provide additional keyword arguments to ``sqlalchemy.create_engine``.

.. code-block:: python

    import pytest

    from sqlalchemy import Column, Integer, String
    from sqlalchemy.ext.declarative import declarative_base


    class Base(object):
        pass


    DeclarativeBase = declarative_base(cls=Base)


    class User(DeclarativeBase):
        __tablename__ = "users"

        id = Column(Integer, primary_key=True)
        name = Column(String)


    @pytest.fixture(scope='session')
    def model_base():
        return DeclarativeBase


    @pytest.fixture(scope='session')
    def db_engine_options():
        return dict(client_encoding='utf8')


    def test_users(db_session):
        user = User(id=1, name='Joe')
        db_session.add(user)
        db_session.commit()
        saved_user = db_session.query(User).get(user.id)
        assert saved_user.id > 0
        assert saved_user.name == 'Joe'

When running tests you can pass database test url with ``--test-db-url`` parameter or override ``db_url`` fixture.
By default SQLite memory database will be used.

.. code-block:: shell

    py.test test --test-db-url=sqlite:///test_db.sql
    py.test test --test-db-url=mysql+mysqlconnector://root:password@localhost:3306/nameko_sqlalchemy_test

Decorators
----------

transaction_retry
^^^^^^^^^^^^^^^^^
This decorator automatically retries the wrapped function when a database connection error occurs.
If the optional ``session`` argument is passed it will issue a rollback on it before retrying so the transaction can be processed again.
The ``session`` argument can either be the ``sqlalchemy.orm.session.Session`` or an ``operator.attrgetter`` object if the session is a class attribute.


Usage
"""""

.. code-block:: python

    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker

    from nameko_sqlalchemy import transaction_retry


    engine = create_engine('postgresql://username:password@localhost/test')
    Session = sessionmaker(bind=engine)
    db_session = Session()

    @transaction_retry()
    def get_example_data():
        db_session.query(ExampleModel).all()

    example_data = get_example_data()


or using with the ``Database`` dependency provider

.. code-block:: python

    from sqlalchemy.ext.declarative import declarative_base
    from nameko_sqlalchemy import Database, transaction_retry


    DeclBase = declarative_base(name='examplebase')


    class ExampleService:
        name = 'exampleservice'

        db = Database(DeclBase)

        @entrypoint
        @transaction_retry
        def get_examples(self):
            with self.db.get_session() as session:
                return session.query(ExampleModel).all()

        @entrypoint
        def get_examples_with_retry_inside(self):
            with self.db.get_session() as session:
                @transaction_retry(session=session)
                def foo():
                    return session.query(ExampleModel).all()

                return foo()

        @entrypoint
        @transaction_retry
        def create_example_without_using_context_manager(self):
            session = self.db.get_session()
            session.add(ExampleModel(data='hello'))
            session.commit()

        @entrypoint
        @transaction_retry(session=operator.attrgetter('db.session'))
        def create_example_with_worker_scoped_session(self):
            self.db.session.add(ExampleModel(data='hello'))
            self.db.session.commit()

.. caution::
    Using the decorator may cause unanticipated consequences when the decorated function uses more than one transaction.

It should only be used around single transactions because all transactions inside the decorator will be re-executed if there is a connection error during any of them. Take a look at the following example:

.. code-block:: python

    class ExampleService:

        db = Database(DeclBase)

        @entrypoint
        @transaction_retry
        def method(self):
            with self.db.get_session() as session:
                session.add(something)

            do_something()  # during this a network error occurs

            with self.db.get_session() as session:
                session.add(something_else)  # throws error because the db connection is gone, method will be executed again


Since the method is retried all of the statements are executed twice, including the ones that didn't fail. As a result of that ``something`` will be added twice.
In order to avoid that one may want to do something like this:

.. code-block:: python

    class ExampleService:

        db = Database(DeclBase)

        @entrypoint
        def method(self):
            with self.db.get_session() as session:
                @transaction_retry(session=session)
                def add_two_things():
                    session.add(something)
                    do_something()
                    session.add(something_else)

                add_two_things()

In this case the failed transaction will be rolled back (becase the session is passed to the decorator) and records will not be duplicated.

Running the tests
-----------------

Prerequisites
^^^^^^^^^^^^^

Some of the tests use `toxiproxy <https://github.com/Shopify/toxiproxy>`_ to simulate network errors. In order to be able to run those tests you need a toxiproxy server to be in place. You may install it manually or by running the following command (docker is required):

.. code-block:: shell

    make test-deps

This will setup a mysql and a toxiproxy server with a proxy set up to the database.


Running tests by using docker
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Once the containers have been set up the tests can be run by running the following command:

.. code-block:: shell

    make test


Running tests by using py.test command
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Three extra parameters may be passed to `py.test`:

* ``test-db-url``: The database URL
* ``toxiproxy-api-url``: The url of the Toxiproxy HTTP API
* ``toxiproxy-db-url``: The url of the database through Toxiproxy

If ``toxiproxy-api-url`` and ``toxiproxy-db-url`` parameters are provided the tests assume that the toxiproxy endpoint is already set up to a database upstream and this proxy can be disabled and enabled via the HTTP API of toxiproxy.

.. code-block:: shell

    py.test test \
        --test-db-url="mysql+pymysql://test_user:password@database_host:3306/nameko_sqlalchemy_test" \
        --toxiproxy-api-url="http://toxiproxy_server:8474"
        --toxiproxy-db-url="http://toxiproxy_server:3306"

if no ``toxiproxy-api-url`` and ``toxiproxy-db-url`` parameter was provided the tests that require toxiproxy will be skipped.

