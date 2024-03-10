---
title: Asyncio Library in Python and Concurrency tasks
date: 2024-03-10
categories: [Python]
tags: [python, scheduler]     # TAG names should always be lowercase
---
Asyncio Library and Concurrency tasks in Python
===============================================

The `asyncio` library is a Python standard library module used for writing single-threaded concurrent code using coroutines, multiplexing I/O access, and running network clients and servers. It provides a framework that revolves around the event loop, enabling asynchronous programming, which is particularly useful for I/O-bound and high-level structured network code.

Here's a brief overview of its key features:

1. **Event Loop** : The core of asyncio is the event loop. It's the central execution device that manages and distributes the execution of different tasks. It's used to schedule asynchronous tasks and callbacks, handle network I/O events, and run subprocesses.
2. **Coroutines and Tasks** : Coroutines are the fundamental units of work in asyncio, declared using the `async def` syntax. They are used to write asynchronous functions. A Task is then used to schedule coroutines concurrently.
3. **Futures** : A Future is a special low-level awaitable object representing an eventual result of an asynchronous operation.
4. **Synchronization Primitives** : `asyncio` provides primitives like locks, events, conditions, and semaphores, similar to those in the `threading` module, but designed for the asyncio tasks.
5. **Subprocesses** : It supports the creation and management of subprocesses.
6. **Queues** : It provides a queue class that can be used to distribute work between different coroutines.
7. **Streams** : High-level API for working with network connections, allowing easy implementation of protocols and transports.
8. **Exception Handling** : `asyncio` tasks have their own exception handling. Exceptions are stored and can be retrieved when the task is done.

```python
import asyncio

async def main():
    print('Hello')
    await asyncio.sleep(1)
    print('World')

asyncio.run(main())

# Example in concurrency 
async def fetch_data():
    reader, writer = await asyncio.open_connection('example.com', 80)
    request_header = 'GET / HTTP/1.0\r\nHost: example.com\r\n\r\n'
    writer.write(request_header.encode())
    await writer.drain()
    data = await reader.read(100)
    print(data.decode())
    writer.close()
    await writer.wait_closed()

asyncio.run(fetch_data())
```

## Unit test of asycio

```python
import unittest
import asyncio
from your_module import AsyncLocalClient  # Import your AsyncLocalClient class

class TestAsyncLocalClient(unittest.TestCase):

    def setUp(self):
        self.client = AsyncLocalClient()

    def test_command_execution(self):
        loop = asyncio.get_event_loop()
        command = 'echo "Hello World"'
        result = loop.run_until_complete(self.client.run_command(command))
        self.assertIn("Hello World", result.stdout)
        self.assertEqual(result.exit_code, 0)

    def test_command_failure(self):
        loop = asyncio.get_event_loop()
        command = 'ls non_existent_directory'
        result = loop.run_until_complete(self.client.run_command(command))
        self.assertNotEqual(result.exit_code, 0)
        self.assertTrue(result.stderr)

    def test_async_execution(self):
        loop = asyncio.get_event_loop()
        commands = ['sleep 1', 'sleep 2', 'echo "async test"']
        tasks = [self.client.run_command(cmd) for cmd in commands]
        results = loop.run_until_complete(asyncio.gather(*tasks))
        self.assertEqual(len(results), 3)
        self.assertIn("async test", results[-1].stdout)

if __name__ == '__main__':
    unittest.main()

```

run test with `python -m unittest test_async_local_client.py`

## Examples use asyncio

```python
async def call(command):
    vc_client = AsyncLocalClient()
    try:
        tasks = [vc_client.run_command(command)]
        output = await asyncio.gather(*tasks,return_exceptions=True)
    except Exception as e:
        print(f"An error occurred: {type(e).__name__}, {e}")
    return output
# Test run it like:
asyncio.run(call(command), debug=True)

```

+ When need to include the environment using asyncio

```python
env = os.environ.copy()
env['LD_LIBRARY_PATH'] = '[path]/lib:' + env.get('LD_LIBRARY_PATH', '')
```

+ How to create multi-tasks and gather their output

```python
async def call(command):
    client = AsyncLocalClient()
    task = asyncio.create_task(call_async_func(cmds), name=str(idx))
    tasks.append(task)
    outputs = await asyncio.gather(*tasks, return_exceptions=True)
    return outputs

def main():
    results = asyncio.run(call(command))
```
