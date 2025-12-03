# 完整代码示例 - 学生画像分析系统

## 1. 后端API（FastAPI）

### 1.1 项目结构
```
backend/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   ├── models/
│   │   ├── user.py
│   │   ├── mbti.py
│   │   └── career.py
│   ├── routers/
│   │   ├── auth.py
│   │   ├── mbti.py
│   │   └── career.py
│   ├── services/
│   │   ├── ai_agent.py
│   │   └── mbti_calculator.py
│   └── utils/
│       └── security.py
├── requirements.txt
└── Dockerfile
```

### 1.2 主应用程序（app/main.py）

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.routers import auth, mbti, career
from app.database import engine, Base

# 创建数据表
Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="学生画像分析API",
    description="AI驱动的职业指导系统",
    version="1.0.0"
)

# CORS配置
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "https://yourdomain.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 包含路由
app.include_router(auth.router, prefix="/api/auth", tags=["身份认证"])
app.include_router(mbti.router, prefix="/api/mbti", tags=["MBTI评估"])
app.include_router(career.router, prefix="/api/career", tags=["职业指导"])

@app.get("/")
def read_root():
    return {"message": "学生画像分析API", "status": "运行中"}

@app.get("/health")
def health_check():
    return {"status": "健康"}
```

### 1.3 数据库配置（app/database.py）

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from app.config import settings

SQLALCHEMY_DATABASE_URL = settings.DATABASE_URL

engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    pool_pre_ping=True,
    echo=True  # 生产环境设为False
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()

# 依赖项
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### 1.4 用户模型（app/models/user.py）

```python
from sqlalchemy import Column, String, DateTime, UUID
from sqlalchemy.sql import func
from app.database import Base
import uuid

class User(Base):
    __tablename__ = "users"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    email = Column(String(255), unique=True, nullable=False, index=True)
    password_hash = Column(String(255), nullable=False)
    full_name = Column(String(100))
    student_id = Column(String(50))
    major = Column(String(100))
    grade_level = Column(String(20))
    university = Column(String(200))
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
```

### 1.5 MBTI路由（app/routers/mbti.py）

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from pydantic import BaseModel
from typing import Dict, List
from app.database import get_db
from app.models.mbti import MBTIAssessment
from app.services.mbti_calculator import MBTICalculator
from app.utils.security import get_current_user

router = APIRouter()

class MBTIAnswers(BaseModel):
    answers: Dict[str, str]  # {"q1": "A", "q2": "B", ...}
    test_type: str = "standard"

class MBTIResult(BaseModel):
    personality_type: str
    dimension_scores: Dict[str, int]
    description: str

# MBTI题目数据
MBTI_QUESTIONS = [
    {
        "id": "q1",
        "text": "在社交场合中，你更倾向于：",
        "dimension": "EI",
        "options": [
            {"value": "E", "text": "主动与他人交流，感到充满活力"},
            {"value": "I", "text": "安静观察，与少数人深入交谈"}
        ]
    },
    {
        "id": "q2",
        "text": "当面对问题时，你更关注：",
        "dimension": "SN",
        "options": [
            {"value": "S", "text": "具体的事实和细节"},
            {"value": "N", "text": "整体的模式和可能性"}
        ]
    },
    {
        "id": "q3",
        "text": "做决定时，你更依赖：",
        "dimension": "TF",
        "options": [
            {"value": "T", "text": "逻辑分析和客观标准"},
            {"value": "F", "text": "个人价值观和对他人的影响"}
        ]
    },
    {
        "id": "q4",
        "text": "在生活方式上，你更喜欢：",
        "dimension": "JP",
        "options": [
            {"value": "J", "text": "有计划、有组织地安排事务"},
            {"value": "P", "text": "保持灵活、随机应变"}
        ]
    },
    # 添加16-60道题以完成完整评估
]

@router.get("/questions")
async def get_mbti_questions():
    """获取MBTI测试题目"""
    return {"questions": MBTI_QUESTIONS, "total": len(MBTI_QUESTIONS)}

@router.post("/submit", response_model=MBTIResult)
async def submit_mbti_test(
    answers: MBTIAnswers,
    db: Session = Depends(get_db),
    current_user = Depends(get_current_user)
):
    """提交MBTI测试答案并获取结果"""
    # 计算MBTI类型
    calculator = MBTICalculator()
    result = calculator.calculate(answers.answers)
    
    # 保存到数据库
    assessment = MBTIAssessment(
        user_id=current_user.id,
        test_type=answers.test_type,
        personality_type=result["type"],
        dimension_scores=result["scores"],
        raw_answers=answers.answers
    )
    db.add(assessment)
    db.commit()
    db.refresh(assessment)
    
    # 获取性格描述
    description = get_personality_description(result["type"])
    
    return {
        "personality_type": result["type"],
        "dimension_scores": result["scores"],
        "description": description
    }

def get_personality_description(mbti_type: str) -> str:
    """获取MBTI类型的性格描述"""
    descriptions = {
        "INTJ": "建筑师型 - 富有想象力和战略性的思考者，凡事都有计划。适合从事需要创新思维和长期规划的工作。",
        "INTP": "逻辑学家型 - 具有创造性的思考者，对知识有着不可遏制的渴望。擅长技术和研究性工作。",
        "ENTJ": "指挥官型 - 大胆、富有想象力且意志强大的领导者。适合管理和战略规划岗位。",
        "ENTP": "辩论家型 - 聪明好奇的思考者，无法抗拒智力上的挑战。适合咨询和创意工作。",
        "INFJ": "提倡者型 - 安静而神秘，同时鼓舞人心且不知疲倦的理想主义者。适合教育和咨询工作。",
        "INFP": "调停者型 - 诗意、善良的利他主义者，总是热情地捍卫自己的理念。适合创意和人文工作。",
        "ENFJ": "主人公型 - 魅力非凡的领导者，有能力让听众着迷。适合管理和公关工作。",
        "ENFP": "竞选者型 - 热情、有创造力且社交能力强的自由派。适合营销和创业工作。",
        "ISTJ": "物流师型 - 实际且注重事实的个人，可靠性不容置疑。适合管理和技术工作。",
        "ISFJ": "守卫者型 - 非常专注且温暖的守护者，时刻准备捍卫所爱之人。适合医疗和教育工作。",
        "ESTJ": "总经理型 - 出色的管理者，在管理事务或人员方面无与伦比。适合管理和行政工作。",
        "ESFJ": "执政官型 - 极有同情心、受欢迎且关心他人的合作者。适合服务和教育行业。",
        "ISTP": "鉴赏家型 - 大胆而务实的实验者，擅长使用各种工具。适合工程和技术工作。",
        "ISFP": "探险家型 - 灵活迷人的艺术家，时刻准备探索和体验新事物。适合艺术和设计工作。",
        "ESTP": "企业家型 - 聪明、精力充沛且善于察言观色的。适合销售和管理工作。",
        "ESFP": "表演者型 - 自发的、精力充沛的且热情的表演者。适合娱乐和服务行业。",
    }
    return descriptions.get(mbti_type, "未知类型")
```

### 1.6 MBTI计算器服务（app/services/mbti_calculator.py）

```python
from typing import Dict

class MBTICalculator:
    """从问卷答案计算MBTI性格类型"""
    
    def __init__(self):
        self.dimensions = ["EI", "SN", "TF", "JP"]
        
    def calculate(self, answers: Dict[str, str]) -> Dict:
        """
        从答案计算MBTI类型
        
        参数：
            answers: 问题ID到答案的映射（例如：{"q1": "E", "q2": "N"}）
            
        返回：
            包含'type'和'scores'的字典
        """
        scores = {
            "E": 0, "I": 0,
            "S": 0, "N": 0,
            "T": 0, "F": 0,
            "J": 0, "P": 0
        }
        
        # 为每个维度计数答案
        for answer in answers.values():
            if answer in scores:
                scores[answer] += 1
        
        # 确定性格类型
        mbti_type = ""
        mbti_type += "E" if scores["E"] >= scores["I"] else "I"
        mbti_type += "S" if scores["S"] >= scores["N"] else "N"
        mbti_type += "T" if scores["T"] >= scores["F"] else "F"
        mbti_type += "J" if scores["J"] >= scores["P"] else "P"
        
        # 计算百分比得分
        total_ei = scores["E"] + scores["I"]
        total_sn = scores["S"] + scores["N"]
        total_tf = scores["T"] + scores["F"]
        total_jp = scores["J"] + scores["P"]
        
        percentage_scores = {
            "E": round(scores["E"] / total_ei * 100) if total_ei > 0 else 0,
            "I": round(scores["I"] / total_ei * 100) if total_ei > 0 else 0,
            "S": round(scores["S"] / total_sn * 100) if total_sn > 0 else 0,
            "N": round(scores["N"] / total_sn * 100) if total_sn > 0 else 0,
            "T": round(scores["T"] / total_tf * 100) if total_tf > 0 else 0,
            "F": round(scores["F"] / total_tf * 100) if total_tf > 0 else 0,
            "J": round(scores["J"] / total_jp * 100) if total_jp > 0 else 0,
            "P": round(scores["P"] / total_jp * 100) if total_jp > 0 else 0,
        }
        
        return {
            "type": mbti_type,
            "scores": percentage_scores,
            "raw_counts": scores
        }
```

### 1.7 AI职业智能体（app/services/ai_agent.py）

```python
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.tools import tool
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings
from typing import List, Dict
import os

class CareerCounselorAgent:
    """使用LangChain的AI驱动职业咨询智能体"""
    
    def __init__(self):
        self.llm = ChatOpenAI(
            model="gpt-4-turbo-preview",
            temperature=0.7,
            api_key=os.getenv("OPENAI_API_KEY")
        )
        self.embeddings = OpenAIEmbeddings()
        self.vectorstore = self._initialize_vectorstore()
        self.agent_executor = self._create_agent()
    
    def _initialize_vectorstore(self):
        """用职业知识初始化向量存储"""
        vectorstore = Chroma(
            collection_name="careers",
            embedding_function=self.embeddings,
            persist_directory="./chroma_db"
        )
        return vectorstore
    
    @tool
    def search_careers(query: str, mbti_type: str) -> str:
        """搜索与查询和MBTI类型匹配的职业"""
        # 实际应该查询vectorstore
        results = f"为{mbti_type}找到的职业：数据科学家、软件工程师、研究分析师"
        return results
    
    @tool
    def analyze_skill_gap(user_skills: str, target_career: str) -> str:
        """分析当前技能与职业要求之间的差距"""
        gap_analysis = f"对于{target_career}，你需要：Python、机器学习、SQL。你已掌握基础Python。"
        return gap_analysis
    
    def _create_agent(self):
        """创建LangChain智能体"""
        system_template = """
        你是一位专业的AI职业规划顾问，专门为大学生提供个性化的职业指导。
        
        学生信息：
        - MBTI类型: {mbti_type}
        - 专业: {major}
        - 年级: {grade}
        - 技能: {skills}
        - 兴趣: {interests}
        
        你的任务：
        1. 基于学生的MBTI性格类型推荐适合的职业方向
        2. 分析学生当前技能与目标职业的差距
        3. 提供具体的学习路径和课程建议
        4. 解释为什么某个职业适合该性格类型
        
        请用友好、鼓励的语气与学生交流，提供实用且有依据的建议。
        """
        
        prompt = ChatPromptTemplate.from_messages([
            ("system", system_template),
            MessagesPlaceholder(variable_name="chat_history", optional=True),
            ("human", "{input}"),
            MessagesPlaceholder(variable_name="agent_scratchpad")
        ])
        
        tools = [self.search_careers, self.analyze_skill_gap]
        agent = create_openai_tools_agent(self.llm, tools, prompt)
        
        return AgentExecutor(
            agent=agent,
            tools=tools,
            verbose=True,
            max_iterations=5
        )
    
    def get_career_recommendation(
        self,
        user_profile: Dict,
        query: str,
        chat_history: List = None
    ) -> str:
        """从AI智能体获取职业推荐"""
        
        response = self.agent_executor.invoke({
            "mbti_type": user_profile.get("mbti_type", ""),
            "major": user_profile.get("major", ""),
            "grade": user_profile.get("grade", ""),
            "skills": ", ".join(user_profile.get("skills", [])),
            "interests": ", ".join(user_profile.get("interests", [])),
            "input": query,
            "chat_history": chat_history or []
        })
        
        return response["output"]

# 使用示例
"""
agent = CareerCounselorAgent()
user_profile = {
    "mbti_type": "INTJ",
    "major": "计算机科学",
    "grade": "大三",
    "skills": ["Python", "Java", "数据结构"],
    "interests": ["人工智能", "数据分析"]
}

recommendation = agent.get_career_recommendation(
    user_profile,
    "我想知道毕业后适合从事什么工作？"
)
"""
```

---

## 2. 前端（Next.js + React）

### 2.1 带可视化的结果页面（app/results/page.tsx）

```tsx
'use client';
import { useEffect, useState } from 'react';
import { useSearchParams } from 'next/navigation';
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card';
import { RadarChart, PolarGrid, PolarAngleAxis, PolarRadiusAxis, Radar, ResponsiveContainer } from 'recharts';

interface MBTIScores {
  E: number;
  I: number;
  S: number;
  N: number;
  T: number;
  F: number;
  J: number;
  P: number;
}

export default function ResultsPage() {
  const searchParams = useSearchParams();
  const [mbtiType, setMbtiType] = useState('');
  const [scores, setScores] = useState<MBTIScores | null>(null);
  const [description, setDescription] = useState('');

  useEffect(() => {
    // 从API获取结果
    const type = searchParams.get('type');
    if (type) {
      fetchResults(type);
    }
  }, [searchParams]);

  const fetchResults = async (type: string) => {
    const res = await fetch(`/api/mbti/history?latest=true`, {
      credentials: 'include'
    });
    const data = await res.json();
    
    setMbtiType(data.personality_type);
    setScores(data.dimension_scores);
    setDescription(data.description);
  };

  const chartData = scores ? [
    { dimension: '外向 vs 内向', value: scores.E, fullMark: 100 },
    { dimension: '实感 vs 直觉', value: scores.S, fullMark: 100 },
    { dimension: '思考 vs 情感', value: scores.T, fullMark: 100 },
    { dimension: '判断 vs 知觉', value: scores.J, fullMark: 100 },
  ] : [];

  return (
    <div className="container mx-auto p-6 max-w-4xl">
      <h1 className="text-4xl font-bold mb-2 text-center">你的性格类型</h1>
      <div className="text-6xl font-black text-center mb-8 bg-gradient-to-r from-purple-600 to-blue-600 bg-clip-text text-transparent">
        {mbtiType}
      </div>

      <Card className="mb-6">
        <CardHeader>
          <CardTitle>性格描述</CardTitle>
        </CardHeader>
        <CardContent>
          <p className="text-lg">{description}</p>
        </CardContent>
      </Card>

      <Card className="mb-6">
        <CardHeader>
          <CardTitle>维度分析</CardTitle>
        </CardHeader>
        <CardContent>
          <ResponsiveContainer width="100%" height={400}>
            <RadarChart data={chartData}>
              <PolarGrid />
              <PolarAngleAxis dataKey="dimension" />
              <PolarRadiusAxis angle={90} domain={[0, 100]} />
              <Radar name="你的得分" dataKey="value" stroke="#8884d8" fill="#8884d8" fillOpacity={0.6} />
            </RadarChart>
          </ResponsiveContainer>
          
          <div className="mt-6 space-y-4">
            {scores && (
              <>
                <DimensionBar label="外向(E)" value={scores.E} inverse="内向(I)" inverseValue={scores.I} />
                <DimensionBar label="实感(S)" value={scores.S} inverse="直觉(N)" inverseValue={scores.N} />
                <DimensionBar label="思考(T)" value={scores.T} inverse="情感(F)" inverseValue={scores.F} />
                <DimensionBar label="判断(J)" value={scores.J} inverse="知觉(P)" inverseValue={scores.P} />
              </>
            )}
          </div>
        </CardContent>
      </Card>

      <div className="flex gap-4 justify-center">
        <Button onClick={() => window.location.href = '/careers'} size="lg">
          探索职业方向
        </Button>
        <Button onClick={() => window.location.href = '/chat'} variant="outline" size="lg">
          与AI顾问对话
        </Button>
      </div>
    </div>
  );
}

function DimensionBar({ label, value, inverse, inverseValue }: any) {
  return (
    <div className="flex items-center gap-4">
      <span className="w-20 text-sm font-medium text-right">{label}</span>
      <div className="flex-1 h-8 bg-gray-200 rounded-full overflow-hidden flex">
        <div 
          className="bg-blue-500 flex items-center justify-end pr-2 text-white text-xs font-bold"
          style={{ width: `${value}%` }}
        >
          {value}%
        </div>
        <div 
          className="bg-orange-500 flex items-center justify-start pl-2 text-white text-xs font-bold"
          style={{ width: `${inverseValue}%` }}
        >
          {inverseValue}%
        </div>
      </div>
      <span className="w-20 text-sm font-medium">{inverse}</span>
    </div>
  );
}
```

### 2.2 AI聊天界面（app/chat/page.tsx）

```tsx
'use client';
import { useState } from 'react';
import { Card } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Avatar } from '@/components/ui/avatar';

interface Message {
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
}

export default function ChatPage() {
  const [messages, setMessages] = useState<Message[]>([
    {
      role: 'assistant',
      content: '你好！我是你的AI职业规划顾问。我已经了解了你的MBTI性格类型和背景信息。有什么我可以帮助你的吗？',
      timestamp: new Date()
    }
  ]);
  const [input, setInput] = useState('');
  const [loading, setLoading] = useState(false);

  const sendMessage = async () => {
    if (!input.trim()) return;

    const userMessage: Message = {
      role: 'user',
      content: input,
      timestamp: new Date()
    };
    
    setMessages([...messages, userMessage]);
    setInput('');
    setLoading(true);

    try {
      const res = await fetch('/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        credentials: 'include',
        body: JSON.stringify({
          message: input,
          chat_history: messages
        })
      });

      const data = await res.json();
      
      const assistantMessage: Message = {
        role: 'assistant',
        content: data.response,
        timestamp: new Date()
      };
      
      setMessages(prev => [...prev, assistantMessage]);
    } catch (error) {
      console.error('发送消息失败:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="container mx-auto p-6 max-w-4xl h-screen flex flex-col">
      <h1 className="text-3xl font-bold mb-4">AI职业顾问</h1>
      
      <Card className="flex-1 flex flex-col overflow-hidden">
        <div className="flex-1 overflow-y-auto p-6 space-y-4">
          {messages.map((msg, idx) => (
            <div key={idx} className={`flex gap-3 ${msg.role === 'user' ? 'flex-row-reverse' : ''}`}>
              <Avatar className="w-10 h-10">
                {msg.role === 'assistant' ? '🤖' : '👤'}
              </Avatar>
              <div className={`max-w-[70%] rounded-lg p-4 ${
                msg.role === 'user' 
                  ? 'bg-blue-500 text-white' 
                  : 'bg-gray-100 text-gray-900'
              }`}>
                <p className="whitespace-pre-wrap">{msg.content}</p>
                <span className="text-xs opacity-70 mt-2 block">
                  {msg.timestamp.toLocaleTimeString()}
                </span>
              </div>
            </div>
          ))}
          {loading && (
            <div className="flex gap-3">
              <Avatar className="w-10 h-10">🤖</Avatar>
              <div className="bg-gray-100 rounded-lg p-4">
                <div className="flex gap-1">
                  <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce"></div>
                  <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce delay-100"></div>
                  <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce delay-200"></div>
                </div>
              </div>
            </div>
          )}
        </div>

        <div className="border-t p-4">
          <div className="flex gap-2">
            <Input
              value={input}
              onChange={(e) => setInput(e.target.value)}
              onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
              placeholder="输入你的问题..."
              className="flex-1"
              disabled={loading}
            />
            <Button onClick={sendMessage} disabled={loading}>
              发送
            </Button>
          </div>
        </div>
      </Card>
    </div>
  );
}
```

---

## 3. Docker部署

### 3.1 docker-compose.yml

```yaml
version: '3.8'

services:
  postgres:
    image: pgvector/pgvector:pg15
    environment:
      POSTGRES_DB: student_profile_db
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: your_secure_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://admin:your_secure_password@postgres:5432/student_profile_db
      REDIS_URL: redis://redis:6379
      OPENAI_API_KEY: ${OPENAI_API_KEY}
    depends_on:
      - postgres
      - redis
    volumes:
      - ./backend:/app

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:8000
    depends_on:
      - backend

volumes:
  postgres_data:
```

### 3.2 后端Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用程序
COPY . .

# 运行迁移并启动服务器
CMD alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

### 3.3 Requirements.txt

```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
psycopg2-binary==2.9.9
pydantic==2.5.0
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.6
langchain==0.1.0
langchain-openai==0.0.2
langchain-community==0.0.10
chromadb==0.4.18
openai==1.6.1
redis==5.0.1
alembic==1.13.0
```

---

## 4. 快速启动命令

```bash
# 克隆并设置
git clone <your-repo>
cd student-profile-system

# 启动所有服务
docker-compose up -d

# 查看日志
docker-compose logs -f backend

# 运行迁移
docker-compose exec backend alembic upgrade head

# 停止服务
docker-compose down
```

访问：
- 前端：http://localhost:3000
- 后端API：http://localhost:8000
- API文档：http://localhost:8000/docs
