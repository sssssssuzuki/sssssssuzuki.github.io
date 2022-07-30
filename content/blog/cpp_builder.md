+++
title = "C++で型を使ってコンパイル時に値がセットされているか確認するBuilderパターン"
date = 2022-07-30
[taxonomies]
categories = ["blog"]
tags = ["C++", "programming", "design pattern"]
+++

Builderパターンは便利だが, 普通にやると値が設定されたかどうかを実行時にしか検査できない.
できればこれをコンパイル時に検査してエラーを表示したい.

以下のようにBuilder型にフィールドがセットされているかどうかを表す型情報を持たせることで, この問題を解決できる.

```cpp
class Empty;
class Fill;

class Foo
{
public:
	template<typename T1, typename T2>
	class Builder
	{
	public:
		Builder() : _value1(0), _value2(0)
		{
		}

		Builder<Fill, T2>& value1(const double v)
		{
			static_assert(std::is_same_v<T1, Empty>, "value1 is already filled");

			_value1 = v;

			return *reinterpret_cast<Builder<Fill, T2>*>(this);
		}

		Builder<T1, Fill>& value2(const double v)
		{
			static_assert(std::is_same_v<T2, Empty>, "value2 is already filled");

			_value2 = v;

			return *reinterpret_cast<Builder<T1, Fill>*>(this);
		}

		Foo build() const
		{
			static_assert(std::is_same_v<T1, Fill>, "value1 is not filled");
			static_assert(std::is_same_v<T2, Fill>, "value2 is not filled");

			return { _value1, _value2 };
		}

	private:
		Builder(const double value1, const double value2) : _value1(value1), _value2(value2)
		{
		}

		double _value1;
		double _value2;
	};

	static Builder<Empty, Empty> builder() {
		return {};
	}

private:
	Foo(const double value1, const double value2) : _value1(value1), _value2(value2)
	{
	}

	double _value1;
	double _value2;
};
```

少し記述は冗長になってしまうが, 以下のように設定していない値があるとコンパイル時にエラーを出せる.
また, 二重に値を指定してもエラーが出る.

```cpp
	Foo foo = Foo::builder().value1(1.0).value2(2.0).build(); // Ok
	//Foo foo1 = Foo::builder().value2(2.0).build(); // Error: 'value1 is not filled'	
	//Foo foo2 = Foo::builder().value1(1.0).build(); // Error: 'value2 is not filled'	
	//Foo foo3 = Foo::builder().value1(1.0).value1(1.0).value2(2.0).build(); // Error: 'value1 is already filled'	
```

SFINAEでも同等のことはできるとおもうけど, `static_assert`のほうがエラーメッセージがわかりやすくて良い.
