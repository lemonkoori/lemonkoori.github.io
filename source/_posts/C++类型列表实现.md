---
title: C++类型列表实现
date: 2026/3/11 13:34:00
categories:
	- c++
tags:
	- 模板
toc: true
---
```c++
template<typename... Ts>
struct TypeList {};
```
## 列表合并
```c++
template<typename... L>
struct TypeListMerge;

template<typename... T1s, typename... T2s, typename... Rest>
struct TypeListMerge<TypeList<T1s...>, TypeList<T2s...>, Rest...> {
    using type = TypeListMerge<TypeList<T1s..., T2s...>, Rest...>::type;
};

template<typename L>
struct TypeListMerge<L> {
    using type = L;
};
```
## 列表划分
```c++
template<typename L, typename P, template <typename, typename> typename Comp>
struct TypeListPartition;

template<typename... Ts, typename P, template<typename,typename> typename Comp>
struct TypeListPartition<TypeList<Ts...>, P, Comp> {
    using satisfied = TypeListMerge<std::conditional_t<
        Comp<P, Ts>::value, TypeList<Ts>, TypeList<> >...>::type;
    using rest = TypeListMerge<std::conditional_t<
        !Comp<P, Ts>::value, TypeList<Ts>, TypeList<> >...>::type;
};
```
一个可行的 `Comp` 实现
```c++
template<typename T>
constexpr std::string_view type_str() {
    return std::source_location::current().function_name();
}

template<typename T1, typename T2>
struct TypeComp {
    static constexpr bool value = type_str<T1>() < type_str<T2>();
};
```

## 列表排序
```c++
template<typename L, template<typename,typename> typename Comp>
struct TypeListSort;

template<typename First, typename... Rest, template<typename,typename> typename Comp>
struct TypeListSort<TypeList<First, Rest...>, Comp> {
    using parts = TypeListPartition<TypeList<Rest...>, First, Comp>;
    using type = TypeListMerge<typename TypeListSort<typename parts::satisfied, Comp>::type, TypeList<First>,
        typename TypeListSort<typename parts::rest, Comp>::type>::type;
};

template<typename T, template<typename,typename> typename Comp>
struct TypeListSort<TypeList<T>, Comp> {
    using type = TypeList<T>;
};

template<template<typename,typename> typename Comp>
struct TypeListSort<TypeList<>, Comp> {
    using type = TypeList<>;
};
```