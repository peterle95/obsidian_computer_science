---
memory: to_finish
tags:
  - to_learn
language:
  - Leetcode
review-date:
last-reviewed: ""
scheda: to_finish
visit-count: 0
confidence-level: 1
consecutive-correct: 0
last-struggle-date: ""
cssclasses:
palace:
palace-room:
locus:
palace-order:
---

```dataviewjs
const currentPage = dv.current();
let visitCount = currentPage.file.frontmatter["visit-count"] || 0;
let confidence = currentPage.file.frontmatter["confidence-level"] || 1;
let streak = currentPage.file.frontmatter["consecutive-correct"] || 0;

const container = this.container.createEl('div');
container.style.cssText = `
    background: #2a2a2a; border: 1px solid #404040; border-radius: 6px;
    padding: 12px; margin: 10px 0; display: inline-block;
`;

// Status display
const status = container.createEl('div');
status.innerHTML = `
    <strong>Learning Progress</strong><br>
    Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
`;
status.style.cssText = 'margin-bottom: 10px; font-size: 13px; color: #cccccc;';

// Quick feedback buttons
const buttonContainer = container.createEl('div');
['Got it! ✅', 'Struggled ⚠️', 'Failed ❌'].forEach((label, index) => {
    const btn = buttonContainer.createEl('button');
    btn.textContent = label;
    btn.style.cssText = `
        margin-right: 8px; padding: 4px 8px; border: none; border-radius: 3px;
        cursor: pointer; font-size: 11px;
        background: ${['#28a745', '#ffc107', '#dc3545'][index]};
        color: ${index === 1 ? '#000' : '#fff'};
    `;
    
    btn.addEventListener('click', async () => {
        visitCount++;
        if (index === 0) { // Got it
            confidence = Math.min(5, confidence + 0.5);
            streak++;
        } else { // Struggled or failed
            confidence = Math.max(1, confidence - 0.5);
            streak = 0;
        }
        
        const file = app.vault.getAbstractFileByPath(currentPage.file.path);
        await app.fileManager.processFrontMatter(file, (fm) => {
            fm["visit-count"] = visitCount;
            fm["confidence-level"] = confidence;
            fm["consecutive-correct"] = streak;
            fm["last-reviewed"] = new Date().toISOString().split('T')[0];
            if (index > 0) fm["last-struggle-date"] = new Date().toISOString().split('T')[0];
        });
        
        status.innerHTML = `
            <strong>Learning Progress</strong><br>
            Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
        `;
    });
});
```

# **Exercise**:
---
Given an integer `x`, return `true` _if_ `x` _is a_ _**palindrome**__, and_ `false` _otherwise_.

**Example 1:**

**Input:** x = 121
**Output:** true
**Explanation:** 121 reads as 121 from left to right and from right to left.

**Example 2:**

**Input:** x = -121
**Output:** false
**Explanation:** From left to right, it reads -121. From right to left, it becomes 121-. Therefore it is not a palindrome.

**Example 3:**

**Input:** x = 10
**Output:** false
**Explanation:** Reads 01 from right to left. Therefore it is not a palindrome.

**Constraints:**

- `-231 <= x <= 231 - 1`

**Follow up:** Could you solve it without converting the integer to a string?
# **Starting code:**
---

```js
/**
 * @param {number} x
 * @return {boolean}
 */
var isPalindrome = function(x) {
  
};
```


# **Solution:**
---
```js
/**
 * @param {number} x
 * @return {boolean}
 */
var isPalindrome = function(x) {
    let s = x.toString();
    let j = s.length - 1;
    for(let i = 0; i < s.length/2; i++)
    {
            if(s[i] === s[j])
            {
                j--;
                continue;
            }
            else
                return false

    }
    return true;
};
```

# **Explanation:**
---
### **Approach: Convert to String and Compare From Both Ends**

A palindrome reads the same forward and backward.  
To check this, we convert the number into a string, then compare characters from the beginning and the end moving toward the center.

If every matching pair is equal, the number is a palindrome. If any pair differs, it is not.

### **Step 1: Convert the Integer to a String**

We first turn the integer into a string:
```js
let s = x.toString();
```
This makes it easy to access each digit by index.

### **Step 2: Set a Pointer at the End**

We use a variable `j` to track the last character in the string:
```js
let j = s.length - 1;
```
### **Step 3: Compare Digits From Both Sides**

We loop from the start of the string up to the middle:
```js
for (let i = 0; i < s.length / 2; i++)
```
For each position:

- compare `s[i]` with `s[j]`
    
- if they are not equal, return `false`
    
- if they match, move `j` one step left
    

### **Step 4: Return True if All Pairs Match**

If the loop finishes without finding a mismatch, then the number reads the same forward and backward, so we return `true`.

---

### **Example Walkthrough**

**Input:** `x = 121`

- Convert to string: `"121"`
- Compare first and last characters:
    
    - `'1' === '1'` ✓
        
- Middle character does not need a pair
- No mismatches found
    
**Output:** `true`

---
### **Another Example**

**Input:** `x = -121`

- Convert to string: `"-121"`
- Compare first and last characters:
    
    - `'-' !== '1'` ✗
        
**Output:** `false`

---
### **Time & Space Complexity**

- **Time:** `O(n)` — we compare at most half of the digits
    
- **Space:** `O(n)` — converting the integer to a string uses extra space