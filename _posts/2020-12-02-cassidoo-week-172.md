---
layout: post
title:  "Cassidoo Week #172"
---

Here is the interview question of cassidoo newsletter #172:

Given an array of integers and a target value, return the number of pairs of array elements that have a difference equal to a target value.

Examples:

```
$ arrayDiff([1, 2, 3, 4], 1)
$ 3 // 2 - 1 = 1, 3 - 2 = 1, and 4 - 3 = 1
```

And here is my solution in JavaScript:

```js
function arrayDiff(array, diff) {
  const size = array.length;
  if (size <= 1) {
    return 0;
  }

  let nb = 0;

  const mid = size / 2;
  for (let i = 0; i <= mid; ++i) {
    const v1 = array[i];
    for (let j = i + 1; j < size; ++j) {
      const v2 = array[j];

      const d1 = v1 - v2;
      if (d1 === diff) {
        nb++;
      }

      const d2 = v2 - v1;
      if (d2 === diff) {
        nb++;
      }
    }
  }

  return nb;
}
```

Feel free to ping me on twitter ([@mickaeljeanroy](https://twitter.com/mickaeljeanroy)) with a better solution!
