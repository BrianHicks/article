Away from Components
====================

### How we went away from an application split into component to focus on our affordances and functionalities

## In the beginning

When we started working on our Elm application (a planning viewer customized for
our field of work), we mostly thought about the application in terms of
components : a grid, a datepicker, filtering tools, and an item tool side panel.

Since our code was a few klocs, it made sense to split it into a few modules,
and the modules naturally followed our line of thinking: one module for the
grid, one module for the toolbar with submodules for the datepicker, filter and
a few other things.

Our data model was also split along these lines:

```
type alias App = { user : Person
                 , toolbar : Toolbar
                 , grid : Grid
                 }
```

## The problem

We rapidly encountered a few problems, such as:

* how should we apply filter parameters on the grid ?
* Since our toolbar update function doesn't see the grid, how do we pass filter
  criteria or the date picked ?
* How do I keep on the Toolbar datepicker pickedDate and the Grid currentDate 
  synchronized ?

Our solution to these initial problems were heterogenous and made our code less
normalized:

* We abused messaging to pass pure model transformations from one component to
  another
* In one case, the update function returned not only `(model, Cmd msg)`, but
  also additional info for the parent update function to perform further
  actions.

As our code base grew, our messages didn't mean inbound side-effect and our
commands didn't mean outbound side-effect anymore [1]. We were building an app
of objects communicating with each other through the messaging system.

Elm is an opinionated language, and questions such has "how do I transform a msg
into a Cmd msg" are heavily discouraged in the litterature. We could probably
have proceeded this way, but this would have resulted in much pain as our
metaphorical toes would have knocked against every corner of the language.

## Our solution

The first step we took was, for one component, to get rid of messages intended
to update the model without side effects, and messages intended to be run as
command proxies. 

This change left us short of tools to update the model from separate components.
Rather than question our choice, we decided to question our data segregation,
and went on to carry the whole model in every specific update:

The Elm records being what they are, we promptly wrote helpers such as: 

```
onGrid : (Grid -> Grid) -> App -> App


model |> onGrid filter
```

By exporting model transformations from the grid, we could now apply changes 
on the grid directly in the toolbar update function. 

## Mixing commands and pure model transformations

While we were at it, we thought about updating our functions in order to avoid 
mixing pure transformations and Cmd emissions:

```
action : App -> (App, Cmd msg)
```

or 

```
transformApp : App -> App

action :App -> Cmd msg
```

In the end, we decided it was one step too far. The `action` signature given above 
looks very much like a IO monadic action in Haskell: `action :: a -> IO a`. We 
figured it would probably better to accept the mix of pure data transformation and 
outbound IO that is core to our domain, and use something equivalent to Haskell's 
bind operator (`IO a -> (a -> IO b) -> IO b`):

```
bind : (App -> (App, Cmd msg))-> (App, Cmd msg) -> (App, Cmd msg)
```

## Enter Don Norman

Removing much of the boundaries between components left us wondering whether the 
component approach was still relevant: why have a Toolbar component when you 
actually mostly use grid actions in that module?

Don Norman wrote an influential design book [2] where he exposes the concept of 
affordance: an affordance is an interaction that the object you design offers. 
Rather than modularizing components, we can modularize our application's affordances.

In our particular case, this means that rather than having Toolbar, Grid and Calendar,
we should have Filter, GetData and Navigate. While Grid used to be our central 
module, it completely disappeared, because in the end, a grid is merely the data 
we shape, not an action relevant to our users' workflow.

Because we work with graphic material, the things we see in the browser and the 
components we design in the view have a strong pull when it comes to shaping the 
way we think about our applications. But having a strong logical coupling between 
our view and our application logic is not a fatality. When the domain description 
clashes with the view logic, we should privilege the former.

## Conclusion

We now have an application where it is easier to locate a specific behaviour code. 
Building the view feels a bit alien in that organization, but it's not much of a
problem, because when thinking about the view, the actual product documents the
way the view is organized.

##### Notes

[1] The relation between Cmd/msgs and I/O side-effects is a key aspect of Elm
design. It may be hard for newcomers to understand that aspect of the functional
paradigm, but messages not modeling side-effect inputs or commands not modelling
side-effect outputs are a fundamental design flaw.

[2] The Design of Everyday Things, ISBN: 0465050654
