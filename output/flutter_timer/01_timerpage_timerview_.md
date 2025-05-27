# Chapter 1: TimerPage/TimerView

Welcome to the exciting world of building a timer application with Flutter! In this first chapter, we'll dive into the basic building blocks you see on the screen: the `TimerPage` and the `TimerView`.

Think of our timer app like a physical timer you might use in your kitchen. There are two main parts:

1.  **The whole timer device:** This is like our `TimerPage`. It's the main container that holds everything together and sets up all the internal workings.
2.  **What you see on the front:** This is like our `TimerView`. It's the screen that shows the time counting down and the buttons (like Start, Pause, Reset) you press to control the timer.

So, the `TimerPage` is the main screen that gets our timer ready to go, and the `TimerView` is the part you actually look at and interact with. They work together to make the visual part of our timer app.

Let's look at how these concepts are put together in our code.

### TimerPage: Setting the Stage

The `TimerPage` is simple but important. Its main job is to set up something called a [TimerBloc](05_timerbloc_.md). We'll learn more about what a [TimerBloc](05_timerbloc_.md) is later, but for now, just know it's like the "brain" of our timer. It handles all the logic – like counting down the time, keeping track of whether the timer is running or paused, and so on.

Here's a simplified look at the `TimerPage` code:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter_timer/ticker.dart'; // Don't worry about Ticker yet
import 'package:flutter_timer/timer/timer.dart'; // Imports TimerBloc and more

class TimerPage extends StatelessWidget {
  const TimerPage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider( // Setting up the TimerBloc
      create: (_) => TimerBloc(ticker: const Ticker()),
      child: const TimerView(), // Displaying the TimerView
    );
  }
}
```

In this code:

*   We use `BlocProvider` to make our `TimerBloc` available to other parts of the app that need it (specifically the `TimerView` and its internal parts).
*   We then display the `TimerView` (`const TimerView()`), which is what the user will actually see.

So, `TimerPage` is like preparing the stage and setting up the main actor (the `TimerBloc`).

### TimerView: What You See and Touch

The `TimerView` is where all the visual elements of our timer come together. This is where you'll see the time displayed and the buttons to control the timer.

Here's a simplified look at the structure of the `TimerView`:

```dart
class TimerView extends StatelessWidget {
  const TimerView({super.key});

  @override
  Widget build(BuildContext context) {
    return const Scaffold( // Basic layout structure
      body: Stack( // Allows stacking widgets
        children: [
          Background(), // A background gradient (visual detail)
          Column( // Arranges items vertically
            mainAxisAlignment: MainAxisAlignment.center, // Centers content
            children: <Widget>[
              Padding( // Adds space around the TimerText
                padding: EdgeInsets.symmetric(vertical: 100),
                child: Center(child: TimerText()), // Displays the time
              ),
              Actions(), // Contains the buttons
            ],
          ),
        ],
      ),
    );
  }
}
```

Inside the `TimerView`:

*   We use a `Scaffold` for the basic app layout.
*   We use a `Stack` to place the background *behind* the main content (the time and buttons).
*   A `Column` organizes the time display (`TimerText`) and the buttons (`Actions`) one below the other, centered on the screen.
*   `TimerText` and `Actions` are separate widgets that handle displaying the time and the buttons, respectively. We'll look at these in more detail in the next chapters: [Actions (Widget)](02_actions__widget__.md) and [TimerText (Widget)](03_timertext__widget__.md).

The `TimerView` doesn't do any counting or logic itself. It just shows information it gets and tells the "brain" ([TimerBloc](05_timerbloc_.md)) when a button is tapped.

### How TimerPage and TimerView Work Together (High Level)

Imagine our timer:

1.  When the app starts, the `TimerPage` is built.
2.  The `TimerPage` creates the `TimerBloc` (the brain) and provides it to the rest of the app.
3.  The `TimerPage` then shows the `TimerView`.
4.  The `TimerView` displays the initial time using the [TimerText](03_timertext__widget__.md) widget.
5.  The `TimerView` shows the initial buttons using the [Actions (Widget)](02_actions__widget__.md) widget.
6.  When you tap a button in the [Actions (Widget)](02_actions__widget__.md) widget (part of the `TimerView`), it sends a message to the [TimerBloc](05_timerbloc_.md) (the brain).
7.  The [TimerBloc](05_timerbloc_.md) updates the time.
8.  The `TimerView` notices the time has changed (because it's watching the [TimerBloc](05_timerbloc_.md)) and updates the [TimerText](03_timertext__widget_.md) to show the new time.
9.  The [TimerBloc](05_bloc_.md) might also tell the `TimerView` that the timer is now running (or paused), so the `TimerView` updates the [Actions (Widget)](02_actions__widget_.md) to show the correct buttons (Pause instead of Start).

It's like the `TimerView` is the face and hands of a clock, and the [TimerBloc](05_timerbloc_.md) is the gears and motor inside. The face shows the time, and the buttons on the outside let you interact, but the internal mechanism ([TimerBloc](05_timerbloc_.md)) is what actually keeps track of the time.

### Conclusion

In this chapter, we learned about `TimerPage` and `TimerView`, the foundational visual parts of our timer application. `TimerPage` sets up the "brain" ([TimerBloc](05_timerbloc_.md)), and `TimerView` is the layout that displays the time and the control buttons. They work together, with `TimerView` showing information from the [TimerBloc](05_timerbloc_.md) and telling the [TimerBloc](05_timerbloc_.md) when a user taps a button.

Next, we'll look closer at the [Actions (Widget)](02_actions__widget__.md) widget, which is responsible for displaying the buttons you use to control the timer.

[Next Chapter: Actions (Widget)](02_actions__widget__.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)