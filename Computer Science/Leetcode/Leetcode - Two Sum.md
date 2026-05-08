---
memory: to_finish
tags:
  - will_learn
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
Given an array of integers `nums` and an integer `target`, return _indices of the two numbers such that they add up to `target`_.

You may assume that each input would have **_exactly_ one solution**, and you may not use the _same_ element twice.

You can return the answer in any order.

**Example 1:**

**Input:** nums = [2,7,11,15], target = 9
**Output:** [0,1]
**Explanation:** Because nums[0] + nums[1] == 9, we return [0, 1].

**Example 2:**

**Input:** nums = [3,2,4], target = 6
**Output:** [1,2]

**Example 3:**

**Input:** nums = [3,3], target = 6
**Output:** [0,1]

**Constraints:**

- `2 <= nums.length <= 104`
- `-109 <= nums[i] <= 109`
- `-109 <= target <= 109`
- **Only one valid answer exists.**

**Follow-up:** Can you come up with an algorithm that is less than `O(n2)` time complexity?

# **Starting code:**
---

```js
var twoSum = function(nums, target) {

};
```


# **Solution:**
---
```js
var twoSum = function(nums, target) {
    for(let i = 0; i < nums.length; i++) {
        for(let j = i + 1; j < nums.length; j++) {
            if(nums[i] + nums[j] === target) {
                return [i, j];
            }
        }
    }
};
```

# **Explanation:**
---


The Two Sum problem asks us to find two numbers in an array that add up to a specific target value, then return their indices.

### Algorithm Approach: Nested Loop (Brute Force)

**Time Complexity:** O(n²)  
**Space Complexity:** O(1)

---

### Step-by-Step Breakdown

#### **Step 1: Function Setup**

```javascript
var twoSum = function(nums, target) {
```

- Define a function that takes two parameters:
    - `nums`: an array of integers
    - `target`: the sum we're looking for

#### **Step 2: Outer Loop - First Number**

```javascript
for(let i = 0; i < nums.length; i++) {
```

- Start at index `0` and iterate through each element
- `i` represents the index of the **first number** in our potential pair
- Loop continues until we've checked every element

#### **Step 3: Inner Loop - Second Number**

```javascript
for(let j = i + 1; j < nums.length; j++) {
```

- For each `i`, start `j` at `i + 1` (the next element)
- This ensures we:
    - Don't use the same element twice
    - Don't check pairs we've already examined
    - Check all remaining combinations

#### **Step 4: Check if Pair Sums to Target**

```javascript
if(nums[i] + nums[j] === target) {
```

- Add the values at indices `i` and `j`
- Compare the sum to our target value
- Use strict equality (`\===`) to check for exact match

#### **Step 5: Return the Indices**

```javascript
return [i, j];
```

- If we find a matching pair, immediately return their indices as an array
- The problem guarantees exactly one solution exists, so we can return immediately


---

### Example Walkthrough

**Input:** `nums = [2, 7, 11, 15]`, `target = 9`

|Iteration|i|j|nums[i]|nums[j]|Sum|Match?|
|---|---|---|---|---|---|---|
|1|0|1|2|7|9|✓ **Return [0, 1]**|

**Input:** `nums = [3, 2, 4]`, `target = 6`

|Iteration|i|j|nums[i]|nums[j]|Sum|Match?|
|---|---|---|---|---|---|---|
|1|0|1|3|2|5|✗|
|2|0|2|3|4|7|✗|
|3|1|2|2|4|6|✓ **Return [1, 2]**|

---

### Key Points

1. **Why start j at i + 1?**  
    To avoid using the same element twice and prevent duplicate checking
    
2. **Why use nested loops?**  
    We need to check every possible pair combination
    
3. **When does it stop?**  
    As soon as a valid pair is found, we return immediately
    
4. **Trade-offs:**
    
    - ✓ Simple and easy to understand
    - ✓ No extra space needed
    - ✗ Slower for large arrays (O(n²) time)