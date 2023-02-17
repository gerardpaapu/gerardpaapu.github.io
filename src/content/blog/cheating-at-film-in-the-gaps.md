---
title: 'Cheating at Film in the gaps'
description: 'How I learned a little bit about SQL'
pubDate: 'Feb 18 2023'
---

# Cheating at Film in the Gaps

A friend of mine makes a daily puzzle, "Film in the Gaps" where the objective is
to find credits (actors, producers, directors, etc) to join movies in a daisy
chain from one movie to another, in the fewest steps

[Go try it now, it'll make more sense](https://filminthega.ps)

It's like the Kevin Bacon game, but instead of going from actor to actor, we're going
from movie to movie and everyone plays the same two movies every day.

I'm pretty bad at movie trivia, but I'm okay at computers so ... I should be
able to figure out how to cheat at this game right?

My first thoughts were hitting a movie data API. Film in the Gaps uses data from
[tmdb](https://themoviedb.org/) and they do have an API, but I couldn't imagine
if I had to wait half a second for an HTTP request to complete how I would ever
complete a search like this in a reasonable time.

So I think I'm looking for a solution in a local database.

After a little googling, I found that [IMDB](https://www.imdb.com/) publishes partial dumps of their
database for free. [You can get them here](https://datasets.imdbws.com/)

So I downloaded a bunch of them. I got `title.basics`, `title.principals`, and `name.basics`

I needed to load them into a database so I started reading the `sqlite3` docs. SQlite is
my weapon-of-choice for this sort of thing, I'm not a data-scientist or statistician so this might
be more about familiarity than anything else, but broadly I'm a big fan.

BTW, when I show the commands I'm going show only the important stuff but when I was actually doing this
I was running `gzip -h` or `sqlite> .help` all the time.

So, I needed to unzip each file and import it into sqlite. This might be obvious, but
I didn't realise that you could import `tsv` and `csv` from the sqlite3 cli.

```sh
$ gzip -d title.basics.tsv.gz
$ sqlite3 data.sqlite3
SQLite version 3.31.1 2020-01-27 19:55:54
Enter ".help" for usage hints.
sqlite> .mode tsv
sqlite> .import title.basics.tsv TEMP.title_basics
```

I imported each of these into the `TEMP` database because I'm pretty sure I'm
going to throw away a lot of this data before it's in a suitable format

So now I've got three tables.

`TEMP.title_basics` - all the important information about a "title" including
the name, whether it's a tv show episode or a movie, and whether it's... uhhh...
adult.

`TEMP.name_basics` - all the important information about a "name" (a person)
including their actual name and a bunch of stuff we don't care about really

`TEMP.title_principals` - the main credits for each title, so ... essentially
it's a join table between `title_basics` and `name_basics`, i.e. which `name`s
are in each `title`

Across each of these tables we use two fields as ids `tconst` the id of a title and
`nconst`, the id of a `name`. `title_principals` uses one the `tconst` and `nconst`
and never the real name of a movie

In theory I can start searching already, and I tried to but the queries were so
complicated and so slow that I couldn't really get it done.

I don't know if this is a good time to bring this up, but I'm really not very
good at SQL, so ... a lot of this is feeling around in the dark.

So this is how we would find a link between two titles if they are connected by
a common name (2 steps).

```SQL
SELECT *
FROM TEMP.title_principals start
JOIN TEMP.title_principals end ON end.nconst = start.nconst -- the same actor is in both titles
WHERE start.tconst = 'tt0236348' -- Josie and the Pussycats
AND   end.tconst   = 'tt0110598' -- Muriel's Wedding
```

This isn't really how the puzzles are designed though.

Usually they will be at least 4 steps. Start -> Name -> Title -> Name -> End

So let's extend this by one title.

```SQL
SELECT *
FROM TEMP.title_principals start
JOIN TEMP.title_principals mid ON mid.nconst = start.nconst -- The same actor is in start and mid
JOIN TEMP.title_principals end ON end.nconst = end.nconst   -- The same actor is in mid and end
WHERE start.tconst = 'tt0236348' -- Josie and the Pussycats
AND   end.tconst   = 'tt0110598' -- Muriel's Wedding
```

So ... I won't tell you how long I squinted at this, but it was a while.

This query will only connect the titles if they are all connected by the same name,
e.g. if there's one or more actor who is in all three movies

Now I'm wandering in the desert (you can skip this step if you're good at SQL)

Maybe I can ... **send in more joins**?

```SQL
SELECT *
FROM TEMP.title_principals start
JOIN TEMP.title_principals start_name ON start.nconst = start_name.nconst -- all the actors in start
JOIN TEMP.title_principals mid        ON start_name.tconst = mid.nconst   -- all the films those actors have been in
JOIN TEMP.title_principals mid_name   ON mid.nconst = mid_name.nconst     -- all the actors in those films
JOIN TEMP.title_principals end        ON mid_name.tconst = mid.nconst     -- all the films those actors have been in
WHERE start.tconst = 'tt0236348' -- Josie and the Pussycats
AND   end.tconst   = 'tt0110598' -- Muriel's Wedding
```

I don't know if you've ever run a query that joins five tables, but I believe it's not BeSt PrAcTiCe for performance.

This search takes like ... several minutes on my machine. It's not ideal. (Also, I think it might just be wrong).

I googled around for "HOW MAKE SQLITE FAST", and came across a command called `EXPLAIN QUERY PLAN`
which will get sqlite3 to show you what it's going to do while running your query

```
QUERY PLAN
|--SCAN TABLE title_principals AS start
|--SEARCH TABLE title_principals AS start_name USING AUTOMATIC COVERING INDEX (nconst=?)
|--SEARCH TABLE title_principals AS mid USING AUTOMATIC PARTIAL COVERING INDEX (tconst=?)
|--SEARCH TABLE title_principals AS mid_name USING AUTOMATIC COVERING INDEX (nconst=?)
`--SEARCH TABLE title_principals AS end USING AUTOMATIC PARTIAL COVERING INDEX (tconst=?)
```

Now, my friends tell me that scanning is bad and searching is good. Searching
means it can use some sort of index to find what it's looking for scanning means
that it's looking at every flippin' row in the table. Searching with a "partial"
covering index is also worse than using a covering index.

SQLite has generated an automatic covering index for `nconst` (names) but only
a partial for `tconst` (titles)

So ... it's going to look at every row once, and then it can skip a bunch of
rows while it's doing the other searches

and it's... very slow

So I added two indexes to the table:

```sql
CREATE INDEX principals_tconst ON principals(tconst);
CREATE INDEX principals_nconst ON principals(nconst)
```

This led to a much better query plan (I think), and the same query runs almost instantly

```
QUERY PLAN
|--SEARCH TABLE principals AS mid_name USING INDEX principals_tconst (tconst=?)
|--SEARCH TABLE principals AS mid USING INDEX principals_nconst (nconst=?)
|--SEARCH TABLE principals AS start_name USING INDEX principals_tconst (tconst=?)
|--SEARCH TABLE principals AS start USING INDEX principals_nconst (nconst=?)
`--SEARCH TABLE principals AS end USING INDEX principals_tconst (tconst=?)
```

but it's also not finding the solutions... so I've got something wrong, and also the
query is really intense.

So I spent some time away from the problem, staring into space, that sort of thing.

(I'm imagining any data-folk have died from intense eye-rolling by now)

It came to me in a dream that if I had a table that modelled what I cared about
my queries would make more sense.

What I **actually care about** is "Can I connect movie A to movie B, and if so... how?"

So what I really want in a row is:

1. a movie
2. another movie that is connected to it
3. a name (actor, director, etc.) that connects them

First I wanted to get rid of any titles I didn't care about so ... first I
created a new table `movie_principals` which only includes movies (not TV shows
or Webisodes), and excludes "Adult" films.

```sql
CREATE TABLE movie_principals AS
SELECT
    title_principals.*,
    t.primaryTitle, -- the actual title of a film
    n.primaryName
FROM title_principals
JOIN title_basics t ON title_principals.tconst = p.tconst
JOIN name_basics n ON title_principals.nconst = n.tconst
WHERE t.type    = 'movie'
AND   t.isAdult = 0 -- sweatdrop
```

This gave me a subset of the total data set that I actually cared about. It also
attached the details from the other tables that I cared about.

This isn't very normalised, but it made the queries I was going to write easier.

Then I created a new table with the schema I wanted.

`left_id` is the `tconst` of a film
`right_id` is the `tconst` of another film
`name_id` is the `nconst` of a name they have in common

There's the name and title in plain text too, because I'm bad at normalising my
data.

```sql
CREATE TABLE connected
(
  left_id TEXT,
  right_id TEXT,
  name_id TEXT,
  left_title TEXT,
  right_title TEXT,
  name TEXT,
  CONSTRAINT unique_connection UNIQUE (left_id, right_id)
);

CREATE INDEX connected__tconst on connected(left_id, right_id);

CREATE INDEX connected__left_title on connected(left_title);

CREATE INDEX connected__right_title on connected(right_title);
```

I've also kind of haphazardly put indexes on just whatever I want to search on. I think there's
a good chance I should have put separate indexes on `left_id` and `right_id`.

Next I wrote a query, joining `movie_principals` onto itself and grouping by
the two title ids, to keep the uniqueness constraint, inserting the results of
this query into my new table.

```sql
INSERT INTO connected
SELECT
    l.tconst as left_id,
    r.tconst as right_id,
    l.nconst as name_id, -- should be the same as r.tconst
    l.primaryTitle as left_title,  -- this isn't very normalised
    r.primaryTitle as right_title, -- but I'm lazy and I don't know what I'm doing
    l.primaryName as name
FROM movie_principals l
JOIN movie_principals r ON l.nconst = r.nconst
GROUP BY (l.tconst, r.tconst)
```

So because we're representing the edges of our graph (two movies connected by a person),
we can do fewer joins. We're selecting all the connections that start with `Josie and the Pussycats`
and all the connections _from those_ that end in `Muriel's Wedding`

```sql
SELECT
  a.left_title,
  a.name,
  b.left_title,
  b.name,
  b.right_title
FROM connected a
JOIN connected b ON b.left_id = a.right_id
WHERE a.left_title = 'Josie and the Pussycats'
AND b.right_title = 'Muriel''s Wedding'
```

This has a pretty good looking query plan (I see the words `SEARCH` and `INDEX`
and I don't see the words `SCAN` or `PARTIAL`)

```
QUERY PLAN
|--SEARCH TABLE connected AS start USING INDEX connected__left_title (left_title=?)
`--SEARCH TABLE connected AS end USING INDEX connected__tconst (left_id=?)
```

This runs _fast_ and even better, it gives us a correct result:

```
Josie and the Pussycats|Rachael Leigh Cook|Blow Dry|Rachel Griffiths|Muriel's Wedding
Josie and the Pussycats|Tara Reid|My Boss's Daughter|Martin McGrath|Muriel's Wedding
```

Or in other words:

1. [Rachel Leigh Cook](https://www.imdb.com/name/nm0000337) was in [Josie and the Pussycats](https://www.imdb.com/title/tt0236348)
2. Rachel Leigh Cook was also in [Blow Dry](https://www.imdb.com/title/tt0212380) with [Rachel Griffiths](https://www.imdb.com/name/nm0341737)
3. Rachel Griffiths was also in [Muriel's Wedding](https://www.imdb.com/title/tt0110598)

## Conclusions?

I think some of our students don't really have a sense of what is big data and
what is small data, talking about "the principals of all the titles on imdb" you
might think that's "big", but it turns out that it's a tsv file you can download
for free that's less than a gig.

I also think it's really easy to underestimate SQLite. I did all of this with
only two tools, `gzip` and `sqlite3`, and when I got the schema right the
queries were **fast**.

I really thought that "graph search" was a bad fit for a SQL engine, but at least
when we're searching only a few steps **after I got the schema right**, it was really
great at it.

Go play [Film in the Gaps](https://filminthega.ps).

Also, apparently in 2001 Rachel Leigh Cook was in [a movie about a hairdressing competition](https://www.imdb.com/title/tt0212380)
