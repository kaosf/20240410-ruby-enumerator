---
marp: true
class: invert
---
<!-- headingDivider: 1 -->

# 外部イテレータ と 多重ループの平坦化

# Author

ka

![icon](https://www.kaosfield.net/icon.webp)

[kaosfield](https://www.kaosfield.net/)

![kaosfield QR](kaosfield-qr.svg)

# License

CC BY-NC-SA 4.0

[![CC BY-NC-SA 4.0](https://licensebuttons.net/l/by-nc-sa/4.0/88x31.png)](http://creativecommons.org/licenses/by-nc-sa/4.0/)

Copyright (C) 2024 ka

# About this contents

GitHub Pages: [https://kaosf.github.io/20240410-ruby-enumerator](https://kaosf.github.io/20240410-ruby-enumerator)

Repository: [kaosf/20240410-ruby-enumerator - GitHub](https://github.com/kaosf/20240410-ruby-enumerator)

![GitHub Pages QR](gh-pages-qr.svg)

# RubyのEnumerator

```rb
e = Enumerator.new do |y|
  p y.class
  y.yield 1
end

result = e.next
#=> Enumerator::Yielder

p result
#=> 1
```

# #next

```rb
e = Enumerator.new do |y|
  y.yield 1
  y.yield 2
  y.yield 3
end

p e.next
#=> 1
p e.next
#=> 2
p e.next
#=> 3
```

# #each

```rb
e = Enumerator.new do |y|
  y.yield 1
  y.yield 2
  y.yield 3
end

e.each do
  p _1
end
#=> 1
#=> 2
#=> 3
```

# Array#each

each メソッドは Enumerator クラスのインスタンスを返す

```rb
e = [1, 2, 3].each
p e.class
#=> Enumerator

p e.next
#=> 1
p e.next
#=> 2
p e.next
#=> 3
```

# 内部イテレータと外部イテレータ 1

## 内部イテレータ

制御の面倒を見て貰う

```rb
[1, 2, 3].each { p _1 }
```

# 内部イテレータと外部イテレータ 2

## 外部イテレータ

自分で制御の面倒を見る

```rb
e = [1, 2, 3].each

loop do
  p e.next
rescue
  break
end
```

# 多重ループの平坦化

```rb
xs = [1000, 2000]
ys = [100, 200]
zs = [10, 20]
ws = [1, 2]

xs.each do |x|
  ys.each do |y|
    zs.each do |z|
      ws.each do |w|
        p x + y + z + w
      end
    end
  end
end
```

#

```rb
xs = [1000, 2000]
ys = [100, 200]
zs = [10, 20]
ws = [1, 2]

xs_enumerator = Enumerator.new do |yielder|
  xs.each do |x|
    yielder.yield x
  end
end
xys_enumerator = Enumerator.new do |yielder|
  xs_enumerator.each do |x|
    ys.each do |y|
      yielder.yield x, y
    end
  end
end
xyzs_enumerator = Enumerator.new do |yielder|
  xys_enumerator.each do |x, y|
    zs.each do |z|
      yielder.yield x, y, z
    end
  end
end
xyzws_enumerator = Enumerator.new do |yielder|
  xyzs_enumerator.each do |x, y, z|
    ws.each do |w|
      yielder.yield x, y, z, w
    end
  end
end

xyzws_enumerator.each do |x, y, z, w|
  p x + y + z + w
end
```

# 注釈

結局ネストが深いままでは？

ネストが深くなったままなのが問題ではない

制御を自分で面倒見られるようになることでループがその外側と内側のループから独立することが出来た

どの階層のループもそれぞれ「外側から何を受け取るか」「内側に何を渡すか」だけに集中することが出来る構造になっている

#

例えば z のループの直前で x と y を用いた前処理を実施する必要が出た場合でも

```rb
xys_enumerator = Enumerator.new do |yielder|
  xs_enumerator.each do |x|
    ys.each do |y|
      do_something_with x, y
      yielder.yield x, y
    end
  end
end
```

#

あるいは

```rb
xyzs_enumerator = Enumerator.new do |yielder|
  xys_enumerator.each do |x, y|
    do_something_with x, y
    zs.each do |z|
      yielder.yield x, y, z
    end
  end
end
```

# その他

```rb
xs = [1000, 2000]
ys = [100, 200]
zs = [10, 20]
ws = [1, 2]

xyzws_enumerator = Enumerator.new do |yielder|
  xs.each do |x|
    ys.each do |y|
      zs.each do |z|
        ws.each do |w|
          yielder.yield x, y, z, w
        end
      end
    end
  end
end

xyzws_enumerator.each do |x, y, z, w|
  p x + y + z + w
end
```

# デメリット

レキシカルスコープが分散されてしまいクロージャの恩恵が得られないため必要な値はすべて明示的に yielder を介してやりとりしなければならない
