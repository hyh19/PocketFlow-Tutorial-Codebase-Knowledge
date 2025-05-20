# Chapter 9: RestorableRouteFuture

Welcome to the final chapter of our Flutter navigation series! We've covered a lot, from the basics of the [Navigator](01_navigator_.md) and [Route](02_route_.md)s to more advanced topics like [Page](04_page_.md)s, [RoutePredicate](05_routepredicate_.md)s, and [TransitionDelegate](08_transitiondelegate_.md).

Today, we'll look at a specialized helper class called **RestorableRouteFuture**. This abstraction is less about the core mechanics of navigation and more about enhancing your app's ability to *remember* where the user was and what they were doing if the app is interrupted and later restored by the operating system.

## Why do we need RestorableRouteFuture?

Imagine your app is running, and the user navigates through a few screens. Then, they tap a button that opens a dialog (which is also a [Route](02_route_.md)). While that dialog is open, the operating system decides it needs to free up memory and closes your app in the background (this is common on mobile devices). Later, the user reopens your app, and the OS tries to restore it to the exact state it was in.

By default, the [Navigator](01_navigator_.md) can often restore the history stack (if configured to do so with `restorationScopeId` and, for Page-based navigation, `Page.restorationId`). However, if you pushed a route imperatively (using methods like `Navigator.push`) and then needed to get a *result* back from it (like a value selected in a dialog), simply restoring the route stack doesn't automatically provide you with the *original Future* that the `push` call returned. If you were awaiting that Future (`await Navigator.push(...)`) to process the result, your code might break after restoration if the screen is rebuilt without the original Future being available.

**RestorableRouteFuture** is designed to solve this problem when you're using state restoration. It helps you maintain a connection to the route you pushed and get access to its return value even after the app is restored.

Think of it like this:

When you imperatively push a route that returns a value (like a dialog that returns the user's selection):

```dart
// Without RestorableRouteFuture, before restoration
Object? result = await Navigator.push(context, MySelectionDialogRoute());
// Do something with result
```

If the app restarts and restores the dialog route, the screen that was originally calling this code is rebuilt. The `await Navigator.push` line was already "completed" (or interrupted) by the time of restoration. The screen needs a way to re-establish its connection to the dialog route and get its result when the dialog is eventually closed.

That's where **RestorableRouteFuture** comes in. It acts as a persistent "handle" to a route pushed imperatively using the *restorable* Navigator methods, ensuring you can still get the return value after state restoration.

## Core Concept: The Persistent Handle

`RestorableRouteFuture` doesn't push the route itself. Instead, it works together with the *restorable* methods on `NavigatorState`, specifically:

*   `NavigatorState.restorablePush`
*   `NavigatorState.restorablePushNamed`
*   `NavigatorState.restorablePushReplacement`
*   `NavigatorState.restorablePushReplacementNamed`
*   `NavigatorState.restorablePopAndPushNamed`
*   `NavigatorState.restorablePushAndRemoveUntil`
*   `NavigatorState.restorablePushNamedAndRemoveUntil`
*   `NavigatorState.restorableReplace`
*   `NavigatorState.restorableReplaceRouteBelow`

These methods, unlike their non-restorable counterparts (`push`, `pushNamed`, etc.), return an opaque `String` ID instead of a `Future`. This ID is the key to state restoration. The [Navigator](01_navigator_.md) can use this ID to later find the corresponding route if it's restored.

This ID is what `RestorableRouteFuture` stores during state restoration.

The `RestorableRouteFuture` helps orchestrate this process:

1.  You register a `RestorableRouteFuture` as part of your state object that needs to push a restorable route and receive its result.
2.  When you want to push the route, you call `myRestorableRouteFuture.present(arguments)`.
3.  Behind the scenes, `present` calls a callback (`onPresent`) that *you* provide. This `onPresent` callback is responsible for calling one of the `NavigatorState.restorable...` methods and returning the `String` ID it gives back.
4.  `RestorableRouteFuture` stores this ID.
5.  The restorable Navigator method pushes or manages the route and returns the ID.
6.  If the app is interrupted and restored, the `RestorableRouteFuture` (being a restorable property) restores the stored ID.
7.  When your widget (that owns the `RestorableRouteFuture`) is rebuilt, the `RestorableRouteFuture` uses the restored ID to find the corresponding route within the restored [Navigator](01_navigator_.md).
8.  It then hooks onto that route's `popped` Future.
9.  When the route eventually completes (e.g., the dialog is dismissed), the `RestorableRouteFuture`'s `onComplete` callback (which *you* provided) is triggered with the route's result.

## Use Case: Pushing a Restorable Dialog and Getting a Result

Let's walk through the core use case: showing a dialog that needs to return a value and ensuring this works correctly with state restoration.

First, we need a simple dialog route that returns a value (e.g., a boolean). We'll use a simple `showDialog` for this, as `showDialog` internally creates a `DialogRoute`, which is restorable.

```dart
// A simple dialog that returns a boolean value
Future<bool?> showConfirmationDialog(BuildContext context) {
  return showDialog<bool>( // showDialog returns a Future<T>
    context: context,
    builder: (BuildContext context) {
      return AlertDialog(
        title: const Text('Confirm?'),
        actions: <Widget>[
          TextButton(
            child: const Text('Cancel'),
            onPressed: () {
              Navigator.pop(context, false); // Pop with result false
            },
          ),
          TextButton(
            child: const Text('OK'),
            onPressed: () {
              Navigator.pop(context, true); // Pop with result true
            },
          ),
        ],
      );
    },
  );
}
```

Next, we need a widget that will push this dialog and process its result, and this widget needs to support state restoration itself. This requires extending `StatefulWidget` and using `RestorationMixin`. The state object will hold the `RestorableRouteFuture`.

```dart
import 'package:flutter/material.dart';
import 'confirmation_dialog.dart'; // Contains showConfirmationDialog

class MainScreen extends StatefulWidget {
  const MainScreen({super.key});

  @override
  State<MainScreen> createState() => _MainScreenState();
}

// Our state object with RestorationMixin
class _MainScreenState extends State<MainScreen> with RestorationMixin {
  // Declare our RestorableRouteFuture property
  // <bool?> indicates it's tracking a Future<bool?>
  late final RestorableRouteFuture<bool?> _dialogRouteFuture;

  String _lastDialogResult = 'No dialog shown yet.';

  @override
  String? get restorationId => 'mainScreen'; // Provide a restoration ID for the state

  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {
    // Initialize the RestorableRouteFuture during state restoration
    _dialogRouteFuture = RestorableRouteFuture<bool?>(
      // navigatorFinder: (context) => Navigator.of(context), // Default, can be omitted
      onPresent: (navigator, arguments) {
        // This callback is called by _dialogRouteFuture.present()

        // We call the restorable dialog function here
        // If using a custom restorable route, you'd call
        // navigator.restorablePush() or navigator.restorablePushNamed()

        // showDialog internally uses a restorable DialogRoute, so we just call it
        final Future<bool?> dialogFuture = showConfirmationDialog(context);

        // We need to find the DialogRoute itself to get its restoration ID.
        // This is a bit indirect because showDialog hides the route creation.
        // A common pattern is to find the route by type or name after the push.
        // Since showDialog adds a ModalRoute, we can try finding the top ModalRoute.
        // NOTE: This might need adjustment for more complex scenarios with nested routes.
        final ModalRoute<bool?>? pushedRoute = ModalRoute.of<bool?>(context);

        // The restorable methods on NavigatorState return the ID directly.
        // If showDialog didn't hide the route like this, onPresent
        // would look more like this:
        //
        // final String id = navigator.restorablePush<bool?>(myStaticRouteBuilder, arguments: arguments);
        // return id;

        // For showDialog, we need its internal restoration ID IF it has one.
        // DialogRoute is restorable and sets restorationId if Navigator allows.
        final String? routeId = pushedRoute?.restorationScopeId.value;

        if (routeId == null) {
           // If restoration is not active or route is not restorable,
           // we can't get an ID. Handle gracefully, maybe just link future directly.
           // For this example, we'll assert or return empty ID if not getting one.
           assert(false, 'DialogRoute is not restorable or Navigator is not configured for restoration.');
           // If your route builder was truly static and used navigator.restorablePush,
           // you would have the id here already.
        }

        // showDialog returns the future, but present expects the route ID.
        // This highlights that RestorableRouteFuture is often best used
        // with the *restorable* Navigator methods that return IDs.
        // Since showDialog is convenient, and DialogRoute is restorable,
        // we can cheat a bit if the route is findable and has an ID.

        // Correct way with restorablePush if you had a static dialog route builder:
        /*
        @pragma('vm:entry-point')
        static Route<bool?> _myConfirmationDialogRoute(BuildContext context, Object? arguments) {
           return DialogRoute<bool?>(
             context: context,
             builder: (context) => AlertDialog(...),
             settings: arguments as RouteSettings? // Example of using arguments
           );
        }
        // Inside onPresent:
        final String id = navigator.restorablePush<bool?>(_myConfirmationDialogRoute, arguments: null);
        return id;
        */

        // Back to showDialog example - we need the ID from the route it made.
        // This is less robust than using the restorable methods that return IDs.
        // Let's assume for example purposes that showDialog ensures the route
        // gets an ID if the Navigator is restorabe AND we can find it reliably.
        // The ModalRoute.of(context) trick works best if the Navigator *just* pushed it.
        if (pushedRoute != null && pushedRoute.restorationScopeId.value != null) {
             return pushedRoute.restorationScopeId.value!;
        }

        // Fallback/Error if not getting an ID - the RestorableRouteFuture
        // won't be able to reconnect after restoration in this case.
        print('Warning: Could not get a restorable ID for the dialog route.');
        return ''; // Indicate no restorable ID
      },
      onComplete: (bool? result) {
        // This callback is called when the dialog is popped
        setState(() {
          _lastDialogResult = 'Dialog completed with: ${result ?? 'null'}';
        });
      },
    );

    // Register the RestorableRouteFuture property with the RestorationMixin
    registerForRestoration(_dialogRouteFuture, 'dialog_future');
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Restorable Dialog Example')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Text(_lastDialogResult),
            const SizedBox(height: 20),
            ElevatedButton(
              onPressed: () {
                // Check if the dialog is already on screen (restored or newly pushed)
                if (!_dialogRouteFuture.isPresent) {
                   // Push the restorable dialog using the RestorableRouteFuture
                   _dialogRouteFuture.present(); // Optionally pass arguments here
                } else {
                   print('Dialog is already present.');
                }
              },
              child: const Text('Show Restorable Dialog'),
            ),
          ],
        ),
      ),
    );
  }
}
```

To make this runnable and test restoration, wrap your `MainScreen` in a `MaterialApp` or `WidgetsApp` that has `useInheritedMediaQuery: true` (for dialogs) and sets a `restorationScopeId`.

```dart
import 'package:flutter/material.dart';
import 'main_screen.dart'; // Contains MainScreen

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      restorationScopeId: 'app', // Important: Enable restoration for the app
      useInheritedMediaQuery: true, // Usually needed for dialogs
      home: const MainScreen(),
    );
  }
}
```

To test state restoration:

1.  Run the app.
2.  Tap the "Show Restorable Dialog" button.
3.  From the dialog, either tap OK/Cancel OR send the app to the background (e.g., Home button on Android, swipe up on iOS) and wait for the system to potentially kill the app process.
4.  (On Android, typically use "Don't keep activities" developer option for reliable process killing).
5.  Reopen the app using the multitasking switcher.
6.  If restoration occurred, the dialog should reappear. Dismiss it using OK/Cancel.
7.  The text now showing `_lastDialogResult` should update based on the selected value.

If you had used `await Navigator.push(context, ...)` directly without `RestorableRouteFuture`, and the app process was killed while the dialog was open, when the app is restored and the `MainScreen` is rebuilt, there would be no active Future to `await`, and your `onComplete` logic might never run correctly. `RestorableRouteFuture` bridges this gap by finding the restored route and re-attaching the completion logic.

## RestorableRouteFuture Under the Hood (Simplified)

`RestorableRouteFuture<T>` is a subclass of `RestorableProperty<String?>`. This means it's designed to be registered with a `RestorationMixin` and it saves/restores a nullable `String`. This `String` is the opaque ID returned by the `NavigatorState.restorable...` methods.

```dart
// Simplified structure
class RestorableRouteFuture<T> extends RestorableProperty<String?> {
   final NavigatorFinderCallback navigatorFinder;
   final RoutePresentationCallback onPresent;
   final RouteCompletionCallback<T>? onComplete;

   String? _restoredRouteId; // The ID saved/restored

   Route<T>? _currentRoute; // The actual Route object found via the ID

   // Called by RestorationMixin
   @override
   String? createDefaultValue() => null;

   @override
   void initWithValue(String? value) {
       if (value != null) {
          _restoredRouteId = value;
          _hookOntoRouteFuture(_restoredRouteId!); // Try to find the route
       }
   }

   @override
   Object? toPrimitives() {
       // Save the ID if the route is currently present and restorable
       return _currentRoute?.restorationScopeId.value;
   }

   @override
   String fromPrimitives(Object? data) {
       // Restore the ID
       return data! as String;
   }

   void present([Object? arguments]) {
       assert(!isPresent); // Must not push if already tracking a route
       assert(isRegistered); // Must be registered

       final NavigatorState navigator = navigatorFinder(state.context);

       // Calls the user-provided callback to push the route and get its ID
       final String routeId = onPresent(navigator, arguments);

       _hookOntoRouteFuture(routeId); // Store the ID and hook onto the future
       notifyListeners(); // Notify the property has changed (saved the ID)
   }

   // Hook onto the route's popped future
   void _hookOntoRouteFuture(String id) {
       // Find the route in the navigator using the ID
       _currentRoute = _navigator._getRouteById<T>(id);
       assert(_currentRoute != null); // Must find the route!

       // Add listener to notify if restoration ID changes (e.g., route becomes non-restorable)
       _currentRoute!.restorationScopeId.addListener(notifyListeners);

       // Listen for the route completion (when it's popped)
       _currentRoute!.popped.then((dynamic result) {
           if (_disposed) return; // Avoid errors if the property is disposed

           // Clean up listeners and references
           _currentRoute?.restorationScopeId.removeListener(notifyListeners);
           _currentRoute = null;
           notifyListeners(); // Notify that the route is no longer present

           // Call the user-provided completion callback
           onComplete?.call(result as T);
       });
   }

   bool get isPresent => _currentRoute != null;

   Route<T>? get route => _currentRoute;

}
```

```mermaid
sequenceDiagram
    participant AppState as YourState (with RestorationMixin)
    participant RestorableRouteFuture
    participant User
    participant MainScreenWidget
    participant ElevatedButton
    participant NavigatorState
    participant RestorableRoute (e.g., DialogRoute)
    participant RestorableProperty

    alt Initial Push (No Restoration)
        User->>MainScreenWidget: App starts
        MainScreenWidget->>AppState: Build
        AppState->>RestorableRouteFuture: Register as RestorableProperty
        ElevatedButton->>AppState: Tap "Show Dialog"
        AppState->>RestorableRouteFuture: present()
        RestorableRouteFuture->>AppState: navigatorFinder(context)
        AppState->>NavigatorState: Get NavigatorState
        RestorableRouteFuture->>AppState: onPresent(NavigatorState, arguments)
        AppState->>NavigatorState: Call restorablePush/showDialog (creates RestorableRoute)
        NavigatorState->>RestorableRoute: Create Route
        RestorableRoute->>NavigatorState: Get Restoration ID from self
        NavigatorState->>AppState: Return Restoration ID ("dialog-id-123")
        AppState->>RestorableRouteFuture: Returns "dialog-id-123" to present()
        RestorableRouteFuture->>RestorableRouteFuture: Store ID ("dialog-id-123")
        RestorableRouteFuture->>RestorableRouteFuture: _hookOntoRouteFuture("dialog-id-123")
        RestorableRouteFuture->>NavigatorState: _getRouteById("dialog-id-123")
        NavigatorState-->>RestorableRouteFuture: Return RestorableRoute object
        RestorableRouteFuture->>RestorableRouteFuture: Store RestorableRoute object (_currentRoute)
        RestorableRouteFuture->>RestorableProperty: notifyListeners() (Saves ID)
        NavigatorState->>Overlay: Show RestorableRoute content
    end

    alt App Interrupted & Restored
        User->>OperatingSystem: Background/Close App
        OperatingSystem->>Restoration: Save state (includes Navigator history and RestorableProperty IDs)
        User->>OperatingSystem: Reopen App
        OperatingSystem->>Restoration: Restore state
        Restoration->>AppState: Restore RestorableProperty (RestorableRouteFuture)
        RestorableProperty->>RestorableRouteFuture: initWithValue("dialog-id-123")
        RestorableRouteFuture->>RestorableRouteFuture: _hookOntoRouteFuture("dialog-id-123")
        RestorableRouteFuture->>NavigatorState: _getRouteById("dialog-id-123")
        NavigatorState-->>RestorableRouteFuture: Return *restored* RestorableRoute object
        RestorableRouteFuture->>RestorableRouteFuture: Store RestorableRoute object (_currentRoute)
        RestorableRouteFuture->>RestorableProperty: notifyListeners()
        NavigatorState->>Overlay: Rebuild restored RestorableRoute content
    end

    alt Dialog Completed (After Initial Push or Restoration)
        User->>RestorableRoute: Interact and tap OK/Cancel
        RestorableRoute->>NavigatorState: Navigator.pop(result)
        NavigatorState->>RestorableRoute: didPop(result)
        RestorableRoute->>RestorableRoute: Completes Future<T>
        RestorableRouteFuture->>RestorableRoute: Future completes successfully
        RestorableRouteFuture->>RestorableRouteFuture: _currentRoute.restorationScopeId -> removeListener
        RestorableRouteFuture->>RestorableRouteFuture: _currentRoute = null
        RestorableRouteFuture->>RestorableProperty: notifyListeners() (Clears saved ID)
        RestorableRouteFuture->>AppState: onComplete(result)
        AppState->>AppState: Update UI with result
    end
```

This diagram shows that `RestorableRouteFuture`'s job is to store the restorable ID when `present()` is called and, crucially, use that stored ID to find the corresponding *restored* route after the app restarts. It then hooks onto that restored route's completion `Future` so that the `onComplete` callback fires correctly.

## Conclusion

`RestorableRouteFuture` is a specialized helper class for managing navigation in Flutter apps that utilize state restoration. It's designed to work with the `NavigatorState.restorable...` methods to provide a persistent handle to routes pushed imperatively. This ensures that you can reliably receive the result of a route (like a dialog) even if the app process is killed and restored while that route is active. By registering a `RestorableRouteFuture` as a restorable property in your state, you enable your widget to correctly process results from routes pushed with state restoration in mind, bridging the gap between imperative navigation calls and the declarative nature of state restoration.

This concludes our exploration of Flutter's navigation concepts! You've now encountered the major pieces of the navigation puzzle, from basic stack management to advanced techniques for state restoration and customizable transitions. Happy navigating!

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)