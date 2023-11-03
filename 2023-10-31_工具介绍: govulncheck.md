---
tags: Cloud Native Security News,Project Introduction
spec version: v0.2.4
version: v0.1.0
changelog-v0.1.0: init
---
# 1. 工具介绍: govulncheck go官方漏洞扫描工具

项目地址：https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck

## 1.1 What

Govulncheck 扫描并报告影响 Go 代码的已知漏洞。它使用源代码的静态分析或二进制符号表来缩小报告范围，仅提供可能影响应用程序的报告。

默认情况下，govulncheck 向 Go 漏洞数据库 https://vuln.go.dev/ 发出请求。设置 GOVULNDB 环境变量可以指定自定义的数据库，只要实现 https://go.dev/security/vuln/database 中的规范即可。

govulncheck 必须使用 Go 版本 1.18 或更高版本构建。在使用二进制扫描时，只支持读取使用 Go 1.18 及更高版本编译的二进制文件。

## 1.2 Why [Optional]

Why need it, why is it rather than others

## 1.3 How to use

```
$ cd my-module
$ govulncheck ./...
```

Govulncheck 仅报告适用于当前 Go 版本的漏洞。请注意，不同的构建配置可能具有不同的已知漏洞。

要控制哪些文件将被处理，请使用 -tags 标志提供逗号分隔的构建标签列表，并使用 -test 标志表示应包括测试文件。

## 1.4 Try

我试用了一下，效果还不错：

```
~/pentest_target/testproject master                                                     10:56:12
❯ govulncheck ./...
govulncheck is an experimental tool. Share feedback at https://go.dev/s/govulncheck-feedback.

Scanning for dependencies with known vulnerabilities...
Found 8 known vulnerabilities.

Vulnerability ssst0n3/testproject#1: GO-2022-1039
  Programs which compile regular expressions from untrusted
  sources may be vulnerable to memory exhaustion or denial of
  service. The parsed regexp representation is linear in the size
  of the input, but in some cases the constant factor can be as
  high as 40,000, making relatively small regexps consume much
  larger amounts of memory. After fix, each regexp being parsed is
  limited to a 256 MB memory footprint. Regular expressions whose
  representation would use more space than that are rejected.
  Normal use of regular expressions is unaffected.

  Call stacks in your code:
      test/test_data/challenge.go:45:22: testproject, ..., which eventually calls regexp/syntax.Parse

  Found in: regexp/syntax@go1.18
  Fixed in: regexp/syntax@go1.19.2
  More info: https://pkg.go.dev/vuln/GO-2022-1039

Vulnerability ssst0n3/testproject#2: GO-2022-0533
...
```

## 1.5 Source Code

代码量不大，可以简单分析一下该工具的大致流程。

---

支持对源码和二进制扫描。

https://github.com/golang/vuln/blob/ebf31f7dc3ef1da77fcf563ee9becce538d33890/cmd/govulncheck/main.go#L81

```go
func doGovulncheck(patterns []string, sourceAnalysis bool) error {
    ...
    if sourceAnalysis {
        ...
        pkgs, err = loadPackages(patterns, dir)
        ...
        res, err = govulncheck.Source(ctx, cfg, pkgs)
    } else {
        ...
        res, err = gvc.Binary(ctx, cfg, f)
        ...
    }
    ...
}
```

### 源码扫描

源码扫描时，先采集所有的package。

https://github.com/golang/vuln/blob/ebf31f7dc3ef1da77fcf563ee9becce538d33890/cmd/govulncheck/main.go#L191

```go
import (
    ...
    "golang.org/x/tools/go/packages"
    ...
)

func loadPackages(patterns []string, dir string) ([]*vulncheck.Package, error) {
    ...
    pkgs, err := packages.Load(cfg, patterns...)
    ...
    packages.Visit(pkgs, nil, func(p *packages.Package) {
    ...
}
```

---

从采集的package中列出所有的module

https://github.com/golang/vuln/blob/ebf31f7dc3ef1da77fcf563ee9becce538d33890/vulncheck/source.go#L57

```go
func Source(ctx context.Context, pkgs []*Package, cfg *Config) (_ *Result, err error) {
    ...
    mods := extractModules(pkgs)
    modVulns, err := fetchVulnerabilities(ctx, cfg.Client, mods)
    ...
}
```

---

从远端的漏洞数据库查询module相关的漏洞。

https://github.com/golang/vuln/blob/ebf31f7dc3ef1da77fcf563ee9becce538d33890/vulncheck/fetch.go#L68

```go
func fetchVulnerabilities(ctx context.Context, client client.Client, modules []*Module) (moduleVulnerabilities, error) {
    ...
    for _, mod := range modules {
        ...
        vulns, err := client.GetByModule(ctx, modPath)
        ...
    }
    ...
}
```

### 二进制扫描

从二进制文件中解析module，用与源码扫描类似的方法，从远端的漏洞数据库查询module相关的漏洞。

https://github.com/golang/vuln/blob/ebf31f7dc3ef1da77fcf563ee9becce538d33890/vulncheck/binary.go#L40

```go
func Binary(ctx context.Context, exe io.ReaderAt, cfg *Config) (_ *Result, err error) {
    ...
    mods, packageSymbols, bi, err := binscan.ExtractPackagesAndSymbols(exe)
    ...
    cmods := convertModules(mods)
    ...
    modVulns, err := fetchVulnerabilities(ctx, cfg.Client, cmods)
    ...
}
```

---

> 本文使用[云原生安全资讯：项目推荐](https://github.com/cloud-native-security-news/spec/blob/main/project-introduction.md)作为文档基线
