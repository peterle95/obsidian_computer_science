## ✌️ Reviewed Today
```dataview
TABLE language, tags
FROM "Computer Science"
WHERE contains(tags, "learning") OR contains(tags, "learned") OR contains(tags, "to_learn") OR contains(tags, "will_learn") OR contains(tags, "mastered") AND last-reviewed = date(today)
SORT language ASC
```


## ✌️ Reviewed Yesterday
```dataview
TABLE language, tags
FROM "Computer Science"
WHERE contains(tags, "learning") OR contains(tags, "learned") OR contains(tags, "to_learn") OR contains(tags, "will_learn") AND last-reviewed = date(yesterday)
SORT language ASC
```
