---
title:  "Postgres full-text search is Good Enough!"
date:   2015-07-13
description: Postgres full-text search is Good Enough!  
share: true
---

When you have to build a web application, you are often asked to add search. The magnifying glass is something that we now add to wireframes without even knowing what we are going to search.

The search has become an important feature and we've seen a big increase in the popularity of tools like [elasticsearch](http://elasticsearch.org/) and [SOLR](https://lucene.apache.org/solr/) which are both based on [lucene](https://lucene.apache.org/). They are great tools but before going down the road of Weapons of Mass ~~Destruction~~ Search, maybe what you need is something a bit lighter which is simply **good enough**! 

What do you I mean by 'good enough'? I mean a search engine with the following features:

- [Stemming](http://en.wikipedia.org/wiki/Stemming)
- Ranking / Boost
- Support Multiple languages
- Fuzzy search for misspelling
- Accent support 

Luckily PostgreSQL supports all these features.

This post is aimed at people who:

- use PostgreSQL and don't want to install an extra dependency for their search engine.
- use an alternative database (eg: MySQL) and have the need for better full-text search features. 
 
In this post, we are going to progressively illustrate some of the full-text search features in Postgres based on the following tables and data:

{% highlight sql %}

CREATE TABLE author(
   id SERIAL PRIMARY KEY,
   name TEXT NOT NULL
);

CREATE TABLE post(
   id SERIAL PRIMARY KEY,
   title TEXT NOT NULL,
   content TEXT NOT NULL,
   author_id INT NOT NULL references author(id) 
);

CREATE TABLE tag(
   id SERIAL PRIMARY KEY,
   name TEXT NOT NULL 
);

CREATE TABLE posts_tags(
   post_id INT NOT NULL references post(id),
   tag_id INT NOT NULL references tag(id)
 );

INSERT INTO author (id, name) 
VALUES (1, 'Pete Graham'), 
       (2, 'Rachid Belaid'), 
       (3, 'Robert Berry');

INSERT INTO tag (id, name) 
VALUES (1, 'scifi'), 
       (2, 'politics'), 
       (3, 'science');

INSERT INTO post (id, title, content, author_id) 
VALUES (1, 'Endangered species', 
        'Pandas are an endangered species', 1 ), 
       (2, 'Freedom of Speech', 
        'Freedom of speech is a necessary right', 2), 
       (3, 'Star Wars vs Star Trek', 
        'Few words from a big fan', 3);


INSERT INTO posts_tags (post_id, tag_id) 
VALUES (1, 3), 
       (2, 2), 
       (3, 1);

{% endhighlight %}

     
It's a traditional blog-like application with `post` objects, which have a `title` and `content`. A `post` is associated to an `author` via a foreign key. A `post` itself can have multiple tags. 
 
### What is Full-Text Search.

First, let's look at the definition:

>In text retrieval, **full-text search** refers to techniques for searching a single computer-stored **document** or a collection in a full-text database. The full-text search is distinguished from searches based on metadata or on parts of the original texts represented in databases.

> -- [Wikipedia](http://en.wikipedia.org/wiki/Full_text_search)

This definition introduces the concept of a document, which is important. When you run a search across your data, you are looking into meaningful entities for which you want to search, these are your documents! The PostgreSQL documentation explains it amazingly.

>A document is the unit of searching in a full-text search system; for example, a magazine article or email message. 

> -- <site>[Postgres documentation](http://www.postgresql.org/docs/9.3/static/textsearch-intro.html#TEXTSEARCH-DOCUMENT)

This document can be across multiple tables and it represents a logical entity which we want to search for.

### Build our document

In the previous section, we introduced the concept of document. A document is not related to our table schema but to data; together these represent a meaningful object.
Based on our example schema, the document is composed of:

- `post.title`
- `post.content`
- `author.name` of the `post`
- all `tag.name` associated to the `post`

To create our document based on this criteria imagine this SQL query:

{% highlight sql %}

SELECT post.title || ' ' || 
       post.content || ' ' ||
       author.name || ' ' ||
       coalesce((string_agg(tag.name, ' ')), '') as document
FROM post
JOIN author ON author.id = post.author_id
JOIN posts_tags ON posts_tags.post_id = posts_tags.tag_id
JOIN tag ON tag.id = posts_tags.tag_id
GROUP BY post.id, author.id;

              document
--------------------------------------------------
Endangered species Pandas are an endangered species Pete Graham politics
Freedom of Speech Freedom of speech is a necessary right Rachid Belaid politics
Star Wars vs Star Trek Few words from a big fan Robert Berry politics
3 rows)

{% endhighlight %}

               
As we are grouping by `post` and `author`, we are using `string_agg()` as the aggregate function because multiple `tag` can be associated to a `post`. Even if `author` is a foreign key and a `post` cannot have more than one `author`, it is required to add an aggregate function for `author` or to add `author` to the `GROUP BY`.

We also used `coalesce()`. When a value can be `NULL` then it's good practice to use the `coalesce()` function, otherwise the concatenation will result in a `NULL` value too.

At this stage, our document is simply a long `string` and this doesn't help us; we need to transform it into the right format via the function `to_tsvector()`.

{% highlight sql %}
SELECT to_tsvector(post.title) || 
       to_tsvector(post.content) ||
       to_tsvector(author.name) ||
       to_tsvector(coalesce((string_agg(tag.name, ' ')), '')) as document
FROM post
JOIN author ON author.id = post.author_id
JOIN posts_tags ON posts_tags.post_id = posts_tags.tag_id
JOIN tag ON tag.id = posts_tags.tag_id
GROUP BY post.id, author.id;
           
               document
 --------------------------------------------------
 'endang':1,6 'graham':9 'panda':3 'pete':8 'polit':10 'speci':2,7
 'belaid':12 'freedom':1,4  'necessari':9 'polit':13 'rachid':11 'right':10 'speech':3,6
 'berri':13 'big':10 'fan':11 'polit':14 'robert':12 'star':1,4 'trek':5 'vs':3 'war':2 'word':7
(3 rows)         
{% endhighlight %}
      
This query will return our document as `tsvector` which is a type suited to full-text search. Let's try to convert a simple string into a tsvector.

{% highlight sql %}
SELECT to_tsvector('Try not to become a man of success, but rather try to become a man of value');

query will return the following result:

                             to_tsvector
----------------------------------------------------------------------
'becom':4,13 'man':6,15 'rather':10 'success':8 'tri':1,11 'valu':17
(1 row)
{% endhighlight %}

Something weird just happened. First there are fewer words than in the original sentence, some of the words are different (`try` became `tri`) and they are all followed by numbers. Why?

A `tsvector` value is a sorted list of distinct lexemes which are words that have been normalized to make different variants of the same word look alike.
For example, normalization almost always includes folding upper-case letters to lower-case and often involves removal of suffixes (such as 's', 'es' or 'ing' in English). This allows searches to find variant forms of the same word without tediously entering all the possible variants.

The numbers represent the location of the lexeme in the original string. For example, "man" is present at position 6 and 15. Try counting the words and see for yourself.

By default, Postgres uses `'english'` as text search configuration for the function `to_tsvector` and it will also ignore english stopwords. 
That explains why the `tsvector` results have fewer elements than the ones in our sentence. We see later a bit more about languages and text search configuration.


### Querying

We have seen how to build a document, but the goal here is to find the document. For running a query against a `tsvector` we can use the `@@` operator which is documented [here](http://www.postgresql.org/docs/9.3/static/functions-textsearch.html#FUNCTIONS-TEXTSEARCH). Let's see some examples on how to query our document.

{% highlight sql %}
> select to_tsvector('If you can dream it, you can do it') @@ 'dream';
 ?column?
----------
 t
(1 row)

> select to_tsvector('It''s kind of fun to do the impossible') @@ 'impossible';

 ?column?
----------
 f
(1 row)
{% endhighlight%}

The second query returns false because we need to build a `tsquery` which creates the same lexemes and, using the operator `@@`, casts the string into a tsquery. The following shows the difference between casting and using the function `to_tsquery()`

{% highlight sql %}
SELECT 'impossible'::tsquery, to_tsquery('impossible');
   tsquery    | to_tsquery
--------------+------------
 'impossible' | 'imposs'
(1 row)
{% endhighlight%}
    
But in the case of 'dream' the stem is equal to the word.

{% highlight sql %}
SELECT 'dream'::tsquery, to_tsquery('dream');
   tsquery    | to_tsquery
--------------+------------
 'dream'      | 'dream'
(1 row)
{% endhighlight%}
    
    
From now on we will use `to_tsquery` for querying documents.

{% highlight sql %}
SELECT to_tsvector('It''s kind of fun to do the impossible') @@ to_tsquery('impossible');

 ?column?
----------
 t
(1 row)
{% endhighlight %}
    
A tsquery value stores lexemes that are to be searched for, and combines them honoring the Boolean operators & (AND), | (OR), and ! (NOT). Parentheses can be used to enforce grouping of the operators
 
 
{% highlight sql %}

> SELECT to_tsvector('If the facts don't fit the theory, change the facts') @@ to_tsquery('! fact');

 ?column?
----------
 f
(1 row)

> SELECT to_tsvector('If the facts don''t fit the theory, change the facts') @@ to_tsquery('theory & !fact');

 ?column?
----------
 f
(1 row)

> SELECT to_tsvector('If the facts don''t fit the theory, change the facts.') @@ to_tsquery('fiction | theory');

 ?column?
----------
 t
(1 row)

{% endhighlight %}

We can also use startwith query style when using `:*`.

{% highlight sql %}
> SELECT to_tsvector('If the facts don''t fit the theory, change the facts.') @@ to_tsquery('theo:*');

 ?column?
----------
 t
(1 row)
{% endhighlight %}
    
Now that we know how to make a full-text search query, we can come back to our initial table schema and try to query our documents.

{% highlight sql %}
SELECT pid, p_title
FROM (SELECT post.id as pid,
             post.title as p_title,
             to_tsvector(post.title) || 
             to_tsvector(post.content) ||
             to_tsvector(author.name) ||
             to_tsvector(coalesce(string_agg(tag.name, ' '))) as document
      FROM post
      JOIN author ON author.id = post.author_id
      JOIN posts_tags ON posts_tags.post_id = posts_tags.tag_id
      JOIN tag ON tag.id = posts_tags.tag_id
      GROUP BY post.id, author.id) p_search
WHERE p_search.document @@ to_tsquery('Endangered & Species');

 pid |      p_title
-----+--------------------
   1 | Endangered species
(1 row)
{% endhighlight %}
    
This will find our document which contains `Endangered` and `Species` or lexemes close enough.

### Language support

Postgres provides built-ins text search for many languages: Danish, Dutch, English, Finnish, French, German, Hungarian, Italian, Norwegian, Portuguese, Romanian, Russian, Spanish, Swedish, Turkish.

{% highlight sql %}
SELECT to_tsvector('english', 'We are running');
 to_tsvector
-------------
 'run':3
(1 row)

SELECT to_tsvector('french', 'We are running');
        to_tsvector
----------------------------
 'are':2 'running':3 'we':1
(1 row)
{% endhighlight %}
    
A column name can be used to create the `tsvector` based on our starting model. Let's assume that `post` can be written in different languages and post contains a column `language`.

{% highlight sql %}
ALTER TABLE post ADD language text NOT NULL DEFAULT('english');
{% endhighlight %}
    
We can now rebuild our document to use this language column.

{% highlight sql %}
SELECT to_tsvector(post.language::regconfig, post.title) || 
       to_tsvector(post.language::regconfig, post.content) ||
       to_tsvector('simple', author.name) ||
       to_tsvector('simple', coalesce((string_agg(tag.name, ' ')), '')) as document
FROM post
JOIN author ON author.id = post.author_id
JOIN posts_tags ON posts_tags.post_id = posts_tags.tag_id
JOIN tag ON tag.id = posts_tags.tag_id
GROUP BY post.id, author.id;
{% endhighlight %}

Without the explicit cast `::regconfig` the query will have generated an error:

{% highlight sql %}
ERROR:  function to_tsvector(text, text) does not exist
{% endhighlight %}
    
`regconfig` is the object identifier type which represents the text search configuration in Postgres: http://www.postgresql.org/docs/9.3/static/datatype-oid.html

Fow now the lexemes of our document will be built using the right language based on `post.language`.

We also used `simple` which is one of the built in search text configs that Postgres provides. `simple` doesn't ignore stopwords and doesn't try to find the [stem](http://en.wikipedia.org/wiki/Stemming) of the word. With `simple` every group of characters separated by a space is a lexeme; the `simple` text search config is pratical for data like a person's name for which we may not want to find the [stem](http://en.wikipedia.org/wiki/Stemming) of the word. 

{% highlight sql %}
SELECT to_tsvector('simple', 'We are running');
        to_tsvector
----------------------------
 'are':2 'running':3 'we':1
(1 row)
{% endhighlight %}

### Accented Character

When you build a search engine supporting many languages you will also hit the accent problem. In many languages, accents are very important and can change the meaning of the word. Postgres ships with an extension call `unaccent` which is useful to unaccentuate content.  

{% highlight sql %}
CREATE EXTENSION unaccent;
SELECT unaccent('èéêë');

 unaccent
----------
 eeee
(1 row)
{% endhighlight %}

Let's add some accented content to our `post` table.

{% highlight sql %}
INSERT INTO post (id, title, content, author_id, language) 
VALUES (4, 'il était une fois', 'il était une fois un hôtel ...', 2,'french')
{% endhighlight %}

If we want to ignore accents when we build our document, then we can simply do the following:

{% highlight sql %}
SELECT to_tsvector(post.language, unaccent(post.title)) || 
       to_tsvector(post.language, unaccent(post.content)) ||
       to_tsvector('simple', unaccent(author.name)) ||
       to_tsvector('simple', unaccent(coalesce(string_agg(tag.name, ' '))))
FROM post
JOIN author ON author.id = post.author_id
JOIN posts_tags ON posts_tags.post_id = posts_tags.tag_id
JOIN tag ON author.id = post.author_id
GROUP BY p.id
{% endhighlight %}


That works, but it's a bit cumbersome with more room for mistakes. We can also build a new text search config with support for unaccented characters.
 
{% highlight sql %}
CREATE TEXT SEARCH CONFIGURATION fr ( COPY = french );
ALTER TEXT SEARCH CONFIGURATION fr ALTER MAPPING
FOR hword, hword_part, word WITH unaccent, french_stem;
{% endhighlight %}

When we are using this new text search config, we can see the lexemes 

{% highlight sql %}
SELECT to_tsvector('french', 'il était une fois');
 to_tsvector
-------------
 'fois':4
(1 row)

SELECT to_tsvector('fr', 'il était une fois');
    to_tsvector
--------------------
 'etait':2 'fois':4
(1 row)
{% endhighlight %}
    
This gives us the same result as applying `unaccent` first and building the `tsvector` from the result.

{% highlight sql %}
SELECT to_tsvector('french', unaccent('il était une fois'));
    to_tsvector
--------------------
 'etait':2 'fois':4
(1 row)
{% endhighlight %}
 
The number of lexemes is different because `il était une` are stopwords in French. Is it an issue to keep these stop words in our document? I don't think so as `etait` is not really a stopword as it's misspelled.

{% highlight sql %}
SELECT to_tsvector('fr', 'Hôtel') @@ to_tsquery('hotels') as result;
 result
--------
 t
(1 row)
{% endhighlight %}

If we create an unaccented search config for each language that our `post` can be written in and we keep this value in `post.language` then we can keep our previous document query. 

{% highlight sql %}
SELECT to_tsvector(post.language, post.title) || 
       to_tsvector(post.language, post.content) ||
       to_tsvector('simple', author.name) ||
       to_tsvector('simple', coalesce(string_agg(tag.name, ' ')))
FROM post
JOIN author ON author.id = post.author_id
JOIN posts_tags ON posts_tags.post_id = posts_tags.tag_id
JOIN tag ON author.id = post.author_id
GROUP BY p.id
{% endhighlight %}

If you need to create unaccented text search config for each language supported by Postgres then you can use this [gist](https://gist.github.com/rach/9289959)

Our document will now likely increase in size because it can include unaccented stopwords but we query it without caring about accented characters. This can be useful e.g. for somebody with an english keyboard searching french content.

### Ranking

When you build a search engine you want to be able to get search results ordered by relevance. The ranking of documents is based on many factors which are roughly explained in this documentation.

>Ranking attempts to measure how relevant documents are to a particular query, so that when there are many matches the most relevant ones can be shown first. PostgreSQL provides two predefined ranking functions, which take into account lexical, proximity, and structural information; that is, they consider how often the query terms appear in the document, how close together the terms are in the document, and how important is the part of the document where they occur.

>-- [PostgreSQL documentation](http://www.postgresql.org/docs/9.3/static/textsearch-controls.html#TEXTSEARCH-RANKING)

To order our results by relevance PostgreSQL provides a few functions but in our example we will be using only 2 of them: `ts_rank()` and `setweight()`.

The function `setweight` allows us to assign a weight value to a `tsvector`; the value can be `'A', 'B', 'C' or 'D' `

{% highlight sql %}
SELECT pid, p_title
FROM (SELECT post.id as pid,
             post.title as p_title,
             setweight(to_tsvector(post.language::regconfig, post.title), 'A') || 
             setweight(to_tsvector(post.language::regconfig, post.content), 'B') ||
             setweight(to_tsvector('simple', author.name), 'C') ||
             setweight(to_tsvector('simple', coalesce(string_agg(tag.name, ' '))), 'B') as document
      FROM post
      JOIN author ON author.id = post.author_id
      JOIN posts_tags ON posts_tags.post_id = posts_tags.tag_id
      JOIN tag ON tag.id = posts_tags.tag_id
      GROUP BY post.id, author.id) p_search
WHERE p_search.document @@ to_tsquery('english', 'Endangered & Species')
ORDER BY ts_rank(p_search.document, to_tsquery('english', 'Endangered & Species')) DESC;
{% endhighlight %}

In the query above, we have assigned different weights to the different fields of a document. `post.title` is more important than the `post.content` and as important as the `tag` associated. The least important is the `author.name`. 

This means that if we were to search for the term 'Alice' a document that contains that term in its title would be returned before a document that contains the term in its content and document that with an author of that name would be returned last.

Based on the weights assigned to parts of our document the `ts_rank()` returns a floating number which represents the relevancy of our document against the query.

{% highlight sql %}
SELECT ts_rank(to_tsvector('This is an example of document'), 
               to_tsquery('example | document')) as relevancy;
 relevancy
-----------
 0.0607927
(1 row)


SELECT ts_rank(to_tsvector('This is an example of document'), 
               to_tsquery('example ')) as relevancy;
 relevancy
-----------
 0.0607927
(1 row)


SELECT ts_rank(to_tsvector('This is an example of document'), 
               to_tsquery('example | unkown')) as relevancy;
 relevancy
-----------
 0.0303964
(1 row)


SELECT ts_rank(to_tsvector('This is an example of document'),
               to_tsquery('example & document')) as relevancy;
 relevancy
-----------
 0.0985009
(1 row)


SELECT ts_rank(to_tsvector('This is an example of document'), 
               to_tsquery('example & unknown')) as relevancy;
 relevancy
-----------
 1e-20
(1 row)
{% endhighlight %}

However, the concept of relevancy is vague and very application-specific. Different applications might require additional information for ranking, e.g., document modification time. The built-in ranking functions such as `ts_rank` are only examples. You can write your own ranking functions and/or combine their results with additional factors to fit your specific needs. 

To illustrate the paragraph above, if we wanted to promote newer posts against older ones we could divide the ts_rank value by the age of the document +1 (to avoid dividing by zero).

### Optimization and Indexing

Optimizing the search on one table is straight forward. PostgreSQL supports function based indexes so you can simply create a GIN index around the `tsvector()` function.

{% highlight sql %}

CREATE INDEX idx_fts_post ON post 
USING gin(setweight(to_tsvector(language, title),'A') || 
	   setweight(to_tsvector(language, content), 'B'));

{% endhighlight %}


GIN or GiST indexes? These two indexes could be subject of a blog post themselves. GiST can produce false matches which then requires a extra table row lookup to confirm the match. On the other hand, GIN is faster to query but bigger and slower to build.

> As a rule of thumb, GIN indexes are best for static data because lookups are faster. For dynamic data, GiST indexes are faster to update. Specifically, GiST indexes are very good for dynamic data and fast if the number of unique words (lexemes) is under 100,000 while GIN indexes will handle 100,000+ lexemes better but are slower to update.

> -- [Postgres doc : Chap 12 Full Text Search](http://www.postgresql.org/docs/9.1/static/textsearch-indexes.html)

For our example, we will be using GIN but the choice can be argued and you need to take your own decision based on your data.

We have a problem in our schema example; the document is spread across multiple tables with different weights. For a better performance it's necessary to denormalize the data via triggers or materialized view. 

You don't always need to denormalise and in some cases you can add a function based index as we did above. Alternatively you can easily denormalise data from the same table via the postgres trigger function `tsvector_update_trigger(...)` or `tsvector_update_trigger_column(...)`. See the [Postgres doc](http://www.postgresql.org/docs/9.3/static/textsearch-features.html#TEXTSEARCH-UPDATE-TRIGGERS) for more detailed information.

If it's acceptable of having some delay before a document can be found via search query then this may be a good use case for using a Materialized View so we can build an extra index on it.


{% highlight sql %}
CREATE MATERIALIZED VIEW search_index AS 
SELECT post.id,
       post.title,
       setweight(to_tsvector(post.language::regconfig, post.title), 'A') || 
       setweight(to_tsvector(post.language::regconfig, post.content), 'B') ||
       setweight(to_tsvector('simple', author.name), 'C') ||
       setweight(to_tsvector('simple', coalesce(string_agg(tag.name, ' '))), 'A') as document
FROM post
JOIN author ON author.id = post.author_id
JOIN posts_tags ON posts_tags.post_id = posts_tags.tag_id
JOIN tag ON tag.id = posts_tags.tag_id
GROUP BY post.id, author.id
{% endhighlight %}

Then reindexing the search engine will be as simple as periodically running   `REFRESH MATERIALIZED VIEW search_index;`.

We can now add an index on the materialized view.
   
{% highlight sql %}
CREATE INDEX idx_fts_search ON search_index USING gin(document);
{% endhighlight %}
     
And querying will become much simpler too.
    
{% highlight sql %}
SELECT id as post_id, title
FROM search_index
WHERE document @@ to_tsquery('english', 'Endangered & Species')
ORDER BY ts_rank(p_search.document, to_tsquery('english', 'Endangered & Species')) DESC;
{% endhighlight %}


If you cannot afford delay then you may have to investigate the alternative method using triggers.

There is not one way to build your document store; it will depend on what comprises your document: single table, multiple table, multiple languages, amount of data ...

[Thoughtbot.com](http://thoughtbot.com/) published a good article on ["Implementing Multi-Table Full Text Search with Postgres in Rails"](http://robots.thoughtbot.com/implementing-multi-table-full-text-search-with-postgres) which I advise reading.


### Mispelling

PostgreSQL comes with a very useful extenstion called `pg_trgm`. See [pg_trgm doc](http://www.postgresql.org/docs/9.3/static/pgtrgm.html).

{% highlight sql %}
CREATE EXTENSION pg_trgm;
{% endhighlight %}

 This provides support for trigram which is a [N-gram with](http://en.wikipedia.org/wiki/N-gram) `N == 3`. N-grams are useful because they allow finding strings with similar characters and, in essence, that's what a misspelling is - a word that is similar but not quite right.

{% highlight sql %}
SELECT similarity('Something', 'something');
 similarity
------------
     1
(1 row)

SELECT similarity('Something', 'samething');
 similarity
------------
  0.538462
(1 row)

SELECT similarity('Something', 'unrelated');
 similarity
------------
     0
(1 row)

SELECT similarity('Something', 'everything');
 similarity                                          
------------
   0.235294
(1 row)

SELECT similarity('Something', 'omething');
 similarity
------------
   0.583333
(1 row)
{% endhighlight %}
    
With the examples above you can see that similarity returns a float to represent the similarity between two strings. To detect misspelling is then a matter of collecting the lexemes used by our documents and comparing the similarities with our search input. I found that 0.5 is a good number to test the similarity of misspelling. First we need to create this list of unique lexemes used by our documents.

    
{% highlight sql %}
CREATE MATERIALIZED VIEW unique_lexeme AS
SELECT word FROM ts_stat(
'SELECT to_tsvector('simple', post.title) || 
    to_tsvector('simple', post.content) ||
    to_tsvector('simple', author.name) ||
    to_tsvector('simple', coalesce(string_agg(tag.name, ' ')))
FROM post
JOIN author ON author.id = post.author_id
JOIN posts_tags ON posts_tags.post_id = posts_tags.tag_id
JOIN tag ON tag.id = posts_tags.tag_id
GROUP BY post.id, author.id');
{% endhighlight %}
     
The query above builds a view with one column called `word` from all the unique lexemes of our documents. We used `simple` because our content can be in multiple languages. Once we create this materialized view we need to add an index to make the similarity query faster.
     
{% highlight sql %}
CREATE INDEX words_idx ON search_words USING gin(word gin_trgm_ops);
{% endhighlight %}

Luckily unique lexemes used in a search engine is not something that will change rapidly, so probably we won't have to refresh the materialized view too often via:  
  
{% highlight sql %}
REFRESH MATERIALIZED VIEW unique_lexeme;
{% endhighlight %}
  
Once we have built this table finding the closest match is very simple.

{% highlight sql %}
SELECT word 
WHERE similarity(word, 'samething') > 0.5 
ORDER BY word <-> 'samething'
LIMIT 1; 
{% endhighlight %}

This query returns a lexeme which is similar enough (`>0.5`) to the search input `samething` ordered by the closest first. The operator `<->` returns the "distance" between the arguments, that is one minus the `similarity()` value. 

When you decide to handle misspelling in your search you may want to not look for misspellings on every query. Instead, you could query for misspellings only when the search returns no results and use the results of that query to provide some suggestions to your user. It is also possible that your data may contain misspellings if it comes from some source of informal communication such as a social network in which case you may obtain good results by appending the similar lexeme to your `tsquery`.

["Super Fuzzy Searching on PostgreSQL"](http://bartlettpublishing.com/site/bartpub/blog/3/entry/350) is a good reference article about the use of trigrams for using misspellings and search with Postgres. 

In my use case the unique lexemes table has never been bigger than 2000 rows but from my understanding if you have more 1M unique lexemes used across your document then you may be meet performance issues with this technique.

### About MySQL and RDS 

#### Does it work on Postgres RDS? 

All the examples illustrated work on RDS. From what I'm aware the only restrictions on search features imposed by RDS are those that require access to the file system such as custom dictionaries, ispell, synonyms and thesaurus. See [the related issue on the aws forum](https://forums.aws.amazon.com/message.jspa?messageID=527000#527000).

#### I'm using MYSQL should I use the builtin full-text search?

I wouldn't. Without starting a flame war, MySQL full-text search features are very limited. By default, there is no support for stemming nor any language support. I came accross a stemming function which can be installed, but MYSQL doesn't support function based indexes.

Then what can you do? Based on what we have discussed above if Postgres fulfills your use case then think about moving to Postgres. This can easily be done via tools like [py-mysql2pgsql](https://github.com/philipsoutham/py-mysql2pgsql/). Or you can investigate more advanced solutions like SOLR and Elasticsearch. 


###Conclusion

We have seen how to build a decent multi-language search engine based on a non-trivial document. This article is only an overview, but it should give you enough background and examples to get you started with your own. I may have made some mistakes in this article and I would appreciate if you report them to blog@lostpropertyhq.com

The full-text search feature included in Postgres is awesome and quite fast (enough). It will allow your application to grow without depending on another tool. Is Postgres Search the silver bullet? Probably not if your core business needs revolve around search. 

Some features are missing but in a lot of use cases you won't need them. It goes without saying that it's critical that you analyze and understand your needs to know which road to take.

Personally I hope to see the full-text search continuing to improve in Postgres and maybe a few of these features being included:

- Additional built-in language support. eg: Chinese, Japanese...
- Foreign data wrapper around Lucene. Lucene is still the most advanced tool for full-text search and it will have a lot of benefits to see integration with Postgres.
- More boost or scoring feature for the ranking of results would be first-rate. Elasticsearch and SOLR offer advanced solutions already.
- A way to do fuzzy `tsquery` without having to use trigram would be nice. Elasticsearch offers a simple way to do fuzzy search queries.
- Being able to create and edit dynamically features such as dictionary content, synonyms, thesaurus via SQL this removing the need to add files to the filesystem


Postgres is not as advanced as ElasticSearch and SOLR, but these two are dedicated full-text search tools whereas full-text search is only a feature of PostgreSQL and a pretty good one.
