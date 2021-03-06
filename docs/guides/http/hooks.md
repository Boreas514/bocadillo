# Hooks

## What are hooks?

Hooks allow you to call arbitrary code before and after a view is executed. They materialize as the `@before()` and `@after()` decorators located in the `bocadillo.hooks` module.
 
These decorators take a **hook function**, which is a synchronous or asynchronous function with the following signature: `(req: Request, res: Response, params: dict) -> None`.

::: tip CHANGED IN v0.9.0
Hooks decorators are now located in a separate `hooks` module. Use `@hooks.<hook>` instead of `@api.<hook>`.

Plus, they must now be the first decorators of a view, instead of `@api.route()`.
:::

## Example

```python
from asyncio import sleep
from bocadillo import HTTPError, hooks

def validate_has_my_header(req, res, params):
    if 'x-my-header' not in req.headers:
        raise HTTPError(400)

async def validate_response_is_json(req, res, params):
    await sleep(1)  # for the sake of example
    assert res.headers['content-type'] == 'application/json'

@api.route('/foo')
@hooks.before(validate_has_my_header)
@hooks.after(validate_response_is_json)
async def foo(req, res):
    res.media = {'message': 'valid!'}
```

::: tip
The ordering of decorators is important: **hooks should always be a view's first decorators**.
:::

## Hooks and reusability

As a first level of reusability, you can pass extra positional or keyword arguments to `@api.before()` and `@api.after()`, and they will be handed over to the hook function:

```python
def validate_has_header(req, res, params, header):
    if header not in req.headers:
        raise HTTPError(400)

@api.route('/foo')
@hooks.before(validate_has_header, 'x-my-header')
async def foo(req, res):
    pass
```

A hook function only just needs to be a callable, so it can be a class that implements `__call__()` too. This is another convenient way of building reusable hooks functions:

```python
class RequestHasHeader:
    
    def __init__(self, header):
        self.header = header
       
    def __call__(self, req, res, params):
        if self.header not in req.headers:
            raise HTTPError(400)

@api.route('/foo')
@hooks.before(RequestHasHeader('x-my-header'))
async def foo(req, res):
    pass
```

You can also use hooks on class-based views:

```python
def show_content_type(req, res, view, params):
    print(res.headers['content-type'])

@api.route('/')
# applied on all method views
@hooks.after(show_content_type)
class Foo:

    @hooks.before(RequestHasHeader('x-my-header'))
    async def get(self, req, res):
        res.media = {'header': req.headers['x-my-header']}
```
