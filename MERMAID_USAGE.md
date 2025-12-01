# Mermaid 使用说明

你的 Jekyll 网站现在已经支持 Mermaid 图表了！

## 使用方法

在 Markdown 文件中，使用代码块格式，语言标识符设置为 `mermaid`：

````markdown
```mermaid
graph TD
    A[Start] --> B[Decision]
    B -->|Yes| C[Action 1]
    B -->|No| D[Action 2]
    C --> E[End]
    D --> E
```
````

## 支持的图表类型

Mermaid 支持多种图表类型，包括：

### 流程图 (Flowchart)
```mermaid
graph LR
    A[开始] --> B[处理]
    B --> C[结束]
```

### 序列图 (Sequence Diagram)
```mermaid
sequenceDiagram
    participant A as 用户
    participant B as 系统
    A->>B: 请求
    B-->>A: 响应
```

### 甘特图 (Gantt Chart)
```mermaid
gantt
    title 项目计划
    dateFormat  YYYY-MM-DD
    section 阶段1
    任务1 :a1, 2024-01-01, 30d
    任务2 :a2, after a1, 20d
```

### 类图 (Class Diagram)
```mermaid
classDiagram
    class Animal {
        +String name
        +int age
        +eat()
    }
    class Dog {
        +bark()
    }
    Animal <|-- Dog
```

### 状态图 (State Diagram)
```mermaid
stateDiagram-v2
    [*] --> 空闲
    空闲 --> 运行中: 开始
    运行中 --> 暂停: 暂停
    暂停 --> 运行中: 继续
    运行中 --> [*]: 结束
```

## 注意事项

- Mermaid 代码块必须使用 `mermaid` 作为语言标识符
- 图表会在页面加载时自动渲染
- 如果图表没有显示，请检查浏览器控制台是否有错误

