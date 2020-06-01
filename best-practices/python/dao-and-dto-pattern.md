# Backend Patterns - DAO and DTO Patterns

Patterns for Separating Application Business Logic from the Data Access Layer.

## Glossary

- [Intro](#intro)
- [Benefits and Downsides](#benefits-and-downsides)
- [Example](#example)
- [Value Proposition](#value-proposition)
  - [Why](#why)
  - [How](#how)
  - [What](#why) - [Maintainability](#maintainability) - [Extensibility](#extensibility) - [Performance](#performance)

## Intro

What are the DAO and DTO patterns?

_Data Access Object_ (DAO) is an abstraction that isolates data access (ie, database) to a specific interface.
In contrast, _Data Transfer Object_ (DTO) is an abstraction that encapsulates data in a read-only interface.

These two patterns are typically used together in order to allow for a separation of concerns.

## Benefits and Downsides

At a high level, below are the benefits and downsides, which we will dig into more in the _Example_ and _Value Proposition_ sections.

Benefits:

- Less coupling of business logic to database technology (DAO)
- Less coupling of business logic to database session management (DAO)
- Capability to batch together multiple calls into one object (imagine objects in a one-to-many relationship configuration - we preload and "cache" the associations into an object property)
- Dedicated API for interacting with data schema in question (DTO)

Downsides:

- Upfront effort necessary to implement in a clean and concise manner
- Somewhat easy to abuse these patterns

## Example

First, let's illustrate the DAO / DTO pattern with a code example:

```py
# services/finance/models.py
class Finance(namedtuple('Finance', ['user_id', 'balance'])):
    @classmethod
    def from_dao(cls, dao):
        return cls(
            user_id=dao.user_id,
            balance=dao.balance,
        )


# services/finance/db.py
from contextlib import ContextManager
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base
from app import db

engine = create_engine("db_url")
Session = sessionmaker(bind=engine)


class FinanceDAO(db.Model):
    user_id = db.Column(db.Integer())
    balance = db.Column(db.Integer())
```

Above, we have the `Finance` class, which represents the DTO. We also have the `FinanceDAO` class, which represents the DAO. The DTO is the read-only implementation of the `FinanceDAO` class. The `FinanceDAO` class is the code-level abstraction for the data layer (or database), and is the mechanism that we use to interact with the data layer.

```py
class FinanceContextManager(ContextManager):
    def __init__(self):
        self.session = None

    def __enter__(self):
        self.session = Session()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.session.close()

    def get_finance_by_id(self, finance_id):
        dao = self.session.query(FinanceDAO).filter_by(id=finance_id).one_or_none()
        return dao

    def get_finance_by_user_id(self, user_id):
        dao = self.session.query(FinanceDAO).filter_by(user_id=user_id).one_or_none()
        return dao

    def create_finance(self, user_id, balance):
        dao = FinanceDAO(
            user_id=user_id, debt=debt
        )
        self.session.add(dao)
        self.session.commit()
        return dao

    def update_finance(self, finance_id, balance=None):
        dao = self.get_finance_by_id(finance_id)

        if not dao:
            raise Exception("does not exist")

        if dao.balance != balance:
            dao.balance = balance

        self.session.add(dao)
        self.session.commit()

        return dao
```

In addition to the DAO and DTO, we have the `FinanceContextManager`. This is a context manager that wraps up our interactions with the database into one interface. It also will contain the database session configuration, and is the mechanism through which database sessions are opened and closed.

```py
# services/finance/api.py
from services.finance.db import FinanceContextManager


def get_finance_by_id(finance_id):
    with FinanceContextManager() as manager:
        finance_dao = manager.get_finance_by_id(finance_id)
        return Finance.from_dao(finance_dao)


def get_finance_by_user_id(user_id):
    with FinanceContextManager() as manager:
        finance_dao = manager.get_finance_by_user_id(user_id)
        return Finance.from_dao(finance_dao)


def create_finance(user_id, balance):
    with FinanceContextManager() as manager:
        finance_dao = manager.create_finance(user_id=user_id, balance=balance)
        return Finance.from_dao(finance_dao)


def update_finance(finance_id=None, user_id=None, balance=None):
    if not (finance_id or user_id):
        return None

    finance_dao = None
    if finance_id:
        finance = get_finance_by_id(finance_id)
    elif user_id:
        finance = get_finance_by_user_id(user_id)

    with FinanceContextManager() as manager:
        try:
            finance_dao = manager.update_finance(finance.id, balance=balance)
        except Exception as e:
            raise e

        return Finance.from_dao(finance_dao)
```

Finally, we have the DAO / DTO API implementation for the `Finance` data type. These API methods are the "public" methods that we use to make CRUD changes to the underlying table that maps to the `Finance` data type.

```py
# tasks.py
from services.finance import api as finance_api


def update_user_balance():
    user_id = 1
    new_balance = 10000
    finance = finance_api.update_finance(user_id=user_id, balance=new_balance)
```

The above is an example of the DTO / DAO mechanism in action.

## Value Proposition

### Why

Why is this pattern useful?

The DAO / DTO pattern is a means of accomplishing separation of business logic from the database access layer. The DAO / DTO pattern affords several benefits:

- separation of concerns
- separation of a resource's client interface from data access layer
- restriction of resource API to a generic interface
- simplified process of migrating data access layer

In short, this allows us to isolate the data access layer completely from the backend code that gets passed around within the business logic, which should help in regards to the ease of maintainability over the long run.

### How

How do we accomplish isolation of the data access layer from the business logic?

Isolation of the data access layer is accomplished by separating the DTO class (Finance) from the data access model (FinanceDAO). We've also isolated the data access methods to the `finance/db.py` file, which should only be accessed by methods defined in the `finance/api.py` file. Ideally the data access methods should never be accessed outside of the context of the `finance/api.py` file. The DTO class also does not interact with database sessions at all, which reduces the likelihood of running into issues with session management (detached sessions, dirty sessions, etc).

### What

What does this provide for us?

#### Maintainability

Maintainability is a key win. By introducing separation of concerns at the data access level, we're able to clearly delineate between DB access logic and business logic. We're also able to more granularly introduce exception handling at each level, as well as other improvements. The code is also easier to reason about, particularly as it requires engineers to think more about how they're interacting with the database. Finally, since the data access layer is isolated to its own file, it's more straightforward replace the DAO with something else (such as a DynamoDB model, RPC object, etc). The business logic is defined via the `Finance` DTO, so it should theoretically be as simple as plugging the data access interface into the DTO.

#### Extensibility

By separating the business logic (Finance) from the data access layer (FinanceDAO), it's easier to extend the business logic. Say that you wanted to do some downstream events, it's simpler to do so via the `api.py` file. For example, if you wanted to make an API call to a 3rd party API upon creation of a `FinanceDAO` object, you can update the `api.create_finance` method to do so.

#### Performance

In terms of performance, it's arguable that by isolating the DB context manager to the `db.py` file, it's more straightforward to tweak database performance for better results. By isolating transactions within a context manager, the database interactions have been greatly simplified and made more consistent versus having multiple implementations of database transaction handling.
