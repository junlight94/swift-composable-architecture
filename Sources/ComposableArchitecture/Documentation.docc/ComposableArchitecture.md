# ``ComposableArchitecture``

Composable Architecture(TCA)는 애플리케이션을 일관되고 이해하기 쉬운 방식으로 구축하기 위한 라이브러리로, 
구성(composition), 테스트(testing), 사용 편의성(ergonomics)을 중심으로 설계되었습니다. 
TCA는 SwiftUI, UIKit 등 다양한 UI 프레임워크에서 사용할 수 있으며, iOS, macOS, tvOS, watchOS 등 모든 Apple 플랫폼에서 활용할 수 있습니다.

## Additional Resources

- [GitHub Repo](https://github.com/pointfreeco/swift-composable-architecture)
- [Discussions](https://github.com/pointfreeco/swift-composable-architecture/discussions)
- [Point-Free Videos](https://www.pointfree.co/collections/composable-architecture)

## Overview

이 라이브러리는 다양한 목적과 복잡성을 가진 애플리케이션을 구축하는 데 사용할 수 있는 핵심 도구들을 제공합니다. 
또한 애플리케이션 개발 시 일상적으로 직면하는 여러 문제를 해결하기 위한 명확한 가이드를 제공합니다. 아래는 TCA가 제공하는 주요 기능과 해결책입니다:

* **상태 관리 (State management)**

    애플리케이션의 상태를 단순한 값 타입으로 관리하는 방법과, 여러 화면에서 상태를 공유하여 
    한 화면에서의 변경 사항을 다른 화면에서도 즉시 반영되도록 하는 방법을 제공합니다.

* **구성 (Composition)**

    큰 기능을 더 작고 독립적인 컴포넌트로 분리하는 방법을 제시합니다. 
    이렇게 분리된 컴포넌트는 자체적인 모듈로 추출할 수 있으며, 이후 쉽게 결합하여 전체 기능을 구현할 수 있습니다.

* **사이드 이펙트 (Side effects)**

    애플리케이션의 일부가 외부(예: 네트워크, 데이터베이스)와 상호작용할 때, 
    이를 가장 테스트 가능하고 이해하기 쉬운 방식으로 처리하는 방법을 제공합니다.

* **테스트 (Testing)**

    아키텍처로 구축한 기능을 테스트하는 방법뿐만 아니라, 여러 구성 요소로 이루어진 기능의 통합 테스트를 작성하는 방법을 제공합니다.
    또한, 애플리케이션의 사이드 이펙트가 비즈니스 로직에 어떤 영향을 미치는지 이해할 수 있도록 엔드 투 엔드 테스트를 작성하는 방법을 제공합니다.
    이를 통해 비즈니스 로직이 예상대로 동작한다는 강력한 보장을 할 수 있습니다.

* **사용 편의성 (Ergonomics)**

    위의 모든 기능을 단순한 API로 구현하며, 최소한의 개념과 복잡도로 효율적으로 작업할 수 있도록 설계되었습니다.

## Topics

### Essentials

- <doc:GettingStarted>
- <doc:DependencyManagement>
- <doc:Testing>
- <doc:Navigation>
- <doc:SharingState>
- <doc:Performance>
- <doc:FAQ>

### Tutorials

- <doc:MeetComposableArchitecture>
- <doc:BuildingSyncUps>

### State management

- ``Reducer``
- ``Effect``
- ``Store``
- <doc:SharingState>

### Testing

- ``TestStore``
- <doc:Testing>

### Integrations

- <doc:SwiftConcurrency>
- <doc:SwiftUIIntegration>
- <doc:ObservationBackport>
- <doc:UIKit>

### Migration guides

- <doc:MigrationGuides>

## See Also

The collection of videos from [Point-Free](https://www.pointfree.co) that dive deep into the
development of the library.

* [Point-Free Videos](https://www.pointfree.co/collections/composable-architecture)
