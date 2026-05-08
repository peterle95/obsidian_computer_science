
```dataview
TABLE last-reviewed, language
FROM "Computer Science"
WHERE contains(tags, "learned") AND last-reviewed 
SORT last-reviewed ASC
```
