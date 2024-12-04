# ``ComposableArchitecture/Reducer``

``Reducer`` 프로토콜은 주어진 액션에 따라 애플리케이션의 현재 상태를 다음 상태로 변화시키는 방법과, 
필요한 경우 스토어가 이후 실행해야 할 ``Effect``를 정의합니다. 이 프로토콜을 준수하는 타입은 특정 기능의 도메인, 로직, 동작을 표현합니다.

``Reducer``를 직접 구현할 수도 있지만, ``Reducer()`` 매크로를 사용하면 Reducer를 더 간결하고 강력하게 작성할 수 있습니다.

* [Conforming to the Reducer protocol](#Conforming-to-the-Reducer-protocol)
* [Using the @Reducer macro](#Using-the-Reducer-macro)
  * [@CasePathable and @dynamicMemberLookup enums](#CasePathable-and-dynamicMemberLookup-enums)
  * [Automatic fulfillment of reducer requirements](#Automatic-fulfillment-of-reducer-requirements)
  * [Destination and path reducers](#Destination-and-path-reducers)
    * [Navigating to non-reducer features](#Navigating-to-non-reducer-features)
    * [Synthesizing protocol conformances on State and Action](#Synthesizing-protocol-conformances-on-State-and-Action)
    * [Nested enum reducers](#Nested-enum-reducers)
  * [Gotchas](#Gotchas)
    * [Autocomplete](#Autocomplete)
    * [#Preview and enum reducers](#Preview-and-enum-reducers)
    * [CI build failures](#CI-build-failures)

## Conforming to the Reducer protocol

Reducer 프로토콜을 준수하기 위해 최소한 다음 세 가지를 제공해야 합니다:
``Reducer`` 프로토콜을 준수하려면, 해당 기능이 필요한 상태를 나타내는 ``Reducer/State`` 타입과, 사용자가 수행할 수 있는 액션이나 효과로부터 시스템에 전달될 액션을 나타내는 ``Reducer/Action`` 타입을 정의해야 합니다. 
또한, 네비게이션 등 다른 기능과 조합하여 기능을 구성하는 ``Reducer/body-20w8t`` 프로퍼티를 제공해야 합니다.

간단한 예로, “카운터” 기능의 상태는 정수를 포함하는 구조체로 모델링할 수 있습니다:

```swift
struct CounterFeature: Reducer {
  @ObservableState
  struct State {
    var count = 0
  }
}
```

> Note: 여기서 State에 ``ObservableState()``를 추가하여 뷰가 상태 변경을 자동으로 관찰할 수 있도록 설정했습니다. 
앞으로 릴리스될 라이브러리 버전에서는 이 매크로가 ``Reducer()`` 매크로에 의해 자동으로 적용될 예정입니다.

액션은 증가 버튼과 감소 버튼을 눌렀을 때를 처리하는 두 가지 케이스로 나뉩니다:

```swift
struct CounterFeature: Reducer {
  // ...
  enum Action {
    case decrementButtonTapped
    case incrementButtonTapped
  }
}
```

기능의 로직은 액션이 시스템으로 들어올 때 해당 기능의 현재 상태를 변경함으로써 구현됩니다. 
이는 리듀서의 ``Reducer/body``에 ``Reduce``를 구성하여 가장 쉽게 처리할 수 있습니다.

```swift
struct CounterFeature: Reducer {
  // ...
  var body: some ReducerOf<Self> {
    Reduce { state, action in
      switch action {
      case .decrementButtonTapped:
        state.count -= 1
        return .none
      case .incrementButtonTapped:
        state.count += 1  
        return .none
      }
    }
  }
}
```

``Reduce`` 리듀서의 첫 번째 역할은 주어진 액션에 따라 기능의 현재 상태를 변경하는 것입니다. 
두 번째 역할은 비동기로 실행될 효과를 반환하고, 이 효과의 결과 데이터를 시스템으로 다시 전달하는 것입니다. 
현재 Feature는 실행해야 할 효과가 없으므로, ``Effect/none``을 반환합니다.

하지만, 만약 기능에서 효과를 동반한 작업이 필요하다면 추가적인 구현이 필요합니다. 
예를 들어, 해당 기능이 타이머를 시작하거나 정지할 수 있는 기능을 갖추고, 타이머가 틱(tick)할 때마다 count가 증가한다고 가정해 봅시다. 이 기능은 다음과 같이 구현할 수 있습니다:

```swift
struct CounterFeature: Reducer {
  @ObservableState
  struct State {
    var count = 0
  }
  enum Action {
    case decrementButtonTapped
    case incrementButtonTapped
    case startTimerButtonTapped
    case stopTimerButtonTapped
    case timerTick
  }
  enum CancelID { case timer }

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      switch action {
      case .decrementButtonTapped:
        state.count -= 1
        return .none

      case .incrementButtonTapped:
        state.count += 1
        return .none

      case .startTimerButtonTapped:
        return .run { send in
          while true {
            try await Task.sleep(for: .seconds(1))
            await send(.timerTick)
          }
        }
        .cancellable(CancelID.timer)

      case .stopTimerButtonTapped:
        return .cancel(CancelID.timer)

      case .timerTick:
        state.count += 1
        return .none
      }
    }
  }
}
```

> Note: 이 예제에서는 Task.sleep을 사용한 무한 루프로 타이머를 동작시킵니다. 
이 방법은 간단히 구현할 수 있지만, 작은 오차가 누적되어 정확도가 떨어질 수 있습니다.
기능에 타이머를 사용하는 clock을 주입하면 더 나은 방식으로 구현할 수 있습니다. 자세한 내용은 <doc:DependencyManagement>와 <doc:Testing> 문서를 참고하세요.

## Using the @Reducer macro

``Reducer`` 프로토콜을 직접 준수하여 구현할 수도 있지만, 위에서 살펴본 것처럼 ``Reducer()`` 매크로를 사용하면 기능 구현의 많은 부분을 자동화할 수 있습니다. 
최소한으로는 리듀서를 @Reducer로 주석 처리하면 되고, Reducer 프로토콜 준수 선언조차 생략할 수 있습니다:

```diff
+@Reducer
-struct CounterFeature: Reducer {
+struct CounterFeature {
   @ObservableState
   struct State {
     var count = 0
   }
   enum Action {
     case decrementButtonTapped
     case incrementButtonTapped
   }
   var body: some ReducerOf<Self> {
     Reduce { state, action in
       switch action {
       case .decrementButtonTapped:
         state.count -= 1
         return .none
       case .incrementButtonTapped:
         state.count += 1  
         return .none
       }
     }
   }
 }
```

``Reducer()`` 매크로는 여러 작업을 자동으로 처리해줍니다:

### @CasePathable and @dynamicMemberLookup enums

`@Reducer` 매크로는 Action 열거형(enum)에 [`@CasePathable`][casepathable-docs] 매크로를 자동으로 적용합니다.

```diff
+@CasePathable
 enum Action {
   // ...
 }
```

[Case paths][casepaths-gh]는 열거형(enum) 케이스에 키 경로(key path)의 기능과 편의성을 제공하는 도구로, 리듀서를 조합하는 데 필수적인 도구입니다.

특히, Action 열거형에 이 매크로가 적용되면 라이브러리의 다양한 API에서 열거형 케이스를 지정할 때 키 경로 구문을 사용할 수 있습니다. 예를 들어, 다음과 같은 API에서 활용할 수 있습니다:
``Reducer/ifLet(_:action:destination:fileID:filePath:line:column:)-4ub6q``,
``Reducer/forEach(_:action:destination:fileID:filePath:line:column:)-9svqb``, ``Scope``, 그 외 여러 API

또한, 기능의 ``Reducer/State``가 열거형(enum)일 경우, 즉 하나의 상태가 여러 상호 배타적인 값 중 하나로 모델링될 때 유용한 경우, 
``Reducer()``는 `@CasePathable` 매크로뿐 아니라 `@dynamicMemberLookup`도 자동으로 적용합니다.

```diff
+@CasePathable
+@dynamicMemberLookup
 enum State {
   // ...
 }
```

이를 통해 State의 케이스를 키 경로 구문으로 지정할 수 있을 뿐만 아니라, 점 체이닝(dot-chaining) 구문을 사용해 상태에서 특정 케이스를 선택적으로 추출할 수도 있습니다. 
이는 열거형으로 정의된 옵션을 기반으로 네비게이션을 제어할 수 있는 라이브러리의 연산자를 사용할 때 특히 유용합니다:

```swift
.sheet(
  item: $store.scope(state: \.destination?.editForm, action: \.destination.editForm)
) { store in
  FormView(store: store)
}
```

`state: \.destination?.editForm` 구문은 `State` 열거형에 `@dynamicMemberLookup`과 `@CasePathable`이 모두 적용되었기 때문에 가능합니다.

### Automatic fulfillment of reducer requirements


``Reducer()`` 매크로는 작성하지 않은 ``Reducer`` 프로토콜 요구사항을 자동으로 채워줍니다. 예를 들어, 아래와 같이 간단한 코드도 컴파일됩니다:

```swift
@Reducer
struct Feature {}
```

`@Reducer` 매크로는 빈 ``Reducer/State`` 구조체, 빈 ``Reducer/Action`` 열거형, 그리고 빈 ``Reducer/body-swift.property``를 자동으로 추가합니다. 
이로 인해 Feature는 로직도, 동작도 없는 비활성 리듀서로 동작하게 됩니다.

이 요구사항이 자동으로 충족되면, 이후 실제 구현을 천천히 추가해 나가는 데 유용합니다. 
예를 들어, 이 Feature 리듀서는 아직 도메인을 구현하지 않아도, 라이브러리의 네비게이션 도구를 사용해 상위 도메인에 통합할 수 있습니다. 
이후 준비가 되면 실제 로직과 동작을 구현해 나갈 수 있습니다.

### Destination and path reducers

Composable Architecture에서 자주 사용되는 패턴 중 하나는 기능이 이동할 수 있는 대상(destination)을 열거형 상태에서 각 대상 기능에 해당하는 케이스로 표현한 리듀서로 모델링하는 것입니다. 
이 패턴은 <doc:TreeBasedNavigation> 및 <doc:StackBasedNavigation> 문서에서 자세히 설명되어 있습니다.

이러한 도메인 모델링 방식은 매우 강력하지만, 약간의 반복적인 코드(boilerplate)를 동반할 수 있습니다. 
예를 들어, 특정 기능이 3개의 다른 기능으로 이동할 수 있다고 가정하면, 아래와 같은 `Destination` 리듀서를 작성해야 할 수도 있습니다:

```swift
@Reducer
struct Destination {
  @ObservableState
  enum State {
    case add(FormFeature.State)
    case detail(DetailFeature.State)
    case edit(EditFeature.State)
  }
  enum Action {
    case add(FormFeature.Action)
    case detail(DetailFeature.Action)
    case edit(EditFeature.Action)
  }
  var body: some ReducerOf<Self> {
    Scope(state: \.add, action: \.add) {
      FormFeature()
    }
    Scope(state: \.detail, action: \.detail) {
      DetailFeature()
    }
    Scope(state: \.edit, action: \.edit) {
      EditFeature()
    }
  }
}
```

이 코드가 최악은 아니지만, 24줄에 달하는 반복적인 코드가 많고, 새로운 대상을 추가하려면 여러 곳에 변경을 가해야 합니다.
구체적으로는 ``Reducer/State`` 열거형에 새로운 케이스를 추가하고, ``Reducer/Action`` 열거형에 해당 케이스를 추가하며, ``Reducer/body-swift.property``에 새로운 Scope를 정의해야 합니다.

하지만 이제 ``Reducer()`` 매크로를 사용하면, 아래와 같은 간단한 선언만으로 이러한 모든 코드를 자동으로 생성할 수 있습니다:

```swift
@Reducer
enum Destination {
  case add(FormFeature)
  case detail(DetailFeature)
  case edit(EditFeature)
}
```

24줄의 코드가 6줄로 줄어들었습니다. `@Reducer` 매크로를 이제 열거형(enum)에 적용할 수 있으며, 각 케이스는 해당 케이스의 로직과 동작을 관리하는 리듀서를 포함합니다.

또한, 이 스타일의 `Destination` 열거형 리듀서와 함께 ``Reducer/ifLet(_:action:)`` 연산자를 사용할 때, 자동으로 추론으로 인해서 후행 클로저를 완전히 생략할 수 있습니다.

```diff
 Reduce { state, action in
   // Core feature logic
 }
 .ifLet(\.$destination, action: \.destination)
-{
-  Destination()
-}
```

이 패턴은 <doc:StackBasedNavigation>과 같이 경로 기반의 네비게이션을 처리할 때 자주 사용되는 Path 리듀서에도 적용됩니다. 
이 경우, ``Reducer/forEach(_:action:)`` 연산자의 후행 클로저를 생략할 수 있습니다:

```diff
Reduce { state, action in
  // Core feature logic
}
.forEach(\.path, action: \.path)
-{
-  Path()
-}
```

Further, for `Path` reducers in particular, the ``Reducer()`` macro also helps you reduce
boilerplate when using the initializer 
``SwiftUI/NavigationStack/init(path:root:destination:fileID:filePath:line:column:)`` that comes with the library. 
In the last trailing closure you can use the ``Store/case`` computed property to switch on the 
`Path.State` enum and extract out a store for each case:

또한, 특히 `Path` 리듀서의 경우, ``Reducer()`` 매크로는 라이브러리에서 제공하는
``SwiftUI/NavigationStack/init(path:root:destination:fileID:filePath:line:column:)`` 초기화를 사용할 때 반복적인 코드를 줄이는 데 도움을 줍니다. 
마지막 후행 클로저에서는 ``Store/case`` 계산 속성을 사용하여 `Path.State` 열거형을 스위치 구문으로 처리하고, 각 케이스에 대해 적절한 스토어를 추출할 수 있습니다:

```swift
NavigationStack(path: $store.scope(state: \.path, action: \.path)) {
  // Root view
} destination: { store in
  switch store.case {
  case let .add(store):
    AddView(store: store)
  case let .detail(store):
    DetailView(store: store)
  case let .edit(store):
    EditView(store: store)
  }
}
```

#### Navigating to non-reducer features

Composable Architecture 리듀서로 모델링되지 않은 기능으로 이동하거나 해당 기능을 표시해야 할 때가 종종 있습니다.
이런 상황은 Composable Architecture로 구축되지 않은 기존 기능(legacy features)이나, 
매우 단순하여 완전한 리듀서가 필요하지 않은 기능에서 발생할 수 있습니다.

이러한 경우, ``ReducerCaseIgnored()`` 및 ``ReducerCaseEphemeral()`` 매크로를 사용해 리듀서가 필요하지 않은 케이스를 주석 처리할 수 있습니다.
이 매크로들에 대한 자세한 내용은 해당 문서를 참고하세요.

예를 들어, 특정 기능이 여러 다른 기능으로 네비게이션할 수 있다고 가정해 보겠습니다.
이 중 대부분은 Composable Architecture로 구축된 기능이지만, 하나는 그렇지 않은 경우입니다:

```swift
@Reducer
enum Destination {
  case add(AddItemFeature)
  case edit(EditItemFeature)
  @ReducerCaseIgnored
  case item(Item)
}
```


이 상황에서는 `.item` 케이스가 전체 리듀서가 아닌 단순한 항목(item)을 보유하고 있습니다.
이러한 이유로 `@Reducer` 매크로의 일부 확장에서 이 케이스를 제외해야 합니다.

그런 다음, 이 케이스에서 뷰를 표시하려면 다음과 같이 할 수 있습니다:

```swift
.sheet(item: $store.scope(state: \.destination?.item, action: \.destination.item)) { store in
  ItemView(item: store.withState { $0 })
}
```

> Note: ``Store/withState(_:)``는 `.item` 케이스에 포함된 값에 ``ObservableState()`` 매크로가 적용되지 않았기 때문에 필요합니다. 
또한, 해당 값에는 ``ObservableState()`` 매크로가 적용될 필요도 없습니다. `withState`를 사용하면 상태 관찰(observation) 없이 스토어에 있는 상태에 접근할 수 있는 방법을 제공합니다.

#### Synthesizing protocol conformances on State and Action

`@Reducer`를 열거형(enum)에 사용할 때, `State`와 `Action` 타입은 자동으로 생성됩니다. 따라서, 
이러한 타입에 대해 `Equatable`, `Hashable` 등의 준수를 추가하려면 직접 확장(extend)해야 합니다:

```swift
@Reducer
enum Destination {
  // ...
}
extension Destination.State: Equatable {}
```

> Note: In Swift <6 the above extension causes a compiler error due to a bug in Swift.
>
> To work around this compiler bug, the library provides a version of the `@Reducer` macro that
> takes two ``ComposableArchitecture/_SynthesizedConformance`` arguments, which allow you to
> describe the protocols you want to attach to the `State` or `Action` types:
>
> ```swift
> @Reducer(state: .equatable, .sendable, action: .sendable)
> enum Destination {
>   // ...
> }
> ```

#### Nested enum reducers

There may be times when an enum reducer may want to nest another enum reducer. To do so, the parent
enum reducer must specify the child's `Body` associated value and `body` static property explicitly:

```swift
@Reducer
enum Modal { /* ... */ }

@Reducer
enum Destination {
  case modal(Modal.Body = Modal.body)
}
```

### Gotchas

#### Autocomplete

Applying `@Reducer` can break autocompletion in the `body` of the reducer. This is a known
[issue](https://github.com/apple/swift/issues/69477), and it can generally be worked around by
providing additional type hints to the compiler:

 1. Adding an explicit `Reducer` conformance in addition to the macro application can restore
    autocomplete throughout the `body` of the reducer:

    ```diff
     @Reducer
    -struct Feature {
    +struct Feature: Reducer {
    ```

 2. Adding explicit generics to instances of `Reduce` in the `body` can restore autocomplete
    inside the `Reduce`:

    ```diff
     var body: some Reducer<State, Action> {
    -  Reduce { state, action in
    +  Reduce<State, Action> { state, action in
    ```

#### #Preview and enum reducers

The `#Preview` macro is not capable of seeing the expansion of any macros since it is a macro 
itself. This means that when using destination and path reducers (see
<doc:Reducer#Destination-and-path-reducers> above) you cannot construct the cases of the state 
enum inside `#Preview`:

```swift
#Preview {
  FeatureView(
    store: Store(
      initialState: Feature.State(
        destination: .edit(EditFeature.State())  // 🛑
      )
    ) {
      Feature()
    }
  )
}
```

The `.edit` case is not usable from within `#Preview` since it is generated by the ``Reducer()``
macro.

The workaround is to move the view to a helper that be compiled outside of a macro, and then use it
inside the macro:

```swift
#Preview {
  preview
}
private var preview: some View {
  FeatureView(
    store: Store(
      initialState: Feature.State(
        destination: .edit(EditFeature.State())
      )
    ) {
      Feature()
    }
  )
}
```

You can use a computed property, free function, or even a dedicated view if you want. You can also
use the old, non-macro style of previews by using a `PreviewProvider`:

```swift
struct Feature_Previews: PreviewProvider {
  static var previews: some  View {
    FeatureView(
      store: Store(
        initialState: Feature.State(
          destination: .edit(EditFeature.State())
        )
      ) {
        Feature()
      }
    )
  }
}
```

#### Error: External macro implementation … could not be found

When integrating with the Composable Architecture, one may encounter the following error:

> Error: External macro implementation type 'ComposableArchitectureMacros.ReducerMacro' could not be
> found for macro 'Reducer()'

This error can show up when the macro has not yet been enabled, which is a separate error that
should be visible from Xcode's Issue navigator.

Sometimes, however, this error will still emit due to an Xcode bug in which a custom build
configuration name is being used in the project. In general, using a build configuration other than
"Debug" or "Release" can trigger upstream build issues with Swift packages, and we recommend only
using the default "Debug" and "Release" build configuration names to avoid the above issue and
others.

#### CI build failures

When testing your code on an external CI server you may run into errors such as the following:

> Error: CasePathsMacros Target 'CasePathsMacros' must be enabled before it can be used.
>
> ComposableArchitectureMacros Target 'ComposableArchitectureMacros' must be enabled before it can
> be used.

You can fix this in one of two ways. You can write a default to the CI machine that allows Xcode to
skip macro validation:

```shell
defaults write com.apple.dt.Xcode IDESkipMacroFingerprintValidation -bool YES
```

Or if you are invoking `xcodebuild` directly in your CI scripts, you can pass the
`-skipMacroValidation` flag to `xcodebuild` when building your project:

```shell
xcodebuild -skipMacroValidation …
```

[casepathable-docs]: https://swiftpackageindex.com/pointfreeco/swift-case-paths/main/documentation/casepaths/casepathable()
[casepaths-gh]: http://github.com/pointfreeco/swift-case-paths


## Topics

### Implementing a reducer

- ``Reducer()``
- ``State``
- ``Action``
- ``body-swift.property``
- ``Reduce``
- ``Effect``

### Composing reducers

- ``ReducerBuilder``
- ``CombineReducers``

### Embedding child features

- ``Scope``
- ``ifLet(_:action:then:fileID:filePath:line:column:)-2r2pn``
- ``ifCaseLet(_:action:then:fileID:filePath:line:column:)-7sg8d``
- ``forEach(_:action:element:fileID:filePath:line:column:)-6zye8``
- <doc:Navigation>

### Sharing state

- <doc:SharingState>
- ``Shared``
- ``PersistenceKey``

### Supporting reducers

- ``EmptyReducer``
- ``BindingReducer``
- ``Swift/Optional``

### Reducer modifiers

- ``dependency(_:_:)``
- ``transformDependency(_:transform:)``
- ``onChange(of:_:)``
- ``signpost(_:log:)``
- ``_printChanges(_:)``

### Supporting types

- ``ReducerOf``

### Deprecations

- <doc:ReducerDeprecations>
