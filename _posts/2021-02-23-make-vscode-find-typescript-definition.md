---
layout: post
title: "vscode의 타입 정의 추적"
---

let t = {
    aaa: 1,
    b: 'hello',
};

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