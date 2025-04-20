---
title: 바이브 코딩에 대한 짧은 견해
categories: go
tags:
  - Go
published: false
---
아주 단순하고 짧은 코드이지만 기록용으로,

go.mod를 프로젝트 루트 디렉토리로 인식하여, 프로젝트 루트 리렉토리를 리턴하는 함수

```go
package common

import (
	"fmt"
	"os"
	"path/filepath"
	"sync"
)

var (
	projectRoot string
	once        sync.Once
)

func FindProjectRoot() (string, error) {
	dir, err := os.Getwd() // 현재 작업 디렉토리
	if err != nil {
		return "", err
	}

	for {
		// go.mod 파일이 있는지 확인, 있으면 프로젝트 루트 반환
		if _, err := os.Stat(filepath.Join(dir, "go.mod")); err == nil {
			return dir, nil
		}

		parent := filepath.Dir(dir)
		if parent == dir { // 루트 디렉토리까지 도달
			return "", fmt.Errorf("go.mod not found")
		}
		dir = parent
	}
}

func GetProjectRoot() string {
	once.Do(func() {
		var err error
		projectRoot, err = FindProjectRoot()
		if err != nil {
			panic(err)
		}
	})
	return projectRoot
}

```
