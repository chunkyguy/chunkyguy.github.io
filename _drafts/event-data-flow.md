# Dealing with nil

### Swift crash

```swift
// In some other module outside of reach
func f(_ str: String? = nil) -> String {
  return str!
}

func crashAtRuntime() {
  let s = f()

  // can't apply optional chaining
  // let c = s?.count ?? 0

  let c = s.count
  print("len: \(c)")
}
```