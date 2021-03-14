---
layout: post
title:  "Cassidoo Week #176"
tags: ['cassidoo', 'algorithm']
---

Here is the interview question of [cassidoo](https://cassidoo.co/) newsletter #176:

You’re given a string of characters that are only 2s and 0s. Return the index of the first occurrence of “2020” without using the indexOf (or similar) function, and -1 if it’s not found in the string.

Example:

```
$ find2020(‘2220000202220020200’)
$ 14
```

For this question, I will first implement a very naïve solution which is easy to understand and to read:

```js
const SUBSTRING = '2020';

function find2020(str) {
  if (!str || str.length < SUBSTRING.length) {
    return -1;
  }

  let i = 0;
  let j = 0;

  for (; i < str.length; ++i) {
    const c = str[i];
    if (c === SUBSTRING[j]) {
      j++;
    } else if (j > 0) {
      i -= j;
      j = 0;
    }

    if (j === SUBSTRING.length) {
      return i - (j - 1);
    }
  }

  return -1;
}
```

In this solution, I iterate over the input string and I compare characters by characters until I find an exact match. The trick here is if I find a mismatch, I have to go back in the input string to do the same comparisons again (and in this question, with a string containing only 2 or 0, this situation may happens a lot...).

The advantage of this solution is that it is easy to understand and really easy to implement.
The drawbacks is that it is really inefficient.
Searching a pattern efficiently in a string is a well known algorithm: the **KMP algorithm**.

I will not explain this algorithm in details, you can find a very good one here: [https://www.geeksforgeeks.org/kmp-algorithm-for-pattern-searching/](https://www.geeksforgeeks.org/kmp-algorithm-for-pattern-searching/).

So, here is my implementation of the KMP algorithm:

```js
function lps(subString) {
  const output = [];
  const n = subString.length;

  let i = 0;
  let j = -1;

  while (i !== n) {
    const c1 = subString[i];
    const c2 = j >= 0 ? subString[j] : '';

    if (c1 === c2) {
      output[i] = j + 1;
      j++;
      i++;
    } else if (j > 0) {
      j = output[j - 1];
    } else {
      output[i] = 0;
      i++;
      j = 0;
    }
  }

  return output;
}

function kmpSearch(input, subString, lpsTable) {
  let i = 0;
  let m = 0;

  const n = subString.length;
  const l = input.length;

  while (true) {
    if ((m + i) === l) {
      return -1;
    }

    const c1 = subString[i];
    const c2 = input[m + i];
    if (c1 === c2) {
      i = i + 1;
      if (i === n) {
        return m;
      }
    }

    else {
      const e = i === 0 ? -1 : lpsTable[i - 1];
      m = m + i - e;
      if (i > 0) {
        i = e;
      }
    }
  }
}

function find2020KMP(input) {
  if (!input || input.length < SUBSTRING.length) {
    return -1;
  }

  const lpsTable = lps(SUBSTRING);
  return kmpSearch(input, SUBSTRING, lpsTable);
}
```

It is far more complicated to implement, but it is really faster!
