---
layout: single
title:  "[Laravel] Collection을 여러 기준으로 정렬하기"
excerpt: "Collection 정렬 시 주의할 점"
date:   2020-08-08 12:00:00 +0900
categories: laravel collection
---

{: .notice--info}
**해당 이슈는 이미 Laravel GitHub 레포지토리에 등록되어 있습니다.**  
[https://github.com/laravel/ideas/issues/11](https://github.com/laravel/ideas/issues/11)

{: .notice}
> 테스트 환경  
> * PHP 7.3  
> * Laravel 6

# Collection을 여러 번 정렬하면 의도치 않은 결과가 나올 수 있다
Laravel의 [Collection](https://laravel.com/api/6.x/Illuminate/Support/Collection.html) 클래스는 기본적으로 여러 정렬 메소드를 포함하고 있습니다.

* [`sort(callable $callback = null)`](https://laravel.com/api/6.x/Illuminate/Support/Collection.html#method_sort)
* [`sortBy(callable|string $callback, int $options = SORT_REGULAR, bool $descending = false)`](https://laravel.com/api/6.x/Illuminate/Support/Collection.html#method_sortBy)
* [`sortByDesc(callable|string $callback, int $options = SORT_REGULAR)`](https://laravel.com/api/6.x/Illuminate/Support/Collection.html#method_sortByDesc)
* [`sortKeys(int $options = SORT_REGULAR, bool $descending = false)`](https://laravel.com/api/6.x/Illuminate/Support/Collection.html#method_sortKeys)
* [`sortKeysDesc(int $options = SORT_REGULAR)`](https://laravel.com/api/6.x/Illuminate/Support/Collection.html#method_sortKeysDesc)

그렇다면 만약 다음과 같은 Collection이 있다고 가정했을 때

{% highlight php %}
>>> $collection = Illuminate\Support\Collection::times(5, function ($number) {
...     return ['first' => (int)($number / 5), 'second' => $number % 5];
... });
=> Illuminate\Support\Collection {#3040
     all: [
       [
         "first" => 0,
         "second" => 1,
       ],
       [
         "first" => 0,
         "second" => 2,
       ],
       [
         "first" => 0,
         "second" => 3,
       ],
       [
         "first" => 0,
         "second" => 4,
       ],
       [
         "first" => 1,
         "second" => 0,
       ],
     ],
   }
{% endhighlight %}

이 Collection을 여러 기준을 이용해 정렬한다면 아마 Query Builder와 같은 방식으로 다음과 같이 정렬을 수행하면 될 거라고 생각할 것입니다.

{% highlight php %}
$collection->sortBy('first')->sortBy('second');
{% endhighlight %}

하지만 실제 정렬을 수행한 결과는 의도한 결과와는 다를 수도 있습니다.

{% highlight php %}
>>> $collection->sortBy('first')->sortBy('second');
=> Illuminate\Support\Collection {#3042
     all: [
       4 => [
         "first" => 1,
         "second" => 0,
       ],
       0 => [
         "first" => 0,
         "second" => 1,
       ],
       1 => [
         "first" => 0,
         "second" => 2,
       ],
       2 => [
         "first" => 0,
         "second" => 3,
       ],
       3 => [
         "first" => 0,
         "second" => 4,
       ],
     ],
   }
{% endhighlight %}

이처럼 `first` 필드로 정렬한 결과는 사라지고 `second` 필드로 정렬한 결과만 남았습니다.  
이는 Collection에서 정렬 메소드를 호출하면 매번 독립적으로 정렬을 수행하기 때문입니다.

여기서 두 정렬의 순서를 바꾼다면 어떤 일이 생길까요?

{% highlight php %}
>>> $collection->sortBy('second')->sortBy('first');
=> Illuminate\Support\Collection {#3047
     all: [
       1 => [
         "first" => 0,
         "second" => 2,
       ],
       0 => [
         "first" => 0,
         "second" => 1,
       ],
       2 => [
         "first" => 0,
         "second" => 3,
       ],
       3 => [
         "first" => 0,
         "second" => 4,
       ],
       4 => [
         "first" => 1,
         "second" => 0,
       ],
     ],
   }
{% endhighlight %}

분명히 `second` 필드로 먼저 정렬을 했는데도 `first`가 0인 항목들이 정렬되어있지 않습니다.  
이는 Collection의 정렬 메소드가 unstable sort를 수행하기 때문입니다.

## Collection의 정렬 메소드가 unstable한 이유
Collection의 정렬 메소드들은 내부적으로 PHP의 기본 정렬 함수를 사용하여 정렬합니다.

* [`uasort(array &$array, callable $value_compare_func): bool`](https://www.php.net/manual/en/function.uasort.php)
* [`asort(array &$array, int $sort_flags = SORT_REGULAR): bool`](https://www.php.net/manual/en/function.asort.php)
* [`arsort(array &$array, int $sort_flags = SORT_REGULAR): bool`](https://www.php.net/manual/en/function.arsort.php)
* [`ksort(array &$array, int $sort_flags = SORT_REGULAR): bool`](https://www.php.net/manual/en/function.ksort.php)
* [`krsort(array &$array, int $sort_flags = SORT_REGULAR): bool`](https://www.php.net/manual/en/function.krsort.php)

그리고 PHP의 기본 정렬 함수는 unstable한 정렬을 수행합니다.

> **Note:**  
> If two members compare as equal, their relative order in the sorted array is undefined.

다시 말하자면 두 항목의 우선순위가 서로 같다면, 정렬된 후 두 항목이 처음 순서를 유지한다는 것을 보장하지 않습니다.  

# 여러 기준으로 Collection을 정렬하기 위한 방법
이를 해결하려면 `sort()` 메소드에 콜백을 넣어서 우리가 원하는 기준대로 정렬하면 됩니다.  

## 모든 필드에 대해 기준이 있는 경우
`first` 필드로 먼저 오름차순 정렬한 뒤 만약 `first` 필드가 같다면 `second` 필드로 오름차순 정렬을 해보겠습니다.

{% highlight php %}
>>> $collection->sort(function ($a, $b) {
...     if ($a['first'] !== $b['first']) {
...         return $a['first'] <=> $b['first'];
...     }
...     return $a['second'] <=> $b['second'];
... });
=> Illuminate\Support\Collection {#2989
     all: [
       [
         "first" => 0,
         "second" => 1,
       ],
       [
         "first" => 0,
         "second" => 2,
       ],
       [
         "first" => 0,
         "second" => 3,
       ],
       [
         "first" => 0,
         "second" => 4,
       ],
       [
         "first" => 1,
         "second" => 0,
       ],
     ],
   }
{% endhighlight %}

위와 같이 의도한 대로 결과가 나오는 것을 볼 수 있습니다.

## 일부 필드에 대한 기준만 있는 경우
그런데 만약 위처럼 모든 필드에 대한 정렬 기준이 없다면, 기준이 없는 필드의 순서는 원래 순서와 같음이 보장되지 않을 것입니다.  
이를 해결하기 위해서는 각 항목의 위치를 항목에 추가해서 정렬 기준에 넣으면 될 것입니다.

본 예제를 위해 새로운 Collection을 만들어 보겠습니다.

{% highlight php %}
>>> $collection = Illuminate\Support\Collection::times(17, function ($number) {
...     return ['first' => (int)($number / 5), 'second' => $number % 5];
... });
=> Illuminate\Support\Collection {#3021
     all: [
       [
         "first" => 0,
         "second" => 1,
       ],
       [
         "first" => 0,
         "second" => 2,
       ],
       // ...중략
       [
         "first" => 3,
         "second" => 2,
       ],
     ],
   }
{% endhighlight %}

이를 단순히 `first` 필드를 기준으로 정렬하면 다음과 같습니다.

{% highlight php %}
>>> $collection->sortBy('first')
=> Illuminate\Support\Collection {#3066
     all: [
       0 => [
         "first" => 0,
         "second" => 1,
       ],
       2 => [
         "first" => 0,
         "second" => 3,
       ],
       // ...중략
       16 => [
         "first" => 3,
         "second" => 2,
       ],
     ],
   }
{% endhighlight %}

위와 같이 순서가 이상한 것을 볼 수 있습니다.  
이제 이 Collection을 제대로 정렬하기 위해 각 항목에 `position` 값을 넣어주겠습니다.

{% highlight php %}
>>> $tempCollection = $collection->pipe(function ($collection) {
...     $position = 0;
...     return $collection->map(function ($value) use (&$position) {
...         $item = compact('position', 'value');
...         $position++;
...         return $item;
...     });
... });
=> Illuminate\Support\Collection {#2990
     all: [
       [
         "position" => 0,
         "value" => [
           "first" => 0,
           "second" => 1,
         ],
       ],
       [
         "position" => 1,
         "value" => [
           "first" => 0,
           "second" => 2,
         ],
       ],
       // ...중략
       [
         "position" => 16,
         "value" => [
           "first" => 3,
           "second" => 2,
         ],
       ],
     ],
   }
{% endhighlight %}

이렇게 기존 항목은 `value` 항목으로 옮기고 새로 `position` 항목을 추가해 주었습니다.  
이제 정렬 기준에 `position` 항목을 추가해서 정렬하면 됩니다.

{% highlight php %}
>>> $collection = $tempCollection->sort(function ($a, $b) {
...     if ($a['value']['first'] !== $b['value']['first']) {
...         return $a['value']['first'] <=> $b['value']['first'];
...     }
...     return $a['position'] <=> $b['position'];
... })->pluck('value');
=> Illuminate\Support\Collection {#3068
     all: [
       [
         "first" => 0,
         "second" => 1,
       ],
       [
         "first" => 0,
         "second" => 2,
       ],
       // ...중략
       [
         "first" => 3,
         "second" => 2,
       ],
     ],
   }
{% endhighlight %}

위와 같이 정렬이 잘 된 것을 확인할 수 있습니다.
