---
title: "Modifying Source Code using AST"
layout: post
excerpt: " "
tags:
  - python
---

## Objective

We will modify the following function to return `num * num` instead of `2 * num` at runtime.

```
def func(num):
    return 2 * num
```
We will be working with the AST representation of the function. So let's first get the AST of the return value we want.

```
import ast

source_code = "return num * num"
print(ast.dump(ast.parse(source_code)))

[Output]
Module(body=[Return(value=BinOp(left=Name(id='num', ctx=Load()), op=Mult(), right=Name(id='num', ctx=Load())))], type_ignores=[])
```

We need the **value** field from the above output. Note that we need to add `ast` before each method names.

```
value = ast.BinOp(
            left=ast.Name(id="num", ctx=ast.Load()),
            op=ast.Mult(),
            right=ast.Name(id="num", ctx=ast.Load()),
        )
```

Now we can sub-class `ast.NodeTransformer` to modify the AST of the function.  
After modifying the AST, we will replace the original function source_code with the modified one.

```
import ast, functools, inspect


# Original Source
def func(num):
    return 2 * num

class AST_Modifier(ast.NodeTransformer):
    def visit_Return(self, node):
        # Update node with AST value we obtained above
        node.value = ast.BinOp(
            left=ast.Name(id="num", ctx=ast.Load()),
            op=ast.Mult(),
            right=ast.Name(id="num", ctx=ast.Load()),
        )

        return self.generic_visit(node)


def transform(method):
    # Get source code of method
    source_code = inspect.getsource(method)
    # Generate AST
    tree = ast.parse(source_code)
    # Get Modified AST
    AST_Modifier().visit(tree)
    ast.fix_missing_locations(tree)
    # Compile the source code
    code = compile(tree, inspect.getfile(method), "exec")
    # Update the original function definition
    namespace = method.__globals__
    exec(code, namespace)
    new_function = namespace[method.__name__]
    return functools.update_wrapper(new_function, method)

print(func(3))
transform(func)
print(func(3))

[Output]
6
9
```

Clearly all that we have achieved here is a decorator *without a wrapper function*. A valid use case for these gymnastics is left as a job for the reader!


_References:_  
- <https://stackoverflow.com/questions/58579837/changing-function-code-using-a-decorator-and-execute-it-with-eval>  
- <https://stackoverflow.com/questions/31078439/modify-function-in-decorator>
- <https://stackoverflow.com/questions/768634/parse-a-py-file-read-the-ast-modify-it-then-write-back-the-modified-source-c>
- <https://greentreesnakes.readthedocs.io/en/latest/examples.html>
