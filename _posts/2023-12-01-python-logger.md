---
title:  "[파이썬] 기본 로거"
categories: "python"
tag: ["Python", "Logger"]

---

파이썬으로 코딩을 하다 보면 늘 로거를 새로 짰는데, 기본 패키지를 이용해 필수적인 동작을 하는 로거를 블로그에 공유해 놓는다. 이러면 그때그때 빼서 쓰면 되겠지ㅋ

기본적으로 파일위치, 시간(ISO-8601), 레벨, 라인넘버 등등 출력이 되고, 자정마다 파일 로테이션해서 30일까지 갖고 있는다.

##### logger.py

```python
from logging.handlers import TimedRotatingFileHandler
import logging
import sys

def get_logger(filename):
    logger = logging.getLogger(__name__)

    fmt = logging.Formatter(
        "%(name)s: %(asctime)s | %(levelname)s | %(filename)s:%(lineno)s | %(process)d >>> %(message)s"
    )
    stdout = logging.StreamHandler(stream=sys.stdout)
    stdout.setFormatter(fmt)
    logger.addHandler(stdout)

    fileHandler = TimedRotatingFileHandler(filename, backupCount=30, when="midnight")
    fileHandler.setFormatter(fmt)
    logger.addHandler(fileHandler)

    logger.setLevel(logging.DEBUG)
    return logger

    # Example Usage
    #logger.debug("A DEBUG message")
    #logger.info("An INFO message")
    #logger.warning("A WARNING message")
    #logger.error("An ERROR message")
    #logger.critical("A CRITICAL message")
```

요런식으로 불러서 쓰면 되고,

##### app.py
```python
from common.logger import get_logger

logger = get_logger("app.log")

logger.info("Value : %d, Time : %s",value, str(ts))
logger.warning("Result is empty list for some reason")
```

이렇게 출력 된다.

```
common.logger: 2023-12-01 16:37:12,336 | INFO | app.py:37 | 12362 >>> Deleyed : None
```