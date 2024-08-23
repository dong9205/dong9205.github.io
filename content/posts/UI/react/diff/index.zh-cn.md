---
weight: 0
title: "发版内容diff"
subtitle: ""
date: 2024-07-30T10:26:40+08:00
lastmod: 2024-07-30T10:26:40+08:00
draft: false
author: "Derrick"
authorLink: "https://www.p-pp.cn/"
summary: ""
license: ""
tags: [""]
resources:
- name: "featured-image"
  src: ""
categories: 
- "documentation"

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: false
hiddenFromSearch: false

toc:
  enable: true
  auto: true

mapbox:
share:
  enable: true
comment:
  enable: true

lightgallery: true
---

# 发版内容diff

## 前端
> https://github.com/otakustay/react-diff-view.git

### 介绍
该工具是将git diff的结果进行解析，然后对不同的内容进行渲染，从而实现git diff的展示。

## 后端
> https://github.com/go-git/go-git.git

该工具实现了git diff的功能，首先调用go-diff对两份数据进行diff，然后最结果进行解析，解析为git diff一样的效果

## 示例

> 对prometheus的[go.mod](https://raw.githubusercontent.com/prometheus/prometheus/main/go.mod)和alertmanager的[go.mod](https://raw.githubusercontent.com/prometheus/alertmanager/main/go.mod)进行diff, 并在前端展示

### 下载文件
```bash
wget https://raw.githubusercontent.com/prometheus/prometheus/main/go.mod -O prometheus-go.mod
wget https://raw.githubusercontent.com/prometheus/alertmanager/main/go.mod -O alertmanager-go.mod
```
### git diff
```bash
git diff -U1 prometheus-go.mod alertmanager-go.mod
```

### 使用go-git包进行diff
> 代码比较臃肿，直接拿的git-diff的测试代码，没优化
```go
package main

import (
	"bufio"
	"bytes"
	"fmt"
	"os"
	"testing"

	"github.com/go-git/go-git/v5/plumbing"
	"github.com/go-git/go-git/v5/plumbing/filemode"
	gitdiff "github.com/go-git/go-git/v5/plumbing/format/diff"
	"github.com/go-git/go-git/v5/utils/diff"
	"github.com/sergi/go-diff/diffmatchpatch"
)

type testFile struct {
	path string
	mode filemode.FileMode
	seed string
}

func (t testFile) Hash() plumbing.Hash {
	return plumbing.ComputeHash(plumbing.BlobObject, []byte(t.seed))
}

func (t testFile) Mode() filemode.FileMode {
	return t.mode
}

func (t testFile) Path() string {
	return t.path
}

type testPatch struct {
	message     string
	filePatches []testFilePatch
}

func (t testPatch) FilePatches() []gitdiff.FilePatch {
	var result []gitdiff.FilePatch
	for _, f := range t.filePatches {
		result = append(result, f)
	}

	return result
}

func (t testPatch) Message() string {
	return t.message
}

type testFilePatch struct {
	from, to *testFile
	chunks   []testChunk
}

func (t testFilePatch) IsBinary() bool {
	return len(t.chunks) == 0
}

func (t testFilePatch) Files() (gitdiff.File, gitdiff.File) {
	// Go is amazing
	switch {
	case t.from == nil && t.to == nil:
		return nil, nil
	case t.from == nil:
		return nil, t.to
	case t.to == nil:
		return t.from, nil
	}

	return t.from, t.to
}

func (t testFilePatch) Chunks() []gitdiff.Chunk {
	var result []gitdiff.Chunk
	for _, c := range t.chunks {
		result = append(result, c)
	}
	return result
}

type testChunk struct {
	content string
	op      gitdiff.Operation
}

func (t testChunk) Content() string {
	return t.content
}

func (t testChunk) Type() gitdiff.Operation {
	return t.op
}

// readFile reads the contents of a file and returns it as a string.
func readFile(filePath string) (string, error) {
	file, err := os.Open(filePath)
	if err != nil {
		return "", err
	}
	defer file.Close()

	scanner := bufio.NewScanner(file)
	var content string
	for scanner.Scan() {
		content += scanner.Text() + "\n"
	}

	if err := scanner.Err(); err != nil {
		return "", err
	}

	return content, nil
}

func TestCustomDiff(t *testing.T) {
	// if len(os.Args) != 3 {
	// 	t.Logf("Usage: %s <file1> <file2>\n", os.Args[0])
	// 	os.Exit(1)
	// }

	file1Path := "prometheus-go.mod"
	file2Path := "alertmanager-go.mod"
	text1, err := readFile(file1Path)
	if err != nil {
		fmt.Println("Error reading file1:", err)
		os.Exit(1)
	}
	chunks := make([]testChunk, 0)

	text2, err := readFile(file2Path)
	if err != nil {
		fmt.Println("Error reading file2:", err)
		os.Exit(1)
	}
	diffs := diff.Do(text1, text2)
	for _, chunk := range diffs {
		switch chunk.Type {
		case diffmatchpatch.DiffDelete:
			chunks = append(chunks, testChunk{
				content: chunk.Text,
				op:      gitdiff.Delete,
			})
		default:
			chunks = append(chunks, testChunk{
				content: chunk.Text,
				op:      gitdiff.Operation(chunk.Type),
			})
		}

		t.Log("------------------chunk------------------")
		t.Log(chunk.Type)
		t.Log(chunk.Text)
	}
	buffer := bytes.NewBuffer(nil)
	e := gitdiff.NewUnifiedEncoder(buffer, 1).SetColor(gitdiff.NewColorConfig())
	p := testPatch{
		message: "",
		filePatches: []testFilePatch{{
			from: &testFile{
				mode: filemode.Regular,
				path: file1Path,
				seed: text1,
			},
			to: &testFile{
				mode: filemode.Regular,
				path: file2Path,
				seed: text2,
			},
			chunks: chunks,
		}},
	}

	err = e.Encode(p)
	if err != nil {
		t.Fatal(err)
	}
	fmt.Println(buffer.String())
}

```
### 前端
```react
import { parseDiff, Diff, Hunk } from "react-diff-view";
import "react-diff-view/style/index.css";

const diffText1 = `
DIFF的结果
`;

function renderFile({ oldRevision, newRevision, type, hunks }) {
  return (
    <Diff
      key={oldRevision + "-" + newRevision}
      viewType="split"
      diffType={type}
      hunks={hunks}
    >
      {(hunks) => hunks.map((hunk) => <Hunk key={hunk.content} hunk={hunk} />)}
    </Diff>
  );
}

export default function App({}) {
  const files1 = parseDiff(diffText1);
  // console.log(files);
  return (
    <div>
      <div>{files1.map(renderFile)}</div>
    </div>
  );
}

```


