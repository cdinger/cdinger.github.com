---
layout: post
title: "In search of better PeopleSoft effective date queries"
lead: "A join-based approach for speeding up PeopleSoft effective date queries and un-burdening your brain with WHERE clauses that are a mile long."
---

If you work anywhere near a PeopleSoft product you know it's common practice to write effective date queries like so:

{% highlight SQL %}
SELECT *
FROM ps_acad_prog_tbl t
WHERE t.effdt=(
    SELECT MAX(effdt)
    FROM ps_acad_prog_tbl
    WHERE institution=t.institution
      AND acad_prog=t.acad_prog
      AND effdt <= sysdate
  )
{% endhighlight %}

That's fine. PeopleSoft actually generates queries like this. Oracle's query analyzer will show that this has a cost of 516.

If we re-work that query to *join* a relation that contains the `effdt` we want (instead of the [correlated subquery](http://en.wikipedia.org/wiki/Correlated_subquery) to find it for each row),
it brings the cost down to 12—an order of magnitude lower.

{% highlight SQL %}
SELECT ps_acad_prog_tbl.*
FROM ps_acad_prog_tbl
  JOIN (
    SELECT institution, acad_prog, max(effdt) as effdt
    FROM ps_acad_prog_tbl
    WHERE effdt <= sysdate
    GROUP BY institution, acad_prog
  ) ps_acad_prog_tbl_eff_keys
    ON ps_acad_prog_tbl.institution = ps_acad_prog_tbl_eff_keys.institution
      AND ps_acad_prog_tbl.acad_prog = ps_acad_prog_tbl_eff_keys.acad_prog
      AND ps_acad_prog_tbl.effdt = ps_acad_prog_tbl_eff_keys.effdt
{% endhighlight %}

Big deal, right? This is working on a dinky table, who cares?

## It gets worse

It starts to matter when you're dealing with millions of records. In those cases, the effects are magnified. Consider
this query against `ps_acad_prog` (6.9 million rows for this sample institution):

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

Pretty standard stuff. This indicates a execution cost of 19,254,013. It's a big table after all.

Using our same method from above, the same results can be returned much faster:

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

This query costs only 38,171. That's not just much lower. Thats three orders of magnitude lower.
