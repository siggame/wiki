Curses
======

.. code-block:: python3

    import curses
    # async for real-time games
    import asyncio


    loop = asyncio.get_event_loop()

    with open(
        '/dev/pts/3', # Open a second terminal, and type tty. Replace this argument with the output
        'w'
    ) as fp:
        
        # A function that prints to the newly opened terminal
        debug = lambda *args:print(*args,file=fp)

        @curses.wrapper
        # Changes Terminal Environment for unbuffered input - User does not need to 
        # press enter to put input in
        def _start(stdscr):
            """
            args:
                stdscr - a window object initialized by curses
            """
            # y_pos & x_pos for where our character will go
            y_pos, x_pos = 0,0

            # we can find the "edges" of the screen by this command which uses the
            # standard screen object
            max_y, max_x = stdscr.getmaxyx()
            
            stdscr.nodelay(1) # Does not block main thread
            
            async def main():
                while True:
                    if (ch := stdscr.getch()) != curses.ERR:
                        # We are creating a ch varaible fom stdscr.getch() and 
                        # comparing that to curses.ERR := is called the 
                        # "walrus operator"
                        debug(ch)
                        key = chr(ch) #chr function converts `ch` (a byte object) to a string
                        if key == "q":
                            exit()
                     
                        elif key == "w":
                            y_pos-=1
                            if(y_pos<0):
                                y_pos=0
                        elif key == "a":
                            x_pos-=1
                            if(x_pos<0):
                                x_pos=0
                        elif key == "s":
                            y_pos+=1
                            if(y_pos == max_y):
                                y_pos = max_y - 1
                        elif key == "d":
                            x_pos+=1
                            if(x_pos == max_x):
                                x_pos = max_x - 1
                        stdscr.erase() # Deletes previous move
                        stdscr.move(y_pos, x_pos) # Move our object to y,x spot
                        stdscr.insch('@') # choose our symbol that we move around

                    await asyncio.sleep(1/120) #created event loop


            task = loop.create_task(main()) #created event loop with main function

            # Call this function once our task is complete
            @task.add_done_callback
            def on_complete(_):

                # Retrieve our exception
                if task.exception():
                    debug(task.exception())
                    exit()
                    #then exit after we print it

            loop.run_forever() # want the program to continue running until we dont


