---
title: "Trie"
date: 2023-01-22T16:23:50-08:00
draft: false
---

# Tries, the Word Search Data Structure!

A bunch of years ago I learned about Tries. I thought they were pretty nifty! Today I decided to write about them.

### What's a trie?

In essence, a Trie is a data structure comprized of a bunch of nested Maps/Dictionaries. So, if you enter "alex" into a Trie, your internal structure looks like this:

```json
{ "a": {
    "l": {
        "e": {
            "x": {}
        }
    }
  }
}
```

To better demonstrate, if we added "alan" to that same Trie, the underlying data structure would look like:

```json
{
    "a": {
        "l": {
            "e": {
                "x": {}
            },
            "a": {
                "n": {}
            }
        }
    }
}
```

The power in this is that, both search and insert are O(n) time complexity. Very performant!

### But how?

I whipped up a quick Trie in Python to give a better representation of how it works. The code is:

```python
"""
Build a Trie
"""

from typing import Dict

import pytest


class Trie:
    """A class implementing a Trie"""
    store: Dict

    def __init__(self) -> None:
        self.store = {}

    def insert(self, word: str) -> None:
        """Insert a word into the trie"""
        level = self.store
        for char in word:
            if char not in level:
                level[char] = {}

            level = level[char]

    def search(self, word: str) -> bool:
        """Search the trie for a given word"""
        level = self.store
        for char in word:
            if char in level:
                level = level[char]
            else:
                return False

        return True


@pytest.mark.parametrize("word", ["alex", "aaaa", "", "trie"])
def test_trie(word: str):
    t = Trie()
    t.insert(word)
    assert t.search(word)

@pytest.mark.parametrize("word", ["banks"])
def test_not_found(word: str):
    j = Trie()
    assert j.search(word) == False
```

### Conclusion

That's pretty much it! Enjoy!