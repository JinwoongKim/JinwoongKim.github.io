---
title: 바이브 코딩에 대한 짧은 견해
categories: go
tags:
  - Go
published: false
---
아주 단순하고 짧은 코드이지만 기록용으로,

go.mod를 프로젝트 루트 디렉토리로 인식하여, 프로젝트 루트 리렉토리를 리턴하는 함수

common.go

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

common_test.go
```go
package common

import (
	"os"
	"path/filepath"
	"testing"
)

func TestFindProjectRoot(t *testing.T) {
	// 현재 디렉토리에서 프로젝트 루트를 찾을 수 있어야 함
	root, err := FindProjectRoot()
	if err != nil {
		t.Errorf("FindProjectRoot() failed: %v", err)
	}

	// go.mod 파일이 실제로 존재하는지 확인
	if _, err := os.Stat(filepath.Join(root, "go.mod")); err != nil {
		t.Errorf("go.mod not found in project root: %v", err)
	}
}

func TestGetProjectRoot(t *testing.T) {
	// GetProjectRoot를 여러 번 호출해도 같은 값을 반환하는지 확인
	root1 := GetProjectRoot()
	root2 := GetProjectRoot()

	if root1 != root2 {
		t.Errorf("GetProjectRoot() returned different values: %s != %s", root1, root2)
	}

	// 반환된 경로가 실제로 존재하는지 확인
	if _, err := os.Stat(root1); err != nil {
		t.Errorf("Project root path does not exist: %v", err)
	}

	// go.mod 파일이 실제로 존재하는지 확인
	if _, err := os.Stat(filepath.Join(root1, "go.mod")); err != nil {
		t.Errorf("go.mod not found in project root: %v", err)
	}
}

func TestFindProjectRootInvalidDir(t *testing.T) {
	// 임시 디렉토리 생성
	tmpDir, err := os.MkdirTemp("", "test")
	if err != nil {
		t.Fatalf("Failed to create temp directory: %v", err)
	}
	defer os.RemoveAll(tmpDir)

	// 현재 디렉토리 저장
	originalDir, err := os.Getwd()
	if err != nil {
		t.Fatalf("Failed to get current directory: %v", err)
	}

	// 임시 디렉토리로 이동
	if err := os.Chdir(tmpDir); err != nil {
		t.Fatalf("Failed to change directory: %v", err)
	}
	defer os.Chdir(originalDir)

	// go.mod가 없는 디렉토리에서 에러를 반환하는지 확인
	_, err = FindProjectRoot()
	if err == nil {
		t.Error("FindProjectRoot() should fail in directory without go.mod")
	}
}

```
