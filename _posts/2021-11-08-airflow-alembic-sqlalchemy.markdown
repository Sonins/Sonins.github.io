---
layout: post
title:  "Airflow가 Metadata DB를 사용하는 방법"
categories: Airflow Database Sqlalchemy alembic
---
## Airflow
![Airflow](https://raw.githubusercontent.com/Sonins/Sonins.github.io/main/_posts/assets/1200px-AirflowLogo.png)  
Airflow는 Apache 오픈 소스 그룹에서 관리하는 대형 오픈소스 프로젝트이다. Airflow는 작업 스케줄링 도구로, 주기적인 작업의 일정을 정의하고 그 실행 결과를 모니터링 해준다.
Airflow는 Metadata DB에 Airflow의 설정, 연결, 그리고 DAG 실행 결과등 여러가지 Airflow의 메타데이터를 저장한다. Metadata DB로 Postgresql이나 Mysql 같은 관계형 데이터베이스를 사용한다.
Airflow는 Python으로 작성되었다. 따라서 Metadata DB를 어떻게 조작하는지 보면, 앞으로 Python에서 데이터베이스를 조작할 때 참고가 될 수 있을 것 같아 이렇게 정리해둔다.

## ORM
Airflow는 Metadata DB를 sqlalchemy를 통해서 조작한다. sqlalchemy는 데이터베이스 내 객체와 파이썬 객체를 연결하는 ORM(Object Relational Mapping)을 사용한다. 쉽게 말하자면, ORM으로 정의된 파이썬 객체 하나는 데이터베이스 테이블 내 row 하나와 연결된다. 간단한 ORM 예를 살펴보면,
```python
class SerializedDagModel(Base):

    __tablename__ = 'serialized_dag'

    dag_id = Column(String(ID_LEN), primary_key=True)
    fileloc = Column(String(2000), nullable=False)
    # The max length of fileloc exceeds the limit of indexing.
    fileloc_hash = Column(BigInteger, nullable=False)
    data = Column(sqlalchemy_jsonfield.JSONField(json=json), nullable=False)
    last_updated = Column(UtcDateTime, nullable=False)
    dag_hash = Column(String(32), nullable=False)
    ...

    @classmethod
    @provide_session
    def read_all_dags(cls, session: Session = None) -> Dict[str, 'SerializedDAG']:
        """Reads all DAGs in serialized_dag table.
        :param session: ORM Session
        :returns: a dict of DAGs read from database
        """
        serialized_dags = session.query(cls)

        dags = {}
        for row in serialized_dags:
            log.debug("Deserializing DAG: %s", row.dag_id)
            dag = row.dag

            # Coherence check
            if dag.dag_id == row.dag_id:
                dags[row.dag_id] = dag
            else:
                log.warning(
                    "dag_id Mismatch in DB: Row with dag_id '%s' has Serialised DAG with '%s' dag_id",
                    row.dag_id,
                    dag.dag_id,
                )
        return dags
```
[models/serialized_dag.py](https://github.com/apache/airflow/blob/main/airflow/models/serialized_dag.py)  
위 코드에서 ORM을 사용하는 것을 볼 수 있다. SerialzedDagModel은 'serialized_dag' 테이블의 row 하나와 매핑된다. `SerializedDagModel.read_all_dags()`는 `session.query(cls)` 메소드로 serialized_dag 테이블의 모든 row를 조회한다. 이때, `@provide_session` 데코레이터는 무엇일까.  

sqlalchemy를 사용할 때, 일반적으로 데이터베이스를 조작 할 때와 마찬가지로 연결 객체를 생성해야한다. Airflow는 settings.py 파일에서 sqlalchemy engine과 sesison 객체를 정의하고 있다.

```python
from sqlalchemy.engine import Engine
from sqlalchemy.orm.session import Session as SASession
...

engine: Optional[Engine] = None
Session: Optional[SASession] = None
...

def configure_orm(disable_connection_pool=False):
    """Configure ORM using SQLAlchemy"""
    ...
    global engine
    global Session

    engine = create_engine(SQL_ALCHEMY_CONN, connect_args=connect_args, **engine_args)
    Session = scoped_session(
        sessionmaker(
            autocommit=False,
            autoflush=False,
            bind=engine,
            expire_on_commit=False,
        )
    )
    ...

```
[settings.py](https://github.com/apache/airflow/blob/main/airflow/settings.py)  
위 코드에서 Session 글로벌 변수에 `scoped_session(sessionmaker())`로 데이터베이스 연결 세션을 할당한다. 이제 이 연결은 settings.Session을 통해 Airflow 프로젝트 전체에서 접근 할 수 있다. 다음과 같이 접근 가능하다.
```python
from airflow import settings
...

some_object = SomeObject(id = some_id) # ORM object
settings.Session.add(some_object)
settings.Session.commit()
```
위와 같이 조작 할 수도 있지만, airflow는 [utils/session.py](https://github.com/apache/airflow/blob/main/airflow/utils/session.py) 라이브러리 내에서 session 접근 방법을 정의하고 있다. 이 때 `@provide_session`이 정의된다.
```python
from airflow import settings


@contextlib.contextmanager
def create_session() -> Iterator[settings.SASession]:
    """Contextmanager that will create and teardown a session."""
    session: settings.SASession = settings.Session()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()
...

RT = TypeVar("RT")

def provide_session(func: Callable[..., RT]) -> Callable[..., RT]:
    """
    Function decorator that provides a session if it isn't provided.
    If you want to reuse a session or run the function as part of a
    database transaction, you pass it to the function, if not this wrapper
    will create one and close it for you.
    """
    session_args_idx = find_session_idx(func)

    @wraps(func)
    def wrapper(*args, **kwargs) -> RT:
        if "session" in kwargs or session_args_idx < len(args):
            return func(*args, **kwargs)
        else:
            with create_session() as session:
                return func(*args, session=session, **kwargs)

    return wrapper
```
[utils/session.py](https://github.com/apache/airflow/blob/main/airflow/utils/session.py)  
`create_session()` 함수에서 트랜잭션 커밋과 롤백, 그리고 세션 종료 조건까지 한꺼번에 정의한다. `@contextlib.contextmanager` 데코레이터를 통해 이 함수는 이제 `with` 구문으로 접근 할 수 있다.
`with` 구문이 종료 될 때, 아직 커밋되지 않은 트랜잭션은 문제가 없으면 세션에 커밋하고, 그렇지 않으면 모두 롤백된다. 두 경우 모두 세션은 종료된다.  
`provide_session()`는 `with create_session() as session`을 통해 session을 생성한다. 그리고 이 session을 데이터베이스를 조작하는 함수의 session 파라미터로 전달한다. 데이터베이스에 접근하는 함수들은 `@provide_session` 데코레이터로 session에 데이터베이스 연결 세션을 받을 수 있다. 함수가 종료되고 `provide_session()` 내 with 구문을 벗어나면, 커밋 혹은 롤백 한 뒤 세션은 종료될 것이다.  

## 스키마 관리
Airflow는 alembic으로 Metadata DB 스키마를 관리한다. 
alembic은 sqlalchemy와 호환이 잘 되는 관계형 데이터베이스 migration 관리 도구이다. 스키마를 관리할 때 버전 관리 방식을 사용한다. 즉, 스키마의 변경 사항을 스크립트로 작성하고, `alembic upgrade` 명령어를 사용하면 alembic이 스키마를 자동으로 변경한다.
alembic script의 간단한 예제를 살펴보면
```python
"""Adding extra to Log
Revision ID: 502898887f84
Revises: 52d714495f0
Create Date: 2015-11-03 22:50:49.794097
"""
import sqlalchemy as sa
from alembic import op

# revision identifiers, used by Alembic.
revision = '502898887f84'
down_revision = '52d714495f0'
branch_labels = None
depends_on = None


def upgrade():
    op.add_column('log', sa.Column('extra', sa.Text(), nullable=True))


def downgrade():
    op.drop_column('log', 'extra')

```
[migrations/versions/502898887f84_adding_extra_to_log.py](https://github.com/apache/airflow/blob/main/airflow/migrations/versions/502898887f84_adding_extra_to_log.py)  
위 스크립트는 log 테이블에 새 Column인 extra를 추가한다.
`alembic revision` 명령어를 실행하면 스크립트의 초안이 alembic 스크립트 디렉토리 안에 생성된다. 보통 `alembic revision -m "message"`를 쓴다. `-m` 옵션으로 스크립트 초안에 간단한 메시지를 포함한다. `git commit -m`과 비슷하다. 위 예시의 경우 스크립트 초안을 `alembic revision -m "Adding extra to Log"`로 만들었을 것이다.  
alembic은 터미널에 명령어를 직접 입력해서 실행 할 수도 있지만, 파이썬 코드로도 실행 가능하다.
[utils/db.py](https://github.com/apache/airflow/blob/main/airflow/utils/db.py) 안에 alembic 명령어를 실행하는 함수가 작성 되어있다.
다음 예시는 `alembic upgrade`로 버전 스크립트를 모두 읽어와 스키마를 업그레이드한다.
```python
@provide_session
def upgradedb(session=None):
    """Upgrade the database."""
    # alembic adds significant import time, so we import it lazily
    from alembic import command

    config = _get_alembic_config()

    config.set_main_option('sqlalchemy.url', settings.SQL_ALCHEMY_CONN.replace('%', '%%'))

    errors_seen = False
    for err in _check_migration_errors(session=session):
        if not errors_seen:
            log.error("Automatic migration is not available")
            errors_seen = True
        log.error("%s", err)

    if errors_seen:
        exit(1)

    with create_global_lock(session=session, pg_lock_id=2, lock_name="upgrade"):
        log.info("Creating tables")
        command.upgrade(config, 'heads')
    add_default_pool_if_not_exists()
```
[utils/db.py](https://github.com/apache/airflow/blob/main/airflow/utils/db.py)  
위 예시는 `with create_global_lock()`으로 시작하는 구문에서 `command.upgrade(config, 'heads')`로 데이터베이스 스키마를 업그레이드 한다. 위 함수는 Airflow 커맨드 라인과 연결되어 있다. 쉘에 `airflow upgradedb` 명령어를 입력하면 실행된다. 또 다른 예로, `airflow initdb`는 데이터베이스를 처음 초기화 할 때 사용하는 명령어다. 이 명령어는 `initdb()` 함수를 실행한다. `initdb()`에도 `upgrededb()`가 포함되어 있다.

