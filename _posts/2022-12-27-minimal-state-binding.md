---
layout: post
title:  "A minimal implementation of State and Binding with Swift"
date:   2022-12-27 22:10:00 +0200
categories: architecture combine swift ios
published: true
---
One of my most favorite thing about SwiftUI is neither the declarative API nor the mystical fancy syntax but rather the philosophy of one directional flow of data and event. I've been trying to put that philosophy into practice for quite a while now with whatever framework I use.

The core concepts of the philosophy are basically thinking about who owns the data and who handles the events. And once you've figured out these decisions then the rest of the plumbing can be done with whatever tools you have handy. Could be `RxSwift`, `Combine`, completion handlers, notification center or even `KVO`.  The cherry on the top could be fancy syntactic sugar with `@propertyWrapper` or `@dynamicMemberLookup`.

From SwiftUI I like the idea of using `@State` and `@Binding` to give a clear hint on who owns the data and who is just borrowing it. With that in mind I came up with this minimal `State` and `Binding` constructs:

```swift
@dynamicMemberLookup
class Variable<T> {
    var value: T {
        get { sub.value }
        set { sub.value = newValue }
    }
    
    var stream: AnyPublisher<T, Never> {
        return sub.eraseToAnyPublisher()
    }
    
    subscript<P>(dynamicMember keyPath: WritableKeyPath<T, P>) -> P {
        get { sub.value[keyPath: keyPath] }
        set { sub.value[keyPath: keyPath] = newValue }
    }
    
    fileprivate let sub: CurrentValueSubject<T, Never>
    
    init(_ sub: CurrentValueSubject<T, Never>) {
        self.sub = sub
    }
}

class State<T>: Variable<T> {
    var binding: Binding<T> {
        return Binding(self)
    }

    init(_ value: T) {
        super.init(CurrentValueSubject(value))
    }
}

class Binding<T>: Variable<T> {
    init(_ state: State<T>) {
        super.init(state.sub)
    }
}
```

The idea is that the `State` can be initialized with the data to indicate that it owns the data, while `Binding` can not. So the only way to create a `Binding` object is from a `State`. All the common functionality like reading and writing data is moved to a shared base class `Variable` which apart from exposing the `value` also provides a `stream` that be used to listen to changes and also supports `dynamicMemberLookup` to provide a shortcut to accessing properties of the wrapped type.

Here's an example to illustrate the usage:

```swift
struct Story {
    var text: String?
}
```

```swift
class ViewController: UIViewController {
    // owns the data
    let story = State(Story())

    let label = UILabel(frame: .zero)
    var cancellables = Set<AnyCancellable>()

    override func viewDidLoad() {
        super.viewDidLoad()
        playground.run()
        view.addSubview(label)
        label.backgroundColor = .white
        label.textAlignment = .center
        view.backgroundColor = .gray

        // update the word count at every keystroke
        story.stream
            .map(\.text)
            .map { $0?.count ?? 0}
            .map { "Words: \($0)" }
            .assign(to: \.title, on: self)
            .store(in: &cancellables)
        
        navigationItem.rightBarButtonItem = UIBarButtonItem(barButtonSystemItem: .edit,
                                                            target: self,
                                                            action: #selector(onEdit))
    }
    
    override func viewDidLayoutSubviews() {
        super.viewDidLayoutSubviews()
        label.frame = CGRect(origin: .zero, size: CGSize(width: view.bounds.width - 40,
                                                         height: 100))
        label.center = view.center
    }
    
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        // update the content whenever view is displayed
        // notice that we're not using:
        //      label.text = story.value.text
        label.text = story.text
    }
    
    @objc func onEdit() {
        let textEditVwCtrl = TextEditViewController(story: story.binding)
        navigationController?.pushViewController(textEditVwCtrl, animated: true)
    }
}
```

```swift
class TextEditViewController: UIViewController, UITextViewDelegate {
    // borrows the data with a read-write access
    private let story: Binding<Story>

    init(story: Binding<Story>) {
        self.story = story
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    let textVw = UITextView(frame: .zero)

    override func viewDidLoad() {
        super.viewDidLoad()
        view.addSubview(textVw)
        textVw.text = story.text
        textVw.delegate = self
        title = "Story Editor"
        view.backgroundColor = .white
    }

    override func viewDidLayoutSubviews() {
        super.viewDidLayoutSubviews()
        textVw.frame = view.bounds
    }

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        textVw.becomeFirstResponder()
    }

    func textViewDidChange(_ textView: UITextView) {
        // notice that we're not using:
        //      story.value.text = textView.text
        story.text = textView.text
    }
}
```

I've a feeling the syntax could be made even fancier with the help of `@propertyWrapper` to look something like what SwiftUI does with `@State var story = Story()` and 
`@Binding var storyCopy = $story` but maybe I'll leave that as the task for the reader.
