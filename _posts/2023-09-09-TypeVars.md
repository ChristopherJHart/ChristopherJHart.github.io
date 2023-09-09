---
layout: post
title: Practical Generic Type Hinting in Python
---

If you've ventured into type hinting in Python, you're probably aware of the fundamentals. You're comfortable hinting function arguments and return types using basic types like `int` or `str`, and you may even be familiar with more complex typing like `List[str]` or `Dict[str, int]`. But the prospect of *generic* type hinting may seem confusing. What are they? Why do they exist? How do I use them? What value do they add?

One of my favorite pieces of wisdom from [The Grug Brained Developer](https://grugbrain.dev/) is regarding type systems like Python's type hinting:

> grug very like type systems make programming easier. for grug, **type systems most value when grug hit dot on keyboard and list of things grug can do pop up magic.** this 90% of value of type system or more to grug

In my opinion, this is the key value that type hinting brings to Python software development, especially if you're not leveraging a static type checker like [mypy](https://mypy.readthedocs.io/en/stable/). Type hinting allows your IDE of choice to best assist you through suggestions or autocomplete options while writing software. This enables you to write software quicker and with more confidence.

The purpose of generic type hinting is to clarify the types of objects that are passed into or returned from a function in situations where the type of an object might otherwise be ambiguous or "swallowed up" by the type hinting of the function itself.

Let's demonstrate this with a simple example: We have a function named `chunk_list` that takes in a list, and chunks it up into a list of smaller lists.

```python
def chunk_list(big_list: List[Any], chunk_size: int) -> List[List[Any]]:
    """Break up a big list into smaller lists."""
    for i in range(0, len(big_list), chunk_size):
        yield big_list[i : i + chunk_size]  # noqa: E203
```

At first glance, the type hinting of the `chunk_list` function seems fine. The list passed into the function through the `big_list` argument could contain any type, so the `Any` type hinting seems appropriate. The return type of `List[List[Any]]` also seems logically appropriate, as we're returning a list of lists, and each list could contain any type.

Let's use this function to chunk up a list of integers into smaller lists.

```python
import sys
from typing import Any, List

def chunk_list(big_list: List[Any], chunk_size: int) -> List[List[Any]]:
    """Break up a big list into smaller lists."""
    for i in range(0, len(big_list), chunk_size):
        yield big_list[i : i + chunk_size]  # noqa: E203


def main() -> None:
    """Main entry point."""
    big_list = list(range(5))

    for chunk in chunk_list(big_list, 2):
        print(chunk)


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        sys.exit(1)

```

We can run this program and see that it works as advertised:

```
christopher@ubuntu-playground:~/GitHub/typevar-example$ python3 main.py 
[0, 1]
[2, 3]
[4]
```

The problem is when we try to modify our code further to use the `chunk` variable pulled from the `chunk_list` function. Because we've type hinted the return value as `Any`, our IDE doesn't know what options to offer us when trying to interact with this object. If we hover our cursor over the `chunk` object, it tells us it's of type `List[Any]`.

![]({{ site.baseurl }}/images/2023/typevars/chunk_cursor_hover.png)

If we extract the first item from the list `chunk`, we still get no help from our IDE.

![]({{ site.baseurl }}/images/2023/typevars/first_item_chunk_cursor_hover.png)

If we attempt to use the first item, our IDE is still of no help. To use this item properly, we have to mentally know that the first item is an `int` and that we can use it as such.

![]({{ site.baseurl }}/images/2023/typevars/first_item_chunk_dot.png)

Let's refactor our function to use generic type hinting through `TypeVar` and see how that helps us. We define a new TypeVar named `T` and use it to type hint the `big_list` argument and the return value of the function.

```python
import sys
from typing import List, TypeVar

T = TypeVar("T")


def chunk_list(big_list: List[T], chunk_size: int) -> List[List[T]]:
    """Break up a big list into smaller lists."""
    for i in range(0, len(big_list), chunk_size):
        yield big_list[i : i + chunk_size]  # noqa: E203


def main() -> None:
    """Main entry point."""
    big_list = list(range(5))

    for chunk in chunk_list(big_list, 2):
        print(chunk)


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        sys.exit(1)

```

If we re-run this program, it executes precisely the same:

```
christopher@ubuntu-playground:~/GitHub/typevar-example$ python3 main.py 
[0, 1]
[2, 3]
[4]
```

The real power from our use of generics comes when we interact with the `chunk` variable. If we hover over it, we see that it's of type `List[int]`.

![]({{ site.baseurl }}/images/2023/typevars/typed_chunk_cursor_hover.png)

If we extract the first item from the list and hover our cursor over it, our IDE confirms it's of type `int`.

![]({{ site.baseurl }}/images/2023/typevars/first_item_typed_chunk_cursor_hover.png)

Finally, if we attempt to use the first item, our IDE knows it's an `int` and offers us all the options we'd expect.

![]({{ site.baseurl }}/images/2023/typevars/first_item_typed_chunk_dot.png)

This is the power of generic type hinting. It allows us to be more confident in our code and write it more quickly. It also allows us to be more confident in the code of others, as our IDE can help us more easily understand what types of objects are being passed around code that may be unfamiliar to us.

[The Grug Brained Developer](https://grugbrain.dev/) put it best:

> always most value type system come: **hit dot see what grug can do, never forget!**
