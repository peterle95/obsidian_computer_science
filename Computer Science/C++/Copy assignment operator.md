---
memory: to_finish
tags:
  - mastered
language:
  - C++
review-date: ""
last-reviewed: 2025-06-10
scheda: done
---
# **Core Explanation:**

* Purpose: The copy assignment operator (operator=) handles assigning one object's value to another *existing* object
* Unlike the copy [[Copy constructor]] which creates a new object, assignment happens between existing objects

Here's a basic example:

```cpp
class MyClass {
private:
    int* data;

public:
    // Constructor
    MyClass(int value) 
    {
        data = new int(value);
    }

    // Copy assignment operator
    MyClass& operator=(const MyClass& other) 
    {
        // Guard against self-assignment
        if (this != &other) 
        {
            // 1. Clean up existing resources
            delete data;
            
            // 2. Perform deep copy
            data = new int(*other.data);
        }
        // 3. Return *this for chaining
        return *this;
    }

    // Destructor
    ~MyClass() 
    {
        delete data;
    }
};

// Usage:
MyClass obj1(5);
MyClass obj2(10);
obj2 = obj1;  // Copy assignment operator called
```

Key Points:
* Returns a reference to the current object (MyClass&)
* Takes a const reference parameter
* Must handle self-assignment (when an object is assigned to itself)
* Should clean up existing resources before copying
* Should perform deep copy for pointers

Common Mistakes:
* Forgetting to return *this
* Not handling self-assignment
* Memory leaks from not cleaning up old resources
* Shallow copying when dealing with pointers
