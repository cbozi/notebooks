debounce throttle区别

To put it in simple terms:

- Throttling will delay executing a function. It will reduce the notifications of an event that fires multiple times.

- Debouncing will bunch a series of sequential calls to a function into a single call to that function. It ensures that one notification is made for an event that fires multiple times.

You can visually see the difference here

If you have a function that gets called a lot - for example when a resize or mouse move event occurs, it can be called a lot of times. If you don't want this behaviour, you can Throttle it so that the function is called at regular intervals. Debouncing will mean it is called at the end (or start) of a bunch of events.

