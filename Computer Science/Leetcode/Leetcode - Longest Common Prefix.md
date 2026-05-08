---
memory: to_finish
tags:
  - to_learn
language:
  - Leetcode
review-date:
last-reviewed: ""
scheda: done
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
Write a function to find the longest common prefix string amongst an array of strings.

If there is no common prefix, return an empty string `""`.

**Example 1:**

**Input:** strs = ["flower","flow","flight"]
**Output:** "fl"

**Example 2:**

**Input:** strs = ["dog","racecar","car"]
**Output:** ""
**Explanation:** There is no common prefix among the input strings.

**Constraints:**

- `1 <= strs.length <= 200`
- `0 <= strs[i].length <= 200`
- `strs[i]` consists of only lowercase English letters if it is non-empty.
# **Starting code:**
---

```js
/**
 * @param {string[]} strs
 * @return {string}
 */
var longestCommonPrefix = function(strs) {
    
};
```


# **Solution:**
---
```js
var longestCommonPrefix = function(strs) {
  if (!strs.length) 
    return "";

  let prefix = strs[0];

  for (let i = 1; i < strs.length; i++) 
  {
    while (strs[i].indexOf(prefix) !== 0) 
    {
      prefix = prefix.slice(0, prefix.length - 1);
      if (prefix === "") 
        return "";
    }
  }

  return prefix;
};
```

# **Explanation:**
---

### **Approach: Prefix Shortening**

We start by assuming the first string is the common prefix. Then we compare this prefix with every other string in the array.

If the current string does not start with the prefix, we shorten the prefix by removing its last character. We keep doing this until the current string starts with the prefix, or until the prefix becomes empty.

By the end of the loop, the remaining prefix is the longest common prefix shared by all strings.

### **Step 1: Handle Edge Case**

If the array is empty, return an empty string:

```js
if (!strs.length) 
   return "";
```

This prevents errors and handles invalid or empty input safely.

### **Step 2: Start With the First String**

We use the first string as our initial prefix:

```js
let prefix = strs[0];
```

This works because the longest common prefix cannot be longer than the first string.

### **Step 3: Compare With Each Remaining String**

Loop through the rest of the strings:

```js
for (let i = 1; i < strs.length; i++)
```

For each string, check whether it starts with the current prefix.

### **Step 4: Shrink the Prefix Until It Matches**

If the current string does not begin with `prefix`, shorten `prefix` by one character:

```js
while (strs[i].indexOf(prefix) !== 0)
```

Then:

```js
prefix = prefix.slice(0, prefix.length - 1);
```

This keeps shrinking the prefix until it becomes a valid prefix of the current string.

### **Step 5: Final Check**

If at any point the prefix becomes an empty string, return `""` immediately:

```js
if (prefix === "") return "";
```

That means there is no common prefix among all strings.

### **Example Walkthrough**

**Input:** `["flower", "flow", "flight"]`

- Start with `prefix = "flower"`
    
- Compare with `"flow"`
    
    - `"flow"` does not start with `"flower"`
        
    - Shorten to `"flowe"`
        
    - Shorten to `"flow"`
        
    - `"flow"` starts with `"flow"`
        
- Compare with `"flight"`
    
    - `"flight"` does not start with `"flow"`
        
    - Shorten to `"flo"`
        
    - `"flight"` does not start with `"flo"`
        
    - Shorten to `"fl"`
        
    - `"flight"` starts with `"fl"`
        

**Result:** `"fl"`

### **Time & Space Complexity**

- **Time:** `O(N * M)`
    
    - `N` = number of strings
        
    - `M` = length of the prefix/string in the worst case  
        In the worst case, we may shorten the prefix character by character for each string.
        
- **Space:** `O(1)`
    
    - We only use a few extra variables, so the additional space is constant.