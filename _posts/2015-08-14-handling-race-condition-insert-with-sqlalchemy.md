---
title:  "Handling concurrent INSERT with SQLAlchemy"
date:   2015-08-17
description: If you have some unique constraints on a table then you may hit some race condition problem in some cases.    
---

In some cases, you may want to have a unique column other than a primary key id. 
E.g: email, passport number, national id, vat number ... 
If you have some unique constraints on table then you may hit some race condition problem in some edge cases. In this post, we will only cover this case base on Postgres, but I assume that the behavior is similar with other RDBMS: Mysql, Oracle, ... 

To clarify the context, race condition problem doesn't apply to unique constraint when you are using a sequence for the value like a primary key with autoincrement set. The reason is because sequences are not transactional. You can easily test it by opening 2 `psql` and play with sequences. In other words `nexval(sequence)` will never give you the same value, no matter the transaction and the transaction isolation doesn't apply to it. 

To illustrate our problem, we have a model called `Link`. This model is used to stored an url and we don't want any duplicate of urls in the table so we made this column unique.  

{% highlight python %}
from sqlalchemy.ext.declarative import declared_attr, as_declarative
from sqlalchemy import Unicode, Column, Integer

@as_declarative
class Base(object):
    @declared_attr
    def __tablename__(cls):
        return cls.__name__.lower()

    id = Column(Integer, primary_key=True)

engine = create_engine('postgresql://test@/test')
DBSession = scoped_session(sessionmaker(bind=engine))
Base.metadata.bind = engine

class Link(Base):
    url = Column('url', Unicode, index=True, unique=True, nullable=False)
{% endhighlight %}

the `link` table is composed of 2 columns: 
 - an `id` column which is autoincrement primary key 
 - an `url` column with an unique constraint.    

We want to be able to insert new link, but if we try to insert an existing url then we want to retrieve the link of object. 
This operation is commonly referred as `get_or_create` and it's a commonly used in Django.   
If we don't care about race condition then a first version of code may look like this: 

{% highlight python %}

def get_link_by_url(url):
    return DBSession.query(Link).filter(Link.url == url).first()

def get_or_create_link(url):

    # 1. Looking for an existing Link object for these url value
    link = self.get_link_by_url(url)  

    if link:
        # 2. A Link object exist and we return it 
        return link 

    # 3. A Link object doesn't exist so we create an instance
    link = Link(url=url)
    DBSession.add(link) 

    # 4. We insert the link object in the table inside this transaction  
    DBSession.flush()
    return link

{% endhighlight %}

This code assume that you have a transaction block created before calling the function `get_or_create_link`. In the case of a web framework, often a transaction start and end with the request. 

The code above have a problem because it doesn't handle the case 
when a Link object has been inserted in the DB 
between the time we look into the DB if it exists (step 2) and we create the record (step 4). It will be bad luck but it happens! To write the code solving this problem, we need to understand how the queries behave. 
Let's imagine that we have 2 transactions called t1 and t2 and the operations happens as the timeline below.

{% highlight sql %}

_               T1               ___                T2              _
                                  |
                                  |
BEGIN;                            x
                                  |
                                  |
DOING STUFF...                    x   
                                  |
                                  |
SELECT * FROM link                x
WHERE url = 'http://google.com'   |
=> (0 rows)                       |
                                  |
INSERT INTO link (url)            x   BEGIN;
VALUES ('http://google.com');     |       
=> 1 row inserted                 |       
                                  x   DOING STUFF... 
                                  |
                                  |
                                  x   SELECT * FROM link
                                  |   WHERE url = 'http://google.com'
                                  |   => (0 rows)
                                  |
                                  x   INSERT INTO link (url)
                                  |   VALUES ('http://google.com');
                                  |   => waiting
                                  |
                                  |
DOING OTHER STUFF...              x   ... still waiting ...
                                  |
                                  |
COMMIT;                           x   Bam! Integrity Error!  
                                  |
                                  |
                                  x   ROLLBACK;
                                  |
                                  | 
BEGIN;                            x
                                  |
SELECT * FROM link                x
WHERE url = 'http://yahoo.com'    |
=> (0 rows)                       |
                                  |
INSERT INTO link (url)            x   BEGIN;
VALUES ('http://yahoo.com');      |       
=> 1 inserted                     |       
                                  X   SELECT * FROM link 
                                  |   WHERE url = 'http://yahoo.com'
                                  |   => (0 rows)
                                  |
                                  x   INSERT INTO link (url)
                                  |   VALUES ('http://yahoo.com');
                                  |   => waiting
                                  |
                                  |
DOING OTHER STUFF...              x   ... still waiting ...
                                  |
                                  |
ROLLBACK;                         x   => 1 row inserted  
                                  |
                                  V
{% endhighlight %}

The example above try to illustrate the race condition between 2 transactions. 
Luckily for us, RDBMS are welldone and the potentially conflicting insert is 
waiting that other transaction COMMIT or ROLLBACK. 
You can read a bit more about it [here](http://www.postgresql.org/docs/9.1/static/transaction-iso.html#XACT-READ-COMMITTED)

To handle the race condition in this case you want to have a sql behavior as the following:

{% highlight sql %}

_               T1               ___                T2              _
                                  |
                                  |
BEGIN;                            x
                                  |
                                  |
DOING STUFF...                    x   
                                  |
                                  |
SELECT * FROM link                x
WHERE url = 'http://google.com'   |
=> (0 rows)                       |
                                  |
                                  |
SAVEPOINT sp;                     x
                                  |
                                  |
INSERT INTO link (url)            x   BEGIN;
VALUES ('http://google.com');     |       
=> 1 row inserted                 |       
                                  x   DOING STUFF... 
                                  |
                                  |
RELEASE sp;                       x   SELECT * FROM link
                                  |   WHERE url = 'http://google.com'
                                  |   => (0 rows)
                                  |
                                  |
                                  x   SAVEPOINT sp;
                                  |
                                  |
                                  x   INSERT INTO link (url)
                                  |   VALUES ('http://google.com');
                                  |   => waiting
                                  |
                                  |
DOING OTHER STUFF...              x   ... still waiting ...
                                  |
                                  |
COMMIT;                           x   Bam! Integrity Error!  
                                  |
                                  |
                                  x   ROLLBACK sp;
                                  |
                                  |
                                  x   SELECT * FROM link
                                  |   WHERE url = 'http://google.com'
                                  |   => (1 rows)
                                  |
                                  | 
                                  x   DOING OTHER STUFF;
                                  |
                                  |
                                  x   COMMIT;
                                  |
                                  V
{% endhighlight %}

In the example able the race condition got handled and the code can continue as expected without having to rollback all the transaction!    
This workflow can be translated into Python and SQLAlchemy

{% highlight python %}

def get_or_create_link(url):

    # 1. Looking for an existing Link object for these url value
    link = self.get_link_by_url(url)

    if link:
        # 2. A Link object exist and we return it 
        return link

    # 3. A Link object doesn't exist so we create an instance
    link = Link(url=url)

    # 4. We create a savepoint in case of race condition 
    DBSession.begin_nested()
    try:
        DBSession.add(link) 

        # 5. We try to insert and release the savepoint
        DBSession.commit()     
    except IntegrityError, e:
        # 6. The insert fail do to a concurrent transaction  
        DBSession.rollback()
        # 7. We get the Link which exist now
        link = get_link_by_url(url) # 7)
    return link
{% endhighlight %}

This code has been written taking in consideration that the model has only one unique constraint. If you understand it then it will be easy for you to your own version to handle your use case.

Some other people wrote about the similar topic lik is this [post](http://skien.cc/blog/2014/01/15/sqlalchemy-and-race-conditions-implementing/) which try to provide a generic version of `get_or_create`. I personally don't encourage this approach because of the difficulty in dealing with concurrency of the get/create pattern is not something a generic approach can eliminate; decisions will have to be made in how the SELECT or INSERT is to solve the problem is approached (see Mike Bayer [comment](http://skien.cc/blog/2014/01/15/sqlalchemy-and-race-conditions-implementing/#comment-1202648190)).

It worth to mention that there is the useful SQLAlchemy's UniqueObject [recipe](https://bitbucket.org/zzzeek/sqlalchemy/wiki/UsageRecipes/UniqueObject), this recipe use the session to keep track of the unique key which helps in some context.  
I hope this post will help you to understand a bit more race condition even if it only discuss the simple case of INSERT with a unique constraint. With concurrency, different usecase may require different understanding on how the database works: Locking, transaction isolation, etc 
