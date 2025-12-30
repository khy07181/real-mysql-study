# Week 4: 인덱스 (2) ⭐ - 문제

## 📚 학습 범위

- 08장: 복합 인덱스, 인덱스 스캔 방식, 커버링 인덱스

---

## 문제

### Q1. 복합 인덱스 컬럼 순서 (연호, 진영)

복합 인덱스 (a, b, c)가 있을 때, 다음 쿼리 중 인덱스를 효율적으로 사용할 수 있는 것은?

```sql
1) WHERE a = 1
2) WHERE b = 1
3) WHERE a = 1 AND c = 1
4) WHERE a = 1 AND b = 1 AND c = 1
5) WHERE a = 1 ORDER BY b
6) WHERE b = 1 ORDER BY a
```

### Q2. 복합 인덱스 설계 (진영, 하영)

다음 쿼리를 위한 최적의 복합 인덱스를 설계하세요:

```sql
SELECT * FROM orders
WHERE user_id = 100
  AND status = 'completed'
ORDER BY created_at DESC
LIMIT 10;
```

### Q3. 커버링 인덱스 (하영, 영섭)

커버링 인덱스란 무엇이며, 어떤 장점이 있나요?

### Q4. 인덱스 조건 푸시다운 (영섭, 지훈)

인덱스 조건 푸시다운(Index Condition Pushdown, ICP)이란 무엇인가요?

### Q5. 인덱스 머지 (지훈, 주호)

인덱스 머지(Index Merge)란 무엇이며, 어떤 종류가 있나요?

### Q6. 인덱스 힌트 (주호, 현서)

인덱스 힌트(USE INDEX, FORCE INDEX, IGNORE INDEX)의 차이점과 사용 시 주의사항은 무엇인가요?

### Q7. 유니크 인덱스 vs 일반 인덱스 (현서, 연호)

유니크 인덱스와 일반 세컨더리 인덱스의 성능 차이를 설명하세요.

### Q8. 실무/면접 질문

"인덱스가 많으면 무조건 좋을까요? 인덱스의 단점과 적정 개수를 결정하는 기준은 무엇인가요?"
