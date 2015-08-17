---
title:  "Prefetch ID with SQLAlchemy and PosgreSQL"
date:   2015-08-18
description: If you need to know the value of the row id before INSERT, this post shows you how.    
---

I wanted to store external URL of project as short link (a bit like bitly) so a harmful URL could be block globally in the system. To avoid making the URL being able to be guessed, I used [hashids](http://hashids.org/python/) which generates short, unique, non-sequential ids from numbers. With hashids I can easily convert the primary key id into an alternative id which can be exposed.

    http://mydomain.com/AW3d -> http://google.com 
    (301 Moved Permanently to http://google.com)  

To illustrate our problem, we have a model called `Link`. This model is used to stored a URL and a corresponding hashid for the short URL.  

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

Even if a transaction block protect us from an accidental INSERT by the application error halfway. I want to avoid having nullable column if I can to preserve the integrity of our db. In some cases, having a NULL value can make sens but we don't want a link object without hashid.

To be able to create the hashid before insert, we need to know what will be the primary id. Luckily, PostgreSQL has sequences and when you define a primary key Integer column, SQLAlchemy will automatically use the SERIAL [type](http://www.postgresql.org/docs/9.4/static/datatype-numeric.html#DATATYPE-SERIAL). SERIAL are not true types, but a notational convenience for creating unique identifier columns. Behind the scene, Postgres create a not null Integer column with a default value equal to the sequence next value. 

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

Let see first, a solution if we were to allow NULL for the hashid column. A function to create a Link object could look like this:

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

In our case, we decided not to allow NULL in the hashid column so the code above would give us a violation error of the not-null constraint. 

The SERIAL type created a sequence of the format `tablename_columm_seq` and using the sequence we can prefetch the id. The rewritten function to create a Link object using sequence could look like this:

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

Because the sequences are non-transactional, then there is no problem with prefetching the id from it. I'm not aware of any problems that could happen. 

I hope this post may help if you hit a similar use case, I can imagine some situation when I wanted to know the ID before inserting my record.
