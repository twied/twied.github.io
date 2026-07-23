---
title: "The std::map antipattern"
date: 2026-07-15
tags:
  - cpp
excerpt: >
  Here is an antipattern I see every now and then in C++ codebases, and have
  even written myself:
  ```cpp
  std::map<int, Widget>
  ```
  where the map of the key is repeated in the value struct, just sitting there,
  taking up memory and begging to go out of sync.
  
  The solution is a feature introduced in C++14, that allows heterogeneous
  lookup for `std::set`...
---

Here is a pattern I see every now and then in C++ codebases, and have even 
written myself:

```cpp
struct Employee {
    int id;
    std::string name;
    ...
};

std::map<int, Employee> employees;
```

The key is the employee's id. The value is the employee, which also contains
the id. The id is stored twice: once as the map key, once inside the
struct. Every insertion duplicates it, every lookup compares against the copy
in the key, and the copy inside the value just sits there, taking up memory
and begging to go out of sync.

This is one of those little code smells that irk me and taunt me, but this time
I actually have a solution. C++14 came in to the rescue.

```cpp
struct Employee {
    int id;
    std::string name;
    ...
};

bool operator<(const Employee& lhs, const Employee& rhs) {
    return lhs.id < rhs.id;
}

bool operator<(const Employee& lhs, const int& rhs) {
    return lhs.id < rhs;
}

bool operator<(const int& lhs, const Employee& rhs) {
    return lhs < rhs.id;
}


std::set<Employee, std::less<>> employees;
```

Make the container a `std::set` instead of a `std::map`. Using `std::less<>` 
enables heterogeneous lookup. No need to construct a full object, 
`std::set::find()` can then accept an argument of the key type.

```
int main() {
    employees.insert({1, "Neo" });
    employees.insert({2, "Trinity" });
    employees.insert({3, "Morpheus" });

    if (auto search = employees.find(1); search != employees.end())
        std::cout << "Found " << search->name << '\n';
    else
        std::cout << "Not found\n";

    return 0;
}
```
