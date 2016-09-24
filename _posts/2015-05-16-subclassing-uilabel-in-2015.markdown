---
layout: post
title:  "Subclassing UILabel in 2015"
date:   2015-05-16 11:53:54 +0530
categories: uikit
---

`UILabel` is one the most smart view in `UIKit` family. It knows a lot
about itself. If you constraint it to a certain width, the UILabel can
calculate the height for itself. Another smart thing about `UILabel` is
that is saves you machine cycles by not redrawing its content unless
something is actually modified. And if you use `NSAttributedString`, you
can in fact draw a more sophisticated text content.

Given so many good things with a `UILabel`, it is very tempting to
subclass a `UILabel` to render our custom text content. Say if you’re
working on a game or app, say a chat messenger, you may like to have a
view where you render you text and have some decorations around it, like
a speech bubble with an optional image. And, [if you’ve ever worked on
such a task, you know how painful it actually
is!](https://twitter.com/madgarden/status/578189016031363072).

A `UILabel` is the perfect fit whenever you’ve a need to render some
text on screen. Recently had to undergo such a job of subclassing
`UILabel`, and it surprisingly hard to find a good source of information
that shows how to properly subclass `UILabel`. It might work for the
text data you have right now, but there are so many screen sizes and
edge cases possible. If you’re one of those person, I don’t want you to
discover this knowledge the hard way like I did, I’ll share the
knowledge I’ve acquired from my adventure.

Even if you’re not directly using `UILabel` but rendering your content
on an OpenGL canvas using Freetype library or something, the knowledge
of how `UILabel` works should be really helpful in designing your own
view. But, you can take my word for it, that rendering text is one of
the most challenging things in the UI problem domains. You have to
render all sort of glyphs, in all sort of writing directions with a lot
of kerning, ligatures and other text attributes. If you’re one of those
adventure seeker, you should definitely try building your own text
rendering system. Just for the inspiration, these are from the days when
I was building one:

> Fixing font rendering the long way
> [pic.twitter.com/hJtCBKLqBN](http://t.co/hJtCBKLqBN)
>
> — Sidharth Juyal (@chunkyguy) [June 26,
> 2013](https://twitter.com/chunkyguy/status/349802071815487488)

> One day work = FreeType font rendering
> [pic.twitter.com/FJMrcPPu](http://t.co/FJMrcPPu)
>
> — Sidharth Juyal (@chunkyguy) [February 6,
> 2013](https://twitter.com/chunkyguy/status/299153486690541568)

Someday, I should probably also share my experiences with building a
UILabel-like view from scratch, I mean on top of freetype. But, today we
just extend over the work `UILabel` already does for us. Let’s begin
with creating a new project.

We simply create a new project with a single view and add a
`RoundedRectLabel` class. Next, we write our basic code to render the
label on screen.

``` {.brush: .cpp; .title: .; .notranslate title=""}

class RoundedRectLabel: UILabel {

}



class ViewController: UIViewController {

    override func viewDidLoad() {

        super.viewDidLoad()

        

        view.backgroundColor = .lightGrayColor()

        

        let chatBubbleLabel = RoundedRectLabel()

        chatBubbleLabel.text = "No one expects the Spanish Inquisition! Our chief weapon is surprise. Fear and surprise. Two chief weapons, fear, surprise, and ruthless efficiency! Er, among our chief weapons are: fear, surprise, ruthless efficiency, and near fanatical devotion to the Pope! Um, I'll come in again."

        chatBubbleLabel.textColor = .blackColor()

        chatBubbleLabel.backgroundColor = .cyanColor()

        chatBubbleLabel.numberOfLines = 0

        chatBubbleLabel.textAlignment = .Justified

        view.addSubview(chatBubbleLabel)

        

        chatBubbleLabel.setTranslatesAutoresizingMaskIntoConstraints(false)

        view.addConstraint(NSLayoutConstraint(item: chatBubbleLabel, attribute: .CenterY, relatedBy: .Equal, toItem: view, attribute: .CenterY, multiplier: 1.0, constant: 0.0))

        view.addConstraints(NSLayoutConstraint.constraintsWithVisualFormat("H:|-[chatBubbleLabel]-|", options: NSLayoutFormatOptions(0), metrics: nil, views: ["chatBubbleLabel": chatBubbleLabel]))

    }

    

    override func prefersStatusBarHidden() -> Bool {

        return true

    }

}
```

[![](http://i.imgur.com/8ejJI06.png "source: imgur.com")](http://imgur.com/8ejJI06)

Now the two important methods to subclass within a `UILabel` are:

``` {.brush: .cpp; .title: .; .notranslate title=""}

    func textRectForBounds(bounds: CGRect, limitedToNumberOfLines numberOfLines: Int) -> CGRect

    func drawTextInRect(rect: CGRect)
```

The `textRectForBounds(_:limitedToNumberOfLines:)` provides us an
opportunity to update the drawing area of the text within the `UILabel`.
And the `drawTextInRect()` is for you to do you custom drawing within
the view.

If you notice, this is a little different than your usual UIView
subclassing, where you typically draw you content in `drawRect()`.
Another thing to keep in mind is that you typically also don’t want to
modify the `intrinsicContentSize()`. Overriding these two methods will
do whatever you want to do with you custom `UILabel`.

For sake of getting a deeper understanding, let’s print out the order of
execution.

``` {.brush: .cpp; .title: .; .notranslate title=""}

class RoundedRectLabel: UILabel {

    

    override func textRectForBounds(bounds: CGRect, limitedToNumberOfLines numberOfLines: Int) -> CGRect {

        let textRect = super.textRectForBounds(bounds, limitedToNumberOfLines: numberOfLines)

        println("\(__FUNCTION__): \(textRect)")

        return textRect

    }

    

    override func drawTextInRect(rect: CGRect) {

        super.drawTextInRect(rect)

        println("\(__FUNCTION__): \(rect)")

    }



    override func intrinsicContentSize() -> CGSize {

        let size = super.intrinsicContentSize()

        println("\(__FUNCTION__): \(size)")

        return size

    }

}
```

Here’s the output:

``` {.brush: .cpp; .title: .; .notranslate title=""}

textRectForBounds(_:limitedToNumberOfLines:): (0.0, 0.0, 2122.5, 20.5)

intrinsicContentSize(): (2122.5, 20.5)

textRectForBounds(_:limitedToNumberOfLines:): (0.0, 0.0, 342.5, 142.0)

intrinsicContentSize(): (342.5, 142.0)

drawTextInRect: (0.0, 0.0, 343.0, 142.0)
```

The interesting thing to notice here is that whatever is calculated from
the `super.textRectForBounds(_:limitedToNumberOfLines:)` is also passed
on to the `intrinsicContentSize()`. Another interesting thing is that,
the methods are called multiple times. First time with some estimated
values by `UIKit` and second time after the values have been evaluated
accurately enough.

Not implementing the `intrinsicContentSize()` also means that, you don’t
need to `invalidateIntrinsicContentSize()` yourself. Since, the default
drawing mode of `UILabel` is `UIViewContentMode.Redraw`, the content is
redrawn automatically whenever the content updates. To confirm, if you
update the text after a while, you can see the drawing methods being
invoked again.

``` {.brush: .cpp; .title: .; .notranslate title=""}

class ViewController: UIViewController {

    override func viewDidLoad() {

        super.viewDidLoad()

        

        view.backgroundColor = .lightGrayColor()

        

        let chatBubbleLabel = RoundedRectLabel()

        chatBubbleLabel.text = "No one expects the Spanish Inquisition! Our chief weapon is surprise. Fear and surprise. Two chief weapons, fear, surprise, and ruthless efficiency! Er, among our chief weapons are: fear, surprise, ruthless efficiency, and near fanatical devotion to the Pope! Um, I'll come in again."

        chatBubbleLabel.textColor = .blackColor()

        chatBubbleLabel.backgroundColor = .cyanColor()

        chatBubbleLabel.numberOfLines = 0

        chatBubbleLabel.textAlignment = .Justified

        view.addSubview(chatBubbleLabel)

        

        chatBubbleLabel.setTranslatesAutoresizingMaskIntoConstraints(false)

        view.addConstraint(NSLayoutConstraint(item: chatBubbleLabel, attribute: .CenterY, relatedBy: .Equal, toItem: view, attribute: .CenterY, multiplier: 1.0, constant: 0.0))

        view.addConstraints(NSLayoutConstraint.constraintsWithVisualFormat("H:|-[chatBubbleLabel]-|", options: NSLayoutFormatOptions(0), metrics: nil, views: ["chatBubbleLabel": chatBubbleLabel]))

        

        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, Int64(NSEC_PER_SEC * 3)), dispatch_get_main_queue()) { () -> Void in

            println("---------")

            chatBubbleLabel.text = "All right, but apart from the sanitation, the medicine, education, wine, public order, irrigation, roads, the fresh-water system, and public health, what have the Romans ever done for us?"

        }

    }

    

    override func prefersStatusBarHidden() -> Bool {

        return true

    }

}
```

Output:

``` {.brush: .cpp; .title: .; .notranslate title=""}

textRectForBounds(_:limitedToNumberOfLines:): (0.0, 0.0, 2122.5, 20.5)

intrinsicContentSize(): (2122.5, 20.5)

textRectForBounds(_:limitedToNumberOfLines:): (0.0, 0.0, 342.5, 142.0)

intrinsicContentSize(): (342.5, 142.0)

drawTextInRect: (0.0, 0.0, 343.0, 142.0)

---------

textRectForBounds(_:limitedToNumberOfLines:): (0.0, 0.0, 1403.0, 20.5)

intrinsicContentSize(): (1403.0, 20.5)

textRectForBounds(_:limitedToNumberOfLines:): (0.0, 0.0, 323.5, 101.5)

intrinsicContentSize(): (323.5, 101.5)

drawTextInRect: (0.0, 0.0, 343.0, 101.5)
```

This is good, because it means if you’re subclassing a UILabel, you need
to focus on just these two methods and the `UILabel` will take care of
the rest. So, lets focus first on the `drawTextInRect()`.

The first thing is to calculate the edge insets or padding you wish to
give your label.

``` {.brush: .cpp; .title: .; .notranslate title=""}

    let edgeInsets = UIEdgeInsets(top: 10, left: 20, bottom: 10, right: 20)

    

    override func drawTextInRect(rect: CGRect) {

        let textRect = UIEdgeInsetsInsetRect(rect, edgeInsets)

        super.drawTextInRect(textRect)

    }
```

[![](http://i.imgur.com/s1QfIyp.png "source: imgur.com")](http://imgur.com/s1QfIyp)

As you can see, we’ve the required padding from all edges as we wanted,
but the text is truncated at the end. This is because the `UILabel`
currently is trying to draw the text within the original frame it had
calculated. So the next task is to provide the new drawing frame to
`UILabel`. This is done within
`textRectForBounds(_:limitedToNumberOfLines:)`. The important question
is how do we calculate what frame size do we need to draw the entire
text content?

The `NSString` has a handy extension that does this calculations. The
method we’re particularly interested in is

``` {.brush: .cpp; .title: .; .notranslate title=""}

func boundingRectWithSize(size: CGSize, options: NSStringDrawingOptions, attributes: [NSObject : AnyObject]!, context: NSStringDrawingContext!) -> CGRect
```

We just need to provide this method our estimated size of drawing area
and other rendering attributes, and it will return the actual frame we
need to render this text. This brings us to the part II of the problem:
How do we calculate the estimated size of the drawing area?

Let’s see, we know the width that we wish to draw in.

``` {.brush: .cpp; .title: .; .notranslate title=""}

let estimatedWidth = CGRectGetWidth(textRect) - (edgeInsets.left + edgeInsets.right)
```

But, `UILabel` passes the same original width to
`textRectForBounds(_:limitedToNumberOfLines:)` and `drawTextInRect()`.
So, this means if the original width of the `UILabel` is 100 pts and we
wish to have a padding of 10 pts from all edges. If `NSString` API
calculates the required height for width = 80, is lets say it’s 200. The
size passed down to `drawTextInRect()` is 100×200, where we again shrink
the size down to 80×180. In order to compensate for this second
clipping, we must calculate the height for an 2 times smaller width.

``` {.brush: .cpp; .title: .; .notranslate title=""}

let estimatedWidth = CGRectGetWidth(rect) - (2 * (edgeInsets.left + edgeInsets.right))
```

But what about the height? Don’t worry, this is just an estimated
height, we can provide a very high value, and hope for the best.

``` {.brush: .cpp; .title: .; .notranslate title=""}

let estimatedWidth = CGRectGetWidth(rect) - (2 * (edgeInsets.left + edgeInsets.right))

let estimatedHeight = CGFloat.max

let calculatedFrame = NSString(string: text).boundingRectWithSize(CGSize(width: estimatedWidth, height: estimatedHeight), options: .UsesLineFragmentOrigin, attributes: [NSFontAttributeName: font], context: nil)
```

Next, remember, the `boundingRectWithSize(...)` will try to wrap our
text in as less space as possible, because that is the default behavior.
So, we need to explicitly provide the extra top and bottom padding to
the size calculated.

``` {.brush: .cpp; .title: .; .notranslate title=""}

let calculatedWidth = ceil(CGRectGetWidth(calculatedFrame))

let calculatedHeight = ceil(CGRectGetHeight(calculatedFrame))

let finalHeight = (calculatedHeight + edgeInsets.top + edgeInsets.bottom)

rect.size = CGSize(width: calculatedWidth, height: finalHeight)
```

The `ceil()` should raise fractional mathematical value to a renderable
screen value. The entire code is below.

``` {.brush: .cpp; .title: .; .notranslate title=""}

class RoundedRectLabel: UILabel {

    

    let edgeInsets = UIEdgeInsets(top: 10, left: 20, bottom: 10, right: 20)

    

    override func textRectForBounds(bounds: CGRect, limitedToNumberOfLines numberOfLines: Int) -> CGRect {

        var rect = super.textRectForBounds(bounds, limitedToNumberOfLines: numberOfLines)



        if let text = text {

            let estimatedWidth = CGRectGetWidth(rect) - (2 * (edgeInsets.left + edgeInsets.right))

            let estimatedHeight = CGFloat.max

            let calculatedFrame = NSString(string: text).boundingRectWithSize(CGSize(width: estimatedWidth, height: estimatedHeight), options: .UsesLineFragmentOrigin, attributes: [NSFontAttributeName: font], context: nil)

            let calculatedWidth = ceil(CGRectGetWidth(calculatedFrame))

            let calculatedHeight = ceil(CGRectGetHeight(calculatedFrame))

            let finalHeight = (calculatedHeight + edgeInsets.top + edgeInsets.bottom)

            rect.size = CGSize(width: calculatedWidth, height: finalHeight)

        }

        

        return rect

    }

    

    override func drawTextInRect(rect: CGRect) {

        let textRect = UIEdgeInsetsInsetRect(rect, edgeInsets)

        super.drawTextInRect(textRect)

    }

}
```

[![](http://i.imgur.com/JEY3NfN.png "source: imgur.com")](http://imgur.com/JEY3NfN)

To finish it off, we just need some custom drawing code. Feel free to
draw whatever you like.

``` {.brush: .cpp; .title: .; .notranslate title=""}

    override func drawTextInRect(rect: CGRect) {

        

        UIColor.cyanColor().setFill()

        UIColor.blackColor().setStroke()

        

        let edgePath = UIBezierPath(roundedRect: rect, cornerRadius: 50)

        edgePath.lineWidth = 5.0

        edgePath.lineJoinStyle = kCGLineJoinRound

        edgePath.fill()

        edgePath.stroke()

        

        let textRect = UIEdgeInsetsInsetRect(rect, edgeInsets)

        super.drawTextInRect(textRect)

    }
```

[![](http://i.imgur.com/3aPDAxf.png "source: imgur.com")](http://imgur.com/3aPDAxf)

[![](http://i.imgur.com/XlIJB5y.png "source: imgur.com")](http://imgur.com/XlIJB5y)

The entire code is also available at github.
<https://github.com/chunkyguy/RoundedRectLabel>.

