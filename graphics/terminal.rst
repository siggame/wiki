Terminal Functionality
======================

When creating terminal-based toolkit for game rendering, three features
must be met. Unbuffered input, nonblocking input, and detecting terminal size
change.

**Unbuffered Input**

Unbuffered input is sending keys without needing to press enter. On Windows, this
is a simple affair.


.. code-block:: python
    
    import msvcrt

    msvcrt.getch() # read and return one byte from stdin

On Linux, unbuffered input requires more work

.. code-block:: python

    import sys
    import tty
    import termios
    import contextlib

    @contextlib.contextmanager
    def unbuffered():
        fd = sys.stdin.fileno()
        old = termios.tcgetattr(fd) # save the old terminal settings
        tty.setraw(fd)

        try:
            # Return our getch function
            yield lambda: sys.stdin.read(1)
        finally:
            # Restore the old terminal settings
            termios.tcsetattr(fd, termios.TCSANOW, old)


    with unbuffered() as getch:
        # do stuff
        while True:
            print(getch())
    
    # Finally block executes when the above block exits

The contex manager ensures that when the program exits, the terminal setting is
restored. If ``tcsetattr`` is not called on program exit, the terminal will remain in unbuffered mode.

**Non-blocking Input**

Normally, reading from stdin will block the thread. The portable workaround is as follows

.. code-block:: python3
    
    import asyncio

    loop = asyncio.get_event_loop()

    async def ainput():
        return await loop.run_in_executor(None, input)

While this is simple, a thread needs to be created every time stdin is read. 
A slightly more complicated solution is to create a dedicated thread to read input
and push the result to a queue.

.. code-block:: python3

    import asyncio
    import threading

    loop = asyncio.get_event_loop()
    queue = asyncio.Queue()

    def reader():
        while True:
            asyncio.run_coroutine_threadsafe(queue.put(input()), loop)
    
    async def ainput():
        return await queue.get()
    
    async def main():
        while True:
            print(await agetch())

    threading.Thread(target=reader, daemon=True).start()
    loop.create_task(main())
    loop.run_forever()

While cheaper than the first method, this solution is still subject to GIL. The
main loop still needs to release GIL to allow the reader thread to read from stdin.
Unlike the previous example, we call ``input`` since it only blocks the reader thread.


Ideally, reading from stdin simply does not block. If there is data, stdin returns the data.
Otherwise, stdin returns an error. As far as I know, non-blocking I/O is for the most part non portable.

On linux

.. code-blocks: python3
    
    # TODO
    pass

On windows

.. code-blocks: python3

    # TODO
    pass


**Detecting terminal size change event**

* [ ] - TODO