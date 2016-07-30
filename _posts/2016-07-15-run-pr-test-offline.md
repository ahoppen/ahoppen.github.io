---
layout: post
title: Testing swiftc locally before a pull request
updated: 2016-07-15 09:00
---

After I missed running a testsuite that existed on the CI, but weren't included in the normal `build-script`, I decided to take a closer look on which parameters the CI actually used and came up with the following script.

```
#!/bin/bash
~/swift/utils/build-script --test --validation-test --release --assertions --llbuild --swiftpm --skip-test-ios --skip-test-tvos --skip-test-watchos --reconfigure
if [ $? -eq 0 ]; then
  say Success
else
  say Failure
fi;
```

It runs both the normal and the validation test suite in release mode (this saves quite a lot of time!) and runs the LLVM and Swift package manager tests as well (I missed both of them once).

Since the tests in simulators are pretty slow and seldom uncover new errors (expecially if you're working more on the frontend side) I decided to exclude them.

Lastly, I decided to go for `--reconfigure` just to make sure that old build scripts are not to blame for any failure.

So, now my pre-PR script runs about 10 to 20 minutes depending on how many changes I made since the last run and I can happily do other stuff until my MacBook tells me it is finished.