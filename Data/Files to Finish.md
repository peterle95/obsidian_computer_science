
##  👀 Files to Finish 
```dataview
TABLE scheda, language
FROM "Computer Science"
WHERE scheda = "to_finish"  AND contains(language, "JavaScript")
SORT ASC
```

##  👀 Memory to Finish 
```dataview
TABLE tags, language
FROM "Computer Science/JavaScript"
WHERE memory = "to_finish" 
SORT tags ASC
```
