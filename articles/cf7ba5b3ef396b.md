---
title: "Python 3.11の型ヒントに導入される\"PEP 673 – Self Type\"を訳してみた"
emoji: "🏷️"
type: "tech"
topics:
  - "python"
published: true
published_at: "2022-04-02 11:11"
---

## この記事は

Python 3.11の型ヒント関連の新機能として、[PEP 673 – Self Type](https://peps.python.org/pep-0673/)が採択されていたので、読み解いてみます。
13k字超の大作になってしまった...

## 長いので先に感想

- 冗長だった型表現が簡潔に書けてよさそう
- Adaptor / Factoryパターンの設計をする時に型がつけられそう
    - Pyramid, Zope / Ploneの型付けに期待...?
- まだ抽象クラス・メタクラスを使う時はしばしば冗長
    - これは意図して冗長にしている認識。簡潔に書こうとすると混乱が始まる

以下拙訳を掲載。訳の下に個人的所感を書いています

---

## 要約

> This PEP introduces a simple and intuitive way to annotate methods that return an instance of their class. This behaves the same as the TypeVar-based approach specified in [PEP 484](https://peps.python.org/pep-0484) but is more concise and easier to follow.

自分自身を返すメソッドに型をつけられるようになる。その振る舞いは`TypeVar`ベースのアプローチと同じだけれど、より明確で型を追いやすくなっている。

## どう変わるのか?

### 今までだと

> A common use case is to write a method that returns an instance of the same class, usually by returning self.

```python
class Shape:
    def set_scale(self, scale: float):
        self.scale = scale
        return self


Shape().set_scale(0.5)  # => Shapeに推論される
```

まぁこれは普通に推論される。これだけだと利点がわからないので続きに


```python
class Shape:
    def set_scale(self, scale: float) -> Shape:
        self.scale = scale
        return self


Shape().set_scale(0.5)  # => Shape
```

これも明示的に書いているのでそうなる。では次はどうなのか?

サブクラスが継承元のメソッドを使っていて、その先で継承先のメソッドをチェーンする時

```python
class Circle(Shape):
    def set_radius(self, r: float) -> Circle:
        self.radius = r
        return self


Circle().set_scale(0.5)  # *Shape*であり, Circleではない
Circle().set_scale(0.5).set_radius(2.7)
# => error: Shape has no attribute "set_radius"
```

![](https://storage.googleapis.com/zenn-user-upload/c099691777c7-20220331.png)

エラーになってしまう。

### Workaroundはあるけれど、冗長

`typing.TypeVar` を使えば回避できるが、ちょっと冗長

```python
from typing import TypeVar
TShape = TypeVar("TShape", bound="Shape")


class Shape:
    def set_scale(self: TShape, scale: float) -> TShape:
        self.scale = scale
        return self


class Circle(Shape):
    def set_radius(self, radius: float) -> Circle:
        self.radius = radius
        return self


Circle().set_scale(0.5).set_radius(2.7)  # => Circle
```

### なので、Python 3.11ではこうやって書けるようになる

`typing.Self`を使うことによって自分自身を動的に定義できるようになる

```python
from typing import Self


class Shape:
    def set_scale(self, scale: float) -> Self:
        self.scale = scale
        return self
	

class Circle(Shape):
    def set_radius(self, radius: float) -> Self:
        self.radius = radius
        return self


Circle().set_scale(0.5)  # => Circleに推論される!
```

## その他の仕様

:::message alert
ここから細かい話なので折りたたんであります。適宜読み飛ばしてください
:::

:::details classmethodでは...

今まで通りclassmethodを書くと、

```python
class Shape:
    def __init__(self, scale: float) -> None: ...

    @classmethod
    def from_config(cls, config: dict[str, float]) -> Shape:
        return cls(config["scale"])	
```

ただしこれは継承先のメソッドを呼んでも全部`Shape`に推論されてしまうので、問題がある

```python
class Circle(Shape): ...
    def circumference

shape = Shape.from_config({"scale": 7.0})
# => Shape

circle = Circle.from_config({"scale": 7.0})
# => Circleではなく *Shape* になってしまうので...

circle.circumference()
# error: `Shape` has no attribute `circumference`
```

先程と同様にworkaroundもあるけれど、やはり冗長

```python
Self = TypeVar("Self", bound="Shape")

class Shape:
    @classmethod
    def from_config(
        cls: type[Self], config: dict[str, float]
    ) -> Self:
        return cls(config["scale"])
```

なので、こんな感じで書けるようになる

```python
from typing import Self


class Shape:
    @classmethod
    def from_config(cls, config: dict[str, float]) -> Self:
        return cls(config["scale"])
```

:::

:::details 引数に自分自身を取る時

他の用途として、メソッドの引数に自分自身を取る時、

```python
Self = TypeVar("Self", bound="Shape")


class Shape:
    def difference(self: Self, other: Self) -> float: ...

    def apply(self: Self, f: Callable[[Self], None]) -> None: ...
```

と書けるが、冗長なので(ry

```python
from typing import Self


class Shape:
    def difference(self, other: Self) -> float: ...

    def apply(self, f: Callable[[Self], None]) -> None: ...
```

スッキリ書けるようになる。もちろん`self`は`Self`型、つまり`Shape`を指す
言うまでもなく、selfを明示的に書くことも可能


```python
class Shape:
    def difference(self: Self, other: Self) -> float: ...
```

:::

:::details Attributeに型をつける時は

例えば`LinkedList`はAttributeが現在のclassのsub-classでなければならないとき、

```python
from dataclasses import dataclass
from typing import Generic, TypeVar

T = TypeVar("T")

@dataclass
class LinkedList(Generic[T]):
    value: T
    next: LinkedList[T] | None = None

# OK
LinkedList[int](value=1, next=LinkedList[int](value=2))
# だめ
LinkedList[int](value=1, next=LinkedList[str](value="hello"))
```

ただし、`next` Attributeに型`LinkedList[T]`を付与してしまうと無効な構造ができてしまう

```python
from dataclasses import dataclass
from typing import Generic, TypeVar

T = TypeVar("T")


@dataclass
class LinkedList(Generic[T]):
    value: T
    next: LinkedList[T] | None = None


# OK
LinkedList[int](value=1, next=LinkedList[int](value=2))
# だめだけど通ってしまう
LinkedList[int](value=1, next=LinkedList[str](value="hello"))
```

そこで、`next: Self | None`(訳注: typing.Optionalを用いても良さそう)を使うと、このように表現できる
```python
from typing import Self


@dataclass
class LinkedList(Generic[T]):
    value: T
    next: Self | None = None


@dataclass
class OrdinalLinkedList(LinkedList[int]):
    def ordinal_value(self) -> str:
        return as_ordinal(self.value)


xs = OrdinalLinkedList(value=1, next=LinkedList[int](value=2))
# TypeError: Expected OrdinalLinkedList, got LinkedList[int].

if xs.next is not None:
    xs.next = OrdinalLinkedList(value=3, next=None)  # OK
    xs.next = LinkedList[int](value=3, next=None)  # だめ
```

これは、`Self`型を含む各Attributeをその型を返すPropertyとして扱うのと同等で、`typing.TypeVar`を使ってこのようにも書ける

```python
from dataclasses import dataclass
from typing import Any, Generic, TypeVar

T = TypeVar("T")
Self = TypeVar("Self", bound="LinkedList")


class LinkedList(Generic[T]):
    value: T

    @property
    def next(self: Self) -> Self | None:
        return self._next

    @next.setter
    def next(self: Self, next: Self | None) -> None:
        self._next = next


class OrdinalLinkedList(LinkedList[int]):
    def ordinal_value(self) -> str:
        return str(self.value)
```

:::

:::details Generic Classでは

`Self`はGenericなclassのメソッドでも使えるので、

```python
class Container(Generic[T]):
    value: T
    def set_value(self, value: T) -> Self: ...
```

と書くことができて、`typing.TypeVar`を使って(ry

```python
Self = TypeVar("Self", bound="Container[Any]")

class Container(Generic[T]):
    value: T
    def set_value(self: Self, value: T) -> Self: ...
```

振る舞いはメソッドが呼び出されたObjectの型引数を保持するので、具象型をのメソッド`Container[int]`呼び出すと`Self`は`Container[int]`になるし、ジェネリック型`Container[T]`のメソッドを呼び出すともちろん`Container[T]`となる。

```python
def object_with_concrete_type() -> None:
    int_container: Container[int]
    str_container: Container[str]
    reveal_type(int_container.set_value(42))  # => Container[int]
    reveal_type(str_container.set_value("hello"))  # => Container[str]

def object_with_generic_type(
    container: Container[T], value: T,
) -> Container[T]:
    return container.set_value(value)  # => Container[T]
```

しかし、ここでは`set_value`メソッドで`self.value`の正確な型を指定しない。
ある型チェッカーでは`TypeVar(“Self”, bound=Container[T])`とclass-localな型変数を使って`Self`とすると、正確な型`T`が推論されるかもしれない。
ただし、class-localな型変数は標準の型システムの機能ではないので、`self.value`に`Any`を推論させることもできる。これは型チェッカーの実装に委ねられている。

ここで、`Self[int]`のような型引数で`Self`を使うのは許されない。不必要に`self`引数の型が曖昧になり、複雑になるからで

```python
class Container(Generic[T]):
    def foo(
        self, other: Self[int], other2: Self,
    ) -> Self[str]:  # Rejected
        ...
```

こんな時は、明示的に書くことを推奨される

```python
class Container(Generic[T]):
    def foo(
        self: Container[T],
        other: Container[int],
        other2: Container[T]
    ) -> Container[str]: ...
```

:::

:::details Protocolでも使える

Protocolの中でも

```
from typing import Protocol, Self


class ShapeProtocol(Protocol):
    scale: float

    def set_scale(self, scale: float) -> Self:
        self.scale = scale
        return self
```

の用に使えて、これは`typing.TypeVar`を使って(ry

```python
from typing import Protocol, Self


class ShapeProtocol(Protocol):
    scale: float

    def set_scale(self, scale: float) -> Self:
        self.scale = scale
        return self
```

Protocolにバインドされた`TypeVars`の動作の詳細については、[PEP 544](https://peps.python.org/pep-0544#self-types-in-protocols)を参照。
特定のclassがprotocolと互換性があるかチェックする時は、もしprotocolがメソッドやAttributeの型付けを`Self`でしている場合、または対応するメソッドやAttributeの型付けが`Self`か具象型`Foo`かそのサブクラスになっていれば、クラス`Foo`はそのprotocolと互換性があることになる。

```python
from typing import Protocol


class ShapeProtocol(Protocol):
    def set_scale(self, scale: float) -> Self: ...


class ReturnSelf:
    scale: float = 1.0

    def set_scale(self, scale: float) -> Self:
        self.scale = scale
        return self


class ReturnConcreteShape:
    scale: float = 1.0

    def set_scale(self, scale: float) -> ReturnConcreteShape:
        self.scale = scale
        return self


class BadReturnType:
    scale: float = 1.0

    def set_scale(self, scale: float) -> int:
        self.scale = scale
        return 42


class ReturnDifferentClass:
    scale: float = 1.0

    def set_scale(self, scale: float) -> ReturnConcreteShape:
        return ReturnConcreteShape(...)


def accepts_shape(shape: ShapeProtocol) -> None:
    y = shape.set_scale(0.5)
    reveal_type(y)


def main() -> None:
    return_self_shape: ReturnSelf
    return_concrete_shape: ReturnConcreteShape
    bad_return_type: BadReturnType
    return_different_class: ReturnDifferentClass

    accepts_shape(return_self_shape)  # OK
    accepts_shape(return_concrete_shape)  # OK
    accepts_shape(bad_return_type)  # だめ
    # サブクラスを返さないのでこれはだめ
    accepts_shape(return_different_class)
```

:::

:::details `Self`に型付けできる場所

`Self`で型付けすることはclassのコンテキストだけで有効なので、常にカプセル化されたclassを参照する。
ネストされたclassを含むコンテキストでは、`Self`は常に一番内側のclassを参照する。

```python
class ReturnsSelf:
    def foo(self) -> Self: ... # OK

    @classmethod
    def bar(cls) -> Self:  # OK
        return cls()

    def __new__(cls, value: int) -> Self: ...  # OK

    def explicitly_use_self(self: Self) -> Self: ...  # OK

    # OK (他の型にネストもできる)
    def returns_list(self) -> list[Self]: ...

    # OK (同上)
    @classmethod
    def return_cls(cls) -> type[Self]:
        return cls


class Child(ReturnsSelf):
    # OK (Selfを使うメソッドを上書きできる)
    def foo(self) -> Self: ...


class TakesSelf:
    def foo(self, other: Self) -> bool: ...  # OK


class Recursive:
    # OK (SelfかNoneを返す@propertyとして扱われる)
    next: Self | None


class CallableAttribute:
    def foo(self) -> int: ...

    # OK (@property returning the Callableを返す@propertyとして扱われる)
    bar: Callable[[Self], int] = foo

class HasNestedFunction:
    x: int = 42

    def foo(self) -> None:

        # OK (HasNestedFunctionに束縛される)
        def nested(z: int, inner_self: Self) -> Self:
            print(z)
            print(inner_self.x)
            return inner_self

        nested(42, self)  # OK


class Outer:
    class Inner:
        def foo(self) -> Self: ...  # OK (Innerに束縛される)
```

ただし、次のようなものは型チェッカーによって怒られてしまう。

```python
def foo(bar: Self) -> Self: ...  # classの中でないのでダメ

bar: Self  # ダメ (同上)

class Foo:
    # Unknownとして扱われるのでダメ
    def has_existing_self_annotation(self: T) -> Self: ...

class Foo:
    def return_concrete_type(self) -> Self:
        return Foo()  # ダメ (下記参照)

class FooChild(Foo):
    child_value: int = 42

    def child_method(self) -> None:
        # 実行時評価ではFooになってしまう
        y = self.return_concrete_type()

        y.child_value
        # RuntimeError: Foo has no attribute child_value

class Bar(Generic[T]):
    def bar(self) -> T: ...

class Baz(Bar[Self]): ...  # 循環参照が起こるのでダメ
```

`Self`を含む型エイリアスも許可されない。定義の外側で`Self`をサポートするのは、型チェッカーに特別な処理を必要とする可能性がある。
classの外側で`Self`を使うのは他の部分にも反することで、エイリアスの利便性を高めることは見合わない。

```python
TupleSelf = Tuple[Self, Self]  # だめ


class Alias:
    def return_tuple(self) -> TupleSelf:  # だめ
        return (self, self)
```

同じ理由で、メタクラスで使うことも許されていない。
メタクラスでは、異なるメソッドのシグネチャで違う型を参照する必要がある。例えば`__mul__`は戻り値の型`Self`は包含クラス`MyMetaclass`ではなくて実装の`Foo`を参照する。一方で`__new__`では戻り値の型`Self`はメタクラスを参照するので、混乱の元になる。

```python
class MyMetaclass(type):
    def __new__(cls, *args: Any) -> Self:  # MyMetaclassを返す
        return super().__new__(cls, *args)

    def __mul__(cls, count: int) -> list[Self]:  # Fooを返す
        return [cls()] * count

class Foo(metaclass=MyMetaclass): ...
```
:::

## 実装上の振る舞い

`Self`はsubscriptableではないので、`typing.NoReturn`と同様の実装になる

```python
@_SpecialForm
def Self(self, params):
    """Used to spell the type of "self" in classes.

    Example::

      from typing import Self

      class ReturnsSelf:
          def parse(self, data: bytes) -> Self:
              ...
              return self

    """
    raise TypeError(f"{self} is not subscriptable")
```

## リファレンス実装 (訳文ママ)

- Mypyの実装例: <https://github.com/Gobot1234/mypy/commit/ff779e8f30eed84b98c1d374c9d5666ac3b29b73>
- Pyright: v1.1.184で対応
- CPythonの実装: <https://github.com/python/typing/pull/933>

## 資料 (訳文ママ)

2016年ごろからMypyでも[同様の議論](https://github.com/python/mypy/issues/1212)が始まっていた。
しかし、そこで最終的に取られたアプローチは「以前は...」の例で示したような冗長な`typing.TypeVar`を使うものだった。
これについて議論している他のissueには、[Mypy#2354](https://github.com/python/mypy/issues/2354)がある。

PradeepはPyCon Typing Summit 2021で具体的な提案をしました。([動画](https://youtu.be/ld9rwCvGdhc?t=3260) [資料](https://drive.google.com/file/d/1x-qoDVY_OvLpIV1EwT7m3vm4HrgubHPG/view))
James は typing-sig で独自にこの提案を持ち出しました。[該当のメールスレッド](https://mail.python.org/archives/list/typing-sig@python.org/thread/SJAANGA2CWZ6D6TJ7KOPG7PZQC56K73S/#B2CBLQDHXQ5HMFUMS4VNY2D4YDCFT64Q)

TypeScriptには[`this`型](https://typescriptlang.org/docs/handbook/2/classes.html#this-types)があり、Rustには[`Self`型](https://doc.rust-lang.org/std/keyword.SelfTy.html)があります。

最後に、このPEPにフィードバックをくれた方たちに謝辞を述べます。
Jia Chen, Rebecca Chen, Sergei Lebedev, Kaylynn Morgan, Tuomas Suutari, Eric Traut, Alex Waygood, Shannon Zhu, and Никита Соболев

---

長すぎ!