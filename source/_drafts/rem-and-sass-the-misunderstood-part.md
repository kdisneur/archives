---
layout: post
title: REM and SASS, the misunderstood part
tags:
  - sass
  - css
  - frontend
---
CSS3 has introduced a new size unit: REM. I'm not going to explain you what is REM, lot of blog posts already do this
very well: [font-size with REM](http://snook.ca/archives/html_and_css/font-size-with-rem),
[confused about REM and EM](https://j.eremy.net/confused-about-rem-and-em/),... The purpose of this article is to explain
how lot of us misuse this unit.

## Old habits

Some times ago, before [the invention of the "responsive" websites](http://blog.cleancoder.com/uncle-bob/2014/10/08/GOML1-ResponsiveDesign.html)
we were working with fixed size, usually `966px`. Almost every developers were working like this. It was a lot
easier than percentage (and media queries were not yet created) so fixed size was the standard. So, because everything
were fixed, we were able to use pixels to size every piece of design. We were able to do **pixel** perfect websites.

Now, with the new browsers abilities, the multitude of devices and of course, the rise of the responsive design, we have
to change our mind and think about mobile first and user experience on every screen size. So we've started to work with
media queries and so on... But it sounds difficult to abandon our old pixel friend.

## REM and SASS, the bad path

Because it's the new standard, we have started to use `em` and `rem` in our stylesheets. It's a good news but lot of us
use them with the old spirit in mind.

Instead of learning a new way of thinking: relative, we continue to think in absolute values. For example, I already see
this kind of mixin on number of projects:

```scss
/* _variables.scss */
$base-font-size: 12px;

/* _helpers.scss */
@mixin font-size($size) {
  font-size: $size;
  font-size: $size / $base-font-size * 1rem;
}

/* application.scss */
.my-element {
  @include font-size(18px);
}
```

What does the mixin? It simply converts `18px` to it's `rem` equivalent for the new browsers and also keep the `px`
notation for the legacy browsers (IE8, I'm looking at you).

Why it's a smell? Simply because the greatest strength of `rem` is you can change the base font and, automatically, all
the website will scale proportionally. Awesome.

With this implementation, if you change the base font, the `rem` value will be updated to still represent `18px`,
nothing will scale... It's just a more complicated way to write `font-size: 18px` and doesn't solve any problem.

## REM and SASS, a better path

As I said before, IE8 doesn't handle `rem` so using such mixin to fallback in pixel is a really good idea, we just have
to inverse the way the previous mixin work and we could have something really handy:

```scss
/* _variables.scss */
$base-font-size: 12px;

/* _helpers.scss */
@mixin font-size($size) {
  font-size: $size * $base-font-size * 1px;
  font-size: $size;
}

/* application.scss */
.my-element {
  @include font-size(1.5rem);
}
```

Now, we define our size to be `1.5rem`, so, with the current font-size, it equals to `18px`. And, now, if we increase the
base font size, `my-element` will be updated proportionally and automatically.

We really have to change our mind, if we want to grab the full power of the new browsers abilities.  We have to
**clear our mental cache and update ourself**.
