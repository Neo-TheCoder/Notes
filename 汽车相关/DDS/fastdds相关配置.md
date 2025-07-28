# 内存分配mode
## enumerator PREALLOCATED_WITH_REALLOC_MEMORY_MODE
```cpp
  wqos.endpoint().history_memory_policy = eprosima::fastrtps::rtps::PREALLOCATED_WITH_REALLOC_MEMORY_MODE;
```

Default size preallocated, requires reallocation when a bigger message arrives.
Smaller memory footprint at the cost of an increased allocation count.

在初始化时预先分配一定数量的内存，当需要更多内存时，会重新分配更大的内存块。
这种模式可以在一定程度上提高内存管理的效率，同时也可以减少内存碎片化的问题。

## enumerator PREALLOCATED_MEMORY_MODE
Preallocated memory.

Size set to the data type maximum. Largest memory footprint but smallest allocation count.
enumerator PREALLOCATED_WITH_REALLOC_MEMORY_MODE

默认的内存分配方式是`eprosima::fastrtps::rtps::PREALLOCATED_MEMORY_MODE`，即预先分配内存，并在需要时进行重用，而不进行重新分配。这种方式可以提高内存管理的效率，但可能会导致内存碎片化的问题。

