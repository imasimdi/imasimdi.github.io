# ECS - 모른 것만 필기
- 가급적 클라우드는 AWS-managed 를 사용하는 것을 추천
## ECS 클러스터 생성
- Cluster = Container들을 묶어 놓는 곳
    - 쿠버네티스 용어인데.. 가져온 듯
    - 서비스가 컨테이너다
    - 네임 스페이스: 클러스터 안에서도 트리 처럼 나눠진다. 개인적으로는 비추, 불편해진다
    - fargate vs ec2: fargate가 더 싼데 ec2는 성능이 더 좋다. 상시는 ec2가 좋다 ..?
- Task: 임무(task)를 먼저 생성하고 Task를 활용해서 Container를 가동
    - ECS용어로 Contianer 가동은 Service 추가
    - MSA에서 주로 사용되기 때문에, Service = Container라고 생각
    - task role이 중요하다. 권한이 필요함
        ECR, RDS, DYNAMO DB 등등
    - entry point: run할 때 커맨드 (볼륨 마운트 등등)
- Service
    - rolling vs blue/green -> code deploy면 blue/green, rolling은 깃헙액션
    - 로드밸런스랑 같은 VPC에 있어야한다.
    - Load balance - 포트 중요, 리스너는 로드밸런스가 받는 포트