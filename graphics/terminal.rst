Terminal Functionality
======================

When creating terminal-based toolkit for game rendering, three features
must be met. Unbuffered input, nonblocking input, and detecting terminal size
change.

Unbuffered Input
****************

Unbuffered input is sending keys without needing to press enter.

**Linux**

.. code-block:: python3

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

**Windows**

.. code-block:: python3
    
    import msvcrt

    msvcrt.getch() # read and return one byte from stdin


Non-blocking Input
******************

Normally, reading from stdin will block the thread. The portable workaround is as follows

.. code-block:: python3
    

    import asyncio

    loop = asyncio.get_event_loop()

    async def ainput():
        # Run input in separate thread
        return await loop.run_in_executor(None, input)

    # Prints the input on enter
    async def main():
        while True:
            print(await ainput())
    
    # To demonstrate nonblocking, print n seconds passed since program start
    async def count():
        i = 0
        while True:
            print(i)
            await asyncio.sleep(1)
            i += 1
    
    loop.create_task(main())
    loop.create_task(count())
    loop.run_forever()


While this is simple, a thread needs to be created every time stdin is read. 
A slightly more complicated solution is to create a dedicated thread to read input
and push the result to a queue.

.. code-block:: python3

    import asyncio
    import threading

    loop = asyncio.get_event_loop()
    queue = asyncio.Queue()
    
    # Dedicated thread worker that blocks on call to input. This won't block
    # main thread. However, when we put the result into our queue, we will have
    # to context swap back to main thread which adds overhead.
    def reader():
        while True:
            asyncio.run_coroutine_threadsafe(queue.put(input()), loop)
    
    # Wait for input from queue.
    async def ainput():
        return await queue.get()
    

    # Prints the input on enter
    async def main():
        while True:
            print(await ainput())
    
    # To demonstrate nonblocking, print n seconds passed since program start
    async def count():
        i = 0
        while True:
            print(i)
            await asyncio.sleep(1)
            i += 1
    
    # Start our thread worker
    threading.Thread(target=reader, daemon=True).start()
    
    # Register our coroutines to event loop
    loop.create_task(count())
    loop.create_task(main())
    # Run the loop
    loop.run_forever()

While cheaper than the first method, this solution is still subject to GIL. The
main loop still needs to release GIL to allow the reader thread to read from stdin.
Unlike the previous example, we call ``input`` since it only blocks the reader thread.


Ideally, reading from stdin simply does not block. If there is data, stdin returns the data.
Otherwise, stdin returns an error. As far as I know, non-blocking I/O is for the most part non portable.

**On linux**

.. code-block:: python3
    
    import fcntl
    import contextlib
    import os
    import sys

    @contextlib.contextmanager
    def create_read_noblock():
        
        # Retrieve current flags for stdin file descriptor
        old = fcntl.fcntl(0, fcntl.F_GETFL)
        
        # Toggle on the O_NONBLOCK flag by doing a bitwise OR with old flags.
        # This ensures we save all the previous settings
        fcntl.fcntl(0, fcntl.F_SETFL, old | os.O_NONBLOCK)
        try:
            yield lambda n: sys.stdin.read(n)
        finally:
            # Restore the old settings (with O_NONBLOCK toggled off)
            fcntl.fcntl(0, fcntl.F_SETFL, old)


    with create_read_noblock() as read:
        while True:
            # What if we called read(2) instead?
            if (c := read(1)):
                print(type(c))
        

On Windows

.. code-block:: python3

    # TODO
    pass


This example only implements non-blocking read. We actually want our program to 
assume the position of both *unbuffered input* and *non-blocking* read.


Detecting terminal size change event
************************************

Knowing the size of the terminal is essential for proper rendering. Python
provides a platform independent way of checking for terminal size.

.. code-block:: python3

    import shutil
    nrow, ncols = shutil.get_terminal_size()

However, since the size of a terminal may also change during execution of the
program, we also want to know when the size change occurs.

**Linux**

.. code-block:: python3

    """
    A program that prints out the size of the terminal on resize
    """
    
    import signal
    import shutil

    # Handler for SIGWINCH that is called when the program receives
    # SIGWINCH signal
    def Handle_SIGWINCH(*args):
        row, col = shutil.get_terminal_size()
        print(row, col)

    # Install the signal handler
    signal.signal(signal.SIGWINCH, Handle_SIGWINCH) 

    # Show the current size of the terminal
    row, col = shutil.get_terminal_size()
    print(row, col)
    
    # Try changing the size of the terminal while this process is running
    while True:
        pass

..

**Windows**

Unfortunately, SIGWINCH is not available on Windows. More unfortunate is that I
do not know how to handle terminal size change on Windows.

.. code-block:: python3

    # TODO Learn how to handle term size change
    pass


Integration
***********

As a challenge for the reader, implement a program that unifies unbuffered and
non-blocking reads, and can detect terminal size change.

.. code-block:: python3

    from typing import Tuple, Any
    import asyncio

    EVENT_NONE = 0 # Nothing happened
    EVENT_TERM_SIZE = 1 # Terminal size change
    EVENT_KEY = 2 # Key pressed

    async def event() -> Tuple[int, Any]:
        """
        Retrieve an event.

        Returns:
            tuple((EVENT_NONE, "")) if nothing happened
            tuple((EVENT_TERM_SIZE, (NEW_ROW, NEW_COL))) if terminal resized
            tuple((EVENT_KEY, str)) if key was pressed
        """
        pass

    loop = asyncio.get_event_loop()

    async def main():
        while True:
            event_code, event_data = await event()
            if event_code == EVENT_NONE:
                pass
            elif event_code == EVENT_TERM_SIZE:
                nrow, ncol = event_data
                print(f"Terminal size: {nrow}, {ncol}")
            elif event_code == EVENT_KEY:
                print(f"Pressed {event_data}")
    

    loop.create_task(main())
    loop.run_forever()
