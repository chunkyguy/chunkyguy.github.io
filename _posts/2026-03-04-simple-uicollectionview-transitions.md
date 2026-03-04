---
layout: post
title: This one liner to add beautiful transition between UICollectionViews
date: 2026-03-04 21:58 +0100
categories: swift ios uikit
published: true
---

UIKit comes with a bag full of techniques to build mind blowing animations and transitions but this one liner has to my favorite.

```swift
useLayoutToLayoutNavigationTransitions = true
```

So how do you make it work? Simple first you need to have 2 `UICollectionViewController` instances in a `UINavigationController`. Then just before you push the second screen set the `useLayoutToLayoutNavigationTransitions = true`. And done!

<video controls width="300">
  <source src="/assets/2026-03-04-simple-uicollectionview-transitions/demo.mp4" type="video/mp4" />
</video>

Did I say 2 `UICollectionViewController`? Sorry I mean one. The only thing that is changing is the `collectionViewLayout` that is powering the `UICollectionView`.

```swift
class TileLayout: UICollectionViewFlowLayout {
  let scale: CGFloat

  init(size: CGSize, scale: CGFloat) {
    self.scale = scale
    super.init()
    let edge = (size.width * scale) - 12
    itemSize = CGSize(width: edge, height: edge)
    sectionInset = UIEdgeInsets(top: 8, left: 8, bottom: 8, right: 8)
    minimumInteritemSpacing = 0
    minimumLineSpacing = 8
  }

  required init?(coder: NSCoder) {
    fatalError("init(coder:) has not been implemented")
  }
}
```

```swift
struct DataSource {
  let colors: [UIColor]

  init(colors: [UIColor]) {
    self.colors = colors
  }

  init() {
    let count = 48
    let colors = (0..<count)
      .map { 1 - (CGFloat($0) / CGFloat(count)) }
      .map { UIColor(hue: $0, saturation: 0.7, brightness: 0.8, alpha: 1) }
    self.init(colors: colors)
  }
}
```

```swift
class ViewController: UICollectionViewController {
  static let cellId = "cell-id"

  let dataSource: DataSource
  private var selectedIndex: IndexPath?
  private var scale: CGFloat {
    (collectionViewLayout as? TileLayout)?.scale ?? 1
  }

  init(
    dataSource: DataSource,
    selectedIndex: IndexPath?,
    collectionViewLayout: UICollectionViewLayout
  ) {
    self.dataSource = dataSource
    self.selectedIndex = selectedIndex
    super.init(collectionViewLayout: collectionViewLayout)
  }

  required init?(coder: NSCoder) {
    fatalError("init(coder:) has not been implemented")
  }

  override func viewDidLoad() {
    super.viewDidLoad()
    title = "Zoom: \(scale)"
    collectionView.showsVerticalScrollIndicator = false
    collectionView.register(
      UICollectionViewCell.self,
      forCellWithReuseIdentifier: ViewController.cellId
    )
  }

  override func viewDidAppear(_ animated: Bool) {
    super.viewDidAppear(animated)
    if let selectedIndex {
      collectionView.scrollToItem(
        at: selectedIndex,
        at: .centeredVertically,
        animated: false
      )
    }
  }

  override func collectionView(
    _ collectionView: UICollectionView,
    numberOfItemsInSection section: Int
  ) -> Int {
    dataSource.colors.count
  }

  override func collectionView(
    _ collectionView: UICollectionView,
    cellForItemAt indexPath: IndexPath
  ) -> UICollectionViewCell {
    let cell = collectionView.dequeueReusableCell(
      withReuseIdentifier: ViewController.cellId,
      for: indexPath
    )
    cell.backgroundColor = dataSource.colors[indexPath.item]
    return cell
  }

  override func collectionView(
    _ collectionView: UICollectionView,
    didSelectItemAt indexPath: IndexPath
  ) {
    selectedIndex = indexPath
    let detailsVwCtrl = ViewController(
      dataSource: dataSource,
      selectedIndex: indexPath,
      collectionViewLayout: TileLayout(
        size: view.bounds.size,
        scale: min(scale * 2, 1.0)
      )
    )
    detailsVwCtrl.useLayoutToLayoutNavigationTransitions = true
    navigationController?.pushViewController(
      detailsVwCtrl,
      animated: true
    )
    collectionView.deselectItem(at: indexPath, animated: true)
  }
}
```

```swift
class SceneDelegate: UIResponder, UIWindowSceneDelegate {

  var window: UIWindow?

  func scene(
    _ scene: UIScene,
    willConnectTo session: UISceneSession,
    options connectionOptions: UIScene.ConnectionOptions
  ) {
    guard let windowScene = (scene as? UIWindowScene) else { return }

    window = UIWindow(windowScene: windowScene)
    window?.rootViewController = UINavigationController(
      rootViewController: ViewController(
        dataSource: DataSource(),
        selectedIndex: nil,
        collectionViewLayout: TileLayout(
          size: windowScene.screen.bounds.size,
          scale: 0.25
        )
      )
    )
    window?.makeKeyAndVisible()
  }

  // ...
}
```