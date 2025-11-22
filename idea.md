# MAGE 개선 아이디어

## 1. 멀티-모델 앙상블 전략

### 1.1 모델별 강점 활용
- **Claude API 강점 활용**
  - 긴 컨텍스트 처리 능력 활용하여 복잡한 RTL 스펙 이해
  - 상세한 추론 과정 생성 (reasoning steps)을 통한 디버깅 용이성
  - RTLEditor 단계에서 Claude의 뛰어난 코드 수정 능력 활용

- **Gemini API 강점 활용**
  - Gemini 2.0의 빠른 응답 속도로 후보 생성 단계 가속화
  - 대량의 RTL 후보 생성 시 비용 효율성 (Flash 모델 사용)
  - 멀티모달 입력 지원 - 파형 이미지나 블록 다이어그램 입력 가능

### 1.2 하이브리드 아키텍처 제안
```python
class HybridTopAgent:
    def __init__(self):
        self.claude_llm = Anthropic(model="claude-3-5-sonnet-20241022")
        self.gemini_llm = Vertex(model="gemini-2.0-flash-001")
        self.gemini_pro_llm = Vertex(model="gemini-2.0-pro-001")

    def run_instance_hybrid(self, spec: str):
        # 1. Claude로 초기 RTL 생성 (높은 품질)
        rtl_code = self.rtl_gen_claude.chat(spec)

        # 2. Gemini Flash로 대량 후보 생성 (빠른 속도, 저비용)
        candidates = self.rtl_gen_gemini_flash.gen_candidates(
            spec, candidates_num=50  # 기존 20개에서 증가
        )

        # 3. Gemini Pro로 후보 평가 및 선택
        best_candidates = self.evaluate_candidates_gemini_pro(candidates)

        # 4. Claude로 최종 편집 및 수정 (높은 정확도)
        final_rtl = self.rtl_edit_claude.chat(best_candidates[0])
```

### 1.3 앙상블 투표 메커니즘
- 3개 모델(Claude, Gemini Pro, GPT-4o)이 각각 RTL 생성
- SimReviewer로 각 결과 평가 후 가장 적은 mismatch 선택
- 또는 3개 결과를 LLM에게 보여주고 최적 조합 선택 요청

## 2. SystemVerilog 코드 품질 개선

### 2.1 향상된 프롬프트 엔지니어링

#### 2.1.1 업계 표준 코딩 가이드라인 추가
```python
INDUSTRY_CODING_STANDARDS = r"""
SystemVerilog 코딩 표준 (업계 Best Practices):

1. 네이밍 컨벤션:
   - 클록: clk, clk_i
   - 리셋: rst_n (active-low), rst (active-high)
   - 입력 포트: _i 또는 _in 접미사
   - 출력 포트: _o 또는 _out 접미사
   - 레지스터: _r, _reg, _q 접미사
   - 와이어: _w, _wire, _d 접미사

2. FSM 구현:
   - 3-always block 스타일 사용 (state register, next state logic, output logic)
   - One-hot encoding vs Binary encoding 명시
   - 모든 state에 대한 default case 처리

3. 클록 도메인 크로싱 (CDC):
   - 비동기 신호는 2-FF synchronizer 사용
   - Gray code를 사용한 FIFO 포인터 처리

4. 린팅 룰 준수:
   - 모든 combinational always block에 default 값 명시
   - Latch 생성 방지
   - Full case, Parallel case 주의

5. 파라미터화:
   - Magic number 사용 금지
   - 모든 비트폭을 파라미터로 정의
"""
```

#### 2.1.2 검증 관점의 프롬프트 추가
```python
VERIFICATION_AWARE_PROMPT = r"""
검증 가능한 RTL 작성 가이드:

1. Assertion 추가:
   - 중요한 불변 조건에 대해 SVA (SystemVerilog Assertion) 추가
   - 예: assert property (@(posedge clk) disable iff (rst) (valid |-> ##1 ready));

2. Coverage Point 고려:
   - 주요 상태 전이에 대한 커버리지 고려
   - Corner case 처리 명시

3. 디버깅 용이성:
   - 의미있는 신호명 사용
   - 복잡한 로직에 주석 추가

4. 합성 가능성:
   - 조합 루프 방지
   - 초기화 블록은 합성 불가 명시
"""
```

### 2.2 정적 분석 통합

#### 2.2.1 Lint 검사 추가
```python
def check_lint_rules(rtl_path: str) -> Tuple[bool, List[str]]:
    """
    Verilator lint 또는 상용 lint 도구 실행
    """
    # Verilator lint
    cmd = f"verilator --lint-only -Wall {rtl_path}"
    is_pass, lint_output = run_bash_command(cmd, timeout=30)

    warnings = parse_lint_warnings(lint_output)
    critical_warnings = filter_critical_warnings(warnings)

    return len(critical_warnings) == 0, critical_warnings
```

#### 2.2.2 코드 메트릭 분석
```python
class RTLQualityMetrics:
    def analyze(self, rtl_code: str) -> Dict[str, Any]:
        return {
            "lines_of_code": self.count_loc(rtl_code),
            "cyclomatic_complexity": self.calc_complexity(rtl_code),
            "register_count": self.count_registers(rtl_code),
            "combinational_depth": self.estimate_comb_depth(rtl_code),
            "parameterization_score": self.check_parameterization(rtl_code),
        }
```

### 2.3 코드 리뷰 에이전트 추가
```python
class RTLCodeReviewer:
    """
    LLM 기반 코드 리뷰어 - 생성된 RTL의 품질 평가
    """
    def review(self, rtl_code: str, spec: str) -> Dict[str, Any]:
        review_prompt = f"""
        다음 RTL 코드를 검토하고 개선점을 제안하세요:

        평가 기준:
        1. 스펙 준수도 (0-10점)
        2. 코딩 스타일 (0-10점)
        3. 합성 가능성 (0-10점)
        4. 유지보수성 (0-10점)
        5. 성능 (클록 주파수 예상) (0-10점)

        <spec>{spec}</spec>
        <rtl_code>{rtl_code}</rtl_code>
        """
        response = self.llm.chat(review_prompt)
        return self.parse_review(response)
```

## 3. 상용 시뮬레이터 지원

### 3.1 Simulator Backend 추상화
```python
from enum import Enum
from abc import ABC, abstractmethod

class SimulatorType(Enum):
    IVERILOG = "iverilog"
    VCS = "vcs"
    XCELIUM = "xcelium"  # xrun
    QUESTA = "questa"
    VERILATOR = "verilator"

class SimulatorBackend(ABC):
    @abstractmethod
    def compile(self, rtl_files: List[str], tb_files: List[str],
                output_dir: str) -> Tuple[bool, str]:
        """RTL과 TB를 컴파일"""
        pass

    @abstractmethod
    def simulate(self, top_module: str, output_dir: str) -> Tuple[bool, str]:
        """시뮬레이션 실행"""
        pass

    @abstractmethod
    def parse_results(self, sim_output: str) -> Tuple[bool, int, str]:
        """시뮬레이션 결과 파싱"""
        pass
```

### 3.2 Xcelium (xrun) Backend 구현
```python
class XceliumBackend(SimulatorBackend):
    def __init__(self, xrun_path: str = "xrun"):
        self.xrun_path = xrun_path
        self.compile_options = [
            "-64bit",
            "-sv",
            "-access +rwc",  # 디버그 접근
            "-linedebug",
            "-timescale 1ns/1ps",
        ]
        self.elab_options = [
            "-elaborate",
            "-messages",
        ]
        self.sim_options = [
            "-R",  # Run immediately
            "-xmlibdirname ./xcelium.d",
        ]

    def compile(self, rtl_files: List[str], tb_files: List[str],
                output_dir: str) -> Tuple[bool, str]:
        all_files = tb_files + rtl_files
        cmd = f"{self.xrun_path} {' '.join(self.compile_options)} "
        cmd += f"{' '.join(self.elab_options)} "
        cmd += f"-top tb {' '.join(all_files)} "
        cmd += f"-work {output_dir}/work"

        is_pass, output = run_bash_command(cmd, timeout=120)
        return is_pass, output

    def simulate(self, top_module: str, output_dir: str) -> Tuple[bool, str]:
        # Xcelium은 compile과 simulate가 통합 가능
        return True, ""

    def parse_results(self, sim_output: str) -> Tuple[bool, int, str]:
        # xrun 출력 파싱
        is_pass = "SIMULATION PASSED" in sim_output
        mismatch_cnt = 0
        if not is_pass:
            # mismatch 카운트 추출 로직
            match = re.search(r"(\d+) MISMATCHES", sim_output)
            if match:
                mismatch_cnt = int(match.group(1))
        return is_pass, mismatch_cnt, sim_output
```

### 3.3 VCS Backend 구현
```python
class VCSBackend(SimulatorBackend):
    def __init__(self, vcs_path: str = "vcs"):
        self.vcs_path = vcs_path
        self.compile_options = [
            "-sverilog",
            "-full64",
            "-debug_access+all",
            "-timescale=1ns/1ps",
            "-kdb",  # Knowledge Database for Verdi
        ]

    def compile(self, rtl_files: List[str], tb_files: List[str],
                output_dir: str) -> Tuple[bool, str]:
        all_files = tb_files + rtl_files
        simv_path = f"{output_dir}/simv"
        cmd = f"{self.vcs_path} {' '.join(self.compile_options)} "
        cmd += f"-o {simv_path} {' '.join(all_files)}"

        is_pass, output = run_bash_command(cmd, timeout=120)
        return is_pass, output

    def simulate(self, top_module: str, output_dir: str) -> Tuple[bool, str]:
        simv_path = f"{output_dir}/simv"
        cmd = f"{simv_path} +ntb_random_seed_automatic"
        is_pass, output = run_bash_command(cmd, timeout=120)
        return is_pass, output
```

### 3.4 Simulator Manager
```python
class SimulatorManager:
    def __init__(self, sim_type: SimulatorType):
        self.backend = self._create_backend(sim_type)

    def _create_backend(self, sim_type: SimulatorType) -> SimulatorBackend:
        if sim_type == SimulatorType.IVERILOG:
            return IVerilogBackend()
        elif sim_type == SimulatorType.XCELIUM:
            return XceliumBackend()
        elif sim_type == SimulatorType.VCS:
            return VCSBackend()
        elif sim_type == SimulatorType.VERILATOR:
            return VerilatorBackend()
        else:
            raise ValueError(f"Unsupported simulator: {sim_type}")

    def run_simulation(self, rtl_files: List[str], tb_files: List[str],
                      output_dir: str) -> Tuple[bool, int, str]:
        # 컴파일
        compile_ok, compile_log = self.backend.compile(
            rtl_files, tb_files, output_dir
        )
        if not compile_ok:
            return False, 0, compile_log

        # 시뮬레이션
        sim_ok, sim_log = self.backend.simulate("tb", output_dir)

        # 결과 파싱
        return self.backend.parse_results(sim_log)
```

### 3.5 설정 파일 통합
```python
# gen_config.py에 추가
class SimulationConfig(BaseModel):
    simulator: SimulatorType = SimulatorType.IVERILOG
    simulator_path: Optional[str] = None  # xrun, vcs 등의 경로
    compile_options: List[str] = []
    simulation_options: List[str] = []
    use_gui: bool = False  # 디버깅 시 GUI 사용
    generate_waveform: bool = True  # VCD/FSDB 생성

# test_top_agent.py에 추가
args_dict = {
    # 기존 옵션들...
    "simulator": "xcelium",  # "iverilog", "vcs", "xcelium", "verilator"
    "simulator_path": "/tools/cadence/xcelium/bin/xrun",
    "generate_coverage": True,  # 커버리지 수집
}
```

## 4. 고급 검증 기능

### 4.1 커버리지 기반 피드백
```python
class CoverageGuidedGenerator:
    def __init__(self, llm, simulator: SimulatorManager):
        self.llm = llm
        self.simulator = simulator

    def improve_with_coverage(self, rtl_code: str, spec: str,
                             tb_code: str) -> str:
        # 1. 초기 시뮬레이션 + 커버리지 수집
        coverage_report = self.simulator.get_coverage_report()

        # 2. 낮은 커버리지 부분 식별
        uncovered_cases = self.analyze_coverage_gaps(coverage_report)

        # 3. LLM에게 미달 커버리지 정보 제공하여 RTL 개선
        improvement_prompt = f"""
        다음 RTL 코드의 커버리지가 부족합니다:

        커버되지 않은 케이스:
        {uncovered_cases}

        RTL 코드를 수정하여 모든 케이스를 커버하세요:
        <rtl_code>{rtl_code}</rtl_code>
        """

        improved_rtl = self.llm.chat(improvement_prompt)
        return improved_rtl
```

### 4.2 형식 검증 (Formal Verification) 통합
```python
class FormalVerificationAgent:
    def generate_assertions(self, spec: str, rtl_code: str) -> List[str]:
        """
        스펙으로부터 SVA assertion 자동 생성
        """
        prompt = f"""
        다음 스펙에 대한 SystemVerilog Assertion을 생성하세요:
        <spec>{spec}</spec>
        <rtl_code>{rtl_code}</rtl_code>

        생성할 assertion 종류:
        1. 인터페이스 프로토콜 체크
        2. 데이터 무결성 체크
        3. FSM 상태 전이 검증
        4. 타이밍 제약 조건
        """
        assertions = self.llm.chat(prompt)
        return self.parse_assertions(assertions)

    def run_formal_check(self, rtl_with_assertions: str) -> bool:
        """
        JasperGold 또는 Questa Formal 실행
        """
        # 형식 검증 도구 실행
        pass
```

## 5. 파이프라인 최적화

### 5.1 병렬 후보 평가
```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class ParallelCandidateEvaluator:
    def __init__(self, simulator: SimulatorManager, max_workers: int = 8):
        self.simulator = simulator
        self.executor = ThreadPoolExecutor(max_workers=max_workers)

    async def evaluate_candidates_parallel(self,
                                          candidates: List[str],
                                          tb_path: str,
                                          output_base_dir: str) -> List[Tuple[str, int]]:
        """
        여러 RTL 후보를 병렬로 시뮬레이션하여 평가
        """
        tasks = []
        for i, rtl_code in enumerate(candidates):
            output_dir = f"{output_base_dir}/candidate_{i}"
            task = self._simulate_candidate(rtl_code, tb_path, output_dir)
            tasks.append(task)

        results = await asyncio.gather(*tasks)
        return results

    async def _simulate_candidate(self, rtl_code: str, tb_path: str,
                                  output_dir: str) -> Tuple[str, int]:
        loop = asyncio.get_event_loop()
        # 시뮬레이션을 별도 스레드에서 실행
        is_pass, mismatch_cnt, _ = await loop.run_in_executor(
            self.executor,
            self.simulator.run_simulation,
            [f"{output_dir}/rtl.sv"],
            [tb_path],
            output_dir
        )
        return rtl_code, mismatch_cnt
```

### 5.2 캐싱 전략 개선
```python
class SmartCache:
    """
    LLM 응답 캐싱 + 시뮬레이션 결과 캐싱
    """
    def __init__(self):
        self.llm_cache = {}  # spec hash -> RTL code
        self.sim_cache = {}  # (RTL hash, TB hash) -> sim result

    def get_cached_rtl(self, spec: str) -> Optional[str]:
        spec_hash = self._hash(spec)
        return self.llm_cache.get(spec_hash)

    def get_cached_sim_result(self, rtl_code: str, tb_code: str) -> Optional[Tuple[bool, int]]:
        cache_key = (self._hash(rtl_code), self._hash(tb_code))
        return self.sim_cache.get(cache_key)
```

## 6. 멀티모달 입력 지원 (Gemini 활용)

### 6.1 블록 다이어그램 입력
```python
class MultiModalRTLGenerator:
    def __init__(self, gemini_llm):
        self.llm = gemini_llm  # Gemini는 이미지 입력 지원

    def generate_from_diagram(self, diagram_path: str, text_spec: str) -> str:
        """
        블록 다이어그램 + 텍스트 스펙으로 RTL 생성
        """
        prompt = f"""
        다음 블록 다이어그램과 텍스트 스펙을 기반으로 SystemVerilog RTL을 생성하세요:

        <text_spec>{text_spec}</text_spec>
        """

        # Gemini는 이미지를 직접 입력으로 받을 수 있음
        with open(diagram_path, 'rb') as f:
            image_data = f.read()

        response = self.llm.complete(
            prompt,
            image_documents=[image_data]
        )
        return response.text
```

### 6.2 타이밍 다이어그램 파싱
```python
class TimingDiagramParser:
    def parse_timing_diagram(self, diagram_image: str) -> Dict[str, Any]:
        """
        타이밍 다이어그램 이미지를 파싱하여 프로토콜 추출
        """
        prompt = """
        이 타이밍 다이어그램을 분석하여 다음을 추출하세요:
        1. 클록 주기
        2. 신호 간 타이밍 관계
        3. 프로토콜 시퀀스
        4. 셋업/홀드 타임

        JSON 형식으로 출력하세요.
        """
        # Gemini로 타이밍 다이어그램 분석
        protocol_info = self.gemini_llm.analyze_image(diagram_image, prompt)
        return json.loads(protocol_info)
```

## 7. 자동 테스트벤치 개선

### 7.1 Constrained Random 테스트벤치
```python
class ConstrainedRandomTBGenerator:
    def generate_uvm_testbench(self, spec: str, rtl_interface: str) -> str:
        """
        UVM 기반 Constrained Random 테스트벤치 생성
        """
        prompt = f"""
        다음 RTL 인터페이스에 대한 UVM 테스트벤치를 생성하세요:

        <interface>{rtl_interface}</interface>
        <spec>{spec}</spec>

        요구사항:
        1. UVM sequence item 정의
        2. UVM driver/monitor/agent 생성
        3. Constrained random stimulus
        4. Functional coverage 포함
        5. Scoreboard 구현
        """
        return self.llm.chat(prompt)
```

### 7.2 Corner Case 생성
```python
class CornerCaseGenerator:
    def identify_corner_cases(self, spec: str) -> List[str]:
        """
        스펙으로부터 corner case 식별
        """
        prompt = f"""
        다음 스펙을 분석하여 테스트해야 할 corner case를 나열하세요:
        <spec>{spec}</spec>

        고려사항:
        1. 경계 값 (0, max, overflow)
        2. 동시 이벤트
        3. 리셋 타이밍
        4. Back-to-back 트랜잭션
        5. 파이프라인 stall/flush 조건
        """
        corner_cases = self.llm.chat(prompt)
        return self.parse_test_cases(corner_cases)

    def generate_directed_tests(self, corner_cases: List[str]) -> str:
        """
        각 corner case에 대한 directed test 생성
        """
        # LLM으로 각 케이스별 테스트 생성
        pass
```

## 8. 성능 모니터링 및 분석

### 8.1 생성 품질 메트릭
```python
class QualityMetrics:
    def __init__(self):
        self.metrics = {
            "first_pass_rate": 0.0,  # 첫 시도 성공률
            "avg_iterations": 0.0,    # 평균 반복 횟수
            "avg_token_cost": 0.0,    # 평균 토큰 비용
            "avg_time": 0.0,          # 평균 생성 시간
            "syntax_error_rate": 0.0, # 문법 오류율
            "avg_mismatch_before_fix": 0.0,  # 수정 전 평균 mismatch
        }

    def log_generation_result(self, task_id: str, result: Dict[str, Any]):
        """
        각 생성 결과를 로깅하여 메트릭 계산
        """
        pass

    def generate_report(self) -> str:
        """
        품질 메트릭 리포트 생성
        """
        pass
```

### 8.2 A/B 테스트 프레임워크
```python
class ABTestFramework:
    def compare_models(self,
                      model_a_config: Dict,
                      model_b_config: Dict,
                      test_set: List[str]) -> Dict[str, Any]:
        """
        두 모델 설정을 비교 평가
        """
        results_a = self.run_batch(model_a_config, test_set)
        results_b = self.run_batch(model_b_config, test_set)

        return {
            "model_a": self.compute_metrics(results_a),
            "model_b": self.compute_metrics(results_b),
            "winner": self.determine_winner(results_a, results_b),
        }
```

## 9. 구현 우선순위

### Phase 1 (단기 - 1-2개월)
1. ✅ 상용 시뮬레이터 백엔드 추가 (xrun, VCS)
2. ✅ 개선된 프롬프트 (업계 표준, 검증 가이드)
3. ✅ Claude API를 RTLEditor에 우선 적용

### Phase 2 (중기 - 3-4개월)
1. ✅ Gemini Flash를 후보 생성에 활용
2. ✅ 병렬 평가 파이프라인 구축
3. ✅ Verilator lint 통합
4. ✅ 코드 리뷰 에이전트 추가

### Phase 3 (장기 - 5-6개월)
1. ✅ 하이브리드 멀티모델 아키텍처
2. ✅ 커버리지 기반 피드백 루프
3. ✅ UVM 테스트벤치 자동 생성
4. ✅ 형식 검증 통합

## 10. 프로덕션 레벨 구현 상세

### 10.1 멀티-에이전트 코드 생성 파이프라인

#### 10.1.1 스펙 분석 및 분해 에이전트 (Spec Analyzer)
```python
class SpecAnalyzerAgent:
    """
    스펙을 상세 분석하고 구현 가능한 하위 모듈로 분해
    """
    def __init__(self, claude_llm):
        self.llm = claude_llm  # Claude의 긴 컨텍스트와 분석 능력 활용

    def analyze_spec(self, spec: str) -> SpecAnalysis:
        prompt = f"""
당신은 RTL 설계 전문가입니다. 주어진 스펙을 상세히 분석하세요.

<specification>
{spec}
</specification>

다음 형식으로 분석 결과를 JSON으로 출력하세요:
{{
    "module_name": "모듈 이름",
    "complexity_score": 1-10,  // 1=매우 간단, 10=매우 복잡
    "design_type": "combinational|sequential|mixed|fsm|datapath",
    "clock_domains": ["clk", ...],  // 클록 도메인 목록
    "reset_strategy": "sync|async|both",
    "interface": {{
        "inputs": [
            {{"name": "port명", "width": 비트폭, "type": "data|control|clock|reset"}},
            ...
        ],
        "outputs": [
            {{"name": "port명", "width": 비트폭, "type": "data|status|control"}},
            ...
        ],
        "parameters": [
            {{"name": "파라미터명", "default_value": "기본값", "description": "설명"}},
            ...
        ]
    }},
    "functional_blocks": [
        {{
            "name": "블록명",
            "description": "기능 설명",
            "inputs": ["신호명", ...],
            "outputs": ["신호명", ...],
            "type": "combinational|sequential|fsm"
        }},
        ...
    ],
    "timing_requirements": {{
        "critical_paths": ["경로 설명", ...],
        "setup_constraints": ["제약 조건", ...],
        "multicycle_paths": ["멀티사이클 경로", ...]
    }},
    "edge_cases": [
        "엣지 케이스 1",
        "엣지 케이스 2",
        ...
    ],
    "potential_issues": [
        {{
            "issue": "문제점",
            "severity": "high|medium|low",
            "mitigation": "해결 방안"
        }},
        ...
    ],
    "implementation_strategy": {{
        "approach": "top-down|bottom-up|mixed",
        "recommended_order": ["블록1", "블록2", ...],
        "decomposition": {{
            "should_decompose": true|false,
            "submodules": [
                {{
                    "name": "서브모듈명",
                    "reason": "분해 이유",
                    "interface": "인터페이스 설명"
                }},
                ...
            ]
        }}
    }}
}}

중요:
1. 스펙에서 명시되지 않은 부분은 업계 표준 관례를 따라 추론하세요
2. FSM이 필요한 경우 상태 전이 다이어그램을 텍스트로 설명하세요
3. 타이밍 제약이 있는 경우 상세히 기술하세요
4. 파이프라인, FIFO, 메모리 등 특수 구조가 필요한지 판단하세요
"""
        response = self.llm.complete(prompt)
        analysis = json.loads(response.text)
        return SpecAnalysis(**analysis)

    def generate_micro_architecture(self, analysis: SpecAnalysis) -> str:
        """
        분석 결과를 바탕으로 마이크로 아키텍처 문서 생성
        """
        prompt = f"""
다음 분석 결과를 바탕으로 상세한 마이크로 아키텍처 문서를 작성하세요:

<analysis>
{json.dumps(analysis.dict(), indent=2)}
</analysis>

마이크로 아키텍처 문서에 포함할 내용:
1. 블록 다이어그램 (ASCII art)
2. 각 블록의 상세 기능
3. 블록 간 인터페이스 프로토콜
4. 타이밍 다이어그램 (주요 시나리오)
5. 상태 머신 정의 (필요시)
6. 레지스터/신호 목록 및 용도
7. 데이터 경로 및 제어 경로
8. 예상되는 합성 결과 (게이트 수, 플립플롭 수 추정)
"""
        response = self.llm.complete(prompt)
        return response.text
```

#### 10.1.2 멀티-스테이지 RTL 생성 (Multi-Stage Generation)
```python
class MultiStageRTLGenerator:
    """
    단계별로 RTL을 정제하는 생성기
    """
    def __init__(self, claude_llm, gemini_pro_llm, gemini_flash_llm):
        self.claude = claude_llm
        self.gemini_pro = gemini_pro_llm
        self.gemini_flash = gemini_flash_llm

    def generate_stage1_skeleton(self, spec_analysis: SpecAnalysis) -> str:
        """
        Stage 1: Claude로 고품질 스켈레톤 생성
        - 모듈 선언, 포트 정의, 주요 신호 선언
        - 블록 구조 및 주석
        """
        prompt = f"""
당신은 20년 경력의 시니어 RTL 디자이너입니다.
다음 분석 결과를 바탕으로 SystemVerilog RTL 스켈레톤을 작성하세요.

<spec_analysis>
{json.dumps(spec_analysis.dict(), indent=2)}
</spec_analysis>

생성 규칙:
1. 모듈 선언 및 파라미터 정의
2. 모든 포트 선언 (logic 타입 사용)
3. 내부 신호 선언 (용도별로 그룹화 및 주석)
4. 주요 기능 블록을 주석으로 구조화
5. FSM이 필요한 경우 state enum 정의
6. 실제 로직은 TODO 주석으로 표시

출력 형식:
- 각 섹션을 명확히 구분 (// ===== Section Name ===== 형식)
- 복잡한 신호는 용도 주석 추가
- 타이밍에 민감한 부분은 TIMING_CRITICAL 주석
- 합성 불가능한 부분은 SIMULATION_ONLY 주석

예시:
```systemverilog
// ===== Module Declaration =====
module TopModule #(
    parameter int WIDTH = 8  // Data width
) (
    // ===== Clock and Reset =====
    input  logic clk,
    input  logic rst_n,  // Active-low async reset

    // ===== Input Ports =====
    input  logic [WIDTH-1:0] data_in,
    input  logic             valid_in,

    // ===== Output Ports =====
    output logic [WIDTH-1:0] data_out,
    output logic             valid_out
);

// ===== Internal Signals =====
// State Machine
typedef enum logic [1:0] {{
    IDLE = 2'b00,
    PROCESS = 2'b01,
    DONE = 2'b10
}} state_t;
state_t current_state, next_state;

// Datapath signals
logic [WIDTH-1:0] data_reg;
logic [WIDTH-1:0] result;

// ===== State Register =====
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        current_state <= IDLE;
    end else begin
        current_state <= next_state;
    end
end

// ===== Next State Logic =====
always_comb begin
    // TODO: Implement next state logic
    next_state = current_state;
end

// ===== Output Logic =====
always_comb begin
    // TODO: Implement output logic
    data_out = '0;
    valid_out = 1'b0;
end

endmodule
```
"""
        response = self.claude.complete(prompt)
        return response.text

    def generate_stage2_implementation(self,
                                      skeleton: str,
                                      spec_analysis: SpecAnalysis,
                                      micro_arch: str) -> List[str]:
        """
        Stage 2: Gemini Flash로 다양한 구현 변형 생성 (빠르고 저렴)
        """
        base_prompt = f"""
다음 RTL 스켈레톤의 TODO 부분을 구현하세요.

<skeleton>
{skeleton}
</skeleton>

<micro_architecture>
{micro_arch}
</micro_architecture>

<spec_analysis>
{json.dumps(spec_analysis.dict(), indent=2)}
</spec_analysis>

구현 가이드:
1. 모든 TODO 주석을 실제 로직으로 교체
2. FSM: 3-always block 패턴 (state_reg, next_state_logic, output_logic)
3. Datapath: 파이프라인 스테이지 명확히 구분
4. Control: 각 제어 신호의 생성 조건 명확히
5. Edge case: 모든 엣지 케이스 처리
6. Reset: 모든 레지스터의 리셋 동작 정의

코딩 스타일:
- Blocking assignment (=): combinational always_comb
- Non-blocking assignment (<=): sequential always_ff
- 모든 combinational 블록에 default 값 명시 (latch 방지)
- X-propagation 방지: 모든 케이스 커버
"""

        # 여러 변형 생성 (temperature 조절)
        candidates = []
        for i, temp in enumerate([0.3, 0.5, 0.7, 0.85, 1.0]):
            variation_prompt = base_prompt + f"""

구현 변형 {i+1}:
- Temperature: {temp}
- 초점: {"보수적, 안전한 구현" if temp < 0.5 else "최적화된 구현" if temp < 0.8 else "창의적 구현"}
"""
            # Gemini Flash로 빠르게 생성
            response = self.gemini_flash.complete(
                variation_prompt,
                temperature=temp
            )
            candidates.append(response.text)

        return candidates

    def generate_stage3_refinement(self,
                                   candidates: List[str],
                                   lint_results: List[Dict],
                                   spec_analysis: SpecAnalysis) -> str:
        """
        Stage 3: Claude로 최적 후보 선택 및 정제
        """
        candidates_with_lint = [
            {
                "code": code,
                "lint_warnings": lint_results[i],
                "index": i
            }
            for i, code in enumerate(candidates)
        ]

        prompt = f"""
당신은 RTL 설계 리뷰 전문가입니다.
다음 RTL 구현 후보들을 평가하고 최적의 코드를 선택하거나 개선하세요.

<candidates>
{json.dumps(candidates_with_lint, indent=2)}
</candidates>

<spec_analysis>
{json.dumps(spec_analysis.dict(), indent=2)}
</spec_analysis>

평가 기준:
1. 기능 정확성 (스펙 완전 구현 여부)
2. 코드 품질 (가독성, 유지보수성)
3. 합성 효율성 (예상 게이트 수, 타이밍)
4. Lint 위반 심각도
5. 엣지 케이스 처리 완전성
6. 업계 표준 준수

작업:
1. 각 후보를 평가하고 점수 부여 (0-100점)
2. 최고 점수 후보 선택
3. 선택된 후보의 문제점 개선
4. 최종 RTL 코드 출력

출력 형식:
<evaluation>
[각 후보 평가 내용]
</evaluation>

<selected_candidate>
[선택된 후보 번호 및 이유]
</selected_candidate>

<improvements>
[적용할 개선 사항]
</improvements>

<final_rtl>
```systemverilog
[최종 RTL 코드]
```
</final_rtl>
"""
        response = self.claude.complete(prompt)
        return self._extract_final_rtl(response.text)
```

#### 10.1.3 체계적 코드 리뷰 에이전트
```python
class SystematicCodeReviewer:
    """
    다층 검토 시스템: 문법 → 의미 → 타이밍 → 검증성
    """
    def __init__(self, claude_llm, gemini_pro_llm):
        self.claude = claude_llm
        self.gemini_pro = gemini_pro_llm

    def layer1_syntax_review(self, rtl_code: str) -> SyntaxReviewResult:
        """
        Layer 1: 문법 및 스타일 검토
        """
        prompt = f"""
다음 SystemVerilog RTL 코드의 문법 및 스타일을 검토하세요.

<rtl_code>
{rtl_code}
</rtl_code>

검토 항목:
1. SystemVerilog 문법 오류
2. 린트 규칙 위반 (Latch 생성, Full case, Parallel case)
3. 코딩 스타일 위반
   - 네이밍 컨벤션
   - Blocking vs Non-blocking assignment
   - 클록 도메인 혼용
4. 잠재적 합성 문제
   - Combinational loop
   - Multiple driver
   - Inferred latch
5. X-propagation 위험
6. 경고 메시지 해석

JSON 형식으로 출력:
{{
    "syntax_errors": [
        {{"line": 라인번호, "error": "오류내용", "severity": "error|warning"}}
    ],
    "style_violations": [
        {{"line": 라인번호, "issue": "위반내용", "suggestion": "수정방안"}}
    ],
    "synthesis_issues": [
        {{"type": "latch|loop|driver", "location": "위치", "fix": "해결방법"}}
    ],
    "overall_score": 0-100,
    "pass": true|false
}}
"""
        response = self.claude.complete(prompt)
        return SyntaxReviewResult(**json.loads(response.text))

    def layer2_semantic_review(self,
                               rtl_code: str,
                               spec: str,
                               spec_analysis: SpecAnalysis) -> SemanticReviewResult:
        """
        Layer 2: 의미 및 기능 검토
        """
        prompt = f"""
당신은 RTL 검증 전문가입니다.
다음 RTL 코드가 스펙을 정확히 구현했는지 검토하세요.

<specification>
{spec}
</specification>

<spec_analysis>
{json.dumps(spec_analysis.dict(), indent=2)}
</spec_analysis>

<rtl_code>
{rtl_code}
</rtl_code>

심층 분석:
1. 기능 완전성
   - 스펙의 모든 요구사항 구현 여부
   - 누락된 기능 식별
   - 과도하게 추가된 기능

2. 로직 정확성
   - FSM 상태 전이 정확성
   - Datapath 연산 정확성
   - Control signal 생성 로직

3. 엣지 케이스 처리
   - Overflow/Underflow
   - Reset 동작
   - Back-pressure 처리
   - Simultaneous events

4. 인터페이스 프로토콜
   - Handshake 시퀀스
   - Valid/Ready 관계
   - 타이밍 관계

5. 데이터 무결성
   - 데이터 손실 가능성
   - 데이터 재정렬 문제
   - Stale data 사용

JSON 출력:
{{
    "completeness": {{
        "implemented_features": ["기능1", ...],
        "missing_features": ["누락기능1", ...],
        "extra_features": ["추가기능1", ...]
    }},
    "logic_issues": [
        {{
            "location": "위치",
            "issue": "문제점",
            "severity": "critical|major|minor",
            "expected_behavior": "예상동작",
            "actual_behavior": "실제동작",
            "fix_suggestion": "수정방안"
        }}
    ],
    "edge_cases": [
        {{
            "case": "엣지케이스",
            "handled": true|false,
            "issue": "문제점 (미처리시)"
        }}
    ],
    "overall_correctness_score": 0-100,
    "pass": true|false,
    "critical_issues": ["치명적문제1", ...]
}}
"""
        response = self.claude.complete(prompt)
        return SemanticReviewResult(**json.loads(response.text))

    def layer3_timing_review(self, rtl_code: str) -> TimingReviewResult:
        """
        Layer 3: 타이밍 및 성능 검토
        """
        prompt = f"""
RTL 코드의 타이밍 특성을 분석하세요.

<rtl_code>
{rtl_code}
</rtl_code>

분석 항목:
1. Critical Path 식별
   - 가장 긴 조합 로직 경로
   - 예상 지연 시간
   - 병목 구간

2. Setup/Hold 위반 가능성
   - 비동기 신호 처리
   - CDC (Clock Domain Crossing) 문제
   - Metastability 위험

3. 파이프라인 효율성
   - 파이프라인 스테이지 균형
   - Bubble 발생 가능성
   - Throughput 분석

4. 리소스 사용량 추정
   - LUT 수
   - Flip-flop 수
   - BRAM/DSP 사용

5. 클록 주파수 예상
   - 목표 주파수 달성 가능성
   - 최적화 포인트

JSON 출력:
{{
    "critical_paths": [
        {{
            "path": "경로설명",
            "estimated_delay_ns": 예상지연,
            "bottleneck": "병목위치"
        }}
    ],
    "timing_violations": [
        {{
            "type": "setup|hold|cdc",
            "location": "위치",
            "severity": "critical|warning",
            "fix": "해결방안"
        }}
    ],
    "resource_estimation": {{
        "luts": 추정값,
        "ffs": 추정값,
        "brams": 추정값
    }},
    "max_frequency_mhz": 예상최대주파수,
    "optimization_suggestions": ["최적화제안1", ...],
    "timing_score": 0-100
}}
"""
        response = self.gemini_pro.complete(prompt)
        return TimingReviewResult(**json.loads(response.text))

    def layer4_verifiability_review(self,
                                    rtl_code: str,
                                    spec_analysis: SpecAnalysis) -> VerifiabilityReviewResult:
        """
        Layer 4: 검증 용이성 검토
        """
        prompt = f"""
RTL 코드의 검증 용이성을 평가하세요.

<rtl_code>
{rtl_code}
</rtl_code>

<spec_analysis>
{json.dumps(spec_analysis.dict(), indent=2)}
</spec_analysis>

평가 기준:
1. 관측성 (Observability)
   - 내부 상태 모니터링 용이성
   - 디버그 신호 충분성
   - FSM 상태 외부 노출

2. 제어성 (Controllability)
   - 테스트 모드 지원
   - 강제 설정 가능 여부
   - 초기화 용이성

3. 커버리지 달성 용이성
   - 모든 브랜치 도달 가능성
   - Corner case 자극 방법
   - 커버하기 어려운 코드

4. Assertion 필요 위치
   - 불변 조건 (Invariant)
   - 프로토콜 규칙
   - 데이터 무결성

5. 예상 버그 패턴
   - 자주 실수하는 부분
   - 검증 중 발견될 가능성이 높은 버그

JSON 출력:
{{
    "observability_score": 0-100,
    "controllability_score": 0-100,
    "suggested_assertions": [
        {{
            "location": "위치",
            "assertion": "SVA 코드",
            "purpose": "목적"
        }}
    ],
    "debug_signals_to_add": [
        {{"signal": "신호명", "purpose": "목적"}}
    ],
    "hard_to_verify_areas": [
        {{
            "location": "위치",
            "difficulty": "어려운이유",
            "suggestion": "검증방안"
        }}
    ],
    "potential_bugs": [
        {{
            "location": "위치",
            "bug_pattern": "버그유형",
            "likelihood": "high|medium|low",
            "test_scenario": "발견방법"
        }}
    ],
    "verifiability_score": 0-100
}}
"""
        response = self.claude.complete(prompt)
        return VerifiabilityReviewResult(**json.loads(response.text))

    def generate_comprehensive_review_report(self,
                                            syntax: SyntaxReviewResult,
                                            semantic: SemanticReviewResult,
                                            timing: TimingReviewResult,
                                            verifiability: VerifiabilityReviewResult) -> str:
        """
        종합 검토 리포트 생성
        """
        prompt = f"""
다음 4가지 계층의 검토 결과를 종합하여 최종 리포트를 작성하세요.

<syntax_review>
{json.dumps(syntax.dict(), indent=2)}
</syntax_review>

<semantic_review>
{json.dumps(semantic.dict(), indent=2)}
</semantic_review>

<timing_review>
{json.dumps(timing.dict(), indent=2)}
</timing_review>

<verifiability_review>
{json.dumps(verifiability.dict(), indent=2)}
</verifiability_review>

종합 리포트 형식:
1. Executive Summary
   - 전체 품질 점수 (0-100)
   - Pass/Fail 판정
   - 주요 발견사항 (Top 5)

2. 치명적 이슈 (Critical Issues)
   - 즉시 수정 필요 항목
   - 우선순위

3. 개선 권장사항 (Recommendations)
   - 코드 품질 개선
   - 성능 최적화
   - 검증성 향상

4. 상세 분석 결과
   - 계층별 점수 및 설명

5. 수정 계획 (Remediation Plan)
   - 수정 순서
   - 예상 소요 시간
   - 리스크
"""
        response = self.claude.complete(prompt)
        return response.text
```

#### 10.1.4 지능형 RTL 편집기 (Intelligent Editor)
```python
class IntelligentRTLEditor:
    """
    시뮬레이션 실패 원인을 분석하고 정밀하게 수정
    """
    def __init__(self, claude_llm):
        self.claude = claude_llm
        self.edit_history = []

    def analyze_simulation_failure(self,
                                   rtl_code: str,
                                   testbench: str,
                                   sim_log: str,
                                   spec: str,
                                   vcd_file: Optional[str] = None) -> FailureAnalysis:
        """
        시뮬레이션 실패 원인 심층 분석
        """
        # VCD 파형이 있으면 파형 분석도 포함
        vcd_analysis = ""
        if vcd_file:
            vcd_analysis = self._analyze_waveform(vcd_file, sim_log)

        prompt = f"""
당신은 RTL 디버깅 전문가입니다.
시뮬레이션 실패 원인을 철저히 분석하세요.

<specification>
{spec}
</specification>

<rtl_code>
{add_lineno(rtl_code)}  # 라인 번호 추가
</rtl_code>

<testbench>
{testbench}
</testbench>

<simulation_log>
{sim_log}
</simulation_log>

{"<waveform_analysis>" + vcd_analysis + "</waveform_analysis>" if vcd_analysis else ""}

분석 절차:
1. Mismatch 패턴 분석
   - 어느 시간대에 mismatch 발생?
   - 어떤 신호에서 발생?
   - Mismatch 값: Expected vs Actual
   - 일시적인가 지속적인가?

2. 근본 원인 추적 (Root Cause Analysis)
   - Mismatch 신호의 생성 로직 추적
   - 입력 신호 상태 확인
   - 제어 신호 상태 확인
   - FSM 상태 확인 (해당시)

3. 버그 분류
   - 타이밍 버그 (순서, 지연)
   - 로직 버그 (잘못된 연산, 조건)
   - 초기화 버그 (리셋 동작)
   - 엣지 케이스 미처리

4. 영향 범위 파악
   - 버그가 영향을 미치는 다른 신호
   - 연쇄 오류 가능성

5. 수정 전략 수립
   - 수정할 코드 위치 (라인 번호)
   - 수정 방법 (구체적)
   - 부작용 검토

JSON 출력:
{{
    "mismatches": [
        {{
            "time": "시간",
            "signal": "신호명",
            "expected": "예상값",
            "actual": "실제값",
            "pattern": "일시적|지속적|주기적"
        }}
    ],
    "root_causes": [
        {{
            "bug_type": "timing|logic|initialization|edge_case",
            "location": {{
                "file": "rtl.sv",
                "line_start": 시작라인,
                "line_end": 종료라인,
                "code_snippet": "해당코드"
            }},
            "description": "버그설명",
            "why_wrong": "왜 틀렸는지",
            "correct_behavior": "올바른 동작",
            "confidence": 0-100
        }}
    ],
    "fix_strategies": [
        {{
            "root_cause_index": 0,  # 위 root_causes 인덱스
            "fix_type": "replace|insert|delete|refactor",
            "location": {{
                "line_start": 시작라인,
                "line_end": 종료라인
            }},
            "old_code": "기존코드",
            "new_code": "수정코드",
            "explanation": "수정이유",
            "side_effects": ["부작용1", ...],
            "verification_points": ["확인사항1", ...]
        }}
    ],
    "priority": 1-5,  # 1=가장 급함
    "estimated_fix_complexity": "trivial|easy|medium|hard|very_hard"
}}
"""
        response = self.claude.complete(prompt)
        return FailureAnalysis(**json.loads(response.text))

    def _analyze_waveform(self, vcd_file: str, sim_log: str) -> str:
        """
        VCD 파형 파일을 분석하여 텍스트 설명 생성
        """
        # VCD 파일 파싱 (pyvcd 사용 등)
        # Mismatch 시점의 모든 신호 값 추출
        # 시간대별 신호 변화 추적
        # Claude에게 전달할 텍스트 형식으로 변환
        pass

    def apply_fix_with_validation(self,
                                  rtl_code: str,
                                  fix_strategy: dict,
                                  spec_analysis: SpecAnalysis) -> Tuple[str, FixValidation]:
        """
        수정 적용 및 검증
        """
        # 1. 수정 적용
        modified_code = self._apply_fix(rtl_code, fix_strategy)

        # 2. 수정 검증
        validation_prompt = f"""
다음 RTL 수정이 올바른지 검증하세요.

<original_code>
{fix_strategy['old_code']}
</original_code>

<modified_code>
{fix_strategy['new_code']}
</modified_code>

<fix_explanation>
{fix_strategy['explanation']}
</fix_explanation>

<full_rtl>
{add_lineno(modified_code)}
</full_rtl>

검증 항목:
1. 문법 오류 없는지
2. 수정이 의도한 대로 동작할지
3. 다른 부분에 부작용 없는지
4. 새로운 버그 발생 가능성
5. 더 나은 수정 방법 있는지

JSON 출력:
{{
    "is_valid": true|false,
    "syntax_ok": true|false,
    "logic_ok": true|false,
    "side_effects_ok": true|false,
    "issues_found": ["문제1", ...],
    "alternative_fix": "더 나은 수정방법 (있다면)",
    "confidence": 0-100
}}
"""
        response = self.claude.complete(validation_prompt)
        validation = FixValidation(**json.loads(response.text))

        self.edit_history.append({
            "fix_strategy": fix_strategy,
            "validation": validation,
            "timestamp": time.time()
        })

        return modified_code, validation

    def iterative_refinement(self,
                            initial_rtl: str,
                            spec: str,
                            testbench: str,
                            simulator: SimulatorManager,
                            max_iterations: int = 10) -> Tuple[str, List[dict]]:
        """
        반복적 개선 루프
        """
        current_rtl = initial_rtl
        iteration_results = []

        for iteration in range(max_iterations):
            logger.info(f"Refinement iteration {iteration + 1}/{max_iterations}")

            # 1. 시뮬레이션
            is_pass, mismatch_cnt, sim_log = simulator.run_simulation(
                rtl_files=[self._write_temp(current_rtl, "rtl.sv")],
                tb_files=[testbench],
                output_dir=f"/tmp/iter_{iteration}"
            )

            iteration_result = {
                "iteration": iteration,
                "is_pass": is_pass,
                "mismatch_cnt": mismatch_cnt
            }

            if is_pass:
                logger.info(f"Success at iteration {iteration + 1}!")
                iteration_result["status"] = "success"
                iteration_results.append(iteration_result)
                break

            # 2. 실패 분석
            failure_analysis = self.analyze_simulation_failure(
                rtl_code=current_rtl,
                testbench=testbench,
                sim_log=sim_log,
                spec=spec
            )

            # 3. 수정 전략 선택 (가장 confidence 높은 것)
            best_fix = max(
                failure_analysis.fix_strategies,
                key=lambda x: x['confidence']
            )

            # 4. 수정 적용
            current_rtl, fix_validation = self.apply_fix_with_validation(
                rtl_code=current_rtl,
                fix_strategy=best_fix,
                spec_analysis=spec_analysis
            )

            iteration_result.update({
                "status": "fixed",
                "failure_analysis": failure_analysis.dict(),
                "applied_fix": best_fix,
                "fix_validation": fix_validation.dict()
            })
            iteration_results.append(iteration_result)

            # 5. 수정이 유효하지 않으면 대안 시도
            if not fix_validation.is_valid and len(failure_analysis.fix_strategies) > 1:
                logger.warning("Primary fix invalid, trying alternative...")
                # 다음 후보 시도
                pass

        return current_rtl, iteration_results
```

### 10.2 고급 테스트벤치 생성

#### 10.2.1 계층적 테스트벤치 생성
```python
class HierarchicalTBGenerator:
    """
    UVM 스타일의 계층적 테스트벤치 생성
    """
    def __init__(self, claude_llm):
        self.claude = claude_llm

    def generate_uvm_environment(self,
                                 spec_analysis: SpecAnalysis,
                                 rtl_interface: str) -> Dict[str, str]:
        """
        UVM 환경 전체 생성: Agent, Driver, Monitor, Scoreboard, Test
        """
        # 1. Transaction 정의
        transaction_code = self._generate_transaction(spec_analysis)

        # 2. Driver 생성
        driver_code = self._generate_driver(spec_analysis, rtl_interface)

        # 3. Monitor 생성
        monitor_code = self._generate_monitor(spec_analysis, rtl_interface)

        # 4. Agent 생성
        agent_code = self._generate_agent(spec_analysis)

        # 5. Scoreboard 생성
        scoreboard_code = self._generate_scoreboard(spec_analysis)

        # 6. Sequence 생성
        sequences_code = self._generate_sequences(spec_analysis)

        # 7. Environment 생성
        env_code = self._generate_environment(spec_analysis)

        # 8. Test 생성
        test_code = self._generate_test(spec_analysis)

        return {
            "transaction": transaction_code,
            "driver": driver_code,
            "monitor": monitor_code,
            "agent": agent_code,
            "scoreboard": scoreboard_code,
            "sequences": sequences_code,
            "environment": env_code,
            "test": test_code
        }

    def _generate_scoreboard(self, spec_analysis: SpecAnalysis) -> str:
        """
        참조 모델을 포함한 Scoreboard 생성
        """
        prompt = f"""
다음 스펙에 대한 UVM Scoreboard를 생성하세요.

<spec_analysis>
{json.dumps(spec_analysis.dict(), indent=2)}
</spec_analysis>

Scoreboard 요구사항:
1. 참조 모델 (Reference Model) 포함
   - SystemVerilog로 기대 동작 구현
   - DUT와 독립적으로 검증 가능
   - Golden RTL이 아닌 고수준 모델

2. Checker 로직
   - DUT 출력 vs 참조 모델 출력 비교
   - Mismatch 발견 시 상세 정보 출력
   - Coverage 수집

3. 에러 리포팅
   - Mismatch 타임스탬프
   - 입력 조건
   - 예상 값 vs 실제 값
   - 차이 발생 원인 추정

예시 구조:
```systemverilog
class scoreboard extends uvm_scoreboard;
    `uvm_component_utils(scoreboard)

    // Reference model
    function automatic logic [WIDTH-1:0] reference_model(
        input logic [WIDTH-1:0] in_data,
        input logic             control
    );
        // High-level behavioral model
        return in_data + 1;  // 예시
    endfunction

    // Checker
    virtual task run_phase(uvm_phase phase);
        forever begin
            // Get transactions from DUT monitor
            // Get transactions from reference model
            // Compare
            if (dut_output != ref_output) begin
                `uvm_error("MISMATCH", $sformatf(
                    "Time: %0t, Input: %h, Expected: %h, Got: %h",
                    $time, input_data, ref_output, dut_output
                ))
            end
        end
    endtask
endclass
```
"""
        response = self.claude.complete(prompt)
        return response.text
```

#### 10.2.2 Constrained Random 테스트 생성
```python
class ConstrainedRandomTestGenerator:
    """
    제약 기반 랜덤 테스트 생성
    """
    def generate_constraints(self, spec_analysis: SpecAnalysis) -> str:
        """
        스펙으로부터 제약 조건 추출 및 SystemVerilog constraint 생성
        """
        prompt = f"""
다음 스펙의 유효한 입력 범위에 대한 SystemVerilog constraint를 생성하세요.

<spec_analysis>
{json.dumps(spec_analysis.dict(), indent=2)}
</spec_analysis>

Constraint 요구사항:
1. 유효 범위 제약
2. 프로토콜 제약 (valid/ready 관계 등)
3. 엣지 케이스 가중치
   - Normal case: 70%
   - Boundary case: 20%
   - Corner case: 10%

예시:
```systemverilog
class transaction extends uvm_sequence_item;
    rand logic [7:0] data;
    rand logic       valid;
    rand int         delay;  // Clock cycles between transactions

    // Valid range constraints
    constraint c_data {{
        data dist {{
            [0:10] := 10,        // Low values: 10% weight
            [11:244] := 70,      // Normal range: 70% weight
            [245:255] := 20      // High values: 20% weight
        }};
    }}

    // Protocol constraints
    constraint c_valid {{
        valid dist {{ 1 := 80, 0 := 20 }};  // Valid 80% of time
    }}

    // Delay constraints
    constraint c_delay {{
        delay inside {{ [0:5] }};  // 0-5 cycle delay
        delay dist {{ 0 := 50, [1:5] := 50 }};
    }}

    // Corner case mode (can be enabled selectively)
    constraint c_corner_mode {{
        corner_case_mode -> {{
            data inside {{ 0, 255 }};  // Only extremes
        }}
    }}
endclass
```
"""
        response = self.claude.complete(prompt)
        return response.text

    def generate_directed_corner_cases(self, spec_analysis: SpecAnalysis) -> List[str]:
        """
        Directed corner case 테스트 생성
        """
        prompt = f"""
다음 스펙에 대한 corner case 테스트 시나리오를 생성하세요.

<spec_analysis>
{json.dumps(spec_analysis.dict(), indent=2)}
</spec_analysis>

각 corner case에 대해:
1. 테스트 시나리오 설명
2. SystemVerilog sequence 코드
3. 예상 결과
4. Assertion 코드

Corner case 카테고리:
- Boundary values (0, max, overflow)
- Reset during operation
- Back-to-back transactions
- Simultaneous events
- Protocol violations (if applicable)
- Resource contention
- Stall conditions
- Error injection

JSON 배열로 출력:
[
    {{
        "name": "테스트케이스명",
        "description": "설명",
        "sequence_code": "SystemVerilog 코드",
        "expected_result": "예상결과",
        "assertions": "Assertion 코드"
    }},
    ...
]
"""
        response = self.claude.complete(prompt)
        return json.loads(response.text)
```

### 10.3 형식 검증 (Formal Verification) 통합

```python
class FormalVerificationEngine:
    """
    Assertion 생성 및 형식 검증 실행
    """
    def __init__(self, claude_llm):
        self.claude = claude_llm

    def generate_protocol_assertions(self,
                                    spec: str,
                                    rtl_code: str,
                                    spec_analysis: SpecAnalysis) -> str:
        """
        프로토콜 검증을 위한 SVA 생성
        """
        prompt = f"""
다음 RTL 코드에 대한 SystemVerilog Assertion (SVA)를 생성하세요.

<specification>
{spec}
</specification>

<rtl_code>
{rtl_code}
</rtl_code>

<spec_analysis>
{json.dumps(spec_analysis.dict(), indent=2)}
</spec_analysis>

Assertion 카테고리:

1. 인터페이스 프로토콜
```systemverilog
// Valid/Ready handshake
property p_valid_ready_handshake;
    @(posedge clk) disable iff (!rst_n)
    valid && !ready |=> $stable(data) until_with ready;
endproperty
assert property (p_valid_ready_handshake)
    else $error("Data changed before handshake completed");
```

2. FSM 상태 전이
```systemverilog
// Valid state transitions only
property p_valid_state_transition;
    @(posedge clk) disable iff (!rst_n)
    (state == IDLE) |=> (state inside {{IDLE, START}});
endproperty
```

3. 데이터 무결성
```systemverilog
// No data loss
property p_no_data_loss;
    @(posedge clk) disable iff (!rst_n)
    $rose(valid_in) |-> ##[1:$] valid_out;
endproperty
```

4. 타이밍 제약
```systemverilog
// Response within N cycles
property p_response_latency;
    @(posedge clk) disable iff (!rst_n)
    request |-> ##[1:MAX_LATENCY] response;
endproperty
```

5. 불변 조건 (Invariants)
```systemverilog
// Counter never exceeds maximum
property p_counter_bound;
    @(posedge clk) disable iff (!rst_n)
    counter <= MAX_COUNT;
endproperty
```

모든 중요 동작에 대해 assertion을 생성하고,
각 assertion에 명확한 주석과 오류 메시지를 포함하세요.
"""
        response = self.claude.complete(prompt)
        return response.text

    def generate_formal_testbench(self,
                                  rtl_code: str,
                                  assertions: str) -> str:
        """
        Formal verification용 테스트벤치 생성
        """
        prompt = f"""
다음 RTL과 Assertion을 포함하는 Formal verification 테스트벤치를 생성하세요.

<rtl_code>
{rtl_code}
</rtl_code>

<assertions>
{assertions}
</assertions>

테스트벤치 요구사항:
1. Bind 문으로 assertion 모듈 연결
2. Assume 문으로 입력 제약
3. Cover 문으로 중요 시나리오 커버리지

예시:
```systemverilog
module formal_tb;
    // DUT signals
    logic clk, rst_n;
    logic [7:0] data_in;
    logic valid_in;
    logic [7:0] data_out;
    logic valid_out;

    // DUT instantiation
    TopModule dut (.*);

    // Assertions module
    module assertions_m (
        input clk, rst_n,
        input [7:0] data_in, data_out,
        input valid_in, valid_out
    );
        // Include all assertions here
        ...
    endmodule

    // Bind assertions
    bind TopModule assertions_m assertions_inst (.*);

    // Input assumptions (constraints)
    assume property (@(posedge clk) disable iff (!rst_n)
        valid_in |-> data_in < 200  // Constrain input range
    );

    // Coverage
    cover property (@(posedge clk) disable iff (!rst_n)
        ##1 valid_in ##1 !valid_in ##1 valid_in  // Back-to-back valid
    );
endmodule
```
"""
        response = self.claude.complete(prompt)
        return response.text
```

## 11. 예상 효과

### 품질 개선
- **Pass Rate 향상**: 현재 대비 15-25% 증가 예상
- **Iteration 감소**: 평균 시뮬레이션 반복 횟수 30% 감소
- **코드 품질**: 업계 표준 준수로 유지보수성 향상

### 비용 효율성
- **토큰 비용 절감**: Gemini Flash 활용으로 후보 생성 비용 60% 절감
- **시간 단축**: 병렬 평가로 전체 생성 시간 40% 단축
- **상용 시뮬레이터**: 정확도 향상으로 재작업 비용 감소

### 활용성 확대
- **산업 적용**: 상용 시뮬레이터 지원으로 실제 프로젝트 적용 가능
- **복잡도 대응**: 멀티모달 입력으로 복잡한 설계 스펙 처리
- **검증 자동화**: UVM 테스트벤치 자동 생성으로 검증 시간 단축
