---
layout: post
title: "In search of better PeopleSoft effective date queries, Part 2"
lead: "A continued exploration of alternative techniques for making PeopleSoft effective date queries more readable and maintainable by composing subrelations and destructuring with the WITH clause."
---

In [my previous post](/2014/10/faster-peoplesoft-effective-date-queries/), I made the argument that
this:

{% highlight SQL %}
SELECT ps_acad_prog.*
FROM ps_acad_prog
  JOIN (
    SELECT ps_acad_prog.emplid, ps_acad_prog.acad_career, ps_acad_prog.stdnt_car_nbr, ps_acad_prog.effdt, max(ps_acad_prog.effseq) as effseq
    FROM ps_acad_prog
      JOIN (
        SELECT emplid, acad_career, stdnt_car_nbr, max(effdt) as effdt
        FROM ps_acad_prog
        WHERE effdt <= sysdate
        GROUP BY emplid, acad_career, stdnt_car_nbr
      ) max_effdt_ps_acad_prog
        ON ps_acad_prog.emplid = max_effdt_ps_acad_prog.emplid
          AND ps_acad_prog.acad_career = max_effdt_ps_acad_prog.acad_career
          AND ps_acad_prog.stdnt_car_nbr = max_effdt_ps_acad_prog.stdnt_car_nbr
          AND ps_acad_prog.effdt = max_effdt_ps_acad_prog.effdt
    GROUP BY ps_acad_prog.emplid, ps_acad_prog.acad_career, ps_acad_prog.stdnt_car_nbr, ps_acad_prog.effdt
  ) effdt_and_effseq_acad_prog
    ON ps_acad_prog.emplid = effdt_and_effseq_acad_prog.emplid
      AND ps_acad_prog.acad_career = effdt_and_effseq_acad_prog.acad_career
      AND ps_acad_prog.stdnt_car_nbr = effdt_and_effseq_acad_prog.stdnt_car_nbr
      AND ps_acad_prog.effdt = effdt_and_effseq_acad_prog.effdt
      AND ps_acad_prog.effseq = effdt_and_effseq_acad_prog.effseq
{% endhighlight %}

is better than this:

{% highlight SQL %}
SELECT *
FROM ps_acad_prog p
WHERE effdt = (
    SELECT max(effdt)
    FROM ps_acad_prog
    WHERE emplid = p.emplid
      AND acad_career = p.acad_career
      AND stdnt_car_nbr = p.stdnt_car_nbr
      AND effdt <= sysdate
  )
  AND effseq = (
    SELECT max(effseq)
    FROM ps_acad_prog
    WHERE emplid = p.emplid
      AND acad_career = p.acad_career
      AND stdnt_car_nbr = p.stdnt_car_nbr
      AND effdt = p.effdt
  )
{% endhighlight %}

But that first query is *way* bigger. Other than a speed boost, it may not clear why I think
the first query is better, so I thought I'd dig into it a bit more. I'd like to
explain more about why I think this is better (not just *faster*) and offer up
some alternative notation.

### Readability/Reasonability

I think the biggest reason to avoid correlated subqueries is not the performance
benefit, it's the cognitive load that it forces upon us. A big, gnarly query
with many correlated subqueries makes it difficult to reason about.

When you encounter a big query that's doing more than you can take in at once,
what's the first thing you do? You break it down. Start executing pieces of the
query to get an idea of the relations that are at play in your mind. Correlated
subqueries make that difficult because you cannot select a piece of the query and
run it individually. Especially in the case of PeopleSoft, these subqueries are
wired to other relations outside of the immediate scope.

Effective date subqueries also often get thrown into the `WHERE` clause
alongside conditions that represent real business logic, clouding the intention
of the query. The goal should be to shove all of this accidental complexity
aside, leaving the minimal amount of code to represent the core of what the query
is trying to accomplish.

The following query speaks for itself, doesn't it? This should be our goal.
Any effective dated nonsense that this requires should take a back seat to this
distilled statement.

{% highlight SQL %}
SELECT * FROM effective_acad_prog_tbl WHERE acad_prog='123ASDF'
{% endhighlight %}

### Destructuring with `WITH`

Since we're specifically talking PeopleSoft/Oracle here, we have the often
neglected `WITH` clause at our disposal. We can use `WITH` to destructure the
ugly peripheral bits into meaningful relation names.

In Clojure and other lisps, destructuring is done with the `let` form:

{% highlight clojure %}
(let [acad-prog-eff-keys (...)
      effective-acad-prog (...)]
  (do-something effective-acad-prog))
{% endhighlight %}

Oracle's `WITH` clause is very similar:

{% highlight SQL %}
WITH acad_prog_tbl_eff_keys AS (
  SELECT institution, acad_prog, max(effdt) as effdt
  FROM ps_acad_prog_tbl
  WHERE effdt <= sysdate
  GROUP BY institution, acad_prog
),
effective_acad_prog_tbl AS (
  SELECT ps_acad_prog_tbl.*
  FROM ps_acad_prog_tbl
    JOIN acad_prog_tbl_eff_keys
      ON ps_acad_prog_tbl.institution = ps_acad_prog_tbl_eff_keys.institution
        AND ps_acad_prog_tbl.acad_prog = ps_acad_prog_tbl_eff_keys.acad_prog
        AND ps_acad_prog_tbl.effdt = ps_acad_prog_tbl_eff_keys.effdt
)
SELECT * FROM effective_acad_prog_tbl WHERE acad_prog = '123ASDF'
{% endhighlight %}

`WITH` lets us nicely break up (and push aside) the non-core queries and clauses,
leaving the business logic to speak for itself.

### Composability

So my point here is really composability. Regardless of whether you're using
`WITH` clauses, building queries by composing discrete meaningfully-named
relations will help build clearer, faster, and more maintainable queries.
