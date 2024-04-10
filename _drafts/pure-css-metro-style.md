---
layout: post
title:  Pure CSS Metro Style
categories: css
---

用 CSS 实现 windows 的 Metro 风格样式。

## preview

https://codepen.io/forhot2000/pen/jyVRbq

## source code

html

```html
<div class="container bg-gray">

  <div class="row bg-yellow">
    <div class="card card-small bg-green">
      <label class="center">A block of long title</label>
    </div>
    <div class="card bg-gray">
      <span class="">text must contains in an element with class [center | [top|bottom]-[left|right] ], otherwise, it will display error in iOS.</span>
    </div>
    <div class="card card-wide bg-red"></div>
    <div class="card card-small bg-red">
      <label class="center">A block of long description</label>
    </div>
    <div class="card card-small bg-red"></div>
  </div>

  <div class="row bg-green">
    <div class="card card-small bg-red">
      <label class="center">A block of long description</label>
    </div>
    <div class="card card-small bg-blue"></div>
    <div class="card card-wide bg-blue">
      <label class="center">A block of long description</label>
    </div>
    <div class="card card-wide bg-yellow">
      <label class="center">center</label>
      <label class="top-left">top-left</label>
      <label class="top-right">top-right</label>
      <label class="bottom-left">bottom-left</label>
      <label class="bottom-right">bottom-right</label>
    </div>
  </div>

  <div class="row bg-green">
    <div class="card card-small bg-red">
      <label class="center">A block of long description</label>
    </div>
    <div class="card card-small bg-blue"></div>
    <div class="card card-wide bg-blue">
      <label class="center">A block of long description</label>
    </div>
    <div class="card card-wide bg-yellow">
      <label class="center">there is a image, wait loading...</label>
      <img class="center fullfill" src="https://www.baidu.com/img/bd_logo1.png" />
      <label class="top-left">top-left</label>
      <label class="top-right">top-right</label>
      <label class="bottom-left">bottom-left</label>
      <label class="bottom-right">bottom-right</label>
    </div>
  </div>

</div>
```

scss

```scss
$font-size: 10pt;
$small-space: 5px;
$space: ($small-space * 2);
$small-size: 80px;
$normal-size: ($small-size * 2 + $space);
$large-size: ($normal-size * 2 + $space);

body {
  height: 100%;
  margin: 0;
  padding: 0;
}

.container {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: gray;
  padding: $space;
  white-space: nowrap;
  overflow: auto;
}

.row {
  box-sizing: border-box;
  font-size: 0;
  width: ($large-size + $space * 2);
  // min-height: ($large-size + $space * 2);
  padding: $space 0 0 $space;
  margin: 0 $space $space 0;
  overflow-y: auto;
  white-space: normal;
  display: inline-block;
}

.card {
  box-sizing: border-box;
  font-size: $font-size;
  width: $normal-size;
  height: $normal-size;
  display: inline-block;
  position: relative;
  padding: $small-space;
  margin: 0 $space $space 0;
  overflow: hidden;
}

.card.card-small {
  width: $small-size;
  height: $small-size;
}

.card.card-wide {
  width: $large-size;
}

.center {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 100%;
  transform: translate(-50%, -50%);
  text-align: center;
  padding: $small-space;
}

.center.fullfill {
  padding: 0;
}

.top-left {
  position: absolute;
  top: 0;
  left: 0;
  padding: $small-space;
}

.top-right {
  position: absolute;
  top: 0;
  right: 0;
  padding: $small-space;
}

.bottom-left {
  position: absolute;
  bottom: 0;
  left: 0;
  padding: $small-space;
}

.bottom-right {
  position: absolute;
  bottom: 0;
  right: 0;
  padding: $small-space;
}

.bg-red {
  background-color: red;
  color: black;
}

.bg-blue {
  background-color: blue;
  color: white;
}

.bg-gray {
  background-color: gray;
  color: white;
}

.bg-yellow {
  background-color: yellow;
  color: black;
}

.bg-green {
  background-color: green;
  color: black;
}
```
