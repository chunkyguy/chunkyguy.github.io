---
layout: post
title: Let's build a calculator
date: 2024-03-06 22:21 +0100
categories: js
published: true
---

Let's build a calculator!

I saw this challenge on the [frontendmentor website](https://www.frontendmentor.io/challenges/calculator-app-9lteq5N29) and the very next day while traveling to work I was trying to figure out how would I construct a calculator. I was thinking more like in terms of a real mechanical calculator rather than a software and then trying to figure out a circuit diagram. 

This is what I came up with:

![Calculator circuit diagram]({{site.url}}/assets/basic-calc/sketch.jpg)

Let me explain what is this mess. 

The data goes through 5 different stages, starting from **I**nput that the user provides with the user interface and then depending on the type of the input it might go to the **B**uffer or the **O**perations, both of which are string types. But any change to the **O**perations would trigger an update to the stored **R**esult which is a number type and most likely a floating type. The result is computed using the last stored value and the value in the buffer. And finally, every input should eventually refresh the text on **D**isplay.

So for any numeric input we just update our buffer. The special case is with `0` since we don't want multiple leading `0s`. Finally we concatenate the buffer and send the output string for display.  
```js
let buf = ["0"];

function handleInput(key) {
  switch (key) {
    // ....

    default:
      if (!"0123456789".includes(key)) return undefined;
      // if the only digit in buffer is a 0 then replace it with the input
      if (buf.length === 1 && buf[0] === "0") {
        buf[0] = key;
      } else {
        buf.push(key);
      }
      return buf.join("");
  }
}
```

If the input is a command type (like `+-*/=`) then we compute the result with the value in buffer and the last result and the last cached command. Remember that `+` operation is actually performed after the user presses the `=` button. For example:

| Input | Cache | Display |
|-------|-------|---------|
| 5     |       | 5       |
| +     | +     | 5       |
| 3     | +     | 5       |
| =     | =     | 8       |

Another caveat is to avoid keeping a default result value like say a `0`. As this might not work for some operations like `*` where then anything that multiplies with `0` becomes `0`. It's better to just keep it `undefined` initially.
```js
let res = undefined;
let fn = undefined;

const add = (res, curr) => res + curr;
const sub = (res, curr) => res - curr;
const mul = (res, curr) => res * curr;
const div = (res, curr) => res / curr;
const eql = (res, curr) => res;

function computeResult(next) {
  let curr = parseFloat(buf.join(""));
  curr = isNaN(curr) ? 0 : curr;

  if (res === undefined) {
    res = curr;
  } else {
    res = fn(res, curr);
  }

  fn = next;

  return res.toString();
}

function handleInput(key) {
  switch (key) {
    case "+":
      return computeResult(add);

    case "-":
      return computeResult(sub);

    case "*":
      return computeResult(mul);

    case "/":
      return computeResult(div);

    case "=":
      return computeResult(eql);

    // ...
  }
}
```

Then there is the special case of handling `.` key. In this case we simply add the `.` to the buffer is it isn't there already.

```js
function handleInput(key) {
  switch (key) {
    case ".":
      if (!buf.includes(".")) {
        buf.push(".");
      }
      return buf.join("");

    // ...
  }
}
```

Similar buffer manipulations can be performed for other commands like when user presses the `DEL` button which just pops the buffer.

```js
function handleInput(key) {
  switch (key) {
    case "Backspace":
      buf.pop();
      if (buf.length === 0) {
        buf = ["0"];
      }
      return buf.join("");

    // ...
  }
}
```

You must have realized that although internally we keep the computed result in `res` as `number` but we return back a `string`. This is to avoid dealing with the complexities that comes when dealing with floating point numbers. The drawing output to display is then simply the matter of string manipulation, and in my personal case since my calculator can not fit more than 13 digits I can simply not display more than 13 chars.

```js
const outFld = document.querySelector("#output");

function draw(text) {
  if (text === undefined) return;
  outFld.innerHTML = text.slice(0, 13);
}
```

If you want you can play with the final result [here](https://whackylabs.com/frontendmentor/calculator-app/) or go play with the [source code](https://github.com/chunkyguy/frontendmentor/tree/main/calculator-app)