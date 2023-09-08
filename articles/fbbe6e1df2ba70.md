---
title: "枠なし NSWindow でトラックパッドのジェスチャの通知を受け取る"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["macOS", "Cocoa"]
published: true
---

## TL;DR

`NSWindow` を使って作った枠なしのウインドウでトラックパッドのジェスチャの通知を受け取りたければ、`NSWindow` クラスの `canBecomeKeyWindow` メソッドをオーバーライドして、`YES` を返すようにした上で、そのウインドウをアクティブにすると良いっぽい。

## 本編

通常、`NSWindow` を利用して作ったウインドウでは、そのウインドウの delegate のクラスで [`magnifyWithEvent`](https://developer.apple.com/documentation/appkit/nsresponder/1525862-magnifywithevent) や [`rotateWithEvent`](https://developer.apple.com/documentation/appkit/nsresponder/1525572-rotatewithevent) や [`swipeWithEvent`](https://developer.apple.com/documentation/appkit/nsresponder/1524275-swipewithevent) を実装することで、例えば二本指での拡大縮小や回転などの、トラックパッド上でジェスチャが行われたという情報を受け取ることができます。

```objc:main.m
#import <AppKit/AppKit.h>

@interface AppDelegate : NSObject<NSApplicationDelegate>
@end

@interface Window : NSWindow
@end

@implementation AppDelegate
- (void)applicationDidFinishLaunching:(NSNotification *)aNotification {
    static Window *window;
    window = [[Window alloc]
        initWithContentRect:NSMakeRect(0, 0, 400, 200)
                  styleMask:NSWindowStyleMaskTitled | NSWindowStyleMaskClosable
                    backing:NSBackingStoreBuffered
                      defer:NO];
    [window setTitle:@"test window"];
    [window orderFront:window];
    [window center];
}
@end

@implementation Window
-(void)scrollWheel:(NSEvent *)event {  // ウインドウ上でスクロールしたら呼ばれる
    NSLog(@"scrollWheel");
}
-(void)magnifyWithEvent:(NSEvent *)event {  // ウインドウ上でピンチズームをしたら呼ばれる
    NSLog(@"magnifyWithEvent");
}
@end

int main(void) {
    [NSApplication sharedApplication];
    [NSApp setDelegate:[[AppDelegate alloc] init]];
    [NSApp setActivationPolicy:NSApplicationActivationPolicyRegular];
    [NSApp activateIgnoringOtherApps:YES];
    [NSApp run];
}
```

ただ、ここで以下のようにウインドウを枠なし（タイトルバーなし？）にすると、`scrollWheel` 関数は依然呼び出されるもののなぜか `magnifyWithEvent` 関数が呼ばれなくなります。

```diff
diff --git a/main.m b/main.m
index b9419b7..8b8fd09 100644
--- a/main.m
+++ b/main.m
@@ -11,7 +11,7 @@
     static Window *window;
     window = [[Window alloc]
         initWithContentRect:NSMakeRect(0, 0, 400, 200)
-                  styleMask:NSWindowStyleMaskTitled | NSWindowStyleMaskClosable
+                  styleMask:NSWindowStyleMaskBorderless
                     backing:NSBackingStoreBuffered
                       defer:NO];
     [window setTitle:@"test window"];
```

詳しいことはよくわからないのですが、調べてみた感じだとどうやら `magnifyWithEvent` 等のトラックパッドジェスチャの通知はウインドウがキーウインドウで、かつアクティブな時のみしか飛んでこないようです。
以下のように `canBecomeKeyWindow` メソッドを `YES` を返すようにオーバーライドすると、再度 `magnifyWithEvent` が呼ばれるようになりました。

```diff
diff --git a/main.m b/main.m
index 8b8fd09..99aa31a 100644
--- a/main.m
+++ b/main.m
@@ -27,6 +27,9 @@
 -(void)magnifyWithEvent:(NSEvent *)event {
     NSLog(@"magnifyWithEvent");
 }
+-(BOOL)canBecomeKeyWindow {
+    return YES;
+}
 @end
 
 int main(void) {
```

## まとめ

`NSWindow` クラスの `canBecomeKeyWindow` メソッドが `YES` を返すようにオーバーライドすることで、枠なしのウインドウでもトラックパッドのジェスチャが行われた時に通知が飛んでくるようになります。
Cocoa ド素人なので、もしいろいろ変なことを書いてたらすみません。



## References

- https://developer.apple.com/documentation/appkit/nsresponder/1525862-magnifywithevent?language=objc
- https://developer.apple.com/documentation/appkit/nswindow/1419543-canbecomekeywindow
- https://developer.apple.com/documentation/appkit/nswindowstylemask/nswindowstylemaskborderless
