---
title:  "Prefetch ID with SQLAlchemy and PosgreSQL"
date:   2015-08-18
description: If need to know the value of the autoincrement before inserting, this post shows you how to get it.    
---

For some projects, I wanted to store external URL as short link (like bitly) so an harmful URLs could be block globaly in the system.  To not expose the primary key id, I used [hashid](http://hashids.org/python/) which can generates short, unique, non-sequential ids from numbers.

PostgreSQL has sequence and when you define a primary key Integer column, 
SQLAlchemy will automatically create a SERIAL [type](http://www.postgresql.org/docs/9.4/static/datatype-numeric.html#DATATYPE-SERIAL). SERIAL are not true types, but merely a notational convenience for creating unique identifier columns.

{% highlight sql %}
CREATE TABLE tablename (
    colname SERIAL
);
{% endhighlight %}

is equivalent to specifying:

{% highlight sql %}
CREATE SEQUENCE tablename_colname_seq;
CREATE TABLE tablename (
    colname integer NOT NULL DEFAULT nextval('tablename_colname_seq')
);
ALTER SEQUENCE tablename_colname_seq OWNED BY tablename.colname;
{% endhighlight %}


To illustrate our problem, we have a model called `Link`. This model is used to stored an URL and a corresponding hashid for the short URL.  

{% highlight python %}
from sqlalchemy.ext.declarative import declared_attr, as_declarative
from sqlalchemy import Unicode, String, Column, Integer

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
    hashid = Column('url', String, index=True, unique=True, nullable=False)
{% endhighlight %}

Even if technically the transactions protect us from accident insert by the application. I want to avoid having nullable column as it can compromise the integrity of our db with only exception if it's necessary or having null column makes sens.  In our case, having a link object without hashid doesn't make sense. If we were to allow NULL for the hashid column then we could have a function as the following to create Link object.

{% highlight python %}
from hashids import Hashids

def create_link(url):
    hashids = Hashids()
    link = Link(
        id=nextid,
        url=url,
        )
    DBSession.add(link)
    DBSession.flush(link)
    link.hashid = hashids.encode(link.id)
    DBSession.commit(link)
    return link
    
{% endhighlight %}

In the case, we decided not to allow NULL in the hashid column then we want to know what will be the id before inserting the Link object in the DB. Luckily, we have sequence allow us to do that.
The SERIAL type is automatically creating a sequence of the format `tablename_columm_seq`.
Using that we can rewrite the function to create a Link object.

{% highlight python %}
from hashids import Hashids
from sqlalchemy.schema import Sequence


def create_link(url):
    hashids = Hashids()
    nextid = DBSession.execute(Sequence("link_id_seq"))
    hashid = hashids.encode(nextid)
    link = Link(
        id=nextid,
        url=url,
        hashid=hashid
    )
    DBSession.add(link)
    DBSession.commit(link)
    return link
    
{% endhighlight %}

Because the sequence are non-transactionnal, then there is no problem with prefetching the id from it. 

I hope this post may help if you hit a similar use case, I can imagine some situation when I wanted to know the ID before inserting my record.
