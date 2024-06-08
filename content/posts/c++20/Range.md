+++
title = "Range"
date = "2024-06-08T16:53:38+08:00"
author = "Civitasv"
authorTwitter = "" #do not include @
cover = ""
tags = ["C++", "Range"]
keywords = ["cpp", "range"]
description = "介绍 C++20 引入的 Range 的使用方法"
showFullContent = false
readingTime = true
hideComments = false
color = "" #color from the theme settings
+++

## 背景

ranges 表示可迭代序列，比如 array、vector、initialized_list。

> The ranges library is an extension and generalization of the algorithms and iterator libraries that makes them more powerful by making them composable and less error-prone.

## 使用

```cpp
int range_example() {
  auto const ints = {0, 1, 2, 3, 4, 5};
  auto even = [](int i) { return 0 == i % 2; };

  std::cout << *std::ranges::begin(ints) << std::endl;

  // Traditional functional composing syntax:
  for (int i : std::ranges::filter_view(ints, even)) {
    std::cout << i << '\n';
  }

  // Pipe syntax of composing the views:
  for (int i : ints | std::views::filter(even)) {
    std::cout << i << '\n';
  }

  return 0;
}
```

## Reference

1. [Ranges](https://en.cppreference.com/w/cpp/ranges)
