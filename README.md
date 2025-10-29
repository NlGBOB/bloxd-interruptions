### Minimizing Interrupts In Bloxd: Core Performance Principles

In the Bloxd environment, script performance is critically tied to an internal "interruption counter." Nearly every function call, conditional check, or complex operation can increase this counter. The goal of efficient code is to minimize these interruptions to ensure your code doesn't cause a `World tick: Interrupted` error. This document outlines key principles, patterns, and preferred methods to achieve that.


### Avoid Template Literals

Template literals are convenient but costly. Each embedded expression (`${...}`) adds one interruption point. Simple string concatenation using the `+` operator is uninterruptible and therefore significantly better.

*   **Inefficient (1 interruption):**
    ```js
    const number = 123;
    const string = `${number} is your number`;
    ```

*   **Efficient (0 interruptions):**
    ```js
    const number = 123;
    const string = number + " is your number";
    ```

### Don't call functions when it's not necessary

Every function call increases the interrupt counter by one. Where possible, use built-in operators and direct assignment, which often have zero interruption cost. (`+`, `=`, `-`, `*`, `/`, `delete`, `^`, bitwise operations and some others are atomic, meaning they don't create interruption points)
Also, if the function you're trying to call is atomic (doesn't contain branching, loops, function calls), then you can just copy the contents of that function and include the code where you normally would've called the function.

#### Efficient Math Operations

Instead of using `Math.min()` or `Math.max()` for **two integers**, you can use bitwise operations which are completely free of interruptions.

**Important:** These bitwise methods only work for integers, not for floating-point numbers (e.g., `1.55`). If your numbers might be floats, you must use `Math.min()` and `Math.max()`.

*   **Finding the Minimum of two integers (0 interruptions):**
    ```js
    let a = 5;
    let b = 1;
    let smaller = b ^ ((a ^ b) & -(a < b)); // Result: 1
    ```

*   **Finding the Maximum of two integers (0 interruptions):**
    ```js
    let a = 5;
    let b = 1;
    let larger = a ^ ((a ^ b) & -(a < b)); // Result: 5
    ```

For finding the min/max value in a **list of numbers** (including floats), `Math.min()` and `Math.max()` with the spread operator (`...`) is the most efficient method. It costs a fixed 2 interruptions, regardless of the list's size (1 for the spread operator, 1 for the `Math` function call).

*   **Finding Min/Max in a List (2 interruptions):**
    ```js
    const list = [4, 1, 9, -2, 5.5];
    const minVal = Math.min(...list); // -2
    const maxVal = Math.max(...list); // 9
    ```

#### Uninterruptible Array Manipulation

Avoid function calls like `list.push()` when you can achieve the same result with direct assignment.

*   **Appending an Element (0 interruptions):**
    ```js
    let list = [];
    list[list.length] = 5; // Appends 5 to the end
    list[list.length] = 10; // Appends 10 to the end. list is now [5, 10]
    ```

*   **Removing Elements from the End (0 interruptions):**
    ```js
    list.length--; // Removes the last element
    list.length = list.length - 2; // Removes the last two elements
    ```

#### Using Objects as Sets

A `Set` is useful for storing unique values, but every method (`.add()`, `.has()`, `.delete()`) is a function call that costs 1 interruption. You can achieve the same result with 0 interruptions by using a plain object, where object keys are inherently unique.

*   **The Inefficient `Set` Approach:**
    ```js
    const mySet = new Set(); // 1 interruption (at initialization)
    mySet.add(1); // 1 interruption
    mySet.add(2); // 1 interruption

    // This check costs 2 interruptions: 1 for the 'if', 1 for .has()
    if (mySet.has(1)) {
        // ...
    }

    mySet.delete(1); // 1 interruption
    ```

*   **The Efficient "Object-as-Set" Approach:**
    ```js
    const myObject = {}; // 0 interruptions
    myObject[1] = true; // 0 interruptions
    myObject[2] = true; // 0 interruptions

    // This check costs only 1 interruption for the myObject[1] check
    if (myObject[1]) {
        // ...
    }

    delete myObject[1]; // 0 interruptions
    ```
    This same principle applies to general-purpose objects: direct property access (`myObject[key]`), assignment (`myObject[key] = value`), and deletion (`delete myObject[key]`) are all free of interruptions.

### Understanding Branching Costs

Conditional logic is essential but adds to the interruption count. Choosing the right structure is key.

*   **`if / else if / else`**: This is the most efficient and predictable branching structure.
    *   A single `if` statement costs **1 interrupt**.
    *   An `if...else` block costs **1 interrupt** in total, regardless of which branch is taken.
    *   An `if...else if...else` chain costs **1 interrupt for each condition checked**. The first `true` condition stops the evaluation.

*   **`try...catch`**: This structure costs **1 interrupt** in total, similar to `if...else`. It can be more efficient than an `if` check if you need to guard against an error that only happens on the first run.
    *   Consider this common pattern:
        ```js
        // Costs 2 interrupts on every run after the list is created (if list && if list.length)
        if (list && list.length) {
            list[list.length] = 100;
        } else {
            this.list = [100];
        }
        ```
    *   The `try...catch` version is more performant after the first run:
        ```js
        // Costs 1 interrupt on the first run, and also 1 on all subsequent runs
        try {
            list[list.length] = 100; // Throws error if 'list' is undefined
        } catch (e) {
            this.list = [100];
        }
        ```
    *   However, the best solution is to avoid branching entirely by ensuring variables are properly initialized (e.g., `this.list = []`) before the loop begins.

*   **Ternary Operator (`? :`)**: Avoid this. It is less efficient than `if...else`, costing **2 interruptions** every time it is evaluated.

*   **`switch` Statement**: Be cautious with `switch`. Each `case` is evaluated like an `if`, costing **1 interrupt**. If the `default` block is reached, it also costs an additional **1 interrupt**. An `if...else if...else` chain is always better. Don't use switch.

### Conclusion: Iterate Smartly

The principles above help reduce interruptions from individual operations. When it comes to iterating over arrays, the choice of method is even more critical. A standard `for` loop with an `if` statement inside it will cost interruptions for the loop *and* an additional +1 interruption for the `if` check on *every single element*.

Instead, you should leverage the built-in functional array methods. They are highly optimized for their specific tasks.

***

### General-Purpose Iteration Methods

These are the most common methods and constructs for iterating over an array to perform an action for each element.

| Method / Loop Structure | "Interrupts" (Empty List) | "Interrupts" (List of size `n`) | Notes / Pros |
| :--- | :--- | :--- | :--- |
| **`list.forEach((item, index, arr) => { ... })`** | 1 | `1 + n` | **Pro:** The best way to iterate an array in Bloxd because the interruption counter is the lowest it can get. It also cleanly provides access to `(item, index, array)`. <br><br>You can also iterate backwards using the index: <br>`list.forEach((_, i) => { api.log(list[list.length - 1 - i]) });` |
| **`for (const num of list)`** | 3 | `3 + n` | **Pro:** The shortest syntax for when you only need the value of each element. |
| **`for (let i = 0; i < list.length; i++)`** | 1 | `1 + 2n` | **Pro:** Useful for specific cases requiring maximum control. It allows you to `break` out early, iterate backwards natively (`i--`), or skip elements by directly manipulating `i`. |
| **`let i = 0;`<br>`while (i < list.length)`** | 1 | `1 + 2n` | **Pro:** Very flexible. Best used when the loop's continuation condition isn't strictly tied to the array's length or a simple increment (e.g., iterating a linked list). |
| **`for (const [i, num] of list.entries())`** | 4 | `4 + 2n` | **Pro:** A readable way to get both the `index` and `value` simultaneously in a `for...of` structure. |

***

### Functional Data Transformation

These methods iterate to create a new array or value, focusing on immutability (not changing the original array).

| Method | "Interrupts" (Empty List) | "Interrupts" (List of size `n`) | Notes / Pros |
| :--- | :--- | :--- | :--- |
| **`list.reduce((acc, current) => { ... }, initialValue)`** | 1 | `2 + n` | **Pro:** Extremely powerful for "reducing" an array to a single value (e.g., a sum, an object, a string). |
| **`list.map((item, index, arr) => { ... })`** | 3 | `3 + n` | **Pro:** The standard for creating a **new array** by transforming each element of the original. Guarantees the new array has the same length as the original. |
| **`list.filter((item, index, arr) => { ... })`** | 3 | `3 + n` | **Pro:** The standard for creating a **new, smaller array** containing only the elements that pass a specific condition. |

***

### Short-Circuiting Methods (Find & Test)

If your goal is to find an element or test a condition, it is much better to use one of the specialized methods below. Writing your own loop and checking `"if (condition)"` on every iteration is less efficient. Since every `if` increases the interrupt counter by one, but these functional methods do not increase the counter for their internal check, they are significantly more performant for their intended purpose.

| Method | "Interrupts" (Empty List) | "Interrupts" (List of size `n`) | Notes / Pros |
| :--- | :--- | :--- | :--- |
| **`list.find((item, index, arr) => { ... })`** | 1 | `1 + k` (where `k` is the index of the first match) | **Pro:** A way to find the **first element** that satisfies a condition. EDIT: actually a better way is `(if element in list) {list.findIndex...}`. I can't come up with a scenario where you might want to use `find`.
| **`list.findIndex((item, index, arr) => { ... })`** | 1 | `1 + k` (where `k` is the index of the first match) | **Pro:** The most efficient way to find the **index of the first element** that satisfies a condition. |
| **`list.some((item, index, arr) => { ... })`** | 1 | `1 + k` (where `k` is the index of the first `true` result) | **Pro:** The most efficient way to check if **at least one** element in an array meets a condition. |
| **`list.every((item, index, arr) => { ... })`** | 1 | `1 + k` (where `k` is the index of the first `false` result) | **Pro:** The most efficient way to check if **all elements** in an array meet a condition. |

***

### API Calls, Callbacks, and Interruption Safety

It's important to understand the fixed costs associated with interacting with the Bloxd API and its event system.

*   **API Function Calls:** Every API function call, such as `api.getPlayerIds()`, `api.setHealth(myId, 99)`, or `api.log()`, increases the interrupt counter by **1**.

*   **Callback Invocations:** Each time a callback is triggered, it increases the interrupt counter by a flat **10 points**, even if the function assigned to it is empty. If you define a callback in your world code, it will incur this cost every time it's called by the game engine.

Take for example the `onBlockStand` callback. It is called very frequently (reportedly about four times more often than the `tick` callback). In a world with 30 players, this single callback can cause the interrupt counter to rise incredibly fast, making interruptions highly likely.

You have to understand that an interruption happening is inevitable. It is currently impossible to write code in Bloxd that can't be interrupted. However, you can write code that is "interruption-resistant" or, as Sulfrox coined the term, "interruption-safe." This means that even if your code is stopped mid-execution, it is designed to resume its action correctly on the next run.

Thankfully, an interruption-safe framework has been built to allow you to schedule tasks and be sure that they will be completed, even when an interruption happens. You can find it here:
**[https://github.com/NlGBOB/bloxd-scheduler](https://github.com/NlGBOB/bloxd-scheduler)**