---
title: common python code
date: 2024-07-17
categories:
    - :rainbow:Topics
    - :writing_hand_tone1:Python
---

# 常用 Python 代码备忘

<!-- more -->

### `loguru logger -> stderr`

```Python
import sys
from loguru import logger
logger.remove()
logger.add(sys.stderr)
```
