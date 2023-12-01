---
title:  "[파이썬] 기본 로거"
categories: "python"
tag: ["Python", "Logger"]

---

파이썬으로 코딩을 하다 보면 늘 로거를 새로 짰는데, 기본 패키지를 이용해 필수적인 동작을 하는 로거를 블로그에 공유해 놓는다. 이러면 그때그때 복붙하면 되겠지ㅋ

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

아래와 같이 불러 쓰면,

##### app.py
```python
from logger import get_logger

logger = get_logger("app.log")

value=10
st="test"
logger.info("Value : %d, String : %s",value, str(st))
logger.warning("Warning")

```

이렇게 출력 된다.

```
logger: 2023-12-01 17:26:52,735 | INFO | app.py:7 | 72288 >>> Value : 10, String : test
logger: 2023-12-01 17:26:52,736 | WARNING | app.py:8 | 72288 >>> Warning
```