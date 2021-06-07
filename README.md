# go_router
The goal of the go_router package is to simplify use of
[the `Router` in Flutter](https://api.flutter.dev/flutter/widgets/Router-class.html)
as specified by [the
`MaterialApp.router` constructor](https://api.flutter.dev/flutter/material/MaterialApp/MaterialApp.router.html).
By default, it requires an implementation of the
[`RouterDelegate`](https://api.flutter.dev/flutter/widgets/RouterDelegate-class.html) and
[`RouteInformationParser`](https://api.flutter.dev/flutter/widgets/RouteInformationParser-class.html)
classes. These two implementations themselves imply the
definition of a custom type to hold the app state that drives the creation of the
[`Navigator`](https://api.flutter.dev/flutter/widgets/Navigator-class.html).
You can read [an excellent blog post on these requirements on Medium](https://medium.com/flutter/learning-flutters-new-navigation-and-routing-system-7c9068155ade).
This separation of responsibilities allows the Flutter developer to implement a number of routing
and navigation policies at the cost of [complexity](https://www.reddit.com/r/FlutterDev/comments/koxx4w/why_navigator_20_sucks/).

The go_router makes three simplifying assumptions to reduce complexity:
- all routing in the app will happen via URI-compliant names, e.g. `/family/f1/person/p2`
- an entire stack of pages can be constructed from the route name alone
- the concept of "back" in your app is "up" the stack of pages each route name
  produces

These assumptions allow go_router to provide a simpler implementation of your app's custom router.

# Getting Started
To use the go_router package, add go_router to your `pubspec.yaml`:

```yaml
...
dependencies:
  ...
  go_router: ^CURRENT-VERSION
```

To use go_router, add the import to your Dart file:

```dart
import 'package:go_router/go_router.dart';
```

With these two pieces in place, you're ready to create your routes.

# Declarative Routing
The go_router is governed by a set of routes which you specify via a builder
function:

```dart
class App extends StatelessWidget {
  List<GoRoute> _builder(BuildContext context, String location) => [
        GoRoute(
          pattern: '/',
          builder: (context, args) => MaterialPage<FamiliesPage>(
            key: const ValueKey('FamiliesPage'),
            child: FamiliesPage(families: Families.data),
          ),
        ),
        GoRoute(
          pattern: '/family/:fid',
          builder: (context, args) {
            final family = Families.family(args['fid']!);

            return MaterialPage<FamilyPage>(
              key: ValueKey(family),
              child: FamilyPage(family: family),
            );
          },
        ),
        GoRoute(
          pattern: '/family/:fid/person/:pid',
          builder: (context, args) {
            final family = Families.family(args['fid']!);
            final person = family.person(args['pid']!);

            return MaterialPage<PersonPage>(
              key: ValueKey(person),
              child: PersonPage(family: family, person: person),
            );
          },
        ),
      ];
  ...
}
```

In this case, we've defined three routes w/ the following patterns:

- `/`: the home page that shows a list of familes
- `/family/:fid`: a family details page identified by the `fid` variable and
  parsed at run-time for a URI like this: `/family/f1`
- `/family/:fid/person/:person`: a person details page,
  e.g. `/family/f1/person/p2`

These routes will be matched in order and every pattern that matches the location
will be a page on the navigation stack. Each location is able to produce the
entire stack, like so:

location               | navigation stack
-----------------------|-----------------
`/`                    | `FamiliesPage()`
`/family/f1`           | `FamiliesPage()`, `FamilyPage(f1)`
`/family/f1/person/p2` | `FamiliesPage()`, `FamilyPage(f1)`, `PersonPage(p2)`

The order of the patterns in the list of routes dictates the order in the
navigation stack. The navigation stack is used to pop up to the previous page in
the stack when the user press the Back button or your app calls
`Navigation.pop()`.

In addition to the pattern, a `GoRoute` contains a page builder function which
is called to create the page when a pattern is matched. That function can use
the arguments parsed from the pattern to do things like look up data to use
to initialize each page.

In addition, the go_router needs an error handler in case no page is found or
if any of the page builder functions throws an exception, e.g.

```dart
```

With these two functions in hand, you can establish your app's custom routing
policy using the `MaterialApp.router` constructor:

TODO:STARTHERE

# Navigation
You can navigate between pages in your app using the `GoRouter.go` method:

```dart
// navigate using the GoRouter
onTap: () => GoRouter.of(context).go('/family/f1/person/p2')
```

The go_router also provides a simplified version using Dart extension methods:

```dart
// simplified navigate using the GoRouter
onTap: () => context.go('/family/f1/person/p2')
```

The simplified version maps directly to the more fully-specified version, so you can use either.


The route name patterns are defined and implemented in the [`path_to_regexp`](https://pub.dev/packages/path_to_regexp)
package, which gives you the ability to match regular expressions, e.g. `/family/:fid(f\d+)`.

The route builder is called each time that the location changes, in case you'd like to produce the set of declarative
routes based on the current location. It will also be called when data associated with the context changes as described
in the [Conditional Routes](#conditional-routes) section below.

# MaterialApp.router Usage
To configure a `Router` instance for use in your Flutter app, you use the `Material.router` constructor,
passing in the implementation of the `RouterDelegate` and `RouteInformationParser` classes provided by the
`GoRouter` object:

```dart
class App extends StatelessWidget {
  @override
  Widget build(BuildContext context) => MaterialApp.router(
        routeInformationParser: _router.routeInformationParser,
        routerDelegate: _router.routerDelegate,
        title: 'GoRouter Example',
      );

  late final _router = GoRouter(...);
  ...
}

```

In this way, go_router can implement the picky custom routing protocol whereas all you have to do
is implement the builder or, even simpler, provide a set of declarative routes.

# URL Path Strategy
By default, Flutter adds a hash (#) into the URL for web apps:

![URL Strategy w/ Hash](readme/url-strat-hash.png)

If you'd like to turn this off using pure Dart and Flutter, you can, but
[this process is a bit picky](https://flutter.dev/docs/development/ui/navigation/url-strategies), too.
The go_router has built-in support for setting the URL path strategy, however, so you can simply call
`GoRouter.setUrlPathStrategy` and make your choice:

```dart
void main() {
  // turn on the # in the URLs on the web (default)
  // GoRouter.setUrlPathStrategy(UrlPathStrategy.hash);

  // turn off the # in the URLs on the web
  GoRouter.setUrlPathStrategy(UrlPathStrategy.path);

  runApp(App());
}
```

Setting the path instead of hash strategy turns off the # in the URLs as expected:

![URL Strategy w/o Hash](readme/url-strat-no-hash.png)

Finally, when you deploy your Flutter web app to a web server, it needs to be configured such that every URL ends up
at index.html, otherwise the URL path strategy won't be able to route to your pages. If you're using Firebase
hosting, [configuring your app as a "single page app" will cause all URLs to be rewritten to index.html](https://firebase.google.com/docs/hosting/full-config#rewrites).

If you'd like to test locally before publishing, you can use [live-server](https://www.npmjs.com/package/live-server):

```sh
$ live-server --entry-file=index.html build/web
```

# Conditional Routes
If you'd like to change the set of routes based on conditional app state, e.g. whether a user is logged in or not,
you can do so using `InheritedWidget` or one of it's wrappers, e.g.
[the provider package](https://pub.dev/packages/provider), to watch for changing state inside of the route builder.
For example, imagine a simple class to track the app's current logged in state:

```dart
class LoginInfo extends ChangeNotifier {
  var _userName = '';
  bool get loggedIn => _userName.isNotEmpty;

  void login(String userName) {
    _userName = userName;
    notifyListeners();
  }
}
```

Because the `LoginInfo` is a `ChangeNotifier`, it can accept listeners and notify them of data changes via the use of
the `notifyListeners` method. We can then use the provider package to drop the login info into the widget tree like so:

```dart
class App extends StatelessWidget {
  // add the login info into the tree as app state that can change over time
  @override
  Widget build(BuildContext context) => ChangeNotifierProvider<LoginInfo>(
        create: (context) => LoginInfo(),
        child: MaterialApp.router(
          routeInformationParser: _router.routeInformationParser,
          routerDelegate: _router.routerDelegate,
          title: 'Conditional Routes GoRouter Example',
        ),
      );
...
}
```

The use of `ChangeNotifyProvider` creates a new `LoginInfo` object and puts it into the widget tree. Now imagine a login
page that pulls the login info out of the widget tree and changes the login state as appropriate:

```dart
class LoginPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: Text(_title(context))),
        body: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              ElevatedButton(
                // log a user in, letting all the listeners know
                onPressed: () => context.read<LoginInfo>().login('user1'),
                child: const Text('Login'),
              ),
            ],
          ),
        ),
      );
...
}
```

Notice the use of `context.read` from the provider package to walk the widget tree to find the login info and login a
sample user. This causes the listeners to this data to be notified and for any widgets listening for this change to
rebuild. We can then use this data when implementing the `GoRouter` builder, either the raw builder that produces a
`Navigator` or the a declarative builder that returns the list of routes to search. We can
use the login info to decide which routes are allowed like so:

```dart
class App extends StatelessWidget {
  // the routes when the user is logged in
  final _loggedInRoutes = [
    GoRoute(
      pattern: '/',
      builder: (context, args) => MaterialPage<FamiliesPage>(...),
    ),
    GoRoute(
      pattern: '/family/:fid',
      builder: (context, args) => MaterialPage<FamilyPage>(...),
    ),
    GoRoute(
      pattern: '/family/:fid/person/:pid',
      builder: (context, args) => MaterialPage<PersonPage>(...),
    ),
  ];

  // the routes when the user is not logged in
  final _loggedOutRoutes = [
    GoRoute(
      pattern: '/',
      builder: (context, args) => MaterialPage<LoginPage>(...),
    ),
  ];

  late final _router = GoRouter.routes(
    // changes in the login info will rebuild the stack of routes
    builder: (context, location) => context.watch<LoginInfo>().loggedIn ? _loggedInRoutes : _loggedOutRoutes,
    ...
  );
}
```

Here we've defined two lists of routes, one for when the user is logged in and one for when they're not. Then, we use
`context.watch` to read the login info to determine which list of routes to return. And because we used `context.watch`
instead of `context.read`, whenever the login info object changes, the routes builder is automatically called for the
correct list of routes based on the current app state.

# Redirection
Sometimes you want to redirect one route to another one, e.g. if the user is not logged in. You can do that using
an instance of the GoRedirect object, e.g.

```dart
List<GoRoute> _builder(BuildContext context, String location) => [
    GoRoute(
      pattern: '/',
      builder: (context, args) {
        final loggedIn = context.watch<LoginInfo>().loggedIn;
        if (!loggedIn) return const GoRedirect('/login');

        return MaterialPage<FamiliesPage>(
          key: const ValueKey('FamiliesPage'),
          child: FamiliesPage(families: Families.data),
        );
      },
    ),
    ...
    GoRoute(
      pattern: '/login',
      builder: (context, args) {
        final loggedIn = context.watch<LoginInfo>().loggedIn;
        if (loggedIn) return const GoRedirect('/');

        return const MaterialPage<LoginPage>(
          key: ValueKey('LoginPage'),
          child: LoginPage(),
        );
      },
    ),
  ];
```

In this code, if the user is not logged in, we redirect from the `/` to `/login`. Likewise, if the user is logged in,
we redirect from `/login` to `/`.

# Query Parameters
If you'd like to use query parameters for navigation, you can and they will be considered as optional for the purpose
of matching a route but passed along as arguments to any builder. For example, if you'd like to redirect to `/login`
with the original location as a query parameter so that after a successful login, the user can be routed back to the
original location, you can do that using query paramaters:

```dart
List<GoRoute> _builder(BuildContext context, String location) => [
  ...
  GoRoute(
    pattern: '/family/:fid',
    builder: (context, args) {
      final loggedIn = context.watch<LoginInfo>().loggedIn;
      if (!loggedIn) return GoRedirect('/login?from=$location');

      final family = Families.family(args['fid']!);
      return MaterialPage<FamilyPage>(
        key: ValueKey(family),
        child: FamilyPage(family: family),
      );
    },
  ),
  ...
  GoRoute(
    pattern: '/login',
    builder: (context, args) {
      final loggedIn = context.watch<LoginInfo>().loggedIn;
      if (loggedIn) return const GoRedirect('/');

      return MaterialPage<LoginPage>(
        key: const ValueKey('LoginPage'),
        child: LoginPage(from: args['from']),
      );
    },
  ),
];
```

In this example, if the user isn't logged in, they're redirected to `/login` with a `from` query parameter set to the
original location. When the `/login` route is matched, the optional `from` parameter is passed to the `LoginPage`. In
the `LoginPage` if the `from` parameter was passed, we use it to go to the original location:

```dart
```

A query parameter will not override a positional parameter or another query parameter set earlier in the location
string.

# Custom Builder
To implement the mapping between a location and a stack of pags, the app can create an instance of the
`GoRouter` class, passing in a builder function to translate from a route name (aka location) into a
`Navigator` The `Navigator` includes a stack of pages and an implementation of `onPopPage`, which handles
calls to `Navigator.pop` and is called by the Flutter implementation of the Back button.

The following is an example implementation of the builder function that supports three pages:
- `FamiliesPage` at `/`
- `FamilyPage` as `/family/:fid`, e.g. `/family/f1`
- `PersonPage` as `/family/:fid/person/:pid`, e.g. `/family/f1/person/p2`

Furthermore, it creates a stack of pages for each route that's match, e.g. `/family/f1/person/p2` yields a stack of
three pages: `FamiliesPage`, `FamilyPage(fid:f1)` and `PersonPage(fid:f1, pid:p2)`.

```dart
class App extends StatelessWidget {
  ...
  late final _router = GoRouter(builder: _builder);
  Widget _builder(BuildContext context, String location) {
    final locPages = <String, Page<dynamic>>{};

    try {
      final segments = Uri.parse(location).pathSegments;

      // home page, i.e. '/'
      {
        const loc = '/';
        final page = MaterialPage<FamiliesPage>(
          key: const ValueKey('FamiliesPage'),
          child: FamiliesPage(families: Families.data),
        );
        locPages[loc] = page;
      }

      // family page, e.g. '/family/:fid
      if (segments.length >= 2 && segments[0] == 'family') {
        final fid = segments[1];
        final family = Families.family(fid);

        final loc = '/family/$fid';
        final page = MaterialPage<FamilyPage>(
          key: ValueKey(family),
          child: FamilyPage(family: family),
        );

        locPages[loc] = page;
      }

      // person page, e.g. '/family/:fid/person/:pid
      if (segments.length >= 4 && segments[0] == 'family' && segments[2] == 'person') {
        final fid = segments[1];
        final pid = segments[3];
        final family = Families.family(fid);
        final person = family.person(pid);

        final loc = '/family/$fid/person/$pid';
        final page = MaterialPage<PersonPage>(
          key: ValueKey(person),
          child: PersonPage(family: family, person: person),
        );

        locPages[loc] = page;
      }

      // if we haven't found any matching routes OR
      // if the last route doesn't match exactly, then we haven't got a valid stack of pages;
      // the latter allows '/' to match as part of a stack of pages but to fail on '/nonsense'
      if (locPages.isEmpty || locPages.keys.last.toString().toLowerCase() != location.toLowerCase()) {
        throw Exception('page not found: $location');
      }
    } on Exception catch (ex) {
      locPages.clear();

      final loc = location;
      final page = MaterialPage<Four04Page>(
        key: const ValueKey('ErrorPage'),
        child: Four04Page(message: ex.toString()),
      );

      locPages[loc] = page;
    }

    return Navigator(
      pages: locPages.values.toList(),
      onPopPage: (route, dynamic result) {
        if (!route.didPop(result)) return false;

        // remove the route for the page we're showing and go to the next location up
        locPages.remove(locPages.keys.last);
        _router.go(locPages.keys.last);

        return true;
      },
    );
  }
}
```

There's a lot going on here, but it fundamentally boils down to three things:
1. Matching portions of the location to instances of the app's pages using manually parsed URI segments for
   arguments. This mapping is kept in an ordered map so it can be used as a stack of location=>page mappings.
1. Providing an implementation of `onPopPage` that will translate `Navigation.pop` to use the
   location=>page mappings to navigate to the previous page on the stack.
1. Show an error page if any of that fails.

# Examples
You can see the go_router in action via the following examples:
- [`routes.dart`](example/lib/routes.dart): define a routing policy but using a set of declarative `GoRoute` objects
- [`url_strategy.dart`](example/lib/url_strategy.dart): turn off the # in the Flutter web URL
- [`conditional.dart`](example/lib/conditional.dart): provide different routes based on changing app state
- [`redirection.dart`](example/lib/redirection.dart): redirect one route to another based on changing app state
- [`query_params.dart`](example/lib/query_params.dart): optional query parameters will be passed to all page builders
- [`builder.dart`](example/lib/builder.dart): define routing policy by providing a custom builder

You can run these examples from the command line like so:

```sh
$ flutter run example/lib/routes.dart
```

Or, if you're using Visual Studio Code, a [`launch.json`](.vscode/launch.json) file has been provided with
these examples configured.

# TODO
- rearrange README and rename ctors to make the routes case the default
- update README for async id => object lookup
- nesting routing
- custom transition support
- supporting the concept of "back" as well as "up"
- support for shorter locations that result in multiple pages for a single route, e.g. /person/:pid
  could end up mapping to three pages (home, families and person) but will only match two routes
  (home and person). The mapping to person requires two pages to be returned (families and person).
- publish
- ...
- profit!
- BUG: navigating back too fast crashes
