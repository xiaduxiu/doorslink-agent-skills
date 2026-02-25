# Java 验证循环技能

一个全面的 Java/Spring Boot 项目会话验证系统。

## 何时使用

在以下情况下调用此技能：

* 完成功能或重大代码变更后
* 创建 PR 之前
* 当您希望确保质量门通过时
* 重构之后
* 每次提交前

## 验证阶段

### 阶段 1：构建验证

```bash
# Maven 项目
mvn clean compile -q 2>&1 | tail -30

# Gradle 项目
./gradlew build -x test 2>&1 | tail -30
```

如果构建失败，请停止并在继续之前修复。

### 阶段 2：静态分析

```bash
# Maven 项目 - 运行检查
mvn checkstyle:check 2>&1 | tail -20
mvn spotbugs:check 2>&1 | tail -20

# Gradle 项目
./gradlew checkstyleMain 2>&1 | tail -20
./gradlew spotbugsMain 2>&1 | tail -20
```

报告所有静态分析问题。在继续之前修复关键错误。

### 阶段 3：代码规范检查

```bash
# 检查代码格式（如果配置了 spotless）
mvn spotless:check 2>&1 | tail -20

# 或手动检查
grep -rn "TODO\|FIXME\|XXX" --include="*.java" src/ 2>/dev/null | head -20
```

### 阶段 4：测试套件

```bash
# Maven 项目 - 运行测试
mvn test 2>&1 | tail -50

# Gradle 项目
./gradlew test 2>&1 | tail -50
```

报告：

* 总测试数：X
* 通过：X
* 失败：X
* 覆盖率：X%

### 阶段 5：安全扫描

```bash
# 检查硬编码凭证
grep -rn "password\s*=\s*["\']" --include="*.java" src/ 2>/dev/null | head -10
grep -rn "apiKey\|api_key\|secret" --include="*.java" src/ 2>/dev/null | head -10

# 检查敏感日志
grep -rn "System\.out\.print" --include="*.java" src/ 2>/dev/null | head -10
```

### 阶段 6：依赖检查

```bash
# 检查依赖漏洞
mvn dependency:analyze 2>&1 | tail -30

# 检查过时依赖
mvn versions:display-dependency-updates 2>&1 | tail -20
```

### 阶段 7：差异审查

```bash
# 显示变更统计
git diff --stat
git diff HEAD~1 --name-only
```

审查每个更改的文件，检查：

* 意外更改
* 缺失的错误处理
* 潜在的边界情况
* 日志是否使用 SLF4J

## 输出格式

运行所有阶段后，生成验证报告：

```
VERIFICATION REPORT
==================

Build:       [PASS/FAIL]
Static:      [PASS/FAIL] (X issues)
Format:      [PASS/FAIL]
Tests:       [PASS/FAIL] (X/Y passed, Z% coverage)
Security:    [PASS/FAIL] (X issues)
Dependencies:[PASS/FAIL] (X warnings)
Diff:        [X files changed]

Overall:     [READY/NOT READY] for PR

Issues to Fix:
1. ...
2. ...
```

## 持续模式

对于长时间会话，每 15 分钟或在重大更改后运行验证：

```markdown
设置一个心理检查点：
- 完成每个服务方法后
- 完成一个组件后
- 在移动到下一个任务之前

运行: /verify

```

## 与 TDD 工作流的集成

此技能与 java-tdd-workflow 技能互补：
- TDD 技能：确保每个功能都有测试
- 验证技能：确保整体质量门通过

## 与钩子的集成

此技能补充 PostToolUse 钩子，但提供更深入的验证。
钩子会立即捕获问题；此技能提供全面的审查。
