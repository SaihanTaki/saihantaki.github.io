---
title: Test Hugo Capabilites
author: ["me"]
date: "2025-11-15"
description: "Testing hugo markdwon capabilites and Extensions"
summary: "hello"
tags: ["markdown", "extensions"]
ShowToc: true
TocOpen: false
draft: false
weight: 1
math: true
ShowBreadCrumbs: true
plotly: true 

cover:
  image: images/papermod-cover.png
  hiddenInList: true
---

---

This is the summary of the post

<!--more-->

---

## Blockquote with attribution

> Don't communicate by sharing memory, share memory by communicating.
>
> — <cite>Rob Pike[^1]</cite>

[^1]: The above quote is excerpted from Rob Pike's [talk](https://www.youtube.com/watch?v=PAAkCSZUG1c) during Gopherfest, November 18, 2015.


## Custom Shortcode: Tabbed View

{{< tabs >}}
{{< tab name="Python" >}}
```python
print("Hello, World!")
```
{{< /tab >}}
{{< tab name="JavaScript" >}}
```javascript
console.log("Hello, World!");
```
{{< /tab >}}
{{< tab name="Go" >}}
```go
package main

import "fmt"

func main() {
	// Array
	var a [3]int
	a[0] = 1
	a[1] = 2
	a[2] = 3
	fmt.Println("Array:", a)

	// Slice
	s := []int{10, 20, 30, 40}
	fmt.Println("Slice:", s)
	fmt.Println("Slice with append:", append(s, 50))
}
```
{{< /tab >}}
{{< tab name="Quote" >}}
> Don't communicate by sharing memory, share memory by communicating.
>
{{< /tab >}}
{{< /tabs >}}

{{< tabs >}}

{{< tab name="Tab 1" >}}
Content for Tab 1
{{< /tab >}}

{{< tab name="Tab 2" >}}
> Don't communicate by sharing memory, share memory by communicating.
>

{{< /tab >}}

{{< tab name="Go" >}}
```go
fmt.Println("Hello, World!")
```
{{< /tab >}}

{{< /tabs >}}

### marimo wasm iframe
{{< rawhtml >}}
<iframe
	src="/marimo-wasm-blog/index.html?embed=true&show-chrome=false"
	width="100%"
	height="500"
	frameborder="0"
></iframe>

{{< /rawhtml >}}


### Check color text 

This is a colour shortcode {{< colortext "Hello My name is Saihan" "#97E5D7" >}}.
This is a colour shortcode {{< colortext "Hello My name is Saihan" "aquamarine" >}}.

## Plotly 3d plot
{{< plotly json="/test_plotly_3d.json" height="400px" >}}

## plotly widget
{{< plotly json="/test_plotly_widget.json" height="800px" >}}

## Mermaid

```mermaid
sequenceDiagram
    participant Alice
    participant Bob
    Alice->>John: Hello John, how are you?
    loop Healthcheck
        John->>John: Fight against hypochondria
    end
    Note right of John: Rational thoughts <br/>prevail!
    John-->>Alice: Great!
    John->>Bob: How about you?
    Bob-->>John: Jolly good!
```

## GoAT Diagrams

```goat
      .               .                .               .--- 1          .-- 1     / 1
     / \              |                |           .---+            .-+         +
    /   \         .---+---.         .--+--.        |   '--- 2      |   '-- 2   / \ 2
   +     +        |       |        |       |    ---+            ---+          +
  / \   / \     .-+-.   .-+-.     .+.     .+.      |   .--- 3      |   .-- 3   \ / 3
 /   \ /   \    |   |   |   |    |   |   |   |     '---+            '-+         +
 1   2 3   4    1   2   3   4    1   2   3   4         '--- 4          '-- 4     \ 4

```





