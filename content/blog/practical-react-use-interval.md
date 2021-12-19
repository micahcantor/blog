+++
title = "Practical React: The Power of useInterval"
date = 2020-12-20
description = "A case study in switching from manually managing setInterval to the useInterval custom hook."
[taxonomies]
tags = ["react"]
+++

In Dan Abramov's blog post ["Making setInterval Declarative with React Hooks"](https://overreacted.io/making-setinterval-declarative-with-react-hooks/), he writes about a custom React hook to wrap the native JavaScript function, setInterval. The first time I read the article, I thought it was interesting, but I was not convinced by the power of custom hooks until I got the chance to use it in action. That opportunity came when I was designing a music progress bar component. The switch from using `setInterval` to `useInterval` resulted in a dramatic reduction in code size and a cleaner design.

The design of `useInterval` is fairly simple. It looks similar to `useEffect`, where the first parameter is a callback function. It differs though in the second parameter, which is a number in milliseconds:

```tsx
let [count, setCount] = useState(0);
useInterval(() => {    
  // Your custom logic here    
  setCount(count + 1);  
}, 1000);
```

Importantly, the second parameter can also be a ternary conditional expression as well, which is key to the design I explain in this article. If it is a conditional, like `js›shouldCount ? 1000 : null` then it will only run the callback function if `shouldCount` is true. 

I won't go into the gory details about how `useInterval` works behind the scenes here. For that, read Abramov's blog; it will do a much better job than I can.

## The original progress bar design

With some omissions for brevity, here is what the original progress bar looked like:

```tsx
export const ProgressBar = ({ elapsed, songs, runtime, isPaused, deviceID, room, dispatch }) => {

  const [runtimeMS, setRuntimeMS] = useState(null);
  const [progressActive, setProgressActive] = useState(false);
  const [progressIntervalID, setProgressIntervalID] = useState(null);

  { ... }

  /* Updates progress bar timers on pause state change or when song ends */
  useEffect(() => {
    /* if the timer is active and the song ends, stop the timer */
    if (elapsed >= runtimeMS && progressIntervalID) {
      clearInterval(progressIntervalID);
      setProgressActive(false);
      onProgressComplete();
    }
    /* if the timer is not yet active and song is not paused, start the timer */
    else if (!isPaused && !progressActive) {
      const intervalID = setInterval(() =>
        dispatch({ type: 'increment-elapsed', increment: 50 }),
        50 // every 50 ms
      );
      setProgressActive(true);
      setProgressIntervalID(intervalID);
    }
    /* if the timer is active and the song is paused, stop the timer */
    else if (isPaused && progressActive) {
      setProgressActive(false);
      clearInterval(progressIntervalID);
    }
  }, [elapsed, isPaused, progressActive, runtimeMS, progressIntervalID, onProgressComplete, dispatch])

  /* Fires when the first song in queue has been updated (song skip) -- resets progress bar values to their initial state */
  useEffect(() => {
    dispatch({ type: 'set-elapsed', elapsed: 0 });
    setRuntimeMS(parseFloat(runtime));
    setProgressActive(false);
  }, [songs[0], runtime, dispatch])

  /* Clean up function that clears timers on dismount */
  useEffect(() => {
    return () => {
      clearInterval(progressIntervalID);
    }
  }, [progressIntervalID])

  { ... }

  return (
    <SliderInput className="w-full pb-6 mb-1 md:mb-0" min={0} max={runtimeMS} value={elapsed} onChange={onChange}>
  );
```

As you can see, even with some of the other logic removed, this component was a bit of a behemoth. Most of the ugliness stems from the three separate `useEffect` calls that were responsible for updating the progress bar. 

The first function intended to carefully manage the state of the timers when the pause state changed or the song ended. This probably would have been cleaner if it were split up into multiple reducers managing the state of the progress bar and the pause/play button, but that's not the design I had at the time. Instead, the function would trigger when `isPaused` changed. Then, if the song were playing (`!isPaused`) and the timer was not already active, it would create a new interval timer that updated every 50 ms. If the song was paused and the timer was already active, it would stop the clock. It would do the same when the song ended (`elapsed >= runtimeMS`).

Alongside this, the second function would reset the progress bar when a song had been skipped, and the third function would clear all timers when React dismounted the component. Adding to the overhead is the need for local state variables for tracking the id of the setInterval function, and for tracking whether the timer was active.

Yikes. I won't pretend this was good design. It's confusing, difficult to read, and very fragile for such an important component in a music player. There may have been ways to improve the design while retaining setInterval, but after coming across `useInterval`, I decided to rewrite it using the custom hook in order to simplify the component.

## Switching to useInterval

First of all, I installed [an implementation of useInterval](https://github.com/donavon/use-interval) with `npm i @use-it/interval`. Here's the new version:

```tsx
export const ProgressBar = ({ elapsed, songs, runtime, isPaused, deviceID, room, dispatch }) => {
  
  const [shouldIncrement, setShouldIncrement] = useState(false);
  
  /* Interval that increments elapsed every 50 ms if shouldIncrement is enabled */
  useInterval(() => {
    dispatch({ type: 'increment-elapsed', increment: 50 });
  }, shouldIncrement ? 50 : null)

  /* When pause state changes and a song is in queue, set should increment */
  useEffect(() => {
    setShouldIncrement(!isPaused && songs.length > 0)
  }, [isPaused, songs])

  /* When the song ends, go to the next song */
  useEffect(() => {
    if (elapsed >= runtime) { /* when the progress bar reaches the end */
      if (songs.length === 0) { /* turn off increment if this was the last song in queue*/
        setShouldIncrement(false);
      }
      dispatch({ type: 'next-song' });
    }
  }, [elapsed, runtime, room, songs, dispatch])

  { ... }

  return (
    <SliderInput className="w-full pb-6 mb-1 md:mb-0" min={0} max={runtimeMS} value={elapsed} onChange={onChange}>
  );
```

You should be able to see at a glance that this has a much cleaner design. Instead of the confusing and long first `useEffect` in the original design, most of that behavior is now handled by a conditional usage of `useInterval`. Again, I'll emphasize here the importance of the second parameter to `useInterval`: `js› shouldIncrement ? 50 : null`. This simple ternary sets the callback function to trigger every 50 ms if `shouldIncrement` is true. In that callback function, all we have to do is increment the state of the progress bar. This kind of conditional is a powerful and expressive change to the design of setInterval which makes this hook such a joy to use.

How do we know what `shouldIncrement` should be? that's what the other two `useEffect` calls are for. The first updates shouldIncrement if a song is playing and there is a song in queue. The second one simply turns off the interval when the song reaches the end.

This new design is simpler, easier to read, and less buggy since manually managing the timer has been abstracted away with `useInterval`. That means there is no need for additional state variables to hold the id of the timer and whether it is active in this new design.

Now, it's certainly not perfect, since it still relies on `useEffect` to update state when other state changes. That is usually not a best practice, but fixing that problem involved a better global state management design that I did not understand how to implement at the time I wrote this component. I'm also still a student of React, so forgive me for any other design errors.

Still, switching to `useInterval` resulted in a marked improvement, and if your React code requires carefully managing timers, I would recommend considering a switch to it.