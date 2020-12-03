---
layout: post
title:  "Cassidoo Week #172"
---

Here is the interview question of [cassidoo](https://cassidoo.co/) newsletter #172:

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
  
  const map = new Map();
  for (let i = 0; i < size; ++i) {
    const value = array[i];
    if (!map.has(value)) {
      map.set(value, 0);
    }

    map.set(value, map.get(value) + 1);
  }


  let nb = 0;

  for (let i = 0; i < size; ++i) {
    const value = array[i];
    const lookingFor = diff + value;
    if (map.has(lookingFor)) {
      nb += map.get(lookingFor);
    }
  }

  return nb;
}
```

This solution has a complexity of O(n), which is great!
Note that this solution takes care of duplications in given array, if the array contains only unique elements, we can make it simpler using only a set.

A naïve solution could be;

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

This solution has a complexity of O(n^2), which is not very good, but much easier to implement.

Feel free to ping me on twitter ([@mickaeljeanroy](https://twitter.com/mickaeljeanroy)) with a better solution!
