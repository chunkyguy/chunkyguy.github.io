---
layout: post
title: Getting started with LiteRT (Tensorflow Lite)
date: 2024-11-04 21:00 +0200
categories: swift tensorflow ml
published: true
---

So Google recently renamed TensorflowLite to LiteRT. And yes that was a genius move indeed. Because now for the time in my life I actually want to try TFLite ... yea, I mean LiteRT.

![Setup](/assets/hello-litert/meme.jpg)

In the real world you'd ideally think like a regular ML developer and start with discovering a dataset that then you'd use to train a model. 
And then as a next step you'd invent a problem that could be solved by your trained model. 

But for this experiment we are going to keep things simple and build the hello world of Machine Learning universe, the Dogs vs Cat exercise.

### Setup
So coincidentally enough I've found a trained model from [a flutter project](https://github.com/offfahad/cat-and-dog-detector-app-flutter) that can tell if a given photo is of a dog or a cat.

Nice! So then following the instructions at the [official getting started docs](https://ai.google.dev/edge/litert/ios/quickstart) we need to add tensorflowlite as a dependency to our project.

```
use_frameworks!
# pod 'TensorFlowLiteSwift'
pod 'TensorFlowLiteSwift', '~> 0.0.1-nightly', :subspecs => ['CoreML', 'Metal']
```

In case you're wondering why are we using the nightly build and not the stable release. It's because as of today the latest stable release of `TensorFlowLiteC` [won't work on iOS simulator](https://github.com/tensorflow/tensorflow/issues/47400) but according to the [last comment](https://github.com/tensorflow/tensorflow/issues/47400#issuecomment-1126611267) on the issue looks like `TensorFlowLiteC` is now also shipped as xcframework but only in the nightly releases.

And then while we wait for the `pod install` to finish we can shamelessly rip sample images of various dogs and cats from the [Introduction to TensorFlow Lite](https://www.udacity.com/enrollment/ud190) udacity course as our test set. 

And we are all set!

### Building the app

For the UI we just need a simple image view, a label and a button. The button obviously would reveal the answer and then randomly load the next image.

![Setup](/assets/hello-litert/setup.png)

Now for the interesting bit. First we need a `PetClassifier` that takes in an image and returns a text.

```swift
class PetClassifier {

  init?(named: String, labels: [String]) {
    // ...
  }

  
  func labelForImage(_ image: UIImage) -> String? {
    // ...
  }
}
```

And then in the UI layer we can use our `PetClassifier` to update the label when tapped on the 'Evaluate' button.

```swift
class ViewController: UIViewController {

  var classifier: PetClassifier?
  @IBOutlet var answerLabel: UILabel!

  override func viewDidLoad() {
    super.viewDidLoad()
    classifier = PetClassifier(named: "dogvscat", labels: ["Cat", "Dog"])
  }

  @IBAction func handleTap() {
    answerLabel.text = classifier?.labelForImage(selectedImage) ?? "Potato"
  }
}
```

Finally, loading the model is pretty easy. We just need to instantiate `Interpreter` with the path to our tflite model.

```swift
class PetClassifier {
  let interpreter: Interpreter
  let labels: [String]
    
  init?(named: String, labels: [String]) {
    guard let modelPath = Bundle.main.path(forResource: named, ofType: "tflite") else {
      return nil
    }
    
    do {
      var options = Interpreter.Options()
      options.threadCount = Self.threadCount
      interpreter = try Interpreter(modelPath: modelPath, options: options)
      self.labels = labels
    } catch {
      print(error)
      return nil
    }
  }

  // ...
}
```

### Invoking the model

To get the answer from the model is a 4 step process:
  1. Prepare input
  2. Send input data
  3. Read output data
  4. Parse output

```swift
/*
 * 1. Prepare input
 */
// user provided image
let image: UIImage 
// image size used for training model
let inputWidth = 224
let inputHeight = 224

// convert image to pixel buffer for further manipulation
let pixelBuffer = ImageUtils.pixelBufferCreate(image: image)
// crop image to size used for training model
let scaledPixelBuffer = ImageUtils.pixelBufferCreateWith(
  pixelBuffer: pixelBuffer,
  resizedTo: CGSize(width: Self.inputWidth, height: Self.inputHeight)
)
// Remove the alpha component from the image buffer to get the RGB data.
let rgbData = ImageUtils.pixelBufferCreateRGBData(
    pixelBuffer: scaledPixelBuffer,
    byteCount: Self.inputWidth * Self.inputHeight * 3
)

/*
 * 2. Send input data 
 */
interpreter.allocateTensors()      
interpreter.copy(rgbData, toInputAt: 0)
interpreter.invoke()

/*
 * 3. Read output data
 */
let outputTensor = try interpreter.output(at: 0)
let results: [Float] = outputTensor.data.withUnsafeBytes {
  Array($0.bindMemory(to: Float.self))
}

/*
 * 4. Parse output
 */      
// Create a zipped array of tuples [(labelIndex: Int, confidence: Float)].
// Sort the zipped results by confidence value
let inferences = zip(labels.indices, results)
  .sorted { $0.1 > $1.1 }
  .map { (label: labels[$0.0], confidence: $0.1) }
      
let bestInference = inferences.first
```

And there you have it. That is how offload your brain to your computer.

![Setup](/assets/hello-litert/final.png)

The `ImageUtils` from this experiment are available [here](https://gist.github.com/chunkyguy/1e5244c4dcee8436d700f99380629989). But there are probably better libraries for these operations. For example, the [CoreMLHelpers](https://github.com/hollance/CoreMLHelpers)

## References
- [TensorFlow Lite is now LiteRT](https://developers.googleblog.com/en/tensorflow-lite-is-now-litert/)
- [Kaggle](https://www.kaggle.com/models?framework=tfLite)
- [HuggingFace](https://huggingface.co/models?library=tflite)
- [LLM Inference guide for iOS](https://ai.google.dev/edge/mediapipe/solutions/genai/llm_inference/ios)