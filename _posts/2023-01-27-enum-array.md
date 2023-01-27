---
layout: post
title:  "Enum Array"
date:   2023-01-27 21:30:00 +0200
categories: swift algorithm
published: true
---
So today I happened to need something which is like an `Array` but can't grow boundless. And I also need a named lookup, like what `Dictionary` does. But I don't want to deal with `Optional` types. I want it to be guaranteed to be populated with all the data during the `init` time, so lookups can never be `nil`. Does such a thing already exists? In Swift I mean. I guess not, so obviously the next step to make such a thing. I'm calling it `EnumArray`.

```swift
/// A sort of an array but also a dictionary with enum as key, and no optionals
struct EnumArray<K, V> where K: RawRepresentable, K: CaseIterable, K.RawValue == Int {
    private(set) var values: [V]

    private init(_ values: [V]) {
        self.values = values
        assert(values.count == K.allCases.count, "Corrupt data")
    }

    init(_ action: (K) -> V) {
        self.init(K.allCases.map(action))
    }

    subscript(key: K) -> V {
        get { values[key.rawValue] }
        mutating set { values[key.rawValue] = newValue }
    }
}
```

How do we use an `EnumArray`? Easy, first we create an `enum` backed by `Int`. Next we create our `EnumArray`. Then we use it.

```swift
enum Week: Int, CaseIterable {
    case mon = 0, tue, wed, thu, fri, sat, sun
}

typealias WeekPlan = EnumArray<Week, String>

var plan = WeekPlan { day in
    switch day {
    case .mon: return "Learn SwiftUI"
    case .tue: return "Learn SwiftUI"
    case .wed: return "Learn SwiftUI"
    case .thu: return "Learn SwiftUI"
    case .fri: return "Learn Jetpack Compose"
    case .sat: return "What am I doing?"
    case .sun: return "Learn SwiftUI"
    }
}

for day in Week.allCases {
    print("\(plan[day]) on Day \(day)")
}
```

And to update the only way is via a `Dictionary` like subscript based setter

```swift
func fixFriday() {
    plan[.fri] = "Learn SwiftUI"
}
```

So there, we get all the good things of all three worlds! Enjoy!