---
layout: post
title:  "Cassidoo Week #173"
---

Here is the interview question of [cassidoo](https://cassidoo.co/) newsletter #173:

Given two non-negative integers n1 and n2 represented as strings, return the product of n1 and n2, also represented as a string. Neither input will start with 0, and don’t just convert it to an integer and do the math that way.

Examples:

```
$ stringMultiply(“123”, “456”)
$ “56088”
```

This puzzle looks simple but in fact is quite complicated. Let's take the and let's do all the step one by one:

As we saw at school, multiplying two numbers can be done "by hand" like:

```
    123
x   456
---------
  00738
+ 06150
+ 49200
---------
  56088
```

The first step is to iterate over digits of `456` (in reverse order) and multiply each digit with `123`, for example:

```
1-1 6 * 3 = 18 => 8 carry 1
1-2 6 * 2 + 1 = 13 => 3 carry 1
1-3 6 * 1 + 1 = 7, carry 0

2-1 Append one zero
2-2 5 * 3 = 15 => 5 carry 1
2-3 5 * 2 + 1 = 11 => 1 carry 1
2-4 5 * 1 + 1 = 6, carry 0

3-1 Append two zero
3-2 4 * 3 = 12 => 2 carry 1
3-3 4 * 2 + 1 = 9 carry 1
3-4 4 * 1 + 0 = 4, carry 0
```

And now, the idea is to sum all these sub-results one by one:

```
0- carry 0
1- 8 + 0 + 0 + carry = 8 + 0 + 0 + 0 = 8 carry 0
2- 3 + 5 + 0 + carry = 3 + 5 + 0 + 0 = 8 carry 0
3- 7 + 1 + 2 + carry = 7 + 1 + 2 + 0 = 10 => 0 carry 1
4- 0 + 6 + 9 + carry = 0 + 6 + 9 + 1 = 16 => 6 carry 1
5- 0 + 0 + 4 + carry = 0 + 0 + 4 + 1 = 5 carry 0
```

And, here it is, we have our results: `56088`.

And here is my solution in JavaScript:

```js
function stringMultiply(x, y) {
  if (x === '0' || y === '0') {
    return '0';
  }

  return computeSumOfProducts(
    computeProducts(x, y)
  );
}

function computeProducts(x, y) {
  const products = [];

  // For example, suppose x = 123 and y = 45
  //   123
  // x  45
  //
  // We want to evaluate:
  //
  // i == 0
  // 1-1 5 * 3 => 15 => 5, carry = 1
  // 1-2 5 * 2 => 10 + 1 => 11 => 1, carry = 1
  // 1-3 5 * 1 => 5 + 1 => 6
  // 1-4 carry = 0
  // ==> [5, 1, 6]
  //
  // i == 1 => add one zero => 0
  // 2-3 4 * 3 => 12 => 2, carry = 1
  // 2-3 4 * 2 => 8 + 1 => 9
  // 2-3 4 * 1 => 4
  // 2-4 carry = 0
  // ==> [0, 2, 9, 4]

  // Start with each digit of `y` in reverse order.
  for (let i = 0; i < y.length; ++i) {
    const nb1 = Number(y[y.length - i - 1]);
    const result = [];

    // Populate with zero until position of `i`.
    for (let k = 0; k < i; ++k) {
      result.push(0);
    }

    // Do the math and do not forget carry at each step.
    let carry = 0;

    for (let j = x.length - 1; j >= 0; --j) {
      const nb2 = Number(x[j]);

      let product = (nb1 * nb2) + carry;

      if (product >= 10) {
        carry = Math.floor(product / 10);
        product = product % 10;
      } else {
        carry = 0;
      }

      result.push(product);
    }

    if (carry) {
      result.push(carry);
    }

    products.push(result);
  }

  return products;
}

function computeSumOfProducts(products) {
  // For example, suppose previous results when x = 123 and y = 45
  // We have two numbers to sum up:
  // - [5, 1, 6]
  // - [0, 2, 9, 4]
  //
  // We start with the array with the bigger size: it is the last one.
  //
  // i == 0
  // 1- 5 + 0 => 5, carry = 0
  // 2- 1 + 2 => 3, carry = 0
  // 3- 6 + 9 => 15 => 5, carry = 1
  // 4- 4 + undefined + carry => 4 + 0 + 1 = 5, carry = 0
  // 5- carry = 0
  //
  // ==> The final result should be: 5535

  const size = products[products.length - 1].length;

  let carry = 0;
  let product = '';

  for (let i = 0; i < size; ++i) {
    let sumOfDigits = 0;
    for (let j = 0; j < products.length; ++j) {
      sumOfDigits += (products[j][i] || 0);
    }

    let sum = sumOfDigits + carry;
    if (sum >= 10) {
      carry = Math.floor(sum / 10);
      sum = sum % 10;
    } else {
      carry = 0;
    }

    product = sum.toString() + product;
  }

  if (carry) {
    product = carry.toString() + product;
  }

  return product;
}
```

The complexity here is O(m * n) where m is the number of digits of `x` and n is the number of digits of `y`.

I'm curious if we can do better here...

This solution could be optimized by computing and positioning the sum during the first iteration but that does not change the complexity, so... maybe I'll do it for the challenge :)

If someone has a better solution, feel free to ping me on [twitter](https://twitter.com/mickaeljeanroy)!
