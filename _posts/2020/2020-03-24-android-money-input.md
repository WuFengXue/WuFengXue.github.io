---
layout: post
title: 'android 【符号+金额】输入（右对齐）的一种简单实现'
subtitle: '观察推测 + 源码验证'
date: 2020-03-24
categories: 技术
cover: '../../../assets/img/post_bg/20200324.jpeg'
tags: android UI
---


## 1. 产品需求

* 无输入时，提示：请输入金额
![hint](../../../assets/img/2020/03/money-input/hint.png)
* 有输入时显示：符号 + 金额（如：¥123）
* 要求右对齐
![input](../../../assets/img/2020/03/money-input/input.png)
* 要求结果只返回数字部分
![output](../../../assets/img/2020/03/money-input/output.png)

## 2. 默认实现

&#160; &#160; &#160; &#160;首先想到的方案：

* 【TextView（显示符号） + EditText（显示金额）】
* 有输入时显示符号，无输入时隐藏符号

&#160; &#160; &#160; &#160;写完后发现存在一个问题：当输入内容的宽度小于提示的宽度时，符号和金额会分离，如：¥&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;123。

![sol_default](../../../assets/img/2020/03/money-input/sol_default.png)

## 3. 修改字符串

&#160; &#160; &#160; &#160;之后想到可以尝试直接修改字符串：

* 【EditText（同时显示符号和金额）】
* 首次输入后自动在前面追加符号
* 删除最后一个输入后自动移除符号

&#160; &#160; &#160; &#160;经过一系列的修改，基本可以满足需求，但存在两个小缺点：

1）、获取数值时需要对字符串进行截取

```java
        // 吐司展示当前输入金额
        String moneyStr = etMoney.getText()
                .toString();
        toastMoney(TextUtils.isEmpty(moneyStr)
                ? 0.0d
                // 注意：此处需截取符号（首个字符）之后的内容
                : Double.parseDouble(moneyStr.substring(1)));
```

2）、光标可以切到符号之前（暂时无法解决，虽然可以禁止在符号之前输入数字）

![sol_modify_string](../../../assets/img/2020/03/money-input/sol_modify_string.png)

## 4.用图片表示符号

&#160; &#160; &#160; &#160;之后又想到一个思路：

* 【EditText（同时显示符号和金额）】
* 符号用图片实现（android:drawableStart）

&#160; &#160; &#160; &#160;但是在用 tools 属性查看布局的预览效果时，发现还是会出现分离问题。

## 5.修改提示（_本文的主题_）

&#160; &#160; &#160; &#160;虽然用图片的思路无效，但是期间我发现一个现象：图片好像刚好位于提示的左侧。在布局中调整提示的长度，最后的预览效果也证实了这个发现。

&#160; &#160; &#160; &#160;基于上述发现，我推测：EditText 的宽度 = Math.max(文本的宽度, 提示的宽度)，后来去查看[_源码_](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/java/android/widget/TextView.java#8525)，发现了类似的代码：

```java
@RemoteView
public class TextView extends View implements ViewTreeObserver.OnPreDrawListener {
    ……
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
             ……
             if (mHint != null) {
                ……
                if (hintWidth > width) {
                    width = hintWidth;
                }
             }
             ……
    }
    ……
}
```

&#160; &#160; &#160; &#160;于是新的思路出来了：

* 【TextView（显示符号） + EditText（显示金额）】
* __有输入时显示符号、置空提示，无输入时隐藏符号、还原提示（核心点）__

&#160; &#160; &#160; &#160;之后的实现便水到渠成了。

&#160; &#160; &#160; &#160;xml 布局如下：

```xml
            <!-- 金额符号 -->
            <TextView
                android:id="@+id/tv_money_symbol_modify_hint"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="@string/common_rmb_symbol"
                android:textColor="@android:color/black"
                android:textSize="14sp"
                android:visibility="gone"
                app:layout_constraintBaseline_toBaselineOf="@id/tv_label_modify_hint"
                app:layout_constraintRight_toLeftOf="@id/et_money_modify_hint" />

            <!-- 金额输入 -->
            <EditText
                android:id="@+id/et_money_modify_hint"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginEnd="20dp"
                android:background="@null"
                android:gravity="end"
                android:hint="@string/money_input_hint"
                android:inputType="numberDecimal"
                android:maxLines="1"
                android:textColor="@android:color/black"
                android:textSize="14sp"
                app:layout_constraintBaseline_toBaselineOf="@id/tv_label_modify_hint"
                app:layout_constraintRight_toRightOf="parent" />
```

&#160; &#160; &#160; &#160;java 代码如下：

```java
        // 获取控件
        final TextView tvMoneySymbol = findViewById(R.id.tv_money_symbol_modify_hint);
        final EditText etMoney = findViewById(R.id.et_money_modify_hint);
        // 设置输入变化监听
        etMoney.addTextChangedListener(new TextWatcher() {
            @Override
            public void beforeTextChanged(CharSequence charSequence, int i, int i1, int i2) {
                // do nothing
            }

            @Override
            public void onTextChanged(CharSequence charSequence, int i, int i1, int i2) {
                // do nothing
            }

            @Override
            public void afterTextChanged(Editable editable) {
                String text = editable.toString();
                // 只在无输入时显示符号
                tvMoneySymbol.setVisibility(TextUtils.isEmpty(text)
                        ? View.GONE : View.VISIBLE);
                // 在有输入时将提示置空，以解决符号和金额分离的问题（思路核心点）
                etMoney.setHint(TextUtils.isEmpty(text)
                        ? getString(R.string.money_input_hint) : "");
            }
        });
        // 设置提交按钮点击监听
        findViewById(R.id.btn_submit_modify_hint)
                .setOnClickListener(new View.OnClickListener() {
                    @Override
                    public void onClick(View view) {
                        // 吐司展示当前输入金额
                        String moneyStr = etMoney.getText()
                                .toString();
                        toastMoney(TextUtils.isEmpty(moneyStr)
                                ? 0.0d
                                : Double.parseDouble(moneyStr));
                    }
                });
```

## 6.完整的效果图

![money_input](../../../assets/img/2020/03/money-input/money_input.gif)

## 7.最后总结

&#160; &#160; &#160; &#160;本文主要讨论了实现“【符号+金额】输入（右对齐）”的几种思路，按照思路的生成顺序做了简单的描述，最终提出了一种简单的实现（动态设置提示）。

## 附：[配套源码](https://github.com/WuFengXue/AndroidLabs#金额输入)