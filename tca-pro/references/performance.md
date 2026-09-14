# Performance

Four pitfalls account for nearly all performance problems in TCA apps.

## Contents

- Sharing logic with actions
- Store scoping over computed properties
- High-frequency actions
- CPU-intensive work in a reducer

## Sharing logic with actions

Sending an action is not a method call. An action travels through every layer of the app, and each layer's reducer can intercept it. Using a dedicated action to share logic between cases costs an extra full pass through the system for every user interaction.

```swift
// Avoid
case .buttonTapped:
  state.count += 1
  return .send(.sharedComputation)

case .toggleChanged:
  state.isEnabled.toggle()
  return .send(.sharedComputation)

case .sharedComputation:
  return .run { send in /* shared work */ }
```

Beyond the cost, this is inflexible — the shared logic can only run *after* the core logic — and it bloats tests, which must now assert on an internal action the user never triggered.

```swift
// Prefer
case .buttonTapped:
  state.count += 1
  return self.sharedComputation(state: &state)

case .toggleChanged:
  state.isEnabled.toggle()
  return self.sharedComputation(state: &state)

// ...

func sharedComputation(state: inout State) -> Effect<Action> {
  // shared work
  return .run { send in /* shared effect */ }
}
```

A method has full access to dependencies, can mutate `inout State`, and can return an `Effect`. It can also be called *before* the core logic when that is what you need:

```swift
case .buttonTapped:
  let sharedEffect = self.sharedComputation(state: &state)
  state.count += 1
  return sharedEffect
```

The same applies to a parent invoking logic in a child: `.send(.child(.refresh))` works, but extracting a helper both domains call is cheaper.

## Store scoping over computed properties

Since 1.5, a scoped store holds a reference to the root store and derives its state on the fly rather than caching it. Scoping along a **stored** property is a plain getter and costs nothing:

```swift
ChildView(store: store.scope(\.child, action: \.child))
```

Scoping along a **computed** property re-runs that computation every time state is read from the scoped store — potentially many times per render, and worse the closer the scope is to the root.

```swift
// Avoid
extension ParentFeature.State {
  var computedChild: ChildFeature.State {
    ChildFeature.State(/* heavy computation */)
  }
}

ChildView(store: store.scope(\.computedChild, action: \.child))
```

Scope only along stored properties of child features. Push derived values toward the leaves — compute them in the child view, or store the result in state when it is genuinely part of the domain. If you suspect this is happening, put a `print` in the computed property and count the calls.

## High-frequency actions

Only send actions that represent something significant. Dozens of actions per second is almost always a design problem.

**Reporting progress:** do not send an action per step.

```swift
// Avoid: 100,000 actions to move a Double from 0 to 1
for await event in self.eventsClient.events() {
  await send(.progress(Double(count) / Double(max)))
  count += 1
}

// Prefer: at most 100 actions
let interval = max / 100
for await event in self.eventsClient.events() {
  defer { count += 1 }
  if count.isMultiple(of: interval) {
    await send(.progress(Double(count) / Double(max)))
  }
}
```

**Sliders and continuous gestures:** a binding straight to the store sends an action per pixel of drag. Hold the value locally and send once:

```swift
@State private var opacity = 1.0

Slider(value: $opacity, in: 0...1) { isEditing in
  if !isEditing { store.send(.setOpacity(opacity)) }
}
```

Do this only when profiling shows the direct binding is a problem — local `@State` duplicates the source of truth, which has its own cost.

## CPU-intensive work in a reducer

Reducers run on the main thread. Move heavy computation into an effect, which runs on the cooperative thread pool, and yield periodically so you do not monopolize a pool thread:

```swift
case .buttonTapped:
  return .run { send in
    var result = Result()
    for (index, value) in someLargeCollection.enumerated() {
      // intense computation
      if index.isMultiple(of: 1_000) {
        await Task.yield()
      }
    }
    await send(.computationResponse(result))
  }

case let .computationResponse(result):
  state.result = result
  return .none
```

## Reentrancy

Sending an action while another action is being processed emits a runtime warning and is undefined behavior; a future version will make it a precondition failure. If you see that warning, you are sending synchronously from inside action-processing code — restructure so the second action goes through an effect instead.
