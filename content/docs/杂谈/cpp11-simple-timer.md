---
title: "利用 chrono 实现简单的计时器"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 利用 chrono 实现简单的计时器

通过使用 C++11 的 `chrono` 库实现简单的计时器，可以精确到毫秒。

```cpp
#include <iostream>
#include <chrono>

// A simple timer
using std::chrono::system_clock;
using std::chrono::duration_cast;
using std::chrono::milliseconds;

int main() {
	// Begin timer
	auto start = system_clock::now();

	for (int i = 1; i < 1000000000; ++i);

	// End timer
	auto end = system_clock::now();
	// Calculate time in ms
	auto duration = duration_cast<milliseconds>(end - start);
	std::cout << "It takes " << duration.count() << " ms";
}
```