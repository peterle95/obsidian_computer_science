## 🔄 Due for Review
```dataview
TABLE review-date, language
FROM "Computer Science"
WHERE contains(tags, "learning") OR contains(tags, "learned") OR contains(tags, "to_learn") AND review-date AND review-date <= date(today)
SORT review-date ASC
```
## 🧠 Review Date
```dataview
TABLE review-date, language, tags
FROM "Computer Science"
WHERE contains(tags, "learned") OR contains(tags, "to_learn") OR contains(tags, "learning") AND review-date 
SORT review-date ASC
```
