---
layout: post
title: "vscode의 타입 정의 추적"
tags: 
    - typescript 
    - vscode
---

회사 프로젝트에서 일종의 오브젝트 데코레이터?를 만들어 쓰고 있다. 어떤 스트럭쳐를 넘겨주면 정해진 규칙에 따라 프로퍼티를 달아주고 어떤 프로퍼티는 값을 바꿔주고 어디는 접근 제한자를 달아주고...와이어프레임을 넘겨주면 규칙에 따라 살을 붙여준다. 복잡하지만 패턴화된 변수 선언을 위한 타자 줄이기.

{% highlight ts %}
let arm = makebody({
    hand: {
        finger: {
            nail: {
            }
        }
    }
});
arm.hand.bone.rotate();
arm.hand = {};  // error
/*
arm : {
    readonly hand: {
        readonly finger: {
            readonly nail: {
                readonly bone: Bone,
            }
            readonly bone: Bone,
        }
        readonly bone: Bone,
    }
    readonly bone: Bone,
}
오브젝트를 일종의 tree로 보고, 
모든 노드를 readonly + bone: Bone 추가,
arm 의 타입은 위와 같이 결정된다.
*/
{% endhighlight %}

만들어서 잘 쓰던 와중에, 위의 와이어프레임 오브젝트(makebody의 인수)와 데코레이트된 오브젝트(makebody의 결과)에 접근하는 코드 사이를 vscode의 `Go to...` 으로 오갈 수 없는 걸 발견했다. 

{% highlight ts %}
let finger = makebody({
    nail: {
        color: color.red,   // find all reference로 찾아갈 수 없다.
    }
});
finger.nail.color = color.red;  // go to definition으로 찾아갈 수 없다.
{% endhighlight %}

정식 변수 선언도 아니라 안되는 게 당연한가...싶었지만 `find all reference` 나 `rename`도 안되는 건 너무 불편하다. 게다가 타입 추론이 당연한 언어이니만큼 추론의 근거를 선언문이라고 볼 수도 있지 않을까? vscode가 어디까지 쫓아가는지를 테스트해보자. 아래의 모든 코드에서 `prop`을 찾아가고 바꾸고 할 수 있는지 테스트해보았다.

{% highlight ts %}
let decorated = {prop: 1};
decorated.prop;
{% endhighlight %}
당연히 잘 된다.

{% highlight ts %}
let decorated = Object.assign({trait: 0, {prop: 1});
decorated.prop;
{% endhighlight %}
이 경우 decorated의 타입은 `{trait: {}} & {prop: {}}` 가 되는데, 잘 찾아간다. 게다가 Object.assign에 넘기는 인자에서 반환값 타입의 근거를 찾아내는데, 

{% highlight ts %}
assign<T, U, V>(target: T, source1: U, source2: V): T & U & V;
{% endhighlight %}
제네릭판 `Object.assign`의 프로토타입. 반환 타입의 근거가 인자의 타입에서 올 경우 vscode는 쫓아간다.

{% highlight ts %}
function makeReadonly<T>(writable: T){
    return writable as {readonly [K in keyof T]: T[K]};
}
let decorated = makeReadonly({prop: 1});
decorated.prop;
{% endhighlight %}
decorated.prop은 원본과 다르게 readonly. 접근 제한자를 붙이더라도 vscode는 붙기 전의 선언을 쫓아간다. 

{% highlight ts %}
function makeNumber<T>(src: T){
    return src as any as {[K in keyof T]: number};
}
let decorated = makeNumber({prop: '1'});
decorated.prop;
{% endhighlight %}
타입을 바꿔줘도 쫓아간다. mapped type은 그냥 쫓아가는 듯.

{% highlight ts %}
function addTrait<T>(src: T){
    type recursiveDecorate<T> = {[K in keyof T]: recursiveDecorate<T[K]>} & {trait?: number};
    return src as recursiveDecorate<T>;
}
let traited = addTrait({
    shell0: {
        shell1: {
            prop: 1,
        }
    }
});
traited.shell0.trait;
traited.shell0.shell1.trait;
traited.shell0.shell1.prop;
{% endhighlight %}
재귀 타이핑. 역시 쫓아간다. 이쯤 되면 되는 게 당연하고 내 코드에 뭔가 문제가 있음이 확실해졌다.

{% highlight ts %}
type numberKeys<T> = { [K in keyof T]: T[K] extends number ? K : never }[keyof T];
type notnumberKeys<T> = { [K in keyof T]: T[K] extends number ? never : K }[keyof T];
type readonlyNumbers<T> = {readonly [P in numberKeys<T>]: number} & {[P in notnumberKeys<T>]: T[P]}
function makeReadonlyNumbers<T>(src: T){
    return src as any as readonlyNumbers<T>;
}

let restricted = makeReadonlyNumbers({
    n: 1,
    s: 'a',
});
restricted.n;
restricted.s;
restricted.n = 2;   // error. n is readonly
{% endhighlight %}
드디어 찾았다. 위의 `restricted.n`과 `restricted.s` 는 정의를 찾아가지 못한다. 위의 코드는 타입의 프로퍼티 중 조건에 맞는 일부에만 접근 제한자를 붙이기 위해 만들어졌다(number 에만 readonly). 실제 위와 똑같은 코드가 프로젝트에서도 사용중이었고 문제의 원인이었다. 위의 코드를 타입스크립트가 제공하는 유틸리티 타입을 사용하도록 바꿔보자.

{% highlight ts %}
type readonlyNumbers<T> = Readonly<Pick<T, numberKeys<T>>> & Pick<T, notnumberKeys<T>>
{% endhighlight %}
이제 잘 찾아간다. 문제는 해결되었지만, 유틸리티 타입을 찾아가서 문제의 원인을 좀 더 파헤쳐보자. 일단 경우를 좁히기 위해 intersection type과 `Readonly` 관련은 빼고 `Pick`부터 살펴보자.

{% highlight ts %}
type numberKeys<T> = { [K in keyof T]: T[K] extends number ? K : never }[keyof T];
type numbers1<T> = {[P in numberKeys<T>]: T[P]}
type numbers2<T> = Pick<T, numberKeys<T>>
let signature = {n: 1};
let typed1 = signature as numbers1<typeof signature>;
let typed2 = signature as numbers2<typeof signature>;
typed1.n = 2;
typed2.n = 2;
{% endhighlight %}
위의 `typed2.n` 만이 `signature.n` 을 찾아간다. 와 `Pick`를 찾아와서 inlining해보자.

{% highlight ts %}
type Pick<T, K extends keyof T> = {[P in K]: T[P];};
{% endhighlight %}
이상하다. `Pick`의 내용은 위의 numbers1에 이미 inlining되어 있다. 혹시 vscode가 제공된 유틸리티 타입에 대해서만 특별한 처리를 하는 건가 했지만 그렇진 않다(내용을 복사해서 Pick2를 만들어도 같은 결과).


재귀 제네릭(인터섹션 + 접근 제한자) 왜 나는 안돼는거야?

mapped type




function addc<T>(src: T){
    return Object.assign(t, {c: 3});
}

// type TT<T> = treeBranch<T>;
// type TT<T> = {readonly [P in keyof T]: string} & {c: number};
type stringKeys<T> = { [K in keyof T]: T[K] extends string ? K : never }[keyof T];
type tonumber<T> = {[K in keyof T]: number};
// type TT<T> = {readonly [P in stringKeys<T>]: T[P]} & {c: number};
type TT<T> = Readonly<tonumber<Pick<T, stringKeys<T>>>> & {c: number};
function addc2<T>(src: T): TT<T>{
    return null as TT<T>;
}

let k = addc(t);
let k2 = addc2(t);

k.aaa = 2;
let j = k2.b;