- javascript객체는 prototypical inheritance를 할 수 있다. 
객체 Derived가 다른 객체 Proto를 prototype으로 가질 때, Derived가 갖지 않은 property read는 Proto로 포워딩된다.
하지만 property write가 일어나면 Derived는 직접 값을 저장하고, 이 시점부터 Proto의 property를 shadowing한다.

<img src="https://docs.google.com/drawings/d/e/2PACX-1vT9-3Tnf-54Q9ABUj7Gu2HR3RkteWCT6CisOzJTJnPU8O1MVvA2_2qmiudvmJefAgCjShS7NCUHBxpP/pub?w=927&h=665">

- angularJS는 ng-model attribute로 scope의 property와 DOM간의 양방향 바인딩을 만든다. 
ng-if는 평가된 결과가 falsable이면 해당 DOM을 제거하는데, 이 과정에서 결과와 상관없이 
원래 scope을 prototypical inherit하는 child scope을 만들어서 바인딩한다.

- 이 때 해당 DOM의 ng-model은 처음 의도되었던 것과 다른 scope을 보게 되는데, 
이 scope이 원래 scope을 prototype으로 가지게 되면서 읽기는 가능하지만 쓰기는 불가능한 상태가 된다. 
parent에서 초기값을 가져오고, 변경된 값은 child에 쓰게 된다. controller는 변화된 값을 알지 못하게 된다.

- 이 경우 값이 읽혀지지만 써지지는 않는( 하지만 화면에는 반영되는 ) 요상한 상태가 되고, 
dot notation을 사용하면 또 정상동작하는( ng-model='wrapper.value' ) 상태가 되어 혼돈에 빠져든다.
wrapper객체 자체는 언제나 읽기만 일어나므로, shadowing이 일어나지 않기 때문.
ng-model='$parent.value'로 접근할 수 있지만, ng-if를 빼면 $parent도 빼줘야 하므로 차라리 wrapper가 낫다.



<img src="https://docs.google.com/drawings/d/e/2PACX-1vSO755xFBA0ZEaU5kW0IeU67DrzUsBbhWGKy3sS2Ah4ecDh630R4cfbmv0yVkXJlJzLKEXbeY-zV3He/pub?w=1089&h=1263">


