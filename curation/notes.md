# Transaction-time joins

Easy to time-slice two tables to a current-time table, and do a join.

What we're wanting to do is a sequenced transaction-time join. This means we
take two TT records and need to produce a new TT record based on the period of
overlap between the two records.

How would we write this?

Basic:

```
query {
  for (d <-- Tables.disease)
    where (d.disease_id == diseaseID)
    [ (disease = d,
       synonyms =
         for (s <-- Tables.disease2synonym)
         where (s.disease_id == diseaseID)
         [
          (dbLinkIDs =
            for (synLink <-- Tables.disease_synonym2database_link)
            where (synLink.disease2synonym_id == s.disease2synonym_id)
            [synLink.disease_database_link_id],
           synonym = s)
         ],
      dbLinks =
        for (ddl <-- Tables.disease_database_link)
        where (ddl.disease_id == diseaseID)
        [ ddl ]
      )
    ]
}
```


I guess in this case we have one-to-many relationships, hence the shredding. I'm
not sure this is an ideal example.
  - I think this is a further research project, actually -- the book examples
      are all 1NF. I can't easily see even what the *result* of a temporal
      shredded query would look like...

  - The best thing to do for now would probably be to do a sequenced selection
      on the disease ID, and then time-slice the one-to-many relations.


OK -- simpler example. Employee is associated with a salary band. Salary bands
are updated yearly. Both are transaction-time tables. Do a sequenced selection
of employees and salaries over time.

Basic query:

```
for (e <-- employees)
  for (s <-- salaries)
  where (e.salary_band == p.salary_band)
  [(e.name, s.salary)]
```

Sequenced query ("When was it recorded for each employee that their data or
their salary changed?")

This following one is a little weird, as `e` and `s` are floating about
unconstrained, so it's easy to use them erroneously and select the entire table.

```
for (e <-t- employees)
  for (s <-t- salaries)
  tt_join ((e, s) where e.salary_band == p.salary_band as es)
  [(ttData(es).name, ttData(es).salary, ttFrom(es))]
```

Can we write it without an explicit join?

```
for (e <-t- employees)
  for (s <-t- salaries)
  where (ttData(e).salary_band == ttData(p).salary_band && overlaps(e, s))
  [ withTTMetadata((name = ttData(e).name, salary = ttData(s).salary), e, s) ]
```

A bit weird. It's perfectly possible here to forget the `overlaps` clause and
then have `withTTMetadata` be undefined...

Perhaps we could do the following:

```
join {
  for (e <-t- employees)
    for (s <-t- salaries)
    where (e.salary_band == p.salary_band)
    [ (name = ttData(e).name, salary = ttData(s).salary) ]
}
```

here, we modify the rules so that we don't have explicit access to the temporal
metadata until the end (where it's computed from the different joins).

I have no idea how to formalise or implement this yet though.


---

Instead of having the classical "iteration-style", we can have an explicit
joining operator:

The following is a theta-join, which constructs a map between the
joined table names and their contents.

```
for (es <-t- tt_join(employees, salaries))
  where (ttProject(es, employees).salary_band == ttProject(es, salaries).salary_band)
  [(ttProject(es, employees).name, ttProject(es, salaries).salary_band)]
```

We can specialise from theta-join to equijoin -- this assumes that `salary_band`
is the only shared column:

```
for (es <-t- tt_join((employees, salaries) on salary_band)
  [(ttData(es).name, ttData(es).salary, ttFrom(es))]
```


