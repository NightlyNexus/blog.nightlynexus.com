---
layout:     post
title:      Covariant return types for View’s onSaveInstanceState
date:       2016-08-08 09:14:09
summary:    When implementing custom Views that use the onSaveInstanceState hook, consider exposing the final Parcelable implementation.
categories: Android
---
When implementing custom Views that use the [onSaveInstanceState](https://developer.android.com/reference/android/view/View.html#onSaveInstanceState()) hook, consider exposing the final Parcelable implementation.

Knowing the final type of a Parcelable instance is [helpful](http://blog.nightlynexus.com/optimized-android-parcelable-reading-and-writing/), especially for consumers who save the View tree manually (not using Android's Window traversal). Consumers of a custom View who use the `onSaveInstanceState` hook can make use of this knowledge if `onSaveInstanceState` uses Java’s covariance and returns the View’s implementation of the Parcelabale saved state (e.g. `@Override public MyProfileView.SavedState onSaveInstanceState()`).
