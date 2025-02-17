# Gammo - A pure-Ruby HTML5 parser

[![Build Status](https://travis-ci.org/namusyaka/gammo.svg?branch=master)](https://travis-ci.org/namusyaka/gammo)
[![GitHub issues](https://img.shields.io/github/issues/namusyaka/gammo)](https://github.com/namusyaka/gammo/issues)
[![GitHub forks](https://img.shields.io/github/forks/namusyaka/gammo?color=brightgreen)](https://github.com/namusyaka/gammo/network)
[![GitHub stars](https://img.shields.io/github/stars/namusyaka/gammo?color=brightgreen)](https://github.com/namusyaka/gammo/stargazers)
[![GitHub license](https://img.shields.io/github/license/namusyaka/gammo?color=brightgreen)](https://github.com/namusyaka/gammo/blob/master/LICENSE.txt)
[![Documentation](http://img.shields.io/:yard-docs-38c800.svg)](http://www.rubydoc.info/gems/gammo/frames)

Gammo provides a pure Ruby HTML5-compliant parser and CSS selector / XPath support for traversing the DOM tree built by Gammo.
The implementation of the HTML5 parsing algorithm in Gammo conforms [the WHATWG specification](https://html.spec.whatwg.org/multipage/parsing.html). Given an HTML string, Gammo parses it and builds DOM tree based on the tokenization and tree-construction algorithm defined in WHATWG parsing algorithm, these implementations are provided without any external dependencies.

Gammo, its naming is inspired by [Gumbo](https://github.com/google/gumbo-parser). But Gammo is a fried tofu fritter made with vegetables.

```ruby
require 'gammo'
require 'open-uri'

parser = URI.open('https://google.com') { |f| Gammo.new(f.read) }
document = parser.parse #=> #<Gammo::Node::Document>

puts document.css('title').first.inner_text #=> 'Google'
```

* [Overview](#overview)
   * [Features](#features)
* [Tokenizaton](#tokenizaton)
   * [Token types](#token-types)
* [Parsing](#parsing)
   * [Notes](#notes)
* [Node](#node)
* [DOM Tree Traversal](#dom-tree-traversal)
   * [XPath 1.0 (experimental)](#xpath-10-experimental)
      * [Example](#example)
      * [Axis Specifiers](#axis-specifiers)
      * [Node Test](#node-test)
      * [Operators](#operators)
      * [Functions](#functions)
         * [Node set functions](#node-set-functions)
         * [String Functions](#string-functions)
         * [Boolean Functions](#boolean-functions)
         * [Number Functions](#number-functions)
   * [CSS Selector (experimental)](#css-selector-experimental)
      * [Example](#example)
      * [Groups of selectors](#groups-of-selectors)
      * [Simple selectors](#simple-selectors)
         * [Type selector &amp; Universal selector](#type-selector--universal-selector)
         * [Attribute selectors](#attribute-selectors)
         * [Class selectors](#class-selectors)
         * [ID selectors](#id-selectors)
         * [Pseudo-classes](#pseudo-classes)
      * [Combinators](#combinators)
* [Performance](#performance)
* [References](#references)
* [License](#license)
* [Release History](#release-history)

## Overview

### Features

- [Tokenization](#tokenization): Gammo has a tokenizer for implementing [the tokenization algorithm](https://html.spec.whatwg.org/multipage/parsing.html#tokenization).
- [Parsing](#parsing): Gammo provides a parser which implements the parsing algorithm by the above tokenization and [the tree-construction algorithm](https://html.spec.whatwg.org/multipage/parsing.html#tree-construction).
- [Node](#node): Gammo provides the nodes which implement [WHATWG DOM specification](https://dom.spec.whatwg.org/) partially.
- [DOM Tree Traversal](#dom-tree-traversal): Gammo provides a way of DOM tree traversal (CSS selector / XPath).
- [Performance](#performance): Gammo does not prioritize performance, and there are a few potential performance notes.

## Tokenizaton

`Gammo::Tokenizer` implements the tokenization algorithm in WHATWG. You can get tokens in order by calling `Gammo::Tokenizer#next_token`.

Here is a simple example for performing only the tokenizer.

```ruby
def dump_for(token)
  puts "data: #{token.data}, class: #{token.class}"
end

tokenizer = Gammo::Tokenizer.new('<!doctype html><input type="button"><frameset>')
dump_for tokenizer.next_token #=> data: html, class: Gammo::Tokenizer::DoctypeToken
dump_for tokenizer.next_token #=> data: input, class: Gammo::Tokenizer::StartTagToken
dump_for tokenizer.next_token #=> data: frameset, class: Gammo::Tokenizer::StartTagToken
dump_for tokenizer.next_token #=> data: end of string, class: Gammo::Tokenizer::ErrorToken
```

The parser described below depends on this tokenizer, it applies the WHATWG parsing algorithm to the tokens extracted by this tokenization in order.

### Token types

The tokens generated by the tokenizer will be categorized into one of the following types:

<table>
  <thead>
    <tr>
      <th>Token type</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>Gammo::Tokenizer::ErrorToken</code></td>
      <td>Represents an error token, it usually means end-of-string.</td>
    </tr>
    <tr>
      <td><code>Gammo::Tokenizer::TextToken</code></td>
      <td>Represents a text token like "foo" which is inner text of elements.</td>
    </tr>
    <tr>
      <td><code>Gammo::Tokenizer::StartTagToken</code></td>
      <td>Represents a start tag token like <code>&lt;a&gt;</code>.</td>
    </tr>
    <tr>
      <td><code>Gammo::Tokenizer::EndTagToken</code></td>
      <td>Represents an end tag token like <code>&lt;/a&gt;</code>.</td>
    </tr>
    <tr>
      <td><code>Gammo::Tokenizer::SelfClosingTagToken</code></td>
      <td>Represents a self closing tag token like <code>&lt;img /&gt;</code></td>
    </tr>
    <tr>
      <td><code>Gammo::Tokenizer::CommentToken</code></td>
      <td>Represents a comment token like <code>&lt;!-- comment --&gt;</code>.</td>
    </tr>
    <tr>
      <td><code>Gammo::Tokenizer::DoctypeToken</code></td>
      <td>Represents a doctype token like <code>&lt;!doctype html&gt;</code>.</td>
    </tr>
  </tbody>
</table>

## Parsing

`Gammo::Parser` implements processing in [the tree-construction stage](https://html.spec.whatwg.org/multipage/parsing.html#tree-construction) based on the tokenization described above.

A successfully parsed parser has the `document` accessor as the root document (this is the same as the return value of the `Gammo::Parser#parse`). From the `document` accessor, you can traverse the DOM tree constructed by the parser.

```ruby
require 'gammo'
require 'pp'

document = Gammo.new('<!doctype html><input type="button">').parse

def dump_for(node, strm)
  strm << node.to_h
  return unless node && (child = node.first_child)
  while child
    dump_for(child, (strm.last[:children] ||= []))
    child = child.next_sibling
  end
  strm
end

pp dump_for(document, [])
```

### Notes

Currently, it's not possible to traverse the DOM tree with css selector or xpath like [Nokogiri](https://nokogiri.org/).
However, Gammo plans to implement these features in the future.

## Node

The nodes generated by the parser will be categorized into one of the following types:

<table>
  <thead>
    <tr>
      <th>Node type</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>Gammo::Node::Error</code></td>
      <td>Represents error node, it usually means end-of-string.</td>
    </tr>
    <tr>
      <td><code>Gammo::Node::Text</code></td>
      <td>Represents the text node like "foo" which is inner text of elements.</td>
    </tr>
    <tr>
      <td><code>Gammo::Node::Document</code></td>
      <td>Represents the root document type. It's always returned by <code>Gammo::Parser#document</code>.</td>
    </tr>
    <tr>
      <td><code>Gammo::Node::Element</code></td>
      <td>Represents any elements of HTML like <code>&lt;p&gt;</code>.</td>
    </tr>
    <tr>
      <td><code>Gammo::Node::Comment</code></td>
      <td>Represents comments like <code>&lt;!-- foo --&gt;</code></td>
    </tr>
    <tr>
      <td><code>Gammo::Node::Doctype</code></td>
      <td>Represents doctype like <code>&lt;!doctype html&gt;</code></td>
    </tr>
  </tbody>
</table>

For some nodes such as `Gammo::Node::Element` and `Gammo::Node::Document`, they contain pointers to nodes that can be referenced by itself, such as `Gammo::Node#next_sibling` or `Gammo::Node#first_child`. In addition, APIs such as `Gammo::Node#append_child` and `Gammo::Node#remove_child` that perform operations defined in DOM living standard are also provided.

## DOM Tree Traversal

CSS selector and XPath-1.0 are the way for traversing DOM tree built by Gammo.

### XPath 1.0 (experimental)

Gammo has an original lexer/parser for XPath 1.0, it's provided as a helper in the DOM tree built by Gammo. 
Here is a simple example:

```ruby
document = Gammo.new('<!doctype html><input type="button">').parse
node_set = document.xpath('//input[@type="button"]') #=> "<Gammo::XPath::NodeSet>"

node_set.length #=> 1
node_set.first #=> "<Gammo::Node::Element>"
```

**Since this is implemented by full scratch, Gammo is providing this support as a very experimental feature.**
Please [file an issue](/issues/new) if you find bugs.

#### Example

Before proceeding at the details of XPath support, let's have a look at a few simple examples.
Given a sample HTML text and its DOM tree:

```ruby
document = Gammo.new(<<-EOS).parse
<!DOCTYPE html>
<html>
<head>
</head>
<body>
  <h1>namusyaka.com</h1>
  <p class="description">Here is a sample web site.</p>
  <ul>
    <li>hello</li>
    <li>world</li>
  </ul>
  <ul id="links">
    <li>Google <a href="https://google.com/">google.com</a></li>
    <li>GitHub <a href="https://github.com/namusyaka">github.com/namusyaka</a></li>
  </ul>
</body>
</html>
EOS
```

The following XPath expression gets all `li` elements and prints those text contents:

```ruby
document.xpath('//li').each do |elm|
  puts elm.inner_text
end
```

The following XPath expression gets all `li` elements under the `ul` element having the `id=links` attribute:

```ruby
document.xpath('//ul[@id="links"]/li').each do |elm|
  puts elm.inner_text
end
```

The following XPath expression gets each text node for each `li` element under the `ul` element having the `id=links` attribute:

```ruby
document.xpath('//ul[@id="links"]/li/text()').each do |elm|
  puts elm.data
end
```

#### Axis Specifiers

In the combination with Gammo, the axis specifier indicates navigation direction within the DOM tree built by Gammo. Here is list of axes. As you can see, Gammo fully supports the all of axes.

<table>
  <thead>
    <tr>
      <th>Full Syntax</th>
      <th>Abbreviated Syntax</th>
      <th>Supported</th>
      <th>Notes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>ancestor</code></td>
      <td></td>
      <td>yes</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ancestor-or-self</code></td>
      <td></td>
      <td>yes</td>
      <td></td>
    </tr>
    <tr>
      <td><code>attribute</code></td>
      <td><code>@</code></td>
      <td>yes</td>
      <td><code>@abc</code> is the alias for <code>attribute::abc</code></td>
    </tr>
    <tr>
      <td><code>child</code></td>
      <td><code></code></td>
      <td>yes</td>
      <td><code>abc</code> is the short for <code>child::abc</code></td>
    </tr>
    <tr>
      <td><code>descendant</code></td>
      <td></td>
      <td>yes</td>
      <td></td>
    </tr>
    <tr>
      <td><code>descendant-or-self</code></td>
      <td><code>//</code></td>
      <td>yes</td>
      <td><code>//</code> is the alias for <code>/descendant-or-self::node()/</code></td>
    </tr>
    <tr>
      <td><code>following</code></td>
      <td></td>
      <td>yes</td>
      <td></td>
    </tr>
    <tr>
      <td><code>following-sibling</code></td>
      <td></td>
      <td>yes</td>
      <td></td>
    </tr>
    <tr>
      <td><code>namespace</code></td>
      <td></td>
      <td>yes</td>
      <td></td>
    </tr>
    <tr>
      <td><code>parent</code></td>
      <td><code>..</code></td>
      <td>yes</td>
      <td><code>..</code> is the alias for <code>parent::node()</code></td>
    </tr>
    <tr>
      <td><code>preceding</code></td>
      <td></td>
      <td>yes</td>
      <td></td>
    </tr>
    <tr>
      <td><code>preceding-sibling</code></td>
      <td></td>
      <td>yes</td>
      <td></td>
    </tr>
    <tr>
      <td><code>self</code></td>
      <td><code>.</code></td>
      <td>yes</td>
      <td><code>.</code> is the alias for <code>self::node()</code></td>
    </tr>
  </tbody>
</table>

#### Node Test

Node tests consist of specific node names or more general expressions. Although particular syntax like `:` should work for specifying namespace prefix in XPath, Gammo does not support it yet as it's [not a core feature in HTML5](https://html.spec.whatwg.org/multipage/introduction.html#html-vs-xhtml).

<table>
  <thead>
    <tr>
      <th>Full Syntax</th>
      <th>Supported</th>
      <th>Notes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>text()</code></td>
      <td>yes</td>
      <td>Finds a node of type text, e.g. <code>hello</code> in <code>&lt;p&gt;hello &lt;a href="https://hello"&gt;world&lt;/a&gt;&lt;/p&gt;</td>
    </tr>
    <tr>
      <td><code>comment()</code></td>
      <td>yes</td>
      <td>Finds a node of type comment, e.g. <code>&lt;!-- comment --&gt;</code></td>
    </tr>
    <tr>
      <td><code>node()</code></td>
      <td>yes</td>
      <td>Finds any node at all.</td>
    </tr>
  </tbody>
</table>

Also note that the `processing-instruction` is not supported. There is no plan to support it.

#### Operators

- The `/`, `//` and `[]` are used in the path expression.
- The union operator `|` forms the union of two node sets.
- The boolean operators: `and`, `or`
- The arithmetic operators: `+`, `-`, `*`, `div` and `mod`
- Comparison operators: `=`, `!=`, `<`, `>`, `<=`, `>=`

#### Functions

XPath 1.0 defines four data types (nodeset, string, number, boolean) and there are various functions based on the types. Gammo supports those functions partially, please check it to be supported before using functions.

##### Node set functions

<table>
  <thead>
    <tr>
      <th>Function Name</th>
      <th>Supported</th>
      <th>Specification</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>last()</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-last</td>
    </tr>
    <tr>
      <td><code>position()</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-position</td>
    </tr>
    <tr>
      <td><code>count(node-set)</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-count</td>
    </tr>
  </tbody>
</table>

##### String Functions

<table>
  <thead>
    <tr>
      <th>Function Name</th>
      <th>Supported</th>
      <th>Specification</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>string(object?)</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-string</td>
    </tr>
    <tr>
      <td><code>concat(string, string, string*)</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-concat</td>
    </tr>
    <tr>
      <td><code>starts-with(string, string)</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-starts-with</td>
    </tr>
    <tr>
      <td><code>contains(string, string)</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-contains</td>
    </tr>
    <tr>
      <td><code>substring-before(string, string)</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-substring-before</td>
    </tr>
    <tr>
      <td><code>substring-after(string, string)</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-substring-after</td>
    </tr>
    <tr>
      <td><code>substring(string, number, number?)</code></td>
      <td>no</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-substring</td>
    </tr>
    <tr>
      <td><code>string-length(string?)</code></td>
      <td>no</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-string-length</td>
    </tr>
    <tr>
      <td><code>normalize-space(string?)</code></td>
      <td>no</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-string-normalize-space</td>
    </tr>
    <tr>
      <td><code>translate(string, string, string)</code></td>
      <td>no</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-string-translate</td>
    </tr>
  </tbody>
</table>

##### Boolean Functions

<table>
  <thead>
    <tr>
      <th>Function Name</th>
      <th>Supported</th>
      <th>Specification</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>boolean(object)</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-boolean</td>
    </tr>
    <tr>
      <td><code>not(object)</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-not</td>
    </tr>
    <tr>
      <td><code>true()</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-true</td>
    </tr>
    <tr>
      <td><code>false()</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-false</td>
    </tr>
    <tr>
      <td><code>lang()</code></td>
      <td>no</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-lang</td>
    </tr>
  </tbody>
</table>

##### Number Functions

<table>
  <thead>
    <tr>
      <th>Function Name</th>
      <th>Supported</th>
      <th>Specification</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>number(object?)</code></td>
      <td>no</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-number</td>
    </tr>
    <tr>
      <td><code>sum(node-set)</code></td>
      <td>no</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-sum</td>
    </tr>
    <tr>
      <td><code>floor(number)</code></td>
      <td>no</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-floor</td>
    </tr>
    <tr>
      <td><code>ceiling(number)</code></td>
      <td>yes</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-ceiling</td>
    </tr>
    <tr>
      <td><code>round(number)</code></td>
      <td>no</td>
      <td>https://www.w3.org/TR/1999/REC-xpath-19991116/#function-round</td>
    </tr>
  </tbody>
</table>

### CSS Selector (experimental)

Gammo has an original lexer/parser for CSS Selector, it's provided as a helper in the DOM tree built by Gammo.
Here is a simple example:

```ruby
document = Gammo.new('<!doctype html><input type="button">').parse
node_set = document.css('input[type="button"]') #=> "<Gammo::CSSSelector::NodeSet>"

node_set.length #=> 1
node_set.first #=> "<Gammo::Node::Element>"
```

Since this is implemented by full scratch, Gammo is providing this support as a very experimental feature. Please file an issue if you find bugs.

#### Example

Before proceeding at the details of CSS Selector support, let's have a look at a few simple examples. Given a sample HTML text and its DOM tree:

```ruby
document = Gammo.new(<<-EOS).parse
<!DOCTYPE html>
<html>
<head>
</head>
<body>
  <h1>namusyaka.com</h1>
  <p class="description">Here is a sample web site.</p>
  <ul>
    <li>hello</li>
    <li>world</li>
  </ul>
  <ul id="links">
    <li>Google <a href="https://google.com/">google.com</a></li>
    <li>GitHub <a href="https://github.com/namusyaka">github.com/namusyaka</a></li>
  </ul>
</body>
</html>
EOS
```

The following CSS selector gets all `li` elements and prints thoese text contents:

```ruby
document.css('li').each do |elm|
  puts elm.inner_text
end
```

The following CSS selector gets all `li` elements under the `ul` element having the `id=links` attribute:

```ruby
document.xpath('ul#links li').each do |elm|
  puts elm.inner_text
end
```

#### Groups of selectors

Gammo supports [groups of selectors](https://www.w3.org/TR/2018/REC-selectors-3-20181106/#grouping), this means you can use `,` to traverse DOM tree by multiple selectors.

```ruby
require 'gammo'

@doc = Gammo.new(<<-EOS).parse
<!DOCTYPE html>
<html>
<head>
<title>hello</title>
<meta charset="utf8">
</head>
<body>
<p id="hello">hello</p>
<p id="world">world</p>
EOS

@doc.css('#hello, #world').map(&:inner_text).join(' ') #=> 'hello world'
```

#### Simple selectors

##### Type selector & Universal selector

Gammo supports the basic grammar of type selector and universal selector, but not namespaces.

##### Attribute selectors

See more details: [6.3. Attribute selectors](https://www.w3.org/TR/2018/REC-selectors-3-20181106/#attribute-selectors)

<table>
  <thead>
    <tr>
      <th>Syntax</th>
      <th>Supported</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>[att]</code></td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>[att=val]</code></td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>[att~=val]</code></td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>[att|=val]</code></td>
      <td>yes</td>
    </tr>
  </tbody>
</table>

##### Class selectors

Supported. See more details: [6.4. Class selectors](https://www.w3.org/TR/2018/REC-selectors-3-20181106/#class-html)

##### ID selectors

Supported. See more details: [6.5. ID selectors](https://www.w3.org/TR/2018/REC-selectors-3-20181106/#id-selectors)

##### Pseudo-classes

Partially supported. See the table below.

<table>
  <thead>
    <tr>
      <th>Class name</th>
      <th>Supported</th>
      <th>Can support?</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>:link</code></td>
      <td>no</td>
      <td>no</td>
    </tr>
    <tr>
      <td><code>:visited</code></td>
      <td>no</td>
      <td>no</td>
    </tr>
    <tr>
      <td><code>:hover</code></td>
      <td>no</td>
      <td>no</td>
    </tr>
    <tr>
      <td><code>:active</code></td>
      <td>no</td>
      <td>no</td>
    </tr>
    <tr>
      <td><code>:focus</code></td>
      <td>no</td>
      <td>no</td>
    </tr>
    <tr>
      <td><code>:target</code></td>
      <td>no</td>
      <td>no</td>
    </tr>
    <tr>
      <td><code>:lang</code></td>
      <td>no</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:enabled</code></td>
      <td>yes</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:disabled</code></td>
      <td>yes</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:checked</code></td>
      <td>yes</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:root</code></td>
      <td>yes</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:nth-child</code></td>
      <td>yes</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:nth-last-child</code></td>
      <td>no</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:nth-of-type</code></td>
      <td>no</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:nth-last-of-type</code></td>
      <td>no</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:first-child</code></td>
      <td>no</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:last-child</code></td>
      <td>no</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:first-of-type</code></td>
      <td>no</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:last-of-type</code></td>
      <td>no</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:only-child</code></td>
      <td>no</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:only-of-type</code></td>
      <td>no</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:empty</code></td>
      <td>no</td>
      <td>yes</td>
    </tr>
    <tr>
      <td><code>:not</code></td>
      <td>yes</td>
      <td>yes</td>
    </tr>
  </tbody>
</table>

#### Combinators

See more details: [8. Combinators](https://www.w3.org/TR/2018/REC-selectors-3-20181106/#combinators)

<table>
  <thead>
    <tr>
      <th>Syntax</th>
      <th>Supported</th>
      <th>Desc</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>h1 em</code></td>
      <td>yes</td>
      <td>Descendant combinator</td>
    </tr>
    <tr>
      <td><code>h1 > em</code></td>
      <td>yes</td>
      <td>Child combinator</td>
    </tr>
    <tr>
      <td><code>math + p</code></td>
      <td>yes</td>
      <td>Next-sibling combinator</td>
    </tr>
    <tr>
      <td><code>h1 ~ pre</code></td>
      <td>yes</td>
      <td>Subsequent-sibling combinator</td>
    </tr>
  </tbody>
</table>

## Performance

As mentioned in the features at the beginning, Gammo doesn't prioritize its performance.
Thus, for example, Gammo is not suitable for very performance-sensitive applications (e.g. performing Gammo parsing synchronously from an incoming request from an end user).
Instead, the goal is to work well with batch processing such as crawlers.
Gammo places the highest priority on making it easy to parse HTML by peforming it without depending on native-extensions and external gems.

## References

This was developed with reference to the following softwares.

- [x/net/html](https://godoc.org/golang.org/x/net/html): I've been working on this package, it gave me strong reason to make this happen.
- [Blink](https://www.chromium.org/blink): Blink gave me great impression about tree construction.
- [html5lib-tests](https://github.com/html5lib/html5lib-tests): Gammo relies on this test.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).

## Release History

- v0.3.0
    - CSS selector support [#11](https://github.com/namusyaka/gammo/pull/11)
- v0.2.0
    - XPath 1.0 support [#4](https://github.com/namusyaka/gammo/pull/4)
- v0.1.0
    - Initial Release
