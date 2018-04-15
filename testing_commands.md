Testing Elm Commands
====================

When programming in Elm, most of our application is described in terms of pure
functions. Since we don't need to mock dependencies to test these functions, it
is quite easy to test these areas. 

At the border of our program, though, we have I/O actions. Inputs are managed by
the Elm architecture through the use of an update function: 

```
update : Message -> Model -> (Model, Cmd Message)
```

The update function accepts any external input having type Message, and manages
its impact on the model. It also output `Cmd Message` values: these are the
effects of our program. 

The `Cmd Message` type indicates that the Elm runtime will execute the Cmd we
pass to it, and that this Cmd will create inputs that respect the `Message`
type. This way, the Elm runtime is guaranteed that it can feed the Message
produced by this command's effect back into the update function.

But let's come back to testing these side-effects.

## Testing Messages (external Inputs)

It is very easy to test the effects of Messages on your model in Elm: since the
update function is pure, you can simply call it with whatever message you expect
to receive and check your model:

```
type Message = GetLetter City String 

update (GetLetter city content), content
    case city of
        Paris -> ({ model | paris = content :: model.paris}, Cmd.none)
        _ -> (model, Cmd.none)

test "" \_ ->
    let (newModel, _) = update (GetLetter Paris "hello") defaultModel
    in  Expect.Equal newModel.paris.mailbox ["hello"]
```

## Testing Commands (program outputs)

When it comes to commands, however, the task is harder. Let's look again at the
previous example:

```
type Message = GetLetter City String 

update (GetLetter city content), content
    case city of
        London -> ({ model | london = content :: model.london}, orderBeer)
        _ -> (model, Cmd.none)

test "" \_ ->
    let (_, command) = update (GetLetter London "hello") defaultModel
    in  Expect.Equal command orderBeer
```

This test has no way to check the command beyond equality. The Cmd x type is
opaque and does not let us extract values to check.

Let's dive in the [internals of Elm's Cmd](https://github.com/elm-lang/core/blob/master/src/Platform/Cmd.elm): 

```
type Cmd msg = Cmd
```

Not much to see here, Cmd offers nothing interesting for us. 

```
batch : List (Cmd msg) -> Cmd msg
batch =
  Elm.Kernel.Platform.batch
```

The batch function is straight out of `Elm.Kernel.Platform`, which is
implemented in JS...

It seems testing commands out of the box is hard in Elm.

## Testable

An existing solution to this problem is using Elm Testable [1].
Elm Testable lets us use alternative libs that provide inspectability. With
these, we can check command content: 

```
-import Http
+import Testable.Cmd
+import Testable.Http as Http
```

Testable wraps your application and lets you execute updates. Testable then
offers an API to verify whether specific commands have been issued.

```
|> assertCalled (Spelling.check "cats")
```

Con is that you have to wrap all your application in the framework to test it.

## A less invasive solution

Another way of testing is to ban command usage in the udpate function [1]. 

For instance: 

```
type MyEffect
    = None
    | GetTime
    | FetchRessource Id

update : Message -> Model -> ( Model, MyEffect )
```

Once we stop using Cmd, it is very easy to test the effects generated inside our
application. We switch from a model where our update issues commands, to a model
where our update describes the output side-effects of our app, in a language
optimized for the business description of these side effects: 

```
test "when someone wants their package, fetch it in the store" \_ ->
    let (_, command) = update (RetrievePackage Paris) defaultModel
    in  Expect.Equal command (FetchRessource <| Id 75001)
```

We are, however, not able to give this function directly to Html.App.program. We
need to create a function to transform our MyEffect into Cmd messages:

```
convertEffects : ( Model, MyEffect ) -> ( Model, Cmd Message )
convertEffects ( m, effect ) =
    case effect of
        GetTime _ ->
            let effect = Task.perform TimeIs Time.now
            in  ( m, effect)

        None ->
            ( m, Cmd.none )

externalUpdate : Message -> Model -> ( Model, Cmd Message )
externalUpdate msg model = convertEffects <| update msg model
```

For this subset of the code, we have the same testing problems. However, the
perimeter of this code is much smaller, and we have the added benefit of
describing our effects in terms of our domain language, with an adapter to Elm's
API. If we then decide to use testable, the dependency perimeter is much narrower.

## Notes

[1] [Elm Testable](http://package.elm-lang.org/packages/rogeriochaves/elm-testable/4.1.1).

[2] [small working example](https://github.com/dojo-developpement-paris/dojo-developpement-paris.github.io/blob/2018-02-23/tests/Spec.elm#L66)
