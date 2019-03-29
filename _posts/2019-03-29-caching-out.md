---
title:  "caching out"
date:   2019-03-29 00:00:00 -0800
excerpt: "exploring a correspondence between functions and key-value stores."
---

Caching is among the most popular optimization techniques in computer science, reducing computation time at the cost of memory space. In this post, we'll explore the relationship between functions and key-value stores as it pertains to caching.

A function `f` of type `(Input) -> Output` describes a complete mapping: for each value of type `Input`, `f` produces a corresponding value of type `Output`.

A key-value store `s` of type `[Key: Value]` describes a partial mapping: for each value of type `Key`, `s` _might_ hold a corresponding value of type `Value`. The type of such a lookup function, `(Key) -> Value?`, describes this relationship.

When used in tandem where `Input == Key` and `Output == Value`, a function can "fill the gaps" in a key-value store; if a value is absent from `s`, it can be produced using `f` instead.

A common caching pattern involves the use of a computationally expensive function together with a key-value store[^1] containing already-computed values. For example:

{% highlight swift %}
class SceneRenderer {
    private var renderedImages: [Scene: Image] = [:]

    func renderImage(for scene: Scene) -> Image {
        if let renderedImage = renderedImages[scene] {
            return renderedImage
        } else {
            let renderedImage = expensivelyDraw(scene)
            renderedImages[scene] = renderedImage
            return renderedImage
        }
    }
}
{% endhighlight %}

This pattern can be bundled into a class that summarizes this correspondence between functions and dictionaries, performing a computation once on-demand and caching its output:

{% highlight swift %}
final class CacheMap<Input: Hashable, Output> {
    private let transform: (Input) -> Output
    private var cache: [Input: Output]

    init(_ transform: @escaping (Input) -> Output) {
        self.transform = transform
        self.cache = [:]
    }

    subscript(input: Input) -> Output {
        if let cachedOutput = cache[input] {
            return cachedOutput
        } else {
            let output = transform(input)
            cache[input] = output
            return output
        }
    }
}
{% endhighlight %}

Implementing our previous example in terms of `CacheMap` is then trivial:

{% highlight swift %}
class SceneRenderer {
    let renderedImages = CacheMap(expensivelyDraw)

    func renderImage(for scene: Scene) -> Image {
        return renderedImages[scene]
    }
}
{% endhighlight %}

`CacheMap` also enables the definition of cache management operations, and more complex definitions may include parameters describing conditions for eviction. We'll elect to hone-in only on the simple case here.

When the set of input values is known to be small, clearing the cache may never be necessary. In these cases, `CacheMap`'s lookup function alone serves the purpose of the class instance. In essence, the written subscript function of type `(Input) -> Output` describes a [memoized](https://en.wikipedia.org/wiki/Memoization) version of the original transformation. We can write a function that describes this relationship:

{% highlight swift %}
func memoize<Input: Hashable, Output>(
    _ transform: @escaping (Input) -> Output
) -> (Input) -> Output {
    let cache = CacheMap(transform)
    return { cache[$0] }
}
{% endhighlight %}

`memoize` is a powerful function: given a function of type `(Input) -> Output`, it returns a function of the same type that caches the results of its computations. By capturing the cache in the returned closure, we retain its functionality despite never exposing it to the callee. Here lies the significance of the magnitude of the input set: this functional transformation forgoes the ability to manage the cache manually[^2].

`memoize`'s behavior is apparent when used with a function that performs a side effect:

{% highlight swift %}
let printIntOnce = memoize { (x: Int) in print(x) }
printIntOnce(1) // Prints '1'
printIntOnce(1) // Does not print
printIntOnce(2) // Prints '2'
{% endhighlight %}

In this example, `printIntOnce` is a function of type `(Int) -> Void`. When called with an integer for the first time, the function prints the input and caches the output (which, in this case, is always the empty tuple `()`). When called with the integer again, the function successfully looks up the cached return value `()` rather than invoking the function.


<br/>

---

<br/>

[^1]: Using Swift's `Dictionary` type keeps this example simple; this post does not focus on the internals of cache management. When developing on Apple platforms, `NSCache` warrants consideration for its auto-eviction policies.

[^2]: This drawback presents a case for a more complex cache implementation than provided by the given definition of `CacheMap`. Passing parameters to `memoize` defining (e.g.) limits on the cache size for use by the underlying cache would help alleviate this loss of functionality.
