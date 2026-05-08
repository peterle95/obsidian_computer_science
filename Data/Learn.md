
## ‼️ Topics Learning
```dataview
TABLE language as "Language", last-reviewed as "Last Reviewed", tags as "Tags"
FROM "Computer Science"
WHERE contains(tags, "learning")
SORT last-reviewed ASC
```


## ⚠️ Important

```dataview
TABLE language as "Language", last-reviewed as "Last Reviewed", cssclasses as "Class"
FROM "Computer Science"
WHERE contains(cssclasses, "important") OR contains(cssclasses, "difficult")
SORT last-reviewed ASC
```

## 🔮 Topics to Learn Future
```dataview
TABLE language as "Language", last-reviewed as "Last Reviewed", tags as "Tags"
FROM "Computer Science"
WHERE contains(tags, "will_lear") AND scheda = "done"
SORT language ASC
```
## 🥸 Topics to Learn
```dataview
TABLE language as "Language", last-reviewed as "Last Reviewed", tags as "Tags"
FROM "Computer Science"
WHERE contains(tags, "to_learn") AND scheda = "done"
SORT last-reviewed ASC
```
