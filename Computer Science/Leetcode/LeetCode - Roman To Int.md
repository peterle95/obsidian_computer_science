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
Roman numerals are represented by seven different symbols: `I`, `V`, `X`, `L`, `C`, `D` and `M`.

**Symbol**       **Value**
I             1
V             5
X             10
L             50
C             100
D             500
M             1000

For example, `2` is written as `II` in Roman numeral, just two ones added together. `12` is written as `XII`, which is simply `X + II`. The number `27` is written as `XXVII`, which is `XX + V + II`.

Roman numerals are usually written largest to smallest from left to right. However, the numeral for four is not `IIII`. Instead, the number four is written as `IV`. Because the one is before the five we subtract it making four. The same principle applies to the number nine, which is written as `IX`. There are six instances where subtraction is used:

- `I` can be placed before `V` (5) and `X` (10) to make 4 and 9. 
- `X` can be placed before `L` (50) and `C` (100) to make 40 and 90. 
- `C` can be placed before `D` (500) and `M` (1000) to make 400 and 900.

Given a roman numeral, convert it to an integer.

**Example 1:**

**Input:** s = "III"
**Output:** 3
**Explanation:** III = 3.

**Example 2:**

**Input:** s = "LVIII"
**Output:** 58
**Explanation:** L = 50, V= 5, III = 3.

**Example 3:**

**Input:** s = "MCMXCIV"
**Output:** 1994
**Explanation:** M = 1000, CM = 900, XC = 90 and IV = 4.

**Constraints:**

- `1 <= s.length <= 15`
- `s` contains only the characters `('I', 'V', 'X', 'L', 'C', 'D', 'M')`.
- It is **guaranteed** that `s` is a valid roman numeral in the range `[1, 3999]`.
# **Starting code:**
---

```js
/**
 * @param {string} s
 * @return {number}
 */

var romanToInt = function(s) 
{
    
};
```


# **Solution:**
---
```js
/**
 * @param {string} s
 * @return {number}
 */

var romanToInt = function(s) 
{
    const symbols = {
        'I': 1,
        'V': 5,
        'X': 10,
        'L': 50,
        'C': 100,
        'D': 500,
        'M':1000
    }

    let total = 0;

    for(let i = 0; i < s.length; i++)
    {
        let curr = s[i];
        let next = s[i+1];

        if(symbols[curr] < symbols[next])
            total -= symbols[curr];
        else
            total += symbols[curr];
    }

    return total;
};
```

# **Explanation:**
---
### **Approach: Scan left → right, add or subtract based on the next symbol**

Roman numerals mostly add values from left to right. The only twist is the _subtractive_ cases (like `IV`, `IX`, `XL`, etc.).  
A simple rule handles all of them:

- If the current symbol is **smaller** than the next symbol, it must be subtractive → **subtract** it.
    
- Otherwise → **add** it.
    

This works because valid Roman numerals only use subtraction in the allowed pairs, and those pairs always show up as a smaller value immediately followed by a larger one.

---

### **Step 1: Map symbols to integer values**

Create a lookup table:

- `I → 1`, `V → 5`, `X → 10`, `L → 50`, `C → 100`, `D → 500`, `M → 1000`
    

This lets us convert each character in `O(1)` time.

---

### **Step 2: Iterate through the string**

Loop through each character `s[i]` and compare it with `s[i+1]`:

- Let `curr = s[i]`
    
- Let `next = s[i+1]` (may be `undefined` on the last char)
    

Then:

- If `value(curr) < value(next)` → `total -= value(curr)`
    
- Else → `total += value(curr)`
    

---

### **Step 3: Return the accumulated total**

After the loop finishes, `total` is the integer value of the Roman numeral.

---

### **Step N: Final Check**

On the last character, `next` is `undefined`.  
In JavaScript, `symbols[undefined]` is `undefined`, so the comparison:

`symbols[curr] < symbols[next]` becomes something like `10 < undefined`, which is `false`, so we correctly **add** the last symbol.

---

### **Example Walkthrough**

**Input:** `s = "MCMXCIV"`

Symbols: `M(1000) C(100) M(1000) X(10) C(100) I(1) V(5)`

|i|curr|next|rule|total|
|---|---|---|---|--:|
|0|M(1000)|C(100)|add|1000|
|1|C(100)|M(1000)|subtract|900|
|2|M(1000)|X(10)|add|1900|
|3|X(10)|C(100)|subtract|1890|
|4|C(100)|I(1)|add|1990|
|5|I(1)|V(5)|subtract|1989|
|6|V(5)|—|add|1994|

✅ Output: `1994`

---

### **Time & Space Complexity**

- **Time:** `O(n)` where `n = s.length` (single pass)
    
- **Space:** `O(1)` (constant-sized map + a few variables)