---
layout:     post
title:      "InputMethodService输入法服务核心解析"
subtitle:   "Talk About Service"
date:       2020-06-02 19:48:00
author:     "Jacky Tallow"
header-img: "img/service-icon.jpg"
tags:
    - 聊聊
---

## InputMehodService核心
- 进行派生和定制
- InputMethodService继承至AbstractInputMethodService实现InputMehotd接口

## onInitializeInterface
- onInitializeInterface方法用于用户界面初始化，主要用于service运行过程中配置信息发生改变的情况（横竖屏转换等）。

## onStartInput
```
    public void onStartInputView(EditorInfo info, boolean restarting) {
        // Intentionally empty
    }
```

## onCreateInputView
- InputMethodService默认创建并返回的输入区域的视图为null
- 改变需要调用setInputView
```
    public void setInputView(View view) {
        mInputFrame.removeAllViews();
        mInputFrame.addView(view, new FrameLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT,
                ViewGroup.LayoutParams.WRAP_CONTENT));
        mInputView = view;
    }
```
- 控制何时显示输入区域
```
@CallSuper
    public boolean onEvaluateInputViewShown() {
        if (mSettingsObserver == null) {
            Log.w(TAG, "onEvaluateInputViewShown: mSettingsObserver must not be null here.");
            return false;
        }
        if (mSettingsObserver.shouldShowImeWithHardKeyboard()) {
            return true;
        }
        Configuration config = getResources().getConfiguration();
        return config.keyboard == Configuration.KEYBOARD_NOKEYS
                || config.hardKeyboardHidden == Configuration.HARDKEYBOARDHIDDEN_YES;
    }
```
- 有了控制何时显示的方法之后updateInputViewShown方法,判断是否显示输入区域
```
 public void updateInputViewShown() {
        boolean isShown = mShowInputRequested && onEvaluateInputViewShown();
        if (mIsInputViewShown != isShown && mWindowVisible) {
            mIsInputViewShown = isShown;
            mInputFrame.setVisibility(isShown ? View.VISIBLE : View.GONE);
            if (mInputView == null) {
                initialize();
                View v = onCreateInputView();
                if (v != null) {
                    setInputView(v);
                }
            }
        }
    }
```
- 它是调用了onEvaluateInputViewShown来判断的
```
   public boolean onEvaluateInputViewShown() {
        if (mSettingsObserver == null) {
            Log.w(TAG, "onEvaluateInputViewShown: mSettingsObserver must not be null here.");
            return false;
        }
        if (mSettingsObserver.shouldShowImeWithHardKeyboard()) {
            return true;
        }
        Configuration config = getResources().getConfiguration();
        return config.keyboard == Configuration.KEYBOARD_NOKEYS
                || config.hardKeyboardHidden == Configuration.HARDKEYBOARDHIDDEN_YES;
    }
```

- onCreateCandidatesView方法用于创建并返回（candidate view）候选词区域的层次视图，该方法只被调用一次（候选次区域第一次显示时），方法可以返回null，此时输入法不存在候选词区域，InputMethodService的默认方法实现返回值为空。
想要改变已经创建的候选词区域视图，我们可以调用setCandidatesView(View)方法，想要控制何时显示候选词视图，我们可以实现setCandidatesViewShown（boolean）方法
```
   public View onCreateCandidatesView() {
        return null; //表示不存在候选词
    }
```
```
    public void setCandidatesView(View view) {
        mCandidatesFrame.removeAllViews();
        mCandidatesFrame.addView(view, new FrameLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT,
                ViewGroup.LayoutParams.WRAP_CONTENT));
    }
```
```
 public void setCandidatesViewShown(boolean shown) {
        updateCandidatesVisibility(shown);
        if (!mShowInputRequested && mWindowVisible != shown) {
            // If we are being asked to show the candidates view while the app
            // has not asked for the input view to be shown, then we need
            // to update whether the window is shown.
            if (shown) {
                showWindow(false);
            } else {
                doHideWindow();
            }
        }
    }
```

## onCreateExtractTextView
- 在输入法全屏模式下使用调用，返回用于显示文本信息的区域
```
   public View onCreateExtractTextView() {
        return mInflater.inflate(
                com.android.internal.R.layout.input_method_extract_view, null);
    }
```

## 开始调用输入法视图后，在编辑框输入的时候，开始调用onStartInputView，在调用完之后才开始调用
```
public void onStartInput(EditorInfo attribute, boolean restarting) {
        // Intentionally empty
    }
```
- 普通的设置可以在onStartInput方法中进行，在onStartInputView方法中进行视图相关的设置，开发者应该保证onCreateInputView方法在该方法被调用之前调用

- onStartCandidatesView方法 候选词视图已经显示时回调该函数，该方法会在onStartInput方法之后被回调，普通的设置可以在onStartInput方法中进行，在onStartCandidatesView方法中进行视图相关的设置，开发者应该保证onCreateCandidatesView方法在该方法被调用之前调用。
```
 public void onStartCandidatesView(EditorInfo info, boolean restarting) {
        // Intentionally empty
    }
```
onFinishCandidatesView（boolean finishingInput）方法 当候选词视图即将被隐藏或者切换到另外的编辑框时调用该方法，finishingInput为true，onFinishInput方法会接着被调用。
```
public void onFinishCandidatesView(boolean finishingInput) {
        if (!finishingInput) {
            InputConnection ic = getCurrentInputConnection();
            if (ic != null) {
                ic.finishComposingText();
            }
        }
    }
```

- onFinishInputView（boolean finishingInput）方法 当候选词视图即将被隐藏或者切换到另外的编辑框时调用该方法，finishingInput为true，onFinishInput方法会接着被调用。
```
    public void onFinishInputView(boolean finishingInput) {
        if (!finishingInput) {
            InputConnection ic = getCurrentInputConnection();
            if (ic != null) {
                ic.finishComposingText();
            }
        }
    }
```
- onFinishInput方法 通知输入法上一个编辑框的文本输入结束，这时，输入法可能会接着调用onStartInput方法来在新的编辑框中进行输入，或者输入法处于闲置状态，当输入法在同一个编辑框中重启时不会调用该方法
```
public void onFinishInput() {
        InputConnection ic = getCurrentInputConnection();
        if (ic != null) {
            ic.finishComposingText();
        }
    }
```

onUnbindInput方法 当绑定的客户端和当前输入法失去联系时调用该方法，通过调用getCurrentInputBinding方法和getCurrentInputConnection方法来判断是否存在联系，若两个方法的返回值无效，则客户端与当前输入法失去联系。
```
  @Override
        public void unbindInput() {
            if (DEBUG) Log.v(TAG, "unbindInput(): binding=" + mInputBinding
                    + " ic=" + mInputConnection);
            onUnbindInput();
            mInputBinding = null;
            mInputConnection = null;
        }
        
        public InputBinding getCurrentInputBinding() {
        return mInputBinding;
        }
```

# 以上是InputMethodService的输入视图，候选词视图，全屏模式

## InputMethodService提供了一个基本的UI元素架构（包括输入视图，候选词视图，全屏模式），但是开发者可以决定各个UI元素如何实现，例如，我们可以用键盘输入的方式实现一个输入视图，也可以用手写的方式来实现输入视图，所有的UI元素都被集成在了InputMethodService提供的窗口中，具体的UI元素包括：
1. soft input view（输入视图）：放置在屏幕底部
2. the candidates view（候选词视图）：放置在输入视图上面
3. extracted text view（提取文本视图）：存在于全屏模式，如果输入法没有运行在全屏模式，客户端会移动或重新调整大小以保证客户端视图在输入法视图的上面，如果输入法运行在全屏模式，输入法窗口会完全覆盖客户端窗口，此时，输入法窗口中会包含extracted
4. text视图，该视图中包含客户端正在被编辑的文字信息。

IME最重要的部分就是对应用产生文本信息，这通过调用InputConnetion接口来实现，我们可以通过getCurrentInputConnection方法来得到InputConnection接口，通过该接口可以产生raw key事件，如果目标编辑框支持，我们可以直接编辑候选词字符串并提交。
- 我们可以从EditorInfo类中获得目标编辑框期望并支持的输入格式，通过调用getCurrentInputEditorInfo方法来获取EditorInfo类，其中最重要的是EditorInfo.inputType，如果inputType的值为EditorInfo.TYPE_NULL，则该目标编辑框不支持复杂的信息，只支持原始的按键信息（字母，数字，符号，不支持表情，汉字等合成信息），Type类型可以是password（密码类型）、电话号码类型等。当用户在不同编辑框之间切换时，输入法框架会调用onFinishInput方法和onStartInput方法，你可以通过这两个方法来重置或初始化输入状态。

## InputMethodService公有方法介绍：
1. public boolean enableHardwareAcceleration()
- 该方法在API-21中被舍弃，从API-21开始，硬件加速功能在支持的设备上始终启用，该方法必须在onCreate方法之前被调用，所以，你可以在构造函数中调用该方法。
```
   public boolean enableHardwareAcceleration() {
        if (mWindow != null) {
            throw new IllegalStateException("Must be called before onCreate()");
        }
        return ActivityManager.isHighEndGfx();
    }
```
2. public int getCandidatesHiddenVisibility（）
- 当候选词区域未显示时，调用该函数获取候选词区域的显示模式（INVISIBLE 或者 GONE），默认的实现会调用isExtractViewShown方法，若isExtractViewShown方法返回true，getCandidatesHiddenVisibility方法返回值为GONE，否则返回值为INVISIBLE。
3. public boolean isExtractViewShown（）
- 返回是否显示extract view视图，该方法只在isFullscreenMode方法返回true的情况下才有可能返回true。这种情况下，是否显示extract,view主要决定于上次调用的setExtractViewShown（boolean）方法。
4. public boolean isFullscreenMode（）
- 判断当前输入法是否运行在全屏模式，该方法的返回值决定于updateFullscreenMode（）。
public void updateFullscreenMode（）：评估当前输入法是否应该运行在全屏模式，当模式改变时重画UI组件，该方法会调用onEvaluateFullscreenMode方法来决定是否开启全屏模式，用户可以通过isFullscreenMode方法来判断当前是否运行在全屏模式。
```
 public void updateFullscreenMode() {
        boolean isFullscreen = mShowInputRequested && onEvaluateFullscreenMode();
        boolean changed = mLastShowInputRequested != mShowInputRequested;
        if (mIsFullscreen != isFullscreen || !mFullscreenApplied) {
            changed = true;
            mIsFullscreen = isFullscreen;
            if (mImm != null && mToken != null) {
                mImm.reportFullscreenMode(mToken, mIsFullscreen);
            }
            mFullscreenApplied = true;
            initialize();
            LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams)
                    mFullscreenArea.getLayoutParams();
            if (isFullscreen) {
                mFullscreenArea.setBackgroundDrawable(mThemeAttrs.getDrawable(
                        com.android.internal.R.styleable.InputMethodService_imeFullscreenBackground));
                lp.height = 0;
                lp.weight = 1;
            } else {
                mFullscreenArea.setBackgroundDrawable(null);
                lp.height = LinearLayout.LayoutParams.WRAP_CONTENT;
                lp.weight = 0;
            }
            ((ViewGroup)mFullscreenArea.getParent()).updateViewLayout(
                    mFullscreenArea, lp);
            if (isFullscreen) {
                if (mExtractView == null) {
                    View v = onCreateExtractTextView();
                    if (v != null) {
                        setExtractView(v);
                    }
                }
                startExtractingText(false);
            }
            updateExtractFrameVisibility();
        }
        
        if (changed) {
            onConfigureWindow(mWindow.getWindow(), isFullscreen, !mShowInputRequested);
            mLastShowInputRequested = mShowInputRequested;
        }
    }
```
5. public InputBinding getCurentInputBinding ()：
- 返回当前活跃的InputBinding，不存在返回null。
```
    public InputBinding getCurrentInputBinding() {
        return mInputBinding;
    }
```


6. public InputConnectionb getCurrentInputConnection()：
- 返回当前活跃的InputConnection，不存在返回null。
```
   public InputConnection getCurrentInputConnection() {
        InputConnection ic = mStartedInputConnection;
        if (ic != null) {
            return ic;
        }
        return mInputConnection;
    }
```
7. public EditorInfo getCurrentInputEditorInfo()：
- 获取编辑框信息。
8. public CharSequence getTextForImeAction（int imeOptions）：
- 返回可以应用于EditorInfo.imeOptions的按钮标签。
9. public boolean isShowInputRequested（）：
- 如果输入法被请求显示输入视图，返回值为true。
10. public void onConfigurationChaged（Configuration reconfigure）：
- 用于处理配置信息改变，InputMethodService的子类一般不需要直接处理配置改变信息，在标准的实现中，当配置信息改变时，该方法会调用相关的UI方法，所以，你可以依靠onCreateInputView等方法来处理配置信息改变的情况。
11. public void onDisplayCompletions（CompletionInfo[] completions）：
- 当客户端报告auto-completion候选词需要输入法显示的时候回调该函数，典型应用于全屏模式，默认的实现不做任何事情。
12. public boolean onEvaluateFullscreenMode()：
- 重写该方法来控制输入法何时运行在全屏模式，默认的实现在landscape mode（横屏模式）时，输入法运行在全屏模式，如果你改变了该方法的返回值，那么你需要在该返回值可能改变时调用updateFullscreenMode()方法，该方法会调用onEvaluateFullscreenMode函数，并当返回值与上一次不同时重绘UI组建
13. public boolean onEvaluateInputViewShown()：
- 重写该方法来控制何时向用户显示输入区域，默认的实现在不存在硬键盘或键盘被遮盖时显示软件盘输入区域，你应该在该方法的返回值可能改变时调用updateInputViewShown()方法
```

```
14. public void onExtractedSelectionChanged（int start，int end）：
- 用户在extractedtext视图中移动光标时调用该方法，默认的实现会把对应的信息传递给底层的编辑框。
15. public void onExtractedTextClicked（）：当用户点击extracted text视图时调用该方法，默认的实现会隐藏候选词视图，当extracted text视图存在竖直滚动条时。
16. public void onExtractingInputChanged（EditorInfo ei）：
- 全屏模式时，输入对象改变时，调用该方法，默认的实现当新的对象 is not a full editor，自动隐藏IME。
- onKeyDown、onKeyLongPress、onKeyMultiple、onKeyUp用于劫持按键信息，返回值为ture，则代表此消息已经被消费

### onKeyDown
```
 public boolean onKeyDown(int keyCode, KeyEvent event) {
        if (event.getKeyCode() == KeyEvent.KEYCODE_BACK) {
            final ExtractEditText eet = getExtractEditTextIfVisible();
            if (eet != null && eet.handleBackInTextActionModeIfNeeded(event)) {
                return true;
            }
            if (handleBack(false)) {
                event.startTracking();
                return true;
            }
            return false;
        }
        return doMovementKey(keyCode, event, MOVEMENT_DOWN);
    }
```
### onKeyLongPress
```
 public boolean onKeyLongPress(int keyCode, KeyEvent event) {
        return false;
    }
```
### onKeyMultipe
```
public boolean onKeyMultiple(int keyCode, int count, KeyEvent event) {
        return doMovementKey(keyCode, event, count);
    }
```
### onKeyUp
```
   public boolean onKeyUp(int keyCode, KeyEvent event) {
        if (event.getKeyCode() == KeyEvent.KEYCODE_BACK) {
            final ExtractEditText eet = getExtractEditTextIfVisible();
            if (eet != null && eet.handleBackInTextActionModeIfNeeded(event)) {
                return true;
            }
            if (event.isTracking() && !event.isCanceled()) {
                return handleBack(true);
            }
        }
        return doMovementKey(keyCode, event, MOVEMENT_UP);
    }
```

17. public void onUpdateExtractedText（int token，ExtractedText text）：
当应用报告新的extracted text信息需要显示时调用该方法，默认的实现会把新的文本放到extract edit视图中。
```
 public void onUpdateExtractedText(int token, ExtractedText text) {
        if (mExtractedToken != token) {
            return;
        }
        if (text != null) {
            if (mExtractEditText != null) {
                mExtractedText = text;
                mExtractEditText.setExtractedText(text);
            }
        }
    }
```
18. public void onUpdateExtractingViews（EditorInfo ei）：
- 全屏模式下当editor信息改变时，回调该函数，用于更新UI信息，例如如何显示按钮等
```
public void onUpdateExtractingViews(EditorInfo ei) {
        if (!isExtractViewShown()) {
            return;
        }
        
        if (mExtractAccessories == null) {
            return;
        }
        final boolean hasAction = ei.actionLabel != null || (
                (ei.imeOptions&EditorInfo.IME_MASK_ACTION) != EditorInfo.IME_ACTION_NONE &&
                (ei.imeOptions&EditorInfo.IME_FLAG_NO_ACCESSORY_ACTION) == 0 &&
                ei.inputType != InputType.TYPE_NULL);
        if (hasAction) {
            mExtractAccessories.setVisibility(View.VISIBLE);
            if (mExtractAction != null) {
                if (mExtractAction instanceof ImageButton) {
                    ((ImageButton) mExtractAction)
                            .setImageResource(getIconForImeAction(ei.imeOptions));
                    if (ei.actionLabel != null) {
                        mExtractAction.setContentDescription(ei.actionLabel);
                    } else {
                        mExtractAction.setContentDescription(getTextForImeAction(ei.imeOptions));
                    }
                } else {
                    if (ei.actionLabel != null) {
                        ((TextView) mExtractAction).setText(ei.actionLabel);
                    } else {
                        ((TextView) mExtractAction).setText(getTextForImeAction(ei.imeOptions));
                    }
                }
                mExtractAction.setOnClickListener(mActionClickListener);
            }
        } else {
            mExtractAccessories.setVisibility(View.GONE);
            if (mExtractAction != null) {
                mExtractAction.setOnClickListener(null);
            }
        }
    }
```
19. public void onUpdateExtractingVisibility（EditorInfo ei）：
- 全屏模式下当editor信息改变时，回调该函数，用于判断extracting（extract text和candidates）是否显示。

20. public void onUpdateSelection（int oldSelStart，int oldSelEnd，int newSelStrat，int newSelEnd，int candidatesStart，int candidatesEnd）：
- 当应用报告新的文本选择区域时回调该函数，无论输入法是否请求更新extracted text，onUpdateSelection都会被调用，尽管如此，当extracted text也改变的时候，输入法不会调用onUpdateSelection，在该方法中要小心处理setComposingText、commitText或者deleteSurroundingText方法，可能引起死循环。
21. public void onWindowHidden（）：
- 当输入法界面已经被隐藏时调用该方法
```
 public void onWindowHidden() {
        // Intentionally empty
    }
```
22. public void onWindowShown（）：
- 当输入法界面已经显示给用户时调用该方法
```
 public void onWindowHidden() {
        // Intentionally empty
    }
```
23. public void requestHideSelf（int flags）：
- 关闭输入法界面，输入法还在运行，但是用户不能通过触控屏幕来输入信息
```
 public final void requestShowSelf(int flags) {
        mImm.showSoftInputFromInputMethodInternal(mToken, flags);
    }
```
24. public void sendDownUpKeyEvents（int keyEventCode）：
- 设置键盘相关事件
```
 public void sendDownUpKeyEvents(int keyEventCode) {
        InputConnection ic = getCurrentInputConnection();
        if (ic == null) return;
        long eventTime = SystemClock.uptimeMillis();
        ic.sendKeyEvent(new KeyEvent(eventTime, eventTime,
                KeyEvent.ACTION_DOWN, keyEventCode, 0, 0, KeyCharacterMap.VIRTUAL_KEYBOARD, 0,
                KeyEvent.FLAG_SOFT_KEYBOARD|KeyEvent.FLAG_KEEP_TOUCH_MODE));
        ic.sendKeyEvent(new KeyEvent(eventTime, SystemClock.uptimeMillis(),
                KeyEvent.ACTION_UP, keyEventCode, 0, 0, KeyCharacterMap.VIRTUAL_KEYBOARD, 0,
                KeyEvent.FLAG_SOFT_KEYBOARD|KeyEvent.FLAG_KEEP_TOUCH_MODE));
    }
```
25. public void sendKeyChar（char charCode）：
- 设置键盘keyCode
```
public void sendKeyChar(char charCode) {
        switch (charCode) {
            case '\n': // Apps may be listening to an enter key to perform an action
                if (!sendDefaultEditorAction(true)) {
                    sendDownUpKeyEvents(KeyEvent.KEYCODE_ENTER);
                }
                break;
            default:
                // Make sure that digits go through any text watcher on the client side.
                if (charCode >= '0' && charCode <= '9') {
                    sendDownUpKeyEvents(charCode - '0' + KeyEvent.KEYCODE_0);
                } else {
                    InputConnection ic = getCurrentInputConnection();
                    if (ic != null) {
                        ic.commitText(String.valueOf(charCode), 1);
                    }
                }
                break;
        }
    }
```