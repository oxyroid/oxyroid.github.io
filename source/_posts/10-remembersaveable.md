---
title: rememberSaveable in compose
date: 2023-11-25 14:09:55
tags: [kotlin, compose]
---

The `rememberSaveable` API functions similarly to `remember`, as explained in the official documentation:

> The `rememberSaveable` API retains state across recompositions and even persists through activity or process recreation using the saved instance state mechanism.

## In which situations is it more appropriate to use `remember {}` instead of `rememberSaveable {}` + `Saver`?

`RememberSaveable` restricts you to what is possible in the `Bundle`. If your data fits within these constraints, `rememberSaveable` is suitable. However, `remember {}` allows more flexibility.

## Compose allows customizing savers, enabling the conversion of any objects to a bundle (for Android platform). It means `rememberSaveable` with customizing savers is better forever than `remember`?

No, dealing with large objects that may be challenging.

## Are there specific scenarios where `remember` is better than `rememberSaveable`?

Yes, required in certain composables, is consistently driven by external data. In such cases, a saver might be unnecessary. Conversely, states reset due to screen rotation may necessitate a saver, such as for an integer tracking button clicks, which shouldn’t be saved to the ViewModel.

By carefully considering these factors, you can choose between `remember` and `rememberSaveable` effectively based on your specific use case and requirements.
