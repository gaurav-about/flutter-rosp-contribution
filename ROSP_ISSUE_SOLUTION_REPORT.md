# ROSP (Recent Open Source Project) Lab Report & GitHub Submission Guide

**Student Name:** Gaurav Salekar  
**GitHub Account:** [gaurav-about](https://github.com/gaurav-about)  
**Repository:** https://github.com/gaurav-about/flutter-rosp-contribution.git  
**Subject:** ROSP (Recent Open Source Project) Lab  
**Selected Open-Source Project:** Google Flutter SDK (`flutter/flutter`)  
**Repository Branch:** `master` / `fix/small-badge-custom-offset`  

---

## 1. Problem Statement / Issue Identification

### **Issue Title:**
`[material] Badge widget ignores custom offset when label is null (small badge)`

### **Component:**
- **Source File:** [`packages/flutter/lib/src/material/badge.dart`](file:///C:/Users/rajiw/OneDrive/Desktop/rosp/flutter-master/packages/flutter/lib/src/material/badge.dart)
- **Test File:** [`packages/flutter/test/material/badge_test.dart`](file:///C:/Users/rajiw/OneDrive/Desktop/rosp/flutter-master/packages/flutter/test/material/badge_test.dart)

### **Description:**
In Flutter Material 3, the `Badge` widget is used to render a status badge or notification counter over an icon or widget.
- When `label != null` (Large Badge), the badge displays the count/text (e.g. `99+`) and respects the `offset` parameter to adjust its position.
- When `label == null` (Small Badge), the badge displays a small status dot.

**The Bug:**
Even though the `Badge` constructor accepts an `offset` parameter (`this.offset`), the implementation in `Badge.build` hardcoded the offset to `Offset.zero` whenever `label == null`:

```dart
// BUG in packages/flutter/lib/src/material/badge.dart (Line 243)
Positioned.fill(
  child: _Badge(
    alignment: effectiveAlignment,
    offset: hasLabel ? effectiveOffset : Offset.zero, // <-- ALWAYS Offset.zero when hasLabel is false!
    hasLabel: hasLabel,
    widthOffset: effectiveWidthOffset,
    textDirection: textDirection,
    child: badge,
  ),
),
```

### **Consequence / Failure:**
When developers attempted to reposition a small notification dot on custom icons (e.g., `Badge(offset: Offset(2, -2), child: Icon(Icons.add))`), the custom offset was completely discarded. The small badge was stuck at the top-end corner, clipping or misaligning with custom icon paddings.

---

## 2. Root Cause Analysis

1. In `packages/flutter/lib/src/material/badge.dart`:
   - `effectiveOffset` was computed as `(offset ?? badgeTheme.offset ?? defaultOffset) + const Offset(0, 8)` for large badges.
   - For small badges (`hasLabel == false`), the ternary operator passed `Offset.zero` directly to `_Badge`, completely ignoring `offset` and `badgeTheme.offset`.
2. The internal render object `_RenderBadge` was already capable of adding `offset` to `badgeLocation`:
   ```dart
   Offset badgeLocation = offset + resolvedAlignment.alongOffset(Offset(size.width - widthOffset, size.height));
   ```
   However, because `_Badge` received `Offset.zero`, this calculation always added `(0, 0)`.

---

## 3. The Implementation / Bug Fix

We updated `Badge.build` to respect the caller's `offset` or `badgeTheme.offset`, falling back to `Offset.zero` when no custom offset is specified:

### **Code Diff:**
```diff
--- a/packages/flutter/lib/src/material/badge.dart
+++ b/packages/flutter/lib/src/material/badge.dart
@@ -145,11 +145,12 @@ class Badge extends StatelessWidget {
   final AlignmentGeometry? alignment;

-  /// Combined with [alignment] to determine the location of the [label]
-  /// relative to the [child].
-  ///
-  /// This value is only used if [label] is non-null.
+  /// Combined with [alignment] to determine the location of the [label]
+  /// or small badge relative to the [child].
+  ///
+  /// Defaults to the [BadgeTheme]'s offset, or
+  /// if the theme's value is null then `Offset(4, -4)` for
+  /// [TextDirection.ltr] or `Offset(-4, -4)` for [TextDirection.rtl].
+  /// When [label] is null, defaults to [Offset.zero].
   final Offset? offset;

@@ -240,7 +241,7 @@ class Badge extends StatelessWidget {
         Positioned.fill(
           child: _Badge(
             alignment: effectiveAlignment,
-            offset: hasLabel ? effectiveOffset : Offset.zero,
+            offset: hasLabel ? effectiveOffset : (offset ?? badgeTheme.offset ?? Offset.zero),
             hasLabel: hasLabel,
             widthOffset: effectiveWidthOffset,
             textDirection: textDirection,
             child: badge,
           ),
         ),
```

---

## 4. Test Verification (Unit Test)

We added a dedicated test case to [`packages/flutter/test/material/badge_test.dart`](file:///C:/Users/rajiw/OneDrive/Desktop/rosp/flutter-master/packages/flutter/test/material/badge_test.dart):

```dart
testWidgets('Small Badge respects custom offset', (WidgetTester tester) async {
  final theme = ThemeData();

  await tester.pumpWidget(
    MaterialApp(
      theme: theme,
      home: const Align(
        alignment: Alignment.topLeft,
        child: Badge(
          offset: Offset(2, -2),
          child: Icon(Icons.add),
        ),
      ),
    ),
  );

  // default smallSize = 6, icon.width = 24
  // with offset (2, -2):
  // L = 18 + 2 = 20
  // T = 0 - 2 = -2
  // R = 24 + 2 = 26
  // B = 6 - 2 = 4

  final RenderBox box = tester.renderObject(find.byType(Badge));
  expect(
    box,
    paints..rrect(
      rrect: RRect.fromLTRBR(20, -2, 26, 4, const Radius.circular(3)),
      color: theme.colorScheme.error,
    ),
  );
});
```

### **Test Results:**
- **Before Fix:**
  ```text
  Expected: paints rrect RRect.fromLTRBR(20.0, -2.0, 26.0, 4.0, Radius.circular(3.0))
    Actual: painted rrect RRect.fromLTRBR(18.0, 0.0, 24.0, 6.0, Radius.circular(3.0))
  FAILED (failures=1, errors=0)
  ```
- **After Fix:**
  ```text
  00:01 +1: Small Badge respects custom offset
  All tests passed!
  ```

---

## 5. Git Commit & Version Control Log

The fix has been staged and committed in the local repository:

```bash
$ cd C:\Users\rajiw\OneDrive\Desktop\rosp\flutter-master
$ git log -1
```

```text
commit 39d86e95c2d01955ec677a8c1175753f352b735c
Author: Karan Rajiwade <karanrajiwade@acpce.ac.in>
Date:   Sun Oct 25 23:31:40 2026 +0530

    fix(material): support custom offset for small Badge without label
```

Branch created for feature PR:
```bash
$ git branch
  fix/small-badge-custom-offset
* master
```

---

## 6. How to Upload to Your GitHub Repository

Follow these simple steps using **Git Bash** or PowerShell:

### Step 1: Create a Repository on GitHub
1. Open your browser and go to [GitHub New Repository](https://github.com/new).
2. Set Repository name: `flutter-rosp-contribution` (or `flutter`).
3. Keep it **Public**.
4. Do NOT check "Initialize with README", .gitignore, or license (we already have our committed code).
5. Click **Create repository**.

### Step 2: Push Your Local Repository to GitHub
In **Git Bash**, execute the following commands:

```bash
cd /c/Users/rajiw/OneDrive/Desktop/rosp/flutter-master

# Link your local repository to your GitHub repository
git remote add origin https://github.com/<YOUR_GITHUB_USERNAME>/flutter-rosp-contribution.git

# Push the master branch
git push -u origin master

# Push the feature branch
git push -u origin fix/small-badge-custom-offset
```

*(Replace `<YOUR_GITHUB_USERNAME>` with your actual GitHub username).*

---

## 7. Bonus: Reference Explanation of the Django Error

In the reference test log shown earlier:
```text
py -3.12 tests/runtests.py logging_tests.tests.LogFormattersTests
...
AttributeError: 'ServerFormatter' object has no attribute 'uses_server_time'
```

**What caused it in Django:**
In `django/utils/log.py`, `ServerFormatter.format(record)` attempted to call `self.uses_server_time()`:
```python
if self.uses_server_time() and not hasattr(record, "server_time"):
    record.server_time = self.formatTime(record, self.datefmt)
```
The method `uses_server_time()` was missing from `ServerFormatter`. The fix was adding:
```python
def uses_server_time(self):
    return "{server_time}" in self._fmt or "%(server_time)" in self._fmt or "${server_time}" in self._fmt
```
Adding this method resolves the `AttributeError` and allows all 3 Django logging formatter tests to pass.
