---
layout: post
title: A Guide To Full Screen WebView And Notch / Cutout

---

I am currently using an Android [phone](https://www.mi.com/global/mi-9-t-pro/) with regular-ish screen which is also my development device. So when I tested my game on a friend's Android phone with a notch, I realized that I had more work to do. Just as a background, my game runs in a WebView: the gameplay itself is rendered using WebGL and most of the UI is written in HTML. So if you are on the same boat (i.e. making a full screen app running in a WebView) and need to adapt to the notches and cutouts, this guide is for you.

## iOS

### Make Your WkWebView Real Fullscreen

There are several ways to achieve this — you can override `safeAreaInsets` for WkWebView (not recommended) or apply a offsetting `additionalSafeAreaInsets` to the `safeAreaInsets`. However, both approaches interfere with WkWebView's ability to set `safe-area-inset-*` [environment variables](https://developer.mozilla.org/en-US/docs/Web/CSS/env). The only approaches that works for me is to set `contentInsetAdjustmentBehavior` to `never`:

```swift
if #available(iOS 11.0, *) {
  webview!.scrollView.contentInsetAdjustmentBehavior = .never
}
```

### Safe Area Inset Variables in CSS

Then you need to add `viewport-fit=cover`  your viewport meta tag. And then your `safe-area-inset-*` CSS environment variables will be correctly set.

```html
<meta name="viewport" content="viewport-fit=cover" />
```

In your CSS, you can write something like

```css
.header {
  padding-top: env(safe-area-inset-top, 30px); /* 30px is our fallback value */
}
```

### Safe Area Inset Variables in JavaScript

Those CSS variables are not directly exposed to JavaScript. And there's no easy way to get them. However, you can get them through intermediaries. First in your CSS stylesheets, add something like:

```css
:root {
  --sait: env(safe-area-inset-top);
  --saib: env(safe-area-inset-top);
  --sail: env(safe-area-inset-top);
  --sair: env(safe-area-inset-top);
}
```

For those of you who are not familiar with CSS variables (like myself as last time I did any CSS was more than 8 years ago), we have defined 4 CSS variables whose values are equal to their environment counterparts. Now you can get them in JavaScript like this:

```javascript
getComputedStyle(document.documentElement).getPropertyValue("--sait") // 36px
```

Now you just need to adjust your layouts and UIs with these values.

## Android

If you think iOS looks tedious, then welcome to the Android world. First, after countless of trying, I cannot even get `safe-area-inset-*` correctly set for `WebView`. But we can start by making WebView fullscreen.

### Make Your WebView Real Fullscreen

Assuming your web view is already in [full screen](https://developer.android.com/training/system-ui/immersive) mode by correctly setting `systemUiVisibility` flag, you just need to add this to your [AppTheme](https://developer.android.com/guide/topics/ui/look-and-feel/themes):

```xml
<item name="android:windowLayoutInDisplayCutoutMode">shortEdges</item>
```

### Exposing Display Cutout Insets from Native To WebView

```kotlin
webView.webViewClient = object : WebViewClient() {
  override fun onPageFinished(view: WebView, url: String) {
    super.onPageFinished(webView, url)
    if (VERSION.SDK_INT >= VERSION_CODES.P) {
      val displayCutout = window.decorView.rootWindowInsets?.displayCutout
      if (displayCutout != null) {
        val density = resources.displayMetrics.density
        // The `safeInsetLeft` here is real pixel, we need to convert to the same px as in browser.
        val safeInsetLeft = (displayCutout.safeInsetLeft / density).roundToInt()
        val safeInsetRight = (displayCutout.safeInsetRight / density).roundToInt()
        val safeInsetTop = (displayCutout.safeInsetTop / density).roundToInt()
        val safeInsetBottom = (displayCutout.safeInsetBottom / density).roundToInt()
        val safeAreaJs = """
      		document.documentElement.style.setProperty('--android-safe-area-inset-left', '${safeInsetLeft}px');
        	document.documentElement.style.setProperty('--android-safe-area-inset-right', '${safeInsetRight}px');
        	document.documentElement.style.setProperty('--android-safe-area-inset-top', '${safeInsetTop}px');
        	document.documentElement.style.setProperty('--android-safe-area-inset-bottom', '${safeInsetBottom}px');
        """.trimIndent()
      	webview.evaluateJavascript(safeAreaJs, null)
      }
    }
  }
}
```

This is what the above code does: it injects some JavaScript when the page finishes loading, which in turn defines some CSS variables that has the correct cutout inset values. You are asking why not directly override the `--sait` variables defined earlier? Because `onPageFinished` doesn't mean your DOM is ready. So depending on how do you load your stylesheets, *your "overrides" might be overridden.*

### Update Inset Variables in CSS and JavaScript

Then you need to update your styles earlier to the following:

```css
:root {
  --sait: var(--android-safe-area-inset-top, env(safe-area-inset-top));
  --saib: var(--android-safe-area-inset-bottom, env(safe-area-inset-bottom));
  --sail: var(--android-safe-area-inset-left, env(safe-area-inset-left));
  --sair: var(--android-safe-area-inset-right, env(safe-area-inset-right));
}
```

Now our `--sait` will have correct values for both iOS (`env(safe-area-inset-top)`) and Android (`var(--android-safe-area-inset-top)`). And then in JavaScript, the same method will also return correct value.

## Before And After

So after all these work, you can finally start to (re)design your game UI to adjust for the notch. I have briefly surveyed some popular games on App Store and Google Play. Most opt for a minimal approach by simply offsetting UI elements (HUD) a bit while leave the game camera rendering on the non-safe area. With CSS variables, this is actually quite simple to achieve. For elements rendered by WebGL (i.e. your game engine), you have some extra work to do. I use cocos2d, which has a `SafeArea` component but it doesn't work in WebView. So I've added my own component based on the [source code](https://github.com/cocos-creator/engine/blob/master/cocos2d/core/components/CCSafeArea.js). See the final result below (the first example is done via CSS, while the second is rendered by cocos2d).

![Notch Before & After](https://i.imgur.com/32t62yq.jpg)

