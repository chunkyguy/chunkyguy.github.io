---
layout: post
title: JSON vs Plist
date: 2026-05-14 18:34 +0200
categories: swift ios json plist
published: true
---

I recently watched an old WWDC video. The actual content of the video is not relevant but there was this one frame in the video that actually caught my eye. It was about parsing speeds between JSON, XML and Plist 

![wwdc-screenshot](/assets/2026-05-14-json-vs-plist/01.png)

According to the video parsing plist is ~22 times faster than json and ~43 times faster than parsing XML. Since the video was from 2010 and the world has really moved on ever since. So I had validate for myself how much of this is true in the year 2026.

I found a JSON vs Plist [benchmarking project](https://github.com/danielpetroianu/FileDeserializeBenchmarking) and cloned it - mainly for the data set. Then I put 4 parsers provided by Apple today in the arena:

1. `JSONSerialization`
```swift
let object = try! JSONSerialization.jsonObject(with: data) as! NSDictionary
let list = List(object)
```

2. `JSONDecoder`
```swift
let list = try! JSONDecoder().decode(List.self, from: data)
```

3. `PropertyListSerialization`
```swift
let object = try! PropertyListSerialization.propertyList(from: data, format: nil) as! NSDictionary
let list = List(object)!
```

4. `PropertyListDecoder`
```swift
let list = try! PropertyListDecoder().decode(List.self, from: data)
```

These are the data objects:

```swift
struct User: Codable {
  var uid: Int
  var firstName: String
  var lastName: String

  init(uid: Int, firstName: String, lastName: String) {
    self.uid = uid
    self.firstName = firstName
    self.lastName = lastName
  }

  init(_ dict: NSDictionary) {
    self.uid = dict["uid"] as! Int
    self.firstName = dict["firstName"] as! String
    self.lastName = dict["lastName"] as! String
  }
}

struct List: Codable {
  var List: [User]

  init(List: [User]) {
    self.List = List
  }

  init(_ dict: NSDictionary) {
    let userDicts = dict["List"] as! [NSDictionary]
    self.List = userDicts.map { User($0) }
  }
}
```

And these are the sample data:

```json
{ 
  "List":[
    {
        "uid": 372796,
        "firstName": "Lupita",
        "lastName": "Jokisch"
    }
  ]
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>List</key>
	<array>
		<dict>
			<key>firstName</key>
			<string>Lupita</string>
			<key>lastName</key>
			<string>Jokisch</string>
			<key>uid</key>
			<integer>372796</integer>
		</dict>
	</array>
</dict>
</plist>
```

Notice that I'm not only benchmarking the parsing time but also the mapping of the data to the data models. Reason being that the modern parsers like `JSONDecoder` return back the model object unlike the old school parsers like `JSONSerialization` that return back the "raw" dictionary. And in any useful scenario we would have to always map the raw dictionary to the data model.

Also to be noted that I used the Xcode build setting `PLIST_FILE_OUTPUT_FORMAT = binary` which automatically converts the xml based plist to binary plist. Otherwise the plist are always slower than json.

This is the result I found for one of the sample sets:

```
-========== START TESTING ==========-
iterations : 10


fileSize : 1x:

| Parser             | Time (µs) |
|--------------------|-----------|
| JSONSerialization  | 38        |
| JSONDecoder        | 38        |
| PListSerialization | 26        |
| PListDecoder       | 57        |


fileSize : 10x:

| Parser             | Time (µs) |
|--------------------|-----------|
| JSONSerialization  | 126       |
| JSONDecoder        | 117       |
| PListSerialization | 111       |
| PListDecoder       | 267       |


fileSize : 100x:

| Parser             | Time (µs) |
|--------------------|-----------|
| JSONSerialization  | 1125      |
| JSONDecoder        | 963       |
| PListSerialization | 965       |
| PListDecoder       | 2402      |


fileSize : 1000x:

| Parser             | Time (µs) |
|--------------------|-----------|
| JSONSerialization  | 11049     |
| JSONDecoder        | 9468      |
| PListSerialization | 10289     |
| PListDecoder       | 27021     |


fileSize : 10000x:

| Parser             | Time (µs) |
|--------------------|-----------|
| JSONSerialization  | 113883    |
| JSONDecoder        | 96854     |
| PListSerialization | 89497     |
| PListDecoder       | 231899    |


fileSize : 100000x:

| Parser             | Time (µs) |
|--------------------|-----------|
| JSONSerialization  | 1133721   |
| JSONDecoder        | 926670    |
| PListSerialization | 843694    |
| PListDecoder       | 2309075   |


-========== TEST ENDED ==========-
```

`PropertyListSerialization` turned out to be the winner in most of the categories but the `JSONDecoder` always came very close followed by `JSONSerialization` and then the last one was always `PListDecoder`.

The source code for this experiment is available here: [github.com/chunkyguy/FileDeserializeBenchmarking](https://github.com/chunkyguy/FileDeserializeBenchmarking)