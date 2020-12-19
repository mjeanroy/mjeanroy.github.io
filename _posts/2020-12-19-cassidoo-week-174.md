---
layout: post
title:  "Cassidoo Week #174"
---

Here is the interview question of [cassidoo](https://cassidoo.co/) newsletter #174:

Given a list of folders in a filesystem and the name of a folder to remove, return the new list of folders after removal.

Examples:

```
$ removeFolder(['/a', '/a/b', '/c/d', '/c/d/e', '/c/f', '/c/f/g'], 'c')
$ ['/a', '/a/b']
$ removeFolder(['/a', '/a/b', '/c/d', '/c/d/e', '/c/f', '/c/f/g'], 'd')
$ ['/a', '/a/b', '/c', '/c/f', '/c/f/g']
```

This one is easier than the question of the previous week and the general idea is this one:

- For each folder:
  - Truncate folder: keep only the part before the folder being removed
  - Returns that result
- Store each result in a set, so I don't have to worry about duplications.

Note that in my algorithm below, I suppose that the second parameter is always a "single" folder, for exemple `c` but not `c/d`.

In a "real" interview, that is probably a question to ask: the global idea remains, but the implementation would be a little different!

Here is my solution in JS:

```js
function removeFolder(folders, toRemove) {
  const results = new Set();

  for (let i = 0; i < folders.length; ++i) {
    const truncatedFolder = truncate(folders[i], toRemove);
    if (truncatedFolder) {
      results.add(truncatedFolder);
    }
  }

  return Array.from(results);
}

function truncate(folder, toRemove) {
  const parts = folder.split('/');
  const results = [];
  for (let i = 0; i < parts.length; ++i) {
    const part = parts[i];
    if (part === toRemove) {
      // This folder is deleted, we can stop here
      // since everything after will be deleted
      break;
    }

    results.push(part);
  }

  return results.join('/');
}
```
