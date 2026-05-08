
```dataview
TABLE confidence-level,visit-count, last-reviewed, tags
FROM "Computer Science"
WHERE scheda = "done" AND confidence-level >= 1 AND confidence-level < 2.5 AND visit-count > 5 OR contains(tags, "learned") AND confidence-level >= 1 AND confidence-level < 2.5 AND visit-count > 3
SORT last-reviewed ASC
```

