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

## 10. 예상 효과

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
