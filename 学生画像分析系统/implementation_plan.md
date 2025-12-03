# 学生画像分析系统 - 实施计划

## 执行摘要

本计划概述了AI驱动的学生画像分析系统的开发，该系统将MBTI性格评估与智能职业规划相结合。系统利用现代Web技术、LLM智能体和心理测评分析，提供个性化的职业指导。

---

## 1. 技术栈详情

### 1.1 前端技术栈
- **框架**：Next.js 14+（App Router）
- **UI组件**：shadcn/ui + Radix UI
- **样式**：Tailwind CSS 3.x
- **图表**：Recharts 或 Chart.js
- **状态管理**：Zustand
- **表单处理**：React Hook Form + Zod验证
- **HTTP客户端**：Axios 或原生fetch

### 1.2 后端技术栈
- **API框架**：FastAPI 0.104+
- **ORM**：SQLAlchemy 2.0
- **数据迁移**：Alembic
- **身份认证**：JWT tokens + httpOnly cookies
- **密码哈希**：bcrypt
- **API文档**：通过FastAPI自动生成（Swagger/OpenAPI）

### 1.3 AI/ML技术栈
- **LLM框架**：LangChain 0.1+
- **LLM提供商**：OpenAI GPT-4 或 Anthropic Claude 3
- **向量存储**：ChromaDB（本地）或Pinecone（生产环境）
- **嵌入模型**：OpenAI text-embedding-3-small
- **可选ML**：scikit-learn（用于MBTI文本分类）

### 1.4 数据库与缓存
- **主数据库**：PostgreSQL 15+
- **扩展**：pgvector（用于语义搜索）
- **缓存**：Redis（用于会话管理和速率限制）

### 1.5 DevOps与部署
- **容器化**：Docker + Docker Compose
- **托管**：Vercel（前端）+ Railway/Render（后端）
- **CI/CD**：GitHub Actions
- **监控**：Sentry（错误跟踪）、LogTail（日志）

---

## 2. 数据库模式（详细版）

```sql
-- 用户表
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(100),
    student_id VARCHAR(50),
    major VARCHAR(100),
    grade_level VARCHAR(20), -- 大一、大二、大三、大四
    university VARCHAR(200),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_users_email ON users(email);

-- MBTI测试结果
CREATE TABLE mbti_assessments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    test_type VARCHAR(50), -- 'standard', 'quick', 'text_analysis'
    personality_type VARCHAR(4), -- INTJ, ENFP等
    dimension_scores JSONB, -- {"E": 30, "I": 70, "S": 40, "N": 60, ...}
    raw_answers JSONB, -- 存储所有问答以供未来分析
    confidence_score FLOAT, -- 如果使用ML，AI的置信度
    test_completed_at TIMESTAMP DEFAULT NOW(),
    test_duration_seconds INT
);
CREATE INDEX idx_mbti_user ON mbti_assessments(user_id);

-- 用户技能与兴趣
CREATE TABLE user_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    interests TEXT[], -- 兴趣数组
    technical_skills TEXT[], -- 编程语言、工具
    soft_skills TEXT[],
    gpa FLOAT,
    certifications JSONB,
    extracurricular JSONB,
    career_goals TEXT,
    preferred_industries TEXT[],
    work_style_preferences JSONB,
    updated_at TIMESTAMP DEFAULT NOW()
);

-- AI职业推荐
CREATE TABLE career_recommendations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    session_id UUID, -- 按会话分组推荐
    recommended_careers JSONB, -- [{"title": "数据科学家", "match_score": 0.85, ...}]
    reasoning TEXT, -- AI解释
    skill_gaps JSONB,
    learning_path JSONB,
    generated_at TIMESTAMP DEFAULT NOW()
);

-- 与AI顾问的聊天会话
CREATE TABLE chat_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(200),
    messages JSONB, -- [{role: 'user', content: '...', timestamp: '...'}]
    summary TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- 职业知识库（用于RAG）
CREATE TABLE career_knowledge (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    career_title VARCHAR(200) UNIQUE,
    description TEXT,
    required_skills TEXT[],
    personality_fit TEXT[], -- 建议的MBTI类型
    education_requirements TEXT,
    avg_salary_range VARCHAR(50),
    growth_outlook VARCHAR(50),
    industry VARCHAR(100),
    embedding VECTOR(1536), -- 用于语义搜索
    metadata JSONB
);
CREATE INDEX ON career_knowledge USING ivfflat (embedding vector_cosine_ops);
```

---

## 3. API设计（RESTful端点）

### 3.1 身份认证
```
POST   /api/auth/register      # 用户注册
POST   /api/auth/login         # 使用邮箱/密码登录
POST   /api/auth/logout        # 使会话失效
GET    /api/auth/me            # 获取当前用户信息
```

### 3.2 MBTI评估
```
GET    /api/mbti/questions     # 获取测试题目
POST   /api/mbti/submit        # 提交答案，获取结果
GET    /api/mbti/history       # 用户的历史测试结果
POST   /api/mbti/analyze-text  # 可选：从文本输入分析MBTI
```

### 3.3 用户画像
```
GET    /api/profile            # 获取用户画像
PUT    /api/profile            # 更新画像（技能、兴趣）
POST   /api/profile/upload-resume  # 解析简历提取技能
```

### 3.4 AI职业顾问
```
POST   /api/career/recommend   # 获取AI职业推荐
GET    /api/career/sessions    # 列出所有会话
GET    /api/career/sessions/:id # 获取特定会话
POST   /api/chat               # 与AI顾问对话
GET    /api/chat/history/:session_id # 获取聊天历史
```

### 3.5 管理（可选）
```
GET    /api/admin/analytics    # 系统使用统计
POST   /api/admin/careers      # 向知识库添加职业
```

---

## 4. AI智能体架构（LangChain）

### 4.1 智能体组件

```python
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.tools import tool

# 定义工具
@tool
def search_careers(query: str, mbti_type: str) -> str:
    """搜索与性格和关键词匹配的职业数据库"""
    # 在career_knowledge表中进行向量搜索
    # 返回前5个匹配项
    pass

@tool
def analyze_skill_gap(user_skills: list, target_career: str) -> str:
    """比较用户技能与职业要求"""
    # 查询career_knowledge，与user_profiles比较
    pass

@tool
def suggest_courses(skill: str) -> str:
    """查找特定技能的学习资源"""
    # 可以集成Coursera/Udemy API或使用网络搜索
    pass

# 智能体提示词
system_prompt = """
你是一位专业的AI职业顾问，专门为大学生服务。

当前学生画像：
- MBTI类型：{mbti_type}
- 专业：{major}
- 年级：{grade}
- 技能：{skills}
- 兴趣：{interests}

你的目标：
1. 根据学生性格提供个性化的职业指导
2. 识别技能差距并建议学习路径
3. 鼓励现实但有抱负的职业探索
4. 始终解释为什么某个职业适合该MBTI类型

请以对话式、鼓励性的语气交流，提供数据驱动的建议。
"""

prompt = ChatPromptTemplate.from_messages([
    ("system", system_prompt),
    MessagesPlaceholder(variable_name="chat_history"),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad")
])

# 初始化智能体
llm = ChatOpenAI(model="gpt-4-turbo-preview", temperature=0.7)
tools = [search_careers, analyze_skill_gap, suggest_courses]
agent = create_openai_tools_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
```

### 4.2 RAG（检索增强生成）

```python
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings

# 初始化向量存储
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma(
    collection_name="career_knowledge",
    embedding_function=embeddings,
    persist_directory="./chroma_db"
)

# 添加职业数据
career_docs = [
    {"title": "数据科学家", "description": "...", "personality_fit": ["INTJ", "INTP"]},
    # ... 更多职业
]
vectorstore.add_documents([Document(page_content=c["description"], metadata=c) for c in career_docs])

# 检索
def get_relevant_careers(query: str, mbti_type: str, k=5):
    enhanced_query = f"{query} 适合{mbti_type}性格类型"
    results = vectorstore.similarity_search(enhanced_query, k=k)
    return results
```

---

## 5. 前端实现

### 5.1 关键页面
1. **着陆页**（`/`）- 营销、功能介绍
2. **仪表盘**（`/dashboard`）- 概览、快速统计
3. **MBTI测试**（`/test/mbti`）- 交互式问卷
4. **结果页面**（`/results`）- 性格分解与图表
5. **职业探索**（`/careers`）- AI推荐
6. **AI聊天**（`/chat`）- 对话式顾问
7. **个人资料**（`/profile`）- 编辑技能、兴趣

### 5.2 示例：MBTI测试组件（React）

```tsx
// app/test/mbti/page.tsx
'use client';
import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { Button } from '@/components/ui/button';
import { Card } from '@/components/ui/card';

const QUESTIONS = [
  { id: 'q1', text: '在社交活动中，你通常：', options: [
    { value: 'E', text: 'A. 感到充满活力，喜欢与人交流' },
    { value: 'I', text: 'B. 感到疲惫，需要独处恢复能量' }
  ]},
  // ... 20-60道题目
];

export default function MBTITest() {
  const [answers, setAnswers] = useState({});
  const [currentQ, setCurrentQ] = useState(0);
  const router = useRouter();

  const handleAnswer = (questionId: string, value: string) => {
    setAnswers({...answers, [questionId]: value});
    if (currentQ < QUESTIONS.length - 1) {
      setCurrentQ(currentQ + 1);
    }
  };

  const submitTest = async () => {
    const res = await fetch('/api/mbti/submit', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ answers })
    });
    const data = await res.json();
    router.push(`/results?type=${data.personality_type}`);
  };

  return (
    <div className="max-w-2xl mx-auto p-6">
      <Card className="p-8">
        <h2 className="text-2xl font-bold mb-4">MBTI 性格测试</h2>
        <p className="text-sm text-gray-500 mb-6">问题 {currentQ + 1} / {QUESTIONS.length}</p>
        
        <div className="space-y-4">
          <p className="text-lg">{QUESTIONS[currentQ].text}</p>
          {QUESTIONS[currentQ].options.map(opt => (
            <Button
              key={opt.value}
              onClick={() => handleAnswer(QUESTIONS[currentQ].id, opt.value)}
              variant={answers[QUESTIONS[currentQ].id] === opt.value ? 'default' : 'outline'}
              className="w-full text-left justify-start"
            >
              {opt.text}
            </Button>
          ))}
        </div>

        {currentQ === QUESTIONS.length - 1 && (
          <Button onClick={submitTest} className="mt-6 w-full">
            查看结果
          </Button>
        )}
      </Card>
    </div>
  );
}
```

---

## 6. 安全与隐私

### 6.1 数据保护
- **加密**：所有流量使用TLS/SSL
- **密码安全**：bcrypt，盐值轮次≥12
- **会话管理**：httpOnly、secure cookies，短TTL
- **速率限制**：基于Redis（每IP每15分钟100次请求）

### 6.2 合规性
- **GDPR/CCPA**：用户数据导出、删除功能
- **匿名化**：存储聚合分析时删除PII
- **同意**：明确选择加入数据在AI训练中的使用

---

## 7. 开发阶段

### 阶段1：MVP（4-6周）
- [ ] 项目设置（单体仓库、Docker）
- [ ] 用户认证系统
- [ ] 基础MBTI测试（20道题）
- [ ] 简单职业匹配（基于规则）
- [ ] 带结果可视化的仪表盘

### 阶段2：AI集成（4周）
- [ ] LangChain智能体实现
- [ ] 使用职业知识库的RAG
- [ ] 聊天界面
- [ ] 技能差距分析

### 阶段3：增强功能（4周）
- [ ] 基于文本的MBTI分析
- [ ] 学习路径推荐
- [ ] 简历解析
- [ ] 管理仪表盘

### 阶段4：生产环境（2周）
- [ ] 性能优化
- [ ] 安全审计
- [ ] 部署
- [ ] 用户测试与迭代

---

## 8. 参考资料与资源

### 研究论文
1. *使用MBTI画像的AI驱动职业路径优化* - [ijsred.com](https://ijsred.com)
2. *基于MBTI的职业选择规则ML模型* - [ResearchGate](https://www.researchgate.net)

### 技术文档
- [LangChain文档](https://python.langchain.com/docs/)
- [FastAPI最佳实践](https://fastapi.tiangolo.com/tutorial/)
- [Next.js App Router](https://nextjs.org/docs/app)
- [pgvector指南](https://github.com/pgvector/pgvector)

### 数据来源
- [O*NET OnLine](https://www.onetonline.org/) - 职业数据库
- [MBTI官方](https://www.myersbriggs.org/) - 评估指南

### 开源项目
- [pypersonality](https://pypi.org/project/pypersonality/) - 从文本预测MBTI
- [MBTI-Test (GitHub)](https://github.com) - Python控制台测试示例

---

## 需要用户审核

> [!IMPORTANT]
> **关键决策点**
> 1. **LLM提供商**：OpenAI GPT-4 vs Anthropic Claude - 影响成本（每次对话约$0.01-0.03）
> 2. **托管**：自托管 vs 托管服务 - 成本与维护之间的权衡
> 3. **MBTI方法**：标准问卷 vs AI文本分析 - 影响准确性与用户体验

> [!WARNING]
> **MBTI科学有效性**
> 虽然MBTI很流行，但一些研究质疑其重测信度。考虑添加免责声明并提供补充评估（例如大五人格）。
