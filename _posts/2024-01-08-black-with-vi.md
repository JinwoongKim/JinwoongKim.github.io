---
title: "[파이썬] vi 편집기 이용시 black 으로 자동 포멧팅 하기"
categories: python
tags:
  - Python
  - black
  - vi
published: true
---

파이썬에는 포멧팅을 해주는 [black](https://github.com/psf/black) 이라는 녀석이 있다.

사용법도 쉽고 성능도 좋아서 종종 쓰는데, 나처럼 vi 를 쓰는 사람은 매번 수행하는게 번거롭다.

찾다보니까 아래 블로그에서 vi 로 편집하다가 나갔을때 자동으로 black 을 적용하는 방법이 있어 정리해본다.

vimrc 파일에 아래와 같이 추가 해주면 된다.

```
augroup python_format
    autocmd!
    autocmd BufWritePost *.py silent !black %
augroup end
```

추가))

`BufWritePost` 는 습관적으로 저장할때마다 포멧팅해서 현재 보고 있는 파일이 업데이트 된다..
그래서 `BufWinLeave` 를 쓰면 파일을 나갈때만 포멧팅돼서 좋다!!

참고
- https://www.meetgor.com/vim-python-black-autoformat/
- https://vimdoc.sourceforge.net/htmldoc/autocmd.html#BufWinLeave