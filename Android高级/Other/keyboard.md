按钮随软键盘弹出收起：弹出时会遮挡页面内容

```java
 //软键盘监听
        View root = binding.getRoot();
        ViewGroup.MarginLayoutParams lp = (ViewGroup.MarginLayoutParams) binding.btnSubmit.getLayoutParams();

        root.getViewTreeObserver().addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
            @Override
            public void onGlobalLayout() {
                Rect rect = new Rect();
                root.getWindowVisibleDisplayFrame(rect);
                int screenHeight = root.getRootView().getHeight();
                int keyboardHeight = screenHeight - rect.bottom;
                LgUtil.d(TAG + " screenHeight:" + screenHeight + " keyboardHeight:" + keyboardHeight + " rect.bootom:" + rect.bottom);

                // 提交按钮随键盘弹起
                if (keyboardHeight != lp.bottomMargin) {
                    lp.bottomMargin = keyboardHeight;
                    binding.btnSubmit.setLayoutParams(lp);
                }
                LgUtil.d(TAG + " btn.margin_bottom:" + lp.bottomMargin);
            }
        });
```
