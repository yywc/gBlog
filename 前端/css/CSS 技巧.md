## 前言

css 大概是每个前端工程师的痛了，没有人不会，也没有人会。我也是从一开始半知半解，直到看了[张旭鑫大佬](https://www.zhangxinxu.com/)的博客，才发现自己如此才疏学浅，于是也就诞生了这一篇文章，用来记录 css 那些黑魔法。

以下是根据 MDN 顺序记录的是相对没有那么为人熟知的，你可能知道，你也有可能不知道，欢迎大家补充更多。

## 1. 多行文本超出显示省略号

![image-20190826103757050](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100725.jpg)

主要涉及知识点:

+ display: -webkit-box/-webkit-inline-box
+ -webkit-box-orient: vertical
+ -webkit-line-clamp: x (x 为你所需要的行数)

```txt
// html
<p>
  In this example the <code>-webkit-line-clamp</code> property is set to <code>3</code>, which means the text is clamped after three lines.
  An ellipsis will be shown at the point where the text is clamped.
</p>

// css
p {
  width: 300px;
  display: -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 3;
  overflow: hidden;
}
```

兼容性：

+ IE：要不起
+ Edge：17+
+ Firefox：68+
+ Chrome、Safari 放心使用

## 2. 模拟 DOM 元素 title 属性

![image-20190826111753393](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100727.png)

主要涉及知识点：

+ data-*：可以通过 HTMLElement.dataset[\*] 来获取内容，也可以在 css 里通过 attr 获取
+ attr()：理论上可以使用在任何 css 属性，但目前只能配合伪元素的 content 使用

```txt
// html
<div>What is the <span data-css-title="this is a information">information</span>?</div>

// css
span[data-css-title] {
  position: relative;
  color: #2d8cf0;
  cursor: help;
}

span[data-css-title]:hover::before {
  position: absolute;
  top: 25px;
  left: 50%;
  border-width: 5px;
  border-style: solid;
  border-color: transparent rgba(70, 76, 91, 0.9) transparent transparent;
  background-color: transparent;
  transform: rotate(90deg) translate3d(-50%, 0, 0);
  content: '';
  z-index: 3;
}

span[data-css-title]:hover::after {
  position: absolute;
  top: 30px;
  left: 50%;
  padding: 8px 12px;
  color: #fff;
  font-size: 13px;
  max-width: 300px;
  white-space: nowrap;
  background-color: rgba(70, 76, 91, 0.9);
  border-radius: 4px;
  box-shadow: 0 1px 6px rgba(0, 0, 0, 0.2);
  box-sizing: border-box;
  transform: translate3d(-50%, 0, 0);
  content: attr(data-css-title);
  z-index: 3;
}
```

兼容性良好可以放心使用，实际项目中可以定义在全局 css 中，通过 data-* 来达到 title 一样的效果。

## 3. 使用计数菌

![image-20190828142039498](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100730.png)

通过 css 我们也可以实现计数功能，主要涉及知识点：

+ counter-reset
+ counter-increament
+ counter()、counters()：不含子元素使用 counter、含子元素使用 counters

```txt
// html
<ul>
  <li>
    <ul>
      <li></li>
      <li></li>
    </ul>
   </li>
  <li>
   <ul>
     <li></li>
     <li></li>
    </ul>
  </li>
</ul>

<!--
<ul>
  <li></li>
  <li></li>
</ul>
-->

// css
ul {
  counter-reset: count;
}

li {
  counter-increment: count;
}

li::before {
  content: counters(count, '.');
  // content: counter(count);
}
```
