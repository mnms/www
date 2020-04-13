---
layout: blog
title: Sample article for demo & other-category
author: someone
published: false
category: other-category
excerpt: |
  Some text here for summary - without a blog, your SEO can tank, you'll have nothing to promote on social media, you'll have no clout with your leads and customers, and you'll have fewer pages to put those valuable calls-to-action that generate inbound leads.
---

## Plain Text

You've probably heard how paramount blogging is to the success of your marketing. But it's important that you learn how to start a blog and write blog posts for it so that each article supports your business.

Without a blog, your SEO can tank, you'll have nothing to promote on social media, you'll have no clout with your leads and customers, and you'll have fewer pages to put those valuable calls-to-action that generate inbound leads.

<!-- more -->

## Markdown

### Lists

Sometimes you want numbered lists:

1. One
2. Two
3. Three

Sometimes you want bullet points:

* Start a line with a star
* Profit!

Alternatively,

- Dashes work just as well
- And if you have sub points, put two spaces before the dash or star:
  - Like this
  - And this

### Table

| # | Col1 | Col2 | Col3 |
| - | ---- | ---- | ---- |
| 1 | Val1 | Val2 | Val3 |
| 2 | Value1 | Value2 | Value3 |
| 3 | Val1 | Val2 | Val3 |
| 4 | Val1 | Val2 | Val3 |
| 5 | AnotherValue1 | AnotherValue2 | AnotherValue3 |
| 6 | Val1 | Val2 | Val3 |
| 7 | LongLongValue1234567890 | LongLongValue1234567890 | VeryVeryLongLongValue1234567890 |

### Quotes

> Coffee. The finest organic suspension ever devised... I beat the Borg with it.

### Link (anchor)

[This text](https://naver.com) leads you to Naver.

### Code

GitHub also supports something called code fencing, which allows for multiple lines without indentation:

```
// Javascript code without syntax highlight
if (isAwesome){
  return true
}
```

And if you'd like to use syntax highlighting, include the language:

```javascript
// Javascript code with syntax highlight
if (isAwesome){
  return true
}
```

```scala
// Scala code : Sample
package examples.actors

import scala.actors._
import scala.actors.Actor._

object Message {
  def main(args: Array[String]) {
    val n = try {
      Integer.parseInt(args(0))
    }
    catch {
      case _ =>
        println("Usage: examples.actors.Message <n>")
        Predef.exit
    }
    val nActors = 500
    val finalSum = n * nActors
    Scheduler.impl = new SingleThreadedScheduler

    def beh(next: Actor, sum: Int) {
      react {
        case value: Int =>
          val j = value + 1; val nsum = sum + j
          if (next == null && nsum >= finalSum) {
            println(nsum)
            System.exit(0)
          }
          else {
            if (next != null) next ! j
            beh(next, nsum)
          }
      }
    }

    def actorChain(i: Int, a: Actor): Actor =
      if (i > 0) actorChain(i-1, actor(beh(a, 0))) else a

    val firstActor = actorChain(nActors, null)
    var i = n; while (i > 0) { firstActor ! 0; i -= 1 }
  }
}
```

### To Do

- [x] This is a complete item
- [ ] This is an incomplete item

## Image

(Markdown only : max-width 80%)<br />
![sample image](/assets/img/clients/SKtelecom.png)

(HTML Tag)<br />
<img src="/assets/img/clients/SKtelecom.png" width="150px" border="0" />

## Equation (MathJAX)

<!-- http://atomurl.net/math/ -->

$$mean = \frac{\displaystyle\sum_{i=1}^{n} x_{i}}{n}$$

When $(a \ne 0)$, there are two solutions to $(ax^2 + bx + c = 0)$ and they are $$x = {-b \pm \sqrt{b^2-4ac} \over 2a}$$

$$x = \lim_{a \rightarrow b}$$

This formula $f(x) = x^2$ is an example.

$$ x = a_0 + \frac{1}{\displaystyle a_1 + \frac{1}{\displaystyle a_2 + \frac{1}{\displaystyle a_3 + a_4}}}$$

$x_1$, $x_2$, $x_3$, $x_4$, ..., $x_i$

## Diagram (MermaidJS)

### Sample: Sequence

<div class="mermaid">
sequenceDiagram
Alice->>John: Hello John, how are you?
loop Healthcheck
    John->>John: Fight against hypochondria
end
Note right of John: Rational thoughts!
John-->>Alice: Great!
John->>Bob: How about you?
Bob-->>John: Jolly good!
</div>

### Sample: Pie Chart

<div class="mermaid">
pie
"Dogs" : 386
"Cats" : 85
"Rats" : 15
</div>

### Daniel's sample1

<div class="mermaid">
graph TD;
subgraph Aggregation Pushdown
Redis -->|Result| Spark
Spark -->|Aggregate| Redis
end
</div>

### Daniel's sample2

<div class="mermaid">
graph TD;
subgraph Aggregation Tree
Ag1{{Aggregation 1}} --> Op1[/Operation 1/]
subgraph Operation Subtree
Op1 --> Op2[/Operation 2/]
Op2 --> Opd1(Operand 1)
Op2 --> Opd2(Operand 2)
Op1 --> Opd3(Operand 3)
end
Ag1 --- Ag2{{Aggregation 2}}
Ag2 --> OpSub1[/Operation Subtree 2\]
Ag2 --- Ag3{{Aggregation 3}}
Ag3 --> OpSub2[/Operation Subtree 3\]
Ag3 --> None((None))
end
</div>

## Another Plain Text

Maybe because, unless you're one of the few people who actually like writing, business blogging kind of stinks. You have to find words, string them together into sentences ... ugh, where do you even start?

→ Download Now: 6 Free Blog Post Templates
Well my friend, the time for excuses is over.

Today, people and organizations of all walks of life manage blogs to share analyses, instruction, criticisms, and other observations of an industry in which they are a rising expert.

After you read this post, there will be absolutely no reason you can't blog every single day -- and do it quickly. Not only am I about to provide you with a simple blog post formula to follow, but I'm also going to give you free templates for creating five different types of blog posts:

The How-To Post
The List-Based Post
The Curated Collection Post
The SlideShare Presentation Post
The Newsjacking Post
With all this blogging how-to, literally anyone can blog as long as they truly know the subject matter they're writing about. And since you're an expert in your industry, there's no longer any reason you can't sit down every day and hammer out an excellent blog post.