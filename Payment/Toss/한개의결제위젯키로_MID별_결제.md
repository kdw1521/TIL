# 결제위젯 연동 키로 MID 별 결제되도록 하기

토스틑 한개의 결제위젯 연동 키로 여러 상점에서 결제가 가능하도록 구성되어있다.  
여러개의 상점(MID)이 있을때 한개의 키로 상점별 결제가 되도록 구성한다.

# 준비

- 토스페이먼츠 구성

# 구현

1. 토스 페이먼츠 콘솔 진입 > 결제 UI 설정 에 진입
2. 라이브, 테스트 2개의 영역이 있는데 우선 테스트 쪽에 추가해준다. - 테스트 UI 추가하기 클릭
3. 결제 UI 를 선택하고, 사용할 상점아이디(MID) 와 `variantKey` 를 설정해준다.
4. 결제 위젯 연동 키와 3번에서 지정한 variantKey로 지정하여 프론트에서 위젯을 로드한다.

결론적으로 각 상점아이디(MID) 별로 결제 UI 를 추가해 variantKey로 구분하여 위젯을 로드한다. 여기서 일어나는 결제데이터를 가지고 있다가 취소 등을 호출해 진행하면 해당 상점으로 연결이 되어있어 문제가 없다.
