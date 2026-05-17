# 1
first
工人工厂智能匹配系统 - 简易多智能体实现 (PoC)
核心功能：工厂岗位发布 -> 工人匹配 + 精排打分 -> 可选议价协商 -> Token 消耗统计
依赖：Python 3.9+ (无额外外部API，使用模拟 LLM 文本)
"""

from typing import List, Dict, Any, Optional, Tuple
from pydantic import BaseModel, Field
from enum import Enum
import json
import random
from datetime import datetime

# ========== 数据模型定义 ==========
class Worker(BaseModel):
    id: str
    name: str
    skills: List[str]  # 技能列表，如 ["焊接", "数控机床"]
    available_hours: str  # 例如 "07:00-19:00"
    commute_radius_km: int
    is_available: bool = True
    preferred_wage: float = 200.0  # 期望日薪 (元)

class FactoryJob(BaseModel):
    id: str
    title: str
    required_skills: List[str]
    shift: str  # "白班", "夜班"
    urgency: int = 5  # 1-10, 越高越紧急
    offered_wage: float = 220.0
    description: str = ""

class MatchResult(BaseModel):
    job_id: str
    worker_id: str
    score: int  # 0-100
    reasoning: str
    status: str  # "pending", "negotiating", "confirmed", "rejected"

# ========== 工具函数（模拟 Agent 使用的工具集）==========
def score_candidate(job: FactoryJob, worker: Worker) -> Tuple[int, str]:
    """模拟 LLM 精排打分工具 (代替一次 LLM 调用)"""
    # 简化逻辑：技能匹配数量 + 时间半径匹配度 + 薪资差距
    skill_match = len(set(job.required_skills) & set(worker.skills))
    max_skill = max(1, len(job.required_skills))
    skill_score = (skill_match / max_skill) * 60
    
    # 通勤半径匹配（假设工厂在园区中心，工人半径覆盖）
    commute_ok = 1 if worker.commute_radius_km >= 10 else 0.5
    commute_score = commute_ok * 20
    
    wage_gap = job.offered_wage - worker.preferred_wage
    if wage_gap >= 0:
        wage_score = 20
    elif wage_gap > -30:
        wage_score = 15
    else:
        wage_score = 5
    
    total = int(skill_score + commute_score + wage_score)
    reason = f"技能匹配{skill_match}/{len(job.required_skills)}，通勤半径{worker.commute_radius_km}km，薪资差额{wage_gap:.0f}元"
    return min(total, 100), reason

def negotiate_term(job: FactoryJob, worker: Worker, round_num: int) -> Tuple[Optional[float], str]:
    """模拟协商工具：返回新的offer或拒绝"""
    if round_num == 1:
        # 工厂首次加价 5%
        new_offer = job.offered_wage * 1.05
        msg = f"工厂提议提高至 {new_offer:.1f} 元"
        return new_offer, msg
    elif round_num == 2:
        if worker.preferred_wage <= job.offered_wage * 1.1:
            msg = "工人接受报价"
            return job.offered_wage, msg
        else:
            msg = "工人放弃协商"
            return None, msg
    else:
        return None, "协商超时失败"

# ========== Agent 定义 ==========
class WorkerAgent:
    def __init__(self, worker: Worker):
        self.worker = worker
    def update_availability(self, available: bool):
        self.worker.is_available = available
        return f"Worker {self.worker.id} 可用状态更新为 {available}"
    def accept_offer(self, wage: float) -> str:
        self.worker.is_available = False
        return f"工人 {self.worker.name} 接受 {wage} 元报价，锁定岗位"
    def reject_offer(self, reason: str) -> str:
        return f"拒绝报价: {reason}"

class FactoryAgent:
    def __init__(self, job: FactoryJob):
        self.job = job
    def get_candidate_scores(self, workers: List[Worker]) -> List[MatchResult]:
        results = []
        for w in workers:
            score, reasoning = score_candidate(self.job, w)
            results.append(MatchResult(
                job_id=self.job.id, worker_id=w.id,
                score=score, reasoning=reasoning, status="pending"
            ))
        results.sort(key=lambda x: x.score, reverse=True)
        return results[:5]  # 返回 Top5

class OrchestratorAgent:
    def __init__(self, all_workers: List[Worker]):
        self.workers = {w.id: w for w in all_workers}
        self.token_usage = {"input": 0, "output": 0}  # 模拟 token 统计
    def log_tokens(self, input_len: int, output_len: int):
        self.token_usage["input"] += input_len
        self.token_usage["output"] += output_len
    def match_job(self, factory_agent: FactoryAgent) -> List[MatchResult]:
        # 粗筛 (实际应用需向量检索，这里用简单过滤)
        available_workers = [w for w in self.workers.values() if w.is_available]
        # 模拟精排阶段 token 消耗: 每个候选对 ~1100 input + 200 output
        self.log_tokens(len(available_workers) * 1100, len(available_workers) * 200)
        top_matches = factory_agent.get_candidate_scores(available_workers)
        # 模拟报告生成 token
        self.log_tokens(300, 150)
        return top_matches
    def run_negotiation(self, job: FactoryJob, worker: Worker) -> Tuple[Optional[float], str]:
        # 模拟 2 轮协商，每轮约 1800 input + 750 output
        self.log_tokens(1800, 750)
        offer = job.offered_wage
        final_msg = ""
        for round_num in range(1, 3):
            new_offer, msg = negotiate_term(job, worker, round_num)
            final_msg = msg
            if new_offer is not None:
                offer = new_offer
            else:
                return None, final_msg
        return offer, final_msg

# ========== 主系统模拟 ==========
class MatchingSystem:
    def __init__(self, workers: List[Worker]):
        self.orchestrator = OrchestratorAgent(workers)
        self.workers = workers
        self.matches_history: List[MatchResult] = []
    def process_job(self, job: FactoryJob):
        print(f"\n处理岗位: {job.title} (ID: {job.id})")
        factory = FactoryAgent(job)
        # 1. 匹配精排
        top_matches = self.orchestrator.match_job(factory)
        if not top_matches:
            print("无可用工人")
            return
        best = top_matches[0]
        worker = next(w for w in self.workers if w.id == best.worker_id)
        print(f"最佳匹配: {worker.name} (得分 {best.score}) - {best.reasoning}")
        # 2. 协商（模拟仅对得分最高的工人进行，且得分>70才触发）
        if best.score > 70 and job.offered_wage < worker.preferred_wage:
            print("启动协商...")
            final_offer, result_msg = self.orchestrator.run_negotiation(job, worker)
            if final_offer:
                # 调用 WorkerAgent 接受报价
                wa = WorkerAgent(worker)
                confirm_msg = wa.accept_offer(final_offer)
                best.status = "confirmed"
                print(confirm_msg)
            else:
                print(f"协商失败: {result_msg}")
                best.status = "rejected"
        else:
            # 直接接受
            wa = WorkerAgent(worker)
            wa.accept_offer(job.offered_wage)
            best.status = "confirmed"
            print(f"直接匹配成功，报价 {job.offered_wage} 元")
        self.matches_history.append(best)
    def report_token_usage(self):
        total = self.orchestrator.token_usage["input"] + self.orchestrator.token_usage["output"]
        print("\n===== Token 统计 (模拟) =====")
        print(f"输入 tokens: {self.orchestrator.token_usage['input']:,}")
        print(f"输出 tokens: {self.orchestrator.token_usage['output']:,}")
        print(f"总 tokens: {total:,}")
        print("(注: 此为基于上述 Agent 调用的模拟值，实际 LLM 用量需按生产环境调整)")

# ========== 示例运行 ==========
if __name__ == "__main__":
    # 构建模拟工人库
    workers_data = [
        Worker(id="W001", name="张三", skills=["焊接", "钳工"], available_hours="07:00-19:00", commute_radius_km=12, preferred_wage=210),
        Worker(id="W002", name="李四", skills=["数控机床", "编程"], available_hours="19:00-07:00", commute_radius_km=8, preferred_wage=250),
        Worker(id="W003", name="王五", skills=["焊接", "质检"], available_hours="07:00-19:00", commute_radius_km=15, preferred_wage=200),
    ]
    # 构建岗位
    job = FactoryJob(
        id="J101", title="急聘高级焊工", required_skills=["焊接"], shift="白班",
        urgency=9, offered_wage=220, description="需精通二保焊"
    )
    # 运行系统
    system = MatchingSystem(workers_data)
    system.process_job(job)
    system.report_token_usage()
